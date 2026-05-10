spec: task
name: "Ch8. 上下文管理：让 Agent 在有限窗口中高效工作"
inherits: project
tags: [part2, context, compaction, fidelity, steering]
depends: [ch05-agent-loop]
estimate: 1d
---

## 意图

LLM context window 是稀缺资源。本章展示 octos 如何通过 compaction、
fidelity 分级、steering、prompt guard、typed summary 和 workspace contract
让 Agent 在有限窗口中高效工作而不丢失关键上下文。当前主分支中
compaction 也是 Agent Loop 的 error recovery 路径，因此本章需要承接 Ch5
的 `CompactAndRetry`，解释压缩、validator、workspace policy 如何共同防止
“压缩后丢任务约束”。

## 决策

- 源码文件: `../octos/crates/octos-agent/src/agent/compaction.rs`, `../octos/crates/octos-agent/src/agent/loop_runner.rs`, `../octos/crates/octos-agent/src/compaction.rs`, `../octos/crates/octos-agent/src/compaction_tiered.rs`, `../octos/crates/octos-agent/src/summarizer.rs`, `../octos/crates/octos-agent/src/workspace_contract.rs`, `../octos/crates/octos-agent/src/workspace_git.rs`, `../octos/crates/octos-agent/src/validators.rs`, `../octos/crates/octos-agent/src/prompt_guard.rs`
- 图表: Compaction 触发流程图、Fidelity 四档对比表、contract-gated tiered compaction 图
- 工程决策侧栏: 为什么 80% 而非动态阈值
- 章节边界: Ch5 只介绍 `CompactAndRetry` 触发点，本章负责解释 compaction policy、summary、validator ledger 和 workspace contract 的完整机制

## 边界

### 允许修改
- octos-book/chapters/ch08-*.md
- octos-book/book/src/part2/ch08.md
- octos-book/book-en/src/part2/ch08.md
- octos-book/assets/ch08-*

### 禁止做
- 不重复讲解 Agent Loop（Ch5 已覆盖）
- 不深入 LLM token 计算原理

## 排除范围

- LLM tokenizer 内部原理
- Prompt engineering 通用技巧

## 完成条件

场景: Compaction 策略完整
  测试: review_ch08_compaction
  Level: source-review
  Targets: ../octos/crates/octos-agent/src/agent/compaction.rs, ../octos/crates/octos-agent/src/compaction.rs
  当 阅读 compaction 小节
  那么 解释了 80% 触发阈值的计算方式
  并且 说明了保留最近 6 条消息的策略
  并且 展示了工具参数剥离和首行摘要的实现

场景: Fidelity 四档对比清晰
  测试: review_ch08_fidelity
  当 阅读 fidelity 小节
  那么 包含 Full/Truncate/Compact/Summary 四档对比表
  并且 每档说明了保留什么、丢弃什么、适用场景
  并且 不把 Summary 级别误写成未来预留，而是说明当前已有 LLM iterative typed summary，但默认仍是 extractive fallback

场景: Steering 机制
  测试: review_ch08_steering
  当 阅读 steering 小节
  那么 解释了如何系统化地构建提示层
  并且 说明了 prompt_layer 的分层结构

场景: Prompt Guard 防篡改
  测试: review_ch08_prompt_guard
  当 阅读 prompt guard 小节
  那么 解释了系统提示防篡改的检测机制
  并且 说明了触发后的处理策略

场景: CompactAndRetry 与上下文压缩衔接
  测试: review_ch08_compact_and_retry_bridge
  Level: source-review
  Targets: ../octos/crates/octos-agent/src/agent/loop_runner.rs, ../octos/crates/octos-agent/src/agent/compaction.rs
  当 阅读 compaction 入口小节
  那么 说明 Ch5 中 `CompactAndRetry` 是 context overflow 的 loop-level recovery
  并且 说明本章分析的是 turn compaction helper 之后的压缩策略
  并且 不把 compaction 仅描述为 token 预算优化

场景: typed SessionSummary 与 runtime state 边界
  测试: review_ch08_session_summary_boundary
  当 阅读 summary 小节
  那么 说明 `SessionSummary` 是 typed compaction summary 而不是长期记忆
  并且 区分 prompt-visible state、runtime control state 和 durable evidence state
  并且 解释 retry bucket 不应该被误写为 prompt 文本状态

场景: contract-gated tiered compaction
  测试: review_ch08_contract_gated_compaction
  Level: source-review
  Targets: ../octos/crates/octos-agent/src/compaction.rs, ../octos/crates/octos-agent/src/compaction_tiered.rs, ../octos/crates/octos-agent/src/agent/compaction.rs
  当 阅读 contract-gated compaction 小节
  那么 解释 tiered compactor 的三层：tier1 本地 tool-result placeholder、tier2 Anthropic `context_management` payload、tier3 full compactor
  并且 说明 tier1 保留 `tool_call_id` 且支持 protected IDs
  并且 说明 tier3 full compactor 保留最近消息、折叠历史和检查 contract-critical artifacts
  并且 说明 workspace policy / validator 约束在压缩后仍需要可恢复
  并且 包含 contract-gated tiered compaction Mermaid 图

场景: validator preservation 与 evidence ledger
  测试: review_ch08_validator_preservation
  Level: source-review
  Targets: ../octos/crates/octos-agent/src/validators.rs, ../octos/crates/octos-agent/src/workspace_git.rs
  当 阅读 validator preservation 小节
  那么 说明 validator outcome 通过 JSONL ledger 持久化
  并且 说明 required validator failure 会阻止 terminal success
  并且 说明 optional validator failure 只产生 warning
  并且 指向 Ch9/Ch12 继续展开 validator runner 的安全执行路径

场景: 80% 阈值侧栏
  测试: review_ch08_threshold_sidebar
  当 阅读工程决策侧栏
  那么 对比了固定阈值 vs 动态阈值 vs 预测式的方案
  并且 解释了 80% 固定阈值在简单性与有效性间的平衡
