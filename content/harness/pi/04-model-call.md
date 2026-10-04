---
title: "Pi 源码分析 04 · 模型调用：streamFn 背后发生了什么"
date: 2026-10-01T20:30:00+08:00
description: "接着第三章往下讲：Agent 调用的 streamFn 背后，pi-ai 怎么先找供应商、再选线协议，请求在进入供应商之前做了哪些转换，各家的原始流怎么归一成 12 种事件，失败、中止、重试和凭据刷新怎么落进流里，以及协议实现怎么做到按需加载。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.2-1-g200387122（2026-10-04），commit `200387122`。这个提交在 v1.0.2 之上只给各包的 CHANGELOG 加了 `[Unreleased]` 段，代码与 v1.0.2 相同。文中所有行为、数字和代码均以该版本为准。

第三章讲到，`runLoop` 每一轮把消息交给 `streamFn`，然后消费它返回的事件流。本章接着往下讲，打开 L1 的 pi-ai，从 `streamFn` 被调用开始，一直讲到供应商的原始流变成统一的事件。

<!--more-->

本章要点：

- **两级分派**：`Models` 先按 `model.provider` 找到供应商，供应商再按 `model.api` 选出协议实现。协议实现是 10 种线协议的共享模块，42 个供应商复用它们。
- **转换在协议实现内部完成**：每个协议实现在拼请求体时先调用 `transformMessages`，丢掉出错或中止的助手消息，给没有结果的工具调用补一条错误结果，把别家模型的思考块降成普通文本。
- **12 种事件，一个活对象**：各家的流都被归一成同一套事件：开头一个 `start`，文本、思考、工具调用各一组 `*_start / *_delta / *_end`，最后以 `done` 或 `error` 结束。事件里的 `partial` 始终指向同一个正在累积的消息，最后交出的也是这个对象。
- **失败都在流里**：经过 `Models` 的调用，从找不到供应商到 HTTP 500，再到模型拒答，都以一个 `error` 事件结束，不会抛到调用方。主要几种协议的请求级重试默认关闭，要显式打开。
- **协议实现按需加载**：只导入供应商不会加载任何 SDK，第一次请求时才 `import` 对应的协议实现和它的 SDK。

---

## 0 · 阅读说明

- 本章只讲已发布的 pi-ai（`@earendil-works/pi-ai`），而且只讲对话模型的流式请求。图像生成、分类模型、延迟返回（deferred）这几条支线只在需要时提一句。
- 文中的事件顺序、错误信息和计数都用探针脚本实际跑过。探针在本地起一个假的 HTTP 服务，按 Anthropic 和 OpenAI 的格式返回 SSE 流，再把模型的 `baseUrl` 指向它，不需要真实的 API key。脚本用的是 npm 上的 `@earendil-works/pi-ai@1.0.2`，和基线是同一个版本。
- 本章的术语：
  - **供应商**（provider）：一个运行时单元，带着自己的 id、凭据规则、模型目录和请求方法，例如 `anthropic`、`deepseek`。
  - **线协议**（api）：和模型服务通信的请求、响应格式，例如 `anthropic-messages`、`openai-completions`。
  - **协议实现**：`src/api/` 下实现某种线协议的模块，对外只有 `stream` 和 `streamSimple` 两个函数。

---

## 1 · 从 streamFn 到 Models

先看接缝的两端。agent-core 里 `StreamFn` 的类型如下：

```typescript
export type StreamFn = (
	model: Model<Api>,
	context: TranscriptContext,
	options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

它上面的注释写明了约定：请求、模型、运行时的失败都不能抛错，也不能返回被拒绝的 Promise，必须编码进返回的流里，最终消息的 `stopReason` 为 `"error"` 或 `"aborted"`，并带上 `errorMessage`。第 3 节到第 5 节会看到 pi-ai 怎么兑现这个约定。

第三章 6.1 节的例子里，交给 `Agent` 的是 `models.streamSimple.bind(models)`。`models` 是 `createModels()`（创建一个空的模型集合）返回的对象，类型是 `MutableModels`。和请求相关的方法有这些：

- `setProvider(provider)`（按 id 注册或替换一个供应商）
- `getModel(provider, id)`（在已注册供应商的目录里同步查找一个对话模型，找不到返回 `undefined`）
- `stream(model, context, options)`（发请求，返回事件流；`options` 是这个线协议专属的选项）
- `streamSimple(model, context, options)`（发请求，返回事件流；`options` 是跨协议统一的选项，多了 `reasoning` 等字段）
- `complete(...)` / `completeSimple(...)`（分别等于 `stream(...).result()` 和 `streamSimple(...).result()`，只返回最终消息）

`stream` 和 `streamSimple` 的区别在选项：

| | `stream` | `streamSimple` |
|---|---|---|
| 选项类型 | 按线协议区分 | 统一的 `SimpleStreamOptions` |
| 思考强度 | 协议自己的字段 | `reasoning` 一个字段 |
| 典型用户 | 要用某家独有参数的调用方 | `Agent` |

`reasoning` 取 `minimal` 到 `max` 六档，`streamSimple` 会把它换算成每种协议自己的参数（3.2 节）。例如 Anthropic 协议的选项里没有 `reasoning`，只有 `thinkingEnabled`（是否开启思考）、`thinkingBudgetTokens`（思考的 token 预算）和 `effort`（自适应思考的强度档位）。

> 📌 coding-agent 交给 `Agent` 的不是 pi-ai 的 `Models`，而是自己的 `ModelRuntime`（coding-agent 的模型运行时，实现了同一个 `Models` 接口）。它的 `streamSimple` 走的是同一套步骤（3.1 节）：规整上下文、先返回流、解析凭据，然后交给供应商。区别是多了虚拟模型的路由，以及 `models.json` 和远程目录的叠加。本章讲 pi-ai 自己的 `Models`。

---

## 2 · 供应商和线协议：两级分派

![图 1 · 两级分派：先找供应商，再选线协议](/images/harness/pi/ch04/fig1-dispatch.png)

`model` 上有两个字段决定请求去哪：`provider` 和 `api`。分派分两级完成：

1. **`Models` 按 `model.provider` 找供应商**：`Models` 内部是一个 `Map<id, Provider>`，这里没有按 `api` 注册实现的全局表。
2. **供应商按 `model.api` 选协议实现**：每个供应商在创建时就带上了自己的协议实现。

内置供应商都由 `createProvider`（把 id、凭据、模型目录和协议实现组装成一个供应商）创建。Anthropic 的工厂函数是这样的（凭据部分省略）：

```typescript
export function anthropicProvider(): Provider<"anthropic-messages"> {
	return createProvider({
		id: "anthropic",
		name: "Anthropic",
		baseUrl: "https://api.anthropic.com",
		auth: {
			apiKey: anthropicApiKeyAuth(),
			oauth: lazyOAuth({ /* … */ }),
		},
		models: Object.values(ANTHROPIC_MODELS),
		api: anthropicMessagesApi(),
	});
}
```

`api` 有两种写法：

- **只给一个实现**：所有模型都走它。42 个内置供应商里大多数是这样，比如 `deepseek`、`groq` 都直接用 `openAICompletionsApi()`。
- **给一张按 `model.api` 索引的表**：同一个供应商下不同模型走不同协议。`github-copilot`、`openrouter`、`opencode` 等 6 个供应商是这样。

```typescript
api: {
	"anthropic-messages": anthropicMessagesApi(),
	"openai-completions": openAICompletionsApi(),
	"openai-responses": openAIResponsesApi(),
},
```

所以供应商不只是一份元数据，它自己就有 `stream` 和 `streamSimple`，`Models` 只负责解析凭据并转交。一些供应商还会在协议实现外面再包一层，例如给请求加上会话头。

两级里任何一级查不到，结果都一样：返回的流里只有一个 `error` 事件。探针的三个例子：

- 未注册的供应商：`Unknown provider: nope`
- 供应商没有这个 `api` 的实现：`Provider mixed has no API implementation for "carrier-pigeon"`
- 找到了供应商但没有凭据：`Provider is not configured: anthropic`

> 💡 自定义供应商也走 `createProvider`。比如本地的 vLLM 服务，给出 `id`、一个 `auth.apiKey.resolve`（返回请求用的凭据）、几个 `api: "openai-completions"` 的模型，再把 `api` 设为 `openAICompletionsApi()`（从 `@earendil-works/pi-ai/api/openai-completions.lazy` 导入），`setProvider` 之后就和内置供应商一样使用。按 `api` 注册实现的全局表另有一套，在 `@earendil-works/pi-ai/compat` 里（全局的 `stream()`、`registerApiProvider()` 等），文件头注释写明这是临时兼容入口，新代码用 `createModels()`。

---

## 3 · 一次请求的路径

![图 2 · 一次 streamSimple 请求的路径](/images/harness/pi/ch04/fig2-request-path.png)

### 3.1 Models 这一段：同步返回，异步准备

`Models.streamSimple` 的全部代码：

```typescript
streamSimple(model: Model<Api>, context: Context, options?: ModelsSimpleStreamOptions): AssistantMessageEventStream {
	const transcript = normalizeContext(context);
	return lazyStream(model, async () => {
		const provider = this.requireChatProvider(model);
		const { requestModel, requestOptions } = await this.applyAuth(model, options);
		return provider.streamSimple(requestModel, transcript, requestOptions as SimpleStreamOptions);
	});
}
```

- `normalizeContext`（把 `systemPrompt` 和 `tools` 折叠成开头的一条 system 消息）是这一段唯一同步执行的一步。它的返回类型 `TranscriptContext` 带一个类型标记，供应商和协议实现只接受这种类型，原始的 `Context` 传不进去。agent-core 在调用 `streamFn` 之前已经做过一次，这里再做一次没有影响，因为上下文里已经没有 `systemPrompt` 和 `tools` 了。
- `lazyStream`（先同步返回一个流，再在背后执行异步的准备工作）把后面所有可能失败的步骤包了起来。准备阶段一旦抛错，它就推入一个 `error` 事件并结束这个流：

```typescript
export function lazyStream(
	model: Model<Api>,
	setup: () => Promise<AsyncIterable<AssistantMessageEvent>>,
): AssistantMessageEventStream {
	const outer = new AssistantMessageEventStream();

	setup()
		.then((inner) => forwardStream(outer, inner))
		.catch((error) => {
			const message = createSetupErrorMessage(model, error);
			outer.push({ type: "error", reason: "error", error: message });
			outer.end(message);
		});

	return outer;
}
```

- `requireChatProvider`（确认是对话模型，并按 `model.provider` 取出供应商）
- `applyAuth`（解析这次请求的凭据，再合并请求头、环境变量和 `baseUrl`）

凭据按这个顺序解析：

1. 请求选项里的 `apiKey`（供应商支持 API key 时生效）
2. 凭据仓库（`CredentialStore`，按供应商 id 存放 API key 或 OAuth 令牌）里存的凭据。只要仓库里存了这个供应商的凭据，就只用它：类型对不上或刷新失败，都不会再退回到环境变量
3. 环境变量这类环境凭据。Anthropic 依次查 `ANTHROPIC_AUTH_TOKEN`（作为 Bearer 请求头发送）、`ANTHROPIC_OAUTH_TOKEN`（订阅账号的 OAuth 令牌）、`ANTHROPIC_API_KEY`（普通 API key），最后是工作负载身份联合

存的是 OAuth 令牌、并且 5 分钟内就要过期时，会在这一步先刷新（5.4 节）。之后选项里显式给的请求头逐个字段覆盖默认值，最后执行只有 `Models` 才有的 `transformHeaders`（发出前改写完整的请求头）。

> 📌 agent-core 的 `getApiKey` 钩子不属于 pi-ai。它在 agent-core 里、调用 `streamFn` 之前执行，结果作为 `options.apiKey` 传进来，也就是上面顺序里的第一项。

### 3.2 协议实现这一段：换算选项，拼请求体

供应商把请求交给协议实现。如果这是第一次请求这种线协议，会先加载协议实现的模块（第 6 节）。以 `anthropic-messages` 为例，`streamSimple` 先换算选项：

- `buildBaseOptions`（复制通用选项，并把 `maxTokens` 夹紧到上下文窗口减去估算的输入 token、再留 4096 的余量）。它还按思考级别合并采样参数：模型的 `samplingParams`、模型 `samplingParamsByThinkingLevel` 里对应级别（按模型支持的级别夹紧后）的那一组、请求的 `samplingParams`，后面的逐键覆盖前面的。只有 OpenAI 兼容的三种协议实现（`openai-completions`、`openai-responses`、`azure-openai-responses`）把这些参数写进请求体，其他协议忽略它们。coding-agent 的 `models.json` 可以给模型配 `samplingParamsByThinkingLevel`
- 没有 `reasoning`：`thinkingEnabled: false`
- 模型声明了自适应思考：`reasoning` 映射成 `effort`（`minimal` 和 `low` 都映射成 `low`）
- 其他模型：按默认预算表取思考 token 数（`minimal` 1024、`low` 2048、`medium` 8192、`high` 16384），给回答至少留 1024 个 token

然后调用同一模块的 `stream`。`stream` 先同步做两件事：创建一个 `AssistantMessageEventStream`（事件流对象），并调用 `resolveTranscript`（模型不接受对话中途的 system 消息时，把所有 system 消息合成开头的一条）。然后返回这个流，真正的请求在一个异步函数里进行，顺序是：

1. 创建 SDK 客户端（检查凭据、组装请求头）
2. `buildParams`（拼出供应商格式的请求体），其中消息部分先经过 `transformMessages`（3.3 节）
3. `onPayload`（查看或替换请求体；返回 `undefined` 表示不改）
4. `retryProviderRequest`（发 HTTP 请求，可重试的失败按需重试，5.3 节）
5. `onResponse`（拿到状态码和响应头，还没开始读响应体）
6. 推入 `start`，然后逐个读取供应商事件。每个事件先交给 `onProviderStreamEvent`（在归一化之前查看解析好的供应商事件），再转换成统一事件

探针把三个钩子和 HTTP 请求的先后打印出来，顺序和上面一致：

```text
onPayload keys=model,messages,max_tokens,stream,betas,system,thinking thinking={"type":"enabled","budget_tokens":8192,"display":"summarized"} max_tokens=64000
HTTP POST /v1/messages?beta=true (model=claude-haiku-4-5, stream=true, msgs=1, …)
onResponse status=200 request-id=req_mock
  raw message_start
event start
  raw content_block_start
  …
```

几个细节：

- `onPayload` 返回的新请求体会整体替换原来的。Anthropic 协议会再把 `stream: true` 补回去，保证仍是流式请求。
- 在 Anthropic 和 OpenAI 系的协议里，`onResponse` 只在最终成功的那次响应上调用一次，重试中失败的响应不会触发它，全部失败时一次也不触发。`mistral-conversations`、`pi-messages` 和 `openai-codex-responses` 不同，每拿到一次响应就调用一次，包括 4xx 和 5xx。
- `onProviderStreamEvent` 收到的是解析之后的事件对象，不是原始字节。三个钩子都会被 `await`，回调慢了会拖慢整个流，回调抛错会让这次请求失败。
- `onPayload` 和 `onProviderStreamEvent` 10 种对话线协议都支持；`onResponse` 有 8 种支持，两种 Google 协议不调用它。

### 3.3 transformMessages：把历史整理成这家能接受的样子

对话历史里可能有别家模型的回复、半截的回复、没有结果的工具调用。`transformMessages`（把历史消息整理成目标模型能接受的形式）在 9 种线协议的请求体构建里被调用，只有 `pi-messages` 不用它。它分两遍处理。

第一遍逐条改写消息：

- `content` 为 `null` 的消息补成空数组（兼容手写的历史和旧的会话文件）
- 模型不支持图片时，用户消息和工具结果里的图片换成一句占位文本
- 助手消息先判断是不是**同一个模型**发的：`provider`、`api`、模型 id 三者都相同才算。不是同一个模型时：
  - 加密的思考块（`redacted`）直接丢掉
  - 普通思考块转成纯文本，空的思考块丢掉
  - 文本块去掉签名
  - 工具调用去掉 Google 的 `thoughtSignature`（Gemini 用来复用思考上下文的签名），id 交给协议自己的规则改写，对应工具结果的 `toolCallId` 也一起改

第二遍处理消息之间的关系：

- `stopReason` 为 `error` 或 `aborted` 的助手消息整条跳过。注释给出的理由：这些是没完成的轮次，可能只有推理没有正文，或者工具调用不完整，重放会让供应商报错，应该让模型从上一个有效状态重来。
- 一条助手消息的工具调用，在下一条助手消息或用户消息出现之前还没有结果，就补一条错误结果，内容是 `No result provided`。对话以这种调用结尾时也一样补上。
- 夹在工具调用和它的结果之间的 system 消息，挪到结果之后。

探针构造了一段有问题的历史，交给 Anthropic 协议，再用 `onPayload` 抓出实际发送的 `messages`：

| # | 历史里的消息 | 实际发送的 |
|---|---|---|
| 1 | 用户 `q1` | 原样 |
| 2 | GPT 的回复：思考 + 文本 + 工具调用 `call_abc\|fc_…` | 思考变成文本；id 里的 `\|` 换成 `_` |
| 3 | 这个调用的结果 | id 跟着改 |
| 4 | 中止的回复：`half an ans` | 不发送 |
| 5 | 用户 `q2` | 原样 |
| 6 | 本模型的回复：带签名的思考 + 工具调用 | 思考和签名保留 |
| 7 | — | 补一条 `No result provided`，`is_error: true` |
| 8 | 用户 `q3` | 原样 |

> ⚠️ 第 2 行的思考内容会以普通文本的身份发给新模型。跨模型切换时，旧模型的推理过程对新模型是可见的，而且不带任何标记。

---

## 4 · 流式归一化

### 4.1 12 种事件

所有协议实现输出同一种事件，`AssistantMessageEvent` 一共 12 种：

- `start`（流开始，`partial` 是一条空的助手消息）
- `text_start` / `text_delta` / `text_end`（一个文本块开始、追加一段文字、结束并给出完整文本）
- `thinking_start` / `thinking_delta` / `thinking_end`（一个思考块开始、追加、结束）
- `toolcall_start` / `toolcall_delta` / `toolcall_end`（一个工具调用开始、追加一段参数 JSON、结束并给出解析好的 `toolCall`）
- `done`（正常结束，`reason` 为 `stop`、`length`、`toolUse` 或 `deferred`）
- `error`（失败结束，`reason` 为 `error` 或 `aborted`）

中间九种都带 `contentIndex`（这个块在 `content` 数组里的位置）。

![图 3 · anthropic-messages：供应商事件怎么变成统一事件](/images/harness/pi/ch04/fig3-normalize.png)

图 3 是探针里 Anthropic 协议的完整映射。供应商的流里有一个思考块、一个文本块和一个工具调用，pi-ai 发出 13 个统一事件，覆盖了除 `error` 以外的 11 种。几处值得注意的地方：

- `start` 在拿到 HTTP 响应后就推入，不等 `message_start`。
- 有的供应商事件不产生统一事件。`message_start` 只记下 `responseId` 和初始 usage；`signature_delta` 只把签名累加到思考块上；`message_delta` 只更新停止原因和 usage。
- 工具参数是流式的 JSON 片段。每来一段，`parseStreamingJson`（容错地解析不完整的 JSON，总能返回一个对象）就把累计的字符串重新解析一遍，所以 `toolcall_delta` 时就能读到部分参数。`toolcall_end` 时再完整解析一次，并删掉暂存的 JSON 字符串。

各协议只保证这几条顺序：`start` 在最前，`done` 或 `error` 在最后，同一个块的 `*_start` 在 `*_delta` 之前、`*_delta` 在 `*_end` 之前。块和块之间怎么交错由协议决定。`openai-completions` 就不一样：它的 `*_end` 全部等流读完才一起发出。探针的记录：

```text
start → thinking_start[0] → thinking_delta[0] → thinking_delta[0] → text_start[1] → text_delta[1] → text_delta[1]
→ toolcall_start[2] → toolcall_delta[2] → toolcall_delta[2] → thinking_end[0] → text_end[1] → toolcall_end[2] → done
```

### 4.2 partial 是一个活对象

事件里的 `partial` 不是那一刻的快照，而是同一个正在累积的 `AssistantMessage`。协议实现一开始创建一个 `output` 对象，所有事件都带着它，最后 `done` 交出的 `message` 也是它。探针验证了两点：

- 第一个事件的 `partial` 和 `result()` 返回的最终消息是同一个对象（`===` 为真）
- 消费方读到第一个 `toolcall_delta` 时，`partial` 里的参数已经是完整的 `{"path":"/tmp/a.txt"}`，因为生产者已经处理完后面的片段

> 📌 要保存中间状态，就自己深复制一份，或者只用事件里的 `delta`。agent-core 在 `start` 时把这个 `partial` 直接放进上下文；它发给订阅者的 `message_update` 里，`message` 只是浅复制，`content` 数组仍和 `partial` 共享。

> 📌 一个流只能有一个消费者。`EventStream`（`AssistantMessageEventStream` 的基类）内部是一个队列，事件被取走就没了。探针让两个 `for await` 同时读同一个流，结果一个拿到 7 个事件，一个拿到 6 个。要分发给多处，自己读一遍再转发。`result()` 可以多处调用，它只是一个 Promise。

### 4.3 最终消息的三个字段

**`stopReason`** 一共 7 个值：`pending`（流还没结束时的初始值）、`stop`（正常说完）、`length`（达到输出上限）、`toolUse`（要调用工具）、`error`（失败）、`aborted`（被中止）、`deferred`（交给了延迟返回）。每种协议有自己的映射函数。Anthropic 的映射：

- `end_turn`（说完了）、`pause_turn`（服务端暂停，可以重新提交）、`stop_sequence`（遇到停止序列）→ `stop`
- `max_tokens`（达到输出上限）→ `length`
- `tool_use`（要调用工具）→ `toolUse`
- `refusal`（模型拒答）→ `error`，`errorMessage` 取 `stop_details.explanation`，没有时用一句默认文本
- `sensitive`（被安全过滤拦下）→ `error`
- 不认识的值 → 抛错，最后也变成 `error`：`Unhandled stop reason: …`

其他协议有各自的规则：`openai-completions` 不认识的 `finish_reason` 直接映射成 `error`；`openai-responses` 只要回复里有工具调用，就把 `stop` 改成 `toolUse`。另外，供应商原始的停止原因保存在 `rawStopReason` 里。

**`usage`** 来自供应商的报告。Anthropic 协议在 `message_start` 记下初值，在 `message_delta` 更新，每次都重新计算 `totalTokens` 并调用 `calculateCost`（按模型目录里的单价算出费用；输入超过阈值时换用分档价格，1 小时缓存写入按 2 倍输入价计）。探针里 input 100、output 42、cacheRead 20，`totalTokens` 是 162，`cost.total` 是 0.000312 美元。

**`errorMessage`** 只在失败时出现，内容由协议实现决定：Anthropic 直接用异常本身的消息；`openai-completions`、`openai-responses`、Google 等协议先经过一个共用的整理函数，SDK 的错误消息里没带响应体时补上"状态码 + 响应体"，响应体最多保留 4000 个字符。

---

## 5 · 失败与中止

### 5.1 失败怎么进到流里

协议实现都用同一个结构：同步返回流，在一个异步函数里 `try` 整个请求，`catch` 时把失败写进同一个 `output` 对象。Anthropic 协议的 `catch`：

```typescript
} catch (error) {
	for (const block of output.content) {
		delete (block as { index?: number }).index;
		// partialJson is only a streaming scratch buffer; never persist it.
		delete (block as { partialJson?: string }).partialJson;
	}
	output.stopReason = options?.signal?.aborted ? "aborted" : "error";
	output.errorMessage = error instanceof Error ? error.message : JSON.stringify(error);
	stream.push({ type: "error", reason: output.stopReason, error: output });
	stream.end();
}
```

两个设计：

- **已收到的内容不丢**：`error` 事件交出的还是那个 `output`，里面有失败前累积的文本、思考和 usage。
- **"不正常的结束"也算失败**：读完流之后还有三道检查。signal 已中止、`stopReason` 还是 `pending`（流没给出停止原因）、映射出的 `stopReason` 是 `error`（例如拒答），这三种都会主动抛错，走进同一个 `catch`。

![图 4 · 每一种失败最后都变成一个 error 事件](/images/harness/pi/ch04/fig4-failures.png)

探针逐个触发了图 4 里的失败。经过 `Models` 调用时，没有一种会同步抛出，`result()` 也都正常 resolve，拿到的是带 `errorMessage` 的消息。按发生的阶段分：

- **准备阶段**：只有一个 `error` 事件，前面没有 `start`，`content` 为空，没有发出 HTTP 请求。
- **HTTP 阶段**：同样只有 `error`。
- **流式阶段**：先有 `start` 和已经收到的那些事件，最后是 `error`。拒答的例子里，三个块都完整收到了，结果仍然是 `error`。

唯一的例外在 `Models` 之外。直接 `import` 协议模块（如 `@earendil-works/pi-ai/api/anthropic-messages`）调用它的 `streamSimple`，缺少凭据时会同步抛出 `No API key for provider: anthropic`；同一模块的 `stream` 则仍然返回一个只有 `error` 的流。`StreamFunction` 类型上的注释也写明了这一点。

### 5.2 中止

`signal` 一路传到 HTTP 请求和读流的循环里，协议实现在 `catch` 时看 `signal.aborted`，决定 `reason` 是 `aborted` 还是 `error`。探针里有两种中止：

- **流式过程中中止**：mock 服务每 40 毫秒发一个事件，在第一个 `text_delta` 后调用 `abort()`，事件停在 `text_delta`，接着是 `error`（`reason: "aborted"`）。`content` 里保留着已经收到的思考块和文本 `"Hello"`。在第一个 `toolcall_delta` 后中止，工具调用的参数停在解析到一半的 `{"path":"/tm"}`。中止只是不再往下读，已经进了队列的事件仍会交给消费者。服务端一次性发完时，中止后还能读到后面的全部事件，最后才是 `error`，`errorMessage` 也换成了 `Request was aborted`。
- **请求开始前 signal 就已中止**：凭据解析的第一步会检查 signal，抛出的异常由 `lazyStream` 接住。`lazyStream` 生成的错误消息一律是 `stopReason: "error"`，所以结果是 `error`，`errorMessage` 为 `This operation was aborted`。

第二种情况在 Agent 里真实存在。第三章 4.5 节讲过：串行执行工具时中止，循环会照常结束这一轮，下一次请求带着已中止的 signal 发出。探针用 `Models.streamSimple` 作为 `streamFn` 重现了这个场景，最后一条助手消息是 `stopReason: "error"`、`errorMessage: "This operation was aborted"`，没有发出第二次 HTTP 请求。循环对 `error` 和 `aborted` 的处理相同，都从提前出口退出，所以运行照样结束。

> ⚠️ 要判断"是不是用户取消的"，不要只看 `stopReason === "aborted"`，还要看你自己的 `signal.aborted`。

### 5.3 重试

重试分三层，各管一件事：

- **请求级**：pi-ai 的 `retryProviderRequest`（重新实现 OpenAI 和 Anthropic SDK 的重试策略，让等待可以被中止）
- **消息级**：pi-ai 导出的 `retryAssistantCall`（拿到 `stopReason: "error"` 的消息后，按错误信息判断要不要整次重来）
- **产品级**：coding-agent 的自动重试（发出 `auto_retry_start` / `auto_retry_end` 事件，用户可以看到并取消）

请求级这一层的几条规则：

- SDK 自带的重试被关掉了：调用 SDK 时固定传 `maxRetries: 0`，因为 SDK 重试等待的计时器不理会 `signal`。
- **`maxRetries` 默认是 0**。选项类型的注释提到"SDK 默认重试 2 次"，那说的是 SDK 本身；经过 pi-ai 的 Anthropic、OpenAI 系和 Google 协议时，不传就不重试。探针里 HTTP 500 默认只发 1 次请求；设成 2 发了 3 次，用时约 1.35 秒。
- 哪些失败可重试：响应头 `x-should-retry`（服务端直接说要不要重试）优先，其次是连接错误、408、409、429 和 5xx。
- 等多久：先看 `retry-after-ms`（毫秒），再看 `retry-after`（秒数或 HTTP 日期），都没有就指数退避，从 0.5 秒起翻倍，最多 8 秒，再减去最多 25% 的随机抖动。
- **`maxRetryDelayMs` 是一道闸，不是上限**：服务器要求的等待超过它（默认 60 秒），就不再等待，立刻失败，并把要求的时长写进错误里。探针里 429 带 `retry-after: 120`，结果是只发 1 次请求，错误为 `Server requested 120s retry delay (max: 60s). 429 …`。这条错误信息能被消息级的判断识别为可重试，交给上层去等，上层的等待用户看得见，也能中止。

> 📌 并非所有协议都直接走 `retryProviderRequest`。Google 协议在它外面包了一层，`openai-codex-responses` 有自己的重试循环（默认也是 0 次）。`mistral-conversations` 和 `pi-messages` 直接用 `fetch`，没有请求级重试。Bedrock 不读 `maxRetries`，由 AWS SDK 按自己的默认策略重试。

### 5.4 OAuth 令牌刷新

刷新发生在 3.1 节的 `applyAuth` 里，每次请求都会检查一次：

- 剩余有效期不足 5 分钟就刷新
- 刷新在凭据仓库的锁里进行，拿到锁后再检查一次有没有过期，所以并发的多个请求只会刷新一次
- 刷新本身有 15 秒超时，也会响应请求的 `signal`
- 刷新得到的新令牌先写回凭据仓库，再用于本次请求

探针用一个自定义的 OAuth 供应商验证：令牌还有 60 秒过期时同时发 3 个请求，`refresh` 只被调用 1 次，3 个请求带的都是新令牌；第 4 个请求不再刷新。让 `refresh` 抛出 `invalid_grant`，请求的结果是流里的 `error`：`OAuth refresh failed for acme: invalid_grant`。凭据仓库里的旧凭据不会被删掉，重新登录就能恢复。

---

## 6 · 按需加载与入口体积

10 个协议实现里有 7 个在模块顶部静态 `import` 了自己的 SDK（`@anthropic-ai/sdk`、`openai`、`@google/genai`、AWS SDK），这些 SDK 都不小。供应商的工厂函数不直接引用协议实现，而是引用一个 `.lazy.ts` 包装，除去 import，它只有一行：

```typescript
export const anthropicMessagesApi = (): ProviderStreams => lazyApi(() => import("./anthropic-messages.ts"));
```

`lazyApi`（返回一个代理对象，第一次调用 `stream` 或 `streamSimple` 时才动态加载真正的模块）把加载也包进 `lazyStream`，加载失败同样变成一个 `error` 事件。之后的加载由运行时的模块缓存去重。Bedrock 更进一步：它用变量拼出 `import` 的路径，打包工具追踪不到，Node 专用的 AWS SDK 就不会被打进浏览器包。

![图 5 · 代码什么时候被加载](/images/harness/pi/ch04/fig5-lazy.png)

探针在 Node 里挂了一个模块解析钩子，记下每一步之后加载过的模块：

| 步骤 | pi-ai 文件 | Anthropic SDK 文件 |
|---|---|---|
| 导入根入口 | 26 | 0 |
| 导入 `providers/anthropic` | 34 | 0 |
| 第一次 `streamSimple` | 44 | 121 |
| 导入 `providers/all` | 186 | 121 |

导入 `providers/all`（全部 42 个供应商的工厂和目录）之后，OpenAI SDK 的文件数仍然是 0。只要还没有请求过那种线协议，它的实现就不会被加载。这也是 README 推荐按供应商子路径导入的原因：每个 `providers/<id>` 只带这一个供应商的目录和它用到的一个或几个 `.lazy` 包装，不会牵连其他供应商。

根入口 `@earendil-works/pi-ai` 的文件头注释写着"side-effect free"：没有生成的模型目录、没有真实的供应商工厂（只带了测试用的 `fauxProvider`）、没有全局注册表、没有 OAuth 实现。供应商在 `providers/*` 下，协议实现在 `api/*` 下，旧的全局 API 在 `compat` 里。`package.json` 的 `sideEffects` 一共只列了三个文件：`compat` 和两个图像相关的文件。要让 SDK 真正留在单独的分块里，打包时还需要开启代码分割；不打包直接在 Node 里运行时，靠的就是上面这种动态 `import`。

第二章提到的 entry-graphs 检查，守的就是这一层的入口：

- 它数的是一个入口沿着静态值导入能走到的**工作区源文件**数，不数 npm 依赖，也不跟踪动态 `import()`，所以 `.lazy.ts` 背后的协议实现和 SDK 不算在内。
- pi-ai 只有两个入口设了预算：
  - `./models`：最多 15 个文件，实际 13 个；不能走到 `providers/`、生成的目录、根入口，以及依赖 TypeBox 的校验工具
  - `./utils/*`：最多 3 个；不能走到 `providers/`、`api/` 和根入口
- 根入口和 `providers/*` 没有设预算。用同样的方法数，根入口 26 个，`providers/anthropic` 21 个，`providers/all` 121 个，`compat` 139 个。

---

## 7 · 用法

### 7.1 单独用 pi-ai 发一次请求

不需要 agent-core，pi-ai 自己就能发请求。下面按 README 的写法，只注册 Anthropic 一个供应商：

```typescript
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6");
if (!model) throw new Error("Model not found");

const s = models.streamSimple(
	model,
	{ systemPrompt: "Answer in one sentence.", messages: [{ role: "user", content: "What is SSE?", timestamp: Date.now() }] },
	{ reasoning: "low", maxRetries: 2 },
);

for await (const event of s) {
	if (event.type === "text_delta") process.stdout.write(event.delta);
	if (event.type === "error") console.error(`\n${event.reason}: ${event.error.errorMessage}`);
}

const message = await s.result();
console.log(`\n${message.stopReason}, ${message.usage.totalTokens} tokens, $${message.usage.cost.total.toFixed(6)}`);
```

- 凭据按 3.1 节的顺序解析，这里会读取 `ANTHROPIC_API_KEY` 等环境变量。
- `maxRetries: 2` 要显式写上，不写就不重试。
- 不需要 `try` / `catch`：失败会以 `error` 事件出现，`result()` 拿到的是带 `errorMessage` 的消息。

### 7.2 自定义 streamFn：包一层再交给 Agent

`StreamFn` 只是一个函数，可以在 `models.streamSimple` 外面包一层，在请求前改选项、在流经过时做记录。下面这个包装做两件事：

- 默认把 `cacheRetention` 设成 `"long"`，让支持的供应商用更长的提示缓存
- 每次请求结束后打印首个增量的耗时、停止原因和 token 数

```typescript
import { Agent, type StreamFn } from "@earendil-works/pi-agent-core";
import { createAssistantMessageEventStream } from "@earendil-works/pi-ai";

function withLogging(inner: StreamFn): StreamFn {
	return async (model, context, options) => {
		const started = Date.now();
		const source = await inner(model, context, { cacheRetention: "long", ...options });
		const out = createAssistantMessageEventStream();
		(async () => {
			let firstDelta: number | undefined;
			for await (const event of source) {
				if (firstDelta === undefined && event.type.endsWith("_delta")) firstDelta = Date.now() - started;
				out.push(event);
			}
			const message = await source.result();
			console.log(`[llm] ${model.provider}/${model.id} msgs=${context.messages.length} stop=${message.stopReason} firstDelta=${firstDelta}ms in=${message.usage.input} out=${message.usage.output}`);
			out.end(message);
		})();
		return out;
	};
}

const agent = new Agent({
	initialState: { model, tools: [readTool], systemPrompt: "be terse" },
	streamFn: withLogging(models.streamSimple.bind(models)),
});
```

`readTool` 是你自己定义的 `AgentTool`。这段代码里的几个选择：

- **转发而不是共享**：一个流只能有一个消费者（4.2 节），所以包装层自己读完源流，再把事件逐个推进一个新的流。`Agent` 读的是新流。
- **事件原样转发**：`partial` 仍是源流里的同一个对象，`Agent` 看到的事件和不包装时完全一样。
- **不在包装层 `try` / `catch`**：`inner` 是 `Models.streamSimple`，失败已经在流里了。如果要包装的是会抛错的函数（例如直接调用协议模块），就要在这里接住异常，自己推一个 `error` 事件，才能守住 `StreamFn` 的约定。
- **`out.end(message)` 兜底**：转发 `done` 或 `error` 事件时，`result()` 就已经确定了，`end(message)` 只是防备源流没有发出结束事件就结束的情况。`Agent` 靠 `result()` 拿最终消息。
- **用工厂函数创建流**：`createAssistantMessageEventStream()`（创建一个空的事件流）。根入口的类型声明把 `AssistantMessageEventStream` 导出成了纯类型，在 TypeScript 里直接 `new` 它会报错。

探针用这个包装跑了一次带工具调用的运行，`Agent` 正常完成了两次请求，日志如下：

```text
[llm] anthropic/claude-fable-5 msgs=2 stop=toolUse firstDelta=42ms in=100 out=9
[llm] anthropic/claude-fable-5 msgs=4 stop=stop firstDelta=3ms in=120 out=5
```

---

## 8 · 关键数字

| 项 | 数量 |
|---|---|
| 内置供应商 | 42 个 |
| 对话线协议 | 10 种 |
| 按 `model.api` 再分派的供应商 | 6 个 |
| `AssistantMessageEvent` 种类 | 12 种 |
| `StopReason` 取值 | 7 个 |
| 请求级重试的默认次数 | 0 |
| `maxRetryDelayMs` 默认值 | 60 秒 |
| OAuth 提前刷新窗口 | 5 分钟 |
| `src/api/` | 39 个文件，12,891 行 |
| `packages/ai` 的提交（北京时间 2026-08-01 至 2026-10-01，不含合并提交） | 246 次，其中 `src/api/` 82 次 |

图像生成另有 1 种线协议，分类模型另有 3 种，不算在 10 种里。

---

## 9 · 术语表

| 术语 | 含义 |
|---|---|
| **供应商（Provider）** | 带 id、凭据、模型目录和请求方法的运行时单元 |
| **线协议（api）** | 请求与响应的格式，多个供应商共用 |
| **协议实现** | `src/api/` 下实现一种线协议的模块 |
| **`Models`** | 注册供应商、查模型、发请求的集合 |
| **`TranscriptContext`** | `normalizeContext` 产出的上下文，供应商只收这种 |
| **`partial`** | 事件里正在累积的助手消息，始终是同一个对象 |
| **`lazyStream`** | 先返回流、再在背后准备，准备失败变成 `error` 事件 |
| **`.lazy.ts`** | 首次请求时才加载协议实现的一行包装 |

---

## 10 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| `Models` 和两级分派 | `packages/ai/src/models.ts` → `ModelsImpl`、`createProvider`、`applyAuth` |
| 先返回流再准备 | `packages/ai/src/api/lazy.ts` → `lazyStream`、`lazyApi` |
| 上下文规整 | `packages/ai/src/utils/transcript.ts` → `normalizeContext`、`resolveTranscript` |
| 历史消息整理 | `packages/ai/src/api/transform-messages.ts` → `transformMessages` |
| 一个完整的协议实现 | `packages/ai/src/api/anthropic-messages.ts` → `stream`、`streamSimple`、`mapStopReason` |
| 另一种事件顺序 | `packages/ai/src/api/openai-completions.ts` → `stream` |
| 统一选项的换算 | `packages/ai/src/api/simple-options.ts` |
| 事件和消息类型 | `packages/ai/src/types.ts` → `AssistantMessageEvent`、`StreamOptions`、`StreamFunction` |
| 事件流 | `packages/ai/src/utils/event-stream.ts` → `EventStream` |
| 请求级重试 | `packages/ai/src/utils/provider-retry.ts` → `retryProviderRequest` |
| 消息级重试 | `packages/ai/src/utils/retry.ts` → `retryAssistantCall`、`isRetryableAssistantError` |
| 凭据解析和 OAuth 刷新 | `packages/ai/src/auth/resolve.ts` → `resolveProviderAuth`、`resolveStoredOAuth` |
| 供应商工厂 | `packages/ai/src/providers/anthropic.ts`、`github-copilot.ts`、`all.ts` |
| 测试用的假供应商 | `packages/ai/src/providers/faux.ts` → `fauxProvider` |
| 旧的全局 API | `packages/ai/src/compat.ts` |
| 入口预算 | `scripts/check-entry-graphs.mjs` |
| 官方说明 | `packages/ai/README.md` |
| coding-agent 的模型运行时 | `packages/coding-agent/src/core/model-runtime.ts` → `ModelRuntime` |

**下一章**：工具。看 `AgentTool` 怎么定义、参数怎么校验，以及 coding-agent 的内置工具是怎么实现的。
