spec: task
name: "Ch5. Agent Loop：一次对话的完整生命周期"
inherits: project
tags: [part2, agent, loop, runtime]
depends: [ch03-llm-providers, ch04-memory-search]
estimate: 1.5d
---

## 意图

octos-agent 是整个系统的心脏。本章聚焦 `agent/loop_runner.rs` 的核心循环，
逐段走读从消息构建到工具调用再到返回结果的完整流程。这是全书最核心的一章，
读者理解了 Agent Loop 就理解了 octos 的灵魂。当前主分支还引入了
`HarnessError`、`LoopRetryState`、持久化 retry bucket 和 `CompactAndRetry`
路径，本章需要把 Agent Loop 从“循环调用 LLM/工具”升级为 typed recovery
state machine 的工程叙事。

## 决策

- 源码入口: `../octos/crates/octos-agent/src/agent/loop_runner.rs`
- 辅助文件: `../octos/crates/octos-agent/src/harness_errors.rs`, `../octos/crates/octos-agent/src/agent/loop_state.rs`, `../octos/crates/octos-agent/src/agent/turn_state.rs`, `../octos/crates/octos-agent/src/loop_detect.rs`
- 源码走读: 以 `loop_runner.rs` 的主循环、`handle_loop_error_with_dispatch`、`PersistentRetryStateGuard` 为主，不再假设单一 `agent.rs` 承载完整 loop
- 图表: Agent Loop 状态流转图（Mermaid）、stop_reason 决策树、`HarnessError -> RecoveryHint -> LoopDecision` state diagram
- 事实边界: `ProviderUnavailable` 的 `RotateAndRetry` 当前没有 agent 内部 provider lane hook，书中必须说明当前 release 会 bail，lane rotation 由外层 provider chain 承担

## 边界

### 允许修改
- octos-book/chapters/ch05-*.md
- octos-book/book/src/part2/ch05.md
- octos-book/book-en/src/part2/ch05.md
- octos-book/assets/ch05-*

### 禁止做
- 不深入单个工具的实现（Ch6 覆盖）
- 不深入沙箱机制（Ch7 覆盖）
- 不深入上下文压缩细节（Ch8 覆盖）

## 排除范围

- 具体 LLM API 请求构建细节（Ch3 已覆盖）
- 频道消息路由（Ch10 覆盖）

## 完成条件

场景: 主循环完整呈现
  测试: review_ch05_main_loop
  当 阅读主循环小节
  那么 包含 `loop_runner.rs` 核心循环的实际代码片段（标注行号）
  并且 逐段解释每个阶段：消息构建→LLM 调用→流式消费→决策

场景: stop_reason 决策树清晰
  测试: review_ch05_stop_reason
  当 阅读 stop_reason 小节
  那么 包含 Mermaid 决策树图展示 EndTurn/ToolUse/MaxTokens 三个分支
  并且 每个分支解释了后续处理逻辑

场景: 迭代上限与预算管理
  测试: review_ch05_budget
  当 阅读预算管理小节
  那么 解释了 50 次迭代上限（`max_iterations`）的配置和触发条件
  并且 说明 `max_timeout` 是 activity timeout 而不是无条件 wall-clock kill
  并且 说明 idle progress timeout 的 300s 无进展保护
  并且 说明了 token 预算的计算与耗尽处理

场景: 循环检测机制
  测试: review_ch05_loop_detect
  当 阅读循环检测小节
  那么 解释了 `loop_detect.rs` 如何识别 Agent 重复行为
  并且 说明当前 `process_message` 检测到循环后不会继续执行同一批工具
  并且 说明 shell spiral 会先尝试通过 retry ledger 返回最近 shell 输出，否则返回去重后的 terminal warning

场景: 源码走读有教学价值
  测试: review_ch05_code_walkthrough
  当 阅读源码走读小节
  那么 代码片段来自实际 `loop_runner.rs`、`harness_errors.rs` 或 `loop_state.rs` 而非编造
  并且 每段代码有中文注释解释设计意图
  并且 读者可以对照源码仓库找到对应位置

场景: HarnessError 错误边界准确
  测试: review_ch05_harness_error_taxonomy
  当 阅读错误处理小节
  那么 解释 `HarnessError` 是 loop 的 typed error boundary
  并且 列出 RateLimited、ContextOverflow、ProviderUnavailable、Authentication、InvalidRequest、ContentFiltered、DelegateDepthExceeded、Internal 等关键 variant
  并且 说明每个 variant 有稳定 `variant_name()` 和唯一 `RecoveryHint`
  并且 不把错误处理写成字符串匹配或泛泛异常重试

场景: LoopRetryState 有界状态机
  测试: review_ch05_loop_retry_state
  当 阅读 retry state machine 小节
  那么 说明 `LoopRetryState` 为每个 `HarnessError` variant 维护独立 bucket 和 hard limit
  并且 说明超过 limit 返回 `Exhausted` 而不是继续重试
  并且 说明 `ContextOverflow` 映射为 `CompactAndRetry`
  并且 说明 shell spiral 虽不是 `HarnessError` 也进入同一 retry ledger
  并且 包含 `HarnessError -> RecoveryHint -> LoopDecision` Mermaid state diagram

场景: ProviderUnavailable 语义不过度承诺
  测试: review_ch05_provider_unavailable_boundary
  当 阅读 provider unavailable / rotate-and-retry 说明
  那么 说明 `ProviderUnavailable` 的恢复语义是 `SwitchProvider` / `RotateAndRetry`
  但是 明确指出当前 agent loop 内没有 in-band provider lane hook
  并且 说明当前 release 在该路径会 bail，由外层 provider chain 或调用者承担 lane rotation

场景: retry state 跨 turn 持久化
  测试: review_ch05_persistent_retry_state
  当 阅读持久化 retry state 小节
  那么 解释 `PersistentRetryStateGuard` 如何从共享 handle hydrate 并在 drop 时写回
  并且 区分 prompt-visible state、runtime control state 和 durable evidence state
  并且 引用 `tests/retry_state_persistence.rs` 作为跨 turn bucket 不重置的证据

场景: CompactAndRetry 已接入主循环
  测试: review_ch05_compact_and_retry
  当 阅读 context overflow 恢复路径
  那么 说明 `handle_loop_error_with_dispatch` 在 `CompactAndRetry` 分支调用 turn compaction helper
  并且 说明该路径是当前实现而非未来设计
  并且 指向 Ch8 展开 compaction policy 与 validator preservation
