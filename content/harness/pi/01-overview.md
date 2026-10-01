---
title: "Pi 源码分析 01 · 开篇：Pi 是什么，以及它的骨架"
date: 2026-10-01T10:00:00+08:00
description: "基于 earendil-works/pi v0.99.2 重写的开篇：三重身份、四层堆栈、依赖规则、一次 prompt 的旅程，以及三个月来的架构演变。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v0.99.2（2026-09-30），commit `8ce69e9d2`。
> 本章是对 [dgzhuya.com 第 1 章](https://www.dgzhuya.com/modules/ch01-overview)（基于 v0.80.2）的重写。保留它"工具 / 教材 / SDK"三重视角的好框架，但所有架构、数字和代码都按最新源码重新核对。三个月里 Pi 从 4 个包长到了 14 个目录，第一章的"骨架图"已经需要重画。

本章不深入实现细节，只回答三个问题：

1. **Pi 是什么？**——一句话定义与三重身份
2. **Pi 的骨架长什么样？**——分层、依赖规则、一次请求怎么穿过各层
3. **和三个月前比，变了什么？**——哪些结论仍成立，哪些已经过时


<!--more-->

---

## 0 · 阅读说明

| 项 | 说明 |
|---|---|
| 源码位置 | 本文所有路径都相对于仓库根目录，如 `packages/agent/src/agent-loop.ts` |
| 行号 | 只在必要时给出；行号会随版本漂移，**以符号名为主、行号为辅** |
| 标记 `*` | 表示实验性包，README 明确写了"API 随时变化" |
| 对照原博客 | 第 9 节给出了数字对照表，附录给出了勘误清单 |

---

## 1 · 一句话认识 Pi

> **Pi 是一个用 TypeScript 写的、极简且可扩展的终端编码 Agent，同时也是一套可以逐层拆开复用的 Agent SDK。**

拆开来看，Pi 有三重身份。后面每一节都会落到其中一个身份上：

| 身份 | 面向谁 | 你拿到的是什么 | 对应的层 |
|---|---|---|---|
| **编码工具** | 想要一个透明、可控的日常 Agent 的开发者 | `pi` 命令：交互 TUI，外加 print / JSON / RPC 模式 | L3 产品层 |
| **学习教材** | 想搞懂"生产级 Agent 怎么做"的人 | 一个能读完的核心：Agent Loop 的主体 `agent-loop.ts` 约 940 行 | L2 引擎层 |
| **开发 SDK** | 要做垂直领域 Agent 的团队 | 三层可独立使用的 npm 包 | L1 / L2 / L3 |

三重身份能共存，靠的是同一个设计决定：**按职责严格分层，每层只依赖下层**。这就是本章的主线。

---

## 2 · 全景：一张图看懂 Pi 的分层

![图 1 · Pi 架构全景](/images/harness/pi/ch01/fig1-overview.png)

Pi 的 14 个包目录（13 个可发布的 npm 包，外加私有的 `pi-evals`）可以归成三类：**主干堆栈**、**能力库**、**实验区**。

### 2.1 主干：四层堆栈

| 层 | 包 | 一句话职责 | 不知道什么 |
|---|---|---|---|
| **L3 产品层** | `@earendil-works/pi-coding-agent` | 把引擎装成"编码 Agent"：内置工具、系统提示词、会话树、扩展、CLI | —— |
| **L2 引擎层** | `@earendil-works/pi-agent-core` | 通用 Agent 运行时：状态、消息队列、Agent Loop、工具执行与事件流 | 不知道"read/bash"是什么，也不知道有 CLI |
| **L1 模型层** | `@earendil-works/pi-ai` | 统一的 LLM API：42 个供应商、10 种线协议、流式事件、token 与成本 | 不知道什么是 Agent |
| **L0 基础层** | `pi-telemetry`、`chord` | 厂商中立的遥测契约；上下文与取消、服务、复制状态 | 不知道什么是 LLM |

> 💡 原博客说 Pi 是"三层堆栈 + 一个 UI 库"。这个说法**对 L1–L3 仍然成立**，只是现在下面多垫了一层基础设施（L0）。日常阅读源码时，L0 几乎可以忽略：agent-core 主要通过 `chord/context` 传递取消信号和遥测父 span。

### 2.2 能力库：零内部依赖、可单独复用

| 包 | 做什么 | 谁在用 |
|---|---|---|
| `pi-tui` | 差分渲染的终端 UI 框架：组件树、编辑器、Markdown、全屏模式。约 1.9 万行，只依赖 `get-east-asian-width` 和 `marked` | coding-agent 的交互模式 |
| `pi-mcp` | 独立的 MCP 客户端，支持 stdio 和 Streamable HTTP，不依赖官方 SDK | coding-agent 的内置 `mcp` 扩展 |
| `pi-codemode` | QuickJS/WASM 沙箱，执行模型写的 JS。脚本唯一的能力是调用注入进来的工具 | coding-agent 的内置 `codemode` 扩展 |

它们的共同点是 `package.json` 里没有任何 `@earendil-works/*` 依赖。这是 Pi 最容易"拆下来就用"的部分。

### 2.3 实验区：Pi 正在长出来的新方向

| 包 | 方向 |
|---|---|
| `pi-durable` * | 持久化 Agent 运行时：消息、工具调用和状态都先落盘再展示，进程崩溃后能接着跑；内置子 Agent 和子任务 |
| `pi-server` * / `pi-protocol` * / `pi-client` * | 远程会话：CBOR 帧协议，把 Agent 跑在服务端，多个前端接入 |
| `pi-session-backend-sqlite-node` | 基于 `node:sqlite` 的会话存储后端 |
| `pi-evals`（私有） | 用 vitest-evals 写的行为评测 |

> ⚠️ coding-agent 里接入 server/protocol/client 的代码在 `src/experimental/` 下，**不会随 npm 包发布**（`package.json` 的 `files` 排除了 `dist/experimental`）。这三个包只是它的 devDependencies。所以对外的运行模式仍然是第 6 节列出的四种。

---

## 3 · 依赖规则：为什么箭头只朝下

![图 2 · 包依赖图](/images/harness/pi/ch01/fig2-deps.png)

上图来自各包 `package.json` 里真实的 `dependencies`，可以总结出三条规则：

**规则 1：单向依赖，底层不知道上层存在。**
`pi-ai` 不 import `pi-agent-core`，`pi-agent-core` 不 import `pi-coding-agent`。所以你可以只拿走下面任意几层，上面的全部丢掉。

**规则 2：类型逐层"加料"，下层类型从不为上层修改。**

- `pi-ai` 定义最基础的类型：`Message`、`Model`、`Tool`（只有 schema，没有 `execute`，因为 LLM 不需要知道工具怎么执行）。
- `pi-agent-core` 在此基础上加入 `AgentTool`（带 `execute`）、`AgentMessage`（可以通过声明合并扩展自定义消息）。
- `pi-coding-agent` 再加入 `ToolDefinition`（带 TUI 渲染）、`BashExecutionMessage`、`CompactionSummaryMessage` 等业务类型。

**规则 3：叶子包零依赖。**
`chord`、`pi-telemetry`、`pi-tui`、`pi-mcp`、`pi-codemode` 都没有内部依赖。新能力（MCP、codemode）先做成独立的叶子包，再由 L3 以"内置扩展"的形式接进来，而不是直接写进引擎。

> 🧠 **思考题**：为什么 MCP 不放进 `pi-agent-core`？
> 因为对引擎来说，MCP 工具和 `read` 工具没有区别，都只是一个 `AgentTool`。把 MCP 放在 L3 的扩展里，引擎就能保持"不知道工具从哪来"。想用别的 MCP 实现，替换扩展就行，不用 fork 引擎。

---

## 4 · 主干三层逐层拆解

### 4.1 L1 · pi-ai：只管调模型

**职责**：把几十家供应商的差异压平，成为一套 `stream / complete` 接口和一套统一的流式事件。

| 概念 | 含义 | 位置 |
|---|---|---|
| `KnownApi`（10 种） | **线协议**：`anthropic-messages`、`openai-completions`、`openai-responses`、`google-generative-ai`、`bedrock-converse-stream`、`mistral-conversations` 等 | `packages/ai/src/types.ts` |
| `KnownProvider`（42 个） | **供应商**：anthropic、openai、google、deepseek、openrouter、zai、moonshotai、xiaomi…… 多个供应商可以共用一种协议 | 同上 |
| `Models` 集合 | 注册供应商、查模型、发请求的入口：`createModels()` / `builtinModels()` | `packages/ai/src/models.ts` |
| 模型类型 | 除了 `chat`，还有 `image`（生图）和 `classifier`（分类） | `types.ts` 中的 `ModelTypeMap` |
| `/compat` 入口 | 旧的全局 API（`getModel` / `stream`），README 标注为"临时兼容，未来移除" | `packages/ai/src/compat.ts` |

**最小示例**（摘自 `packages/ai/README.md`，当前推荐写法）：

```typescript
import type { Context } from "@earendil-works/pi-ai";
import { builtinModels } from "@earendil-works/pi-ai/providers/all";

const models = builtinModels();                          // 注册所有内置供应商
const model = models.getModel("openai", "gpt-4o-mini")!;

const context: Context = {
  systemPrompt: "You are helpful.",
  messages: [{ role: "user", content: "Hello!" }],
};

const s = models.stream(model, context);                 // 认证由供应商解析（如 OPENAI_API_KEY）
for await (const event of s) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const finalMessage = await s.result();
```

> 📌 和原博客的区别：原文从 `pi-ai/compat` 导入 `getModel`，那是旧 API。新代码请用 `Models` 集合。

### 4.2 L2 · pi-agent-core：只管跑循环

**职责**：实现"模型思考 → 调工具 → 看结果 → 再思考"这个循环。它对"编码"一无所知，可以拿来做客服 Agent、数据分析 Agent 等任何场景。

| 组件 | 要点 |
|---|---|
| `Agent` 类（`agent.ts`） | 持有状态（系统提示词、模型、消息、工具）。方法有 `prompt` / `continue` / `steer` / `followUp` / `abort` / `subscribe` / `waitForIdle` |
| `agentLoop`（`agent-loop.ts`，约 940 行） | **双层循环**。内层：有工具调用或插队消息（steering）就继续下一轮。外层：内层结束后检查后续消息（follow-up），有就再来 |
| 钩子 | `beforeToolCall` / `afterToolCall` / `transformContext` / `convertToLlm` / `prepareRequest` / `finishTurn` 等，上层通过它们注入业务逻辑 |
| `AgentEvent`（10 种） | `agent_start/end` · `turn_start/end` · `message_start/update/end` · `tool_execution_start/update/end` |
| `streamFn` | 唯一必填的选项。引擎不直接调 pi-ai，而是调你注入的函数，所以模型层可以替换 |

**最小示例**（摘自 `packages/agent/README.md`）：

```typescript
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6")!;

const agent = new Agent({
  initialState: { systemPrompt: "You are a helpful assistant.", model },
  streamFn: models.streamSimple.bind(models),
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("Hello!");
```

> 📌 和原博客的区别：原文写"model/tools/systemPrompt 在调用 prompt() 时传入"。实际上它们放在构造参数 `initialState` 里（运行中也可以改 `agent.state`）。

### 4.3 L3 · pi-coding-agent：把引擎装成产品

**职责**：在 `Agent` 外面包一层 `AgentSession`。它负责组装系统提示词和上下文文件、注册内置工具、加载扩展和资源、把会话落盘成 JSONL 树、自动压缩与重试，并把事件扩展成 `AgentSessionEvent` 广播给各种前端。

| 子系统 | 关键文件（`packages/coding-agent/src/core/`） |
|---|---|
| 会话编排 | `agent-session.ts`（约 4300 行，L3 的心脏） |
| SDK 入口 | `sdk.ts` 的 `createAgentSession()`，内部 `new Agent({ streamFn, ... })` |
| 工具 | `tools/`：read / bash / edit / write / grep / find / ls / powershell |
| 扩展 | `extensions/`（运行时、类型、事件派发）；内置扩展在 `src/extensions/` |
| 会话存储 | `session-manager.ts`（JSONL 树、分叉、分支摘要） |
| 提示词 | `system-prompt.ts`、`resource-loader.ts`（AGENTS.md、SYSTEM.md、Skills、模板） |
| 模型 | `model-runtime.ts`、`model-registry.ts`（内置目录 + `models.json` + 认证） |

**最小示例**（摘自 `docs/sdk.md` 与 `examples/sdk/02-custom-model.ts`）：

```typescript
import { createAgentSession, ModelRuntime } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const model = modelRuntime.getModel("anthropic", "claude-opus-4-5");

const { session } = await createAgentSession({ model, thinkingLevel: "medium", modelRuntime });
try {
  session.subscribe((e) => { if (e.type === "turn_end") console.log("一轮结束"); });
  await session.prompt("Read the codebase and explain the architecture.");
  console.log(session.getLastAssistantText());
} finally {
  session.dispose();
}
```

> 📌 和原博客的区别有两点：
> - `createAgentSession()` 返回的是 `{ session, extensionsResult, ... }`，不是 session 本身。
> - 模型通过 `ModelRuntime` 获取。
>
> 另外，**SDK 会话不会自动加载内置扩展**。要用 MCP，需要自己加 `createMcpExtension()`。

### 4.4 侧库 · pi-tui：与 Agent 无关的终端 UI

pi-tui 不在堆栈链上。只有交互模式用它，它本身也不认识任何 `pi-*` 包，所以任何 Node.js 终端程序都能用。三个月里它从约 1.2 万行长到约 1.9 万行，新增的主要是：

- 全屏（alt-screen）渲染器 `TuiAltScreen`，可以在常规模式和全屏模式之间切换
- 终端里的 Mermaid 与 LaTeX 渲染
- 鼠标滚轮加速、搜索、可拖拽的滚动条

核心 `tui.ts` 反而从 1714 行瘦身到 1493 行，因为代码拆到了 `tui-main-screen.ts` 和 `tui-alt-screen.ts` 两个渲染器里。

---

## 5 · 把三层串起来：一次 prompt 的旅程

![图 3 · 一次 prompt 的旅程](/images/harness/pi/ch01/fig3-flow.png)

| 步骤 | 层 | 发生了什么 | 扩展能插手的点 |
|---|---|---|---|
| ① 输入 | L3 | 四种前端最终都调用 `AgentSession.prompt()` | `input` |
| ② 准备 | L3 | 展开模板和 Skill；组装系统提示词、AGENTS.md 和当前分支的消息 | `before_agent_start` |
| ③ 进入循环 | L2 | `Agent.prompt()` → `runAgentLoop`；`turn_start` | `context` |
| ④ 调模型 | L1 | `streamFn` → `Models.streamSimple()`，按 `model.api` 选择协议 | `before_provider_request` / `before_provider_headers` |
| ⑤ 网络 | 外部 | HTTP / SSE | `provider_stream_event`（只读观察） |
| ⑥ 归一化 | L1 | 各家原始流 → `AssistantMessageEvent`（text_delta、toolcall_* 等） | —— |
| ⑦ 执行工具 | L2 | 发出 `message_*`；有 toolCall 就执行 `before → execute → after`，然后**回到 ③** | `tool_call`（可拦截）/ `tool_result` |
| ⑧ 落盘 | L3 | 写 JSONL 会话树；超过阈值自动压缩；出错自动重试 | `turn_end`、`session_compact` 等 |
| ⑨ 呈现 | L3 | `AgentSessionEvent` 广播给 TUI / JSON / RPC / SDK | —— |

这张表是读懂后续章节的地图：

- 第 3 章讲 ③⑦ 的循环
- 第 4 章讲 ④⑥
- 第 5 章讲 ⑦ 的工具执行
- 第 6、8 章讲 ② 的上下文
- 第 7 章讲 ⑨ 的事件
- 第 9、10 章讲 ⑧

---

## 6 · 产品层：工具、内置扩展与运行模式

### 6.1 内置工具（8 个）

| 工具 | 默认启用 | 说明 |
|---|---|---|
| `read` / `bash` / `edit` / `write` | ✅ | `DEFAULT_TOOL_NAMES`，见 `settings-manager.ts` |
| `grep` / `find` / `ls` | ❌ | 需要时通过 `--tools` 或 `defaultTools` 开启 |
| `powershell` | ❌ | v0.80.2 之后新增，仅限原生 Windows，例如 `"defaultTools": ["-bash", "+powershell"]` |

`defaultTools` 现在支持 `+name` / `-name` 增量写法，可以只增删个别工具，不用把默认列表整个重写一遍。

### 6.2 内置扩展（v0.99.0 起）

| 扩展 | 作用 | 默认状态 |
|---|---|---|
| `builtin:mcp` | 从 `mcp.json` 连接 MCP 服务器（stdio / HTTP / OAuth），提供 `/mcp` 和 `pi mcp add\|list\|login…` | 已加载，配置了服务器才生效 |
| `builtin:codemode` | `codemode` 工具：模型写 JS，在 QuickJS 沙箱里并行调用其他工具 | 已注册，默认不激活（MCP 工具默认通过它暴露） |
| `builtin:tool-search` | `tool_search` 工具：按需把"延迟声明"的工具暴露给模型 | 已注册，默认不激活 |
| `builtin:llama.cpp` | llama.cpp 供应商与 `/llama` 命令 | 已加载 |

关闭方式：在 `pi config` 的 Built-in 区域关闭，或者在设置里写 `"extensions": ["-builtin:mcp"]`。`--no-extensions` 会把它们一起关掉。前三个是 `replaceable`：如果某个第三方扩展注册了同名的工具或命令，就会**替换**内置版本，不会和它并存。

### 6.3 四种运行模式

| 模式 | 启动方式 | 用途 |
|---|---|---|
| 交互 | `pi`（可加 `--tui-mode fullscreen`） | 日常编码 |
| Print / JSON | `pi -p "..."`、`--mode json` | 脚本、CI；JSON 模式逐行输出事件（v0.84 起 `message_update` 只发增量） |
| RPC | `--mode rpc` | 通过 stdin/stdout 交换 JSONL，接入非 Node 程序 |
| SDK | `createAgentSession()` | 嵌入到你自己的 Node 应用 |

官方文档 `docs/how-pi-works.md` 的原话是："All interfaces use the same agent and session mechanisms." 四种模式只是 L3 之上的四种呈现方式。

### 6.4 配置文件地图

```text
~/.pi/agent/                    # 用户级
├── settings.json               # 默认模型、defaultTools、extensions 开关…
├── models.json                 # 自定义供应商 / 模型（任意 OpenAI 兼容网关）
├── mcp.json                    # MCP 服务器
├── auth.json                   # /login 凭据
├── SYSTEM.md / APPEND_SYSTEM.md
├── extensions/ skills/ prompts/ themes/
└── sessions/                   # JSONL 会话，按工作目录分组

<项目>/.pi/                     # 项目级（需要先信任项目才会加载）
└── settings.json  mcp.json  SYSTEM.md  extensions/  skills/  prompts/  themes/
```

上下文文件：每个目录只取下面几个中的第一个匹配项：`AGENTS.override.md` → `AGENTS.md` → `CLAUDE.md`。`AGENTS.override.md` 是 v0.84 新增的。

---

## 7 · 定制阶梯：从"用 Pi"到"造自己的 Agent"

![图 5 · 定制阶梯](/images/harness/pi/ch01/fig5-levers.png)

原博客把定制手段总结为"五根杠杆"（扩展、技能、提示词模板、主题、Pi 包）。这里换一个角度，**按定制深度排成阶梯**，并把 SDK 也放进来：

| 级 | 手段 | 你写的是 | 典型用途 |
|---|---|---|---|
| ① | 设置 | JSON | 换默认模型、增删工具、接内网网关、挂 MCP |
| ② | 资源 | Markdown | AGENTS.md 项目约定、SYSTEM.md 换人设、Skills 按需加载知识、提示词模板、主题 |
| ③ | 扩展 | TypeScript | `registerTool` / `registerCommand` / `registerShortcut` / `registerProvider` / `registerVirtualModel` / `registerMcpServer`、生命周期钩子、TUI 组件 |
| ④ | Pi Packages | npm / git 包 | 把 ①–③ 打包：`pi install npm:@foo/bar` |
| ⑤ | SDK | 你的应用代码 | 按需选层：L3 `createAgentSession`、L2 `Agent`、L1 `createModels` |

两点需要更正：

- **热重载需要手动触发**：改完扩展后执行 `/reload`，不用重启会话，但它不会监听文件自动生效。会自动监听文件变化的只有主题。
- **官方示例更多了**：`examples/extensions/` 下现在有 70 个顶层示例和 9 个子目录，包括 `subagent/`、`plan-mode/`、`sandbox/`、`permission-gate.ts`、`todo.ts`、`ssh.ts`。

---

## 8 · 减法哲学的演变：Pi 不做什么

原博客引用了官网的 "What we didn't build" 清单。三个月后，这张表需要加一列"现状"：

| 原本不做的 | 当时的理由 | **v0.99.2 现状** |
|---|---|---|
| MCP 支持 | MCP 会一次性把大量工具描述灌进上下文 | ⚠️ **已内置**（`builtin:mcp`），但用两个机制规避原问题：工具默认通过 `codemode` 暴露，或用 `tool_search` 延迟声明，不再一次性灌入上下文。可以关掉 |
| 子 Agent | 增加复杂度，降低可观察性 | 核心仍然没有；有 `examples/extensions/subagent/`；实验性的 `pi-durable` 提供了子 Agent 和子任务 |
| 权限弹窗 | 弹窗疲劳，沦为"安全表演" | 核心仍然默认 YOLO；有 `permission-gate.ts`、`sandbox/` 示例，还有项目信任机制（project trust） |
| 计划模式 | 写到 plan.md 更持久 | 核心仍然没有；有 `examples/extensions/plan-mode/` |
| 后台 bash | tmux 已经解决了 | 不变 |
| 内置待办 | TODO.md 更灵活 | 核心仍然没有；有 `examples/extensions/todo.ts` |

**结论**："减法"没有被放弃，但做法从"不做"演进成了"**做成可关闭、可替换的内置扩展**"。

- 引擎（L2）依然保持干净，MCP 只是 L3 的一个扩展。
- 默认配置变得更"开箱即用"。
- 判断 Pi 哲学的标准不再是"有没有某功能"，而是"这个功能是不是焊死的"。

---

## 9 · 三个月发生了什么

![图 4 · 版本时间线](/images/harness/pi/ch01/fig4-timeline.png)

| 指标 | v0.80.2（原博客） | **v0.99.2（本文）** |
|---|---|---|
| 包目录数 | 4 | **14**（13 个可发布 + 私有 evals） |
| 默认工具 | read · write · edit · bash | 不变 |
| 全部内置工具 | 7 | **8**（+ powershell） |
| 内置扩展 | 无 | **4**（mcp、codemode、tool-search、llama.cpp） |
| `KnownProvider` | 35 | **42** |
| 线协议 `KnownApi` | —— | 10 |
| 模型类型 | chat | **chat · image · classifier** |
| `agent-loop.ts` | 748 行 | 940 行 |
| pi-tui 源码 | 约 12,100 行 | **约 19,200 行** |
| `system-prompt.ts` | 173 行 | 216 行；静态模板约 200 个英文词，原博客写的是 90 词 |
| 运行模式 | 4 | 4（远程 server 模式仍是实验性，不对外发布） |
| `examples/extensions` | 77 项 | 80 项（70 个 `.ts` + 9 个目录 + README） |

---

## 10 · 术语表

| 术语 | 定义 | 别和它混淆 |
|---|---|---|
| **线协议（Api）** | 与 LLM 通信的请求/响应格式，如 `anthropic-messages` | 供应商：多个供应商可以共用一种协议 |
| **供应商（Provider）** | 提供模型和认证的一方，如 `deepseek`、`openrouter` | —— |
| **Models 集合** | pi-ai 中注册供应商、查找模型、发起请求的对象 | 旧的 `/compat` 全局函数 |
| **ModelRuntime** | coding-agent 中的模型运行时：内置目录 + `models.json` + 凭据 | pi-ai 的 `Models` |
| **Agent** | L2 的通用 Agent 实例：状态 + 循环 + 事件 | AgentSession |
| **AgentSession** | L3 对 Agent 的封装：提示词、工具、扩展、会话持久化 | Agent |
| **Turn（轮）** | 一次模型调用，加上它触发的工具执行 | Run：一次 prompt 可能包含多轮 |
| **Steering / Follow-up** | 运行中插队的消息 / 运行结束后排队执行的消息 | —— |
| **扩展（Extension）** | 加载到 L3 的 TypeScript 模块，可以注册工具、命令、钩子、UI | Skill：Skill 是 Markdown 知识，扩展是代码 |
| **内置扩展（builtin:*）** | 随 coding-agent 发布、可关闭和替换的扩展 | 核心功能 |
| **Skill** | 按需加载的指令包，平时只在提示词里放一段描述 | 提示词模板 |
| **会话树** | JSONL 中的条目通过 parent 指针形成一棵树，当前分支提供上下文 | 线性聊天记录 |
| **压缩（Compaction）** | 用摘要条目替换旧消息，原始条目仍然保留在树里 | 删除历史 |
| **codemode** | 让模型写 JS、在沙箱里组合调用工具的执行方式 | 普通的逐个工具调用 |

---

## 11 · 源码导航与后续路线

**按"想搞懂什么"找文件：**

| 想搞懂 | 从这里读 |
|---|---|
| Agent Loop 怎么转 | `packages/agent/src/agent-loop.ts` → `runLoop` |
| 一个请求怎么发到模型 | `packages/ai/src/models.ts` → `streamSimple`；`packages/ai/src/api/` |
| 工具怎么定义和执行 | `packages/coding-agent/src/core/tools/`；`packages/agent/src/agent-loop.ts` → `executeToolCalls` |
| 系统提示词怎么拼 | `packages/coding-agent/src/core/system-prompt.ts` |
| 会话怎么存和分叉 | `packages/coding-agent/src/core/session-manager.ts`；`docs/session-format.md` |
| 扩展能做什么 | `packages/coding-agent/src/core/extensions/types.ts`；`docs/extensions.md` |
| MCP 怎么接入 | `packages/coding-agent/src/extensions/mcp/`；`packages/mcp/` |
| 官方一页纸原理 | `packages/coding-agent/docs/how-pi-works.md` |

**后续章节**（沿用原博客的章节划分，内容按 v0.99.2 重写）：

1. 开篇总览（本章）
2. 三层架构与类型分层
3. Agent Loop
4. 模型调用
5. 工具系统
6. 消息系统
7. 事件驱动
8. 上下文工程
9. 上下文压缩
10. 会话管理

---

## 附录 · 原博客第 1 章勘误清单（对照 v0.99.2）

| # | 原文 | 现状 |
|---|---|---|
| 1 | "四个核心包" | 主干仍是 ai / agent-core / coding-agent + tui，但仓库已有 14 个包目录，见第 2 节 |
| 2 | "刻意不构建 MCP" | v0.99.0 起 MCP 是内置扩展（可关闭） |
| 3 | "pi-tui 约 12000 行" | 约 19,200 行 |
| 4 | "35 个 KnownProvider" | 42 个 |
| 5 | "4 核心 + 3 辅助工具" | 4 个默认 + 4 个可选（新增 powershell） |
| 6 | "静态系统提示词约 90 词" | 静态模板约 200 词 |
| 7 | "改扩展文件，会话立即生效" | 需要执行 `/reload`（无需重启）；只有主题会自动监听文件 |
| 8 | `getModel` 从 `pi-ai/compat` 导入 | compat 是临时兼容入口；推荐用 `createModels()` / `builtinModels()`，L3 用 `ModelRuntime` |
| 9 | `const session = await createAgentSession(...)` | 返回 `{ session, ... }`，需要解构 |
| 10 | "model/tools 在 prompt() 时传入 Agent" | 放在构造参数 `initialState` |
| 11 | "pi-orchestrator（v0.80.x 新增）" | v0.80.3 引入，v0.81.0 更名为 `pi-server`，仍是实验性 |
| 12 | "models.json schema 在 model-registry.ts:158-218" | `model-registry.ts` 现在只有 230 行，schema 相关逻辑见 `model-config.ts` 和 `docs/models.md` |
