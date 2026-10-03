---
title: "Pi 源码分析 01 · 开篇：Pi 是什么，以及它的骨架"
date: 2026-10-01T10:00:00+08:00
description: "基于 earendil-works/pi v1.0.0 之后的 main（a276dabe5）的开篇：三重身份、四层堆栈、依赖规则、一次 prompt 的旅程，以及定制方式。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` main 分支 `v1.0.0-25-ga276dabe5`（2026-10-03），commit `a276dabe5`。文中所有架构、数字和代码均以该版本源码为准。

本章不深入实现细节，只回答两个问题：

1. **Pi 是什么？**——一句话定义与三重身份
2. **Pi 的骨架长什么样？**——分层、依赖规则、一次请求怎么穿过各层、怎么定制

<!--more-->

---

## 0 · 阅读说明

| 项 | 说明 |
|---|---|
| 源码位置 | 路径默认相对于仓库根目录，如 `packages/agent/src/agent-loop.ts`；单独出现的 `docs/`、`examples/` 指 `packages/coding-agent/` 下的目录 |
| 行号 | 只在必要时给出；行号会随版本漂移，**以符号名为主、行号为辅** |
| 标记 `*` | 表示实验性包，README 标注为 experimental，不保证兼容 |

---

## 1 · 一句话认识 Pi

> **Pi 是一个用 TypeScript 写的、极简且可扩展的 Agent Harness：开箱是终端编码 Agent，同时也是一套可以逐层拆开复用的 Agent SDK。**

官方 README 的第一句是 "Pi is a minimal, extensible agent harness that you can make your own."

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

Pi 的 13 个包目录（12 个可发布的 npm 包，外加私有的 `pi-evals`）可以归成三类：**主干堆栈**、**能力库**、**实验区**。

### 2.1 主干：四层堆栈

| 层 | 包 | 一句话职责 | 不知道什么 |
|---|---|---|---|
| **L3 产品层** | `@earendil-works/pi-coding-agent` | 把引擎装成"编码 Agent"：内置工具、系统提示词、会话树、扩展、CLI | —— |
| **L2 引擎层** | `@earendil-works/pi-agent-core` | 通用 Agent 运行时：状态、消息队列、Agent Loop、工具执行与事件流 | 不知道"read/bash"是什么，也不知道有 CLI |
| **L1 模型层** | `@earendil-works/pi-ai` | 统一的 LLM API：42 个供应商、10 种线协议、流式事件、token 与成本 | 不知道什么是 Agent |
| **L0 基础层** | `pi-telemetry` | 厂商中立的遥测契约 | 不知道什么是 LLM |

> 💡 读源码时，L0 几乎可以忽略：agent-core 的内部依赖只有 pi-ai，pi-ai 也只从 pi-telemetry 引入一个 `TelemetryContext` 类型。真正的主线是 L1 → L2 → L3 这三层。

### 2.2 能力库：零内部依赖、可单独复用

| 包 | 做什么 | 谁在用 |
|---|---|---|
| `pi-tui` | 差分渲染的终端 UI 框架：组件树、编辑器、Markdown、全屏模式。约 1.9 万行，只依赖 `get-east-asian-width` 和 `marked` | coding-agent 的交互模式；扩展和工具的渲染接口也用它的组件类型 |
| `pi-mcp` | 独立的 MCP 客户端，支持 stdio 和 Streamable HTTP，不依赖官方 SDK | coding-agent 的内置 `mcp` 扩展 |
| `pi-codemode` | QuickJS/WASM 沙箱，执行模型写的 JS。脚本唯一的能力是调用注入进来的工具 | coding-agent 的内置 `codemode` 扩展 |
| `chord` | 应用组合运行时：上下文与取消、服务、复制状态、RPC。README 说明它不是 Pi 专用的包，其他应用也能用 | pi-durable、pi-protocol、pi-client、pi-server；coding-agent 虽然声明了依赖，但只在不发布的实验代码里用 |

它们的共同点是 `package.json` 里没有任何 `@earendil-works/*` 依赖。这是 Pi 最容易"拆下来就用"的部分。

### 2.3 实验区：Pi 正在探索的方向

| 包 | 方向 |
|---|---|
| `pi-durable` * | 持久化 Agent 运行时：消息、工具调用和状态都先落盘再展示，进程崩溃后能接着跑；内置子 Agent 和子任务 |
| `pi-server` * / `pi-protocol` * / `pi-client` * | 远程会话：CBOR 帧协议；服务端托管基于 pi-durable 的会话，多个前端接入 |
| `pi-evals`（私有） | 用 vitest-evals 写的行为评测 |

> ⚠️ coding-agent 里接入 server/protocol/client 的代码在 `src/experimental/`、`src/cli/experimental/` 和 `src/client/` 下，**不会随 npm 包发布**（`package.json` 的 `files` 排除了这三个目录对应的 `dist/` 产物）。这三个包只是它的 devDependencies。这些目录还直接 import 了 pi-durable 和 chord，同样不发布。所以对外的运行模式仍然是第 6 节列出的四种。

---

## 3 · 依赖规则：为什么箭头只朝下

![图 2 · 包依赖图](/images/harness/pi/ch01/fig2-deps.png)

上图来自各包 `package.json` 里真实的 `dependencies`，可以总结出三条规则：

**规则 1：单向依赖，底层不知道上层存在。**
`pi-ai` 不 import `pi-agent-core`，`pi-agent-core` 不 import `pi-coding-agent`。所以你可以只拿走下面任意几层，上面的全部丢掉。

**规则 2：类型逐层"加料"，下层类型从不为上层修改。**

- `pi-ai` 定义最基础的类型：`Message`、`Model`、`Tool`（只有 schema，没有 `execute`，因为 LLM 不需要知道工具怎么执行）。
- `pi-agent-core` 在此基础上加入 `AgentTool`（带 `execute`）、`AgentMessage`（可以通过 `CustomAgentMessages` 声明合并扩展自定义消息）。
- `pi-coding-agent` 再加入 `ToolDefinition`（带 TUI 渲染）、`BashExecutionMessage`、`CompactionSummaryMessage` 等业务类型。

**规则 3：叶子包零依赖。**
`pi-telemetry`、`chord`、`pi-tui`、`pi-mcp`、`pi-codemode` 都没有内部依赖。像 MCP、codemode 这样的能力，先做成独立的叶子包，再由 L3 以"内置扩展"的形式接进来，而不是直接写进引擎。

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
| `Models` 集合 | 注册供应商、查模型、发请求的入口：`createModels()` / `builtinModels()` | `packages/ai/src/models.ts`；`builtinModels` 在 `providers/all.ts` |
| 模型类型 | 除了 `chat`，还有 `image`（生图）和 `classifier`（分类） | `types.ts` 中的 `ModelTypeMap` |
| `/compat` 入口 | 旧的全局 API（`getModel` / `stream`），README 标注为"临时兼容，未来移除"，新代码不要用 | `packages/ai/src/compat.ts` |

**最小示例**（据 `packages/ai/README.md` 的 Quick Start 改写）：

```typescript
import type { Context } from "@earendil-works/pi-ai";
import { builtinModels } from "@earendil-works/pi-ai/providers/all";

const models = builtinModels();                          // 注册所有内置供应商
const model = models.getModel("openai", "gpt-4o-mini")!;

const context: Context = {
  systemPrompt: "You are helpful.",
  messages: [{ role: "user", content: "Hello!", timestamp: Date.now() }],
};

const s = models.stream(model, context);                 // 认证由供应商解析（如 OPENAI_API_KEY）
for await (const event of s) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const finalMessage = await s.result();
```

### 4.2 L2 · pi-agent-core：只管跑循环

**职责**：实现"模型思考 → 调工具 → 看结果 → 再思考"这个循环。它对"编码"一无所知，可以拿来做客服 Agent、数据分析 Agent 等任何场景。

| 组件 | 要点 |
|---|---|
| `Agent` 类（`agent.ts`） | 持有状态（系统提示词、模型、消息、工具），在构造参数 `initialState` 里传入。方法有 `prompt` / `continue` / `steer` / `followUp` / `abort` / `subscribe` / `waitForIdle` |
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
const model = models.getModel("anthropic", "claude-sonnet-4-6");
if (!model) throw new Error("Model not found");

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model,
  },
  streamFn: models.streamSimple.bind(models),
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    // Stream just the new text chunk
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("Hello!");
```

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
| 模型 | `model-runtime.ts`、`model-registry.ts`、`model-config.ts`（内置目录 + `models.json` + 认证） |

**最小示例**（据 `docs/sdk.md` 与 `examples/sdk/02-custom-model.ts` 改写）：

```typescript
import { createAgentSession, ModelRuntime } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const model = modelRuntime.getModel("anthropic", "claude-opus-4-5");
if (!model) throw new Error("Model not found");

const { session } = await createAgentSession({ model, thinkingLevel: "medium", modelRuntime });
try {
  session.subscribe((e) => { if (e.type === "turn_end") console.log("一轮结束"); });
  await session.prompt("Read the codebase and explain the architecture.");
  console.log(session.getLastAssistantText());
} finally {
  session.dispose();
}
```

> 📌 两个容易踩的点：
> - `createAgentSession()` 返回的是 `{ session, extensionsResult, ... }`，需要解构。
> - **SDK 会话不会自动加载内置扩展**。要用 MCP，需要把 `createMcpExtension()` 加进 `DefaultResourceLoader` 的 `extensionFactories`；MCP 工具默认经 codemode 暴露，所以通常还要加 `createCodemodeExtension()`（或 `createToolSearchExtension()`）。MCP 在 `session_start` 时连接服务器，所以还要调用 `session.bindExtensions()`。

### 4.4 侧库 · pi-tui：与 Agent 无关的终端 UI

pi-tui 不在堆栈链上：主要由交互模式使用（扩展和工具渲染接口里的组件类型也来自它），它本身不认识任何 `pi-*` 包，任何 Node.js 终端程序都能用。核心能力：

- **差分渲染**：只重绘变化的部分，配合同步输出（CSI 2026）避免闪烁
- **两种渲染器**：常规模式 `TuiMainScreen` 和全屏模式 `TuiAltScreen`，运行时可以切换；coding-agent 默认用全屏
- **组件**：`Editor`、`Markdown`（支持 LaTeX；Mermaid 图由 coding-agent 通过 transformer 接入）、`SelectList`、`SettingsList`、`ScrollView` 等

---

## 5 · 把三层串起来：一次 prompt 的旅程

![图 3 · 一次 prompt 的旅程](/images/harness/pi/ch01/fig3-flow.png)

| 步骤 | 层 | 发生了什么 | 扩展能插手的点 |
|---|---|---|---|
| ① 输入 | L3 | 四种前端最终都调用 `AgentSession.prompt()` | —— |
| ② 准备 | L3 | 展开模板和 Skill；组装系统提示词、AGENTS.md 和当前分支的消息 | `input` / `before_agent_start` |
| ③ 进入循环 | L2 | `Agent.prompt()` → `runAgentLoop`；`turn_start` | `context` / `context_with_system` |
| ④ 调模型 | L1 | `streamFn` → `Models` 接口的 `streamSimple()`（coding-agent 里由 `ModelRuntime` 实现）：先按 `model.provider` 找供应商，再按 `model.api` 选协议实现 | `before_provider_request` / `before_provider_headers` |
| ⑤ 网络请求 | 外部 | HTTP / SSE | —— |
| ⑥ 原始流 | 外部 | 各家供应商的原始流事件 | `provider_stream_event`（只读观察） |
| ⑦ 归一化 | L1 | 原始流 → `AssistantMessageEvent`（text_delta、toolcall_* 等） | —— |
| ⑧ 执行工具 | L2 | 发出 `message_*`；有 toolCall 就执行 `before → execute → after`，然后**回到 ③** | `tool_call`（可拦截）/ `tool_result` |
| ⑨ 落盘 | L3 | 写 JSONL 会话树；超过阈值自动压缩；出错自动重试 | `turn_end`、`session_compact` 等 |
| ⑩ 呈现 | L3 | `AgentSessionEvent` 广播给 TUI / JSON / RPC / SDK | —— |

这张表是读懂后续章节的地图：

- Agent Loop 章节讲 ③⑧ 的循环
- 模型调用章节讲 ④–⑦
- 工具系统章节讲 ⑧ 的工具执行
- 会话与压缩章节讲 ⑨
- 上下文工程章节讲 ②
- 扩展与事件、运行模式与界面章节讲 ⑩

---

## 6 · 产品层：工具、内置扩展与运行模式

### 6.1 内置工具（8 个）

| 工具 | 默认启用 | 说明 |
|---|---|---|
| `read` / `bash` / `edit` / `write` | ✅ | `DEFAULT_TOOL_NAMES`，见 `settings-manager.ts` |
| `grep` / `find` / `ls` | ❌ | 需要时通过 `--tools` 或 `defaultTools` 开启 |
| `powershell` | ❌ | 仅限原生 Windows，例如 `"defaultTools": ["-bash", "+powershell"]` |

`defaultTools` 支持 `+name` / `-name` 增量写法，可以只增删个别工具，不用把默认列表整个重写一遍。

### 6.2 内置扩展

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
| 交互 | `pi`（默认全屏；`--tui-mode regular` 切回常规滚动） | 日常编码 |
| Print / JSON | `pi -p "..."`、`--mode json` | 脚本、CI；JSON 模式逐行输出事件，`message_update` 只携带增量 |
| RPC | `--mode rpc` | 通过 stdin/stdout 交换 JSONL，接入非 Node 程序 |
| SDK | `createAgentSession()` | 嵌入到你自己的 Node 应用 |

安装：官方推荐 `curl -fsSL https://pi.dev/install.sh | sh`（Windows 用 `install.ps1`），也可以 `npm install -g --ignore-scripts @earendil-works/pi-coding-agent`，需要 Node.js 22.19 以上。

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
└── settings.json  mcp.json  SYSTEM.md  APPEND_SYSTEM.md  extensions/  skills/  prompts/  themes/
```

上下文文件：每个目录只取下面几个中的第一个匹配项：`AGENTS.override.md` → `AGENTS.md` → `CLAUDE.md`。

---

## 7 · 定制阶梯：从"用 Pi"到"造自己的 Agent"

![图 4 · 定制阶梯](/images/harness/pi/ch01/fig4-levers.png)

Pi 的定制手段可以**按深度排成阶梯**：

| 级 | 手段 | 你写的是 | 典型用途 |
|---|---|---|---|
| ① | 设置 | JSON | 换默认模型、增删工具、接内网网关、挂 MCP |
| ② | 资源 | Markdown | AGENTS.md 项目约定、SYSTEM.md 换人设、Skills 按需加载知识、提示词模板、主题 |
| ③ | 扩展 | TypeScript | `registerTool` / `registerCommand` / `registerShortcut` / `registerProvider` / `registerVirtualModel` / `registerMcpServer`、生命周期钩子、TUI 组件 |
| ④ | Pi Packages | npm / git 包 | 把 ①–③ 打包：`pi install npm:@foo/bar` |
| ⑤ | SDK | 你的应用代码 | 按需选层：L3 `createAgentSession`、L2 `Agent`、L1 `createModels` |

两点补充：

- **重载方式**：改完扩展、Skill 或提示词后执行 `/reload` 即可生效，不用重启会话。当前生效的用户主题（`~/.pi/agent/themes/<name>.json`）改动后会自动重载；其他来源的主题改完后要执行 `/reload`。
- **官方示例**：`examples/extensions/` 下有 70 个顶层示例和 9 个子目录，包括 `subagent/`、`plan-mode/`、`sandbox/`、`permission-gate.ts`、`todo.ts`、`ssh.ts`，是写扩展最好的参考。

---

## 8 · 减法哲学：核心不做什么，以及怎么补

Pi 的核心刻意保持很小。官方 README 的说法是 "Pi ships with powerful defaults but skips features like sub-agents and plan mode." 下面这些常见能力，核心要么不做，要么做成可以关掉、可以替换的扩展：

| 能力 | 核心的态度 | 需要时怎么补 |
|---|---|---|
| MCP | 做成内置扩展 `builtin:mcp`，可以关闭。工具默认通过 `codemode` 暴露，或用 `tool_search` 延迟声明，避免一次性把大量工具描述灌进上下文 | 不需要时 `-builtin:mcp`；也可以换成第三方 MCP 扩展 |
| 子 Agent | 核心没有 | `examples/extensions/subagent/`；实验性的 `pi-durable` 提供子 Agent 与子任务 |
| 权限审批 | 核心默认不弹窗（YOLO） | `permission-gate.ts`、`sandbox/` 示例；项目级配置需要先信任项目才会加载 |
| 计划模式 | 核心没有 | `examples/extensions/plan-mode/`；也可以自己维护一个 plan.md |
| 后台 bash | 核心没有 | 可以把长任务放进 tmux 等终端复用工具里跑（本文建议，官方文档没有专门说明） |
| 待办清单 | 核心没有 | `examples/extensions/todo.ts`；也可以自己维护一个 TODO.md |

背后的逻辑是：**引擎（L2）保持干净，产品层（L3）的能力都是可以拆下来的模块**。评价 Pi 的标准不是"有没有某个功能"，而是"这个功能是不是焊死的"。

---

## 9 · 关键数字

| 指标 | 数值 |
|---|---|
| 包目录 | 13 个（12 个可发布 npm 包 + 私有 evals） |
| 内置工具 | 8 个，默认启用 4 个（read · bash · edit · write） |
| 内置扩展 | 4 个（mcp · codemode · tool-search · llama.cpp） |
| 供应商 `KnownProvider` | 42 个 |
| 线协议 `KnownApi` | 10 种 |
| 模型类型 | chat · image · classifier |
| `agent-loop.ts` | 约 940 行 |
| `agent-session.ts` | 约 4300 行 |
| pi-tui 源码 | 约 1.9 万行 |
| 系统提示词静态模板 | 约 200 个英文词（运行时再拼接工具说明、上下文文件、Skills） |
| 运行模式 | 4 种（交互 · Print/JSON · RPC · SDK） |
| 官方扩展示例 | 70 个顶层 `.ts` + 9 个子目录 |

---

## 10 · 术语表

| 术语 | 定义 | 别和它混淆 |
|---|---|---|
| **线协议（Api）** | 与 LLM 通信的请求/响应格式，如 `anthropic-messages` | 供应商：多个供应商可以共用一种协议 |
| **供应商（Provider）** | 提供模型和认证的一方，如 `deepseek`、`openrouter` | —— |
| **Models 集合** | pi-ai 中注册供应商、查找模型、发起请求的对象 | `/compat` 里的旧全局函数 |
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

## 11 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| Agent Loop 怎么转 | `packages/agent/src/agent-loop.ts` → `runLoop` |
| 一个请求怎么发到模型 | `packages/ai/src/models.ts` → `streamSimple`；`packages/ai/src/api/` |
| 工具怎么定义和执行 | `packages/coding-agent/src/core/tools/`；`packages/agent/src/agent-loop.ts` → `executeToolCalls` |
| 系统提示词怎么拼 | `packages/coding-agent/src/core/system-prompt.ts` |
| 会话怎么存和分叉 | `packages/coding-agent/src/core/session-manager.ts`；`packages/coding-agent/docs/session-format.md` |
| 扩展能做什么 | `packages/coding-agent/src/core/extensions/types.ts`；`packages/coding-agent/docs/extensions.md` |
| MCP 怎么接入 | `packages/coding-agent/src/extensions/mcp/`；`packages/mcp/` |
| 官方一页纸原理 | `packages/coding-agent/docs/how-pi-works.md` |

**后续章节**：

1. 开篇总览（本章）
2. 三层架构与类型分层
3. Agent Loop
4. 模型调用
5. 工具系统
6. 会话
7. 上下文压缩
8. 上下文工程
9. 扩展与事件
10. 运行模式与界面
