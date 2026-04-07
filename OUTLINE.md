# 《构建 AI Agent OS：octos 架构与实现》

> 中文为主，技术术语保留英文。DDIA 风格：前半叙事讲「为什么」，后半深入讲「怎么做」。

## 读者定位

- **A** Rust 初学者 — 通过实战项目学习 Rust
- **B** 有经验的 Rust 开发者 — 学习大型 AI Agent 系统的构建方法
- **C** AI/LLM 应用开发者 — 理解 Agent OS 的设计思路（不要求 Rust 背景）
- **D** octos 用户/贡献者 — 深入内部实现以便扩展和贡献

## 代码规模

- 10 个 crate，约 13 万行 Rust
- 14 个消息频道，91 个 REST 端点，14+ 内置工具

---

## 第一部分：地基 — Rust + AI Agent 的设计哲学

### Ch1. 为什么是 Rust？为什么是 Agent OS？

- 问题空间：多租户 AI Agent 平台面临的核心挑战（安全隔离、并发、性能）
- 语言选型：为什么不用 Python/Go — 性能、安全、类型系统的取舍
- Workspace 策略：10 个 crate 的分层拓扑与职责边界
- **工程决策侧栏**：mono-repo vs multi-repo，为什么选择 workspace

### Ch2. octos-core：用类型系统定义领域语言

- Task 状态机：Pending → InProgress → Blocked / Completed / Failed
- Message 与 MessageRole：跨 Provider 的一致性设计（`as_str()` + Display impl）
- Error 类型：`eyre`/`color-eyre` 而非 `anyhow` 的选择
- `truncate_utf8`：一个小函数背后的 UTF-8 安全哲学
- **工程决策侧栏**：为什么 core crate 零内部依赖

### Ch3. octos-llm：驯服 LLM Provider 的混乱

- `LlmProvider` trait：统一 4 种原生协议 + 10 种 OpenAI 兼容
- Provider 注册表：模型名自动检测（claude→anthropic，gpt→openai）
- 三层容错链：
  - `RetryProvider`：指数退避处理 429/5xx
  - `ProviderChain`：有序故障转移
  - `AdaptiveRouter`：EMA 评分、hedge racing、circuit breaker
- SSE 流式解析：有状态解析器与 1MB 缓冲上限
- **工程决策侧栏**：`Arc<dyn Trait>` 的选择与 trait object 的代价

### Ch4. octos-memory：混合搜索的工程实现

- 存储选型：redb 嵌入式数据库 vs SQLite
- BM25 参数调优：K1=1.2, B=0.75, epsilon 防 NaN
- HNSW 向量索引：L2 归一化 + cosine similarity
- 混合排名：可配置权重融合（默认 0.7 向量 / 0.3 BM25）
- Episode Store：任务完成摘要与 7 天窗口记忆
- **工程决策侧栏**：为什么不用 Qdrant/Milvus 等外部向量数据库

---

## 第二部分：引擎 — Agent 运行时深度解析

### Ch5. Agent Loop：一次对话的完整生命周期

- 主循环：消息构建 → LLM 调用 → 流式消费 → stop_reason 决策树
- stop_reason 分支：EndTurn / ToolUse / MaxTokens
- 50 次迭代上限与 token 预算管理
- 循环检测（loop_detect）：防止 Agent 陷入死循环
- **源码走读**：`agent.rs` 核心 200 行逐段解析

### Ch6. 工具系统：14 个内置工具的设计模式

- `Tool` trait：`spec() -> ToolSpec` + `execute(&Value) -> ToolResult`
- `ToolRegistry`：HashMap 注册 + 按需查找
- ToolPolicy：deny-wins 语义、通配符匹配（`exec*`）、分组（group:fs/web/search/sessions）
- Provider 级策略：`tools.byProvider` 配置
- LRU 工具淘汰：15 个活跃槽位 + 34 个候补，`spawn_only` 不可淘汰
- 参数安全：1MB 大小限制 + `estimate_json_size` 的非分配实现
- Symlink-safe I/O：`O_NOFOLLOW` 消除 TOCTOU 竞态
- **工程决策侧栏**：为什么 LRU 淘汰而非全量注册

### Ch7. 安全纵深：从沙箱到 Prompt 注入防御

- **沙箱三后端**：
  - Bwrap（Linux）
  - macOS sandbox-exec（SBPL 路径注入防护）
  - Docker（mount 模式、CPU/内存/PID 限制、网络隔离）
  - 自动检测与 NoSandbox 回退
- **SSRF 防护**：私有 IP 阻断、IPv6 ULA/链路本地、DNS 失败关闭
- **Prompt 注入检测**：5 类威胁识别
- **凭据脱敏**：7 个正则覆盖 OpenAI/AWS/GitHub/GitLab key
- **工具输出清理**：base64 URI、长十六进制、凭据赋值
- **基础设施安全**：`deny(unsafe_code)` + `SecretString` + `BLOCKED_ENV_VARS`(18 个)
- **ShellTool SafePolicy**：危险命令拒绝（rm -rf /、dd、mkfs、fork bomb）
- **工程决策侧栏**：workspace 级 `deny(unsafe_code)` 的实践意义

### Ch8. 上下文管理：让 Agent 在有限窗口中高效工作

- Context Compaction：80% 触发、保留最近 6 条、旧消息摘要
- 工具参数剥离 + 首行摘要策略
- Fidelity 四档模式（Full / Truncate / Compact / Summary）
- steering 与 prompt_layer：提示工程的系统化
- prompt_guard：系统提示的防篡改
- **工程决策侧栏**：为什么 80% 而非动态阈值

### Ch9. 扩展机制：Skills 与 Plugins 双轨制

- **Skills 轨道**：
  - SKILL.md markdown 前置问题格式
  - SkillsLoader：工作区覆盖内置、优雅降级
  - 结果注入为 XML 技能索引
  - `spawn_only` 工具：自动后台执行、SKILL.md 注入系统提示
- **Plugins 轨道**：
  - manifest.json 声明 + 二进制协议（argv + stdin JSON + stdout JSON）
  - Discovery：目录扫描 + 优先级规则
  - Gating：binary/env/OS 三重检查，失败跳过不致命
- **MCP 集成**：
  - Stdio / HTTP-SSE 双传输
  - 环境变量清理（BLOCKED_ENV_VARS）
  - Schema 验证：最大深度 10、64KB 上限
  - 工具响应 1MB 限制、30 秒超时
- **工程决策侧栏**：为什么需要三种扩展机制

---

## 第三部分：平台 — 从单机到多租户

### Ch10. octos-bus：14 频道的统一消息抽象

- `Channel` trait 与 ChannelManager 架构
- 频道集成：Telegram / Discord / Slack / WhatsApp / 飞书 / 邮件 / 企业微信 / Matrix / Twilio / QQ Bot
- 消息 Coalescing：5 级切割策略（段落 → 换行 → 句子 → 空格 → 硬切）
- 频道特定字符限制（Telegram 4000、Discord 1900）
- MAX_CHUNKS=50 防 DoS
- Session 管理：
  - JSONL 持久化 + LRU 内存缓存
  - `/new` fork 机制 + parent_key 追踪
  - Percent-encoded 文件名 + hash 后缀防碰撞
  - 10MB 文件限制 + 原子 write-then-rename 防崩溃
- **工程决策侧栏**：为什么 JSONL 而非 SQLite

### Ch11. 并发模型：Tokio 异步架构实战

- per-message `tokio::spawn()` + per-session Mutex 序列化
- 信号量限流：`max_concurrent_sessions` 默认 10
- `join_all` 工具并发：单次迭代内多工具并行执行
- 子 Agent 双模式：同步阻塞 vs 后台异步（spawn 工具）
- `AtomicBool` 优雅关停：Release on store, Acquire on load
- Heartbeat 与 Cron：三种调度类型（Every / Cron / At）
- **工程决策侧栏**：per-session Mutex 而非 actor model 的取舍

### Ch12. octos-pipeline：DOT 图驱动的工作流引擎

- Graphviz DOT 语法解析为 PipelineGraph
- 5 种 Handler 类型：Codergen / Gate / Shell / Noop / DynamicParallel
- 并行 fan-out：N 个 concurrent worker 运行时 spawn
- 条件边分支逻辑（condition evaluation）
- Human Gate：5 分钟超时的人在环路机制
- Checkpoint Store：断点续跑
- ModelStylesheet：per-node 模型选择
- Artifact Store：产出物管理
- PipelineResult：output + token usage + per-node summaries + modified files
- **工程决策侧栏**：为什么选 DOT 而非 YAML/JSON 定义工作流

### Ch13. 三种运行模式与配置体系

- **CLI 模式** (`octos chat`)：readline 交互循环
- **Gateway 模式** (`octos gateway`)：多频道消息总线
- **Serve 模式** (`octos serve`)：Web Dashboard + 91 REST 端点 + SSE 流式
- 配置优先级：本地 `.octos/config.json` > 全局 `~/.config/octos/config.json`
- Provider 自动检测：模型名称模式匹配
- 热加载：SHA-256 hash 变更检测
  - 热加载项：system prompt
  - 需重启项：provider / model / hooks
- Feature Flags：email / browser / api 条件编译
- Config Watcher：`config_watcher.rs` 的实现
- **工程决策侧栏**：热加载 vs 全重启的边界划分

### Ch14. 生产化：认证、监控与部署

- **认证三流**：
  - OAuth PKCE + SHA-256 challenge（OpenAI）
  - Device code flow
  - Paste-token（其他 Provider）
  - 凭据存储：`~/.octos/auth.json`，mode 0600
  - 常量时间 bearer token 比较
- **API 安全**：默认绑定 127.0.0.1（`--host` 覆盖）
- **Hooks 生命周期**：
  - 4 事件：before/after × tool_call/llm_call
  - Shell 协议：stdin JSON + exit code（0=allow, 1=deny, 2+=error）
  - 熔断器：3 次连续失败自动禁用
  - argv 数组执行（无 shell 解释）+ tilde 展开
- **监控**：Prometheus 指标端点
- **多租户**：tenant 配置与隔离
- **工程决策侧栏**：为什么 hooks 用 exit code 而非 JSON 响应

---

## 附录

### A. octos 完整 Crate 依赖图

### B. 工具速查表：14 个内置工具 + 分组策略

### C. 配置参考：config.json 完整字段说明

### D. Feature Flags 一览

### E. 从源码构建与贡献指南

---

## 写作约定

- 每章开头用 1-2 段话说明本章解决什么问题（对应哪类读者）
- 关键设计决策用「工程决策侧栏」高亮，解释 why not alternatives
- 代码引用标注文件路径和行号，方便读者对照源码
- 架构图 / 数据流图 / 状态机图用 Mermaid 格式
- 中文为主，类型名、trait 名、crate 名、CLI 命令保留英文
