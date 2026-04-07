spec: project
name: "octos-book"
tags: [book, architecture, rust, ai-agent]
---

## 意图

编写《构建 AI Agent OS：octos 架构与实现》技术书籍，DDIA 风格，
面向 Rust 初学者、资深 Rust 开发者、AI/LLM 应用开发者、octos 贡献者四类读者。
中文为主，技术术语保留英文。

## 约束

- 每章 5000-10000 字（不含代码），代码片段不超过章节篇幅 30%
- 代码引用必须标注源文件路径和行号范围
- 架构图/数据流图/状态机图使用 Mermaid 格式
- 每章开头 1-2 段说明本章解决什么问题及对应读者类型
- 关键设计决策使用「工程决策侧栏」格式高亮
- 中文为主，类型名、trait 名、crate 名、CLI 命令保留英文
- 不编造源码中不存在的实现细节，必须基于实际代码
- 每章结尾提供「延伸阅读」和「思考题」

## 决策

- 书籍目录: `octos-book/`
- 章节文件: `octos-book/chapters/ch{NN}-{slug}.md`
- 图片资源: `octos-book/assets/`
- 源码仓库: `/Users/zhangalex/Work/Projects/FW/octos`
- Mermaid 图表内嵌在 markdown 中

## 边界

### 允许修改
- octos-book/**

### 禁止做
- 不修改 octos 源码仓库
- 不添加与书籍内容无关的文件
- 不使用 AI 生成的虚假代码示例

## 排除范围

- 不涉及 octos 的商业运营策略
- 不涉及具体云厂商部署的 step-by-step 教程
- 不涉及 LLM 模型原理（只讲集成层）

## 完成条件

场景: 章节格式规范
  测试: review_project_chapter_format
  假设 任意一个已完成的章节
  当 检查章节格式
  那么 字数在 5000-10000 字之间（不含代码）
  并且 代码片段不超过章节篇幅 30%
  并且 代码引用标注了源文件路径和行号范围
  并且 架构图/数据流图/状态机图使用 Mermaid 格式

场景: 章节结构规范
  测试: review_project_chapter_structure
  假设 任意一个已完成的章节
  当 检查章节结构
  那么 开头 1-2 段说明了本章解决什么问题及对应读者类型
  并且 关键设计决策使用了「工程决策侧栏」格式
  并且 结尾提供了「延伸阅读」和「思考题」

场景: 语言规范
  测试: review_project_language
  假设 任意一个已完成的章节
  当 检查语言使用
  那么 正文为中文
  并且 类型名、trait 名、crate 名、CLI 命令保留英文

场景: 内容真实性
  测试: review_project_authenticity
  假设 任意一个已完成的章节中引用了源码
  当 对照 octos 仓库检查引用的代码片段
  那么 代码片段来自实际源码而非编造
  并且 文件路径和行号范围可以在仓库中找到
