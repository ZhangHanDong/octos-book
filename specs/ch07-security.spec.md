spec: task
name: "Ch7. 安全纵深：从沙箱到 Prompt 注入防御"
inherits: project
tags: [part2, security, sandbox, ssrf, prompt-injection]
depends: [ch06-tool-system]
estimate: 1.5d
---

## 意图

安全是 AI Agent 平台的生命线。本章以纵深防御的视角，从最外层的沙箱隔离到
最内层的 prompt 注入检测，逐层展示 octos 的安全体系。这一章对所有四类读者
都至关重要：Rust 开发者学安全编码模式，AI 开发者学 Agent 安全。

## 决策

- 源码文件: `../octos/crates/octos-agent/src/sandbox/mod.rs`、`../octos/crates/octos-agent/src/sandbox/bwrap.rs`、`../octos/crates/octos-agent/src/sandbox/macos.rs`、`../octos/crates/octos-agent/src/tools/ssrf.rs`、`../octos/crates/octos-agent/src/prompt_guard.rs`、`../octos/crates/octos-agent/src/sanitize.rs`、`../octos/crates/octos-agent/src/policy.rs`、`../octos/crates/octos-agent/src/tools/shell.rs`
- 图表: 安全纵深分层图
- 工程决策侧栏: workspace 级 `deny(unsafe_code)` 的实践意义

## 边界

### 允许修改
- octos-book/chapters/ch07-*.md
- octos-book/assets/ch07-*

### 禁止做
- 不提供可被利用的攻击代码示例
- 不暴露具体的绕过路径

## 排除范围

- 认证/OAuth（Ch14 覆盖）
- 网络层安全（TLS 等）

## 完成条件

场景: 沙箱三后端完整对比
  测试: review_ch07_sandbox_backends
  当 阅读沙箱小节
  那么 分别解释了 Bwrap、macOS sandbox-exec、Docker 三种后端的能力和限制
  并且 说明了 `SandboxMode::Auto` 的有序探测链：Linux `bwrap`、macOS `sandbox-exec`、Windows `octos-sandbox` helper、Docker、最后 `NoSandbox`
  并且 澄清 Windows 自动模式检查的是 helper 可用性，而不是抽象的 AppContainer 能力
  并且 说明了 `NoSandbox` 会打印无隔离警告，而不是静默降级
  并且 展示了 SBPL 路径注入防护的具体实现

场景: SSRF 防护机制
  测试: review_ch07_ssrf
  当 阅读 SSRF 防护小节
  那么 解释了私有 IP 阻断的具体范围（IPv4 private + IPv6 ULA/link-local）
  并且 说明了 DNS 失败关闭（fail-close）策略
  并且 引用了 `tools/ssrf.rs` 的具体代码

场景: Prompt 注入检测
  测试: review_ch07_prompt_injection
  当 阅读 prompt 注入小节
  那么 列出了 5 类威胁的分类
  并且 说明了当前共有 11 个检测模式
  并且 说明了高/中严重性会替换为 `[injection-blocked:<kind>]`，低严重性仅记录日志
  并且 解释了当前主接线点是在 `sanitize_tool_output()` 这条工具输出回写链路上
  并且 明确写出 prompt guard 是 defense-in-depth，不是安全边界
  但是 不包含可直接利用的攻击 payload

场景: 凭据脱敏覆盖完整
  测试: review_ch07_credential_sanitization
  当 阅读凭据脱敏小节
  那么 列出了 7 类凭据模式（OpenAI、Anthropic、AWS、GitHub、GitLab、Bearer、通用 secret assignment）
  并且 区分了 2 类额外的高噪声/高风险模式（base64 data URI、64+ 字符十六进制串）
  并且 说明了工具输出清理顺序是 data URI、hex、credentials、prompt injection

场景: BLOCKED_ENV_VARS 共享机制
  测试: review_ch07_blocked_env_vars
  当 阅读环境变量安全小节
  那么 列出了 18 个被阻止的环境变量
  并且 说明了该常量定义在沙箱模块中
  并且 说明了其当前复用范围至少包括沙箱、MCP、hooks、browser、site_crawl、exec_env、plugins 与 CLI process manager
  但是 不把共享范围错误地写成一个过窄的封闭集合

场景: ShellTool SafePolicy
  测试: review_ch07_safe_policy
  当 阅读 SafePolicy 小节
  那么 说明了 `ShellTool::new()` 默认注入 `SafePolicy::default()`
  并且 列出了被拒绝的危险命令模式与需要批准的 ask 模式
  并且 说明了 whitespace 归一化在匹配前的作用
  并且 说明了 ask 决策在当前非交互执行路径里会直接拒绝
  并且 明确写出它只捕获常见事故，不是安全边界

场景: 基础设施安全避免漂移数字
  测试: review_ch07_infra_safety
  当 阅读基础设施安全小节
  那么 说明了 workspace 级 `deny(unsafe_code)` 的工程意义
  并且 不把 crate 数量或代码行数写成未经验证的固定事实
  并且 说明了 `SecretString` 在 provider 中作为 `api_key` 存储，并且只在真正组装请求时显式 `expose_secret()`

场景: 分层图与源码锚点完整
  测试: review_ch07_diagram_and_sources
  当 阅读本章总览与各节引用
  那么 包含一张安全纵深分层图
  并且 图中按沙箱、SSRF、工具策略、输出清理、prompt guard 组织主要防线
  并且 主要论述锚定到 `../octos/crates/octos-agent/src/sandbox/mod.rs`、`../octos/crates/octos-agent/src/sandbox/bwrap.rs`、`../octos/crates/octos-agent/src/sandbox/macos.rs`、`../octos/crates/octos-agent/src/tools/ssrf.rs`、`../octos/crates/octos-agent/src/prompt_guard.rs`、`../octos/crates/octos-agent/src/sanitize.rs`、`../octos/crates/octos-agent/src/policy.rs`、`../octos/crates/octos-agent/src/tools/shell.rs`
