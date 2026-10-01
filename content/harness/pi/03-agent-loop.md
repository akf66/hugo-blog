---
title: "Pi 源码分析 03 · Agent Loop：一次运行是怎么转起来的"
date: 2026-10-01T19:30:00+08:00
description: "接着第二章往下讲：当前发布的 pi-agent-core 引擎里，双层循环怎么继续和停下、一轮里钩子和事件的先后顺序、工具的串行与并行以及各种失败的去向，还有 Agent 类和直接调用 agentLoop() 的区别。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` main 分支 `v0.99.2-29-g0f8740bb6`（2026-10-01），commit `0f8740bb6`。本章涉及的 `agent-loop.ts`、`agent.ts`、`types.ts` 与 npm 上发布的 `@earendil-works/pi-agent-core@0.99.2` 完全一致。文中所有行为、数字和代码均以该版本为准。

第二章讲到，L3 在创建 `Agent` 时传入一批钩子，又在创建后改写了另一批。本章接着往下讲，打开 L2 当前发布的引擎，看这些钩子在一次运行里什么时候被调用，循环靠什么继续、什么时候停下。

<!--more-->

本章要点：

- **两层循环**：内层在"还有工具结果要交给模型"或"有插队消息"时继续；外层在内层停下后检查后续消息。`finishTurn` 还能要求再跑一轮。
- **一轮有固定的顺序**：先注入待处理的消息，再依次调用 `prepareRequest`、`transformContext`、`convertToLlm`、`streamFn`，然后执行工具，最后是 `finishTurn` 和 `turn_end`。
- **工具调用失败了，也会变成一条结果交回模型**：找不到工具、参数不合法、被拦截、执行抛错，都变成一条 `isError: true` 的工具结果，循环照常往下走。
- **`Agent` 和 `agentLoop()` 跑的是同一个循环**，区别只在事件出口：`Agent` 会等订阅者处理完，`agentLoop()` 不等。

---

## 0 · 阅读说明

- 本章只讲当前发布的引擎：`Agent` 类和 `agentLoop()` 系列函数。第二章提到的 `harness/` 和 `pi-durable` 两套新引擎不在本章范围内。
- 文中的事件顺序都用一个探针脚本实际跑过。脚本用一个假的 `streamFn` 按剧本返回助手消息，订阅 `Agent` 的事件并记录钩子的调用时刻。本章引用的运行记录都来自这个脚本。
- 后文说的"一轮"（turn），指一次模型请求加上这条回复里所有工具调用的执行，和 `turn_start` / `turn_end` 两个事件对应。

---

## 1 · 三个入口，一个循环

pi-agent-core 对外提供三种启动方式：

- `Agent.prompt()` / `Agent.continue()`（有状态的引擎类：自己保存消息、管理队列，等订阅者处理完事件）
- `agentLoop()`（给一组新消息，返回一个可以 `for await` 的事件流）
- `agentLoopContinue()`（不加新消息，从已有上下文继续，常用于重试）

它们最后都进入同一个内部函数 `runLoop`（双层循环的本体）。中间隔着一层 `runAgentLoop` / `runAgentLoopContinue`（发出 `agent_start` 和第一个 `turn_start`，把提示消息加进上下文），这两个函数也是导出的。

```typescript
export async function runAgentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): Promise<AgentMessage[]> {
	// …
	await emit({ type: "agent_start" });
	await emit({ type: "turn_start" });
	for (const message of initialMessages) {
		await emit({ type: "message_start", message });
		await emit({ type: "message_end", message });
	}

	await runLoop(currentContext, newMessages, config, signal, emit, streamFn ?? getDefaultStreamFn());
	return newMessages;
}
```

循环拿到的所有输入都在 `AgentLoopConfig`（循环配置）里。除了模型和请求选项，主要是这几类函数：

- **请求前**：`prepareRequest`（每次请求前替换上下文、模型、推理强度）、`transformContext`（改写引擎消息列表）、`convertToLlm`（把引擎消息转成模型能理解的消息，唯一必填的函数）、`getApiKey`（每次请求前取一次密钥，适合会过期的令牌）
- **工具**：`toolExecution`（串行还是并行，默认并行）、`beforeToolCall`（执行前检查，可以拦截）、`afterToolCall`（执行后改写结果）
- **轮次边界**：`finishTurn`（一轮结束时决定继续、结束还是照常）、`prepareNextTurn`（下一轮开始前换上下文、模型，或追加消息）
- **队列**：`getSteeringMessages`（取插队消息）、`getFollowUpMessages`（取后续消息）

`streamFn`（模型调用函数）不在配置里，单独作为参数传入。

---

## 2 · 双层循环

![图 1 · runLoop 的双层循环](/images/harness/pi/ch03/fig1-two-loops.png)

`runLoop` 的骨架如下（省略了每一步的细节）：

```typescript
// Check for steering messages at start (user may have typed while waiting)
let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

// Outer loop: continues when queued follow-up messages arrive after agent would stop
while (true) {
	let hasMoreToolCalls = true;

	// Inner loop: process tool calls and steering messages
	while (hasMoreToolCalls || pendingMessages.length > 0) {
		// …第二轮起：prepareNextTurn、turn_start…
		// …注入 pendingMessages，prepareRequest，请求模型，执行工具…
		const decision = await config.finishTurn?.(lastCompletedTurn, signal);
		await emit({ type: "turn_end", message, toolResults });

		if (decision?.action === "end") {
			await emit({ type: "agent_end", messages: newMessages });
			return;
		}

		explicitContinuation = decision?.action === "continue";
		pendingMessages = (await config.getSteeringMessages?.()) || [];
		// …
	}

	// Agent would stop here. Check for follow-up messages.
	const followUpMessages = (await config.getFollowUpMessages?.()) || [];
	if (followUpMessages.length > 0) {
		// Set as pending so inner loop processes them
		explicitContinuation = false;
		pendingMessages = followUpMessages;
		continue;
	}

	// No natural request was selected, so fulfill the continuation decision with one context-only turn.
	if (explicitContinuation) {
		explicitContinuation = false;
		continue;
	}

	// No more messages, exit
	break;
}

await emit({ type: "agent_end", messages: newMessages });
```

### 2.1 内层：工具结果和插队消息

内层每转一圈就是一轮。继续的条件有两个：

- **还有工具调用**：这条助手回复里有工具调用，执行完的结果要交回模型，所以必须再请求一次。只有一种情况例外：这一批工具结果全部带有 `terminate: true`（见 4.4 节）。
- **有插队消息**：每轮结束后调用一次 `getSteeringMessages`。取到消息，就在下一次请求之前把它们加进上下文。

插队消息（steering）是给"运行中途改主意"准备的。用户在 Agent 干活时输入"停，换个做法"，这条消息不会打断正在执行的工具，而是等这一轮的工具全部执行完、`turn_end` 发出之后才被取出，下一次请求时模型就能看到。

### 2.2 外层：后续消息

内层停下，说明模型这次的回复没有工具调用，也没有人插队，Agent 本来要结束了。这时外层调用一次 `getFollowUpMessages`。取到后续消息（follow-up），就把它们当作待处理消息，再进入内层。

两种队列的区别在于什么时候被取出：

| 队列 | 什么时候取 | 适合 |
|---|---|---|
| 插队消息 | 每轮结束后 | 中途纠偏 |
| 后续消息 | Agent 本来要停时 | 做完再说的事 |

探针里，在第一个工具开始执行时同时排进一条插队消息和一条后续消息。运行记录显示：插队消息在工具批次之后的第二次请求前进入上下文，后续消息要等第二次请求的回复结束、内层停下之后，才在第三次请求前进入。

### 2.3 finishTurn 的三种决定

`finishTurn`（一轮结束时调用，在 `turn_end` 之前）返回 `AgentTurnDecision`（这一轮之后怎么走）：

- `undefined`（照常：按工具结果和队列决定是否继续）
- `{ action: "end" }`（立刻结束：发出 `turn_end` 和 `agent_end`，不再取插队和后续消息）
- `{ action: "continue" }`（保证至少还有一次请求）

`continue` 的含义是"至少再请求一次"，不会额外多发请求。如果工具结果、插队消息或后续消息本来就会触发下一次请求，这个决定就算兑现了。只有在都没有的时候，外层才会用当前上下文再跑一轮，也就是图 1 里 06 那条路径。

> ⚠️ `finishTurn` 每轮都会被调用。每次都返回 `continue` 就会无限循环，必须带上停止条件。

### 2.4 两个出口之外的提前退出

除了正常结束，循环还有两个提前出口：

- **助手消息的 `stopReason` 是 `error` 或 `aborted`**：不执行这条消息里的工具调用。`finishTurn` 照常调用，但返回的决定会被忽略，随后直接发出 `turn_end` 和 `agent_end`。
- **`finishTurn` 返回 `end`**：见上一小节。

> 📌 `Agent` 的两个队列默认都是 `one-at-a-time`（每次只取最早的一条），另一种模式是 `all`（一次取完）。按默认设置，连续排进三条插队消息，会分三轮依次注入，每轮一条。

---

## 3 · 一轮的生命周期

![图 2 · 一轮里钩子和事件的先后顺序](/images/harness/pi/ch03/fig2-turn-order.png)

一轮里各个钩子的调用和事件的发出，严格按下面的顺序进行：

1. **准备下一轮**（第二轮起）：调用 `prepareNextTurn`，它可以替换上下文、模型、推理强度，或者返回要追加的消息。如果此时手里还没有待处理的消息，就再取一次插队消息，因为准备工作可能很慢（例如压缩），期间用户可能又输入了内容。然后发出 `turn_start`。
2. **注入待处理消息**：先是 `prepareNextTurn` 追加的消息，再是这次取到的队列消息（插队消息或后续消息，同一轮里只会有其中一种）。每条都发出 `message_start` 和 `message_end`，然后加进上下文。如果可执行的工具和对话里已经声明的工具不一致，这里还会插入一条 system 消息（放在第一条非 system 消息之前），向模型宣告新增和移除的工具。
3. **`prepareRequest`**：这时待处理消息已经入列。它返回的上下文、模型、推理强度，会替换这一次和之后的请求所用的值。它不会去取队列。
4. **`transformContext` → `convertToLlm`**：先在引擎消息层面改写（裁剪历史、注入外部上下文），再转成模型能理解的消息。
5. **`streamFn`**：取到密钥后发出请求。流式返回的过程中，依次发出 `message_start`、若干 `message_update`、`message_end`。
6. **执行工具**：见第 4 节。每个调用都有 `tool_execution_start` / `tool_execution_update` / `tool_execution_end`，最后每条工具结果都发出 `message_start` / `message_end`。
7. **`finishTurn`**：这时助手消息和所有工具结果都已经入列。
8. **`turn_end`**：带上这条助手消息和本轮的工具结果。
9. **取插队消息**：调用 `getSteeringMessages`，结合工具结果决定内层是否继续。

第一轮和后面的轮次有一点不同：第一轮的 `turn_start` 和提示消息的事件，是在进入 `runLoop` 之前由 `runAgentLoop` 发出的，所以第一轮没有第 1 步。

探针里第一轮和第二轮的记录如下（`ev` 开头的是订阅者收到的事件，其余是钩子被调用的时刻）：

```text
  ev agent_start
  ev turn_start
  ev message_start(user)
  ev message_end(user)
prepareRequest
transformContext
convertToLlm
streamFn#0
  ev message_start(assistant)
  ev message_end(assistant)
  ev tool_execution_start(slow)
beforeToolCall(slow)
…（工具执行，见第 4 节）
finishTurn#1
  ev turn_end
prepareNextTurn
  ev turn_start
  ev message_start(user)        ← 插队消息
  ev message_end(user)
prepareRequest
transformContext
convertToLlm
streamFn#1
```

> 💡 `prepareRequest` 和 `transformContext` 都能改上下文，作用范围不同：`prepareRequest` 返回的上下文会替换循环里保存的那一份，影响之后的所有请求；`transformContext` 的结果只用于这一次请求，循环里保存的上下文不变。第二章讲过 coding-agent 怎么用它们：前者从会话树重新算出上下文，后者交给扩展的 `context` 事件。

> 📌 `convertToLlm` 要保留 `system` 消息。提示词和工具声明都放在 system 消息里，`Agent` 默认的转换函数会保留 `system`、`user`、`assistant`、`toolResult` 四种角色。自己写转换函数时，如果把 `system` 也过滤掉，模型就看不到提示词和工具声明了。

---

## 4 · 工具执行

### 4.1 串行还是并行

![图 3 · 同一批三个工具调用：串行与并行](/images/harness/pi/ch03/fig3-sequential-parallel.png)

选择逻辑只有几行：

```typescript
const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
const hasSequentialToolCall = toolCalls.some(
	(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
	return executeToolCallsSequential(currentContext, assistantMessage, toolCalls, config, signal, emit);
}
return executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit);
```

- 默认是并行。
- 全局配置 `toolExecution: "sequential"`，或者这一批里只要有一个工具声明了 `executionMode: "sequential"`（必须单独执行），**整批**都改为串行。

两种模式的执行过程：

- **串行**：每个调用依次经过准备、执行、收尾，结果消息发出之后，下一个调用才开始。
- **并行**：先按顺序逐个准备（查找、校验、`beforeToolCall`），全部准备完之后，再同时执行所有通过的调用。每个调用执行完就立刻发出自己的 `tool_execution_end`，所以 end 事件按**完成顺序**发出；工具结果消息要等全部执行完，再按助手消息里的**原顺序**发出。

探针里，一条助手消息带了 6 个工具调用：`slow`（30ms）、`fast`（5ms）、一个不存在的工具、一个参数不合法的调用、一个会被拦截的调用、一个等 5ms 后抛错的调用。并行模式下，前三个失败的调用在准备阶段就发出了 end；真正执行的三个，end 的记录如下：

```text
  …
  ev tool_execution_end(fast: rewritten)
  ev tool_execution_end(thr!: negative!)
  ev tool_execution_end(slow: slow ok)
  ev message_start(toolResult:slow)
  ev message_start(toolResult:fast)
  ev message_start(toolResult:nope!)
  …
```

`slow` 最慢，最后发出 end，但结果消息仍然从 `slow` 开始排。这样，模型下一次看到的结果顺序和它发出调用的顺序一致，和工具的快慢无关。

### 4.2 beforeToolCall 和 afterToolCall

**`beforeToolCall`**（执行前检查）在参数校验通过之后调用，收到 `BeforeToolCallContext`：

- `assistantMessage`（发出这个调用的助手消息）
- `toolCall`（原始的工具调用块）
- `args`（校验过的参数）
- `context`（当前的引擎上下文）

它返回 `BeforeToolCallResult`：

- `block`（为 true 时不执行这个工具）
- `reason`（拦截原因，会作为错误结果的文本交给模型；不填时是 `Tool execution was blocked`）
- `terminate`（拦截后建议本批结束就停，见 4.4 节）

它还能改参数：`args` 和之后传给 `execute` 的是同一个对象，在钩子里原地修改，工具拿到的就是修改后的值，而且**不会再校验一次**。仓库里有一个测试专门固定这个行为（"should execute mutated beforeToolCall args without revalidation"）。

**`afterToolCall`**（执行后改写结果）在工具执行完、发出 `tool_execution_end` 之前调用。它比 `beforeToolCall` 多收到 `result`（执行结果，任何改写之前的原样）和 `isError`（当前是否算作失败）。返回的 `AfterToolCallResult` 按字段覆盖：

- `content`（整个替换发给模型的内容）
- `details`（整个替换给界面和日志用的数据）
- `isError`（替换失败标记）
- `usage`（替换工具自身的用量统计）
- `terminate`（替换提前结束的标记）
- `structuredContent`（替换结构化结果）

没返回的字段保持原值，也没有深合并。有一个例外：只替换了 `content` 而没给 `structuredContent` 时，原来的结构化结果会被丢掉，因为它可能已经和新内容对不上了。

两个钩子都会收到中止信号，是否响应由钩子自己负责。

### 4.3 每一种失败的去向

![图 4 · 一个工具调用经过的关卡，以及每一关失败时的结果](/images/harness/pi/ch03/fig4-tool-outcomes.png)

一个工具调用依次经过 6 道关卡（`beforeToolCall` 这一关，拦截和抛错都算失败）。无论在哪一关失败，结果都是一条 `isError: true` 的工具结果消息，模型在下一次请求时能看到失败原因并自行调整：

| 关卡 | 失败时交给模型的文本 |
|---|---|
| 找不到工具 | `Tool <name> not found` |
| 参数修补或校验失败 | 校验错误的原文 |
| `beforeToolCall` 拦截 | `reason`，或默认文本 |
| `beforeToolCall` 抛错 | 错误信息 |
| 准备完时已中止 | `Operation aborted` |
| `execute` 抛错 | 错误信息 |
| `afterToolCall` 抛错 | 错误信息 |

补充几点：

- 前 4 关没通过时，工具不会执行，`afterToolCall` 也不会被调用。事件仍然完整：`tool_execution_start` 和 `tool_execution_end` 都会发出。在准备阶段失败的调用，end 紧跟在 start 之后；并行模式下，准备完才发现已中止的调用，要到执行阶段才发出 end，中间可能夹着其他调用的 start。
- 工具也可以不抛错，而是返回 `isError: true`。这时 `content` 作为错误交给模型，`details` 和 `structuredContent` 保留给界面和程序调用方。`AgentTool` 的注释要求失败时必须二选一，不要只在 `content` 里描述失败。
- 工具执行完之后再调用进度回调 `onUpdate`，引擎会直接忽略。
- 助手消息的 `stopReason` 是 `length`（输出达到 token 上限被截断）时，这一批工具调用**全部不执行**。截断的参数可能刚好能解析、能通过校验，但内容不完整，所以每个调用都会得到一条错误结果，提示模型用完整的参数重新调用。循环照常继续。

### 4.4 提前结束：terminate

工具结果、`beforeToolCall` 的拦截结果、`afterToolCall` 的改写，都可以带上 `terminate: true`，意思是"这批工具执行完就别再请求模型了"。判断规则在 `shouldTerminateToolBatch`（判断这一批是否提前结束）里：

```typescript
function shouldTerminateToolBatch(finalizedCalls: FinalizedToolCallOutcome[]): boolean {
	return finalizedCalls.length > 0 && finalizedCalls.every((finalized) => finalized.result.terminate === true);
}
```

必须这一批的**每一个**结果都带上 `terminate`，内层才不会因为工具结果而继续。只要有一个没带，就照常把结果交回模型。即使整批都带了 `terminate`，后面仍然会照常取插队和后续消息，有消息排着就继续跑。

> 📌 拦截一个工具调用，并不会结束整次运行。被拦截的调用会变成一条错误结果交回模型，模型通常会换一种做法再试。要让运行停下，可以在拦截时带上 `terminate: true`（整批都这样才生效），或者在 `finishTurn` 里返回 `{ action: "end" }`。

### 4.5 中止

调用 `Agent.abort()`（或者中止你传给 `agentLoop()` 的 signal）后，按中止发生的时机分几种情况：

- **请求模型时**：按 `StreamFn` 的约定，被中止的请求不抛错，而是返回一条 `stopReason: "aborted"` 的助手消息，循环从 2.4 节的提前出口退出。
- **准备工具时**：`beforeToolCall` 之后、执行之前各检查一次，已中止就返回 `Operation aborted`。并行模式下，后面还没准备的调用直接丢弃。
- **工具执行中**：signal 会传给 `execute`，工具要自己响应中止。
  - 串行模式下，当前调用结束后发现已中止，剩下的调用就不再执行，**也不会生成结果**。
  - 并行模式（默认）下，所有调用在执行前都已经准备好，同时启动；已中止的调用各自得到 `Operation aborted`，每个调用都有结果。

工具批次结束后，循环并不会马上退出，而是照常调用 `finishTurn`、发出 `turn_end`，进入下一轮。下一次请求带着已中止的 signal，`streamFn` 返回 `aborted`，循环这才结束。探针用串行模式，中止发生在第一个工具执行期间，记录是这样的：

```text
tool_execution_start(a) → tool_execution_end(a!: cancelled) → message_start/end(toolResult:a!)
→ turn_end → turn_start → streamFn#1 → message_start/end(assistant, aborted) → turn_end → agent_end
```

第二个调用 `b` 没有任何事件和结果。同样的场景换成并行模式，`b` 会得到一条 `Operation aborted` 的结果。串行模式下留下的这种调用，之后再把这段对话发给模型时，pi-ai 的消息转换会给这种没有结果的调用补一条 `No result provided` 的错误结果，让请求符合供应商的格式要求。

---

## 5 · Agent 类和直接调用 agentLoop() 的区别

![图 5 · 同一个循环，两种事件出口](/images/harness/pi/ch03/fig5-agent-vs-loop.png)

循环里每个事件都是 `await emit(event)` 发出的。区别在于传进来的 `emit` 函数是什么。

`agentLoop()` 传的是一个只往事件流里推的函数：

```typescript
const stream = createAgentStream();

void runAgentLoop(
	prompts,
	context,
	config,
	async (event) => {
		stream.push(event);
	},
	signal,
	streamFn,
).then((messages) => {
	stream.end(messages);
});

return stream;
```

`stream.push` 是同步的：把事件交给正在等待的消费者，或者先放进队列，然后立刻返回。所以循环不会等你的 `for await` 处理完一个事件再继续。

`Agent` 传的是自己的 `processEvents`（先更新状态，再逐个等待订阅者）：

```typescript
for (const listener of this.listeners) {
	await listener(event, signal);
}
```

订阅者按注册顺序依次执行，每一个都被等待。探针里，订阅者在助手消息的 `message_end` 上等待 50ms，再看工具什么时候开始执行：

- `Agent`：订阅者开始 → 订阅者结束 → `execute`
- `agentLoop()`：`execute` → 订阅者开始 → 订阅者结束

这正是 README 里说的：用 `Agent` 时，助手消息的 `message_end` 处理完，才会开始准备工具，所以 `beforeToolCall` 看到的状态里已经有了这条助手消息。

除了等不等订阅者，两者还有这些区别：

| 方面 | `Agent` | `agentLoop()` |
|---|---|---|
| 事件处理 | 逐个等待订阅者 | 推进事件流，不等待 |
| 消息与状态 | 自动维护 `state` | 调用方自己保存 |
| 插队与后续 | 内置两个队列 | 自己传两个取消息的函数 |
| 中止 | `abort()` | 自己传 signal |
| 注入的函数抛错 | 合成一条错误消息 | 未处理的拒绝 |
| 运行中再次调用 | `prompt()` 抛错 | 不限制 |

其中几项展开说明：

- **状态**：`Agent` 在 `message_end` 时把消息追加到 `state.messages`，在工具开始和结束时维护 `state.pendingToolCalls`（正在执行的工具调用 id），在 `turn_end` 时把助手消息的错误记到 `state.errorMessage`。`state.isStreaming`（是否在运行）要等 `agent_end` 的订阅者全部处理完才变回 false，`waitForIdle()`（等这次运行完全结束）也是到这时才返回。
- **抛错**：`convertToLlm`、`transformContext`、`getApiKey`、取消息的函数，按约定都不能抛错。真的抛了，`Agent` 会合成一条 `stopReason` 为 `error`（或 `aborted`）的助手消息，补发 `message_start`、`message_end`、`turn_end`、`agent_end`，`prompt()` 本身正常返回，错误放在 `state.errorMessage` 里。（订阅者自己抛错时不一样：补发事件时它会再抛一次，`prompt()` 会被拒绝。）`agentLoop()` 没有这层保护：内部的 Promise 被拒绝后没有人处理。在 Node 的默认设置下，这是一次未处理的拒绝，进程直接退出；如果宿主注册了 `unhandledRejection` 处理器，或者运行在浏览器里，`stream.end()` 永远不会被调用，`for await` 会一直等下去。
- **再次调用**：运行中调用 `prompt()` 会抛错，提示改用 `steer()`（排一条插队消息）或 `followUp()`（排一条后续消息）。`continue()`（从当前对话继续）要求最后一条消息不是助手消息；如果是，它会先尝试取一批插队消息，没有再取一批后续消息，都没有才抛错。
- **`prepareNextTurn` 的签名**：循环配置里的 `prepareNextTurn` 收到上一轮的完整信息。`Agent` 上有两个版本：`prepareNextTurn`（只收到 signal）和 `prepareNextTurnWithContext`（收到上一轮的信息和 signal）。两个都设置时用后者。

所以：需要状态、队列，或者需要事件处理完成后循环才往下走时，用 `Agent`；只想观察事件、自己管理一切时，可以直接用 `agentLoop()`，但要保证注入的函数不抛错。

---

## 6 · 用法：加一个钩子

### 6.1 引擎层：直接给 Agent 挂钩子

下面的例子做两件事：拦截 `rm -rf`；模型没有回复 `DONE` 时自动续跑，最多续跑 3 次。`Agent` 的钩子都是公开字段，创建后直接赋值即可，下一次运行就会生效。

```typescript
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6");
if (!model) throw new Error("Model not found");

const agent = new Agent({
	initialState: { systemPrompt: "You are a careful build assistant.", model, tools: [bashTool] },
	streamFn: models.streamSimple.bind(models),
});

// 1. 拦截：参数已经校验过，可以放心读取 args
agent.beforeToolCall = async ({ toolCall, args }) => {
	if (toolCall.name === "bash" && /\brm\s+-rf\b/.test((args as { command: string }).command)) {
		return { block: true, reason: "rm -rf 被策略禁止，请换一种做法" };
	}
};

// 2. 自动续跑：只在正常结束、没有工具调用的轮次判断
let nudges = 0;
agent.finishTurn = async ({ message, toolResults }) => {
	if (message.stopReason !== "stop" || toolResults.length > 0) return;
	const text = message.content.flatMap((c) => (c.type === "text" ? [c.text] : [])).join("");
	if (!text.includes("DONE") && nudges < 3) {
		nudges++;
		agent.followUp({ role: "user", content: "还没完成，请继续。全部完成后回复 DONE。", timestamp: Date.now() });
	}
};

await agent.prompt("清理构建目录，然后重新构建");
```

其中 `bashTool` 是你自己定义的 `AgentTool`。这段代码里的几个选择：

- **拦截不会结束运行**：被拦截的调用变成一条错误结果，`reason` 原样交给模型，模型通常会换一种做法。
- **续跑用 `followUp` 而不是 `{ action: "continue" }`**：这一轮的上下文以助手消息结尾，只返回 `continue` 会让循环用这份上下文再请求一次，模型收不到任何新的输入。在 `finishTurn` 里排一条后续消息，内层停下后外层会取到它，再跑一轮。
- **先排除 `error` 和 `aborted`**：这两种情况下 `finishTurn` 仍会被调用，但循环一定会退出，在这里排进的后续消息会留到下一次运行。
- **计数器是停止条件**：没有它，模型一直不回复 `DONE`，就会一直续跑下去。

用假的 `streamFn` 跑这段逻辑：第一次请求的 `rm -rf` 被拦截，模型收到的是拦截原因；后面两次回复没有 `DONE`，各续跑一次；第四次回复了 `DONE`，运行结束。一共 4 次请求，续跑 2 次。

### 6.2 产品层：通过 SDK 挂扩展钩子

用 coding-agent 的 SDK 时，`Agent` 的钩子已经被 `AgentSession` 接管（见第二章第 4 节），应该通过扩展事件来插手。`DefaultResourceLoader` 的 `extensionFactories`（内联扩展）可以在代码里直接写一个扩展：

```typescript
import {
	createAgentSession,
	DefaultResourceLoader,
	getAgentDir,
	isToolCallEventType,
	SessionManager,
} from "@earendil-works/pi-coding-agent";

const resourceLoader = new DefaultResourceLoader({
	cwd: process.cwd(),
	agentDir: getAgentDir(),
	extensionFactories: [
		(pi) => {
			pi.on("tool_call", async (event) => {
				if (isToolCallEventType("bash", event) && /\brm\s+-rf\b/.test(event.input.command)) {
					return { block: true, reason: "rm -rf 被策略禁止，请换一种做法" };
				}
				return undefined;
			});
		},
	],
});
await resourceLoader.reload();

const { session } = await createAgentSession({
	resourceLoader,
	sessionManager: SessionManager.inMemory(),
});
await session.prompt("清理构建目录，然后重新构建");
```

运行时，循环调用 `AgentSession` 装上的 `beforeToolCall`，它再把 `tool_call` 事件派发给扩展；扩展返回的结果原路带回，走的就是本章 4.2 节的拦截逻辑。几点说明：

- `isToolCallEventType`（按工具名收窄事件类型，`event.input` 随之有了准确的类型）。直接比较 `event.toolName === "bash"` 收窄不了，因为自定义工具的 `toolName` 是 `string`。
- 返回值 `{ block, reason, terminate }` 和引擎层的 `BeforeToolCallResult` 一一对应。要改参数，就原地修改 `event.input`。
- 扩展的 `turn_end` 事件对应引擎的 `finishTurn`，但写法不同：处理函数返回 `{ continue: true }`，而且 `AgentSession` 会先检查上下文能不能继续，例如对话以助手消息结尾、又没有排队的消息时，这个请求会被拒绝并报错。想自动续跑，也可以照 6.1 节的思路，在扩展里调用 `pi.sendUserMessage(text, { deliverAs: "followUp" })`（以后续消息的方式发一条用户消息）。运行中调用时，它经 `Agent.followUp` 排进后续队列。

---

## 7 · 三个可以带走的方法

1. **把调度决定集中在轮次边界上。** 除了模型出错或被中止时立即退出，循环继续还是停下，都在每轮结束时判断：工具结果、插队消息、后续消息、`finishTurn` 的决定，都在这里汇总。用户插话、工具出错不会打断正在执行的步骤，只会影响下一轮。
2. **让失败变成数据。** 工具的每一种失败都转成一条错误结果交回模型，模型请求的失败和中止也通过 `stopReason` 表达，而不是抛异常。大多数失败因此都走正常的事件序列，订阅者照常处理即可；抛异常只留给违反约定的情况。
3. **同一个循环，换一个事件出口。** `Agent` 和 `agentLoop()` 没有各写一套循环，只是传入的 `emit` 不同。要不要等订阅者，由调用方选择。

---

## 8 · 关键数字

| 指标 | 数值 |
|---|---|
| `agent-loop.ts` / `agent.ts` / `types.ts` | 940 / 613 / 529 行 |
| 循环的入口 | 3 种，共用一个 `runLoop` |
| `AgentEvent` 种类 | 10 种 |
| 工具调用的关卡 | 6 道，失败都变成错误结果 |
| `finishTurn` 的决定 | 3 种：照常、continue、end |
| 队列模式 | 2 种，默认每次取一条 |
| 默认工具执行模式 | 并行 |
| `agent-loop.ts` 的提交（2026-08-01 至 2026-10-01） | 9 次 |

---

## 9 · 术语表

| 术语 | 含义 |
|---|---|
| **轮（turn）** | 一次模型请求，加上这条回复里所有工具调用的执行 |
| **插队消息（steering）** | 运行中排进的消息，每轮结束后取出，下一次请求前注入 |
| **后续消息（follow-up）** | 运行中排进的消息，等 Agent 本来要停时才取出 |
| **`finishTurn`** | 每轮结束时调用，返回照常、continue 或 end |
| **`terminate`** | 工具结果上的标记；整批都带上时，不再因工具结果请求模型 |
| **`runLoop`** | 双层循环的本体，`Agent` 和 `agentLoop()` 共用 |
| **`emit`** | 循环发出事件的函数；`Agent` 的会等订阅者，`agentLoop()` 的不等 |

---

## 10 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 双层循环 | `packages/agent/src/agent-loop.ts` → `runLoop` |
| 一次请求的组装 | `packages/agent/src/agent-loop.ts` → `streamAssistantResponse` |
| 串行与并行 | `packages/agent/src/agent-loop.ts` → `executeToolCalls`、`executeToolCallsSequential`、`executeToolCallsParallel` |
| 工具调用的关卡 | `packages/agent/src/agent-loop.ts` → `prepareToolCall`、`executePreparedToolCall`、`finalizeExecutedToolCall` |
| 钩子的类型与约定 | `packages/agent/src/types.ts` → `AgentLoopConfig`、`BeforeToolCallResult`、`AfterToolCallResult`、`FinishTurn` |
| Agent 类 | `packages/agent/src/agent.ts` → `processEvents`、`createLoopConfig`、`runWithLifecycle` |
| 官方说明 | `packages/agent/README.md` 的 Event Flow、With Tool Calls、Request preparation and turn finalization |
| 行为测试 | `packages/agent/test/agent-loop.test.ts`、`packages/agent/test/agent.test.ts` |
| 扩展事件怎么映射到钩子 | `packages/coding-agent/src/core/agent-session.ts` → `_installAgentToolHooks`、`_installAgentBoundaryHooks` |
| 拦截示例 | `packages/coding-agent/examples/extensions/protected-paths.ts`、`permission-gate.ts` |
| 孤立工具调用的补全 | `packages/ai/src/api/transform-messages.ts` |

**下一章**：模型调用。看 `streamFn` 背后的 pi-ai 怎么把几十家供应商压成一套流式事件。
