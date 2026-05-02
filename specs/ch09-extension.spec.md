spec: task
name: "Ch9. 扩展机制：Skills、Plugins 与 MCP"
inherits: project
tags: [part2, skills, plugins, mcp, extension]
depends: [ch06-tool-system]
estimate: 1.5d
---

## 意图

octos 当前的扩展能力并不是“一个统一插件系统”，而是三条互补轨道：
Skills、Plugins 与 MCP。本章需要把这三者放回当前源码的真实边界里讲清楚，
尤其要避免继续沿用旧实现中的目录优先级、`spawn_only`、HTTP-SSE、以及
`octos-plugin` SDK 与 runtime loader 混淆的叙事。

## 决策

- 源码文件: `../octos/crates/octos-agent/src/skills.rs`、`../octos/crates/octos-agent/src/plugins/manifest.rs`、`../octos/crates/octos-agent/src/plugins/loader.rs`、`../octos/crates/octos-agent/src/plugins/tool.rs`、`../octos/crates/octos-agent/src/plugins/extras.rs`、`../octos/crates/octos-agent/src/mcp.rs`、`../octos/crates/octos-agent/src/agent/execution.rs`、`../octos/crates/octos-agent/src/tools/registry.rs`
- 图表: Plugin 二进制协议时序图、三种扩展机制对比侧栏

## 边界

### 允许修改
- octos-book/chapters/ch09-*.md
- octos-book/assets/ch09-*
- octos-book/specs/ch09-*.md

### 禁止做
- 不重复讲解 Tool trait 的基础定义
- 不把旧版 discovery/gating 叙事当成当前 runtime 热路径
- 不把 MCP HTTP 路径错误地写成“纯 SSE 通道”

## 排除范围

- 具体 app-skills 的业务实现
- MCP 协议规范的完整定义
- 插件市场、签名分发、安装器的完整设计

## 完成条件

场景: Skills 轨道按当前 loader 讲清楚
  测试: review_ch09_skills_loader
  当 阅读 Skills 小节
  那么 解释了 `SKILL.md` 的 frontmatter 是简化解析而不是完整 YAML
  并且 说明了 `requires_bins` / `requires_env` 如何决定 `available`
  并且 说明了当前 runtime 的 skills 分层优先级来自 profile、project、bundled-app-skills、环境变量目录与 builtins
  并且 说明了子账号不继承父账号安装的 customer skills，account skills/plugin dirs 只来自当前账号自己的 `data_dir/skills`
  并且 不把目录优先级错误地写成旧的“工作区 / 全局 / 内置”三层固定表

场景: XML 技能索引与工具标记准确
  测试: review_ch09_skills_xml
  当 阅读 Skills XML 索引小节
  那么 说明了 XML 使用 `<name>`、`<description>`、`<location>` 子节点
  并且 说明了 `tools="true"` 表示 skill 目录包含 `manifest.json`
  并且 不把 `name` 错写成 XML 属性

场景: spawn_only 语义回到当前实现
  测试: review_ch09_spawn_only
  当 阅读 `spawn_only` 小节
  那么 说明了 runtime 会把 `spawn_only` 工具自动后台化而不是同步执行
  并且 说明了这些工具仍然注册在工具系统里并对模型可见
  并且 说明了主 agent 立刻返回 `spawn_only_message`
  并且 说明了 skill package 含 `spawn_only` 工具时会自动注入 `SKILL.md` prompt fragment
  并且 说明了 subagent 场景下会清除 `spawn_only` 标记并直接执行

场景: Plugin runtime manifest 与执行协议准确
  测试: review_ch09_plugins_runtime
  当 阅读 Plugins 小节
  那么 说明了 runtime manifest 除了 `tools` 之外还支持 `mcp_servers`、`hooks`、`prompts.include`、`binaries`、`spawn_only`、`env`/`env_allowlist`、`risk`、`concurrency_class`
  并且 说明了 `tools` 为空但存在 extras 时会跳过可执行文件搜索
  并且 包含 Plugin 二进制协议时序图
  并且 解释了 argv 传 tool name、stdin 传 JSON 参数、stderr 作为进度流、stdout 优先解析结构化结果
  并且 提到 `file_modified` / `files_to_send` 或自动探测输出文件的语义

场景: Plugin 安全与运行时约束
  测试: review_ch09_plugins_security
  当 阅读 Plugin 安全小节
  那么 解释了 verified copy 如何关闭 TOCTOU 窗口
  并且 说明了 100MB 可执行文件上限
  并且 说明了 `BLOCKED_ENV_VARS` 和 `OCTOS_WORK_DIR` 注入
  并且 说明了 tool 级 `env`/`env_allowlist` 的严格语义与 legacy 兼容路径
  并且 说明了 high/critical risk 会强制请求 approval，缺少 approval bridge 时安全拒绝
  并且 说明了未知 `concurrency_class` fail closed 到 Exclusive
  并且 说明了默认超时是 600 秒而不是把 30 秒写成固定事实
  并且 说明了 Unix 上通过 `symlink_metadata()` 拒绝符号链接

场景: runtime loader 与 SDK 边界清晰
  测试: review_ch09_runtime_vs_sdk
  当 阅读 Plugin loader / discovery / gating 小节
  那么 明确区分了 `octos-agent/src/plugins/*` 是当前 runtime 热路径
  并且 说明了 `crates/octos-plugin` 是 discovery / gating / richer manifest 的 SDK 与 tooling crate
  并且 解释了 `discover_plugins()` 与 `check_requirements()` 的职责
  但是 不声称当前主 agent runtime 每次加载都先走 `octos-plugin::discover_plugins()`

场景: MCP transport 与安全约束准确
  测试: review_ch09_mcp
  当 阅读 MCP 小节
  那么 对比了 stdio 与 HTTP POST 两种传输
  并且 说明了 HTTP 路径接受普通 JSON 或 `text/event-stream` 响应
  并且 说明了 `mcp-session-id` / `Mcp-Session-Id` 的会话亲和
  并且 说明了 schema 最大深度 10 和最大大小 64KB
  并且 说明了 1MB 限制只适用于 stdio 单行响应，而不是所有 HTTP 响应的统一上限
  并且 说明了 `tools/call` 有 60 秒超时
  并且 说明了 HTTP 启动路径使用 SSRF 检查和 DNS pinning

场景: MCP 名称保护与三机制对比
  测试: review_ch09_mcp_protected_names_and_sidebar
  当 阅读 MCP 收尾与工程决策侧栏
  那么 说明了 `PROTECTED_NAMES` 会阻止 MCP tool 覆盖内置工具
  并且 说明了三种扩展机制各自解决的是不同层次的问题
  并且 解释了为什么不能统一成一种扩展机制

场景: 图表与源码锚点完整
  测试: review_ch09_diagram_and_sources
  当 阅读本章总览与图表
  那么 包含一张 Plugin 二进制协议时序图
  并且 包含一个 Skills / Plugins / MCP 的对比侧栏
  并且 主要论述锚定到 `../octos/crates/octos-agent/src/skills.rs`、`../octos/crates/octos-agent/src/plugins/manifest.rs`、`../octos/crates/octos-agent/src/plugins/loader.rs`、`../octos/crates/octos-agent/src/plugins/tool.rs`、`../octos/crates/octos-agent/src/plugins/extras.rs`、`../octos/crates/octos-agent/src/mcp.rs`
