spec: task
name: "Ch14. 生产化：认证、监控与部署"
inherits: project
tags: [part3, auth, hooks, monitoring, production]
depends: [ch13-runtime-modes]
estimate: 1.5d
---

## 意图

从开发到生产的最后一公里。本章展示 octos 的认证三流（OAuth PKCE/device code/
paste-token）、Hooks 生命周期系统、Prometheus/SSE/tracing 监控集成、AppUI UI Protocol 控制面、以及多租户配置。
帮助运维和贡献者将 octos 部署到生产环境。当前主分支还引入了 UI Protocol
capability negotiation、`/api/events/harness` typed SSE、Setup/Admin store
和 harness event 控制面，本章需要把观测与控制的边界写清楚。

## 决策

- 源码文件: `crates/octos-cli/src/auth/`, `crates/octos-agent/src/hooks.rs`
- 辅助文件: `tenant.rs`, `monitor.rs`, API route 文件, `ui_protocol.rs`, `../octos/crates/octos-cli/src/api/events_harness.rs`, `../octos/crates/octos-cli/src/admin_token_store.rs`, `../octos/crates/octos-cli/src/smtp_secret_store.rs`, `../octos/crates/octos-cli/src/api/admin_setup.rs`
- 图表: OAuth PKCE 流程图、Hooks 事件生命周期图、熔断器状态机、UI Protocol capability negotiation、Harness events observability layers、生产控制面总图
- 工程决策侧栏: 为什么 hooks 用 exit code 而非 JSON 响应

## 边界

### 允许修改
- octos-book/chapters/ch14-*.md
- octos-book/assets/ch14-*

### 禁止做
- 不暴露具体的 token 格式或加密密钥
- 不提供云厂商特定的部署 step-by-step

## 排除范围

- 云厂商 IaC 脚本（Terraform/Pulumi）
- Kubernetes 部署 YAML

## 完成条件

场景: 认证三流完整
  测试: review_ch14_auth_flows
  当 阅读认证小节
  那么 分别展示了 OAuth PKCE（带 SHA-256 challenge）、device code、paste-token 的流程
  并且 说明了凭据存储位置（`~/.octos/auth.json`）和 mode 0600 权限
  并且 解释了常量时间 bearer token 比较的安全意义

场景: Hooks 生命周期完整
  测试: review_ch14_hooks
  当 阅读 Hooks 小节
  那么 列出了 4 个事件（before/after × tool_call/llm_call）
  并且 解释了 shell 协议（stdin JSON + exit code 语义）
  并且 说明了熔断器（3 次失败自动禁用）和可配置阈值
  并且 解释了 argv 数组执行（无 shell 解释）的安全意义

场景: 监控集成
  测试: review_ch14_monitoring
  当 阅读监控小节
  那么 说明了 Prometheus 指标端点的暴露方式
  并且 列出了关键指标（请求量、延迟、token 用量等）
  并且 解释了 SSE 实时事件流与 tracing 分别解决什么观测问题

场景: AppUI UI Protocol 控制面
  测试: review_ch14_ui_protocol
  当 阅读 UI Protocol 小节
  那么 说明了 `/api/ui-protocol/ws` 是 AppUI 的 WebSocket JSON-RPC 控制通道
  并且 说明了 `session/open` 返回 profile、workspace、cursor、panes 和 capabilities
  并且 说明了 capability negotiation 和 `harness.task_control.v1` 的方法级门控
  并且 说明了 task list/cancel/restart 这类控制能力不是普通 SSE 事件

场景: 多租户配置
  测试: review_ch14_multi_tenant
  当 阅读多租户小节
  那么 解释了 Profile / Account / User / Session 的分层隔离机制
  并且 说明了子账号通过 `parent_id` 继承父 profile 的 `config.llm`、search、deep_crawl、apps、email 和 env_vars base
  并且 说明了子账号不继承父账号安装的 customer skills
  并且 区分 standalone Gateway 的同进程 actor 隔离和 process-manager 的 per-profile gateway 子进程
  并且 说明了租户间的资源边界仍需要系统级限额补足

场景: exit code 侧栏
  测试: review_ch14_exit_code_sidebar
  当 阅读工程决策侧栏
  那么 对比了 exit code vs JSON 响应 vs gRPC 三种 hook 协议
  并且 解释了 exit code 在简洁性和跨语言兼容性上的优势

场景: UI Protocol capability negotiation 准确
  测试: review_ch14_ui_protocol_capability_negotiation
  当 阅读 UI Protocol capability 小节
  那么 说明 `SessionOpened.capabilities` 来自 client 请求 features 与 server known features 的交集
  并且 说明无 feature header 时返回 `first_server_slice` 以支持 discovery
  并且 说明发送 feature header 时未知 feature 会被丢弃
  并且 说明 `task/list`、`task/cancel`、`task/restart_from_node` 只在协商到 `harness.task_control.v1` 时出现在 `supported_methods`
  并且 包含 UI Protocol capability negotiation sequence diagram

场景: harness events 与 tracing/metrics 边界
  测试: review_ch14_harness_events_observability
  当 阅读可观测性小节
  那么 区分 tracing、Prometheus metrics 和 `/api/events/harness` 的职责
  并且 说明 `/api/events/harness` 是 typed event stream，不是通用 log stream
  并且 说明 `kinds` filter 支持 snake_case、CamelCase 和 kebab-case 归一化
  并且 说明 top-level `kind` 和 nested `payload.kind` 两类 frame 都被兼容

场景: 生产控制面总图
  测试: review_ch14_production_control_plane
  当 阅读生产控制面小节
  那么 说明 admin token store、setup state、SMTP secret store、profile config、gateway/serve/process manager 的关系
  并且 说明 `admin_token.json` 是 hash store
  并且 说明 `smtp_secret.json` 优先于 SMTP env var
  并且 包含生产控制面 Mermaid 图
