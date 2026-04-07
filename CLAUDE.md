# CLAUDE.md — octos-book

## 项目概述

《构建 AI Agent OS：octos 架构与实现》技术书籍项目。DDIA 风格，中文为主，技术术语保留英文。

## 关键文件

- **大纲**: [OUTLINE.md](OUTLINE.md) — 全书 3 部分 14 章 5 附录结构
- **Spec 目录**: `specs/` — 20 个 agent-spec 任务合约，共 86 个验收场景
  - `specs/project.spec.md` — 全书约束（父 spec）
  - `specs/ch{01-14}-*.spec.md` — 各章任务合约
  - `specs/appendix-{a-e}-*.spec.md` — 附录任务合约
- **章节输出**: `chapters/` — 已完成的章节 markdown
- **图片资源**: `assets/` — Mermaid 图表和其他资源
- **源码仓库**: `/Users/zhangalex/Work/Projects/FW/octos` — octos 源码（只读引用，不修改）

## 写作工作流

### 必须使用 /tech-writer skill

**每次写作章节内容时，必须先调用 `/tech-writer` skill。** 这是强制要求，不可跳过。

```
1. 阅读目标章节的 spec: specs/ch{NN}-*.spec.md
2. 调用 /tech-writer skill 进入技术写作模式
3. 深读 octos 源码中对应的文件（spec 的「决策」段列出了源码路径）
4. 按 spec 的验收场景逐项完成写作
5. 写完后用 agent-spec verify 检查验收条件
```

### 写作规范（继承自 project.spec.md）

- 每章 5000-10000 字（不含代码），代码片段不超过章节篇幅 30%
- 代码引用标注源文件路径和行号范围
- 架构图/数据流图/状态机图使用 Mermaid 格式
- 每章开头 1-2 段说明本章解决什么问题及对应读者类型
- 关键设计决策使用「工程决策侧栏」格式高亮
- 每章结尾提供「延伸阅读」和「思考题」
- 不编造源码中不存在的实现细节

### 章节依赖顺序

```
Ch1 → Ch2 → Ch3 ─┐
                  ├→ Ch5 → Ch6 → Ch7
Ch1 → Ch2 → Ch4 ─┘       │       │
                          ├→ Ch9  │
                  Ch5 ────┤       │
                          ├→ Ch8  │
                          ├→ Ch10 → Ch11
                          └→ Ch12
                  Ch10 + Ch5 → Ch13 → Ch14
```

## 命令参考

```bash
# Lint 单个 spec
agent-spec lint specs/ch01-why-rust-why-agent-os.spec.md --min-score 0.7

# Parse spec 查看结构
agent-spec parse specs/ch01-why-rust-why-agent-os.spec.md

# 引用 octos 源码（只读）
cd /Users/zhangalex/Work/Projects/FW/octos
cargo doc --workspace --no-deps  # 生成 API 文档
```

## 读者定位

| 代号 | 读者类型 | 重点章节 |
|------|---------|---------|
| A | Rust 初学者 | Part 1 (Ch1-4) |
| B | 资深 Rust 开发者 | Part 2 (Ch5-9) |
| C | AI/LLM 应用开发者 | Ch1, Ch3, Ch5, Ch8 |
| D | octos 贡献者 | Part 3 (Ch10-14) + 附录 |
