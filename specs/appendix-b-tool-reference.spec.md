spec: task
name: "附录 B. 工具速查表"
inherits: project
tags: [appendix, reference, tools]
depends: [ch06-tool-system]
estimate: 0.5d
---

## 意图

提供 14 个内置工具的速查表，包括名称、参数、分组归属和使用示例，
方便用户和贡献者快速查阅。

## 决策

- 从 `crates/octos-agent/src/tools/` 源码提取工具定义
- 表格格式：工具名 | 分组 | 参数列表 | 一句话描述
- 每个分组（group:fs/web/search/sessions/runtime）单独列出成员

## 边界

### 允许修改
- octos-book/chapters/appendix-b-*.md

### 禁止做
- 不编造工具参数

## 完成条件

场景: 工具表完整
  测试: review_appendix_b_tools
  当 检查工具速查表
  那么 列出了所有 14+ 个内置工具
  并且 每个工具有名称、分组、参数签名、一句话描述

场景: 分组策略表
  测试: review_appendix_b_groups
  当 检查分组策略表
  那么 列出了所有工具分组（group:fs/web/search/sessions/runtime）
  并且 每个分组列出了包含的工具成员
