# 附录 C：配置参考

本附录以当前 `../octos` main 分支的 `octos-cli/src/config.rs`、`octos-cli/src/profiles.rs` 和 `octos-agent` hook/schema 为准。顶层 `config.json` 面向单实例运行；profile 文件面向 `serve/gateway` 的多 profile / 多租户运行。

## 配置文件位置

| 优先级 | 路径 | 说明 |
|--------|------|------|
| 1（最高） | `<cwd>/.octos/config.json` | 项目本地配置 |
| 2 | `<data_dir>/config.json`，通常是 `~/.octos/config.json` | 当前数据目录配置 |
| 3 | 平台 legacy config dir，例如 `~/.config/octos/config.json` | 兼容旧路径；加载时会提示迁移 |
| 4（最低） | 内置默认值 | 代码中定义的默认值 |

## 顶层 `Config` 字段

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `version` | u32? | `null` | 配置迁移版本；当前代码常量为 `1` |
| `provider` | string? | 自动检测/必填取决于运行模式 | LLM Provider 名称 |
| `model` | string? | Provider 默认或模型检测 | 模型 ID |
| `base_url` | string? | Provider 默认 | API endpoint 覆盖 |
| `api_key_env` | string? | Provider 默认 | API key 环境变量名 |
| `model_hints` | object? | `null` | OpenAI-compatible 自定义模型行为提示 |
| `api_type` | string? | Provider 默认 | 协议覆盖，例如 `openai` / `anthropic` |
| `auth_token` | string? | CLI/env 覆盖或生成 | Dashboard/admin token |
| `gateway` | object? | `null` | Gateway 模式配置，见下节 |
| `mcp_servers` | array | `[]` | MCP server 配置 |
| `sandbox` | object | `SandboxConfig::default()` | 工具沙箱配置 |
| `tool_policy` | object? | `null` | 全局工具 allow/deny/tag 策略 |
| `tool_policy_by_provider` | object | `{}` | model/provider 级工具策略，精确 model ID 优先，其次 provider 前缀 |
| `embedding` | object? | `null` | Hybrid memory embedding provider |
| `fallback_models` | array | `[]` | Provider failover chain 的 fallback 模型 |
| `max_iterations` | u32? | Agent 默认，CLI 可覆盖 | 单条消息最大 Agent 迭代数 |
| `hooks` | array | `[]` | Agent 生命周期 hooks |
| `context_filter` | string[] | `[]` | 只向 LLM 暴露匹配 tag 的工具 |
| `sub_providers` | array | `[]` | `spawn` 子 Agent 可选 provider |
| `adaptive_routing` | object? | `null` | AdaptiveRouter 配置 |
| `email` | object? | `null` | `send_email` / 邮件发送配置 |
| `voice` | object? | `null` | ASR/TTS 自动转写与语音回复 |
| `mode` | enum | `"local"` | `local` / `tenant` / `cloud` |
| `tunnel_domain` | string? | env 可覆盖 | 云/租户 tunnel domain |
| `base_domain` | string? | 默认兼容 `crew.ominix.io` | 公开 profile 域名基座，`OCTOS_BASE_DOMAIN` 优先 |
| `frps_server` | string? | env 可覆盖 | frps server 地址 |
| `allow_admin_shell` | bool | `false` | 是否启用 `/api/admin/shell`；生产环境应保持关闭 |
| `dashboard_auth` | object? | `null` | `api` feature 下的 email OTP 登录配置 |
| `monitor` | object? | `null` | `api` feature 下的 watchdog/alert 配置 |
| `credential_pool` | object? | `null` | 顶层 credential pool |
| `content_routing` | object? | `null` | Content classifier / Cheap-Strong routing |
| `appui.default_session_cwd` | path? | `null` | AppUI session 未声明 cwd capability 时的默认工作目录 |

## Gateway 配置

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `gateway.channels[]` | array | CLI channel | 频道配置；每项含 `type`、`allowed_senders`、`settings` |
| `gateway.max_history` | usize | `50` | 注入 LLM 的最大历史消息数 |
| `gateway.system_prompt` | string? | `null` | Gateway 模式系统提示 |
| `gateway.queue_mode` | enum | `"collect"` | `followup` / `collect` / `steer` / `interrupt` / `speculative` |
| `gateway.max_sessions` | usize | `1000` | 内存中保留的最大 session 数 |
| `gateway.max_concurrent_sessions` | usize | `10` | 最大并发 session |
| `gateway.browser_timeout_secs` | u64? | Browser 默认 | 单个 browser action 超时 |
| `gateway.llm_timeout_secs` | u64? | LLM 默认 | LLM HTTP 总超时 |
| `gateway.llm_connect_timeout_secs` | u64? | LLM 默认 | LLM HTTP 连接超时 |
| `gateway.tool_timeout_secs` | u64? | Runtime 默认 | 并行工具调用总等待上限 |
| `gateway.session_timeout_secs` | u64? | Runtime 默认 | 单条 session 消息处理上限 |
| `gateway.max_output_tokens` | u32? | 模型默认 | LLM 单次输出 token 上限 |

## Serve 配置

`serve` 的端口是 CLI 参数而非 `Config` 内嵌字段：`octos serve --port <PORT>`，默认端口为 `50080`。历史文档中的 `serve.host` / `serve.port` 不是当前 `Config` 结构体字段。

## Provider / Embedding / Routing

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `fallback_models[].provider` | string | 必填 | fallback provider |
| `fallback_models[].model` | string? | Provider 默认 | fallback model |
| `fallback_models[].base_url` | string? | Provider 默认 | endpoint 覆盖 |
| `fallback_models[].api_key_env` | string? | Provider 默认 | API key env |
| `fallback_models[].model_hints` | object? | `null` | OpenAI-compatible 模型提示 |
| `fallback_models[].api_type` | string? | Provider 默认 | 协议覆盖 |
| `fallback_models[].cost_per_m` | f64? | `null` | 每百万 token 输出成本 |
| `fallback_models[].strong` | bool | `true` | 是否视为强模型 |
| `embedding.provider` | string | `"openai"` | embedding provider |
| `embedding.api_key_env` | string? | Provider 默认 | embedding API key env |
| `embedding.base_url` | string? | Provider 默认 | embedding endpoint |
| `adaptive_routing.enabled` | bool | `false` | 是否启用 AdaptiveRouter |
| `adaptive_routing.mode` | enum | `"off"` | `off` / `hedge` / `lane` |
| `adaptive_routing.latency_threshold_ms` | u64 | `10000` | 延迟软阈值 |
| `adaptive_routing.error_rate_threshold` | f64 | `0.3` | 错误率阈值 |
| `adaptive_routing.probe_probability` | f64 | `0.1` | 探针概率 |
| `adaptive_routing.probe_interval_secs` | u64 | `60` | 同 provider 探针间隔 |
| `adaptive_routing.failure_threshold` | u32 | `3` | circuit breaker 连续失败阈值 |
| `adaptive_routing.qos_ranking` | bool | `false` | 是否启用质量/吞吐排序 |
| `adaptive_routing.weight_latency/error_rate/priority/cost` | f64 | `0.3/0.3/0.2/0.2` | 评分权重 |
| `content_routing.enabled` | bool | `false` | 关闭时所有 turn 视为 Strong |

## Tool Policy / MCP / Hooks

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `tool_policy.allow` | string[] | `[]` | allow list；支持工具名、`group:<id>`、`prefix*` |
| `tool_policy.deny` | string[] | `[]` | deny list；deny-wins |
| `tool_policy.require_tags` | string[] | `[]` | 只允许含指定 tag 的工具 |
| `tool_policy_by_provider.<name>` | object | `{}` | 对指定 model/provider 叠加工具策略 |
| `mcp_servers[].command` | string? | - | Stdio MCP 启动命令 |
| `mcp_servers[].args` | string[] | `[]` | Stdio MCP 参数 |
| `mcp_servers[].env` | object | `{}` | MCP 子进程环境变量 |
| `mcp_servers[].url` | string? | - | HTTP MCP URL |
| `hooks[]` | object[] | `[]` | Hook 列表 |
| `hooks[].event` | string | 必填 | 事件名，如 `before_tool_call` / `after_tool_call` / `on_turn_end` |
| `hooks[].command` | string[] | 必填 | argv 形式命令，不经过 shell |
| `hooks[].timeout_ms` | u64 | hook 默认 | 超时毫秒 |
| `hooks[].tool_filter` | string[] | `[]` | 空数组表示不按工具过滤 |

## Credential / Email / Voice / AppUI

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `credential_pool.state_path` | string? | `<data_dir>/credential_pool.redb` | 顶层 pool 状态文件 |
| `credential_pool.name` | string | `"default"` | pool metrics 名称 |
| `credential_pool.strategy` | string | `"round_robin"` | `fill_first` / `round_robin` / `random` / `least_used` |
| `credential_pool.credential_ids` | string[] | `[]` | pool 内 credential IDs |
| `credential_pool.default_cooldown_ms` | u64? | `null` | 429 无 reset hint 时的 cooldown |
| `email.provider` | string | 必填 | `smtp` / `feishu` / `lark` |
| `email.smtp_host/smtp_port/username/password/password_env/from_address` | mixed | `null` | SMTP 配置 |
| `email.feishu_app_id/feishu_app_secret/feishu_app_secret_env/feishu_from_address/feishu_region` | mixed | `null` / `cn` | Feishu/Lark 邮件配置 |
| `voice.auto_asr` | bool | `true` | 是否自动转写语音 |
| `voice.auto_tts` | bool | `true` | 是否自动合成语音回复 |
| `voice.default_voice` | string | `"vivian"` | 默认 TTS 音色 |
| `voice.asr_language` | string? | `null` | ASR 语言提示 |
| `appui.default_session_cwd` | path? | `null` | AppUI 默认 workspace cwd |

## Dashboard Auth / Monitor

这些字段只在启用 `api` feature 的 `octos serve` 构建中生效。未配置 `dashboard_auth.smtp` 时，OTP 仍可生成，但邮件发送会退化为日志提示。

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `dashboard_auth.smtp.host` | string | 必填 | OTP SMTP server host |
| `dashboard_auth.smtp.port` | u16 | `465` | SMTP port；`465` 为 implicit TLS，`587` 为 STARTTLS |
| `dashboard_auth.smtp.username` | string | 必填 | SMTP username |
| `dashboard_auth.smtp.password_env` | string | 必填/兼容 | SMTP password env；新安装优先使用 data-dir secret store |
| `dashboard_auth.smtp.from_address` | string | 必填 | OTP 发件地址 |
| `dashboard_auth.session_expiry_hours` | u64 | `24` | Dashboard session 有效期 |
| `dashboard_auth.allow_self_registration` | bool | `false` | 未知 email 是否自动创建用户 |
| `dashboard_auth.static_tokens` | string[] | `[]` | E2E 测试用静态 token；生产环境不要配置 |
| `monitor.alerts_enabled` | bool | `true` | 是否启用主动告警 |
| `monitor.watchdog_enabled` | bool | `true` | 是否启用 watchdog 自动重启 |
| `monitor.health_check_interval_secs` | u64 | `60` | 健康检查间隔 |
| `monitor.max_restart_attempts` | u32 | `3` | 自动重启最大尝试次数 |
| `monitor.telegram_token_env` | string? | `null` | Telegram bot token env |
| `monitor.telegram_alert_chat_ids` | i64[] | `[]` | Telegram 告警 chat IDs |
| `monitor.feishu_app_id_env` | string? | `null` | Feishu app id env |
| `monitor.feishu_app_secret_env` | string? | `null` | Feishu app secret env |
| `monitor.feishu_alert_user_ids` | string[] | `[]` | Feishu 告警 user IDs |

## Profile 配置

Profile 文件位于 `~/.octos/profiles/<id>.json`。`UserProfile` 顶层字段包括 `id`、`name`、`public_subdomain`、`enabled`、`data_dir`、`parent_id`、`config`、`created_at`、`updated_at`。子账号通过 `parent_id` 继承父 profile 的 LLM contract 和低层 env vars。

当前主分支的 profile LLM 配置不再是旧的顶层 `provider` / `model`，而是放在 `config.llm` 下：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `config.llm.primary.family_id` | string? | 主模型家族 ID，例如 `anthropic`、`openai`、`moonshot` |
| `config.llm.primary.model_id` | string? | 主模型 ID |
| `config.llm.primary.route.route_id` | string? | catalog route ID |
| `config.llm.primary.route.label` | string? | route 展示名 |
| `config.llm.primary.route.base_url` | string? | route API endpoint 覆盖 |
| `config.llm.primary.route.api_key_env` | string? | route API key env |
| `config.llm.primary.route.api_type` | string? | 协议覆盖，例如 `anthropic` / `responses` |
| `config.llm.primary.model_hints` | object? | 模型能力、成本、上下文窗口等提示 |
| `config.llm.primary.cost_per_m` | f64? | 每百万 token 成本 |
| `config.llm.primary.strong` | bool? | 是否标记为强模型 |
| `config.llm.fallbacks[]` | object[] | fallback 模型列表，元素结构同 `primary` |

Profile 还包含 typed sections：`search.providers`、`deep_crawl.page_settle_ms/max_output_chars`、`apps.slides`、`robot.realtime`、`channels[]`、`gateway`、`email`、`env_vars`、`hooks[]`、`admin_mode`、`sandbox`、`adaptive_routing`、`cost_budget`、`matrix.swarm_supervisor`、`content_routing`、`credential_pool`。

## 示例配置

```jsonc
{
  // 当前配置版本；缺省也可由迁移逻辑处理
  "version": 1,
  // 主 LLM provider / model
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  // API key env 覆盖
  "api_key_env": "ANTHROPIC_API_KEY",
  // 单 turn 最大 Agent 迭代数
  "max_iterations": 30,
  // 全局工具策略，deny 优先
  "tool_policy": {
    "deny": ["browser"]
  },
  // 针对特定 provider 的工具策略
  "tool_policy_by_provider": {
    "ollama": {
      "allow": ["read_file", "shell", "grep"]
    }
  },
  // Gateway 运行时配置
  "gateway": {
    "max_concurrent_sessions": 20,
    "queue_mode": "collect",
    "channels": [
      {
        "type": "telegram",
        "allowed_senders": [],
        "settings": { "token_env": "TELEGRAM_BOT_TOKEN" }
      }
    ]
  },
  // MCP server 配置
  "mcp_servers": [
    {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env": {}
    }
  ],
  // Hook 使用 argv 数组，不经过 shell
  "hooks": [
    {
      "event": "before_tool_call",
      "command": ["./audit-hook.sh"],
      "timeout_ms": 3000,
      "tool_filter": ["shell"]
    }
  ],
  // AdaptiveRouter 配置
  "adaptive_routing": {
    "enabled": true,
    "mode": "lane",
    "qos_ranking": true
  },
  // AppUI session 默认 cwd
  "appui": {
    "default_session_cwd": "/workspace"
  }
}
```
