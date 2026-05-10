# 第 12 章：octos-pipeline：DOT 图驱动的工作流引擎

> **定位**：本章对照 `../octos/crates/octos-pipeline/src/`，解释 octos 如何把 Graphviz DOT 图解析成带类型的 `PipelineGraph`，以及执行器如何在顺序节点、静态并行、动态并行、检查点恢复、deadline 护栏和父会话资源继承之间切换。前置依赖：第 5 章、第 8 章、第 11 章。适用场景：需要理解多步骤 Agent 编排机制的开发者。

当任务已经不是“单个 Agent 循环 + 几次工具调用”能解决的问题时，就需要显式的工作流编排。典型例子是：先规划研究角度，再并发检索，再汇总分析，最后生成报告。`octos-pipeline` 解决的就是这类问题，但它的当前实现和“传统 DAG 调度器”的直觉并不完全一样：它既有图结构，也有运行时分支选择、并发汇合、模型路由和工具策略继承；同时还通过 `PipelineHostContext` 继承父 session 的文件缓存、子 Agent 输出路由、任务监督器和成本账本，让 pipeline 节点不再像早期实现那样孤立运行。

---

## 12.1 DOT 图如何进入运行时

### 12.1.1 为什么是 DOT

octos 用 Graphviz DOT 定义工作流，而不是 YAML/JSON。原因不是“DOT 更潮”，而是它天然把节点和边作为一等语义：同一个文件既能被执行器解析，也能直接被 Graphviz 渲染成图。

```dot
digraph research {
    graph [label="Deep Research", default_model="cheap"]

    start [label="规划", handler="dynamic_parallel",
           converge="analyze",
           planner_model="strong",
           worker_prompt="围绕 {task} 做资料检索，并保留来源",
           tools="deep_search",
           max_tasks="6"]

    analyze [label="分析", handler="codergen",
             model="strong",
             tools="read_file,write_file"]

    finish [label="结束", shape="Msquare"]

    start -> analyze
    analyze -> finish
}
```

这个例子已经体现了当前 parser 支持的几类关键能力：

- 图级属性：`graph [label=..., default_model=...]`
- 节点属性：`handler`、`model`、`tools`、`converge`、`planner_model` 等
- 边：`A -> B`
- 形状到 Handler 的隐式映射，例如 `Msquare` 会映射到 `Noop`

### 12.1.2 手写 parser，而不是第三方 DOT 库

入口是 `../octos/crates/octos-pipeline/src/parser.rs:21-23` 的 `parse_dot()`，真实工作发生在 `DotParser::parse()`（`../octos/crates/octos-pipeline/src/parser.rs`）。这是一个手写 parser，不依赖外部 DOT 解析库。

当前实现比“只支持一小撮语法”的简化说法要丰富一些：

- `digraph` 名称是可选的；如果模型生成了 `digraph { ... }`，parser 会把图 ID 设成 `"pipeline"`（`../octos/crates/octos-pipeline/src/parser.rs`）
- 支持图级属性 `graph [key=value]`，目前会落到 `label` 和 `default_model`（`../octos/crates/octos-pipeline/src/parser.rs`）
- 支持 `subgraph name { ... }`，并把子图中的节点归档到 `PipelineGraph.subgraphs`（`../octos/crates/octos-pipeline/src/parser.rs`、`../octos/crates/octos-pipeline/src/graph.rs:21-24`）
- 支持边链式写法 `a -> b -> c`，属性会应用到链上的每一条边（`../octos/crates/octos-pipeline/src/parser.rs`）
- 支持 `//`、`/* */`，以及额外的 `#` 行注释；后者明显是在为 LLM 生成的 DOT 做容错（`../octos/crates/octos-pipeline/src/parser.rs`）
- 如果边引用了未显式声明的节点，parser 会自动补出默认节点定义（`../octos/crates/octos-pipeline/src/parser.rs`）

这一层的结果不是“松散的 JSON 树”，而是带语义的 `PipelineGraph`。其核心结构在 `../octos/crates/octos-pipeline/src/graph.rs:10-24` 和 `../octos/crates/octos-pipeline/src/graph.rs:91-140`：

- `PipelineGraph` 有 `id`、`label`、`default_model`、`nodes`、`edges`、`subgraphs`
- `PipelineNode` 除了 `prompt` 和 `handler`，还包含 `model`、`context_window`、`max_output_tokens`、`tools`、`goal_gate`、`max_retries`、`timeout_secs`、`suggested_next`、`converge`、`worker_prompt`、`planner_model`、`max_tasks`、`deadline_secs`、`deadline_action` 和 `checkpoints`

### 12.1.3 属性到节点语义的映射

节点构建发生在 `build_node()`（`../octos/crates/octos-pipeline/src/parser.rs:527-576`）。这里有几个实现细节决定了 DOT 的“作者体验”：

- Handler 解析顺序是：显式 `handler=` 优先，其次是 `shape=` 到 Handler 的映射，最后默认 `Codergen`（`../octos/crates/octos-pipeline/src/parser.rs:527-533`、`../octos/crates/octos-pipeline/src/graph.rs:204-230`）
- `tools="a,b,c"` 会被拆成字符串列表；如果用户写了 `tools=""`，解析结果会是一个只含空字符串的列表，后面执行器会把它当成“显式禁用所有工具”处理（`../octos/crates/octos-pipeline/src/parser.rs:535-538`、`../octos/crates/octos-pipeline/src/handler.rs`）
- `timeout_secs` 不只接受整数秒，也接受 `900s`、`15m`、`2h` 这类后缀写法（`../octos/crates/octos-pipeline/src/parser.rs`）
- `goal_gate` 允许用 `true/false/yes/no/1/0` 表达（`../octos/crates/octos-pipeline/src/parser.rs:520-524`）
- `deadline_secs` 支持 `ms/s/m/h` 和小数秒；`deadline_action` 支持 `abort`、`skip`、`escalate`、`retry:N` / `retry(N)`（`../octos/crates/octos-pipeline/src/parser.rs:540-545`、`../octos/crates/octos-pipeline/src/parser.rs:579-627`）
- `checkpoint="true"` 或 `checkpoint="name1,name2"` 会解析成 `MissionCheckpoint` 声明（`../octos/crates/octos-pipeline/src/parser.rs:629-667`）

这意味着 DOT 在 octos 里不是“纯拓扑描述”，而是一个轻量的工作流 DSL。

---

## 12.2 六种节点语义，而不是五种

`HandlerKind` 的真实枚举在 `../octos/crates/octos-pipeline/src/graph.rs:184-201`。当前源码是 6 种，不是 5 种：

| 类型 | 运行时落点 | 关键属性 | 作用 |
|------|-----------|---------|------|
| `Codergen` | `../octos/crates/octos-pipeline/src/handler.rs:186-758` | `prompt` `model` `tools` `context_window` `max_output_tokens` | 派生完整子 Agent |
| `Shell` | `../octos/crates/octos-pipeline/src/handler.rs:760-834` | `prompt` `timeout_secs` | 执行 shell 命令 |
| `Gate` | `../octos/crates/octos-pipeline/src/handler.rs:836-949` | `prompt` | 计算条件，不做人机等待 |
| `Noop` | `../octos/crates/octos-pipeline/src/handler.rs:951-965` | 无 | 透传输入 |
| `Parallel` | `../octos/crates/octos-pipeline/src/executor.rs:1439-1684` | `converge` | 对已有下游节点做静态 fan-out |
| `DynamicParallel` | `../octos/crates/octos-pipeline/src/executor.rs:1686-2033` | `prompt` `worker_prompt` `planner_model` `max_tasks` `converge` | 先规划任务，再动态 fan-out |

还有一个容易忽略但很重要的事实：并不是 6 种都对应一个 `Handler` 实现。真正实现了 `Handler` trait 的只有 `Codergen`、`Shell`、`Gate`、`Noop`（`../octos/crates/octos-pipeline/src/handler.rs`）。`Parallel` 和 `DynamicParallel` 不是独立 handler 类型，而是 `PipelineExecutor::execute_graph()` 里的专门分支（`../octos/crates/octos-pipeline/src/executor.rs:1439-2033`）。

### 12.2.1 Codergen：节点就是一个子 Agent

`CodergenHandler` 会为节点创建一个完整的 `octos_agent::Agent`，而不是做一次简化版 LLM 调用（`../octos/crates/octos-pipeline/src/handler.rs:186-758`）。这意味着节点天然继承了主 Agent 的很多能力：工具调用、循环式执行、token 统计、文件修改回传、进度事件上报。

它的关键行为有几层：

1. **Provider 解析。** 如果节点声明了 `model`，并且执行器配置了 `ProviderRouter`，handler 会走 `router.resolve()`，然后再包一层 capability-compatible fallback provider（`../octos/crates/octos-pipeline/src/handler.rs`）。这不是单纯的“model name -> provider”映射，而是带回退链的解析。
2. **上下文窗口覆盖。** `context_window` 会包装成 `ContextWindowOverride`（`../octos/crates/octos-pipeline/src/handler.rs`）。
3. **工具注册。** 节点初始工具集来自 `ToolRegistry::with_builtins()`，然后应用一次性缓存的 plugin registration，避免每个节点重复做插件 SHA 校验和可执行文件读取（`../octos/crates/octos-pipeline/src/handler.rs:21-124`）。
4. **工具策略。** 节点自己的 `tools=` 决定 allowlist；但即便允许了很多工具，handler 仍会额外 deny `spawn`、`run_pipeline`、`send_file`、`message`，避免子节点递归失控；随后还会 `clear_spawn_only()`，让原本 spawn_only 的插件工具在 pipeline worker 内同步执行（`../octos/crates/octos-pipeline/src/handler.rs:520-545`）。
5. **父会话资源继承。** 如果 `run_pipeline` 从 session actor 内触发，节点 worker 会继承父会话的 `FileStateCache`、`SubAgentOutputRouter`、`AgentSummaryGenerator`、`CostAccountant` 和 parent session key（`../octos/crates/octos-pipeline/src/host_context.rs:29-84`、`../octos/crates/octos-pipeline/src/handler.rs:662-683`）。
6. **系统提示词与任务输入分离。** 执行器会先把 `{input}` 从 `prompt` 中移除，只保留角色/约束类说明；真正的前驱输出通过 `TaskKind::Code.instruction` 传给子 Agent（`../octos/crates/octos-pipeline/src/executor.rs:2061-2075`、`../octos/crates/octos-pipeline/src/handler.rs:708-717`）。

一个和旧稿差异很大的点是：当前节点并没有 `max_iterations` 这种 DOT 属性。`CodergenHandler` 内部把 `AgentConfig.max_iterations` 固定成 30（`../octos/crates/octos-pipeline/src/handler.rs:620-633`）。真正可调的是：

- `timeout_secs`
- `max_output_tokens`
- `context_window`
- `model`
- `tools`
- `max_retries`

另外，`max_output_tokens` 的默认行为也不是“全局 4096”。如果节点没写这个属性，handler 会退回到 provider 自身的最大输出能力（`../octos/crates/octos-pipeline/src/handler.rs:613-624`）。这对长报告生成很关键。

### 12.2.2 Shell：最简单，但语义很清楚

`ShellHandler` 是最容易完整理解的一种实现（`../octos/crates/octos-pipeline/src/handler.rs:760-834`）：

- 命令来源是 `node.prompt`，没有就退回 `ctx.input`
- 非 Windows 下执行 `sh -c`，Windows 下执行 `cmd /C`
- 默认超时 300 秒，可由 `timeout_secs` 覆盖
- 非零退出码映射成 `OutcomeStatus::Fail`
- 进程启动失败或超时映射成 `OutcomeStatus::Error`

这个区分很重要，因为执行器只会对 `Error` 做重试（`../octos/crates/octos-pipeline/src/executor.rs:2473-2495`）。也就是说：

- “测试跑了但失败”是业务失败，不重试
- “命令根本没起来”或“超时”才是系统错误，可重试

### 12.2.3 Gate：当前是条件节点，不是人工审批节点

这是 `Ch12` 里最容易写错的一块。

当前执行器注册的是 `GateHandler`（`../octos/crates/octos-pipeline/src/executor.rs:1299-1319`），而 `GateHandler` 的真实语义是：

- 把 `node.prompt` 当成条件表达式
- 对直接前驱的 `NodeOutcome` 求值；单前驱保持原始状态，多前驱按 `Error > Fail > Skipped > Pass` 聚合
- 返回 `Pass` 或 `Fail`
- `content` 直接透传，不发起人机交互（`../octos/crates/octos-pipeline/src/handler.rs:836-949`）

如果 `prompt` 为空，它默认把条件视为 `"true"`，于是变成一个 pass-through gate（`../octos/crates/octos-pipeline/src/handler.rs:907-949`）。

`human_gate.rs` 的确存在，而且提供了 `HumanInputProvider`、`ChannelInputProvider`、`HumanRequest`、`HumanResponse`，默认超时 5 分钟（`../octos/crates/octos-pipeline/src/human_gate.rs:14-140`）。但我对照当前源码后可以明确说：这些抽象**没有接进** `PipelineExecutor::build_handlers()` 或 `execute_graph()` 主路径（`../octos/crates/octos-pipeline/src/executor.rs:1299-1319`）。所以“Gate = 人工审批节点”已经不是当前实现的准确说法。

更准确的表述应该是：

- `GateHandler` 是已接线的条件节点
- `human_gate.rs` 是 crate 已提供、但尚未接入默认执行器的人机输入抽象

### 12.2.4 Parallel：静态 fan-out，执行真实下游节点

`Parallel` 是当前章节里最值得补深度的一种类型，因为它不是“动态生成 worker”，而是把图里已经存在的下游节点并发跑掉（`../octos/crates/octos-pipeline/src/executor.rs:1439-1684`）。

它的执行过程是：

1. 收集当前节点所有 outgoing edges 的 target，作为并发目标（`../octos/crates/octos-pipeline/src/executor.rs:1445-1450`）
2. 要求当前节点必须声明 `converge`，否则验证阶段直接报错（`../octos/crates/octos-pipeline/src/validate.rs`）
3. 在派发前检查 pipeline 生命周期级 fan-out 总量上限，默认 `MAX_PIPELINE_FANOUT_TOTAL = 500`；超过上限直接拒绝整批派发，避免半批 worker 已经启动（`../octos/crates/octos-pipeline/src/executor.rs:55-68`、`../octos/crates/octos-pipeline/src/executor.rs:1488-1509`）
4. 为每个目标节点克隆 `PipelineNode`，做变量替换，并在未显式声明模型时填入 `graph.default_model`（`../octos/crates/octos-pipeline/src/executor.rs:1536-1549`）
5. 为每个 LLM-call 分支预留成本预算，然后查它自己的 handler 并并发执行（`../octos/crates/octos-pipeline/src/executor.rs:1551-1560`、`../octos/crates/octos-pipeline/src/executor.rs:1588-1605`）
6. 用 `process_worker_results()` 合并内容、token、summary 和 node outcome（`../octos/crates/octos-pipeline/src/executor.rs:556-642`、`../octos/crates/octos-pipeline/src/executor.rs:1621-1634`）
7. 把汇总后的文本写回“当前 parallel 节点”的 `completed` 结果，再跳到 `converge` 节点（`../octos/crates/octos-pipeline/src/executor.rs:1647-1683`）

这里有两个实现细节值得记住：

- `Parallel` 的并发度受 `ExecutorConfig.max_parallel_workers` 限制，靠 `tokio::sync::Semaphore` 实现；`run_pipeline` 工具默认配置为 8（`../octos/crates/octos-pipeline/src/executor.rs:1513-1518`、`../octos/crates/octos-pipeline/src/tool.rs:325-347`）
- 执行器会用 `parallel_executed` 记住那些已经在 fan-out 阶段跑过的真实图节点，后续顺序遍历遇到它们时只选边，不重复执行（`../octos/crates/octos-pipeline/src/executor.rs:1341-1342`、`../octos/crates/octos-pipeline/src/executor.rs:1374-1393`、`../octos/crates/octos-pipeline/src/executor.rs:1631-1634`）

此外，结果合并并不只是简单字符串拼接。`process_worker_results()` 之后还会调用 `resolve_search_result_files()`，自动扫描 worker 输出里提到的研究目录，把 `_search_results.md` 的内容内联进 merge 结果；如果输出里没有路径，还会扫描工作目录下最近的 `research/` 子目录（`../octos/crates/octos-pipeline/src/executor.rs:636-758`）。这说明当前 Pipeline 已经针对“研究型 fan-out -> 汇总型 converge”做了专门优化。

### 12.2.5 DynamicParallel：先规划，再合成 worker 节点

`DynamicParallel` 和 `Parallel` 的根本区别是：它不直接跑现成的图节点，而是先让 LLM 规划出任务列表，再为每个任务合成一个临时 `PipelineNode`（`../octos/crates/octos-pipeline/src/executor.rs:1786-1844`）。

主流程在 `../octos/crates/octos-pipeline/src/executor.rs:1686-2033`：

1. 解析 `planner_model -> node.model -> graph.default_model` 的 planner provider 选择链（`../octos/crates/octos-pipeline/src/executor.rs:1721-1729`）
2. 用 `node.prompt` 作为规划提示词；若为空则退回内置 planner prompt（`../octos/crates/octos-pipeline/src/executor.rs:1731-1736`）
3. 期望模型返回纯 JSON 数组；若少于 2 个任务或解析失败，则退回 `fallback_tasks()`（`../octos/crates/octos-pipeline/src/executor.rs:404-553`、`../octos/crates/octos-pipeline/src/executor.rs:1750-1778`）
4. 把 `worker_prompt` 里的 `{task}` 替换为具体任务说明，生成一批 synthetic `Codergen` 节点（`../octos/crates/octos-pipeline/src/executor.rs:1786-1844`）
5. 派发前同样检查 pipeline 生命周期级 fan-out 总量上限，并为每个 synthetic worker 预留成本预算（`../octos/crates/octos-pipeline/src/executor.rs:1873-1895`、`../octos/crates/octos-pipeline/src/executor.rs:1906-1924`）
6. 并发执行这些 synthetic 节点，合并结果后跳到 `converge` 节点（`../octos/crates/octos-pipeline/src/executor.rs:1948-2033`）

它还有两个旧稿完全没写到的行为：

- `node.model` 可以写成逗号分隔的 model pool，例如 `"cheap,strong,cheap"`；执行器会把 worker 轮询分配到不同模型上（`../octos/crates/octos-pipeline/src/executor.rs:1791-1816`）
- 当前 `DynamicParallel` **没有**像 `Parallel` 那样再套一层 semaphore；单次 fan-out 主要依赖 `max_tasks`，默认值是 8；跨整个 pipeline 运行还受 `MAX_PIPELINE_FANOUT_TOTAL = 500` 保护（`../octos/crates/octos-pipeline/src/executor.rs:1719`、`../octos/crates/octos-pipeline/src/executor.rs:1873-1895`）

所以如果你问“当前实现里哪种并发更受控”，答案其实是：静态 `Parallel` 的并发度控制更硬，`DynamicParallel` 更依赖 planner 输出和 `max_tasks` 自我约束。

### 12.2.6 Noop：占位，但也很实用

`NoopHandler` 就是把 `ctx.input` 原样返回（`../octos/crates/octos-pipeline/src/handler.rs:951-965`）。它有两个常见用途：

- 作为 start / finish 这类结构节点
- 作为某些条件分支的汇合点或透传点

---

## 12.3 执行引擎不是“拓扑排序器”，而是带路由的图遍历器

```mermaid
flowchart TD
    DOT["DOT / pipeline name / file path"] --> Parse["parse_dot()"]
    Parse --> Validate["validate()"]
    Validate --> Start["find_start_node()"]
    Start --> Loop["execute_graph() loop"]

    Loop --> Kind{node.handler}
    Kind -->|Parallel| PFan["并发执行真实下游节点"]
    Kind -->|DynamicParallel| DPlan["LLM 规划任务"]
    Kind -->|其他| Normal["Handler::execute()"]

    PFan --> Merge["合并结果并跳到 converge"]
    DPlan --> Workers["合成 worker 节点并 join_all"]
    Workers --> Merge
    Normal --> Select["select_next_edge()"]
    Merge --> Select
    Select -->|有后继| Loop
    Select -->|无后继 / goal_gate 成功 / Error| Done["PipelineResult"]
```

**图 12-1：当前 `PipelineExecutor` 的真实主路径。** 它不是先做一次全图拓扑排序，再机械执行所有节点；而是从 start node 出发，在循环里按节点类型分流，并在每一步重新决定下一条边。

### 12.3.1 `run()` 的实际阶段

`PipelineExecutor::run()` 在 `../octos/crates/octos-pipeline/src/executor.rs:801-979`，可以概括成七步：

1. `parse_dot()` 解析 DOT
2. `validate()` 跑 lint 规则
3. `build_handlers()` 构建常规 handler registry
4. `find_start_node()` 决定入口节点
5. 如有 `CostAccountant`，先打开 pipeline 级预算 reservation
6. `execute_graph()` 进入主循环
7. 若 pipeline 成功，跑 terminal validators；最终成功才提交 pipeline 级成本归因，否则 reservation 自动退款

这里最重要的纠偏是：第四步之后不是“拓扑遍历整个 DAG”，而是 `current_node_id` 驱动的增量遍历（`../octos/crates/octos-pipeline/src/executor.rs:1322-2470`）。这也是为什么 `suggested_next`、条件边、label 匹配这些运行时路由策略都能生效。

### 12.3.2 验证规则和 start node 选择

验证器在 `../octos/crates/octos-pipeline/src/validate.rs`，当前不只是“检查一下条件能不能 parse”。

比较重要的几条：

- Rule 1：必须能找到 start node（`start` 节点，或唯一一个无入边节点）（`../octos/crates/octos-pipeline/src/validate.rs`）
- Rule 2：不可达节点只是 warning，不是 error（`../octos/crates/octos-pipeline/src/validate.rs`）
- Rule 6：边条件必须能被 condition parser 解析（`../octos/crates/octos-pipeline/src/validate.rs`）
- Rule 13 / 14：`parallel` 和 `dynamic_parallel` 都必须声明有效的 `converge`（`../octos/crates/octos-pipeline/src/validate.rs`）
- 图中不能有环；环检测发生在 validate 阶段，不等执行时才爆炸（`../octos/crates/octos-pipeline/src/graph.rs:26-88`）

这意味着 octos-pipeline 当前仍然要求 DAG，但执行方式不是“静态 DAG 调度器”，而是“受 DAG 约束的动态图遍历器”。

### 12.3.3 条件语言和边选择顺序

条件表达式的 grammar 写在 `../octos/crates/octos-pipeline/src/condition.rs`。当前运行时真正支持的核心写法是：

- `outcome.status == "pass"`
- `outcome.status != "fail"`
- `outcome.contains("keyword")`
- `!expr`、`expr && expr`、`expr || expr`

例如：

```dot
test -> deploy   [condition="outcome.status == \"pass\""]
test -> rollback [condition="outcome.status == \"fail\""]
report -> refine [condition="outcome.contains(\"missing data\")"]
```

旧稿里那种 `success` / `failure` 简写已经不符合当前 parser。

更值得写清楚的是，执行器选边不是“第一个命中就走”，而是一个 5 步算法（`../octos/crates/octos-pipeline/src/executor.rs:2597-2657`）：

1. 先评估所有带条件的边
2. 如果有多个条件命中，按 `weight` 选最高权重
3. 若无条件命中，检查节点的 `suggested_next`
4. 再看 edge label 是否出现在 outcome content 里
5. 最后才在无条件边里按权重选；如果还没有，就退回目标名最小的边

还有一个微妙但重要的实现现状：`condition.rs` 的 grammar 虽然支持 `context.key == "value"`，但 `select_next_edge()` 和 `GateHandler` 走的是 `evaluate()`，不是 `evaluate_with_context()`（`../octos/crates/octos-pipeline/src/condition.rs`、`../octos/crates/octos-pipeline/src/handler.rs:907-914`、`../octos/crates/octos-pipeline/src/executor.rs:2611-2619`）。也就是说，`context.*` 目前是“语法已定义、主路径未喂值”的状态；真正稳定可用的还是 `outcome.*` 相关条件。

### 12.3.4 进度、统计和终止条件

`PipelineStatusBridge` 定义在 `../octos/crates/octos-pipeline/src/executor.rs:267-307`，桥接了两类外部可见状态：

- `status_words`：当前节点或 fan-out worker 的状态文案
- `token_tracker`：所有子 Agent 的 token 聚合

当 `CodergenHandler` 内部子 Agent 产出 `ProgressEvent` 时，`PipelineNodeReporter` 会把事件重新转成 `run_pipeline` 的进度消息，并且现在也会转发 per-node cost update（`../octos/crates/octos-pipeline/src/handler.rs:126-184`）。所以前端看到的 pipeline 进度，不只是“现在在第几个节点”，而是能继续细到“某个 worker 正在调用哪个工具、花了多少 token / 成本”。

执行终止有三种常见路径：

- 当前节点没有 outgoing edges（`../octos/crates/octos-pipeline/src/executor.rs:2446-2467`）
- 某个 `goal_gate=true` 节点成功，提前结束 pipeline（`../octos/crates/octos-pipeline/src/executor.rs:2391-2417`）
- 某节点返回 `OutcomeStatus::Error`，整个 pipeline 直接停止（`../octos/crates/octos-pipeline/src/executor.rs:2420-2433`）
- 某节点触发 `deadline_secs`，按 `deadline_action` 选择 abort / skip / retry / escalate（`../octos/crates/octos-pipeline/src/executor.rs:2498-2595`）

最终返回的是 `PipelineResult`（`../octos/crates/octos-pipeline/src/executor.rs:248-265`），其中除了 `output` / `success` / `token_usage`，还有：

- `node_summaries`：每个节点的 `model`、耗时、token 和 success
- `files_modified`：所有节点写出的文件，去重后汇总
- `node_costs`：每个 LLM-call 节点的预算预留、实际估算成本、token 和是否绑定到账本

### 12.3.5 当前生效的模型选择路径

当前主路径真正生效的模型选择机制有两层：

- 图级默认：`graph [default_model="cheap"]`
- 节点覆盖：`node [model="strong"]`

这两层分别在 parser 中落到 `PipelineGraph.default_model` 和 `PipelineNode.model`（`../octos/crates/octos-pipeline/src/parser.rs`、`../octos/crates/octos-pipeline/src/parser.rs:553-556`），然后在执行时由 `execute_graph()` 和 `CodergenHandler` 联合应用（`../octos/crates/octos-pipeline/src/executor.rs:2077-2080`、`../octos/crates/octos-pipeline/src/handler.rs`）。

`ModelStylesheet` 模块本身是存在的，支持 `*` / `handler:codergen` / `node:critical_analysis` 这类 selector（`../octos/crates/octos-pipeline/src/stylesheet.rs:28-104`）。但我对照当前源码后，没有找到它被 `PipelineExecutor`、`RunPipelineTool` 或 `PipelineDiscovery` 调用的路径。换句话说：

- `ModelStylesheet` 是 crate 已导出的能力
- 当前默认执行路径用的仍然是 `default_model + node.model`

如果书里把 ModelStylesheet 写成主路径，会高估它在当前版本里的实际地位。

### 12.3.6 父会话继承、成本账本与 workspace policy

当前 `run_pipeline` 已经不是“在 pipeline 内部重新开一套孤立资源”。工具入口会从 `TOOL_CTX` 快照 `PipelineHostContext`，把父 session 的共享资源传给执行器（`../octos/crates/octos-pipeline/src/tool.rs:314-347`、`../octos/crates/octos-pipeline/src/host_context.rs:29-84`）：

- `FileStateCache`：节点 worker 复用父会话的文件状态缓存，避免每个节点重新建立一套视图
- `SubAgentOutputRouter` / `AgentSummaryGenerator`：节点里再触发后台子 Agent 时，输出和摘要仍走父会话的路由
- `TaskSupervisor`：顺序节点会注册成 `pipeline:<node_id>` 子任务，并在完成、失败或跳过时更新状态（`../octos/crates/octos-pipeline/src/executor.rs:1195-1221`、`../octos/crates/octos-pipeline/src/executor.rs:2120-2133`、`../octos/crates/octos-pipeline/src/executor.rs:2318-2343`）
- `CostAccountant`：pipeline 级 reservation 在运行开始打开，成功后以累计 token 提交；节点级 reservation 只用于派发前预算闸门，节点完成后形成 `node_costs` 给 UI / SSE 使用（`../octos/crates/octos-pipeline/src/executor.rs:883-891`、`../octos/crates/octos-pipeline/src/executor.rs:981-1046`、`../octos/crates/octos-pipeline/src/executor.rs:1223-1258`、`../octos/crates/octos-pipeline/src/executor.rs:2290-2316`）

同时，`RunPipelineTool` 会从工作目录读取 workspace policy，自动构造 `PipelineContext`。这让 pipeline 节点继承 compaction policy，terminal 阶段跑 workspace validators；如果配置了 `validators_by_node`，单个节点完成后也会跑 per-node validators（`../octos/crates/octos-pipeline/src/tool.rs:77-125`、`../octos/crates/octos-pipeline/src/executor.rs:1075-1193`、`../octos/crates/octos-pipeline/src/executor.rs:2270-2288`）。

这部分和第 8 章的上下文压缩、第 11 章的 `TaskSupervisor` 是同一条工程线：pipeline 不再只是“多节点 DAG”，而是被纳入父 session 的观测、预算和 workspace contract 边界。

### 12.3.7 `human_gate`、`checkpoint`、`run_dir` 的真实位置

这一章还有三个容易被写成“默认能力”的模块，但当前更准确的定位需要拆成两层：crate 能力、执行器可选接线、默认工具路径。

- `human_gate.rs`：提供 channel-based human input 抽象，默认超时 5 分钟，但未接入 `PipelineExecutor`（`../octos/crates/octos-pipeline/src/human_gate.rs:14-140`）
- `checkpoint.rs`：提供 `CheckpointStore` / `FileSystemCheckpointStore`，执行器现在有 `ExecutorConfig.checkpoint_store` 可选字段；运行开始会从 store 构造 resume skip set，节点成功后会持久化 DOT 中声明的 `checkpoint`（`../octos/crates/octos-pipeline/src/checkpoint.rs:127-224`、`../octos/crates/octos-pipeline/src/executor.rs:195-216`、`../octos/crates/octos-pipeline/src/executor.rs:330-336`、`../octos/crates/octos-pipeline/src/executor.rs:2363-2389`）
- `run_dir.rs`：提供 `RunDir`、`NodeStatus`、`PipelineRunSummary`，约定运行目录是 `{working_dir}/.octos/runs/{run_id}/...`，但默认 `RunPipelineTool` 仍没有把它接成自动 run directory（`../octos/crates/octos-pipeline/src/run_dir.rs:17-114`、`../octos/crates/octos-pipeline/src/tool.rs:325-347`）

所以现在不能再说 checkpoint 完全是“未接线模块”：它已经是执行器的可选能力，只是 `RunPipelineTool` 默认把 `checkpoint_store` 设为 `None`。准确结论是：默认 `run_pipeline` 不自动做人机审批、不自动写 run_dir，也不默认启用 checkpoint；但自定义 executor 配置可以启用 checkpoint resume / persist。

### 12.3.8 `run_pipeline` 工具如何把 pipeline 暴露给 Agent

对最终用户来说，最常见的入口不是直接 new `PipelineExecutor`，而是 `RunPipelineTool`（`../octos/crates/octos-pipeline/src/tool.rs:19-462`）。

它有几层很实际的工程化包装：

- 先尝试把输入当成 inline DOT；若 parse 失败，再尝试把图名解析成预置 pipeline（`../octos/crates/octos-pipeline/src/tool.rs:152-211`）
- 会自动修正常见的 LLM DOT 错误，比如 `digraph{`、缺图名、代码围栏包裹（`../octos/crates/octos-pipeline/src/tool.rs:481-514`）
- 可按名称、路径或内联 DOT 解析 pipeline；搜索路径包括项目级 `.octos/pipelines`、用户级 `data_dir/pipelines`、`data_dir/skills`，额外还能挂 `octos_home/skills`（`../octos/crates/octos-pipeline/src/discovery.rs:14-114`、`../octos/crates/octos-pipeline/src/tool.rs:127-132`）
- 对整个 pipeline 施加 60-1800 秒的总超时钳制；结束后会置位共享 shutdown flag，通知所有 worker 停止（`../octos/crates/octos-pipeline/src/tool.rs:349-368`）
- 如果 pipeline 没产出 markdown 文件但有文本输出，工具会合成一个临时 `.md` 报告，保证 `spawn_only` 交付路径有可发送文件（`../octos/crates/octos-pipeline/src/tool.rs:386-443`）
- `node_costs` 会投射到 `ToolResult.structured_metadata`，供 session actor 把 per-node cost 带回 UI/API completion metadata（`../octos/crates/octos-pipeline/src/tool.rs:444-479`）

这里还有一个值得写进书里的“实现与提示词分离”现象：`run_pipeline` 的 `input_schema()` 会明确告诉模型“不要显式写 `model=`，系统会自动选择模型”（`../octos/crates/octos-pipeline/src/tool.rs:249-285`），但运行时引擎本身依然支持 `default_model` / `node.model`。也就是说：

- 这是对 LLM authoring 的建议
- 不是底层引擎能力被移除了

---

> ### 工程决策侧栏：为什么选 DOT 而不是 YAML/JSON
>
> **YAML（例如 GitHub Actions）**
>
> 优势：人类熟悉，生态成熟。
> 劣势：图结构不是一等语义，`needs:` 这类依赖写法在分支和汇合场景下会越来越别扭。
>
> **JSON（例如 Step Functions）**
>
> 优势：结构化强，schema 友好。
> 劣势：对人类作者不友好，特别是当节点属性和分支条件越来越多时。
>
> **DOT（octos 的选择）**
>
> 优势：
> - 节点和边本身就是 DOT 的原生概念
> - 同一份定义可直接被 Graphviz 渲染
> - `handler` / `model` / `tools` / `converge` 这类属性自然落在节点上
> - 对 LLM 来说，生成一张图往往比生成层层嵌套的 YAML/JSON 更稳定
>
> 代价：
> - 需要自己实现 parser 和验证器
> - DOT 不是大多数工程团队的日常配置语言，学习成本略高

---

## 12.4 本章回顾

1. `octos-pipeline` 当前不是“5 种 handler”，而是 6 种 `HandlerKind`；其中 `Parallel` 和 `DynamicParallel` 其实是执行器分支，不是独立 `Handler` 实现。
2. `Gate` 在当前版本里是条件节点，不是默认接线的人机审批节点；`human_gate.rs` 只是已存在但尚未接入执行主路径的抽象。
3. `PipelineHostContext` 让 pipeline 节点继承父 session 的缓存、输出路由、任务监督和成本账本；这让 pipeline 成为 session runtime 的一部分，而不是旁路执行器。
4. 真正生效的模型选择路径是 `graph.default_model + node.model`；`ModelStylesheet` 仍不是默认主路径。`CheckpointStore` 已经是执行器可选接线，但默认 `RunPipelineTool` 仍未启用；`RunDir` 仍是相邻模块。

---

## 延伸阅读

- **Graphviz DOT Language**：https://graphviz.org/doc/info/lang.html
- **DAG 调度**：可以对照 Airflow / Prefect 看“静态 DAG 调度器”和 octos 这种“带运行时路由的图遍历器”之间的差异

## 思考题

1. `condition.rs` 已经支持 `context.*` grammar，但执行器当前没有把上下文 map 接进去。你会把这层语义接到 `select_next_edge()`，还是保留 outcome-only 的简单模型？
2. `DynamicParallel` 当前没有额外 semaphore，单次 fan-out 主要靠 `max_tasks`，全局靠 `MAX_PIPELINE_FANOUT_TOTAL`。对于高成本 provider，还是否应该给 dynamic workers 加一个和 `Parallel` 相同的 semaphore？

---

> **版本演化说明**
> 本章分析基于当前 `../octos` main 分支中 `crates/octos-pipeline/src/` 的实现。书中凡是涉及 `Gate`、`ModelStylesheet`、`CheckpointStore`、`RunDir`、`PipelineHostContext`、成本账本和 workspace policy 的地方，都应区分“模块存在”“执行器可选接线”和“默认 `RunPipelineTool` 路径”，而不是仅凭模块是否存在来下结论。
