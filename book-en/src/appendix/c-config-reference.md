# Appendix C: Configuration Reference

This appendix reflects the current `../octos` main branch, especially `octos-cli/src/config.rs`, `octos-cli/src/profiles.rs`, and the `octos-agent` hook schema. Top-level `config.json` is for single-instance runtime configuration; profile files are for `serve/gateway` multi-profile and multi-tenant deployments.

## Configuration File Locations

| Priority | Path | Description |
|----------|------|-------------|
| 1 (highest) | `<cwd>/.octos/config.json` | Project-local configuration |
| 2 | `<data_dir>/config.json`, usually `~/.octos/config.json` | Current data-dir configuration |
| 3 | Legacy platform config dir, such as `~/.config/octos/config.json` | Backward-compatible legacy path; loading logs a migration hint |
| 4 (lowest) | Built-in defaults | Defaults defined in code |

## Top-Level `Config` Fields

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `version` | u32? | `null` | Config migration version; current code constant is `1` |
| `provider` | string? | auto-detected / mode-dependent | LLM provider name |
| `model` | string? | provider default or detected | Model ID |
| `base_url` | string? | provider default | API endpoint override |
| `api_key_env` | string? | provider default | API key environment variable |
| `model_hints` | object? | `null` | OpenAI-compatible custom model behavior hints |
| `api_type` | string? | provider default | Protocol override, such as `openai` / `anthropic` |
| `auth_token` | string? | CLI/env override or generated | Dashboard/admin token |
| `gateway` | object? | `null` | Gateway mode configuration |
| `mcp_servers` | array | `[]` | MCP server configuration |
| `sandbox` | object | `SandboxConfig::default()` | Tool sandbox configuration |
| `tool_policy` | object? | `null` | Global tool allow/deny/tag policy |
| `tool_policy_by_provider` | object | `{}` | Model/provider-specific tool policy; exact model ID wins before provider prefix |
| `embedding` | object? | `null` | Hybrid memory embedding provider |
| `fallback_models` | array | `[]` | Fallback models for provider failover |
| `max_iterations` | u32? | Agent default, CLI-overridable | Max Agent iterations per user message |
| `hooks` | array | `[]` | Agent lifecycle hooks |
| `context_filter` | string[] | `[]` | Expose only tools matching at least one tag |
| `sub_providers` | array | `[]` | Provider choices available to child Agents via `spawn` |
| `adaptive_routing` | object? | `null` | AdaptiveRouter configuration |
| `email` | object? | `null` | `send_email` / email sending configuration |
| `voice` | object? | `null` | ASR/TTS auto-transcription and voice replies |
| `mode` | enum | `"local"` | `local` / `tenant` / `cloud` |
| `tunnel_domain` | string? | env-overridable | Cloud/tenant tunnel domain |
| `base_domain` | string? | compatible default `crew.ominix.io` | Public profile base domain; `OCTOS_BASE_DOMAIN` wins |
| `frps_server` | string? | env-overridable | frps server address |
| `allow_admin_shell` | bool | `false` | Enables `/api/admin/shell`; keep disabled in production |
| `dashboard_auth` | object? | `null` | Email OTP login config under the `api` feature |
| `monitor` | object? | `null` | Watchdog/alert config under the `api` feature |
| `credential_pool` | object? | `null` | Top-level credential pool |
| `content_routing` | object? | `null` | Content classifier / Cheap-Strong routing |
| `appui.default_session_cwd` | path? | `null` | Default working directory for AppUI sessions without cwd capability |

## Gateway Configuration

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `gateway.channels[]` | array | CLI channel | Channel config; each entry has `type`, `allowed_senders`, and `settings` |
| `gateway.max_history` | usize | `50` | Max history messages injected into the LLM |
| `gateway.system_prompt` | string? | `null` | Gateway system prompt |
| `gateway.queue_mode` | enum | `"collect"` | `followup` / `collect` / `steer` / `interrupt` / `speculative` |
| `gateway.max_sessions` | usize | `1000` | Max sessions retained in memory |
| `gateway.max_concurrent_sessions` | usize | `10` | Max concurrent sessions |
| `gateway.browser_timeout_secs` | u64? | browser default | Per-browser-action timeout |
| `gateway.llm_timeout_secs` | u64? | LLM default | LLM HTTP request timeout |
| `gateway.llm_connect_timeout_secs` | u64? | LLM default | LLM HTTP connect timeout |
| `gateway.tool_timeout_secs` | u64? | runtime default | Total wait limit for parallel tool calls |
| `gateway.session_timeout_secs` | u64? | runtime default | Per-session-message processing timeout |
| `gateway.max_output_tokens` | u32? | model default | Max output tokens per LLM call |

## Serve Configuration

`serve` port is a CLI argument, not an embedded `Config` field: `octos serve --port <PORT>`, defaulting to `50080`. Historical `serve.host` / `serve.port` docs do not match the current `Config` struct.

## Provider / Embedding / Routing

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `fallback_models[].provider` | string | required | Fallback provider |
| `fallback_models[].model` | string? | provider default | Fallback model |
| `fallback_models[].base_url` | string? | provider default | Endpoint override |
| `fallback_models[].api_key_env` | string? | provider default | API key env |
| `fallback_models[].model_hints` | object? | `null` | OpenAI-compatible model hints |
| `fallback_models[].api_type` | string? | provider default | Protocol override |
| `fallback_models[].cost_per_m` | f64? | `null` | Output cost per million tokens |
| `fallback_models[].strong` | bool | `true` | Whether to treat this as a strong model |
| `embedding.provider` | string | `"openai"` | Embedding provider |
| `embedding.api_key_env` | string? | provider default | Embedding API key env |
| `embedding.base_url` | string? | provider default | Embedding endpoint |
| `adaptive_routing.enabled` | bool | `false` | Enable AdaptiveRouter |
| `adaptive_routing.mode` | enum | `"off"` | `off` / `hedge` / `lane` |
| `adaptive_routing.latency_threshold_ms` | u64 | `10000` | Soft latency threshold |
| `adaptive_routing.error_rate_threshold` | f64 | `0.3` | Error-rate threshold |
| `adaptive_routing.probe_probability` | f64 | `0.1` | Probe probability |
| `adaptive_routing.probe_interval_secs` | u64 | `60` | Probe interval per provider |
| `adaptive_routing.failure_threshold` | u32 | `3` | Circuit-breaker consecutive failure threshold |
| `adaptive_routing.qos_ranking` | bool | `false` | Enable quality/throughput ranking |
| `adaptive_routing.weight_latency/error_rate/priority/cost` | f64 | `0.3/0.3/0.2/0.2` | Scoring weights |
| `content_routing.enabled` | bool | `false` | When disabled, every turn is classified as Strong |

## Tool Policy / MCP / Hooks

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `tool_policy.allow` | string[] | `[]` | Allow list; supports tool names, `group:<id>`, and `prefix*` |
| `tool_policy.deny` | string[] | `[]` | Deny list; deny wins |
| `tool_policy.require_tags` | string[] | `[]` | Allow only tools with matching tags |
| `tool_policy_by_provider.<name>` | object | `{}` | Add policy for a model/provider |
| `mcp_servers[].command` | string? | - | Stdio MCP command |
| `mcp_servers[].args` | string[] | `[]` | Stdio MCP args |
| `mcp_servers[].env` | object | `{}` | MCP child process environment |
| `mcp_servers[].url` | string? | - | HTTP MCP URL |
| `hooks[]` | object[] | `[]` | Hook list |
| `hooks[].event` | string | required | Event name, such as `before_tool_call` / `after_tool_call` / `on_turn_end` |
| `hooks[].command` | string[] | required | argv-form command; not interpreted by a shell |
| `hooks[].timeout_ms` | u64 | hook default | Timeout in milliseconds |
| `hooks[].tool_filter` | string[] | `[]` | Empty array means no tool filter |

## Credential / Email / Voice / AppUI

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `credential_pool.state_path` | string? | `<data_dir>/credential_pool.redb` | Top-level pool state file |
| `credential_pool.name` | string | `"default"` | Pool metrics name |
| `credential_pool.strategy` | string | `"round_robin"` | `fill_first` / `round_robin` / `random` / `least_used` |
| `credential_pool.credential_ids` | string[] | `[]` | Credential IDs in the pool |
| `credential_pool.default_cooldown_ms` | u64? | `null` | Cooldown for 429 responses without reset hints |
| `email.provider` | string | required | `smtp` / `feishu` / `lark` |
| `email.smtp_host/smtp_port/username/password/password_env/from_address` | mixed | `null` | SMTP config |
| `email.feishu_app_id/feishu_app_secret/feishu_app_secret_env/feishu_from_address/feishu_region` | mixed | `null` / `cn` | Feishu/Lark mail config |
| `voice.auto_asr` | bool | `true` | Auto-transcribe voice messages |
| `voice.auto_tts` | bool | `true` | Auto-synthesize voice replies |
| `voice.default_voice` | string | `"vivian"` | Default TTS voice |
| `voice.asr_language` | string? | `null` | ASR language hint |
| `appui.default_session_cwd` | path? | `null` | AppUI default workspace cwd |

## Dashboard Auth / Monitor

These fields only apply to `octos serve` builds with the `api` feature enabled. Without `dashboard_auth.smtp`, OTP codes can still be generated, but delivery falls back to log output.

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `dashboard_auth.smtp.host` | string | required | OTP SMTP server host |
| `dashboard_auth.smtp.port` | u16 | `465` | SMTP port; `465` is implicit TLS and `587` is STARTTLS |
| `dashboard_auth.smtp.username` | string | required | SMTP username |
| `dashboard_auth.smtp.password_env` | string | required / compatible | SMTP password env; new installs prefer the data-dir secret store |
| `dashboard_auth.smtp.from_address` | string | required | OTP sender address |
| `dashboard_auth.session_expiry_hours` | u64 | `24` | Dashboard session lifetime |
| `dashboard_auth.allow_self_registration` | bool | `false` | Whether unknown emails auto-create users |
| `dashboard_auth.static_tokens` | string[] | `[]` | Static tokens for E2E tests; do not configure in production |
| `monitor.alerts_enabled` | bool | `true` | Enable proactive alerts |
| `monitor.watchdog_enabled` | bool | `true` | Enable watchdog auto-restart |
| `monitor.health_check_interval_secs` | u64 | `60` | Health-check interval |
| `monitor.max_restart_attempts` | u32 | `3` | Maximum auto-restart attempts |
| `monitor.telegram_token_env` | string? | `null` | Telegram bot token env |
| `monitor.telegram_alert_chat_ids` | i64[] | `[]` | Telegram alert chat IDs |
| `monitor.feishu_app_id_env` | string? | `null` | Feishu app id env |
| `monitor.feishu_app_secret_env` | string? | `null` | Feishu app secret env |
| `monitor.feishu_alert_user_ids` | string[] | `[]` | Feishu alert user IDs |

## Profile Configuration

Profile files live under `~/.octos/profiles/<id>.json`. `UserProfile` top-level fields include `id`, `name`, `public_subdomain`, `enabled`, `data_dir`, `parent_id`, `config`, `created_at`, and `updated_at`. Sub-accounts use `parent_id` to inherit the parent profile's LLM contract and low-level env vars.

On the current main branch, profile LLM settings are not old top-level `provider` / `model` fields; they live under `config.llm`:

| Field Path | Type | Description |
|------------|------|-------------|
| `config.llm.primary.family_id` | string? | Primary model family ID, such as `anthropic`, `openai`, or `moonshot` |
| `config.llm.primary.model_id` | string? | Primary model ID |
| `config.llm.primary.route.route_id` | string? | Catalog route ID |
| `config.llm.primary.route.label` | string? | Route display label |
| `config.llm.primary.route.base_url` | string? | Route API endpoint override |
| `config.llm.primary.route.api_key_env` | string? | Route API key env |
| `config.llm.primary.route.api_type` | string? | Protocol override, such as `anthropic` / `responses` |
| `config.llm.primary.model_hints` | object? | Model capability, cost, and context-window hints |
| `config.llm.primary.cost_per_m` | f64? | Cost per million tokens |
| `config.llm.primary.strong` | bool? | Whether the model is marked strong |
| `config.llm.fallbacks[]` | object[] | Fallback model list; entries use the same shape as `primary` |

Profile typed sections also include `search.providers`, `deep_crawl.page_settle_ms/max_output_chars`, `apps.slides`, `robot.realtime`, `channels[]`, `gateway`, `email`, `env_vars`, `hooks[]`, `admin_mode`, `sandbox`, `adaptive_routing`, `cost_budget`, `matrix.swarm_supervisor`, `content_routing`, and `credential_pool`.

## Example Configuration

```jsonc
{
  // Current config version; missing values are handled by migration logic.
  "version": 1,
  // Primary LLM provider / model.
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  // API key env override.
  "api_key_env": "ANTHROPIC_API_KEY",
  // Max Agent iterations for one turn.
  "max_iterations": 30,
  // Global tool policy; deny wins.
  "tool_policy": {
    "deny": ["browser"]
  },
  // Provider-specific tool policy.
  "tool_policy_by_provider": {
    "ollama": {
      "allow": ["read_file", "shell", "grep"]
    }
  },
  // Gateway runtime config.
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
  // MCP server config.
  "mcp_servers": [
    {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env": {}
    }
  ],
  // Hooks use argv arrays and are not interpreted through a shell.
  "hooks": [
    {
      "event": "before_tool_call",
      "command": ["./audit-hook.sh"],
      "timeout_ms": 3000,
      "tool_filter": ["shell"]
    }
  ],
  // AdaptiveRouter config.
  "adaptive_routing": {
    "enabled": true,
    "mode": "lane",
    "qos_ranking": true
  },
  // Default cwd for AppUI sessions.
  "appui": {
    "default_session_cwd": "/workspace"
  }
}
```
