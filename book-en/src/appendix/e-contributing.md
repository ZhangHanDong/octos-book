# Appendix E: Building from Source and Contributing

This appendix is for contributors who want to modify `octos` directly from source. It reflects the current `../octos` main branch and aligns with the root `Cargo.toml`, the Build & Test Commands in `CLAUDE.md`, the source-build notes in README, and the CI sharding strategy in `.github/workflows/ci.yml`.

## Environment Requirements

| Tool | Requirement | Notes |
|------|-------------|-------|
| Rust | ≥ `1.85.0` | Root `Cargo.toml` `rust-version` |
| Edition | `2024` | Workspace edition in root `Cargo.toml` |
| Cargo resolver | `2` | Workspace uses resolver v2 |
| Git | 2.x+ | clone, branch, commit |
| Node.js / npm | Only for dashboard changes | `scripts/build-dashboard.sh` builds the embedded admin SPA |
| Chrome / Chromium | Only for browser/deep-crawl work | CDP/browser tools and the deep-crawl skill use it |

The current repository has no `rust-toolchain.toml`, so do not assume automatic toolchain pinning. Your local Rust toolchain must at least satisfy `rust-version = "1.85.0"`.

## Minimal Build Path

```bash
# 1. Clone
git clone https://github.com/octos-org/octos.git
cd octos

# 2. Check the toolchain
rustc --version
cargo --version

# 3. Build the whole workspace
cargo build --workspace

# 4. Run workspace tests
cargo test --workspace

# 5. Format and lint
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
```

These commands match `CLAUDE.md` and the baseline CI checks. Note that `cargo build --workspace` builds workspace members but does not automatically enable `octos-cli/api`. To run `octos serve`, build or install the CLI with the `api` feature enabled.

## Local CLI Install

The README and `CLAUDE.md` currently recommend this local install command:

```bash
cargo install --path crates/octos-cli \
    --features "api,telegram,discord,whatsapp,feishu,twilio,wecom,wecom-bot"
```

This is the repository's canonical default install set:

1. `api` is required by `octos serve`; a binary without `api` is missing serve-related capability.
2. Channel features compile only the transports you select, so remove channels you do not need.
3. This command only installs the CLI into `~/.cargo/bin`; it does not rebuild the dashboard or replace a system service installed by `install.sh`.

For local chat/serve only, this smaller install is enough:

```bash
cargo install --path crates/octos-cli --features api
```

If you changed the dashboard or need to validate the full installer flow, use the local bundle script described in the README:

```bash
./scripts/build-local-bundle.sh --install
./scripts/build-local-bundle.sh --install --tunnel
./scripts/build-local-bundle.sh --skip-dashboard
```

`build-local-bundle.sh` calls `scripts/build-dashboard.sh`, delegates the release build to `scripts/milestone-ci.sh release-bundle` to reuse the release `FEATURES` / `SKILL_CRATES` set, and produces a local tarball that `install.sh` can consume.

## CI-Equivalent Test Matrix

Current CI does not rely on one large `cargo test --workspace --all-features`. To control memory and link pressure, heavy crates are split into dedicated jobs. Local contributors can use the same structure to narrow failures:

| Scenario | Command |
|----------|---------|
| Format check | `cargo fmt --all -- --check` |
| Clippy | `cargo clippy --workspace --all-targets -- -D warnings` |
| Full workspace build | `cargo build --workspace` |
| Compile test binaries only | `cargo test --workspace --no-run` |
| Core types | `cargo test -p octos-core` |
| Memory system | `cargo test -p octos-memory` |
| LLM lib tests | `cargo test -p octos-llm --lib` |
| LLM integration tests | `cargo test -p octos-llm --tests` |
| Message bus | `cargo test -p octos-bus` |
| Pipeline | `cargo test -p octos-pipeline` |
| Plugin SDK | `cargo test -p octos-plugin` |
| Swarm | `cargo test -p octos-swarm` |
| Agent lib | `cargo test -p octos-agent --lib` |
| Agent integration | `cargo test -p octos-agent --tests` |
| CLI lib | `cargo test -p octos-cli --lib` |
| CLI integration | `cargo test -p octos-cli --tests` |
| CLI API auth | `cargo test -p octos-cli --features api api::auth_handlers` |
| Gateway runtime | `cargo test -p octos-cli gateway_runtime::tests --features api -- --nocapture` |
| Harness starter skills | `cargo test -p harness-starter-generic` and the other three starter crates |
| Doc tests | `cargo test --workspace --doc` |

For a single failing test, prefer a narrow selector first:

```bash
cargo test -p octos-agent test_name -- --nocapture
cargo test -p octos-cli gateway_runtime::tests --features api -- --nocapture
```

## Code Conventions

### Rust And Lints

- Workspace edition: `2024`
- Minimum Rust: `1.85.0`
- Workspace lint: `unsafe_code = "deny"`
- Formatting: `cargo fmt --all`
- Lint gate: `cargo clippy --workspace --all-targets -- -D warnings`

### Error Handling

- The project uses `eyre` / `color-eyre`; do not introduce `anyhow`.
- Library functions commonly return `eyre::Result<T>`; CLI binaries initialize `color_eyre::install()`.
- Avoid unexplained `.unwrap()` calls. If a value is an invariant, use `.expect("...")` and state the invariant.

### Concurrency And Platform Boundaries

- Async code uses Tokio; shared trait objects commonly use `Arc<dyn Trait>`.
- Shutdown signaling uses `AtomicBool`; keep store/load ordering semantics consistent.
- Cross-platform command execution must account for Unix `sh -c` and Windows `cmd /C`.
- Security-sensitive paths must reuse existing sandbox, SSRF, symlink-safe I/O, and environment-sanitization code instead of bypassing shared implementations.

### Test Placement

- Prefer inline unit tests in `#[cfg(test)] mod tests`.
- Put integration tests under each crate's `tests/` directory.
- Tests requiring external services should use `#[ignore]` or a clear environment-variable gate.
- Use `tempfile` for temporary directories; do not write to a user's real home directory or fixed system paths.

## TDD Workflow

`CLAUDE.md` requires code changes to follow RED → GREEN → REFACTOR:

1. **RED**: Write a failing test that exposes the behavior; use names like `should_<expected>_when_<condition>`.
2. **GREEN**: Implement the smallest change that passes the target test.
3. **REFACTOR**: Clean up under test coverage, then run the relevant crate tests and CI subset.

In practice, do not start with full CI. Use the smallest selector first, then expand:

```bash
cargo test -p octos-core should_do_x_when_y
cargo test -p octos-agent --lib
cargo test -p octos-agent --tests
cargo clippy --workspace --all-targets -- -D warnings
```

## Pull Request Guide

### Branch Names

```text
feature/add-wechat-channel
fix/ssrf-ipv6-bypass
refactor/tool-registry-lru
docs/update-feature-flags
test/add-gateway-regression
```

### Commit Message

Prefer conventional commit style:

```text
feat(bus): add WeChat channel integration

- Implement WeChatChannel with encryption support
- Add message coalescing for channel limits
- Include session fork handling
```

Common types include `feat`, `fix`, `refactor`, `docs`, `test`, and `chore`. Before submitting, confirm:

1. The diff only contains changes related to the task.
2. Tests matching the change scope have been run.
3. Documentation, specs, example config, and code behavior agree.
4. Security-sensitive changes include dedicated tests or clear review notes.

## Project Structure Quick Reference

```text
octos/
├── Cargo.toml                 # Workspace: 26 members, Rust 2024, rust-version 1.85.0
├── crates/
│   ├── octos-core/            # Core types and UTF-8 helpers
│   ├── octos-llm/             # LLM providers, failover, adaptive routing, credential pool
│   ├── octos-memory/          # episodic memory, MEMORY.md, hybrid search
│   ├── octos-agent/           # agent loop, tools, sandbox, MCP, plugins, hooks
│   ├── octos-bus/             # sessions, channels, coalescing, cron, heartbeat
│   ├── octos-cli/             # CLI, config, serve/gateway, profiles, auth, monitor
│   ├── octos-pipeline/        # DOT workflow engine
│   ├── octos-plugin/          # plugin/skill manifest and gating
│   ├── octos-swarm/           # swarm orchestration primitive
│   ├── octos-dora-mcp/        # Dora-RS to MCP tool bridge
│   ├── octos-sandbox/         # Windows AppContainer helper
│   ├── app-skills/            # news, deep-search, deep-crawl, send-email, harness starters, etc.
│   └── platform-skills/voice/ # voice platform skill
├── dashboard/                 # admin dashboard frontend
├── scripts/                   # install, bundle, release, dashboard build scripts
├── docs/                      # user documentation
└── .github/workflows/         # CI, release, platform build workflows
```
