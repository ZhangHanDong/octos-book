# 附录 E：从源码构建与贡献指南

本附录面向希望直接修改 `octos` 源码的贡献者。内容以当前 `../octos` main 分支为准，主要对齐根 `Cargo.toml`、`CLAUDE.md` 的 Build & Test Commands、README 的 source build 说明，以及 `.github/workflows/ci.yml` 的 CI 分片策略。

## 环境要求

| 工具 | 版本要求 | 说明 |
|------|---------|------|
| Rust | ≥ `1.85.0` | 根 `Cargo.toml` 的 `rust-version` |
| Edition | `2024` | 根 `Cargo.toml` 的 workspace edition |
| Cargo resolver | `2` | workspace 使用 resolver v2 |
| Git | 2.x+ | clone、branch、commit |
| Node.js / npm | 仅 dashboard 改动需要 | `scripts/build-dashboard.sh` 会构建嵌入式 admin SPA |
| Chrome / Chromium | 仅 browser/deep-crawl 相关开发需要 | CDP/browser 工具与 deep-crawl skill 使用 |

当前仓库没有 `rust-toolchain.toml`，因此不要假设会自动 pin 工具链；本地工具链至少要满足 `rust-version = "1.85.0"`。

## 最小构建路径

```bash
# 1. 克隆仓库
git clone https://github.com/octos-org/octos.git
cd octos

# 2. 检查工具链
rustc --version
cargo --version

# 3. 构建整个 workspace
cargo build --workspace

# 4. 运行 workspace 测试
cargo test --workspace

# 5. 格式化与 lint
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
```

这组命令与 `CLAUDE.md` 和 CI 的基础检查保持一致。注意：`cargo build --workspace` 会构建 workspace 成员，但不会自动启用 `octos-cli/api`；如果你要运行 `octos serve`，必须用带 `api` 的 feature 组合构建或安装 CLI。

## 本地安装 CLI

README 和 `CLAUDE.md` 当前推荐的本地安装命令是：

```bash
cargo install --path crates/octos-cli \
    --features "api,telegram,discord,whatsapp,feishu,twilio,wecom,wecom-bot"
```

这个组合是 repo 内 canonical 默认安装集，原因是：

1. `api` 是 `octos serve` 的前提；不带 `api` 的二进制会缺少 serve 相关能力。
2. 频道 feature 只编译对应 transport，不需要的频道可以删掉。
3. 这个命令只安装 CLI 到 `~/.cargo/bin`，不会重建 dashboard，也不会替换 `install.sh` 安装的系统服务。

只需要本地 chat/serve 时，可以收缩为：

```bash
cargo install --path crates/octos-cli --features api
```

如果你修改了 dashboard 或想验证完整安装包流程，使用 README 中的本地 bundle 脚本：

```bash
./scripts/build-local-bundle.sh --install
./scripts/build-local-bundle.sh --install --tunnel
./scripts/build-local-bundle.sh --skip-dashboard
```

`build-local-bundle.sh` 会调用 `scripts/build-dashboard.sh`，再通过 `scripts/milestone-ci.sh release-bundle` 复用发布构建的 `FEATURES` / `SKILL_CRATES` 组合，最后生成可被 `install.sh` 识别的本地 tarball。

## CI 等价测试矩阵

当前 CI 没有只依赖一个巨大的 `cargo test --workspace --all-features`。为控制内存和链接压力，它把重 crate 拆成独立 job。贡献者本地排查时可以按同样方式缩小范围：

| 场景 | 命令 |
|------|------|
| 格式检查 | `cargo fmt --all -- --check` |
| Clippy | `cargo clippy --workspace --all-targets -- -D warnings` |
| 全 workspace 构建 | `cargo build --workspace` |
| 只编译测试二进制 | `cargo test --workspace --no-run` |
| 核心类型 | `cargo test -p octos-core` |
| 记忆系统 | `cargo test -p octos-memory` |
| LLM lib 测试 | `cargo test -p octos-llm --lib` |
| LLM integration 测试 | `cargo test -p octos-llm --tests` |
| 消息总线 | `cargo test -p octos-bus` |
| Pipeline | `cargo test -p octos-pipeline` |
| Plugin SDK | `cargo test -p octos-plugin` |
| Swarm | `cargo test -p octos-swarm` |
| Agent lib | `cargo test -p octos-agent --lib` |
| Agent integration | `cargo test -p octos-agent --tests` |
| CLI lib | `cargo test -p octos-cli --lib` |
| CLI integration | `cargo test -p octos-cli --tests` |
| CLI API 认证 | `cargo test -p octos-cli --features api api::auth_handlers` |
| Gateway runtime | `cargo test -p octos-cli gateway_runtime::tests --features api -- --nocapture` |
| Harness starter skills | `cargo test -p harness-starter-generic` 等四个 starter crate |
| 文档测试 | `cargo test --workspace --doc` |

针对单个失败测试，优先使用：

```bash
cargo test -p octos-agent test_name -- --nocapture
cargo test -p octos-cli gateway_runtime::tests --features api -- --nocapture
```

## 代码规范

### Rust 与 lint

- Workspace edition: `2024`
- Minimum Rust: `1.85.0`
- Workspace lint: `unsafe_code = "deny"`
- 格式化：`cargo fmt --all`
- lint gate：`cargo clippy --workspace --all-targets -- -D warnings`

### 错误处理

- 项目约定使用 `eyre` / `color-eyre`，不要引入 `anyhow`。
- 库层函数通常返回 `eyre::Result<T>`；CLI binary 初始化 `color_eyre::install()`。
- 避免无解释的 `.unwrap()`；如果是不变量，用 `.expect("...")` 写清楚原因。

### 并发与平台边界

- 异步代码使用 Tokio；共享 trait object 常见形式是 `Arc<dyn Trait>`。
- shutdown signaling 使用 `AtomicBool`，store/load 语义需要保持一致。
- 跨平台命令执行要考虑 Unix `sh -c` 与 Windows `cmd /C` 的差异。
- 安全相关路径要遵守现有 sandbox、SSRF、symlink-safe I/O 和 env sanitization 约束，不要绕开公共实现。

### 测试放置

- 单元测试优先放在同文件 `#[cfg(test)] mod tests`。
- integration tests 放在 crate 的 `tests/` 目录。
- 外部服务依赖测试应使用 `#[ignore]` 或清晰的环境变量 gate。
- 临时目录使用 `tempfile`，不要写入用户真实 home 或固定系统路径。

## TDD 工作流

`CLAUDE.md` 明确要求代码变更遵循 RED → GREEN → REFACTOR：

1. **RED**：先写能暴露问题的失败测试，测试名使用 `should_<expected>_when_<condition>` 风格。
2. **GREEN**：写最小实现让目标测试通过。
3. **REFACTOR**：在测试保护下清理结构，再跑相关 crate 测试和必要的 CI 子集。

实践上不要一开始就跑全量 CI。先用最小 selector 锁定问题，再逐步扩大：

```bash
cargo test -p octos-core should_do_x_when_y
cargo test -p octos-agent --lib
cargo test -p octos-agent --tests
cargo clippy --workspace --all-targets -- -D warnings
```

## PR 提交指南

### 分支命名

```text
feature/add-wechat-channel
fix/ssrf-ipv6-bypass
refactor/tool-registry-lru
docs/update-feature-flags
test/add-gateway-regression
```

### Commit Message

推荐使用 conventional commit 风格：

```text
feat(bus): add WeChat channel integration

- Implement WeChatChannel with encryption support
- Add message coalescing for channel limits
- Include session fork handling
```

常用类型包括 `feat`、`fix`、`refactor`、`docs`、`test`、`chore`。提交前至少确认：

1. 只包含当前任务相关改动。
2. 已运行与改动范围匹配的测试。
3. 文档、spec、示例配置与代码行为一致。
4. 安全相关变更有专门测试或清晰审查说明。

## 项目结构速查

```text
octos/
├── Cargo.toml                 # Workspace: 26 members, Rust 2024, rust-version 1.85.0
├── crates/
│   ├── octos-core/            # 核心类型与 UTF-8 工具
│   ├── octos-llm/             # LLM providers、failover、adaptive routing、credential pool
│   ├── octos-memory/          # episodic memory、MEMORY.md、hybrid search
│   ├── octos-agent/           # agent loop、tools、sandbox、MCP、plugins、hooks
│   ├── octos-bus/             # sessions、channels、coalescing、cron、heartbeat
│   ├── octos-cli/             # CLI、config、serve/gateway、profiles、auth、monitor
│   ├── octos-pipeline/        # DOT workflow engine
│   ├── octos-plugin/          # plugin/skill manifest 与 gating
│   ├── octos-swarm/           # swarm orchestration primitive
│   ├── octos-dora-mcp/        # Dora-RS 到 MCP tool bridge
│   ├── octos-sandbox/         # Windows AppContainer helper
│   ├── app-skills/            # news、deep-search、deep-crawl、send-email、harness starters 等
│   └── platform-skills/voice/ # voice platform skill
├── dashboard/                 # admin dashboard 前端
├── scripts/                   # install、bundle、release、dashboard build 脚本
├── docs/                      # 用户文档
└── .github/workflows/         # CI、release、platform build workflow
```
