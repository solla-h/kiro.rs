# Event Stream 协议参考

## 1. 核心摘要

AWS Event Stream 是一种二进制帧协议，用于流式传输结构化消息。每个帧包含固定 12 字节 Prelude、变长 Headers、变长 Payload 和 4 字节 Message CRC。协议使用大端序编码所有多字节整数，通过 CRC-32/ISO-HDLC 双重校验保证数据完整性。

## 2. 帧格式字段说明

| 字段 | 偏移 | 大小 | 说明 |
|------|------|------|------|
| Total Length | 0 | 4B | 整帧总字节数（含自身），范围 16 ~ 16MB |
| Header Length | 4 | 4B | Headers 区域字节数 |
| Prelude CRC | 8 | 4B | 前 8 字节的 CRC32 校验和 |
| Headers | 12 | 变长 | 键值对序列，长度由 Header Length 指定 |
| Payload | 12 + Header Length | 变长 | 消息体（通常为 JSON） |
| Message CRC | Total Length - 4 | 4B | 前 (Total Length - 4) 字节的 CRC32 校验和 |

CRC 算法：CRC-32/ISO-HDLC（多项式 0xEDB88320），实现见 `src/kiro/parser/crc.rs`。

## 3. Header 编码格式

每个 Header Entry 的二进制布局：

| 字段 | 大小 | 说明 |
|------|------|------|
| Name Length | 1B | 头部名称的字节长度 |
| Name | 变长 | UTF-8 编码的头部名称 |
| Value Type | 1B | 值类型标识（0-9） |
| Value | 变长 | 按类型编码的值 |

### Header 值类型

| 类型 ID | 名称 | 值大小 | Rust 类型 |
|---------|------|--------|-----------|
| 0 | BoolTrue | 0B | `bool` |
| 1 | BoolFalse | 0B | `bool` |
| 2 | Byte | 1B | `i8` |
| 3 | Short | 2B (big-endian) | `i16` |
| 4 | Integer | 4B (big-endian) | `i32` |
| 5 | Long | 8B (big-endian) | `i64` |
| 6 | ByteArray | 2B 长度前缀 + 数据 | `Vec<u8>` |
| 7 | String | 2B 长度前缀 + UTF-8 数据 | `String` |
| 8 | Timestamp | 8B (big-endian) | `i64` |
| 9 | Uuid | 16B | `[u8; 16]` |

类型定义见 `src/kiro/parser/header.rs` (`HeaderValueType`, `HeaderValue`)。

## 4. 事件类型分发规则

事件分发为两级路由，实现见 `src/kiro/model/events/base.rs` (`Event::from_frame`)：

**第一级：`:message-type` 头部**

| 值 | 处理方法 | 说明 |
|----|---------|------|
| `"event"` | `parse_event` | 正常业务事件，进入第二级分发 |
| `"error"` | `parse_error` | 服务端错误，读取 `:error-code` 头部和 payload |
| `"exception"` | `parse_exception` | 服务端异常，读取 `:exception-type` 头部和 payload |
| 其他 | 返回 `InvalidMessageType` 错误 | — |

**第二级：`:event-type` 头部**（仅 `message-type = "event"` 时）

| 值 | EventType 枚举 | 事件结构体 |
|----|---------------|-----------|
| `"assistantResponseEvent"` | `AssistantResponse` | `AssistantResponseEvent` |
| `"toolUseEvent"` | `ToolUse` | `ToolUseEvent` |
| `"meteringEvent"` | `Metering` | `()` (无 payload) |
| `"contextUsageEvent"` | `ContextUsage` | `ContextUsageEvent` |
| 其他 | `Unknown` | `Event::Unknown {}` |

## 5. 信息来源

- **帧解析:** `src/kiro/parser/frame.rs` — `parse_frame` 纯函数和 `Frame` 结构体。
- **头部解析:** `src/kiro/parser/header.rs` — `parse_headers`、`HeaderValueType`、`HeaderValue`、`Headers`。
- **CRC 校验:** `src/kiro/parser/crc.rs` — `crc32` 函数，使用 `crc` crate 的 `CRC_32_ISO_HDLC`。
- **错误定义:** `src/kiro/parser/error.rs` — `ParseError` 枚举（12 种变体）。
- **事件分发:** `src/kiro/model/events/base.rs` — `Event::from_frame`、`EventType`、`EventPayload` trait。
- **解码器架构:** `/llmdoc/architecture/event-stream-parser-architecture.md`
- **外部规范:** `https://docs.aws.amazon.com/transcribe/latest/dg/event-stream.html`
