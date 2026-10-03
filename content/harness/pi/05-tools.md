---
title: "Pi 源码分析 05 · 工具：一个工具从定义到执行完都经过了什么"
date: 2026-10-02T10:30:00+08:00
description: "接着第四章往下讲：一个工具在 Pi 里怎么定义、怎么声明给模型、工具集中途变化时三种线协议发法、参数在执行前经过的修补转换和校验、结果的每个字段交给谁，以及 coding-agent 内置工具的默认值与保护措施、工具的四个来源和 tool_search 的按需加载。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.1（2026-10-03），commit `a7229ddc2`。文中所有行为、数字和代码均以该版本为准。

第四章讲到，`Models.streamSimple` 把各家的流归一成统一事件，其中一组是 `toolcall_*`：模型想调用一个工具。本章接着往下讲工具本身：一个工具从被定义出来，到模型看见它、调用它、拿到结果，中间经过了哪些代码。

<!--more-->

本章要点：

- **工具声明写在对话记录里**：引擎把"模型可以调用哪些工具"记在 system 消息的 `toolsAdded` / `toolsRemoved` 上。工具集中途变化时，协议实现按模型能力选三种发法之一：原生增删、就地新增、重发完整列表。
- **参数先修补、再转换、再校验**：`prepareArguments` 修补模型给的原始参数；`validateToolArguments` 在副本上删掉 `null` 的可选参数、转换类型，再按 schema 校验。校验失败时，模型收到逐条错误和修补后、转换前的参数。
- **结果按字段分流**：发给各家供应商的只有 `content`（和一个错误标记）；`details` 给界面和会话，`structuredContent` 给程序调用方。工具结果里的图片，各线协议的发法不同。
- **保护措施写在工具内部**：coding-agent 有 8 个内置工具，默认启用 4 个。输出截断、超时、同文件写入排队都由各工具自己实现，没有路径沙箱。
- **工具有四个来源，模型只看得见其中一部分**：内置、扩展、SDK、MCP。每个工具的 `exposure` 决定它是直接声明给模型，还是等 `codemode` 脚本调用或 `tool_search` 加载。

---

## 0 · 阅读说明

- 本章只讲已发布的路径：pi-ai、pi-agent-core 的 `Agent`、pi-coding-agent。`pi-durable` 等实验引擎不在范围内。
- 下面这些第三章已经讲过，本章只引用，不重复：
  - 一个工具调用要过的 6 道关卡，以及每一关失败时交给模型的文本（第三章 4.3 节）
  - `beforeToolCall` / `afterToolCall` 两个钩子（第三章 4.2 节）
  - 串行与并行（第三章 4.1 节）、`terminate`（4.4 节）、中止（4.5 节）
- 第二章 3.1 节讲过 `Tool → AgentTool → ToolDefinition` 三种类型和 `wrapToolDefinition` 的代码，本章第 2 节在此基础上讲各字段在运行时被谁读取。
- 文中的行为和计数都用探针实际跑过。探针用 npm 上的 1.0.1 三个包，走真实的 `Models.streamSimple`（产品层走 `createAgentSession`），把模型的 `baseUrl` 指向本地 mock 服务；mock 按 Anthropic 或 OpenAI 的格式返回预先写好的 SSE 流，并记录每次收到的请求体。不需要真实的 API key。
- 术语：
  - **声明**：发给模型的工具描述（名字、说明、参数 schema）。
  - **激活**：工具在当前的声明集合里。
  - **线协议**：第四章的用法，指 `anthropic-messages`、`openai-completions` 这类请求格式。

---

## 1 · 一个工具调用的全程

![图 1 · 一个工具调用的全程](/images/harness/pi/ch05/fig1-lifecycle.png)

一个工具从定义到结果交回模型，图 1 拆成 12 步，横跨三个包。下面按 8 段说明：

1. **定义**（coding-agent）：内置工具由 `createReadToolDefinition` 这类工厂函数（按工作目录创建一个工具定义）产出，扩展工具通过 `pi.registerTool` 交进来，都是 `ToolDefinition`。
2. **包装**（coding-agent）：注册表刷新时，每个注册的工具都经 `wrapToolDefinition` 转成 `AgentTool`。
3. **激活**（coding-agent）：`_applyToolLoadout`（按 exposure 和激活名单算出要声明的工具）从包装好的工具里选出要给模型看的那部分，赋值给 `agent.state.tools`。
4. **声明**（agent-core）：每轮开始时，`declareToolChanges`（比较可执行的工具和对话里已声明的工具）把差异写成一条 system 消息。
5. **发出**（pi-ai）：协议实现把对话里的工具声明翻译成这家供应商的格式（第 3 节）。
6. **调用**：模型返回 `toolCall`。
7. **查找到执行**（agent-core）：按名字找到工具，依次经过 `prepareArguments`、`validateToolArguments`、`beforeToolCall`、`execute`、`afterToolCall`（第 4、5 节）。
8. **回到模型**：结果变成 `ToolResultMessage` 进入对话记录；下一次请求时，`transformMessages` 按模型能力处理其中的图片，协议实现再序列化发出。

后面各节按这个顺序展开。

---

## 2 · 三层类型：每个字段由谁读取

第二章讲过三种类型的继承关系。从运行时看，每个字段只有特定的读取方：

**L1 `Tool`**（pi-ai，工具声明）有 4 个字段，全部发给模型或供应商：

- `name`（模型调用时用的名字）
- `description`（模型读到的说明）
- `parameters`（TypeBox 写的参数 schema，同时用于本地校验）
- `constrainedSampling`（请供应商在生成阶段就按 schema 或语法约束参数，第 4.3 节）

**L2 `AgentTool`**（agent-core，可执行的工具）继承 `Tool`，新增的字段只给引擎和界面用：

- `label`（界面上显示的名字）
- `prepareArguments`（校验前修补原始参数）
- `outputSchema`（成功结果里 `structuredContent` 的格式）
- `execute`（执行函数，签名见第 5.1 节）
- `executionMode`（这个工具是否必须单独执行）
- `replay`（持久化运行时用的恢复策略；已发布的路径里没有代码读取它，`wrapToolDefinition` 也不拷贝它）

**L3 `ToolDefinition`**（coding-agent，完整定义）没有继承 `AgentTool`，而是把除 `replay` 之外的字段重新声明了一遍，再加上这些只在 coding-agent 内部使用的字段：

- `promptSnippet`（写进系统提示词工具列表的一行说明；不填的自定义工具不进这个列表）
- `promptGuidelines`（写进系统提示词的使用守则）
- `exposure`（模型怎么够到这个工具，第 7.1 节）
- `namespace`（工具分组，例如一个 MCP 服务器）
- `annotations`（MCP 风格的只读、破坏性等提示）
- `defaultActive`（注册时是否顺带激活）
- `prepareLoadout`（工具组合变化时，改写其他工具的描述或隐藏它们的声明）
- `renderCall` / `renderResult` / `renderShell`（终端里怎么画调用和结果）
- `execute` 多出第五个参数 `ctx`（扩展运行环境，能拿到工作目录、界面，还能调用其他工具）

把工具交给模型之前，引擎会先用 `toToolDeclaration`（只保留四个声明字段，并把 schema 做一次 JSON 往返）剥掉其余字段。所以 `label`、`execute` 这些字段不会出现在请求里，也不会写进会话文件。

---

## 3 · 声明：工具怎么到达模型

### 3.1 声明写在对话记录里

pi-ai 公开的 `Context` 虽然有 `tools` 字段，但它只是简写：`normalizeContext` 会把它折进开头的 system 消息，供应商收到的 `TranscriptContext` 里没有独立的工具列表。工具声明放在 `SystemMessage`（system 消息）上：

- `content`（提示词文本）
- `sections`（按名字替换或删除提示词的某一段）
- `toolsAdded`（从这里开始可用的工具，完整定义）
- `toolsRemoved`（从这里开始不可用的工具，只有名字）

开头那条 system 消息就是系统提示词加初始工具；之后每条 system 消息都是一次变更。按顺序重放所有 system 消息，就得到当前的提示词和工具集。几个辅助函数负责重放：

- `createInitialSystemMessage`（把系统提示词和初始工具做成开头的 system 消息）
- `getCurrentTools`（按顺序应用所有增删，得到当前工具集）
- `getToolStateChanges`（比较两个工具集；同名但定义变了，算一次移除加一次新增）

`Agent` 构造时，`initialState.tools` 就通过 `createInitialSystemMessage` 变成开头那条消息的 `toolsAdded`。

### 3.2 运行中换工具：要经过 prepareNextTurn

`agent.state.tools` 是"可执行的工具"，对话里的 system 消息是"已声明的工具"。两者不一致时，`declareToolChanges` 在下一轮开头补一条 system 消息，把差异写进去（第三章第 3 节的第 2 步）。

要注意的是，`Agent` 在每次运行开始时会给工具集拍一份快照交给循环。只在运行中途给 `agent.state.tools` 赋值，当前这次运行看不到。探针里，第一个工具在执行时把 `extra` 加进 `agent.state.tools`，模型下一轮调用 `extra`，得到的是 `Tool extra not found`。

要让变化在同一次运行里生效，得在 `prepareNextTurnWithContext`（每轮开始前调用，可以替换上下文）里把新工具集交给循环：

```typescript
agent.prepareNextTurnWithContext = async (turn) => ({
	context: { ...turn.context, tools: agent.state.tools.slice() },
});
```

coding-agent 的 `AgentSession` 每一轮都这样做，`prepareRequest` 里还会再同步一次。所以扩展在工具的 `execute` 里调用 `pi.setActiveTools(...)`，从同一次运行的下一次请求起就生效。`tool_search` 就是靠这一点工作的（第 7.3 节）。

### 3.3 三种发法

![图 2 · 工具集变了：同一段对话，三种发给模型的方式](/images/harness/pi/ch05/fig2-declare.png)

对话记录里的 system 消息不会原样发出去，模型也看不到 `toolsAdded` 这几个字。协议实现按模型的 compat 标记选择发法：

**A · 原生增删**：`anthropic-messages`，模型同时声明了 `supportsMidConvoSystemMessages`（接受对话中途的 system 消息）和 `supportsMidConvoToolChanges`（接受中途增删工具）。内置目录里有 6 个模型满足，例如 `claude-fable-5`、`claude-opus-5`。

- 初始工具留在请求顶层的 `tools` 里，缓存断点打在最后一个初始工具上，后面跟一个占位工具 `__pi_deferred_placeholder__`（带 `defer_loading: true`，永远不会启用）。
- 后来加入的工具不进顶层列表，而是在对话中途的 system 消息里用 `tool_addition` 块按值给出完整定义（`tool_definition`）。请求为此带上 `inline-tools-2026-09-15` 这个 beta 标记。
- 移除的工具用 `tool_removal` 块按名字撤下。同名工具换了定义时，只发一个带新定义的 `tool_addition`，新定义直接替换旧定义。
- 顶层列表从第一次请求起就固定不变，所以工具集怎么变，缓存前缀都不会失效。
- 占位工具的作用，源码注释是这样解释的：Anthropic 会为对话中途的工具变更在提示词里加一段隐藏的脚手架；提前放一个占位工具，这段脚手架从第一次请求起就在缓存前缀里，第一次工具变更不会让缓存整段失效。
- 开头一个工具都没有时退回 C：Anthropic 不接受全部是延迟声明的工具列表，占位工具前面至少要有一个初始工具。

**B · 只能就地新增**：工具可以在新增的位置声明，但不能撤下。

- OpenAI Responses 用一个 `additional_tools` 条目（内置目录里有 41 个模型声明支持）。
- 只支持 Responses 工具搜索的模型（内置目录里只有 `openai-codex/gpt-5.5`），改用一对伪造的 `tool_search_call` / `tool_search_output` 条目。
- Kimi 一类的 `openai-completions` 模型用带 `tools` 字段的 system 消息（6 个模型）。
- 历史里只要出现过一次移除或同名重定义，就退回 C。

**C · 全量列表**：其余所有模型。对话中途的 system 消息合并回开头（模型不支持中途 system 消息时），顶层 `tools` 每次都是当前完整的工具集。工具集一变，顶层列表就变，缓存前缀从这里失效。

探针把同一段对话分别发给四个模型，只看第二次请求。这段对话先声明了 `keep`，后来新增 `extra`，有的场景还移除了 `keep`：

```text
anthropic/claude-fable-5   只新增   tools=[keep, __pi_deferred_placeholder__(deferred)]  中途: system[tool_addition:extra（完整定义）]
                           新增+移除 同上                                             中途: system[tool_removal:keep, tool_addition:extra（完整定义）]
openai/gpt-5.4             只新增   tools=[keep]          中途: additional_tools[extra]
                           新增+移除 tools=[extra]         （退回全量）
kimi-k3 (fireworks)        只新增   tools=[keep]          中途: system{tools:[extra]}
                           新增+移除 tools=[extra]         （退回全量）
openai/gpt-4               只新增   tools=[keep, extra]   （全量）
```

> 📌 coding-agent 在这层之上还有一次投影：工具的 `prepareLoadout` 可以要求隐藏某些工具的声明（例如 codemode 的 `only` 模式隐藏所有直接工具）。`AgentSession` 在 `transformContext` 里把这些工具从每条 system 消息的增删里过滤掉，对话记录本身不改（第二章 4 节提到的"再包两层"之一）。

---

## 4 · 参数：从模型的 JSON 到 execute 的 params

![图 3 · 参数从模型到 execute 经过的几步](/images/harness/pi/ch05/fig3-arguments.png)

### 4.1 prepareArguments：工具自带的修补

模型给的参数是流式拼出的 JSON（第四章 4.1 节），按名字找到工具后，先交给工具自己的 `prepareArguments`。它在校验之前运行，用来兼容模型的常见偏差。内置的 `edit` 工具就有一个：

- `edits` 被模型写成了 JSON 字符串（源码注释点名了 Opus 4.6 和 GLM-5.1），解析成数组
- `edits` 是单个对象，包成数组
- 旧格式的顶层 `oldText` / `newText`，转成 `edits: [{ oldText, newText }]`

探针里直接调用 `edit.prepareArguments`，`{ path, oldText: "a", newText: "b" }` 和 `{ path, edits: '[{"oldText":"a","newText":"b"}]' }` 都变成了 `{ path, edits: [{ oldText: "a", newText: "b" }] }`。

### 4.2 validateToolArguments：复制、删 null、转换、校验

修补后的参数交给 pi-ai 的 `validateToolArguments`：

1. `structuredClone`：复制一份，后面的改动都作用在副本上。
2. `normalizeOptionalNulls`：可选参数的值是 `null`、而它的 schema 不允许 `null` 时，删掉这个参数。
3. `Value.Convert`：TypeBox 按 schema 转换类型，例如字符串 `"10"` 转成数字 `10`。
4. 对普通 JSON Schema（不是 TypeBox 构造的）再做一轮自己的转换。
5. 用按 schema 缓存的校验器 `Check`。通过就把副本交给 `execute`；不通过就抛错。

第 2 步和第 4.3 节有关：严格模式会把可选参数改写成"必填但可为 null"，模型于是可能对没打算填的参数给出 `null`，校验前要先删掉。

探针自己写了一个 `read` 工具（`path` 必填，`limit`、`lines` 可选，带一个把 `file_path` 改名为 `path` 的 `prepareArguments`；内置的 `read` 没有这个函数），发了一批调用，`execute` 收到的参数如下：

```text
模型给的参数                              execute 收到的
{"path":"a.txt","limit":"10","lines":"true"}  {"path":"a.txt","limit":10,"lines":true}
{"path":"b.txt","limit":null}                {"path":"b.txt"}
{"file_path":"c.txt"}                        {"path":"c.txt"}          ← 探针给这个工具写的 prepareArguments
{"limit":"many"}                             （没有执行）
```

最后一个调用校验失败，模型在下一次请求里收到一条 `is_error: true` 的工具结果：

```text
Validation failed for tool "read":
  - path: must have required properties path
  - limit: must be number

Received arguments:
{
  "limit": "many"
}
```

- 每条错误前面是参数路径，用点号连接；缺少必填参数时，路径就是那个参数名。
- `Received arguments` 是经过 `prepareArguments` 修补、但还没做类型转换的参数。工具没有 `prepareArguments` 时，就是模型给的原样，模型能对照自己写了什么。
- 抛错之后，就是第三章 4.3 节讲过的流程：变成错误结果，循环照常继续。`beforeToolCall` 拿到的 `args` 是校验后的副本。

### 4.3 constrainedSampling：在供应商那边约束生成

本地校验发生在模型生成之后。`constrainedSampling` 则请供应商在生成阶段就约束参数，有两种：

- `{ type: "json_schema", strict: "prefer" | "require" }`：按 schema 严格生成。`prefer` 在做不到时静默退回普通模式；`require` 做不到就报错。
- `{ type: "grammar", variants: { openai_lark?, openai_regex? } }`：按语法生成。工具的参数里必须恰好有一个必填的字符串属性，模型生成的原文就放进这个属性。

内置的 `read`、`bash`、`powershell`、`edit`、`write` 都设了 `json_schema` + `prefer`；`codemode` 工具用 Lark 语法。

能不能生效，由线协议和模型的 compat 标记共同决定：

| 线协议 | json_schema | grammar |
|---|---|---|
| `openai-responses` | 看 `supportsStrictMode` | 看 `supportsOpenAIGrammarTools` |
| `azure-openai-responses` | 默认支持 | 同上 |
| `openai-codex-responses` | 默认支持 | 同上 |
| `openai-completions` | 看 `supportsStrictMode`（默认关） | 看 `supportsOpenAIGrammarTools` |
| `anthropic-messages` | 看 `supportsStrictTools` | 不生效 |
| `bedrock-converse-stream` | 看 `supportsStrictMode` | 不生效 |
| Google 两种 | 仅 Gemini 3 及以后 | 不生效 |
| `mistral-conversations` | 总是尝试 | 不生效 |

转成严格 schema 时，`makeStrictJsonSchema`（把 schema 改写成严格模式能接受的形式）会把所有属性设为必填、可选属性改为"原类型或 null"、加上 `additionalProperties: false`；遇到 `$ref`、`oneOf` 这类结构就放弃。Anthropic 另有一份不支持的关键字清单（如 `minimum`、`maxItems`）。探针的请求体：

```text
anthropic/claude-fable-5   read  → strict: true, required: [path, limit], additionalProperties: false
                           loose（含 minimum: 1, prefer）→ 没有 strict，原样发出
                           must （含 minimum: 1, require）→ 请求不发出：
                             Tool "must" requires JSON-schema constrained sampling, but minimum: 1 is unsupported.
openai/gpt-5.5             loose → strict: true（OpenAI 接受 minimum）
                           calc（grammar）→ { type: "custom", format: { type: "grammar", … } }
deepseek-flash             calc（grammar）→ 退回普通 function 工具，strict: false
```

内置目录里，声明支持 grammar 工具的有 111 个模型，全部在 OpenAI Responses 这一系（`openai-completions` 的实现也支持，但目录里没有模型打开这个标记）。不支持时，grammar 工具按普通工具发出，模型自由生成那一个字符串参数。

---

## 5 · 执行与结果

### 5.1 execute 的四个参数

```typescript
execute: (
	toolCallId: string,
	params: Static<TParameters>,
	signal?: AbortSignal,
	onUpdate?: AgentToolUpdateCallback<TDetails>,
) => Promise<AgentToolResult<TDetails>>;
```

- `toolCallId`（这次调用的 id，和模型给的一致）
- `params`（校验后的副本）
- `signal`（中止信号；工具要自己响应，第三章 4.5 节）
- `onUpdate`（汇报进度：每次调用发出一个 `tool_execution_update` 事件，带上 `partialResult`；`execute` 结束后再调用，引擎直接忽略）

失败时二选一：抛错，或者返回 `isError: true`。`AgentTool` 的注释明确要求不要只在 `content` 里描述失败。

coding-agent 的 `ToolDefinition.execute` 多一个 `ctx`（`ExtensionToolContext`），常用的是：

- `ctx.cwd`（会话的工作目录）
- `ctx.ui`（和界面交互）
- `ctx.executeTool(name, args, options?)`（调用另一个工具，第 5.4 节）

### 5.2 结果的每个字段交给谁

![图 4 · 工具结果的每个字段交给谁](/images/harness/pi/ch05/fig4-result.png)

`AgentToolResult` 有 6 个字段：

- `content`（文本和图片；**唯一发给模型的内容**）
- `details`（任意 JSON，给界面的 `renderResult`、会话文件和 HTML 导出用）
- `structuredContent`（符合 `outputSchema` 的结构化结果，给程序调用方，不进对话记录）
- `usage`（工具自己的用量，计入会话的总用量，不算进上下文占用）
- `isError`（不抛错地报告失败）
- `terminate`（建议这批结束就停，第三章 4.4 节）

引擎把结果变成 `ToolResultMessage` 放进对话记录时，带上 `content`、`details`、`usage` 和 `isError`，不带 `structuredContent`。各家供应商的协议实现序列化工具结果时只读 `content`、`isError`、`toolCallId` 和 `toolName`。探针让工具在 `details` 里放了一个 `savedTo` 字段，在发给模型的请求体里搜不到它。例外是 `pi-messages`：它是 Pi 自己的网关协议（内置目录里 28 个 Radius 模型），把整段上下文原样发给网关，由网关再转给模型，所以 `details`、`usage`、`nestedCalls` 会到达网关。本节和下一节的列表只讲直连供应商的协议。

`isError` 也不是每家都发：Anthropic 有 `is_error`，Bedrock 有 `status`，Google 用 `error` 字段，Mistral 在文本前加 `[tool error]`；OpenAI 的两种协议不发错误标记，模型只能从文本里看出失败。

`structuredContent` 目前主要给 codemode 用：`bash`（和 `powershell`）声明了 `outputSchema`，codemode 脚本经 `ctx.executeTool` 调用 bash 时，拿到的是 `{ output, truncated, full_output_path, exit_code, wall_time_seconds }` 这个对象，而不是给模型看的那段文本。其中 `output` 最多 1 MiB，比给模型的 50KB 宽得多。

### 5.3 工具结果里的图片

图片先经过 pi-ai 的 `transformMessages`（发请求前整理历史，第四章 3.3 节）：模型不支持图片输入时，每一段连续的图片替换成一行文字 `(tool image omitted: model does not support images)`。之后各协议的发法：

- **Anthropic**：图片作为 `image` 块直接放在 `tool_result` 里。
- **Bedrock**：同样放在 `toolResult` 里。
- **OpenAI Responses 一系**：作为 `input_image` 放在 `function_call_output` 里。
- **OpenAI Completions**：`role: "tool"` 的消息只能放文本。一串连续工具结果里的图片，都挪到这一串之后的一条 user 消息里，开头是 `Attached image(s) from tool result:`；有的供应商要求工具结果后面先跟一条助手消息，就先补一句 `I have processed the tool results.`。
- **Google**：Gemini 2.x 及更早的模型挪到单独的 user 消息里，开头是 `Tool result image:`；其余放在 `functionResponse.parts` 里。
- **Mistral**：作为 `image_url` 块放在 tool 消息里。

探针让一个截图工具返回一行文字加一张 PNG，发给三个模型：

```text
anthropic/claude-fable-5      tool_result.content = [text "screen 1x1", image(base64)]
deepseek-flash（支持图片）     tool: "screen 1x1"
                              user: ["Attached image(s) from tool result:", image_url(data:…)]
deepseek-v4-pro（只支持文本）  tool: "screen 1x1\n(tool image omitted: model does not support images)"
```

coding-agent 的 `read` 工具读图片时还做了两件事：默认把图片缩放到 2000×2000 以内；当前模型不支持图片时，在结果里提前加一句 `[Current model does not support images. The image will be omitted from this request.]`。

### 5.4 嵌套调用：nestedCalls

工具可以通过 `ctx.executeTool` 调用别的工具，codemode 脚本就是这样调用工具的。这种嵌套调用不经过模型，走的是 agent-core 导出的 `runToolCall`（和循环里同一套准备、校验、钩子、执行流程，但不发事件、不产生消息）。coding-agent 在外面补上事件和记录：

- 嵌套调用的 id 是 `<父调用 id>/<序号>`，事件带 `parentToolCallId`（上级调用的 id）。
- 嵌套调用的结果不进对话记录。`NestedCallRecorder`（嵌套调用记录器）只记一份摘要，挂在父调用结果消息的 `nestedCalls` 上，嵌套调用的用量也加进父消息的 `usage`。
- 摘要有上限：最多 256 次调用，每次参数 8KB、合计 32KB，错误信息 500 字符。
- `nestedCalls` 只进会话记录，供压缩时追踪文件操作和 HTML 导出使用，不发给模型。

探针里一个工具先后调用了两次 `read`，第二次参数不合法：

```text
events: start(toolu_0_read_both)
        start(toolu_0_read_both/1 parent=toolu_0_read_both)  end(toolu_0_read_both/1 …)
        start(toolu_0_read_both/2 parent=toolu_0_read_both)  end(toolu_0_read_both/2 …)
        end(toolu_0_read_both)
nestedCalls: [{ id: ".../1", name: "read", status: "ok", … },
              { id: ".../2", name: "read", status: "error", error: "Validation failed for tool \"read\": …" }]
```

嵌套调用一样要过参数校验和 `beforeToolCall`（会话里的权限检查对它们同样有效）。对话记录里仍然只有一条工具结果消息，发给供应商的请求体里也找不到 `nestedCalls`。

---

## 6 · coding-agent 的内置工具

### 6.1 8 个工具，默认 4 个

| 工具 | 默认启用 | 参数 |
|---|---|---|
| `read` | ✅ | `path`、`offset`、`limit` |
| `bash` | ✅ | `command`、`timeout` |
| `edit` | ✅ | `path`、`edits` |
| `write` | ✅ | `path`、`content` |
| `grep` | — | `pattern`、`path`、`glob` 等 7 个 |
| `find` | — | `pattern`、`path`、`limit` |
| `ls` | — | `path`、`limit` |
| `powershell` | — | 同 `bash` |

- 默认名单是 `DEFAULT_TOOL_NAMES`。启动时的选择顺序：命令行 `--tools`，否则 `--no-tools` 给空集（`--no-builtin-tools` 只去掉内置工具，保留扩展工具），否则设置里的 `defaultTools`（支持 `+name` / `-name` 增量写法），否则默认名单；最后去掉 `--exclude-tools` 列出的工具。
- `--tools`（以及 `--no-tools`）同时是注册白名单：不在名单里的工具（包括扩展注册的）连注册表都进不去，`--exclude-tools` 也在注册时就排除。设置里的 `defaultTools` 只决定初始激活，不限制注册。
- `powershell` 在所有平台上都能选，但只有在原生 Windows 上能运行，其他平台上执行时直接报错。会话的 `commandPrefix`、`shellPath` 设置只传给 `bash`。
- 内置扩展再提供两个默认不激活的工具：`codemode` 和 `tool_search`（第 7 节）。连上带资源的 MCP 服务器后，还会出现 `list_mcp_resources`、`list_mcp_resource_templates`、`read_mcp_resource` 三个资源工具。

### 6.2 保护措施

会产生输出的内置工具（`read`、`bash`、`powershell`、`grep`、`find`、`ls`）共用一套截断函数：默认最多 2000 行或 50KB（`DEFAULT_MAX_LINES`、`DEFAULT_MAX_BYTES`），先到哪个算哪个。每个工具的具体做法不同：

**`read`**

- 保留开头（`truncateHead`），并告诉模型从哪一行接着读。探针读一个 5000 行的文件，结果末尾是 `[Showing lines 1-2000 of 5000. Use offset=2001 to continue.]`。
- 给了 `limit` 时先按行数截取，再套用 2000 行 / 50KB 的上限。
- 第一行就超过 50KB 时，提示模型改用 `sed`。

**`bash`**

- 保留结尾（`truncateTail`）；截断时把完整输出写进临时文件，路径告诉模型。探针执行 `seq 1 100000`，结果末尾是 `[Showing lines 98001-100000 of 100000. Full output: /var/folders/…/pi-bash-….log]`。
- **没有默认超时**，schema 的说明直接写着 "no default timeout"。模型可以给 `timeout`（秒），上限约 24.8 天（2³¹−1 毫秒）。
- 超时或中止时，杀掉整个进程组（非 Windows 上进程以独立进程组启动，发 SIGKILL 给 `-pid`；Windows 上用 `taskkill /F /T`），然后**抛错**：探针里 `timeout: 1` 在 1005ms 后抛出 `Command timed out after 1 seconds`，中止在 306ms 后抛出 `Command aborted`。
- 退出码非零时**不抛错**，返回 `isError: true`，文本末尾是 `Command exited with code 3`。

**`edit`**

- 先精确匹配；找不到时退到模糊匹配：做 NFKC 规范化、去掉行尾空白、把弯引号和各种破折号、特殊空格换成 ASCII。探针里文件写的是 `const msg = “hello”;   `（弯引号加行尾空格），模型给的 `oldText` 是直引号，替换照样成功，而且只改了匹配到的那一行，同一个文件里其他行的弯引号和行尾空格保持原样。
- 每处 `oldText` 必须唯一：出现两次时报 `Found 2 occurrences of the text in a.ts. The text must be unique. …`。
- 多处编辑都对照原文件匹配，不能重叠；保留文件原有的 BOM 和 CRLF 换行。
- 工具描述和"找不到"的报错都说要精确匹配（`Could not find the exact text …`），模糊匹配是一层兜底，没有写进说明。

**`write`**：自动创建父目录，然后整个覆盖文件。

**`grep` / `find` / `ls`**

- 默认上限分别是 100 条匹配、1000 个结果、500 个条目，另有 50KB 的总量上限；`grep` 每行最多 500 字符。
- `grep` 用 `rg`，`find` 用 `fd`，本机没有时会自动下载（离线模式下不下载）。两者都会搜索隐藏文件；`find` 总是遵守 `.gitignore`，`grep` 只在 git 仓库里才遵守。

**同一文件的写入排队**：`edit` 和 `write` 都经过 `withFileMutationQueue`（按文件真实路径排队执行写入），对同一个文件的修改一个接一个执行，不同文件之间仍然并行。探针同时发起两个 `edit`，分别改同一个文件的第 1 行和第 2 行，两处修改都保留了下来。

**路径没有沙箱**：相对路径相对工作目录解析，`~` 会展开，开头的 `@` 会去掉，但不检查结果是否在工作目录之内。探针用绝对路径和 `../` 都读到了工作目录外的文件。coding-agent 的安全文档也写明：工作目录只决定资源发现和工具的默认位置，不阻止命令访问 Pi 进程能访问的其他路径。文档给出的隔离办法是用专门的系统用户运行、把整个 Pi 放进容器或虚拟机，或者只把内置工具放进隔离环境；扩展拥有完整权限，不构成安全边界。

### 6.3 可替换的底层操作

每个内置工具的工厂函数都接受一个 `operations` 选项，把真正碰文件系统或 shell 的那几步换掉，用来把工具搬到 SSH 或容器里执行：

- `ReadOperations`（`readFile`、`access`、可选的 `detectImageMimeType`）
- `BashOperations`（`exec`：收到命令、工作目录、输出回调、中止信号、超时和环境变量，返回退出码）
- `EditOperations`（`readFile`、`writeFile`、`access`）
- `WriteOperations`（`writeFile`、`mkdir`）
- `GrepOperations`（`isDirectory`、`readFile`；`rg` 本身仍在本地运行）
- `FindOperations`（`exists`、`glob`；自定义 `glob` 时不再调用 `fd`）
- `LsOperations`（`exists`、`stat`、`readdir`）

截断、模糊匹配、排队这些逻辑都在操作之上，换掉操作之后仍然有效。

---

## 7 · 工具从哪来

![图 5 · 工具从哪来，以及模型能看见哪些](/images/harness/pi/ch05/fig5-sources.png)

### 7.1 四个来源，一张注册表

- **内置工具**：`createAllToolDefinitions` 为每个会话创建全部 8 个，再按名单激活。
- **扩展**：`pi.registerTool(definition)`。注册时先检查 `parameters` 是一个对象（不能是 null 或数组），然后触发一次注册表刷新，所以会话中途注册的工具（例如 MCP 服务器连上之后）也能立即进入注册表。
- **SDK**：`createAgentSession({ customTools })`，和扩展工具进同一张表。
- **MCP 服务器**：内置的 `builtin:mcp` 扩展为每个远端工具调用一次 `pi.registerTool`。

同名时，扩展的工具替换内置工具。工具注册后不能注销，要撤回只能用 `exposure: "hidden"` 重新注册一次。

每个工具的 `exposure` 决定模型怎么够到它。这里的"可被调用"指能被其他工具经 `ctx.executeTool` 调用：

| exposure | 声明给模型 | 可被调用 | 注册时激活 |
|---|---|---|---|
| `direct` | 激活时 | 激活时 | 是 |
| `model-only` | 激活时 | 否 | 是 |
| `codemode` | 激活后 | 是 | 否 |
| `deferred` | 激活后 | 是 | 否 |
| `hidden` | 否 | 否 | 否 |

- 不填时是 `direct`；`defaultActive: false` 可以让 `direct` 工具注册时先不激活。
- `codemode` 和 `deferred` 注册时不激活，所以默认不声明；被 `setActiveTools` 或 `tool_search` 激活后照样声明给模型（第 7.3 节的 `deploy`）。两者的区别：`codemode` 类工具会列在 codemode 工具的描述里，`deferred` 不列，要靠搜索找到。
- `codemode` 和 `tool_search` 两个工具自身是 `model-only`：只给模型直接调用，不允许被脚本再调用。

激活集合由 `AgentSession` 管理：

- `getActiveToolNames()`（当前激活的工具名）
- `setActiveToolsByName(names)`（重设激活集合；扩展里对应 `pi.setActiveTools`）
- `_applyToolLoadout`（只保留已注册、非 hidden 的名字，运行各工具的 `prepareLoadout`，最后执行 `this.agent.state.tools = declared`）

最后那一句赋值，就是第 3 节里引擎拿到工具集的地方。

### 7.2 MCP

- **配置文件**：`~/.pi/agent/mcp.json`；项目被信任时，还会读 `<cwd>/.pi/mcp.json`。格式是常见的 `mcpServers`。扩展也可以用 `pi.registerMcpServer(name, config)`（在当前会话里添加一个服务器）从代码里添加。
- **项目覆盖**：项目里的条目不写 `command`、`url`、`type` 时，只覆盖同名全局服务器的 `enabled`、`exposure`、`toolExposure`，其余配置（包括凭据）沿用全局条目。
- **工具名**：`mcp__<server>__<tool>`，非 `[A-Za-z0-9_]` 的字符换成 `_`，最长 64 个字符；超长或重名时加 8 位哈希后缀。
- **暴露方式**：`codemode`（默认）、`deferred`、`direct`、`hidden`，可以用 `toolExposure` 按工具名（支持通配）单独设置。
- MCP 的 `codemode` 在注册表里记为 `deferred`。MCP 的 `codemode` 和 `deferred` 两种设置，区别只在于扩展顺带激活哪个发现工具：前者激活 `codemode`（设置了 `autoEnableCodemode: false` 时不激活），后者激活 `tool_search`。

所以默认配置下，MCP 工具一个都不直接声明给模型，模型要通过 codemode 脚本里的 `searchTools()` 找到并调用它们。工具很多的 MCP 服务器，不会把上下文撑满。

### 7.3 tool_search：按需加载

`tool_search` 来自内置扩展 `builtin:tool-search`，CLI 默认加载但不激活；SDK 会话要自己把 `createToolSearchExtension()` 加进 `extensionFactories`。它的工作方式：

1. 模型调用 `tool_search({ query, limit? })`，`limit` 默认 8。
2. 在已注册、尚未激活、`exposure` 为 `codemode` 或 `deferred` 的工具里，用 BM25 按名字、描述、schema 文本和命名空间排序。
3. 把命中的工具加进激活集合（`setActiveTools([...当前, ...命中])`）。
4. 返回 `Loaded N tools. They are available from your next call:`，后面每行一个工具名和描述的第一行。

`tool_search` 自己的描述是固定的，MCP 服务器连上或断开都不会改写它，避免它自己的声明变化让缓存失效。

探针用 `createAgentSession` 搭了一个会话：扩展注册一个普通工具 `weather` 和一个 `exposure: "deferred"` 的 `deploy`，设置里写 `defaultTools: ["+tool_search"]`。mock 依次让模型调用 `weather`、`tool_search("deploy service")`、`deploy`：

```text
开始时激活: read, bash, edit, write, tool_search, weather
req1 tools=[read, bash, edit, write, tool_search, weather, __pi_deferred_placeholder__(deferred)]
req2 同上                                           tool_result: "Berlin: 18°C"
req3 同上
     中途 system: tool_addition(deploy 完整定义)      tool_result: "Loaded 1 tool. They are available from your next call:\n- deploy: Deploy a service to production"
req4 同上                                           tool_result: "deployed api"
结束时激活: read, bash, edit, write, tool_search, weather, deploy
```

四次请求的顶层 `tools` 完全一样，`deploy` 的定义只出现在对话中途那条 system 消息里。模型换成 `claude-haiku-4-5`（不支持中途改工具）再跑一遍，req3 起顶层 `tools` 直接变成包含 `deploy` 的完整列表。对话记录里两次都是同样两条 system 消息：开头一条声明 6 个初始工具，加载后一条 `toolsAdded: [deploy]`。

两个细节：

- `weather` 带了 `promptSnippet`，出现在系统提示词的工具列表里；`deploy` 没带，系统提示词里找不到它。
- 恢复会话时，`AgentSession` 从对话记录里重放出上次激活的工具。MCP 服务器重连得比这一步晚，所以还没注册上的名字先记为待定，等工具注册进来再补上激活；下一次运行开始时，还没补上的待定名字就丢弃。

---

## 8 · 用法：写一个自定义工具

### 8.1 引擎层：直接交给 Agent

```typescript
import { Agent, type AgentTool } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";
import { readFile } from "node:fs/promises";
import { Type } from "typebox";

const parameters = Type.Object({
	path: Type.String({ description: "File path" }),
});

const countLines: AgentTool<typeof parameters, { path: string; chars: number }> = {
	name: "count_lines",
	label: "Count lines",
	description: "Count the lines of a text file.",
	parameters,
	// Some models say `file` instead of `path`
	prepareArguments(args) {
		if (args && typeof args === "object" && "file" in args && !("path" in args)) {
			const { file, ...rest } = args as Record<string, unknown>;
			return { ...rest, path: file } as { path: string };
		}
		return args as { path: string };
	},
	async execute(_toolCallId, params, signal, onUpdate) {
		onUpdate?.({ content: [{ type: "text", text: `reading ${params.path}` }], details: { path: params.path, chars: 0 } });
		const text = await readFile(params.path, { encoding: "utf8", signal });
		const lines = text.split("\n").length - (text.endsWith("\n") ? 1 : 0);
		return {
			content: [{ type: "text", text: `${lines} lines` }],
			details: { path: params.path, chars: text.length },
		};
	},
};

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6");
if (!model) throw new Error("Model not found");

const agent = new Agent({
	initialState: { model, systemPrompt: "Use tools when helpful.", tools: [countLines] },
	streamFn: models.streamSimple.bind(models),
});
agent.subscribe((event) => {
	if (event.type === "tool_execution_update") console.log("progress:", event.partialResult.details);
	if (event.type === "tool_execution_end") console.log("details:", event.result.details);
});
await agent.prompt("How many lines does notes.txt have?");
```

这段代码里的几个选择：

- **`signal` 直接交给 `readFile`**：中止时读文件跟着取消，不用自己监听。
- **`content` 只放模型需要的结论**，文件大小这类给界面看的数据放进 `details`。
- **`prepareArguments` 只做兼容**，不做校验。它返回的结果还要经过第 4.2 节的转换和校验。
- **工具集在构造时给出**：`tools` 和 `systemPrompt` 一起变成开头的 system 消息。运行中途要换工具，按第 3.2 节的方式经过 `prepareNextTurnWithContext`。

这段代码在 `strict` + `skipLibCheck` 下能通过 `tsc`。运行时需要 `ANTHROPIC_API_KEY`。探针用 mock 跑了它（把模型的 `baseUrl` 换成 mock 的地址，再给一个假的密钥），输出是：

```text
progress: { path: '/tmp/pi-ch05-notes.txt', chars: 0 }
details: { path: '/tmp/pi-ch05-notes.txt', chars: 8 }
tool_result sent: "4 lines"
```

### 8.2 产品层：通过扩展注册

coding-agent 自带的最小示例 `examples/extensions/hello.ts`：

```typescript
/**
 * Hello Tool - Minimal custom tool example
 */

import { Type } from "@earendil-works/pi-ai";
import { defineTool, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

const helloTool = defineTool({
	name: "hello",
	label: "Hello",
	description: "A simple greeting tool",
	parameters: Type.Object({
		name: Type.String({ description: "Name to greet" }),
	}),

	async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
		return {
			content: [{ type: "text", text: `Hello, ${params.name}!` }],
			details: { greeted: params.name },
		};
	},
});

export default function (pi: ExtensionAPI) {
	pi.registerTool(helloTool);
}
```

- `defineTool`（原样返回定义，只为让 TypeScript 推断出 `params` 的类型）。
- 把文件放进 `~/.pi/agent/extensions/`，或者放进 `<cwd>/.pi/extensions/`（项目被信任后才加载），或者在 SDK 里放进 `DefaultResourceLoader` 的 `extensionFactories`。
- 这个工具没设 `exposure`，按 `direct` 处理，注册即激活。

在这个基础上常加的几项：

- `promptSnippet`：让工具出现在系统提示词的工具列表里。
- `exposure: "deferred"`：工具很多、又不想全部常驻上下文时，配合 `tool_search` 按需加载（第 7.3 节的 `deploy`）。
- `renderCall` / `renderResult`：在终端里自定义调用和结果的显示。
- `ctx.executeTool`：在工具里组合调用其他工具，嵌套调用会记进 `nestedCalls`。

---

## 9 · 三个可以带走的方法

1. **工具声明跟着对话记录走**。把"模型可以调用哪些工具"写进对话本身，重放对话就能还原工具集；协议实现再按各家能力翻译成原生增删、就地新增或全量列表。换模型、恢复会话、压缩历史时，工具集都不会错位。
2. **参数在交给工具前先修补、转换，校验失败时把收到的参数还给模型**。模型的小偏差（字段名、字符串数字、多余的 null）在本地吸收掉；真错了，就给模型一份能对照修正的报错。
3. **结果按读者拆开**。给模型的、给界面的、给程序的放在不同字段，模型上下文里只放它需要的那部分。

---

## 10 · 关键数字

| 项 | 数量 |
|---|---|
| L1 `Tool` 的字段 | 4 个 |
| `AgentToolResult` 的字段 | 6 个 |
| coding-agent 内置工具 | 8 个，默认启用 4 个 |
| `exposure` 取值 | 5 种 |
| MCP 暴露方式 | 4 种 |
| 默认截断 | 2000 行或 50KB |
| `grep` / `find` / `ls` 默认条数 | 100 / 1000 / 500 |
| bash 默认超时 | 无 |
| `tool_search` 默认返回 | 8 个 |
| 嵌套调用记录上限 | 256 次 |
| 支持原生增删工具的模型 | 6 个 |
| 支持 grammar 工具的模型 | 111 个 |
| 内置模型目录 | 1536 个对话模型 |
| 工具相关目录的提交（北京时间 2026-08-01 至 2026-10-01，不含合并提交） | `core/tools/` 23 次，`extensions/` 26 次，`agent-loop.ts` 9 次 |

---

## 11 · 术语表

| 术语 | 含义 |
|---|---|
| **工具声明** | 发给模型的名字、说明和参数 schema |
| **激活** | 工具在当前声明集合里 |
| **`toolsAdded` / `toolsRemoved`** | system 消息上的工具增删 |
| **`exposure`** | 模型怎么够到一个工具 |
| **可被调用** | 能被其他工具经 `ctx.executeTool` 调用 |
| **`constrainedSampling`** | 请供应商在生成时约束参数 |
| **`structuredContent`** | 给程序调用方的结构化结果 |
| **`nestedCalls`** | 工具内部嵌套调用的摘要，只进会话 |
| **`defer_loading`** | Anthropic 的延迟声明标记 |

---

## 12 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 三层工具类型 | `packages/ai/src/types.ts` → `Tool`、`SystemMessage`、`ConstrainedSamplingConfig`；`packages/agent/src/types.ts` → `AgentTool`、`AgentToolResult`；`packages/coding-agent/src/core/extensions/types.ts` → `ToolDefinition`、`ToolExposure` |
| 包装 | `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` → `wrapToolDefinition` |
| 工具声明的重放 | `packages/ai/src/utils/transcript.ts` → `getCurrentTools`、`getToolStateChanges`、`resolveTranscriptTools` |
| 声明变化 | `packages/agent/src/agent-loop.ts` → `declareToolChanges` |
| 三种发法 | `packages/ai/src/api/anthropic-messages.ts` → `buildParams`、`convertMessages`、`DEFERRED_TOOL_PLACEHOLDER`；`openai-responses-shared.ts`；`openai-completions.ts` |
| 参数准备与校验 | `packages/agent/src/agent-loop.ts` → `prepareToolCall`；`packages/ai/src/utils/validation.ts` → `validateToolArguments` |
| 严格 schema 与语法 | `packages/ai/src/api/constrained-sampling.ts` |
| 执行与结果消息 | `packages/agent/src/agent-loop.ts` → `executePreparedToolCall`、`createToolResultMessage`、`runToolCall` |
| 图片的处理 | `packages/ai/src/api/transform-messages.ts` |
| 嵌套调用 | `packages/coding-agent/src/core/nested-tool-calls.ts` |
| 内置工具 | `packages/coding-agent/src/core/tools/`（`read.ts`、`bash.ts`、`edit.ts`、`edit-diff.ts`、`truncate.ts`、`file-mutation-queue.ts`、`path-utils.ts`） |
| 注册与激活 | `packages/coding-agent/src/core/agent-session.ts` → `_refreshToolRegistry`、`_applyToolLoadout`、`setActiveToolsByName` |
| MCP | `packages/coding-agent/src/extensions/mcp/`；`packages/coding-agent/src/core/mcp-servers.ts` |
| tool_search | `packages/coding-agent/src/extensions/tool-search/tool.ts` |
| codemode | `packages/coding-agent/src/extensions/codemode/`；`packages/coding-agent/docs/codemode.md` |
| 扩展示例 | `packages/coding-agent/examples/extensions/hello.ts`、`dynamic-tools.ts` |
| 官方说明 | `packages/coding-agent/docs/extensions.md`、`packages/coding-agent/docs/security.md`；`packages/agent/README.md` |
