---
title: "Pi 源码分析 02 · 三层架构：层与层之间怎么接"
date: 2026-10-01T17:00:00+08:00
description: "接着第一章的骨架往下讲：依赖规则靠什么守住、上层实际用到下层的哪些东西、类型如何逐层加料、L3 怎么把业务注入 L2，以及 L2 正在进行的演进。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.1（2026-10-03），commit `a7229ddc2`。文中所有架构、数字和代码均以该版本源码为准。

第一章画出了 Pi 的骨架：L1 `pi-ai` 调模型，L2 `pi-agent-core` 跑循环，L3 `pi-coding-agent` 做产品，依赖只朝下。本章接着往下讲，打开层与层之间的接口，看看这三层具体是怎么接起来的。

<!--more-->

本章要点：

- **规则靠 CI 守住**：每个公开包在运行时 import 的包，都必须写在自己的 `package.json` 依赖里，CI 会检查这一点。
- **接口很窄**：coding-agent 自己的逻辑只用到 agent-core 的 3 个值；agent-core 的核心部分只借用 pi-ai 的纯函数，调模型这件事交给外部注入的函数。
- **类型逐层加料，从不改下层**：工具、消息、事件三条类型线，各用一种扩展手法。
- **L2 在演进**：除了现在发布的 `Agent`，同一个位置上还有一套持久化引擎 `pi-durable` 在开发。它同样遵守"只朝下依赖"。

---

## 1 · 依赖规则靠什么守住

![图 1 · npm run check 里的三道检查](/images/harness/pi/ch02/fig1-guards.png)

第一章说过，下层不知道上层的存在。在代码里搜一遍可以确认：pi-ai 里找不到 agent-core 和 coding-agent，agent-core 里找不到 coding-agent，连类型导入都没有。

这条规则由 CI 保证。根目录的 `npm run check` 串了八步检查，CI 和发布前都会运行，其中三步跟架构有关：

- **runtime-deps（守方向）**：扫描每个公开包的源码，凡是运行时用到的 import，被导入的包都必须写在这个包的 `dependencies`、`peerDependencies` 或 `optionalDependencies` 里（开发依赖不算）。pi-ai 唯一的内部依赖是 `pi-telemetry`，所以它在运行时 import 不了 agent-core。
- **entry-graphs（守入口体积）**：给重要的导出入口设预算。例如 pi-ai 的 `./models` 入口最多只能牵连 15 个文件，而且不能碰到供应商实现。脚本开头的注释是这样说的："Entry points are cost contracts"。
- **browser-smoke（守可拆）**：用 esbuild 打几个包，其中一个是只带 Anthropic 供应商的 `Agent`，检查它的产物里没有混进其他供应商和兼容层。

所以，分层的规则就写在每个包的 `package.json` 里。只要不往下层的依赖里加上层，反向依赖就进不了主干。

> 📌 这项检查只看运行时 import。`import type` 编译后会被擦除，不在检查范围内；目前反向的类型导入同样为零。

---

## 2 · 上层实际用到下层的哪些东西

![图 2 · 上层在运行时实际用到下层的哪些值](/images/harness/pi/ch02/fig2-seams.png)

`package.json` 只说明"允许依赖"。要知道实际依赖了什么，就得数一数上层在运行时导入了下层的哪些值。

### 2.1 L3 → L2：自身只用 3 个，再把整个包交给扩展

coding-agent 自己的逻辑只从 agent-core 导入了 3 个值：

- `Agent`（引擎类，在 SDK 入口里创建）
- `setDefaultStreamFn`（设置默认的模型调用函数）。第一章说过，`streamFn` 是 `Agent` 唯一不能缺的选项；coding-agent 作为宿主替扩展预先装好一个默认值，扩展自己创建 `Agent` 时不传也能跑
- `runToolCall`（执行单次工具调用。嵌套工具调用借它复用引擎的执行流程）

其余几十处引用都是类型，例如 `AgentMessage`（引擎里流转的消息）、`AgentTool`（引擎能执行的工具），编译后不会留下任何东西。

另外还有一处特殊的导入：给扩展用的虚拟模块表，用 `import * as` 把整个 agent-core 交给扩展：

```typescript
import * as bundledPiAgentCore from "@earendil-works/pi-agent-core";
// …
export const VIRTUAL_MODULES: Record<string, unknown> = {
	// …
	"@earendil-works/pi-agent-core": bundledPiAgentCore,
	// …
};
```

扩展里写 `import { Agent } from "@earendil-works/pi-agent-core"` 时，拿到的就是 coding-agent 自带的这一份，而且是全部导出。编译成二进制或从源码运行时用的是这张表；npm 安装的 Node 版本用的是扩展加载器里的一张别名表，效果相同。所以要分开看：

- coding-agent 内部只依赖 L2 的 3 个值。将来换引擎时，它自己要改的地方很少。
- 但它作为扩展的宿主，把 L2 的完整 API 交给了扩展作者，这部分同样是对外的承诺。

### 2.2 L2 → L1：只借纯函数，不调模型

agent-core 只有 6 个文件：`Agent`、循环、代理流和类型。它唯一的内部依赖是 pi-ai，借用的都是不涉及网络的工具函数：

- `normalizeContext`（把系统提示词和工具列表折叠成开头的一条 system 消息，得到供应商能接收的上下文）
- `validateToolArguments`（按工具的参数 schema 校验并修正模型给出的参数）
- `toToolDeclaration`（去掉工具里的执行函数和展示字段，只留下给模型看的声明）
- `EventStream`（可以用 `for await` 逐个读取的事件流容器）
- `getCurrentTools`（回放历史里的所有 system 消息，算出当前可用的工具）
- `getToolStateChanges`（比较前后两组工具，得出新增和移除了哪些）
- `getCurrentSystemPrompt`（回放所有 system 消息，得到当前的系统提示词文本）
- 等等

真正调模型的那一步，用的是外部注入的 `streamFn`（模型调用函数）。引擎每次请求前的顺序如下：

```typescript
// Apply context transform if configured (AgentMessage[] → AgentMessage[])
let messages = context.messages;
if (config.transformContext) {
	messages = await config.transformContext(messages, signal);
}

// Convert to LLM-compatible messages (AgentMessage[] → Message[])
const llmMessages = await config.convertToLlm(messages);

const llmContext = normalizeContext({ messages: llmMessages });
// …
const response = await streamFunction(config.model, llmContext, {
	...config,
	apiKey: resolvedApiKey,
	signal,
});
```

依次是：先改写上下文（`transformContext`），再转成模型能理解的消息（`convertToLlm`），然后交给 `normalizeContext`。这里只传了消息，因为提示词和工具早已作为 system 消息放在列表里，所以这一步只是把列表标记成供应商入口要求的格式。最后交给 `streamFn`，引擎不知道背后是哪家模型。

在这之前，主循环还会先调一次 `prepareRequest`（请求前替换上下文、模型或推理强度）。第 4 节会讲 L3 怎么用它。

### 2.3 L3 → L1：按需直接使用

依赖规则只要求单向，上层可以直接使用任意下层。不算实验目录，coding-agent 有 26 个文件在运行时直接导入 pi-ai，例如：

- 生成压缩摘要时直接发一次请求、等完整回复。压缩函数优先用传入的 `streamFn` 发请求再取结果，没有传入时才用 pi-ai 的 `completeSimple`（发一次请求，等完整回复）；`AgentSession` 总是传入 `Agent` 的 `streamFunction`，所以产品里的摘要请求都走 `streamFn`。摘要不需要工具、循环和事件，所以不经过 `Agent` 的循环。
- 自动重试时用 `retryDelayMs`（按重试次数算出退避等待时间）。
- 选择推理强度时用 `clampThinkingLevel`（把用户要的推理强度调整到模型实际支持的档位）。

L1 的消息、模型和工具函数是所有上层共用的基础。需要什么就依赖哪一层，这正是"单向依赖"给出的自由。

---

## 3 · 类型如何逐层加料

每一层都在下层类型的基础上增加自己需要的字段，下层类型从不为上层修改。工具、消息、事件三条线，各用一种扩展手法。

### 3.1 工具：先继承，再转换

![图 3 · Tool → AgentTool → ToolDefinition](/images/harness/pi/ch02/fig3-tool-types.png)

- **L1 `Tool`（给模型看的工具声明）**：主要是 `name`（名字）、`description`（描述）、`parameters`（参数 schema），外加可选的 `constrainedSampling`（让供应商按 JSON schema 或语法约束参数的生成），没有执行函数。模型只需要知道工具叫什么、收什么参数。
- **L2 `AgentTool`（引擎能执行的工具）**：继承 `Tool`，加上 `label`（界面上显示的名字）、`execute`（执行函数，可以通过回调汇报进度）、`executionMode`（与其他工具串行还是并行执行）等。
- **L3 `ToolDefinition`（扩展和内置工具的完整定义）**：没有继承 `AgentTool`，而是把它的大部分字段（`replay` 除外）重新声明了一遍，再加上 `promptSnippet` / `promptGuidelines`（写进系统提示词的说明）、`renderCall` / `renderResult`（在终端里怎么显示调用和结果）、`defaultActive`（是否默认启用）、`prepareLoadout`（工具组合变化时调整声明）等。`execute` 还多了一个参数 `ctx`（扩展运行环境：可以访问界面、工作目录、会话，还能调用其他工具）。

L3 的工具交给 L2 之前，要经过 `wrapToolDefinition`（把 `ToolDefinition` 转成 `AgentTool`）。代码里另外几个字段是：`outputSchema`（成功结果的结构化数据格式）、`prepareArguments`（校验前修补模型给的原始参数）、`ctxFactory`（为每次调用创建 `ctx` 的工厂）。

```typescript
/** Wrap a ToolDefinition into an AgentTool for the core runtime. */
export function wrapToolDefinition<TDetails = unknown>(
	definition: ToolDefinition<any, TDetails>,
	ctxFactory?: ToolContextFactory,
): AgentTool<any, TDetails> {
	return {
		name: definition.name,
		label: definition.label,
		description: definition.description,
		parameters: definition.parameters,
		outputSchema: definition.outputSchema,
		constrainedSampling: definition.constrainedSampling,
		prepareArguments: definition.prepareArguments,
		executionMode: definition.executionMode,
		execute: (toolCallId, params, signal, onUpdate, ctx?: ExtensionToolContext) =>
			definition.execute(
				toolCallId,
				params,
				signal,
				onUpdate,
				ctx ?? (ctxFactory?.(toolCallId, signal) as ExtensionToolContext),
			),
	};
}
```

它只拷贝 L2 认识的字段，并把创建 `ctx` 的工厂封进 `execute`：调用方传了 `ctx` 就直接用，没传就为这次调用现造一个。提示词字段留给 L3 拼系统提示词，渲染字段留给终端界面。L2 永远看不到这些字段，L3 怎么改它们都不会影响引擎。

### 3.2 消息：声明合并加进去，发给模型前再转换回来

L2 留了一个空接口，供上层往里加自己的消息类型：

```typescript
export interface CustomAgentMessages {
	// Empty by default - apps extend via declaration merging
}

// …
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

L3 用 TypeScript 的声明合并（在另一个模块里给同名接口追加成员）加入了 4 种消息：

```typescript
declare module "@earendil-works/pi-agent-core" {
	interface CustomAgentMessages {
		bashExecution: BashExecutionMessage;
		custom: CustomMessage;
		branchSummary: BranchSummaryMessage;
		compactionSummary: CompactionSummaryMessage;
	}
}
```

于是在 coding-agent 里，引擎流转的消息有 8 种：pi-ai 的 `system`、`user`、`assistant`、`toolResult`，加上这 4 种：

| 消息 | 含义 | 发给模型时变成 |
|---|---|---|
| `bashExecution` | 用户用 `!` 前缀直接执行的命令 | 一条 user 消息：命令加输出。用 `!!` 执行的不发给模型 |
| `custom` | 扩展插入的消息 | 一条 user 消息 |
| `branchSummary` | 用 `/tree` 切换分支时，为离开的分支生成的摘要 | 一条 user 消息，摘要前后加固定说明 |
| `compactionSummary` | 压缩旧消息后留下的摘要 | 一条 user 消息，同上 |

引擎对这些消息一视同仁地存储和转发，只有在调模型前，才由 `convertToLlm`（把引擎消息转成模型能理解的消息）转换回标准格式。转换函数最后用 TypeScript 的 `never` 做了穷尽检查：以后 L3 新增一种消息却忘了处理，编译就会失败。

### 3.3 事件：外面包一层，再加新事件

- **L1 `AssistantMessageEvent`（模型输出的流式事件）**：12 种。文本、思考、工具调用各有开始、增量、结束 3 种，再加上 `start`（回复开始）、`done`（正常结束）、`error`（出错或被中止）。
- **L2 `AgentEvent`（引擎运行事件）**：10 种。整次运行的开始和结束（`agent_*`）、每一轮的开始和结束（`turn_*`）、每条消息的开始、更新、结束（`message_*`）、每次工具执行的开始、进度、结束（`tool_execution_*`）。L1 的事件被包在 `message_update` 里继续往上传。
- **L3 `AgentSessionEvent`（会话事件）**：在 L2 的基础上新增 13 种，例如 `compaction_start/end`（开始和结束压缩）、`auto_retry_start/end`（开始和结束自动重试）、`queue_update`（排队消息有变化）、`entry_appended`（会话文件里写入了新条目）。另外，`agent_end` 多了 `willRetry`（是否会自动重试），工具事件多了 `parentToolCallId`（嵌套调用时的上级调用）。

三条线放在一起看：**工具靠转换函数隔开，消息靠声明合并扩展，事件靠外包一层再追加**。

---

## 4 · L3 怎么把业务注入 L2

![图 4 · 两批注入点](/images/harness/pi/ch02/fig4-injection.png)

`Agent` 对"编码"一无所知。L3 的业务分两批注入进去。

**第一批：创建 `Agent` 时传入。** SDK 入口里的代码节选如下：

```typescript
const agent = new Agent({
	initialState: { systemPrompt: "", model, thinkingLevel, tools: [], messages: existingSession.messages },
	convertToLlm: convertToLlmWithBlockImages,
	streamFn: async (model, context, options) => {
		const requestOptions = buildRequestOptions(model, options);
		// …缓存预热…
		return modelRuntime.streamSimple(model, context, requestOptions);
	},
	onPayload: transformProviderPayload,
	onResponse: handleProviderResponse,
	onProviderStreamEvent: handleProviderStreamEvent,
	sessionId: sessionManager.getSessionId(),
	transformContext: async (messages) => {
		const runner = extensionRunnerRef.current;
		if (!runner) return messages;
		return runner.emitContext(messages);
	},
	// …其余设置项…
});
```

- `streamFn`（模型调用函数）：交给 coding-agent 的模型运行时，它负责认证和自定义模型
- `convertToLlm`（消息转换）：就是 3.2 节的转换函数，可以按设置屏蔽图片
- `transformContext`（请求前改写上下文）：交给扩展的上下文事件。先是 `context`，处理函数只看到对话消息，system 消息由 pi 还原；再是 `context_with_system`，处理函数看到含 system 消息的完整列表，返回值原样使用
- `onPayload` / `onResponse` / `onProviderStreamEvent`（请求发出前、收到 HTTP 响应头时、收到供应商的原始流事件时）：分别交给扩展的 `before_provider_request`、`after_provider_response`、`provider_stream_event` 事件

注意 `tools: []` 和 `systemPrompt: ""`：这时工具和提示词都还没定，要等扩展加载完才知道。

**第二批：创建后直接改写字段。** `AgentSession` 的构造函数订阅完事件后，依次调用 6 个安装方法，改写下面这些钩子：

| 钩子 | 作用 |
|---|---|
| `beforeToolCall` / `afterToolCall`（工具执行前 / 后） | 派发扩展的 `tool_call`（可拦截）和 `tool_result`（可改结果）事件 |
| `prepareNextTurnWithContext`（下一轮开始前） | 需要时先压缩，再重建系统提示词和工具列表 |
| `prepareRequest`（每次请求前） | 从会话树重新算出这次请求的上下文；用的是虚拟模型时，先路由到具体模型，超过那个模型的阈值就先压缩 |
| `finishTurn`（一轮结束时） | 派发扩展的 `turn_end` 事件，扩展可以要求继续跑 |
| `transformContext`（再包两层） | 按工具 `prepareLoadout` 的要求隐藏部分工具声明；扩展在 `before_agent_start` 里返回了提示词时，用它替换整段系统提示词 |

除了两个工具钩子是直接覆盖，其余的都先保存原来的钩子，再用新函数包住它。所以同一个钩子可以层层叠加，引擎不知道外面包了几层。

**事件向上。** `AgentSession` 收到引擎事件后：

```typescript
// Emit to extensions first, then notify public listeners.
await this._emitExtensionEvent(event);
this._emit(event.type === "agent_end" ? { ...event, willRetry: this._willRetryAfterAgentEnd(event) } : event);

// Handle session persistence
```

先发给扩展，再发给订阅者（终端界面、JSON 模式、RPC、SDK 调用方），最后写进会话文件。

---

## 5 · L2 在演进：另一套持久化引擎

![图 5 · L2 的位置上现在有两套引擎](/images/harness/pi/ch02/fig5-engines.png)

前面讲的都是现在发布出去的 `pi` 所用的引擎。在仓库里，L2 这个位置上还有一套新引擎在开发：

1. **`Agent`（当前发布的引擎）**：agent-core 的 6 个文件，约 2.5k 行，就是前四节讲的内容。
2. **`pi-durable`（持久化引擎，规范里叫 Pico5）**：独立的包，约 17.7k 行，已经发布到 npm，README 开头标着 "Experimental"（API 会在版本之间不经通知地变化）。README 是这样介绍它的："Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown." 也就是说，消息、模型输出、工具调用和你的状态，都先写入存储再展示。进程中途崩溃，重新打开就能接着跑。

两套引擎有一个共同点：**都只往下依赖 pi-ai，都不依赖 coding-agent**。三层结构对它们同样成立，在变的只是 L2 的实现。`pi-durable` 也不依赖 agent-core，内部依赖只有 pi-ai 和 `chord`（第一章实验区里的组合运行时）。

从 2026-08-01 到 2026-10-03 的提交可以看出开发重心在哪里：`agent-loop.ts` 只有 9 次提交，`pi-durable` 从 2026-09-18 创建以来有 78 次（其中 17 次是发版和新增 changelog 段落的例行提交）。coding-agent 的实验目录里已经有三处在用它：

- `experimental/durable`：单进程的终端编码 Agent
- 实验性 server / client 的会话 worker：每个会话一个 `pi-durable` 实例
- `experimental/vacation`：一个示例

实验代码不会影响 npm 用户：coding-agent 发布时排除了 `experimental/` 目录，也没有把 `pi-durable` 列为依赖，所以安装的 `pi` 只跑 `Agent` 这一套引擎。

> 💡 这种演进方式本身就得益于分层：新引擎只要满足"往下依赖 pi-ai、往上接 coding-agent"，就能在实验目录里和现有引擎并行开发，不用动已发布的路径。本系列接下来的 Agent Loop、工具、事件章节讲的是当前发布的 `Agent`；新引擎稳定之后再单独成章。

---

## 6 · 按需取层：每一层都能单独用

单向依赖的直接好处是：可以只拿走下面几层。仓库里每种组合都有现成的代码：

| 你要的层 | 现成代码 |
|---|---|
| 只要 L1 | pi-ai README 的 Quick Start（见第一章 4.1 节） |
| L1 + L2 | 打包检查用的入口，共 13 行，原文见下 |
| L1 + L2 + 能力库 | agent-core 的 `mcp-codemode` 示例：`Agent` 加 MCP 和 codemode，不带 coding-agent |
| L1 + L2 的循环，不要 `Agent` 类 | `agentLoop()`（直接运行一次循环，返回事件流） |
| 全部三层 | coding-agent 的 SDK 示例 01–14，入口是 `createAgentSession()` |

L1 + L2 的最小形态，就是 browser-smoke 检查打包的那个入口：

```typescript
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-5");
if (!model) throw new Error("Anthropic smoke-test model not found");

export const agent = new Agent({
	initialState: { model },
	streamFn: models.streamSimple.bind(models),
});
```

它同时也是一条 CI 断言：只要这样使用 `Agent`，打包产物里就不会混进其他供应商。

> ⚠️ 直接调用 `agentLoop()` 时，事件流只负责通知，不会等你的异步处理完成就继续往下跑。如果你的处理需要在工具执行前完成，就用 `Agent` 类。

---

## 7 · 三个可以带走的方法

1. **用运行时导入衡量层间接口。** `package.json` 只说明"允许依赖"。数一数上层在运行时实际导入了下层的哪些值，就知道接口有多宽，也就知道换掉下层要改多少地方。
2. **上层加料，交给下层前先转换。** `wrapToolDefinition` 和 `convertToLlm` 是同一个模式：上层在自己的类型上随意加字段，交给下层前转换成下层认识的形状。下层的类型因此永远不需要为上层修改。
3. **把规则写进 CI。** 方向、入口体积、可拆性各有一个检查脚本。分层能长期保持，靠的是违反规则时 CI 会失败。

---

## 8 · 关键数字

| 指标 | 数值 |
|---|---|
| coding-agent 自身从 agent-core 导入的运行时值 | 3 个，另有 1 处整包导入交给扩展 |
| coding-agent 中运行时直接导入 pi-ai 的文件（不含实验目录） | 26 个 |
| 反向依赖（包括类型导入） | 0 |
| `npm run check` | 8 步，其中 3 步与架构有关 |
| coding-agent 里引擎流转的消息种类 | 8 种（pi-ai 4 种 + 扩展 4 种） |
| 事件种类 | L1 12 种 → L2 10 种 → L3 新增 13 种 |
| AgentSession 构造时调用的钩子安装方法 | 6 个 |
| L2 位置上的引擎 | 2 套（`Agent` 稳定发布；`pi-durable` 标为 Experimental，只有实验代码在用） |

---

## 9 · 术语表

| 术语 | 含义 |
|---|---|
| **运行时导入** | 编译后仍然存在的 import，代表真实的依赖；`import type` 编译后会被擦除 |
| **声明合并** | TypeScript 允许在另一个模块里给同名接口追加成员，L3 借此扩展引擎的消息类型 |
| **`convertToLlm`** | 每次调模型前，把引擎消息转成模型能理解的消息 |
| **`transformContext`** | 在 `convertToLlm` 之前运行，改写引擎消息列表，例如裁剪历史、注入上下文 |
| **`wrapToolDefinition`** | 把 L3 的工具定义转成 L2 能执行的工具，丢掉提示词和渲染字段 |
| **虚拟模块** | 扩展 import pi 的包时，解析到 coding-agent 自带的那一份 |
| **pi-durable** | L2 位置上的实验性持久化引擎；先写入存储再展示，崩溃后能接着跑 |

---

## 10 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 依赖规则怎么检查 | 根目录 `package.json` 的 `check`；`scripts/check-runtime-deps.mjs`、`scripts/check-entry-graphs.mjs`、`scripts/check-browser-smoke.mjs` |
| L3 怎么创建引擎 | `packages/coding-agent/src/core/sdk.ts` → `new Agent` |
| L3 怎么注入钩子 | `packages/coding-agent/src/core/agent-session.ts` → 构造函数里的 `_install*` |
| 扩展拿到的模块 | `packages/coding-agent/src/core/extensions/virtual-modules.ts` |
| 工具类型怎么转换 | `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` |
| 自定义消息怎么转换 | `packages/coding-agent/src/core/messages.ts` → `convertToLlm` |
| 引擎的请求顺序 | `packages/agent/src/agent-loop.ts` → `streamAssistantResponse` |
| 三条类型线的定义 | `packages/ai/src/types.ts`；`packages/agent/src/types.ts`；`packages/coding-agent/src/core/extensions/types.ts`（`ToolDefinition`）；`packages/coding-agent/src/core/agent-session.ts`（`AgentSessionEvent`）；`packages/coding-agent/src/core/messages.ts`（声明合并） |
| 新引擎 | `packages/durable/README.md`；`packages/durable/docs/spec.md`；`packages/coding-agent/src/experimental/durable/` |

**下一章**：深入当前发布的引擎，看 `agent-loop.ts` 的双层循环、插队消息与后续消息，以及工具的串行与并行执行。
