# Reasonix 深度架构解析：一个 DeepSeek 原生编码代理的工程之道

> 本文基于 Reasonix v0.51.0 代码库，覆盖 50+ 核心源文件和 260+ 测试文件，万字详解项目的架构设计、核心循环、事件系统、修复管道、上下文管理、TUI 和工程实践。

---

## 一、引言：从"能用"到"用得起的编码代理"

2025-2026 年，AI 编码代理（Coding Agent）市场经历了爆炸式增长。从 GitHub Copilot 到 Cursor，从 Claude Code 到各种开源替代品，开发者拥有了前所未有的选择。然而，绝大多数 Agent 方案都有同一个隐痛——**成本**。

当一轮对话消耗几百万 Token，日账单轻松突破数十美元时，"随手用"就变成了"谨慎用"。这个痛点催生了 Reasonix 项目的核心主张：**做一个便宜到可以一直开着的编码代理**。

Reasonix 的答案是拥抱 DeepSeek 的按量计价和前缀缓存（Prefix Caching）机制——该机制下，缓存命中的 Token 成本仅为未命中的 1/50 到 1/120。Reasonix 的所有架构决策都围绕一个核心目标展开：**让每一轮对话的字节前缀与上一轮完全一致**，从而最大化缓存命中率。

这个定位决定了 Reasonix 是一条**深度绑定 DeepSeek 生态的专线**——它不做通用 LLM 网关，不追求模型无关性，而是把每一层都调教到适配 DeepSeek API 的行为特性。正如其 README 所言：**"Opinionated, not general."**

## 二、项目总览与目录哲学

```
src/
├── core/          # 事件内核：Event 类型 + 纯函数归约器 + 压缩逻辑
├── loop.ts        # CacheFirstLoop 主循环（调度核心）
├── loop/          # 子模块：分派、流式、修复、收缩、思考模式、强制摘要
├── context-manager.ts  # 上下文管理与自动折叠
├── tools.ts       # ToolRegistry + 工具定义
├── tools/         # 各工具实现（文件系统、Shell、Web、MCP、计划、选择、待办...）
├── repair/        # 工具调用修复管道：scavenge → flatten → truncation → storm
├── mcp/           # 自研 MCP 客户端（stdio/SSE/Streamable HTTP）
├── client.ts      # DeepSeek API 客户端（手写，非 OpenAI SDK）
├── memory/        # 项目/会话/用户/运行时四层记忆
├── adapters/      # 端口 → 适配器（JSONL 事件源/汇）
├── ports/         # 端口接口：EventSink、MemoryStore、ToolHost、ModelClient...
├── index/         # 语义向量索引（本地 Ollama / OpenAI 嵌入）
├── code/          # SEARCH/REPLACE 编辑块引擎
├── hooks.ts       # 生命周期钩子系统
├── transcript/    # 转录记录 + 离线回放 + 差异比较
├── telemetry/     # 定价表 + 使用统计 + 子代理蒸馏遥测
├── server/        # Web 仪表盘 HTTP 服务器（32 个 API 端点）
├── cli/           # Commander.js CLI 入口 + Ink React TUI
├── frame/         # 预留：帧编译器（未来 ANSI 表格渲染）
├── index.ts       # 库导出入口
└── retry.ts       # HTTP 重试逻辑
```

这个目录结构的组织哲学非常清晰：**按职责垂直切分，每层独立演进**。Core 层定义了纯数据模型；Loop 层是编排逻辑；Tools 层是行为实现；Repair 层弥补模型缺陷；MCP 层打通第三方工具生态。每一层都可以独立测试、替换、升级。

## 三、四大架构支柱

根据 `docs/ARCHITECTURE.md`，Reasonix 的架构由四根支柱支撑：

| 支柱 | 解决的问题 | 核心策略 |
|------|-----------|---------|
| **缓存优先循环** | DeepSeek 前缀缓存只在字节精确匹配时激活 | 不可变前缀 + 仅追加日志 + 易失暂存区分 |
| **工具调用修复** | DeepSeek 在工具调用上有 4 种已知失效模式 | scavenge → flatten → truncation → storm |
| **轻量事件溯源** | 非确定性重放、状态难审计 | 纯函数归约器、仅追加日志、可重现转录 |
| **上下文自动折叠** | 长会话中 token 增长失控 | push-based 折叠决策、阶梯阈值、技能保留 |

下面逐一深入。

## 四、支柱一：缓存优先循环（Cache-First Loop）

### 4.1 问题本质

DeepSeek 的前缀缓存机制是一个 **字节级精确匹配** 的优化：只有当本次请求的字节前缀与上次完全一致时，缓存才生效。绝大多数 Agent 每轮都会注入新的时间戳、重新排序历史消息、或者在系统提示中插入了变化的内容——结果就是缓存命中率常常低于 20%。

### 4.2 三段式上下文分区

CacheFirstLoop 将上下文切分为三个区域，保持严格的不可变性契约：

```
┌─────────────────────────────────────────┐
│ 不可变前缀（IMMUTABLE PREFIX）           │ ← 会话期间固定不变
│   system + tool_specs + few_shots        │   缓存命中候选
├─────────────────────────────────────────┤
│ 仅追加日志（APPEND-ONLY LOG）            │ ← 单调增长
│   [assistant₁][tool₁][assistant₂]...    │   保留前轮的字节前缀
├─────────────────────────────────────────┤
│ 易失暂存（VOLATILE SCRATCH）             │ ← 每轮重置
│   R1 推理、临时计划状态                   │   从不发送到上游
└─────────────────────────────────────────┘
```

**三个不变式：**

1. **前缀锚定**：系统提示 + 工具规范在会话开始时计算一次，做 SHA-256 指纹校验，之后不再改变。
2. **仅追加**：日志消息按追加顺序序列化，永不重写。这保证了从第一轮到当前轮的字节前缀是可重现的。
3. **易失隔离**：R1 推理文本、临时计划草稿等不持久化的内容被放在 `VolatileScratch` 中，从不参与日志序列化。

### 4.3 核心循环调度

CacheFirstLoop 的 `step()` 方法——每个 AI turn 的调度——流程如下：

```
收到流式响应 → 解析 tool_calls
  → 修复调用（scavenge/flatten/storm）
    → 全部被 suppress 且是首轮 → 重写 tail + 续行（给模型一次自纠机会）
    → 全部被 suppress 且已自纠过 → forceSummary（"stuck"）
    → 无 tool_calls → 标记 done，return
  → context.decideAfterUsage(usage, model, foldedThisTurn)
    ├  kind === "fold" → compactHistory(keepRecentTokens)
    ├  kind === "exit-with-summary" → trimTrailingToolCalls() + forceSummary("context-guard")
    └  其他 → 继续
  → dispatchToolCallsChunked(修复后的 calls)
    → 循环回到 next turn
```

关键设计点：**上下文管理是 push-based 而非 ask-based**——`decideAfterUsage` 在每轮之后自动判断是否折叠，不需要模型决定。`_turnSelfCorrected` 机制让重复的 tool call 风暴有一次软着陆机会。

### 4.4 并行工具分派

传统 Agent 串行执行工具，一轮对话可能要串行调用十几次工具。Reasonix 实现了智能并行调度：

- 每个工具声明 `parallelSafe?: boolean`（默认 `false`）
- 调度器将连续的并行安全调用分组，通过 `Promise.allSettled` 并行执行
- 遇到第一个非并行安全调用时，作为串行屏障（保证读-后-写顺序）
- 无论哪个 Promise 先结算，工具结果仍按声明顺序产出和追加，模型看到的形状与串行完全一致

环境变量控制：
- `REASONIX_PARALLEL_MAX=3`（默认，硬上限 16）：分块大小
- `REASONIX_TOOL_DISPATCH=serial`：强制串行（逃生舱）

内置的并行安全工具包括：只读文件系统工具、Web 搜索/抓取、记忆查询、语义搜索、子代理衍生、后台作业查询。所有变异工具默认串行，MCP 桥接工具默认 `false`。

### 4.5 经济效果

以 τ-bench 基准测试的转录数据为例，Reasonix 实现了 **96.7% 的缓存命中率**。一次真实用户的一天使用记录（2026-05-01）：435M 输入 Token，**99.82% 缓存命中**，实际费用 ~$12，而无缓存的等价负载需要 ~$61（flash 模型）。对于 Pro 模型（缓存命中/未命中价差 120 倍），节省比例更高。

## 五、支柱二：工具调用修复管道（Tool-Call Repair）

### 5.1 DeepSeek 的四种失效模式

Reasonix 团队通过大量实测归纳了 DeepSeek 在处理工具调用时的四个系统性问题：

| 失效模式 | 现象 | 频率 |
|---------|------|------|
| **scavenge** | 工具调用 JSON 嵌在 `<think>` 推理块内，不出现在最终消息中 | 高频 |
| **schema flatten** | 嵌套超过 2 层或叶子参数超过 10 个时，模型随机丢弃参数 | 高频 |
| **truncation** | 由于 `max_tokens` 截断，JSON 在结构中间被切断 | 中频 |
| **storm** | 同一工具用相同参数重复调用（无限循环） | 低频但致命 |

### 5.2 四道修复工序

每一轮的工具调用在送达分派器之前，依次经过四道修复工序：

```
原始 model 响应
  → ① scavenge：从 reasoning_content 中正则提取嵌入的工具调用 JSON
  → ② flatten：schema 自动展平为点路径（dispatch 后复原）
  → ③ truncation：修复被截断的 JSON（平衡括号、闭合字符串、填充 null）
  → ④ storm：检测重复调用模式（默认窗口=6，阈值=3）
  → 修复后的工具调用 → dispatchToolCallsChunked
```

#### ① Scavenge（打捞）

DeepSeek 的 R1 模式有时会将工具调用结构化 JSON 输出在推理内容中而非 `tool_calls` 字段。Scavenge 工序使用正则表达式扫描 `reasoning_content`，提取嵌入的 JSON，将其合并到工具调用列表中。

#### ② Flatten（展平）

这是最巧妙的设计之一。在 `ToolRegistry.register()` 时，`analyzeSchema()` 会递归遍历 JSON Schema：

```typescript
function analyzeSchema(schema: JSONSchema): {
  leafCount: number;
  maxDepth: number;
  shouldFlatten: boolean;
}
```

如果 `leafCount > 10 || maxDepth > 2`，工具规范在传给模型之前就被自动展平为一层扁平属性，属性名使用点路径（如 `"parent.child.key"`）。当模型返回参数后，`nestArguments()` 再将其还原为嵌套结构。**这绕过了 DeepSeek API 对复杂参数的隐式丢弃，且对工具实现者完全透明。**

#### ③ Truncation（截断修复）

当模型输出因 `max_tokens` 被截断时，JSON 结构可能不完整。`repairTruncatedJson()` 使用局部修复策略：平衡括号、闭合未完成的字符串、填充 `null` 占位符。最坏情况返回 `"{}"` 作为回退。**注意：这是纯本地修复，不需要额外的 API 调用。**

#### ④ Storm Breaker（风暴拦截）

检测重复调用模式：当同一工具在窗口内（默认 6 次调用）被相同参数调用超过阈值（3 次）时，StormBreaker 会清除这些多余的调用。它只在以下条件判断：如果窗口内的条目全是只读的（如 `read_file` 重复读同一文件），则只保留一条；如果包含变异调用，则只清除只读条目。**这确保了"后写重读"不会被误判为风暴。**

### 5.3 修复报告

整个修复管道会输出 `RepairReport`，包含每道工序的决策日志。这些日志会被写入 TUI 的状态行，让用户可以看到循环在背后做了哪些修复工作——**调试透明性**。

## 六、支柱三：轻量事件溯源内核

### 6.1 为什么不是可变状态？

Reasonix 选择事件溯源（Event Sourcing）而非可变状态，原因有三：

1. **可重现性**：给定同一事件日志，归约器总是产生同一视图。这意味着离线回放（transcript/replay）和在线运行时使用同一套代码。
2. **审计友好**：每一轮的所有决策都有事件记录——工具调用、模型响应、折叠决策、错误——方便调试和成本分析。
3. **非确定性隔离**：DeepSeek 的 API 是非确定性的（同一 prompt 每次输出不同），但事件日志的记录是确定性的——这避免了"状态被污染"的问题。

### 6.2 事件类型（Event Union）

`src/core/events.ts` 定义了 40+ 种事件变体，涵盖一个会话的整个生命周期：

| 类别 | 事件 |
|------|------|
| 会话生命周期 | `session.opened`，`session.compacted`，`session.rewritten` |
| 用户交互 | `user.message`，`user.abort`，`user.slashcommand` |
| 模型交互 | `model.turn.started`，`model.delta`，`model.final` |
| 工具调用 | `tool.call`，`tool.result`，`tool.error` |
| 计划相关 | `plan.proposed`，`plan.approved`，`plan.rejected`，`plan.step_completed` |
| 系统行为 | `error`，`status`，`warning`，`steer` |

每个事件都有 `id: EventId`、`ts: string`（ISO 时间戳）、`turn: number`。

### 6.3 纯函数归约器

`src/core/reducers.ts` 是纯函数集合：

```typescript
reduceMessage(log: Event[], state: ConversationView): ConversationView
reduceBudget(log: Event[], state: BudgetView): BudgetView
reducePlan(log: Event[], state: PlanView): PlanView
reduceWorkspace(log: Event[], state: WorkspaceView): WorkspaceView
reduceCapability(log: Event[], state: CapabilityView): CapabilityView
```

每个归约器都是一个 `(state, event) → newState` 的纯函数，没有 I/O，没有副作用。最终视图通过 `reduceAll(log)` 组合生成。

### 6.4 概念类比

这套架构与 Redux 如出一辙：

| Redux | Reasonix |
|-------|----------|
| `Action` | `Event` |
| `Reducer` | `reduceMessage()` |
| `Store (getState + dispatch)` | `AppendOnlyLog + replay` |
| `Middleware` | Hooks 系统 |
| `createSlice` | 每个视图有独立的归约器 |

不同之处在于：Reasonix 没有中心 Store 对象——事件是持久化到 JSONL 文件的，归约器在需要时从文件重建视图。

## 七、支柱四：上下文自动管理与折叠

### 7.1 Push-Based 折叠策略

大多数 Agent 的上下文管理是被动的——要么让模型自己决定"是否需要总结"，要么在 Token 用尽时报错。Reasonix 采取 **push-based** 策略：`decideAfterUsage()` 在每轮工具调用之后自动判断是否需要折叠。

### 7.2 阶梯阈值

折叠策略由一组精心调校的常量驱动：

| 常量 | 值 | 含义 |
|------|-----|------|
| `HISTORY_FOLD_THRESHOLD` | 0.75 | promptToken > 75% ctx → 触发折叠 |
| `HISTORY_FOLD_TAIL_FRACTION` | 0.20 | 普通折叠保留尾部 20% Token |
| `HISTORY_FOLD_AGGRESSIVE_THRESHOLD` | 0.78 | > 78% → 激进折叠 |
| `HISTORY_FOLD_AGGRESSIVE_TAIL_FRACTION` | 0.10 | 激进折叠只保留尾部 10% |
| `HISTORY_FOLD_MIN_SAVINGS_FRACTION` | 0.30 | 节省 < 30% 则跳过折叠 |
| `FORCE_SUMMARY_THRESHOLD` | 0.80 | > 80% → 强制退出并总结 |
| `TURN_START_FOLD_THRESHOLD` | 0.90 | 轮次开始前估计超限 → 预折叠 |
| `HISTORY_FOLD_SUMMARY_TIMEOUT_MS` | 15000 | 语义摘要的硬超时 |

### 7.3 技能保留

折叠时会解析 `<skill-pin>` 标签中的活跃技能记忆体，将它们的原始内容追加到摘要之后。**这保证了折叠不会丢失活跃技能的状态**——一个正在进行中的代码审查技能，即使历史被折叠，其上下文仍被保留。

### 7.4 决策树

当触发 `fold` 时，`compactHistory()` 会截断早期轮次，但保留尾部（最近）的 Token，使得最近几轮的对话上下文仍然可用。这个过程中，系统提示、固定约束（`# HIGH PRIORITY constraints`）、用户记忆、项目记忆这些"固定部分"被提取出来持久化，不受折叠影响。

## 八、工具系统：从注册到分派的完整链路

### 8.1 ToolRegistry

`src/tools.ts` 定义了 `ToolRegistry` 类，这是所有工具的中心注册表。每个 `ToolDefinition<A, R>` 包含：

- `name`、`description`、`parameters: JSONSchema`
- `readOnly` / `readOnlyCheck` —— 计划模式下阻止变异工具
- `parallelSafe` —— 控制自动并行化
- 速率限制集成（`ToolRateLimiter`）
- 读取跟踪集成（`ReadTracker`）

### 8.2 工具分类

| 类别 | 工具 |
|------|------|
| **文件系统** | `read_file`，`write_file`，`edit_file`，`multi_edit`，`search_files`，`search_content`，`glob`，`list_directory`，`directory_tree`，`get_file_info`，`move_file`，`copy_file`，`delete_file`，`create_directory`，`delete_directory` |
| **Shell** | `run_command`，`run_background`，`wait_for_job`，`job_output`，`stop_job`，`list_jobs` |
| **Web** | `web_search`，`web_fetch` |
| **代码查询** | `find_in_code`，`get_symbols`（基于 tree-sitter） |
| **计划** | `submit_plan`，`revise_plan`，`mark_step_complete` |
| **选择** | `ask_choice` |
| **待办** | `todo_write` |
| **记忆** | `remember`，`forget`，`recall_memory` |
| **子代理** | `run_skill`，`spawn_subagent` |
| **MCP** | 动态注册（来自外部服务器） |

### 8.3 读取跟踪（Read Tracker）

`ReadTracker` 是一个会话级别的、跟踪已读取文件路径的去重机制。当模型在上下文中已经读到某个文件后，再次调用 `read_file` 会被标记为"已缓存"。这既减少了 Token 浪费，也让模型养成"读一次就记住"的习惯。

### 8.4 速率限制

`ToolRateLimiter` 实现了令牌桶算法，为每个工具独立维护速率状态。配置来自 `.reasonix/settings.json` 中的 `rateLimits` 字段。当工具调用被限流时，`RateLimitedError` 被抛出，循环自动重试（而非直接失败）。

## 九、MCP 集成：自研客户端的工程决策

### 9.1 为什么不用官方 SDK？

MCP（Model Context Protocol）是 Anthropic 推出的开放协议，有官方的 `@modelcontextprotocol/sdk` 可用。Reasonix 选择自研而非使用官方 SDK，理由有三：

1. **零运行时依赖**：`@modelcontextprotocol/sdk` 引入了大量 Reasonix 不需要的抽象（资源订阅、提示模板、完整的 JSON-RPC 批处理等），而 Reasonix 只需要 `initialize` + `tools/list` + `tools/call` 三个操作。
2. **服务面调优**：自研客户端排除了所有非必需的 MCP 能力（资源、提示、进度通知等），将代码量压缩到最小，同时避免了 SDK 破坏性变更的风险。
3. **DeepSeek 适配**：自研客户端可以与 Reasonix 的修复管道无缝集成——MCP 工具返回的结果自动经过 flatten、truncation 和 storm 检测。

### 9.2 传输层架构

MCP 客户端实现了三种传输：

| 传输 | 文件 | 适用场景 |
|------|------|---------|
| **StdioTransport** | `mcp/stdio.ts` | 本地命令（npx、pip install 的本地 MCP 服务器） |
| **SseTransport** | `mcp/sse.ts` | 远程 HTTP + SSE 端点 |
| **StreamableHttpTransport** | `mcp/streamable-http.ts` | 2025-03-26 流式 HTTP 规范 |

三种传输都实现了 `McpTransport` 接口，通过工厂函数 `buildTransportFromSpec()` 根据 CLI 参数自动选择。

### 9.3 工具桥接

`bridgeMcpTools()` 将 MCP 服务器的工具列表映射为 `ToolDefinition`，注入到 `ToolRegistry` 中。映射过程中做了三层处理：

1. **参数展平**：MCP 工具的参数 schema 同样经过 `flattenSchema`，以兼容 DeepSeek 的嵌套限制。
2. **结果约束**：工具结果被限制在指定大小以内（默认配置），超过的会被截断并保存到磁盘。
3. **命名空间隔离**：每个 MCP 服务器使用 CLI 指定的前缀来进行命名空间化，避免工具名冲突。

### 9.4 目录集成

`mcp/catalog.ts` 维护了一个知名 MCP 服务器的精选列表，包括文件系统、GitHub、Slack、Puppeteer、PostgreSQL 等。用户可以通过 `add_mcp_server` 工具从目录中选择并自动配置。

## 十、React TUI：终端中的现代 UI

### 10.1 为什么是 Ink？

Reasonix 的终端 UI 选择 Ink（React for Terminal）而非传统的 ANSI 转义码或 blessed 库，原因清晰：

1. **声明式组件模型**：每个 UI 元素（卡片、输入框、状态行）都是 React 组件，状态通过 Props 下推。
2. **与 Web 仪表盘共享逻辑**：TUI 中的事件流、状态计算、格式化逻辑同样被 Web 仪表盘使用。
3. **丰富的第三方生态**：Ink 5 提供了 InkTextInput、InkSpinner 等现成组件。

### 10.2 组件架构

UI 层位于 `src/cli/ui/`，约 85 个文件：

```
ui/
├── App.tsx          # 根组件（4704 行）
├── cards/           # 19 种事件卡片
│   ├── ApprovalCard.tsx
│   ├── DiffCard.tsx
│   ├── ToolCard.tsx
│   ├── PlanCard.tsx
│   ├── ReasoningCard.tsx
│   ├── StreamingCard.tsx
│   └── ...
├── hooks/           # 14 个自定义 Hook
├── layout/          # 布局组件（CardStream、Composer、StatusRow、ToastRail...）
├── primitives/      # 低级组件
├── theme/           # 主题系统
├── state/           # TUI 状态管理
├── effects/         # Loop → Dashboard 事件桥接
├── dashboard/       # Web 仪表盘 SSE 广播
├── slash/           # 斜杠命令解析器 + 自动补全
├── McpHub.tsx, McpBrowser.tsx, McpMarketplace.tsx
├── ModelPicker.tsx, BootSplash.tsx
└── Dashboard.tsx, DashboardPicker.tsx
```

### 10.3 渲染循环

TUI 以 50-150ms 为间隔刷新（取决于终端性能）。渲染循环的核心是 `TickerProvider` 和 `InflightProvider` 的组合——前者驱动定时刷新，后者管理正在执行中的工具调用状态。

`terminal-host.ts` 负责检测终端性能，在老旧 Windows 控制台（如 Conhost）上自动降低绘制频率。

### 10.4 斜杠命令系统

斜杠命令（`/`）解析器支持自动补全和历史浏览。命令包括：`/apply`（应用待审批编辑）、`/reject`（拒绝）、`/plan`（计划模式）、`/skill`（调用技能）、`/memory`（管理记忆）、`/session`（管理会话）、`/clear`（清屏）。

## 十一、Web 仪表盘：另一个界面范式

`src/server/` 实现了完整的 HTTP 服务器，提供 32 个 REST API 端点和 SSE 事件流：

| 端点类别 | 覆盖 |
|---------|------|
| 统计 | 使用/工具/权限 |
| 交互 | 消息/提交/中止/健康 |
| 会话 | 会话管理 |
| 计划 | 计划 CRUD |
| 设置 | 配置读写 |
| 钩子 | 钩子配置 CRUD |
| 记忆/技能/MCP/语义 | 各子系统管理 |
| 文件/浏览/项目树 | 工作区浏览 |
| Git 差异/审查 | 版本控制 |
| 检查点/编辑模式 | 代码检查点 |
| 模型 | 模型选择 |

SSE 事件流（`GET /api/events`）每 25 秒发送心跳以保持连接。Web 仪表盘可以以"独立模式"（`reasonix server`）或"附加模式"（`reasonix code --dashboard`）运行，后者将 TUI 状态实时推送到浏览器。

## 十二、记忆系统：四层金字塔

### 12.1 层级结构

| 层级 | 存储位置 | 作用范围 | 优先级 |
|------|---------|---------|--------|
| **项目记忆** | `<project>/REASONIX.md` | 当前项目 | 最高 |
| **全局记忆** | `~/.reasonix/REASONIX.md` | 所有项目 | 高 |
| **Claude 兼容** | `~/.claude/CLAUDE.md` | 跨平台迁移 | 中 |
| **用户记忆** | `~/.reasonix/memory/{global|<hash>}/` | 按作用域 | 可配 |

### 12.2 用户记忆的语义

用户记忆是以**带前言的 Markdown 文件**存储的。每个记忆文件包含：`name`（标识符）、`description`（一行摘要）、`type`（user/feedback/project/reference）、`scope`（global/project）、`priority`（low/medium/high）、`expires`（可选过期）、`body`（Markdown 正文）。

`MEMORY.md` 索引文件自动生成，列出所有记忆的摘要。**高优先级记忆会被注入到系统提示词的顶部，作为硬性约束**，确保用户偏好始终被遵守。

### 12.3 不可变前缀（ImmutablePrefix）

`ImmutablePrefix` 在会话开始时构建一次，包含：

1. 系统提示（`REASONIX.md` + `MEMORY.md` + 高阶约束）
2. 工具规范（所有注册工具的 JSON Schema）
3. 少量示例（few-shot examples）

通过 SHA-256 指纹检测变动，如果前缀在会话期间发生变化（比如用户修改了 `REASONIX.md`），整个前缀被重新计算——但这种情况极少发生，因为前缀的设计目标就是**跨轮次稳定**。

## 十三、转录、回放与差异分析

### 13.1 转录（Transcript）

转录是 Reasonix 的"收据"系统。`src/transcript/log.ts` 将每个 `LoopEvent` 转换为 `TranscriptRecord`，包含时间戳、轮次、角色、内容和工具参数、使用量（Token 数、缓存命中/未命中）、成本（USD）、模型标识、前缀哈希（用于验证缓存稳定性）、错误信息。

转录写入 JSONL 文件，每行一条记录。轮次开始时写入 `_meta` 头，记录会议元数据。

### 13.2 离线回放（Replay）

`src/transcript/replay.ts` 实现了**无需 API Key 的离线回放引擎**。它从转录文件中重建完整的会话统计数据：每轮成本、缓存命中率、前缀稳定性（前缀哈希是否在轮次间变化）、工具调用总数。分页浏览通过 `groupRecordsByTurn` 实现。

这意味着在发布前，整个 τ-bench 基准测试的成本分析可以在不消耗任何 API 预算的情况下运行——转录是冻结的，计算是纯函数的。

### 13.3 差异比较（Diff）

`src/transcript/diff.ts` 实现了两个转录的差异分析。按轮次编号配对两个会话，检测分歧点——工具调用不同（参数变化、调用顺序变化）、文本相似度 < 75%（使用 Levenshtein 距离 ≤2000 字符，词元重叠 >2000 字符），输出摘要表格和前缀稳定性分析。

这个功能在 A/B 测试中尤为关键——比较带缓存和不带缓存的同一个负载，量化缓存的经济价值。

## 十四、遥测与成本控制

### 14.1 定价表

`src/telemetry/stats.ts` 维护了 DeepSeek 的定价表：

| 模型 | 缓存命中（每百万 Token） | 缓存未命中 | 输出 | 上下文窗口 |
|------|------------------------|------------|------|-----------|
| deepseek-v4-flash | **$0.0028** | $0.14 | $0.28 | 1M |
| deepseek-v4-pro | **$0.003625** | $0.435 | $0.87 | 1M |

缓存命中的价格仅为未命中的 **1/50（flash）到 1/120（pro）**。这正是整个缓存优先策略的财务基础。作为对比，Claude Sonnet 4.6 的报价为 input $3/M、output $15/M——大约是 v4-flash 缓存命中价格的 **1,000 倍以上**。

### 14.2 使用统计

`src/telemetry/usage.ts` 将每轮使用数据追加写入 `~/.reasonix/usage.jsonl`，**仅包含 Token 数和成本，从不存储提示词**。超过 5MB 时自动压缩（清理 >365 天的记录）。`aggregateUsage()` 生成滚动窗口统计（今天/周/月/全部），按模型和会话分组。

### 14.3 子代理蒸馏遥测

`src/telemetry/subagent-distillation.ts` 测量子代理的"压缩率"——输出 Token 数 / 完成 Token 数。当一个子代理接收了 10,000 Token 的上下文，但只返回 500 Token 的结论时，压缩率为 0.05，节省了 9,500 Token。`SubagentTelemetry` 在每个父轮次粒度记录所有衍生的压缩数据，并检测"衍生风暴"（同一轮次 ≥3 个衍生）以帮助优化并行策略。

## 十五、钩子系统：可编程的生命周期

### 15.1 四个生命周期事件

Reasonix 的钩子系统（`src/hooks.ts`，2712 行）在四个关键点注入用户自定义脚本：

| 事件 | 触发时机 | 超时 |
|------|---------|------|
| `PreToolUse` | 工具调用执行前 | 5 秒 |
| `PostToolUse` | 工具调用完成后 | 30 秒 |
| `UserPromptSubmit` | 用户提交消息时 | 5 秒 |
| `Stop` | 会话结束时 | 30 秒 |

### 15.2 钩子脚本协议

每个钩子是一个可执行脚本（通过 `shell: true` 运行）。Stdin 接收 JSON 负载（事件、cwd、工具名称/参数/结果、提示文本）。退出码决定行为：exit 0 放行，exit 2 阻止（在 PreToolUse / UserPromptSubmit 中停止循环），其他退出码为警告但仍允许执行。

`match` 字段支持锚定正则，用于缩小钩子的作用范围。例如，`PreToolUse` 钩子可以只匹配 `^edit_file$`，而不影响其他工具。

### 15.3 配置与热重载

钩子配置在 `.reasonix/settings.json`（项目级）和 `~/.reasonix/settings.json`（全局级）中。服务器提供三个 API 端点：`GET /api/hooks`（获取配置）、`POST /api/hooks/save`（写入）、`POST /api/hooks/reload`（热重载）。

## 十六、SEARCH/REPLACE 代码编辑引擎

### 16.1 设计动机

Reasonix 的代码编辑不采用"直接改文件"的模式，而是让模型先生成 SEARCH/REPLACE 块，用户审批后再由引擎解析并应用。这有三个好处：

1. **用户可见的差异**：在任何修改写入磁盘前，用户可以预览更改。
2. **精确性**：SEARCH 块要求与文件中现有的内容字节精确匹配——这防止了"幻觉文件"问题。
3. **审批门控**：`edit_file` 和 `multi_edit` 工具受到审批门控，用户可以在 `/apply` 前审查所有待处理编辑。

### 16.2 块解析

`src/code/edit-blocks.ts` 使用正则解析 SEARCH/REPLACE 块。每个 `EditBlock` 包含 `path`、`search`、`replace`、`offset`。`applyEditBlocks()` 使用"滚动到第一个差异点"策略进行幂等应用——如果文件已经部分应用过编辑，第二次应用不会重复。

### 16.3 六种应用状态

`ApplyStatus` 枚举定义了六种可能的结果：`applied`（成功应用）、`created`（新文件创建）、`not-found`（SEARCH 内容不存在）、`file-missing`（目标文件不存在）、`path-escape`（路径逃逸）、`error`（其他错误）。

### 16.4 快照与检查点

在执行编辑前，`snapshotBeforeEdits()` 会创建文件的备份快照，存储在 `.reasonix/checkpoints/` 中。如果编辑失败或用户不满意，可以通过 `/rollback` 恢复。

## 十七、语义索引：本地化的代码理解

### 17.1 设计哲学

Reasonix 的语义索引不依赖外部向量数据库或云服务。它的设计哲学是：**足够好，且无需额外运维**。

### 17.2 分块策略

`src/index/semantic/chunker.ts` 实现了**基于行窗口的、语言无关的分块器**：窗口大小 60 行，重叠 12 行，最大字符数 4000，排除模式继承自 gitignore。

选择行窗口而非 AST 分块，是因为 AST 分块需要为每种语言维护语法解析器，而行窗口可以索引任何文本文件（配置文件、Markdown、日志等）。

### 17.3 嵌入生成

嵌入可以通过两种方式生成：**Ollama**（`nomic-embed-text`，本地、离线、免费）和 **OpenAI 兼容端点**（高精度、远程）。`embedAll()` 支持批处理（默认批次大小 = 10），超时 180 秒。

### 17.4 存储与搜索

`SemanticStore` 使用 JSONL 追加写入。搜索时使用**未装箱的 Float32Array 进行线性余弦扫描**——对于 ≤10,000 个块的高索引，这足够快（无需 HNSW 等索引结构）。`search()` 返回按余弦相似度排序的 `SearchHit[]`。

### 17.5 工具集成

`src/index/semantic/tool.ts` 将 `semantic_search` 暴露为循环可调用的工具。这意味着模型可以主动查询语义索引来"记住"之前看到的代码模式。

## 十八、生命周期门控：计划→审批→执行

### 18.1 Engineering Lifecycle Runtime

`src/code/lifecycle.ts` 实现了**计划 → 批准 → 执行 → 完成**的状态机。`setMode("strict")` 启用"先计划后执行"门控——在所有高风险操作之前强制要求计划提交和审批。

### 18.2 风险分类

`src/code/lifecycle-policy.ts` 将工具调用分为三类：`safe`（读取）、`mutation`（编辑）、`high-risk`（删除、批量编辑、包修改、`git push`、`npm install`）。`run_command` 会被解析为 Shell 词元，检测是否包含 `rm`、`sudo`、`curl` 等命令，动态调整风险等级。

### 18.3 待定编辑检查点

`src/code/pending-edits.ts` 将每个 SEARCH/REPLACE 块写入 `<session>.pending.json`。这意味着即使会话被中断（如终端关闭），用户下次启动 Reasonix 时仍然可以 `/apply` 恢复未完成的编辑。

## 十九、计划系统：结构化任务管理

### 19.1 类型设计

`src/tools/plan-types.ts` 定义了核心类型：`PlanStep`（带 risk/targets/acceptance/verification）、`StepEvidence`（verification/diff/checkpoint/manual）、`StepCompletion`（含 evidence 数组支持追溯审计）。每个步骤都可以携带验收标准和验证命令。

### 19.2 工作流

`submit_plan` → 用户审批/修改 → `mark_step_complete`（逐步骤推进）→ `revise_plan`（调整剩余步骤）。这与代码编辑的生命周期门控相呼应——两者的设计理念一致：**审批不是限制，而是反馈闭环**。

## 二十、基础设施与工程实践

### 20.1 技术栈

| 领域 | 选择 | 理由 |
|------|------|------|
| 语言 | TypeScript 5.6+，ES2022 | 类型安全，生态丰富 |
| 模块系统 | ESM（`"type": "module"`） | 现代 Node.js 标准 |
| CLI 框架 | Commander.js | 成熟、类型友好 |
| TUI | Ink 5 + React 18 | 声明式终端 UI |
| 测试 | Vitest 2.x | 快速、兼容 Vite 生态 |
| 格式/Lint | Biome 1.9 | 统一工具、极快 |
| 构建 | tsup + esbuild | 零配置打包 |
| HTTP | undici | Node 22 内建，零依赖 |
| 树解析 | web-tree-sitter | WASM 绑定，多语言 |

### 20.2 CI/CD

GitHub Actions 配置了 7 个工作流：

| 工作流 | 触发条件 | 检查内容 |
|--------|---------|---------|
| **CI** | PR 和 main 分支推送 | `npm ci` → Biome lint → tsc typecheck → tsup build → vitest test:coverage → τ-bench dry-run |
| **CodeQL** | 每周 | 安全扫描 |
| **发布 npm** | 标签 `v*` | 验证 tag == package.json 版本 → `npm publish` |
| **发布桌面** | 标签 `desktop-v*` | Tauri bundle（linux-x64, macos-universal, windows-x64） |

CI 在 Ubuntu 和 Windows 双平台上运行，确保跨平台兼容性。

### 20.3 探测脚本

`scripts/` 目录包含一组**长运行健康检查**脚本（`probe-*`）：`probe-loop-cache.mts`（缓存保持）、`probe-fanout.mts`（并行分派）、`probe-mem-leak.mts`（内存泄漏）、`probe-long-session.mts`（长会话磨损）、`probe-jobs-leak.mts`（后台作业泄漏）。这些脚本通过真实的 DeepSeek API 调用运行，在发布前检测回归。

### 20.4 测试覆盖

约 **260 个测试文件**，覆盖从单元级到集成级到 UI 级的各个层面：

| 测试域 | 文件数 |
|--------|--------|
| 主循环（loop*.test.ts） | ~10 |
| 上下文管理 | ~5 |
| MCP 客户端 | ~25 |
| 桌面 Tauri | ~15 |
| 仪表盘 | ~8 |
| React 组件 | ~12 |
| 斜杠命令 | ~5 |
| 子代理 | ~3 |
| 语义索引 | ~5 |
| 代码查询 | ~5 |
| 计划 | ~6 |
| 工具注册表 | ~5 |

## 二十一、基准测试与真实世界表现

### 21.1 τ-Bench 评估

`benchmarks/tau-bench/` 是 Reasonix 的核心基准测试框架，基于一个模拟零售环境的 8 个任务集。每个任务使用相同的 MCP 工具集，运行**两种模式**——一个故意破坏缓存的基线（每轮注入新时间戳）和一个基于 `CacheFirstLoop` 的 Reasonix 代理。唯一的变量是前缀稳定性，这就隔离了缓存优先架构的成本影响。

成功谓词是**确定性数据库检查**（而非 LLM 评判），确保结果可重现。

### 21.2 关键指标

| 指标 | Reasonix | 基线 |
|------|---------|------|
| 缓存命中率 | 96.7% | <20% |
| 每轮成本（flash） | ~$0.003 | ~$0.14 |

### 21.3 真实用户案例

一位真实用户（2026-05-01）的统计数据：435M 输入 Token，**99.82% 缓存命中率**，实际费用 ~$12，无缓存等价负载费用 ~$61（v4-flash），节省比例约 80%。

## 二十二、总结：架构哲学与取舍

### 22.1 "Opinionated, not general"

Reasonix 的核心架构哲学可以用一句话概括：**每层都为 DeepSeek 调优**。不做通用 LLM 网关，不做模型无关的抽象。好处是极致的效率和成本控制；代价是模型锁定。对于以 DeepSeek 为主要 LLM 的用户来说，这是值得的取舍。

### 22.2 缓存优先不是功能，是不变式

从不可变前缀到仅追加日志，从易失暂存到前缀哈希校验——所有这些设计都不是"可选优化"，而是**系统的不变式**。每个组件都假设前缀必须稳定，日志必须可重现。这种设计强度带来了 96-99% 的缓存命中率，以及比无缓存方案低 50-120 倍的 Token 成本。

### 22.3 修复管道是架构的隐藏支柱

大多数 Agent 框架假设 LLM 的输出是"规范的"——工具调用格式正确、参数完整、不会重复。Reasonix 的经验证明了这是个错误假设。scavenge、flatten、truncation、storm 四道工序每一道都对应一个真实世界的高频失效模式。这个管道使得整个循环对于模型不完美输出具有鲁棒性。

### 22.4 事件溯源是可靠性的基石

事件溯源 + 纯函数归约器的组合，使得 Reasonix 可以轻松实现离线回放、差异分析、会话检查点——这些都是可变状态架构下难以（或需要大量额外工作）实现的能力。代价是更高的内存占用（需要保留完整事件日志），但考虑到 DeepSeek 的百万 Token 窗口，这在实践中是可接受的。

### 22.5 面向未来的展望

从代码库中可以看到 Reasonix 的未来方向：

- **MCP 市场**：`marketplace-overlay/` 的中文本地化和精选目录，表明 Reasonix 正在构建 MCP 工具生态。
- **Tauri 桌面**：`desktop/` 目录和桌面发布工作流，表明 Reasonix 正在从终端工具走向桌面应用。
- **Subagent 蒸馏**：复杂的子代理编排和蒸馏遥测，表明 Reasonix 正在探索多代理协作范式。
- **热重载钩子**：`POST /api/hooks/reload`，表明 Reasonix 正在向"运行时无需重启"的方向演进。

---

## 附录：关键文件速查表

| 文件路径 | 行数 | 核心职责 |
|---------|------|---------|
| `src/loop.ts` | 1052 | CacheFirstLoop 主循环 |
| `src/core/events.ts` | 329 | 40+ 种事件类型 |
| `src/core/reducers.ts` | 239 | 纯函数归约器 |
| `src/tools.ts` | 509 | ToolRegistry |
| `src/repair/flatten.ts` | 99 | Schema 展平 |
| `src/context-manager.ts` | 345 | 上下文自动折叠 |
| `src/client.ts` | 367 | DeepSeek API 客户端 |
| `src/hooks.ts` | 2712 | 生命周期钩子系统 |
| `src/cli/ui/App.tsx` | 4704 | TUI 根组件 |

---

*本文基于 Reasonix v0.51.0 的代码库编写，覆盖了 50+ 核心源文件和 260+ 测试文件。项目采用 MIT 许可证。*
