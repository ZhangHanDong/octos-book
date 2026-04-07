spec: task
name: "Ch8. 上下文管理：让 Agent 在有限窗口中高效工作"
inherits: project
tags: [part2, context, compaction, fidelity, steering]
depends: [ch05-agent-loop]
estimate: 1d
---

## 意图

LLM context window 是稀缺资源。本章展示 octos 如何通过 compaction、
fidelity 分级、steering 和 prompt guard 四种机制，让 Agent 在有限窗口中
高效工作而不丢失关键上下文。

## 决策

- 源码文件: `compaction.rs`, `steering.rs`, `prompt_layer.rs`, `prompt_guard.rs`
- 图表: Compaction 触发流程图、Fidelity 四档对比表
- 工程决策侧栏: 为什么 80% 而非动态阈值

## 边界

### 允许修改
- octos-book/chapters/ch08-*.md
- octos-book/assets/ch08-*

### 禁止做
- 不重复讲解 Agent Loop（Ch5 已覆盖）
- 不深入 LLM token 计算原理

## 排除范围

- LLM tokenizer 内部原理
- Prompt engineering 通用技巧

## 完成条件

场景: Compaction 策略完整
  测试: review_ch08_compaction
  当 阅读 compaction 小节
  那么 解释了 80% 触发阈值的计算方式
  并且 说明了保留最近 6 条消息的策略
  并且 展示了工具参数剥离和首行摘要的实现

场景: Fidelity 四档对比清晰
  测试: review_ch08_fidelity
  当 阅读 fidelity 小节
  那么 包含 Full/Truncate/Compact/Summary 四档对比表
  并且 每档说明了保留什么、丢弃什么、适用场景

场景: Steering 机制
  测试: review_ch08_steering
  当 阅读 steering 小节
  那么 解释了如何系统化地构建提示层
  并且 说明了 prompt_layer 的分层结构

场景: Prompt Guard 防篡改
  测试: review_ch08_prompt_guard
  当 阅读 prompt guard 小节
  那么 解释了系统提示防篡改的检测机制
  并且 说明了触发后的处理策略

场景: 80% 阈值侧栏
  测试: review_ch08_threshold_sidebar
  当 阅读工程决策侧栏
  那么 对比了固定阈值 vs 动态阈值 vs 预测式的方案
  并且 解释了 80% 固定阈值在简单性与有效性间的平衡
