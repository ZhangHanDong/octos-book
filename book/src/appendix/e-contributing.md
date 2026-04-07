# 附录 E：从源码构建与贡献指南

## 环境要求

| 工具 | 版本要求 | 说明 |
|------|---------|------|
| Rust | ≥ 1.85.0 | `Cargo.toml` 中的 `rust-version` |
| Edition | 2024 | Rust 2024 edition |
| 操作系统 | Linux / macOS / Windows | 全平台支持 |
| Git | ≥ 2.x | 代码管理 |

## 构建步骤

```bash
# 1. 克隆仓库
git clone https://github.com/octos-org/octos.git
cd octos

# 2. 检查工具链
rustup show  # 确认 Rust 版本 ≥ 1.85.0

# 3. 构建（默认 features）
cargo build

# 4. 运行测试
cargo test --workspace

# 5. 完整构建（所有 features）
cargo build --all-features

# 6. 完整测试
cargo test --workspace --all-features

# 7. Clippy 检查
cargo clippy --workspace --all-features -- -D warnings

# 8. 格式检查
cargo fmt --all -- --check
```

## 代码规范

### Rust 版本与 Edition

- **Edition**: 2024（`Cargo.toml` 中 `edition = "2024"`）
- **Rust 最低版本**: 1.85.0
- **unsafe**: 全 workspace 禁止（`[workspace.lints.rust] unsafe_code = "deny"`）

### 错误处理

- 使用 `eyre::Result` 和 `color-eyre` 而非 `anyhow`
- 库 crate 返回 `eyre::Result`，binary crate 在 `main()` 中初始化 `color_eyre::install()`
- 避免 `.unwrap()`——使用 `?` 传播或 `.expect("reason")` 说明不变量

### 代码风格

- 遵循 `rustfmt` 默认配置
- `#![warn(clippy::all)]` 在每个 crate 的 `lib.rs`/`main.rs`
- 公有 API 添加文档注释（`///`）
- 模块级文档使用 `//!`

### 测试策略

- 单元测试与代码放在同一文件的 `#[cfg(test)] mod tests` 中
- 集成测试放在 `tests/` 目录
- 使用 `tokio::test` 宏进行异步测试
- 使用 `tempfile` crate 管理临时文件

## TDD 工作流

octos 鼓励 RED → GREEN → REFACTOR 的 TDD 工作流：

1. **RED**：先写失败的测试，明确预期行为
2. **GREEN**：写最少的代码让测试通过
3. **REFACTOR**：在测试保护下重构代码

```bash
# 运行特定测试
cargo test -p octos-core test_task_status

# 运行特定 crate 的所有测试
cargo test -p octos-memory

# 带输出运行测试
cargo test -p octos-agent -- --nocapture
```

## PR 提交指南

### 分支命名

```
feature/add-wechat-channel
fix/ssrf-ipv6-bypass
refactor/tool-registry-lru
docs/ch05-agent-loop
```

### Commit Message

```
feat(bus): add WeChat channel integration

- Implement WeChatChannel with encryption support
- Add message coalescing for 2000-char limit
- Include /new session fork support

Closes #123
```

格式：`type(scope): description`。类型包括 `feat`、`fix`、`refactor`、`docs`、`test`、`chore`。

### Review 流程

1. 所有 PR 必须通过 CI（`cargo test --workspace --all-features` + `cargo clippy` + `cargo fmt --check`）
2. 至少一个 maintainer review
3. 与安全相关的变更（sandbox、policy、sanitize）需要两个 reviewer
4. 不允许 force push 到 `main` 分支

## 项目结构速查

```
octos/
├── Cargo.toml                # Workspace 定义
├── crates/
│   ├── octos-core/           # 核心类型（第 2 章）
│   ├── octos-llm/            # LLM Provider（第 3 章）
│   ├── octos-memory/         # 记忆系统（第 4 章）
│   ├── octos-agent/          # Agent 运行时（第 5-9 章）
│   ├── octos-bus/            # 消息总线（第 10 章）
│   ├── octos-cli/            # CLI 入口（第 13 章）
│   ├── octos-pipeline/       # 工作流引擎（第 12 章）
│   ├── octos-plugin/         # 插件 SDK（第 9 章）
│   ├── octos-sandbox/        # Windows 沙箱（第 7 章）
│   ├── app-skills/           # 应用级 Skills
│   └── platform-skills/      # 平台级 Skills
```
