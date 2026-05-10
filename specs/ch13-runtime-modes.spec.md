spec: task
name: "Ch13. 四种运行模式与配置体系"
inherits: project
tags: [part3, cli, gateway, serve, mcp-serve, config, feature-flags]
depends: [ch05-agent-loop, ch10-message-bus]
estimate: 1d
---

## 意图

octos 提供 CLI/Gateway/Serve/MCP Serve 四种运行模式，配合配置优先级、热加载、
Feature Flags 形成完整的运行时体系。本章帮助用户和贡献者理解如何选择
运行模式以及配置变更如何传播。当前主分支中 `octos serve` 已经成为 AppState
和控制面的汇聚点，`octos mcp-serve` 则把完整 octos session 暴露为一个
coarse-grained MCP tool，本章需要避免把它们写成“chat 外壳”或“内部工具直出”。

## 决策

- 源码文件: `crates/octos-cli/src/main.rs`, `config.rs`, `config_watcher.rs`
- 命令入口: `chat.rs`, `gateway.rs`, `serve.rs`(API routes), `mcp_serve.rs`, `../octos/crates/octos-agent/src/mcp_server.rs`
- 图表: 四种模式架构对比图、配置优先级链路、热加载流程、Serve control-plane composition、MCP serve session-level dispatch
- 工程决策侧栏: 热加载 vs 全重启的边界划分
- MCP Serve 边界: 只暴露 `run_octos_session`，不暴露 octos 内部 tool catalog

## 边界

### 允许修改
- octos-book/chapters/ch13-*.md
- octos-book/book/src/part3/ch13.md
- octos-book/book-en/src/part3/ch13.md
- octos-book/assets/ch13-*

### 禁止做
- 不逐个讲解 91 个 REST 端点（附录 C 覆盖）
- 不重复 Agent Loop 或 Bus 细节

## 排除范围

- REST API 完整参考文档（附录覆盖）
- 前端 Web Dashboard 实现细节

## 完成条件

场景: 四种运行模式对比清晰
  测试: review_ch13_four_modes
  当 阅读运行模式小节
  那么 包含 CLI/Gateway/Serve/MCP Serve 四种模式的架构对比表
  并且 每种模式说明了入口函数、适用场景、支持的功能边界
  并且 Gateway Profile 模式说明子账号继承的是结构化 profile sections 和 env_vars base，而不是旧顶层 provider/model 字段
  并且 Serve 默认端口写为 50080
  并且 MCP Serve 说明默认 stdio、HTTP bearer token 和外层 orchestrator 调用场景

场景: 配置优先级完整
  测试: review_ch13_config_priority
  当 阅读配置体系小节
  那么 解释了本地 `.octos/config.json` > `<data_dir>/config.json` > legacy platform config dir 的优先级
  并且 说明了 Provider 自动检测的模型名匹配规则

场景: 热加载机制
  测试: review_ch13_hot_reload
  当 阅读热加载小节
  那么 解释了 SHA-256 hash 变更检测的实现
  并且 区分了热加载项（system prompt/max_history）、provider/model 的显式运行时切换路径，以及需重启项（base_url/api_key_env/hooks）
  并且 说明了 config_watcher 的工作流程

场景: Feature Flags
  测试: review_ch13_feature_flags
  当 阅读 Feature Flags 小节
  那么 列出了当前 `octos-cli` 主要 feature flag，并说明完整列表见附录 D
  并且 解释了条件编译在 Cargo.toml 中的配置方式

场景: 热加载边界侧栏
  测试: review_ch13_reload_sidebar
  当 阅读工程决策侧栏
  那么 解释了为什么 system prompt 可以热加载，而 provider/model 的文件变更不会被 watcher 自动应用
  并且 说明了这个边界划分的安全和一致性考量

场景: Serve 是控制面汇聚点
  测试: review_ch13_serve_control_plane
  当 阅读 Serve 模式小节
  那么 说明 `octos serve` 同时汇聚 Config/ProfileStore、LlmProvider/RetryProvider、ToolRegistry/ToolPolicy、SessionManager、REST/UI Protocol/event harness 和 SwarmState
  并且 说明 dashboard auth 可从 profile email config 推导
  并且 说明 swarm dispatch policy 从 `config.tool_policy` 和注入型环境变量 denylist 构建
  并且 包含 Serve control-plane composition Mermaid 图

场景: MCP Serve 是 session-level dispatch
  测试: review_ch13_mcp_serve_session_level
  当 阅读 MCP Serve 小节
  那么 说明 MCP server 只暴露 `run_octos_session`
  并且 说明每次调用运行完整 octos session 并返回 aggregate result
  并且 说明外层 caller 看不到内部 tool calls、iteration events 或 progress stream
  并且 说明 stdio 是 parent-trust auth，HTTP transport 需要 `OCTOS_MCP_SERVER_TOKEN`
  并且 包含 MCP serve session-level dispatch Mermaid 图
