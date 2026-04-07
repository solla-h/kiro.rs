# System Prompt Sanitize 功能分析与方案设计

## 1. 背景与目标

- 用户使用 Claude Code 客户端通过 kiro-rs 代理访问模型服务
- **目标：** 隐藏 system prompt 中关于 "Claude Code" 和 "Anthropic" 的痕迹
- **动机：** 进行双盲实验，测试模型提供商的模型服务对不同客户端请求的差异表现
- Anthropic 可能对自家 Claude Code 的 system prompt 做了特殊兼容和优化，需要消除这种偏差

## 2. 数据流分析

请求流转路径：

```
Claude Code 客户端 (带 system prompt + user messages + tools)
  → handlers.rs (post_messages:178 / post_messages_cc:660)
    → converter.rs::convert_request()
      → build_history():593
        → system messages join 为单个 String (converter.rs:601-605)
        → [最佳插入点] ← 在这里插入 sanitize 逻辑
        → 追加 SYSTEM_CHUNKED_POLICY (converter.rs:609)
        → 注入 thinking 标签 (converter.rs:612-620)
        → 转为 user+assistant 历史对 (converter.rs:623-627)
```

## 3. 关键发现

- kiro-rs 自身注入的内容（`SYSTEM_CHUNKED_POLICY`、`WRITE_TOOL_DESCRIPTION_SUFFIX`、`EDIT_TOOL_DESCRIPTION_SUFFIX`、固定 assistant 回复 "I will follow these instructions."）均不包含 "Claude" 或 "Anthropic" 字样
- 所有 Claude/Anthropic 痕迹都来自 Claude Code 客户端发送的原始 system prompt 和 user messages
- `get_models` 端点返回的 `owned_by: "anthropic"` 和模型名中的 "Claude" 是返回给客户端的，不发往上游，无需处理
- Claude Code 不仅在 system prompt 中注入身份信息，在 user messages 中也会注入（如 `<identity>` 标签、CLAUDE.md 内容引用、`system-reminder` 标签等）

## 4. 最佳插入点分析

- **推荐位置：** `converter.rs` `build_history()` 中 system_content join 之后（`converter.rs:605` 之后）
- **优势：** system prompt 已合并为单个 String，只需一次替换；代理自身注入的内容还没追加，不会被误伤
- **备选位置1：** `handlers.rs` 中 `convert_request()` 调用之前修改 `payload.system` — 需要遍历 `Vec<SystemMessage>` 逐个替换，不够干净
- **备选位置2：** 新建 Axum 中间件 — 过度设计，不推荐

## 5. 风险评估表

| 风险 | 等级 | 说明 |
|------|------|------|
| 指令语义破坏 | 中高 | "Claude should use..." 简单替换后丢失主语，影响指令遵循 |
| 模型身份混乱 | 中 | 删除身份锚点后模型可能不确定自己是谁，但不影响编码能力 |
| user messages 泄漏 | 中 | 只处理 system prompt 不够，user messages 也有 Claude Code 痕迹 |
| 工具描述泄漏 | 低 | 内置工具描述不含品牌字样，但 MCP 工具描述不可控 |
| 误伤代码内容 | 低 | 文件路径、模型名中的 "Claude" 可能被误替换 |
| 性能影响 | 极低 | 字符串替换开销可忽略 |
| SYSTEM_CHUNKED_POLICY 被误伤 | 无 | 在 sanitize 之后才追加 |
| thinking 标签被误伤 | 无 | 同上 |

## 6. 可选方案

### 方案 A：仅替换 system prompt（最小改动）

- **改动范围：** 仅 `converter.rs` `build_history()` 中加几行代码
- **优点：** 风险最低，改动最小，适合快速验证
- **缺点：** user messages 中的 Claude Code 痕迹不会被处理，双盲实验可能有泄漏
- **插入位置：** `converter.rs:605` 之后，`converter.rs:609` 之前

### 方案 B：全量替换（system + user + tools）

- **改动范围：** `converter.rs` 中的 `build_history()`、`process_message_content()`、`convert_tools()` 三个函数
- **优点：** 覆盖面最全，双盲实验最严谨
- **缺点：** 改动较大，误伤风险更高
- **需要处理的位置：**
  1. `build_history()` (`converter.rs:593`) — system prompt
  2. `process_message_content()` (`converter.rs:312`) — user messages 中的 text 内容
  3. `convert_tools()` (`converter.rs:519`) — 工具描述
  4. `convert_assistant_message()` (`converter.rs:727`) — 历史 assistant 消息（可选）

### 方案 C：可配置替换规则

- **改动范围：** Config 模型 + `converter.rs` + `config.json`
- **优点：** 最灵活，用户可自定义替换规则，不用每次改代码重新编译
- **缺点：** 实现工作量最大
- **需要通过 `AppState` 或全局 Config 将规则传递到 `converter.rs`**

## 7. 实现建议

- **替换策略：** 用中性词替换而非删除（"Claude" → "Assistant"，"Anthropic" → "the provider"），避免破坏句子结构
- **匹配策略：** 精确模式优先（先替换 "Claude Code" 再替换 "Claude"，避免 "Claude Code" 被部分替换为 "Assistant Code"）
- **调试建议：** 先在 `handlers.rs` 中加日志打印 Claude Code 发来的完整 system prompt，确认具体有哪些需要替换的字符串后再决定最终方案
- **推荐路径：** 方案 A 快速验证 → 根据实验结果决定是否升级到方案 B 或 C

## 8. 待确认事项

- 需要先观察 Claude Code 实际发送的 system prompt 完整内容，确认所有需要替换的字符串模式
- 需要确认是否需要处理 user messages 中的身份信息泄漏
- 需要确认替换后模型行为是否有显著变化（可能需要 A/B 测试）

## 9. Claude Agent SDK 的 system_prompt 能力调研

### 9.1 背景

除了在 kiro-rs 代理层做字符串替换（sanitize），还有另一条路径：通过 Claude Agent SDK 在源头控制 system prompt。Agent SDK 是 Anthropic 官方提供的高层 SDK，用于构建基于 Claude Code 的 AI Agent。

### 9.2 SDK 支持的三种 system_prompt 模式

从 SDK 源码类型定义（`claude_agent_sdk/types.py`）确认：

```python
system_prompt: str | SystemPromptPreset | SystemPromptFile | None = None
```

**模式 1：直接字符串 — 完全替换默认 system prompt**

```python
options = ClaudeAgentOptions(
    system_prompt="You are a helpful assistant..."
)
```

传一个纯字符串，完全替换默认的 Claude Code system prompt。这是双盲实验最直接的方案。

**模式 2：Preset — 保留默认 + 追加**

```python
options = ClaudeAgentOptions(
    system_prompt={
        "type": "preset",
        "preset": "claude_code",
        "append": "额外的指令..."
    }
)
```

使用内置的 `claude_code` 预设（即默认 system prompt），可选追加自定义内容。`append` 是 `NotRequired` 字段。

**模式 3：文件加载**

```python
options = ClaudeAgentOptions(
    system_prompt={
        "type": "file",
        "path": "/path/to/system_prompt.txt"
    }
)
```

从文件读取 system prompt 内容。

TypeScript SDK 对应的类型：
```typescript
systemPrompt: string | {type: 'preset', preset: 'claude_code', append?: string}
```

### 9.3 相关配置项

- `setting_sources`：控制是否加载 CLAUDE.md 文件。默认 `none`（不加载）。设为 `["project"]` 会加载项目级 CLAUDE.md，可能包含 Claude 相关内容
- `allowed_tools`：控制可用工具列表。工具描述由 SDK 内部定义，不受 system_prompt 控制

### 9.4 对双盲实验的意义

- 模式 1（直接字符串）可以在源头完全控制 system prompt，去掉所有 Claude Code / Anthropic 痕迹
- 但工具描述、CLAUDE.md 等其他上下文仍可能包含品牌信息，需要额外处理
- 这是 Agent SDK 层面的能力，当前 kiro-claw 通过 ACP 协议调用 Claude Code CLI 进程，不直接使用 Agent SDK。要利用此能力需要改 kiro-claw 的 backend 实现或单独用 Agent SDK 做实验

### 9.5 与 kiro-rs sanitize 方案的关系

| 维度 | kiro-rs sanitize（方案 A/B/C） | Agent SDK system_prompt |
|------|-------------------------------|------------------------|
| 控制层级 | 中间代理层拦截 | 源头控制 |
| 覆盖范围 | 可处理所有经过代理的请求 | 仅控制 system prompt 部分 |
| 实现复杂度 | 需要修改 kiro-rs 代码 | 需要修改 kiro-claw backend 或独立实验 |
| 灵活性 | 可配置替换规则，处理 user messages 和 tools | 只能替换或追加 system prompt |
| 适用场景 | 所有通过 kiro-rs 的客户端 | 仅 Agent SDK 启动的 Claude Code 实例 |

两种方案互补：Agent SDK 在源头替换 system prompt，kiro-rs 在代理层清理残余的品牌信息（user messages、tool descriptions 等）。
