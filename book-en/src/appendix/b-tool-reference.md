# 附录 B：工具速查表

## 内置工具一览

| 工具名 | 分组 | 核心参数 | 功能 |
|--------|------|---------|------|
| `shell` | runtime | `command: string` | 在沙箱中执行 Shell 命令 |
| `read_file` | fs | `path: string, offset?: int, limit?: int` | 读取文件内容（支持分页） |
| `write_file` | fs | `path: string, content: string` | 写入文件（覆盖） |
| `edit_file` | fs | `path: string, old_string: string, new_string: string` | 精确字符串替换 |
| `diff_edit` | fs | `path: string, diff: string` | Diff 格式编辑 |
| `glob` | search | `pattern: string, path?: string` | 文件模式搜索 |
| `grep` | search | `pattern: string, path?: string, include?: string` | 内容正则搜索 |
| `list_dir` | search | `path: string` | 列出目录内容 |
| `web_search` | web | `query: string` | 网页搜索 |
| `web_fetch` | web | `url: string` | 获取网页内容 |
| `browser` | web | `url: string, action?: string` | 浏览器自动化 |
| `git` | — | `command: string` | Git 操作（feature: git） |
| `code_structure` | — | `path: string` | AST 代码结构分析（feature: ast） |

## 辅助工具

| 工具名 | 类型 | 功能 |
|--------|------|------|
| `spawn` | sessions | 后台异步执行子 Agent |
| `deep_search` | research | 深度网页研究 |
| `recall_memory` | memory | 检索长期记忆 |
| `save_memory` | memory | 保存到长期记忆 |
| `manage_skills` | admin | 管理 skills |
| `activate_tools` | admin | 激活延迟工具 |
| `send_file` | — | 发送文件给用户 |
| `message` | — | 发送消息到频道 |

## 工具分组策略

| 分组名 | 包含工具 | 典型策略 |
|--------|---------|---------|
| `group:fs` | read_file, write_file, edit_file, diff_edit | 文件操作 |
| `group:runtime` | shell | 命令执行 |
| `group:web` | web_search, web_fetch, browser | 网络操作 |
| `group:search` | glob, grep, list_dir | 代码搜索 |
| `group:sessions` | spawn | 后台任务 |
| `group:memory` | recall_memory, save_memory | 记忆操作 |
| `group:research` | deep_search, synthesize_research | 深度研究 |
| `group:admin` | manage_skills, activate_tools, configure_tool | 管理操作 |

## 策略配置示例

```json
{
  "tools": {
    "allow": ["group:fs", "group:search", "shell"],
    "deny": ["browser", "web_fetch"],
    "byProvider": {
      "ollama": {
        "allow": ["read_file", "shell", "grep"]
      }
    }
  }
}
```

**deny-wins 规则**：deny 列表优先于 allow 列表。上例中 `browser` 被禁止，即使它属于某个允许的分组。
