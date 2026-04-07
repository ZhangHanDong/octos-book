spec: task
name: "附录 C. 配置参考"
inherits: project
tags: [appendix, reference, config]
depends: [ch13-runtime-modes]
estimate: 0.5d
---

## 意图

提供 `config.json` 所有字段的完整参考文档，包括类型、默认值、说明，
方便用户配置 octos。

## 决策

- 从 `crates/octos-cli/src/config.rs` 提取配置结构体定义
- 表格格式：字段路径 | 类型 | 默认值 | 说明
- 按功能分组（provider/model/tools/hooks/gateway/serve）

## 边界

### 允许修改
- octos-book/chapters/appendix-c-*.md

### 禁止做
- 不编造不存在的配置字段

## 完成条件

场景: 配置字段完整
  测试: review_appendix_c_fields
  当 检查配置参考表
  那么 覆盖了 `config.rs` 中定义的所有公开配置字段
  并且 每个字段有类型、默认值和功能说明

场景: 示例配置文件
  测试: review_appendix_c_example
  当 检查示例配置
  那么 提供了一个完整的 `config.json` 示例
  并且 示例中每个字段有行内注释
