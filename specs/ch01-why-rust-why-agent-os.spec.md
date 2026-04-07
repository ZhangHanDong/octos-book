spec: task
name: "Ch1. 为什么是 Rust？为什么是 Agent OS？"
inherits: project
tags: [part1, introduction, architecture]
depends: []
estimate: 1d
---

## 意图

作为全书开篇，阐明 octos 项目存在的理由：多租户 AI Agent 平台面临安全隔离、
并发、性能三大挑战，以及为什么 Rust 是解决这些挑战的最佳选择。引导读者理解
10 crate workspace 的分层拓扑，为后续章节建立全局地图。

## 决策

- 源码入口: `Cargo.toml`（workspace 定义）、各 crate 的 `Cargo.toml`
- 对比维度: Python(langchain/autogen) vs Go vs Rust — 安全、并发、性能、生态
- 架构图: 10 crate 依赖拓扑 Mermaid 图
- 工程决策侧栏: mono-repo vs multi-repo

## 边界

### 允许修改
- octos-book/chapters/ch01-*.md
- octos-book/assets/ch01-*

### 禁止做
- 不深入任何单个 crate 的实现细节（后续章节覆盖）
- 不做 Rust 语言教程（假设读者有基础或会查阅）

## 排除范围

- Rust 语法教学
- 具体 LLM API 调用细节

## 完成条件

场景: 问题空间完整阐述
  测试: review_ch01_problem_space
  假设 读者是不了解 Agent OS 的开发者
  当 阅读本章「问题空间」小节
  那么 能理解多租户 AI Agent 平台的三大核心挑战（安全隔离、并发、性能）
  并且 每个挑战有具体的真实场景举例

场景: 语言选型论证有说服力
  测试: review_ch01_language_choice
  假设 读者熟悉 Python 或 Go
  当 阅读本章「语言选型」小节
  那么 能理解 Rust 在安全性、并发模型、性能三方面的具体优势
  并且 论证包含具体数据或代码对比而非泛泛而谈

场景: Workspace 拓扑清晰
  测试: review_ch01_workspace_topology
  假设 读者首次接触 octos 代码库
  当 阅读本章「Workspace 策略」小节
  那么 能说出 10 个 crate 的名称和一句话职责描述
  并且 理解它们之间的依赖方向（上层依赖下层）

场景: 架构图准确
  测试: review_ch01_architecture_diagram
  当 检查本章的 Mermaid 依赖拓扑图
  那么 图中的 crate 依赖关系与 `Cargo.toml` workspace 实际定义一致
  并且 无虚假依赖边

场景: 工程决策侧栏有深度
  测试: review_ch01_sidebar
  当 阅读「mono-repo vs multi-repo」侧栏
  那么 包含至少两个 alternative 的利弊分析
  并且 解释 octos 选择 workspace 的具体原因
