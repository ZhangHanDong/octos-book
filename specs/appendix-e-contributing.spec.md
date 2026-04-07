spec: task
name: "附录 E. 从源码构建与贡献指南"
inherits: project
tags: [appendix, contributing, build]
depends: [ch01-why-rust-why-agent-os]
estimate: 0.5d
---

## 意图

帮助新贡献者从零开始构建 octos、运行测试、提交 PR。
包括环境要求、构建步骤、测试策略和代码规范。

## 决策

- 环境要求从 `rust-toolchain.toml` 和 `Cargo.toml` 的 `rust-version` 提取
- 构建命令与 CLAUDE.md 中的 Build & Test Commands 一致
- 包括 TDD 工作流（RED→GREEN→REFACTOR）

## 边界

### 允许修改
- octos-book/chapters/appendix-e-*.md

### 禁止做
- 不与 CLAUDE.md 中的构建命令矛盾

## 完成条件

场景: 构建步骤可执行
  测试: review_appendix_e_build
  当 新贡献者按照本附录步骤操作
  那么 能成功 clone、构建、运行测试
  并且 构建命令与 CLAUDE.md 一致

场景: 代码规范清晰
  测试: review_appendix_e_conventions
  当 阅读代码规范小节
  那么 覆盖了 edition、rust-version、error handling、lint 等关键规范
  并且 包含 TDD 工作流说明

场景: PR 提交指南
  测试: review_appendix_e_pr
  当 阅读 PR 指南小节
  那么 说明了分支命名、commit message、review 流程
