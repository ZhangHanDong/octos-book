spec: task
name: "Ch3. octos-llm：驯服 LLM Provider 的混乱"
inherits: project
tags: [part1, llm, provider, failover, streaming]
depends: [ch02-core-types]
estimate: 1.5d
---

## 意图

octos-llm（约 13000 行）是整个系统与外部 LLM 的桥梁。本章展示如何用 Rust
trait 抽象统一少数专用协议实现和 10+ 种兼容 Provider，以及如何构建三层容错链
实现生产级可靠性。SSE 流式解析的工程细节对 AI 应用开发者尤其有价值。

## 决策

- 源码目录: `crates/octos-llm/src/`
- 重点文件: `provider.rs`, `retry.rs`, `failover.rs`, `adaptive.rs`, `sse.rs`, `anthropic.rs`, `openai.rs`
- 图表: 三层容错链流程图、Provider 注册表一览表
- 工程决策侧栏: `Arc<dyn Trait>` 的选择与 trait object 的代价

## 边界

### 允许修改
- octos-book/chapters/ch03-*.md
- octos-book/assets/ch03-*

### 禁止做
- 不讲各 LLM 的 API 文档细节（读者查官方文档）
- 不讲 Agent Loop 如何调用 LLM（Ch5 覆盖）

## 排除范围

- 各 LLM 模型的能力对比
- Prompt engineering 技巧

## 完成条件

场景: LlmProvider trait 抽象清晰
  测试: review_ch03_provider_trait
  当 阅读本章 trait 抽象小节
  那么 展示了 `LlmProvider` trait 的完整签名
  并且 解释了 `chat()` 方法如何统一不同 Provider 的请求/响应格式

场景: Provider 注册表完整
  测试: review_ch03_provider_registry
  当 阅读 Provider 注册表小节
  那么 列出了当前注册表中的 15 个 Provider 及其协议/别名/示例模型
  并且 解释了模型名自动检测机制（claude→anthropic, gpt→openai）
  并且 说明了专用实现与兼容适配的边界（如 Ollama 复用 OpenAI 兼容层）

场景: 三层容错链逐层讲解
  测试: review_ch03_failover_chain
  当 阅读三层容错链小节
  那么 分别解释了 RetryProvider、ProviderChain、AdaptiveRouter 的职责
  并且 包含 Mermaid 流程图展示请求在三层间的流转
  并且 AdaptiveRouter 部分包含 EMA 评分和 circuit breaker 机制

场景: SSE 流式解析工程细节
  测试: review_ch03_sse_parsing
  当 阅读 SSE 流式解析小节
  那么 解释了有状态解析器的设计（为什么不用无状态逐行解析）
  并且 说明了 1MB 缓冲上限的安全考量

场景: trait object 侧栏有深度
  测试: review_ch03_trait_object_sidebar
  当 阅读工程决策侧栏
  那么 对比了 `Arc<dyn Trait>` vs 泛型 `impl Trait` vs enum dispatch
  并且 解释了 octos 选择 trait object 的具体场景和代价（vtable 开销、无法内联）
