spec: task
name: "Ch2. octos-core：用类型系统定义领域语言"
inherits: project
tags: [part1, core, types, domain-modeling]
depends: [ch01-why-rust-why-agent-os]
estimate: 1d
---

## 意图

深入 octos-core crate（约 1800 行），展示如何用 Rust 类型系统构建系统级
领域语言。Task 状态机、Message 抽象、Error 设计是整个 octos 的基石，
所有上层 crate 都依赖这些类型。本章让读者理解"零依赖 core crate"的设计哲学。

## 决策

- 源码目录: `crates/octos-core/src/`
- 重点文件: `task.rs`, `message.rs`, `error.rs`, `types.rs`, `utils.rs`
- 图表: Task 状态机 Mermaid 状态图
- 工程决策侧栏: 为什么 core crate 零内部依赖

## 边界

### 允许修改
- octos-book/chapters/ch02-*.md
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
  并且 每个状态转换有对应的源码方法引用（文件路径+行号）

场景: MessageRole 跨 Provider 设计解释清晰
  测试: review_ch02_message_role
  当 阅读本章 Message 小节
  那么 解释了 `as_str()` 和 `Display` impl 如何确保跨 Provider 一致性
  并且 包含实际的 Rust 代码片段（来自源码，非编造）

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
