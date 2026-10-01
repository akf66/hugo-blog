---
title: "Pi 源码分析 02 · 三层架构：稳定的骨架与正在被替换的中间层"
date: 2026-10-01T17:00:00+08:00
description: "用代码检验“Pi 是三层架构”：依赖方向怎么守住、层间接缝有多窄、类型如何逐层加料，以及 L2 位置上并存的四套引擎。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v0.99.2 之后的 main（`v0.99.2-23-g0c453048b`，2026-10-01），commit `0c453048b`。文中所有架构、数字和代码均以该版本源码为准。

上一章把 Pi 画成了 L0–L3 的堆栈。本章先把"Pi 是三层架构"当作一个待检验的说法，用代码去检验它，回答三个问题：

1. **三层是怎么守住的？** 依赖方向由什么保证，层与层之间的接缝有多窄
2. **类型和控制权怎么穿过各层？** 工具、消息、事件三条类型线，以及 L3 往 L2 里注入业务的点
3. **三层还成立吗？** 在 L2 这个位置上，现在并存着四套引擎

<!--more-->

---

## 0 · 先给结论

| 说法 | 结论 | 一句话 | 见 |
|---|---|---|---|
| Pi 是三层架构 | 部分成立 | 只对已发布路径成立 | §1、§5 |
| 只依赖相邻的下一层 | 不成立 | 方向单向，但 L3 可直接用 L1 | §2 |
| L3 重度依赖 L2 | 部分成立 | 自身只用 3 个值，但把 L2 全部导出交给扩展 | §2 |
| 上层类型继承下层 | 部分成立 | 三条线三种手法，不全是 extends | §3 |
| L2 只有一个引擎 | 不成立 | 另有三套只被实验代码使用 | §5 |

> 💡 本章的数字都可以重跑。值导入统计用的是一个小脚本：它识别具名导入、命名空间导入（`import * as`）、默认导入、`export … from` 和动态 `import()`，跳过 `import type` 和 `{ type X }`。行数统计用的是 `find <目录> -name '*.ts' | xargs cat | wc -l`，只统计 `src/`。

---

## 1 · 依赖方向由谁守住

![图 1 · npm run check 里的三道检查](/images/harness/pi/ch02/fig1-guards.png)

先看最基本的一条：**下层不知道上层**。在 `packages/ai/src` 里搜 `pi-agent-core` 和 `pi-coding-agent`，在 `packages/agent/src` 里搜 `pi-coding-agent`，包括 `import type` 在内，命中都是零。

这个结果不是靠自觉维持的。仓库没有用 lint 规则限制 import 方向（`biome.json` 里没有 `noRestrictedImports`），而是在根目录 `package.json` 的 `check` 脚本里串了九步检查。CI（`.github/workflows/ci.yml`）和 `prepublishOnly` 都会执行它。其中三步跟架构直接相关：

- **`check:runtime-deps`（守方向）**：`scripts/check-runtime-deps.mjs` 用 TypeScript AST 找出每个运行时 import、`import()` 和 `require`，要求被引用的包必须出现在本包的 `dependencies` / `optionalDependencies` / `peerDependencies` 里。pi-ai 的 manifest 里唯一的 workspace 依赖是 `pi-telemetry`，所以它在运行时 import 不了 agent-core。
- **`check:entry-graphs`（守入口体积）**：`scripts/check-entry-graphs.mjs` 沿值导入图遍历有预算的入口，限制可达文件数和禁止路径。例如 pi-ai 的 `./models` 入口最多 15 个文件，并且不能碰到 `providers/`。
- **`check:browser-smoke`（守可拆）**：`scripts/check-browser-smoke.mjs` 用 esbuild 打三个包，其中一个是只带 Anthropic 供应商的 `Agent`。它的产物里不能出现 `compat.ts`、`models.generated.ts`、`providers/all.ts`，模型数据只能有 `anthropic.json`，SDK 只能有 `@anthropic-ai/sdk`。

`check-entry-graphs.mjs` 开头的注释说明了它存在的原因："Entry points are cost contracts… one stray `export *` can silently make a narrow entry drag an entire barrel"。

所以准确的说法是：**分层的规则写在每个包的 `package.json` 里，再由 `check-runtime-deps` 保证代码不越过 manifest**。只要不往下层的 manifest 里加上层依赖，反向依赖就进不了主干。

> 📌 `check-runtime-deps` 只检查值导入，`import type` 编译后会被擦除，所以不在检查范围内。目前反向的类型导入同样为零，但这一点靠的是 grep 结果，不是 CI。

---

## 2 · 接缝有多窄

![图 2 · 上层在运行时实际用到下层的哪些值](/images/harness/pi/ch02/fig2-seams.png)

`package.json` 只说明"允许依赖"。要知道"实际依赖"，得看运行时的值导入。

### 2.1 L3 → L2：自己只用 3 个值，但给扩展开了整扇门

在 `packages/coding-agent/src` 里排除 `experimental/` 目录后，只有 3 个文件从 `pi-agent-core` 导入了值。其中两个是 coding-agent 自己的逻辑：

| 值 | 位置 | 用途 |
|---|---|---|
| `Agent` | `core/sdk.ts` | 创建引擎实例 |
| `setDefaultStreamFn` | `core/sdk.ts` | 给扩展自己创建的 `Agent` 或底层循环兜底一个默认的 `streamFn` |
| `runToolCall` | `core/agent-session.ts` | 嵌套工具调用复用引擎的工具执行管线 |

另外有 36 个文件只做类型导入（`AgentMessage`、`AgentTool`、`ThinkingLevel`……），编译后不留痕迹。

`sdk.ts` 里的注释说明了 `setDefaultStreamFn` 为什么由 L3 来设置（节选）：

```typescript
// … for extensions that construct Agent instances
// or invoke low-level agent loops without supplying streamFn. Agent core remains
// provider-agnostic and does not import pi-ai/compat itself.
setDefaultStreamFn(streamSimple);
```

第三个文件是 `core/extensions/virtual-modules.ts`，它用命名空间导入把整个包交给扩展：

```typescript
import * as bundledPiAgentCore from "@earendil-works/pi-agent-core";
import * as bundledPiAiCompat from "@earendil-works/pi-ai/compat";
// …
/** Modules available to extensions in source and compiled binary runtimes. */
export const VIRTUAL_MODULES: Record<string, unknown> = {
	// …
	"@earendil-works/pi-agent-core": bundledPiAgentCore,
	// …
	"@earendil-works/pi-ai": bundledPiAiCompat,
	// …
};
```

`extensions/loader.ts` 加载扩展时，会把这些包名都解析到 coding-agent 自带的那一份：编译成单文件二进制或从源码运行时用这张 `VIRTUAL_MODULES` 表，npm 安装后的 Node 模式则用 `getAliases()` 生成的 jiti 别名，指向同一批入口。于是扩展写 `import { Agent } from "@earendil-works/pi-agent-core"` 时，拿到的就是 coding-agent 自带的那份 L2，而且是全部导出，连根入口导出的 `AgentHarness` 也在里面。同理，扩展 import pi-ai 根入口时拿到的是 `compat` 入口。

所以要分开说：**coding-agent 自己对 L2 的依赖只有 3 个值；但它作为扩展的宿主，把 L1 和 L2 的全部 API 都暴露了出去**。对扩展作者来说，L2 的整个导出面都是公开契约的一部分。

### 2.2 L2 → L1：只借纯函数，不调模型

`pi-agent-core` 的顶层文件（排除 `harness/`）从 pi-ai 导入的值是 `normalizeContext`、`validateToolArguments`、`toToolDeclaration`、`EventStream`、`getCurrentTools`、`getToolStateChanges`、`createInitialSystemMessage`、`getCurrentSystemMessage`、`getCurrentSystemPrompt`、`parseStreamingJson`、`uuidv7`。全部来自 pi-ai 的根入口，没有一个来自 `providers/` 或 `compat`。

真正调模型的那一下，走的是外面注入进来的 `streamFn`。`agent-loop.ts` 的 `streamAssistantResponse` 把顺序写得很清楚：

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

### 2.3 L3 → L1：跨层是允许的

稳定版 coding-agent 有 26 个文件直接值导入 pi-ai（包括上面的 `virtual-modules.ts`），例如：

- `core/compaction/compaction.ts` 用 `completeSimple` 直接调模型生成摘要，不经过 `Agent`
- `core/agent-session.ts` 用 `retryDelayMs`、`getCurrentSystemMessage`
- `core/sdk.ts` 用 `streamSimple`、`clampThinkingLevel`

这说明 Pi 的分层规则是**单向**，而不是**只能依赖相邻层**。L1 的类型和工具函数（消息、模型、重试策略）本来就是所有上层共用的词汇。

> 🧠 **思考题**：既然 L3 可以直接调 L1，为什么生成压缩摘要不走 `Agent`？
> 因为压缩只是"给一段文本，要一段摘要"，不需要工具、不需要循环、不需要事件。走 `Agent` 反而要构造一个临时状态再丢掉。需要什么能力就依赖哪一层，这正是单向依赖相比"只依赖相邻层"多出来的自由。

---

## 3 · 类型怎么逐层加料：三条线，三种手法

### 3.1 工具线：extends 一次，然后另起炉灶

![图 3 · Tool → AgentTool → ToolDefinition](/images/harness/pi/ch02/fig3-tool-types.png)

- **L1 `Tool`**（`ai/src/types.ts:715`）：`name`、`description`、`parameters`、`constrainedSampling?`。只有声明，没有 `execute`。
- **L2 `AgentTool extends Tool`**（`agent/src/types.ts:464`）：新增 `label`、`execute(id, params, signal?, onUpdate?)`、`prepareArguments?`、`outputSchema?`、`executionMode?`、`replay?`。
- **L3 `ToolDefinition`**（`coding-agent/src/core/extensions/types.ts:565`）：**不 extends**，把 AgentTool 的字段（`replay?` 除外）重新声明了一遍。新增 `promptSnippet?`、`promptGuidelines?`、`renderCall?`、`renderResult?`、`renderShell?`、`exposure?`、`namespace?`、`annotations?`、`defaultActive?`、`prepareLoadout?`；`execute` 多了第 5 个参数 `ctx: ExtensionToolContext`。

L3 的工具交给 L2 之前，要经过 `wrapToolDefinition()`（`core/tools/tool-definition-wrapper.ts:8`）：

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

转换时只拷贝 L2 认识的字段，并把 `ctx` 闭包进 `execute`。`prompt*` 字段留给 L3 自己拼系统提示词，`render*` 字段留给 TUI 渲染。L2 永远看不到这些字段，也就不会被它们牵连。

### 3.2 消息线：声明合并上去，convertToLlm 降回来

L2 在 `agent/src/types.ts:365` 留了一个空接口：

```typescript
export interface CustomAgentMessages {
	// Empty by default - apps extend via declaration merging
}

// …
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

L3 在 `core/messages.ts:70` 用 TypeScript 的声明合并往里塞了 4 种消息：

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

于是在 coding-agent 里，`AgentMessage` 有 8 种 `role`：pi-ai 的 `system` / `user` / `assistant` / `toolResult`，加上这 4 种。引擎对它们一视同仁地存储和转发，**只在调模型前**通过 `convertToLlm`（`core/messages.ts:148`）把它们降回 LLM 能理解的 `Message`：

| 自定义消息 | 发给 LLM 时变成 |
|---|---|
| `bashExecution` | `user` 消息：命令加输出代码块。用 `!!` 前缀执行的（`excludeFromContext`）直接丢弃 |
| `custom` | `user` 消息，字符串内容转成 text 块 |
| `branchSummary` | `user` 消息，摘要前后加上固定的前缀和后缀 |
| `compactionSummary` | `user` 消息，同上 |
| `system` / `user` / `assistant` / `toolResult` | 原样保留 |

函数末尾的 `default` 分支用 `const _exhaustiveCheck: never = m` 做穷尽检查：以后 L3 再加一种消息却忘了处理，编译就会失败。

### 3.3 事件线：包一层，再并上新事件

- **L1 `AssistantMessageEvent`**（`ai/src/types.ts:767`）：12 种。`start`；`text_*`、`thinking_*`、`toolcall_*` 各 3 种（start / delta / end）；`done`；`error`。
- **L2 `AgentEvent`**（`agent/src/types.ts:514`）：10 种。`agent_*`、`turn_*`、`message_*`、`tool_execution_*`。L1 的事件只出现在 `message_update.assistantMessageEvent` 里。
- **L3 `AgentSessionEvent`**（`core/agent-session.ts:190`）：把 `agent_end` 换成带 `willRetry` 的版本，给 `tool_execution_*` 加上 `parentToolCallId?`，另外新增 13 种：`agent_settled`、`queue_update`、`compaction_start/end`、`entry_appended`、`session_info_changed`、`thinking_level_changed`、`auto_retry_start/end`、`summarization_retry_*` 3 种、`bash_execution_update`。

三条线放在一起看：**工具线靠转换函数隔离，消息线靠声明合并扩展，事件线靠并集扩展**。共同点是下层的类型从不为上层修改。

---

## 4 · L3 怎么把业务塞进 L2

![图 4 · 两批注入点](/images/harness/pi/ch02/fig4-injection.png)

`Agent` 对"编码"一无所知。L3 的业务分两批注入进去。

**第一批：构造参数。** `core/sdk.ts:387` 创建 `Agent` 时传入（节选）：

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
	// steeringMode、followUpMode、transport、thinkingBudgets、maxRetryDelayMs
});
```

注意 `tools: []` 和 `systemPrompt: ""`：工具和提示词都还没定，要等扩展加载完才知道。

**第二批：构造后赋值。** `AgentSession` 的构造函数订阅完事件后，连着调用六个 `_install*()`，直接改写 `Agent` 的公开字段：

| 方法 | 改写的字段 | 做什么 |
|---|---|---|
| `_installAgentToolHooks` | `beforeToolCall` `afterToolCall` | 派发扩展的 tool_call（可拦截）和 tool_result（可改结果） |
| `_installAgentNextTurnRefresh` | `prepareNextTurnWithContext` | 下一轮开始前刷新状态 |
| `_installAgentRequestProjection` | `prepareRequest` | 每次请求前从会话树重新投影上下文 |
| `_installAgentBoundaryHooks` | `finishTurn` | turn_end 时派发扩展事件，扩展可要求续跑 |
| `_installHiddenDeclarationsProjection` | `transformContext` | 再包一层：去掉 prepareLoadout 要求隐藏的工具声明 |
| `_installAgentForcedPromptProjection` | `transformContext` | 再包一层：强制提示词投影 |

除了 `_installAgentToolHooks` 直接覆盖两个工具钩子，其余五个方法都先保存原来的钩子（`const previousX = this.agent.x`），再用新函数包住它。所以同一个钩子可以层层叠加，引擎完全不知道外面包了几层。

**事件向上。** `AgentSession._handleAgentEvent`（`agent-session.ts:1068`）收到 `AgentEvent` 后：

```typescript
// Emit to extensions first, then notify public listeners.
await this._emitExtensionEvent(event);
this._emit(event.type === "agent_end" ? { ...event, willRetry: this._willRetryAfterAgentEnd(event) } : event);

// Handle session persistence
```

先发给扩展，再发给 `subscribe()` 的订阅者（TUI、JSON 模式、RPC、SDK 调用方），最后落盘。

---

## 5 · 正在被替换的中间层

![图 5 · L2 的位置上现在有四套引擎](/images/harness/pi/ch02/fig5-engines.png)

前四节描述的都是**已发布**的路径。把整个仓库都算进来，L2 这个位置上并存着四套引擎：

1. **`Agent` + `agentLoop`**：`pi-agent-core` 顶层文件，约 2.7k 行。已发布路径上的引擎，`pi` CLI 和 SDK 用的就是它。
2. **`AgentHarness`**（实现类 `Harness`）：`pi-agent-core/src/harness/runtime/` 等，`runtime/` 7.5k 行、`session/` 7.1k 行……从根入口导出，没有实验标记，但目前只有 `coding-agent/src/experimental/` 下的 mini、session-worker、services 在用。
3. **pico3 `Harness`**：`pi-agent-core/src/harness/pico3/`，约 8.1k 行，通过子路径 `./experimental/pico3` 导出，`experimental/micro` 在用。
4. **pi-durable `Harness`（Pico5）**：`packages/durable/src/`，约 17.7k 行。README 写明 "The API changes without notice between releases"。`experimental/durable` 在用，它是 2026-10-01 的提交 `5609b0d6c` 新增的。

几个容易被"三层"这个说法遮住的事实：

**`pi-agent-core` 包的主体不是 `Agent`。** 这个包的 `src/` 共 33,588 行，其中 `harness/` 有 30,899 行（92%），顶层的 `Agent` 和 `agentLoop` 这部分约 2.7k 行。`harness/` 里有自己的 bash/read/edit/write 工具、会话存储、压缩、Skills 和系统提示词，跟 L3 的 `core/tools/`、`core/compaction/` 是**同名的两套实现**。稳定路径用的是 L3 那套：coding-agent 自己的稳定代码不调用 `harness/` 里的任何值，只是通过 §2.1 的虚拟模块把它们原样转交给扩展。

**`pi-durable` 不依赖 `pi-agent-core`。** 它的 workspace 依赖只有 `pi-ai` 和 `chord`（另外两个是 `diff` 和 `typebox`）。走这条路时，依赖链变成 `pi-ai → pi-durable → coding-agent/experimental/durable`，中间层整个换掉了。它的 README 在实验性警告之后，正文第一段是："A durable agent harness. Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown."

**pico3 被 Pico5 当作参考。** `packages/durable/docs/pico-v5-handoff.md` 原文是："Pico3 is reference material only. Preserve useful behavior, not its capability facades, membranes, document routing, view projection, events, or clone chains." 同一份文档的状态一节写着 "Packages 1–23 are implemented in `packages/durable`"。

**最近的开发集中在实验引擎上。** 截至 2026-10-01 的最近 60 天：

| 路径 | 提交数 |
|---|---|
| `packages/agent/src/agent-loop.ts` | 9 |
| `packages/agent/src/agent.ts` | 7 |
| `packages/agent/src/harness/` | 229 |
| `packages/durable/`（首个提交 2026-09-18） | 76 |
| `packages/coding-agent/src/experimental/` | 87 |

**发布边界把实验代码挡在外面，但只挡了一半。** coding-agent 的 `package.json` 用 `files` 排除了 `dist/experimental`，所以 npm 用户拿到的 `pi` 只跑第一套引擎。`pi-agent-core` 则会把 `harness/` 一起发布：根入口有 `export * from "./harness/agent-harness.ts"`，`exports` 里还有 `./experimental/pico3`、`./harness/session` 等子路径。

所以第二章的答案是：**三层作为依赖方向的规则依然成立，四套引擎都往下依赖 pi-ai，没有一套依赖 coding-agent；但"L2 = pi-agent-core 的 Agent"这个等式，只对已发布的路径成立**。本系列接下来的 Agent Loop、工具、事件章节，讲的都是第一套引擎。实验引擎会在它们稳定下来后单独成章。

---

## 6 · 按需取层：每一层都能单独用

单向依赖带来的直接好处是：可以只拿走下面几层。仓库里每种组合都有现成的代码：

| 你拿走的层 | 现成代码 | 说明 |
|---|---|---|
| 只要 L1 | `packages/ai/README.md` 的 Quick Start | 只调模型，见第一章 §4.1 |
| L1 + L2 | `scripts/agent-treeshake-smoke-entry.ts` | 13 行，`check:browser-smoke` 打的三个包之一 |
| L1 + L2 + 能力库 | `packages/agent/examples/mcp-codemode/` | `Agent` 加 `pi-mcp` 和 `pi-codemode`，不带 coding-agent |
| L1 + L2 不要 `Agent` 类 | `agentLoop()` / `agentLoopContinue()` | 见 `packages/agent/README.md` 的 Low-Level API |
| 全部三层 | `packages/coding-agent/examples/sdk/01-minimal.ts` … `14-codemode-mcp.ts` | `createAgentSession()` |

L1 + L2 最小的形态就是那个打包检查的入口，原文如下：

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

它同时是一条 CI 断言：这样用 `Agent`，打包产物里不会混进其他供应商和 compat 层。

> ⚠️ 不用 `Agent` 类、直接调 `agentLoop()` 时，README 提醒这些流只负责观察："they do not wait for your async event handling to settle before later producer phases continue"。如果你的事件处理需要在工具执行前完成，就用 `Agent`。

---

## 7 · 三个可以带走的方法

1. **用值导入衡量接缝，而不是用 package.json。** manifest 只说明"允许依赖"。数一数上层实际导入了下层的哪些值，接缝的宽度就一目了然。Pi 的 L3 自己只用 L2 的 3 个值，这说明换引擎时 coding-agent 内部要改的地方很少。但也要把命名空间导入算进去：虚拟模块把 L2 的全部导出交给了扩展，这部分同样是契约。
2. **上层加料时先降级再下传。** `wrapToolDefinition` 和 `convertToLlm` 是同一个模式：上层可以在自己的类型上随意加字段，交给下层前转换成下层认识的形状。下层的类型因此从不需要为上层修改。
3. **把规则写进 CI。** 依赖方向靠 `check-runtime-deps`，入口体积靠 `check-entry-graphs`，可拆性靠 `check-browser-smoke`。分层能长期保持，靠的是违反规则时 CI 会失败。

---

## 8 · 关键数字

| 指标 | 数值 |
|---|---|
| L3 稳定代码从 L2 导入的运行时值 | 3 个具名值（`Agent`、`setDefaultStreamFn`、`runToolCall`）+ 1 个命名空间导入（交给扩展） |
| L3 稳定代码中直接值导入 pi-ai 的文件 | 26 个 |
| 反向 import（含 `import type`） | 0 |
| `npm run check` 步骤 | 9 步，其中 3 步与架构相关 |
| coding-agent 中 `AgentMessage` 的 role | 8 种（pi-ai 4 种 + 声明合并 4 种） |
| 事件类型 | L1 12 种 → L2 10 种 → L3 新增 13 种 |
| `AgentSession` 构造时安装的钩子 | 6 个 `_install*()` |
| `pi-agent-core` 的 `src/` | 33,588 行，其中 `harness/` 30,899 行（92%） |
| `pi-durable` 的 `src/` | 约 17.7k 行 |
| L2 位置上的引擎 | 4 套（1 套在已发布路径上，3 套只被实验代码使用） |

---

## 9 · 术语表

| 术语 | 定义 | 别和它混淆 |
|---|---|---|
| **值导入** | 编译后仍然存在的 import，代表真实的运行时依赖 | 类型导入：`import type`，编译后被擦除 |
| **接缝** | 上层在运行时实际用到的下层 API 集合；命名空间导入按全部导出计算 | `package.json` 的依赖声明 |
| **虚拟模块** | `virtual-modules.ts` 里的模块表，扩展 import 的 pi 包都从这里解析 | 扩展自己 node_modules 里的包 |
| **声明合并** | TypeScript 允许在别的模块里给同名接口追加成员，L3 借此扩展 `AgentMessage` | 继承（extends） |
| **`convertToLlm`** | 每次调模型前，把 `AgentMessage[]` 降成 `Message[]` | `transformContext`：在它之前运行，只改写 `AgentMessage[]` |
| **`wrapToolDefinition`** | 把 L3 的 `ToolDefinition` 转成 L2 的 `AgentTool`，丢掉提示词和渲染字段 | `createToolDefinitionFromAgentTool`：反方向 |
| **AgentHarness** | `pi-agent-core/harness/` 里的另一套引擎接口，从根入口导出，目前只有实验代码在用 | `Agent`：已发布路径上的引擎 |
| **Pico5 / pi-durable** | 独立包里的持久化 Agent harness，状态先落盘再展示 | pico3：agent-core 里的上一代实现，被 Pico5 当作参考 |

---

## 10 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 依赖方向怎么检查 | 根 `package.json` 的 `check`；`scripts/check-runtime-deps.mjs` |
| 入口体积预算 | `scripts/check-entry-graphs.mjs` 的 `BUDGETS` |
| L3 怎么创建引擎 | `packages/coding-agent/src/core/sdk.ts` → `new Agent` |
| L3 怎么注入钩子 | `packages/coding-agent/src/core/agent-session.ts` → 构造函数和 `_install*` |
| 工具类型怎么转换 | `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` |
| 自定义消息怎么降级 | `packages/coding-agent/src/core/messages.ts` → `convertToLlm` |
| 引擎的调用顺序 | `packages/agent/src/agent-loop.ts` → `streamAssistantResponse` |
| 实验引擎 | `packages/agent/src/harness/`；`packages/durable/README.md`；`packages/durable/docs/pico-v5.md` |

**下一章**：钻进第一套引擎的心脏——`agent-loop.ts` 的双层循环、steering 与 follow-up、工具的串行与并行执行。
