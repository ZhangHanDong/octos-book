spec: task
name: "Ch2. octos-core：用类型系统定义领域语言"
inherits: project
tags: [part1, core, types, domain-modeling]
depends: [ch01-why-rust-why-agent-os]
estimate: 1d
---

## 意图

深入 octos-core crate，展示如何用 Rust 类型系统构建系统级领域语言。
Task 状态机、Message 抽象、Error 设计是整个 octos 的基石，所有上层 crate
都依赖这些类型。当前主分支还把 AppUI / Gateway 所需的
`ClientMessageId`、`ThreadId`、`TurnId`、`SessionSummary` schema version
和 UI Protocol capabilities 放进 core，本章需要解释这些“跨 crate wire identity”
为什么属于 core，而不只是讲基础 enum/struct。

## 决策

- 源码目录: `crates/octos-core/src/`
- 重点文件: `task.rs`, `message.rs`, `error.rs`, `types.rs`, `utils.rs`, `ui_protocol.rs`
- 图表: Task 状态机 Mermaid 状态图、message identity 三元组图、SessionSummary ABI 图
- 工程决策侧栏: 为什么 core crate 零内部依赖

## 边界

### 允许修改
- octos-book/chapters/ch02-*.md
- octos-book/book/src/part1/ch02.md
- octos-book/book-en/src/part1/ch02.md
- octos-book/assets/ch02-*

### 禁止做
- 不讲 LLM 相关逻辑（Ch3 覆盖）
- 不讲工具系统（Ch6 覆盖）

## 排除范围

- Rust enum/struct 基础语法教学
- serde 序列化框架教学

## 完成条件

场景: Task 状态机完整呈现
  测试: review_ch02_task_state_machine
  当 阅读本章 Task 状态机小节
  那么 包含 Mermaid 状态图展示 Pending→InProgress→Blocked/Completed/Failed
  并且 Task、TaskStatus、TaskKind、TaskResult、TokenUsage 的说明都有对应源码引用（文件路径+行号）

场景: MessageRole 跨 Provider 设计解释清晰
  测试: review_ch02_message_role
  当 阅读本章 Message 小节
  那么 解释了 `as_str()` 和 `Display` impl 如何确保跨 Provider 一致性
  并且 包含实际的 Rust 代码片段（来自源码，非编造）

场景: Message identity 三元组准确
  测试: review_ch02_message_identity
  当 阅读 Message identity 小节
  那么 区分 `ClientMessageId`、`ThreadId`、`TurnId` 的职责
  并且 说明 `ClientMessageId` 是客户端乐观 UI / 幂等相关 token
  并且 说明 `ThreadId` 是渲染分组 key，user thread 通常 root at client_message_id
  并且 说明 `TurnId` 是 UI Protocol 的服务器协议 identity
  并且 包含三者关系 Mermaid 图
  并且 说明 `assistant_with_thread` / `tool_with_thread` 强制显式 thread 绑定，避免“最近 user”推导错误

场景: Durable schema version 解释清晰
  测试: review_ch02_schema_version
  当 阅读 TaskResult / SessionSummary 小节
  那么 说明 `TaskResult.schema_version` 是 durable ABI 字段
  并且 说明 `SessionSummary.schema_version` 缺失时默认到当前版本
  并且 说明未来版本会返回 `UnsupportedSessionSummaryVersion` typed error
  并且 连接到 Ch8 的 typed compaction 与 Ch9 的 Harness ABI

场景: SessionKey 语义不夸大
  测试: review_ch02_session_key
  当 阅读 SessionKey 小节
  那么 说明 `SessionKey` 支持 profile 与 topic 维度
  并且 说明当前 core 构造函数不做 channel 合法性拒绝
  并且 说明 `is_channel_name` 用于三段式 key 的 profile/channel 推断
  并且 不把 SessionKey 描述成“构造期让未知 channel 不可构造”

场景: Error 设计选型有说服力
  测试: review_ch02_error_design
  当 阅读本章 Error 小节
  那么 解释了选择 `eyre`/`color-eyre` 而非 `anyhow` 的具体理由
  并且 展示了 octos 中错误类型的实际定义

场景: truncate_utf8 案例深入浅出
  测试: review_ch02_truncate_utf8
  当 阅读 `truncate_utf8` 小节
  那么 解释了 UTF-8 多字节字符在截断时的陷阱
  并且 展示了 octos 的两个变体（in-place 和 copying）的实现差异

场景: 零依赖设计哲学
  测试: review_ch02_zero_dep_sidebar
  当 阅读工程决策侧栏
  那么 解释了 core crate 不依赖 workspace 内任何其他 crate 的原因
  并且 对比了"胖 core"与"瘦 core"两种策略的利弊
