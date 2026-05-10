spec: task
name: "Ch4. octos-memory：混合搜索的工程实现"
inherits: project
tags: [part1, memory, search, bm25, hnsw, vector]
depends: [ch02-core-types]
estimate: 1d
---

## 意图

octos-memory（约 1700 行）实现了 Agent 的长期记忆。本章展示如何在不依赖
外部向量数据库的情况下，用纯 Rust 构建 BM25 + HNSW 混合搜索引擎。
对 AI 应用开发者来说，理解嵌入式混合搜索的工程实现极具参考价值。

## 决策

- 源码目录: `crates/octos-memory/src/`
- 重点文件: `hybrid_search.rs`, `episode.rs`, `memory_store.rs`, `store.rs`
- 图表: 混合搜索流程图（查询→BM25 排名 + 向量排名→权重融合→结果）
- 工程决策侧栏: 为什么不用 Qdrant/Milvus 等外部向量数据库

## 边界

### 允许修改
- octos-book/chapters/ch04-*.md
- octos-book/book/src/part1/ch04.md
- octos-book/book-en/src/part1/ch04.md
- octos-book/assets/ch04-*

### 禁止做
- 不讲 embedding 模型原理（只讲如何调用获取向量）
- 不讲 Agent 如何使用记忆（Ch5 覆盖）

## 排除范围

- 向量数据库横向对比评测
- Embedding 模型选型指南

## 完成条件

场景: redb 存储选型论证
  测试: review_ch04_redb_choice
  当 阅读存储选型小节
  那么 解释了 redb 的嵌入式特性及其与 SQLite 的差异
  并且 说明了选择 redb 而非 SQLite 的具体原因

场景: 源码覆盖范围明确
  测试: review_ch04_source_scope
  当 检查章节源码引用
  那么 说明源码目录是 `crates/octos-memory/src/`
  并且 覆盖 `hybrid_search.rs`, `episode.rs`, `memory_store.rs`, `store.rs`
  并且 说明这些文件在记忆系统中的职责

场景: BM25 实现细节准确
  测试: review_ch04_bm25
  当 阅读 BM25 小节
  那么 解释了 K1=1.2, B=0.75 参数的含义和选择理由
  并且 说明了 epsilon 防 NaN 的工程措施

场景: HNSW 向量索引解释清晰
  测试: review_ch04_hnsw
  当 阅读 HNSW 小节
  那么 解释了 HNSW 索引的层级结构和搜索过程
  并且 说明了 L2 归一化与 cosine similarity 的关系
  并且 引用了 `hnsw_rs` crate 的具体用法

场景: 混合排名融合机制
  测试: review_ch04_hybrid_ranking
  当 阅读混合排名小节
  那么 解释了 `with_weights()` 的权重配置（默认 0.7 向量 / 0.3 BM25）
  并且 包含 Mermaid 流程图展示查询→双路排名→融合→结果
  并且 说明了无 embedding provider 时的 BM25-only 降级策略

场景: Episode Store 生命周期
  测试: review_ch04_episode_store
  当 阅读 Episode Store 小节
  那么 解释了任务完成摘要的写入时机和格式
  并且 说明了 7 天窗口记忆的淘汰机制
  并且 说明 embedding 可后写入、删除使用 tombstone 跳过 HNSW 索引重排

场景: Memory Bank 二级检索
  测试: review_ch04_memory_bank
  当 阅读 MemoryStore 小节
  那么 说明 Memory Bank 的 entity 摘要会注入系统提示
  并且 说明 `recall_memory` 加载完整 entity 页面、`save_memory` 写入/更新 entity

场景: 嵌入式 vs 外部向量库侧栏
  测试: review_ch04_embedded_vs_external
  当 阅读工程决策侧栏
  那么 对比了嵌入式方案与 Qdrant/Milvus 在部署复杂度、性能、运维上的差异
  并且 说明了 octos 场景下嵌入式方案足够的理由
