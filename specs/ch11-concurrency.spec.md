spec: task
name: "Ch11. 并发模型：Tokio 异步架构实战"
inherits: project
tags: [part3, concurrency, tokio, async, mutex]
depends: [ch05-agent-loop, ch10-message-bus]
estimate: 1d
---

## 意图

octos 的并发模型是其性能和可靠性的基石。本章展示 session actor、
actor 内部消息任务、信号量限流、工具并发执行和 TaskSupervisor 等机制如何协同工作，
是 Rust 异步编程的生产级实战教材。当前主分支中 MCP server lifecycle、
background spawn lifecycle、harness events 和 swarm dispatch 都复用或投射到这些
生命周期概念，本章需要补足 `TaskSupervisor` 与 CLI/MCP/Harness 控制面的连接。

## 决策

- 源码分散在多个 crate，以模式为主线而非文件为主线
- 重点代码: session actor、Agent spawn 逻辑、join_all 工具并发、TaskSupervisor、MCP server lifecycle observer、AtomicBool 关停
- 图表: 并发模型全景图（Mermaid）、消息处理并发/串行分界、Task lifecycle projection 图
- 工程决策侧栏: 从共享 Mutex 到 Session Actor 的取舍

## 边界

### 允许修改
- octos-book/chapters/ch11-*.md
- octos-book/assets/ch11-*

### 禁止做
- 不做 Tokio 入门教程（假设读者有基础概念）
- 不重复 Agent Loop 细节（Ch5 已覆盖）

## 排除范围

- Tokio runtime 内部调度原理
- async-std 等其他异步运行时对比

## 完成条件

场景: session actor 分层并发
  测试: review_ch11_session_actor
  当 阅读 session actor 与分层 spawn 小节
  那么 解释了 Gateway dispatch、session actor、actor 内部 agent task、tool task 和后台 subagent 的层级关系
  并且 说明了同一 session 的核心状态由 actor 持有，而不是旧的 per-session Mutex 叙事
  并且 展示了 `tokio::spawn()` 在会话、消息、工具或后台任务中的实际调用语义

场景: 信号量限流
  测试: review_ch11_semaphore
  当 阅读信号量小节
  那么 解释了 `max_concurrent_sessions` 默认 10 的配置
  并且 说明了超限时的排队行为

场景: 工具并发执行
  测试: review_ch11_tool_concurrency
  当 阅读工具并发小节
  那么 解释了 `join_all` 在单次迭代内并行执行多工具的机制
  并且 说明了子 Agent 同步 vs 后台双模式的差异

场景: spawn_only 生命周期监督
  测试: review_ch11_task_supervisor
  当 阅读 TaskSupervisor 小节
  那么 说明了 `spawn_only` 后台任务由 TaskSupervisor 维护状态 ledger
  并且 列出 Spawned/Running/Completed/Failed/Cancelled 等关键状态
  并且 说明了默认 fan-out 上限 200 和 `OCTOS_MAX_CHILDREN_PER_PARENT`
  并且 说明了 workspace contract 与 artifact 验证先于 supervisor 状态更新

场景: 优雅关停
  测试: review_ch11_graceful_shutdown
  当 阅读优雅关停小节
  那么 解释了 `AtomicBool` 的 Release/Acquire 内存序
  并且 展示了 shutdown 信号从接收到传播的完整链路

场景: actor model 侧栏
  测试: review_ch11_actor_sidebar
  当 阅读工程决策侧栏
  那么 对比了共享 Mutex、完全无状态 spawn-per-message 和 Session Actor
  并且 解释了 octos 当前选择 Session Actor 的状态所有权优势和实现复杂度

场景: Task lifecycle 投射到 MCP 和 Harness
  测试: review_ch11_task_lifecycle_projection
  当 阅读 TaskSupervisor / lifecycle 小节
  那么 说明 MCP server 的 `run_octos_session` 通过 lifecycle observer 标记 Running、Verifying、Ready 或 Failed
  并且 说明外层 MCP caller 接收的是 session aggregate outcome，而不是内部工具事件流
  并且 说明 background spawn lifecycle 会通过 harness events / metrics 进入 operator 可观测面
  并且 包含 Task lifecycle projection Mermaid 图
