spec: task
name: "Ch13. 三种运行模式与配置体系"
inherits: project
tags: [part3, cli, gateway, serve, config, feature-flags]
depends: [ch05-agent-loop, ch10-message-bus]
estimate: 1d
---

## 意图

octos 提供 CLI/Gateway/Serve 三种运行模式，配合配置优先级、热加载、
Feature Flags 形成完整的运行时体系。本章帮助用户和贡献者理解如何选择
运行模式以及配置变更如何传播。

## 决策

- 源码文件: `crates/octos-cli/src/main.rs`, `config.rs`, `config_watcher.rs`
- 命令入口: `chat.rs`, `gateway.rs`, `serve.rs`(API routes)
- 图表: 三种模式架构对比图、配置优先级链路、热加载流程
- 工程决策侧栏: 热加载 vs 全重启的边界划分

## 边界

### 允许修改
- octos-book/chapters/ch13-*.md
- octos-book/assets/ch13-*

### 禁止做
- 不逐个讲解 91 个 REST 端点（附录 C 覆盖）
- 不重复 Agent Loop 或 Bus 细节

## 排除范围

- REST API 完整参考文档（附录覆盖）
- 前端 Web Dashboard 实现细节

## 完成条件

场景: 三种运行模式对比清晰
  测试: review_ch13_three_modes
  当 阅读运行模式小节
  那么 包含 CLI/Gateway/Serve 三种模式的架构对比表
  并且 每种模式说明了入口函数、适用场景、支持的功能边界

场景: 配置优先级完整
  测试: review_ch13_config_priority
  当 阅读配置体系小节
  那么 解释了本地 `.octos/config.json` > 全局 `~/.config/octos/config.json` 的优先级
  并且 说明了 Provider 自动检测的模型名匹配规则

场景: 热加载机制
  测试: review_ch13_hot_reload
  当 阅读热加载小节
  那么 解释了 SHA-256 hash 变更检测的实现
  并且 区分了热加载项（system prompt）和需重启项（provider/model/hooks）
  并且 说明了 config_watcher 的工作流程

场景: Feature Flags
  测试: review_ch13_feature_flags
  当 阅读 Feature Flags 小节
  那么 列出了主要的 feature flag（email/browser/api）
  并且 解释了条件编译在 Cargo.toml 中的配置方式

场景: 热加载边界侧栏
  测试: review_ch13_reload_sidebar
  当 阅读工程决策侧栏
  那么 解释了为什么 system prompt 可以热加载而 provider 需要重启
  并且 说明了这个边界划分的安全和一致性考量
