# 附录 D：Feature Flags 一览

## octos-cli Feature Flags

| Feature | 启用功能 | 额外依赖 | 默认 |
|---------|---------|---------|------|
| `api` | Web API 服务器、SSE 流式、监控、用户管理 | axum, tower-http, metrics, rust-embed | 否 |
| `telegram` | Telegram 频道集成 | teloxide | 否 |
| `discord` | Discord 频道集成 | serenity | 否 |
| `slack` | Slack 频道集成 | tokio-tungstenite | 否 |
| `whatsapp` | WhatsApp 频道集成 | — | 否 |
| `feishu` | 飞书频道集成 | — | 否 |
| `email` | 邮件收发集成 | async-imap, lettre, mailparse | 否 |
| `matrix` | Matrix 频道集成 | — | 否 |
| `twilio` | Twilio SMS/Voice 集成 | — | 否 |
| `wecom` | 企业微信集成 | — | 否 |
| `qq-bot` | QQ Bot 集成 | — | 否 |

## octos-agent Feature Flags

| Feature | 启用功能 | 额外依赖 | 默认 |
|---------|---------|---------|------|
| `git` | Git 操作工具（GitTool） | gix | 否 |
| `ast` | AST 代码结构分析工具 | tree-sitter, tree-sitter-{rust,python,js,ts} | 否 |

## octos-bus Feature Flags

| Feature | 启用功能 | 额外依赖 | 默认 |
|---------|---------|---------|------|
| `api` | API 频道（REST/SSE 接入） | — | 否 |
| `telegram` | Telegram 频道实现 | teloxide | 否 |
| `discord` | Discord 频道实现 | serenity | 否 |
| `slack` | Slack 频道实现 | tokio-tungstenite | 否 |
| `email` | 邮件频道实现 | async-imap, lettre | 否 |

## 编译示例

```bash
# 最小 CLI（无频道集成、无 Web 服务器）
cargo build --release

# CLI + Web API
cargo build --release --features api

# 完整 Gateway（所有频道）
cargo build --release --features "api,telegram,discord,slack,email,feishu,whatsapp,matrix"

# 开发者完整构建（所有功能）
cargo build --release --all-features
```

## 设计原则

Feature flags 遵循"最小默认"原则——默认编译只包含 CLI 核心功能，所有频道集成和可选工具通过 flags 按需启用。这确保了：

1. **最小二进制体积**：不需要的功能不编译
2. **最少依赖**：不使用 Telegram？就不引入 teloxide 及其依赖链
3. **最快编译**：开发时只编译需要的功能
