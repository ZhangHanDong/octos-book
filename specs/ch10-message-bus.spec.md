spec: task
name: "Ch10. octos-bus：14 频道的统一消息抽象"
inherits: project
tags: [part3, bus, channel, session, coalescing]
depends: [ch05-agent-loop]
estimate: 1.5d
---

## 意图

octos-bus（约 19600 行）是 Agent 连接外部世界的消息枢纽。本章展示如何用
`Channel` trait 统一 14 种迥异的消息协议，消息 Coalescing 的 5 级切割算法，
以及 JSONL Session 的持久化设计。

## 决策

- 源码目录: `crates/octos-bus/src/`
- 重点文件: `channel.rs`, `bus.rs`, `coalesce.rs`, `session.rs`
- 频道示例: `telegram_channel.rs`, `discord_channel.rs`（选 2 个代表性频道深入）
- 图表: Channel trait 类继承图、Coalescing 5 级切割流程图、Session 生命周期
- 工程决策侧栏: 为什么 JSONL 而非 SQLite

## 边界

### 允许修改
- octos-book/chapters/ch10-*.md
- octos-book/assets/ch10-*

### 禁止做
- 不逐个讲解 14 个频道的完整实现（选 2-3 个代表性深入）
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

场景: JSONL 选型侧栏
  测试: review_ch10_jsonl_sidebar
  当 阅读工程决策侧栏
  那么 对比了 JSONL vs SQLite vs 单 JSON 文件在追加写入、崩溃恢复、并发上的差异
  并且 解释了 JSONL 在 Session 场景下的优势
