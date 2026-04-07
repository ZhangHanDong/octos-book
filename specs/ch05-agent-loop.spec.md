spec: task
name: "Ch5. Agent Loop：一次对话的完整生命周期"
inherits: project
tags: [part2, agent, loop, runtime]
depends: [ch03-llm-providers, ch04-memory-search]
estimate: 1.5d
---

## 意图

octos-agent 是整个系统的心脏（约 32000 行）。本章聚焦 `agent.rs` 的核心循环，
逐段走读从消息构建到工具调用再到返回结果的完整流程。这是全书最核心的一章，
读者理解了 Agent Loop 就理解了 octos 的灵魂。

## 决策

- 源码入口: `crates/octos-agent/src/agent.rs`
- 辅助文件: `turn.rs`, `loop_detect.rs`, `bootstrap.rs`
- 源码走读: 选取 `agent.rs` 核心 ~200 行主循环逐段注释
- 图表: Agent Loop 状态流转图（Mermaid）、stop_reason 决策树

## 边界

### 允许修改
- octos-book/chapters/ch05-*.md
- octos-book/assets/ch05-*

### 禁止做
- 不深入单个工具的实现（Ch6 覆盖）
- 不深入沙箱机制（Ch7 覆盖）
- 不深入上下文压缩细节（Ch8 覆盖）

## 排除范围

- 具体 LLM API 请求构建细节（Ch3 已覆盖）
- 频道消息路由（Ch10 覆盖）

## 完成条件

场景: 主循环完整呈现
  测试: review_ch05_main_loop
  当 阅读主循环小节
  那么 包含 `agent.rs` 核心循环的实际代码片段（标注行号）
  并且 逐段解释每个阶段：消息构建→LLM 调用→流式消费→决策

场景: stop_reason 决策树清晰
  测试: review_ch05_stop_reason
  当 阅读 stop_reason 小节
  那么 包含 Mermaid 决策树图展示 EndTurn/ToolUse/MaxTokens 三个分支
  并且 每个分支解释了后续处理逻辑

场景: 迭代上限与预算管理
  测试: review_ch05_budget
  当 阅读预算管理小节
  那么 解释了 50 次迭代上限（`max_iterations`）的配置和触发条件
  并且 说明了 token 预算的计算与耗尽处理

场景: 循环检测机制
  测试: review_ch05_loop_detect
  当 阅读循环检测小节
  那么 解释了 `loop_detect.rs` 如何识别 Agent 重复行为
  并且 说明了检测到循环后的处理策略

场景: 源码走读有教学价值
  测试: review_ch05_code_walkthrough
  当 阅读源码走读小节
  那么 代码片段来自实际 `agent.rs` 而非编造
  并且 每段代码有中文注释解释设计意图
  并且 读者可以对照源码仓库找到对应位置
