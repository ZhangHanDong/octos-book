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
| `tools.byProvider.<name>.allow` | string[] | — | Provider 级允许列表 |
| `tools.byProvider.<name>.deny` | string[] | — | Provider 级禁止列表 |

## Gateway 配置

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `gateway.channels` | object | {} | 频道配置 |
| `gateway.max_concurrent_sessions` | u32 | 10 | 最大并发会话数 |

## Serve 配置

| 字段路径 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `serve.host` | string | "127.0.0.1" | 绑定地址 |
| `serve.port` | u16 | 3000 | 绑定端口 |

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
| `hooks.before_tool_call` | object[] | 工具调用前钩子 |
| `hooks.after_tool_call` | object[] | 工具调用后钩子 |
| `hooks.before_llm_call` | object[] | LLM 调用前钩子 |
| `hooks.after_llm_call` | object[] | LLM 调用后钩子 |

每个 hook 对象：`{ "command": ["cmd", "arg1"], "timeout_ms": 5000, "tool_filter": "shell" }`

## 示例配置

```json
{
  "model": "claude-sonnet-4-20250514",
  "system_prompt": "You are a helpful coding assistant.",
  "max_iterations": 30,
  "tools": {
    "deny": ["browser"],
    "byProvider": {
      "ollama": { "allow": ["read_file", "shell", "grep"] }
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
  "hooks": {
    "before_tool_call": [
      { "command": ["./audit-hook.sh"], "timeout_ms": 3000 }
    ]
  }
}
```
