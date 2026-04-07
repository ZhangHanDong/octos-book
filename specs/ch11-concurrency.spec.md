spec: task
name: "Ch11. 并发模型：Tokio 异步架构实战"
inherits: project
tags: [part3, concurrency, tokio, async, mutex]
depends: [ch05-agent-loop, ch10-message-bus]
estimate: 1d
---

## 意图

octos 的并发模型是其性能和可靠性的基石。本章展示 per-message spawn、
per-session Mutex 序列化、信号量限流、工具并发执行等机制如何协同工作，
是 Rust 异步编程的生产级实战教材。

## 决策

- 源码分散在多个 crate，以模式为主线而非文件为主线
- 重点代码: Agent spawn 逻辑、session Mutex、join_all 工具并发、AtomicBool 关停
- 图表: 并发模型全景图（Mermaid）、消息处理并发/串行分界
- 工程决策侧栏: per-session Mutex 而非 actor model 的取舍

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

场景: per-message spawn 模型
  测试: review_ch11_message_spawn
  当 阅读 per-message spawn 小节
  那么 解释了每条消息创建独立 task 的原因
  并且 展示了 `tokio::spawn()` 的实际调用代码

场景: per-session Mutex 串行化
  测试: review_ch11_session_mutex
  当 阅读 session Mutex 小节
  那么 解释了为什么同一 session 的消息必须串行处理
  并且 说明了 Mutex 的持有范围和死锁防护

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

场景: 优雅关停
  测试: review_ch11_graceful_shutdown
  当 阅读优雅关停小节
  那么 解释了 `AtomicBool` 的 Release/Acquire 内存序
  并且 展示了 shutdown 信号从接收到传播的完整链路

场景: actor model 侧栏
  测试: review_ch11_actor_sidebar
  当 阅读工程决策侧栏
  那么 对比了 Mutex 模式 vs actor model (actix/ractor) vs channel-based
  并且 解释了 octos 选择 Mutex 的简洁性优势和潜在限制
