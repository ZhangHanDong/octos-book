spec: task
name: "Ch10. octos-bus：消息总线、频道抽象与会话持久化"
inherits: project
tags: [part3, bus, channel, session, coalescing]
depends: [ch05-agent-loop]
estimate: 1.5d
---

## 意图

octos-bus 是 Agent 连接外部世界的消息枢纽。本章展示如何用
`Channel` trait 统一多种迥异的消息协议，消息 Coalescing 的 5 级切割算法，
以及 JSONL Session 的持久化设计。当前主分支还把 AppUI 多线程会话、
thread-bound streaming、message/persisted durable commit observer 和 child-session
contract 接入消息总线，本章需要补足这些生产化语义，而不是停留在“频道适配器”
层面。

## 决策

- 源码目录: `../octos/crates/octos-bus/src/`
- 重点文件: `channel.rs`, `bus.rs`, `coalesce.rs`, `session.rs`, `resume_policy.rs`
- 相邻接线点: `../octos/crates/octos-cli/src/api/events.rs`, `../octos/crates/octos-cli/src/api/ui_protocol.rs`, `../octos/crates/octos-cli/src/api/handlers.rs`
- 频道示例: `telegram_channel.rs`, `discord_channel.rs`（选 2 个代表性频道深入）
- 图表: Channel trait 类继承图、Coalescing 5 级切割流程图、Session 生命周期、thread-bound streaming 流程图、durable commit observer 流程图
- 工程决策侧栏: 为什么 JSONL 而非 SQLite

## 边界

### 允许修改
- octos-book/chapters/ch10-*.md
- octos-book/book/src/part3/ch10.md
- octos-book/book-en/src/part3/ch10.md
- octos-book/book/src/SUMMARY.md
- octos-book/book-en/src/SUMMARY.md
- octos-book/assets/ch10-*

### 禁止做
- 不逐个讲解所有频道的完整实现（选 2-3 个代表性深入）
- 不讲各消息平台的 API 文档

## 排除范围

- 各平台 Bot 注册流程
- 企业微信/飞书加密算法细节

## 完成条件

场景: Channel trait 抽象
  测试: review_ch10_channel_trait
  当 阅读 Channel trait 小节
  那么 展示了 `Channel` trait 的完整签名
  并且 以 Telegram 和 Discord 为例展示了两种不同平台的实现差异
  并且 说明 `edit_message_bound`、`finish_stream_bound`、`send_raw_sse_bound` 用显式 `thread_id` 绑定修复并发 turn 下的 sticky-thread 泄漏
  并且 说明 `health_check()` 是 dashboard 观测扩展点

场景: Coalescing 算法完整
  测试: review_ch10_coalescing
  当 阅读消息 Coalescing 小节
  那么 解释了 5 级切割策略（段落→换行→句子→空格→硬切）的优先级
  并且 说明了频道特定字符限制（Telegram 4000、Discord 1900）
  并且 解释了 MAX_CHUNKS=50 防 DoS 和 UTF-8 安全边界检测

场景: Session 管理完整
  测试: review_ch10_session
  当 阅读 Session 小节
  那么 解释了 JSONL 持久化格式和 LRU 内存缓存
  并且 说明了 `/new` fork 机制和 parent_key 追踪
  并且 解释了 percent-encoded 文件名 + hash 后缀防碰撞
  并且 说明了 10MB 限制和原子 write-then-rename
  并且 说明 rewrite temp path 当前使用 PID + 单调 counter，避免并发 rewrite 共享同一个 `.tmp`

场景: thread_id 持久化与 legacy synthesis
  测试: review_ch10_thread_id_persistence
  当 阅读 Session thread 小节
  那么 说明 User 新写入用 `client_message_id` 或 UUIDv7 派生 `thread_id`
  并且 说明 Assistant/Tool 新写入必须由调用方预先 stamp `thread_id`，否则 fail closed
  并且 说明 legacy JSONL load 会通过 `synthesize_thread_ids()` 在内存中补齐旧记录
  并且 区分 new write path 与 legacy load path 的不同安全策略

场景: message persisted observer
  测试: review_ch10_message_persisted_observer
  当 阅读 durable commit observer 小节
  那么 说明 `MessageCommitObserver` 在 append 成功且内存 mirror 更新后触发
  并且 说明 observer 失败或 panic 不会回滚已经提交的消息
  并且 说明该机制向 UI Protocol 派发 `message/persisted`
  并且 包含 durable commit observer Mermaid 图

场景: child-session contract
  测试: review_ch10_child_session_contract
  当 阅读 child session 小节
  那么 说明 `ChildSessionContract` 持久化 task_id、parent/child session key、terminal_state、join_state、failure_action 和 output_files
  并且 说明该 contract 支撑 background spawn / subagent lifecycle 回写父 session
  并且 不把 child session 写成只靠普通 message history 追踪

场景: JSONL 选型侧栏
  测试: review_ch10_jsonl_sidebar
  当 阅读工程决策侧栏
  那么 对比了 JSONL vs SQLite vs 单 JSON 文件在追加写入、崩溃恢复、并发上的差异
  并且 解释了 JSONL 在 Session 场景下的优势
