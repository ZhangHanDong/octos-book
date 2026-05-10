# 附录 C：配置参考

## 配置文件位置

| 优先级 | 路径 | 说明 |
|--------|------|------|
| 1（最高） | `.octos/config.json` | 项目本地配置 |
| 2 | `~/.config/octos/config.json` | 用户全局配置 |
| 3（最低） | 内置默认值 | 代码中定义的默认值 |

## 核心配置字段

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `provider` | string | (自动检测) | LLM Provider 名称 |
| `model` | string | — | 模型 ID |
| `base_url` | string? | Provider 默认 | API 端点覆盖 |
| `api_key_env` | string? | Provider 默认 | API Key 环境变量名 |
| `system_prompt` | string? | 内置默认 | 自定义系统提示 |
| `max_iterations` | u32 | 50 | Agent 最大迭代次数 |
| `max_tokens` | u32? | 无限制 | Token 预算上限 |
| `max_timeout_secs` | u64 | 600 | 墙钟超时（秒） |
| `tool_timeout_secs` | u64 | 600 | 单工具调用超时（秒） |

## 工具策略

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `tools.allow` | string[] | [] | 允许的工具列表（空=全部允许） |
| `tools.deny` | string[] | [] | 禁止的工具列表（deny-wins） |
| `tool_policy_by_provider.<name>.allow` | string[] | — | Provider / model 级允许列表，精确 model ID 优先，其次 provider 名 |
| `tool_policy_by_provider.<name>.deny` | string[] | — | Provider / model 级禁止列表 |
| `tool_policy_by_provider.<name>.require_tags` | string[] | [] | Provider / model 级标签过滤 |

## Gateway 配置

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `gateway.channels` | object | {} | 频道配置 |
| `gateway.max_concurrent_sessions` | u32 | 10 | 最大并发会话数 |

## Serve 配置

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `serve.host` | string | "127.0.0.1" | 绑定地址 |
| `serve.port` | u16 | 50080 | 绑定端口 |

## Profile LLM 配置

Profile 文件位于 `~/.octos/profiles/<id>.json`。当前主分支的 profile LLM 配置不再只是顶层 `provider` / `model` 字段，而是放在 `config.llm` 下：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `config.llm.primary.family_id` | string | 主模型家族 ID，例如 `anthropic`、`openai` |
| `config.llm.primary.model_id` | string | 主模型 ID |
| `config.llm.primary.route.base_url` | string? | 当前模型路由的 API endpoint 覆盖 |
| `config.llm.primary.route.api_key_env` | string? | 当前模型路由的 API key 环境变量 |
| `config.llm.primary.route.api_type` | string? | OpenAI-compatible / Anthropic / 其他 API 类型提示 |
| `config.llm.primary.model_hints` | object | 模型能力、成本、上下文窗口等提示 |
| `config.llm.primary.cost_per_m` | object? | 每百万 token 成本 |
| `config.llm.primary.strong` | bool? | 是否标记为强模型 |
| `config.llm.fallbacks[]` | object[] | fallback 模型列表，元素结构同 `primary` |

## MCP 服务器

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `mcp_servers.<name>.command` | string | Stdio 模式：启动命令 |
| `mcp_servers.<name>.args` | string[] | 命令参数 |
| `mcp_servers.<name>.env` | object | 传递给子进程的环境变量 |
| `mcp_servers.<name>.url` | string | HTTP 模式：服务器 URL |

## Hooks

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `hooks[]` | object[] | Hook 列表 |
| `hooks[].event` | string | 触发事件，例如 `before_tool_call` / `after_tool_call` |
| `hooks[].command` | string[] | argv 形式的命令，不经过 shell |
| `hooks[].timeout_ms` | u64 | 超时时间 |
| `hooks[].tool_filter` | string[] | 仅匹配指定工具；空数组表示不按工具过滤 |

每个 hook 对象：`{ "event": "before_tool_call", "command": ["cmd", "arg1"], "timeout_ms": 5000, "tool_filter": ["shell"] }`

## 示例配置

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
