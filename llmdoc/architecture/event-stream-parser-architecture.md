# Event Stream 解析器架构

## 1. 身份

- **定义:** AWS Event Stream 二进制帧协议的流式解析器，将字节流解码为结构化事件。
- **用途:** 处理 `generateAssistantResponse` 端点的流式 HTTP 响应，支持容错恢复和 CRC 双重校验。

## 2. 核心组件

- `src/kiro/parser/frame.rs` (`Frame`, `parse_frame`, `PRELUDE_SIZE`, `MIN_MESSAGE_SIZE`, `MAX_MESSAGE_SIZE`): 无状态纯函数，从缓冲区解析单个二进制帧，执行 CRC 双重校验，返回 `Frame`（headers + payload）。
- `src/kiro/parser/decoder.rs` (`EventStreamDecoder`, `DecoderState`, `DecodeIter`): 有状态流式解码器，管理 `BytesMut` 缓冲区，实现四态状态机和差异化容错恢复。
- `src/kiro/parser/header.rs` (`Headers`, `HeaderValue`, `HeaderValueType`, `parse_headers`, `parse_header_value`): 解析帧头部区域，支持 10 种值类型，提供 `:message-type` 等保留头部的快捷访问。
- `src/kiro/parser/crc.rs` (`crc32`): CRC-32/ISO-HDLC 校验和计算（多项式 0xEDB88320）。
- `src/kiro/parser/error.rs` (`ParseError`, `ParseResult`): 12 种错误变体，覆盖数据不足、CRC 失败、长度越界、反序列化失败等场景。
- `src/kiro/model/events/base.rs` (`Event`, `EventType`, `EventPayload`): 事件分发层，按 `:message-type` 和 `:event-type` 头部将 `Frame` 转换为具体业务事件。

## 3. 执行流程（LLM 检索路径）

### 3.1 完整调用链

- **1. 数据注入:** `src/anthropic/handlers.rs` 从 HTTP 响应体读取 chunk，调用 `decoder.feed(&chunk)`。
- **2. 缓冲区管理:** `src/kiro/parser/decoder.rs:147-165` (`feed`) 追加数据到 `BytesMut`，检查 16MB 上限，`Recovering` 态自动转回 `Ready`。
- **3. 帧解析:** `src/kiro/parser/decoder.rs:173-229` (`decode`) 转入 `Parsing` 态，调用 `parse_frame`。
- **4. 纯函数解析:** `src/kiro/parser/frame.rs:75-154` (`parse_frame`) 执行：读 Prelude → 校验长度 → 验证 Prelude CRC → 验证 Message CRC → 解析 Headers → 提取 Payload。
- **5. 事件分发:** `src/kiro/model/events/base.rs:94-102` (`Event::from_frame`) 按 `:message-type` 分发到 `parse_event`/`parse_error`/`parse_exception`。
- **6. 二级分发:** `src/kiro/model/events/base.rs:106-126` (`parse_event`) 按 `:event-type` 分发到 `AssistantResponseEvent`、`ToolUseEvent`、`ContextUsageEvent` 等具体类型。

### 3.2 二进制帧格式

```
[Total Length: 4B][Header Length: 4B][Prelude CRC: 4B][Headers: 变长][Payload: 变长][Message CRC: 4B]
```

- Prelude = Total Length + Header Length（共 8B），Prelude CRC 覆盖这 8 字节
- Message CRC 覆盖从头到 Message CRC 字段之前的所有字节
- Payload 长度 = Total Length - 12 (Prelude) - Header Length - 4 (Message CRC)
- 所有多字节整数使用大端序，消息大小范围：16B ~ 16MB

### 3.3 四态状态机

```
Ready → (feed) → Ready → (decode) → Parsing
  Parsing → 成功 → Ready（error_count 归零）
  Parsing → 数据不足 → Ready
  Parsing → 失败 → error_count++ → Recovering（try_recover 跳过损坏数据）
  Parsing → 失败 → error_count >= 5 → Stopped（终止态）
  Recovering → (feed) → Ready
  Stopped → (try_resume) → Ready
```

### 3.4 容错恢复策略

`src/kiro/parser/decoder.rs:241-304` (`try_recover`) 实现差异化恢复：

- **Prelude 阶段错误**（`PreludeCrcMismatch`、`MessageTooSmall`、`MessageTooLarge`）：帧边界可能错位，跳过 1 字节逐步扫描。
- **Data 阶段错误**（`MessageCrcMismatch`、`HeaderParseFailed`）：帧边界正确但数据损坏，读取 `total_length` 跳过整帧；若无法确定帧长则回退到跳 1 字节。
- **其他错误**：跳过 1 字节。
- **终止条件**：连续错误达到 `max_errors`（默认 5）时进入 `Stopped` 态。

## 4. 设计要点

- `parse_frame` 设计为无状态纯函数，缓冲区生命周期完全由 `EventStreamDecoder` 管理，职责分离清晰。
- CRC 双重校验（Prelude CRC + Message CRC）确保帧头和帧体的完整性，Prelude CRC 先行校验可快速拒绝损坏帧。
- 容错策略区分 Prelude/Data 阶段错误，避免 Prelude 错位时跳过过多有效数据。
- `DecodeIter` 在 `Stopped` 或 `Recovering` 状态时停止迭代，防止错误扩散。
