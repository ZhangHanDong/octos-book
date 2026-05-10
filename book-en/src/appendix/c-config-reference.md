# Appendix C: Configuration Reference

## Configuration File Locations

| Priority | Path | Description |
|----------|------|-------------|
| 1 (highest) | `.octos/config.json` | Project-local configuration |
| 2 | `~/.config/octos/config.json` | User-global configuration |
| 3 (lowest) | Built-in defaults | Defaults defined in code |

## Core Configuration Fields

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `provider` | string | auto-detected | LLM provider name |
| `model` | string | — | Model ID |
| `base_url` | string? | provider default | API endpoint override |
| `api_key_env` | string? | provider default | Environment variable containing the API key |
| `system_prompt` | string? | built-in default | Custom system prompt |
| `max_iterations` | u32 | 50 | Maximum Agent iterations |
| `max_tokens` | u32? | unlimited | Token budget |
| `max_timeout_secs` | u64 | 600 | Wall-clock timeout in seconds |
| `tool_timeout_secs` | u64 | 600 | Timeout for a single tool call |

## Tool Policy

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `tools.allow` | string[] | [] | Allowed tool list; empty means all tools are allowed unless denied |
| `tools.deny` | string[] | [] | Denied tool list; deny wins |
| `tool_policy_by_provider.<name>.allow` | string[] | — | Provider/model-level allow list; exact model ID wins before provider name |
| `tool_policy_by_provider.<name>.deny` | string[] | — | Provider/model-level deny list |
| `tool_policy_by_provider.<name>.require_tags` | string[] | [] | Provider/model-level tag filter |

## Gateway Configuration

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `gateway.channels` | object | {} | Channel configuration |
| `gateway.max_concurrent_sessions` | u32 | 10 | Maximum concurrent active sessions |

## Serve Configuration

| Field Path | Type | Default | Description |
|------------|------|---------|-------------|
| `serve.host` | string | `"127.0.0.1"` | Bind address |
| `serve.port` | u16 | 50080 | Bind port |

## Profile LLM Configuration

Profile files live under `~/.octos/profiles/<id>.json`. On the current main branch, profile LLM settings are not just top-level `provider` / `model` fields; they live under `config.llm`.

| Field Path | Type | Description |
|------------|------|-------------|
| `config.llm.primary.family_id` | string | Primary model family ID, such as `anthropic` or `openai` |
| `config.llm.primary.model_id` | string | Primary model ID |
| `config.llm.primary.route.base_url` | string? | API endpoint override for this model route |
| `config.llm.primary.route.api_key_env` | string? | API key environment variable for this model route |
| `config.llm.primary.route.api_type` | string? | API type hint, such as OpenAI-compatible or Anthropic |
| `config.llm.primary.model_hints` | object | Model capability, cost, and context-window hints |
| `config.llm.primary.cost_per_m` | object? | Cost per million tokens |
| `config.llm.primary.strong` | bool? | Whether the model is marked as a strong model |
| `config.llm.fallbacks[]` | object[] | Fallback model list; elements have the same shape as `primary` |

## MCP Servers

| Field Path | Type | Description |
|------------|------|-------------|
| `mcp_servers.<name>.command` | string | Stdio mode command |
| `mcp_servers.<name>.args` | string[] | Command arguments |
| `mcp_servers.<name>.env` | object | Environment variables passed to the child process |
| `mcp_servers.<name>.url` | string | HTTP mode server URL |

## Hooks

| Field Path | Type | Description |
|------------|------|-------------|
| `hooks[]` | object[] | Hook list |
| `hooks[].event` | string | Event name, such as `before_tool_call` or `after_tool_call` |
| `hooks[].command` | string[] | argv-form command; not interpreted by a shell |
| `hooks[].timeout_ms` | u64 | Timeout in milliseconds |
| `hooks[].tool_filter` | string[] | Only match these tools; empty array means no tool filter |

Each hook object looks like:

```json
{ "event": "before_tool_call", "command": ["cmd", "arg1"], "timeout_ms": 5000, "tool_filter": ["shell"] }
```

## Example Configuration

```json
{
  "model": "claude-sonnet-4-20250514",
  "system_prompt": "You are a helpful coding assistant.",
  "max_iterations": 30,
  "tools": {
    "deny": ["browser"]
  },
  "tool_policy_by_provider": {
    "ollama": {
      "allow": ["read_file", "shell", "grep"]
    }
  },
  "gateway": {
    "max_concurrent_sessions": 20,
    "channels": {
      "telegram": { "token_env": "TELEGRAM_BOT_TOKEN" }
    }
  },
  "mcp_servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    }
  },
  "hooks": [
    {
      "event": "before_tool_call",
      "command": ["./audit-hook.sh"],
      "timeout_ms": 3000,
      "tool_filter": ["shell"]
    }
  ]
}
```
