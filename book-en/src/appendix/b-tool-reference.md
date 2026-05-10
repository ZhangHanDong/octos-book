# Appendix B: Tool Quick Reference

This appendix reflects the current `../octos` main branch, especially `octos-agent/src/tools/`, `octos-agent/src/tools/policy.rs`, and CLI runtime registration paths. octos tools are not a fixed list of 14 items: the base registry, feature flags, profiles, Serve/Admin API, MCP, plugins, and app-skills all affect the final tool set exposed to an Agent.

## Core Built-in Tools

| Tool | Group | Core Parameters | Purpose |
|------|-------|-----------------|---------|
| `shell` | `group:runtime` | `command: string` | Execute shell commands under policy and sandbox constraints |
| `read_file` | `group:fs` | `path: string, offset?: int, limit?: int` | Read file content with pagination |
| `write_file` | `group:fs` | `path: string, content: string` | Write a file and record workspace changes |
| `edit_file` | `group:fs` | `path: string, old_string: string, new_string: string` | Exact string replacement |
| `diff_edit` | `group:fs` | `path: string, diff: string` | Apply a unified-diff-style patch |
| `glob` | `group:search` | `pattern: string, path?: string` | File path pattern search |
| `grep` | `group:search` | `pattern: string, path?: string, include?: string` | Regex content search |
| `list_dir` | `group:search` | `path: string` | List directory contents |
| `web_search` | `group:web` | `query: string, count?: int` | Web search |
| `web_fetch` | `group:web` | `url: string, max_chars?: int, extract_mode?: string` | Fetch a web page and extract text |
| `browser` | `group:web` | `url?: string, action?: string, selector?: string` | Browser automation |
| `spawn` | `group:sessions` | `prompt: string, agent_definition_id?: string, allowed_tools?: string[]` | Run a child Agent asynchronously in the background |
| `recall_memory` | `group:memory` | `query: string, limit?: int` | Retrieve long-term memory |
| `save_memory` | `group:memory` | `name: string, content: string` | Save long-term memory |

## Feature / Runtime-Registered Tools

| Tool | Source | Group | Core Parameters | Purpose |
|------|--------|-------|-----------------|---------|
| `git` | `git` feature | - | `command: string` | Git operations and repository queries |
| `code_structure` | `ast` feature | - | `path: string` | AST code structure analysis |
| `deep_search` | bundled app skill / runtime | `group:research` | `query: string` | Deep web research |
| `synthesize_research` | runtime | `group:research` | `findings: array/object` | Synthesize research findings |
| `deep_crawl` | bundled app skill / runtime | `group:research` | `url: string, max_depth?: int` | Deep crawl a site |
| `run_pipeline` | CLI/Gateway/Serve per-session | - | `dot: string` or pipeline config | Execute a DOT workflow |
| `model_check` | CLI runtime | `group:admin` | `action: string` | Inspect or switch model configuration |
| `configure_tool` | core runtime | `group:admin` | `action: string, tool?: string, key?: string, value?: any` | View, set, or reset configurable tool fields |
| `activate_tools` | core runtime | - | `tools: string[]` | Activate deferred tools or tool groups |
| `manage_skills` | core runtime | `group:admin` | `action: string, name?: string` | Manage local skills |
| `check_background_tasks` | core runtime | - | `status?: string` | Inspect background task status |
| `read_task_output` | core runtime | - | `task_id: string, mode: object` | Read background task output |
| `check_workspace_contract` | core runtime | - | `path?: string` | Check workspace contract state |
| `workspace_log` / `workspace_show` / `workspace_diff` | workspace history | - | Git revision / diff parameters | Inspect workspace change history |
| `send_file` | channel runtime | - | `path: string, caption?: string` | Send a file to the user |
| `message` | channel runtime | `group:delegated` deny list | `text: string` | Send progress messages to the channel |
| `send_app_card` / `show_weather_card` | App UI runtime | media-style capability | card/weather payload | Send structured App UI cards |
| `delegate_task` | delegate runtime | `group:delegated` deny list | `task: string, allowed_tools?: string[]` | Synchronously delegate a child task |

## Serve/Admin API Tools

These tools are registered only in Serve/Admin API contexts and come from `octos-agent/src/tools/admin/`.

| Category | Tools |
|----------|-------|
| Profile management | `admin_list_profiles`, `admin_profile_status`, `admin_start_profile`, `admin_stop_profile`, `admin_restart_profile`, `admin_enable_profile`, `admin_update_profile` |
| System monitoring | `admin_view_logs`, `admin_system_health`, `admin_system_metrics`, `admin_provider_metrics`, `admin_manage_watchdog` |
| Diagnostics | `admin_view_sessions`, `admin_cron_status`, `admin_check_config` |
| Multi-tenant sub-accounts | `admin_list_sub_accounts`, `admin_create_sub_account` |
| Skill / platform management | `admin_manage_skills`, `admin_platform_skills`, `admin_update_octos` |

## Tool Group Policy

Group definitions come from `octos-agent/src/tools/policy.rs`.

| Group | Member Tools | Typical Policy |
|-------|--------------|----------------|
| `group:fs` | `read_file`, `write_file`, `edit_file`, `diff_edit` | File operations |
| `group:runtime` | `shell` | Command execution |
| `group:web` | `web_search`, `web_fetch`, `browser` | Network operations |
| `group:search` | `glob`, `grep`, `list_dir` | Code search |
| `group:sessions` | `spawn` | Background tasks |
| `group:memory` | `recall_memory`, `save_memory` | Memory operations |
| `group:research` | `deep_search`, `synthesize_research`, `deep_crawl` | Deep research |
| `group:admin` | `manage_skills`, `configure_tool`, `model_check` | Administration |
| `group:media` | `mofa_comic`, `mofa_slides`, `mofa_infographic`, `mofa_cards`, `fm_tts`, `fm_voice_list` | Media generation and voice |
| `group:delegated` | `delegate_task`, `spawn`, `send_message`, `message`, `save_memory`, `execute_code` | Canonical deny list for delegated children |

## Policy Example

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

**Deny-wins rule**: deny lists take precedence over allow lists. In the example above, `browser` is denied even if it belongs to an allowed group. Profile-level allow/deny entries also support plain tool names, `group:<id>`, and `<prefix>*` wildcards; spawn-only tools are retained so profile filtering does not break background-task wiring.
