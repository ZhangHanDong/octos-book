spec: task
name: "Ch6. 工具系统：分层注册与工具治理"
inherits: project
tags: [part2, tools, policy, lru, registry]
depends: [ch05-agent-loop]
estimate: 1.5d
---

## 意图

工具系统是 Agent 与外部世界交互的唯一通道。本章要基于当前源码解释
`Tool` trait、ToolRegistry 的分层注册、ToolPolicy 的 deny-wins 语义、
工具曝光控制的混合策略，以及参数与文件访问安全，重点是让读者理解
"哪些工具在什么模式下出现、为什么这样治理"，而不是记忆一个过时的固定数量。

## 决策

- 以 `../octos/crates/octos-agent/src/tools/registry.rs` 中的 `with_builtins_and_sandbox()` 作为基础工具层锚点
- 明确区分基础注册、配置注入、chat 追加、gateway 追加、per-session 追加五层注册路径
- `Tool` trait 以当前真实签名为准：`name`、`description`、`input_schema`、`tags`、`execute`、`as_any`
- `ToolPolicy` 必须覆盖 `allow` / `deny` / `require_tags` 三维，并区分 `apply_policy` 与 `set_provider_policy`
- 工具曝光控制按当前实现解释为"预延迟 + LRU + execute 时自动激活"，`spawn_only` 单独讲，不混入 deferred 语义
- Provider 级配置字段使用 `tool_policy_by_provider`，匹配顺序为精确 model ID 优先，其次 provider 名
- 安全部分同时覆盖 `estimate_json_size`、`resolve_path`、`read_no_follow` / `write_no_follow`
- 图表与侧栏都服务于解释当前混合治理策略，不把系统简化成"纯 LRU"

## 边界

### 允许修改
- octos-book/chapters/ch06-*.md
- octos-book/book/src/part2/ch06.md
- octos-book/book-en/src/part2/ch06.md
- octos-book/specs/ch06-*.spec.md
- octos-book/assets/ch06-*

### 禁止做
- 不深入沙箱执行细节（Ch7 覆盖）
- 不逐个讲解每个工具的完整实现（只选代表性机制）
- 不把 `tools/mod.rs` 的导出列表当成默认注册表

## 排除范围

- MCP 外部工具集成（Ch9 覆盖）
- Plugin 二进制工具协议细节（Ch9 覆盖）
- 浏览器工具内部自动化细节

## 完成条件

场景: Tool trait 与扩展点
  测试: review_ch06_tool_trait
  当 阅读 Tool trait 小节
  那么 展示了当前 `Tool` trait 的真实方法集合
  并且 解释了 `ToolResult` 的结构
  并且 说明了 `execute_with_context()`、`tags()`、`as_any()` 与 `concurrency_class()` 的职责

场景: ToolRegistry 的分层注册
  测试: review_ch06_registry_layers
  当 阅读注册机制小节
  那么 解释了 `with_builtins_and_sandbox()` 的 15 个基础工具加 feature-gated 扩展
  并且 区分了配置注入、chat 追加、gateway 追加、per-session 追加
  并且 明确说明 `tools/mod.rs` 导出列表不等于默认注册表

场景: 混合曝光控制
  测试: review_ch06_activation_lru
  当 阅读 LRU / 激活小节
  那么 解释了 `defer_group()` 预延迟、15 个活跃槽位的 LRU、以及 `execute()` 中的自动激活
  并且 说明了 `activate_tools` 是可选元工具而不是唯一入口
  并且 解释了 LRU 状态是 per-session 的

场景: spawn_only 语义独立
  测试: review_ch06_spawn_only
  当 阅读 `spawn_only` 相关说明
  那么 明确说明它不属于 deferred / LRU 延迟池
  并且 解释了主会话里的自动后台化
  并且 说明当前优先向 LLM 返回 `task_handle` envelope 并通过 `read_task_output` 读取后台输出
  并且 说明了 subagent 中 `clear_spawn_only()` 后按普通工具执行

场景: ToolPolicy 三维过滤
  测试: review_ch06_policy
  当 阅读 ToolPolicy 小节
  那么 解释了 deny-wins 的评估逻辑
  并且 展示了通配符匹配（`web_*`）的实现方向
  并且 覆盖 `group:fs/runtime/web/search/sessions/memory/research/admin/media/delegated`
  并且 说明了 `require_tags` 与空标签工具的语义

场景: 并发调度语义
  测试: review_ch06_concurrency_class
  当 阅读工具执行小节
  那么 解释了 `ConcurrencyClass::Safe` 与 `ConcurrencyClass::Exclusive`
  并且 说明 safe batch 使用 `join_all` 并发执行，包含 exclusive 时走串行 admission

场景: Provider 级策略配置
  测试: review_ch06_provider_policy
  当 阅读 Provider 级策略小节
  那么 使用 `tool_policy_by_provider` 作为配置字段
  并且 说明了 model ID 优先、provider 名回退的匹配顺序
  并且 区分了 `apply_policy()` 与 `set_provider_policy()` 的行为差异

场景: 参数与文件安全
  测试: review_ch06_param_safety
  当 阅读参数安全小节
  那么 解释了 1MB 大小限制的 DoS 防护意义
  并且 说明了 `estimate_json_size` 的非分配实现技巧
  并且 区分了 `resolve_path` 的路径规范化与 `O_NOFOLLOW` 的 symlink-safe I/O

场景: 工程决策侧栏
  测试: review_ch06_strategy_sidebar
  当 阅读工程决策侧栏
  那么 对比了全量注册、纯手动按需加载、预延迟 + LRU + 自动激活三种策略
  并且 用上下文窗口成本和交互轮次解释当前混合治理策略的选择理由
  并且 不把当前实现简化成"纯 LRU"

场景: 纠正过时说法
  测试: review_ch06_drift_corrections
  当 阅读整章时
  那么 不再把工具系统写成固定的单一工具总数
  并且 不再使用 `tools.byProvider` 这种过时配置字段
  并且 不再把 `spawn_only` 解释成 deferred / LRU 延迟池的一部分
