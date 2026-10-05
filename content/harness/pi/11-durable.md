---
title: "Pi 源码分析 番外 · pi-durable：先提交、再展示的持久化引擎"
date: 2026-10-05T17:30:00+08:00
description: "接着第三章和第六章往下讲：pi 仓库里的第二套 Agent 引擎 pi-durable。它把一次运行拆成带检查点的任务，所有改动只走 Session 的一条提交线，提交成功后界面才能看到；进程被杀之后重新打开，生成任务重发请求，工具调用按 replay 策略重跑或记为中断。本章用崩溃探针实际验证这些行为，并和第三章的 Agent 逐项对照。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.3-2-gb9ab918c6（2026-10-05），commit `b9ab918c6`。v1.0.3 之后的两个提交都没有改动 `packages/durable`。pi-durable 的 README 第一行写着 **Experimental. The API changes without notice between releases.**，所以本章是 2026-10-05 这一天的快照：结构和机制以该版本源码为准，API 细节以后可能会变。

第三章讲了 `Agent` 和 `agentLoop` 怎样在内存里跑完一次运行，第六章讲了 coding-agent 怎样在事件到来时把消息写进会话文件。这一章讲仓库里的另一套引擎 `@earendil-works/pi-durable`：它把持久化放进引擎内部，任何东西都先提交到存储，再让界面看到。

<!--more-->

本章要点：

- **与 Agent 并存，不在 pi 命令里**：pi-durable 只依赖 pi-ai、chord 和两个第三方库（diff、typebox），不依赖 pi-agent-core。它的使用者只有 coding-agent 的 `src/experimental/` 目录：一个终端 Agent、一个旅行规划示例和客户端/服务端的会话 worker。这些目录不参与编译，也不随 npm 发布，coding-agent 的 `package.json` 里也没有声明这个依赖。1.0.0 版的 agent-core 删掉了旧的实验引擎 `AgentHarness`，CHANGELOG 里写的接替者就是它。
- **唯一的写路径**：所有改动都排进 `Session` 的一条提交队列，按"回调 → 收集写入 → `storage.commit` → 合并进内存 → 通知监听者"的顺序走。视图、`watch` 和 `watchEvents` 都挂在最后一步上，没有别的发布通道。探针里一次带工具调用的运行共 运行本身有 16 次提交（连同之前创建根对话共 17 次），没有一帧视图出现在某次存储提交进行期间。
- **一次运行是一串任务**：`pi.generation` 任务分 `prepare`、`request`、`retry`、`poll`、`tools` 五个阶段，每个工具调用是一个 `pi.tool` 任务。每次换阶段都提交检查点。工具遵循"效果夹心"：先提交意图（最终参数和 replay 策略），再执行副作用，最后把结果和任务终态放在同一次提交里写入。
- **崩溃之后**：重新打开时，状态为 running 的任务改回 pending，检查点不动；`resume()`，或 `submit()`、`wait()` 这类需要进展的调用，才让调度器开始派发。生成任务把已提交的部分答案追加成 `aborted` 条目（模型看不到它），再发一次请求。工具任务如果已经提交了意图，只有存储的策略和当前注册的工具都声明 `replay: "safe"` 才重跑；否则写入 `interrupted` 错误结果，附带崩溃前已提交的输出。内置的 read、write、edit、bash 都没有声明，一律不重跑。
- **流式进度会丢一点**：部分答案和工具输出默认每 100 ms 提交一次，崩溃最多丢掉这个窗口。SQLite 后端是 WAL 模式加 `synchronous = NORMAL`，JSONL 后端从不在普通提交时刷 `main.jsonl`，所以"已提交"能扛住进程崩溃，不保证扛住掉电。

---

## 0 · 阅读说明

- 本章只讲 `packages/durable`。用到第三章（`Agent`、`agentLoop`、钩子与事件）和第六章（`SessionManager`、`appendMessage`）的结论时不再重复。
- pi-durable 的源码约 1.95 万行，其中 `harness/` 6.7 千行、`storage/` 2.9 千行、`testing/`（给自定义后端用的一致性测试）2.7 千行、`env/` 2.2 千行。本章讲 `session/`、`harness/` 和 `storage/` 的核心路径，`env/` 和 `tools/` 只交代它们的位置。压缩（`pi.compaction`）、任务图和子任务只简单带过。
- 行为都用探针跑过。探针安装 npm 上的 `@earendil-works/pi-durable`、`pi-ai`、`chord` 1.0.3，用 pi-ai 的 faux 供应商代替真实模型：
  - **P1**：装一个提示词片段，把一次问答写到 JSONL 存储，查看磁盘上的文件和转录。
  - **P2**：工具执行到一半时用 `SIGKILL` 杀掉进程，再用同一个 SQLite 文件重新打开；`replay` 分别为默认值和 `"safe"`。
  - **P3**：模型流式输出到一半时 `SIGKILL`，再重新打开。
  - **P4**：给 `MemoryStorage.commit` 包一层日志，把每次存储提交的开始和结束、视图帧、`watchEvents` 批次和钩子调用记在同一条时间线上。
  - **P5**：工具运行时插入 steer 和 follow-up，以及在同样情况下调用 `abort()`。
- 术语：
  - **Session**：一个打开的存储加上一条提交队列。所有读写都经过它。
  - **Harness**：Session 加上调度器、扩展注册表和内置任务。`Harness.open()` 返回它。
  - **对话**（conversation）：一份转录，由不可变的条目组成。`Conversation` 句柄本身不带状态。
  - **条目**（entry）：转录里的一条记录，例如 `pi.user`、`pi.assistant`。
  - **文档**（document）：和转录存在一起的带类型 JSON，在提交里修改。
  - **任务**（task）：带检查点的持久状态机，每换一个阶段都提交一次。
  - **提交**（commit）：一次原子写入，可以同时追加条目、改文档、建任务。

---

## 1 · 它在仓库里的位置

![图 1 · pi-durable 在仓库里的位置：与 Agent 并列的第二套引擎](/images/harness/pi/ch11/durable-fig1-position.png)

### 1.1 两套引擎，两种持久化分工

pi 命令走的是左边这条路。`Agent` 自己不写任何文件，agent-core 的源码里没有一处文件系统 import。持久化是 coding-agent 的事：`AgentSession` 订阅 `Agent` 的事件，在 `message_end` 时调用 `sessionManager.appendMessage` 把消息追加到会话文件（第六章）。

pi-durable 把这件事放进了引擎内部。README 的第二句就是它的承诺：

> Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown. If the process dies mid-turn, reopening the storage picks the work up where it stopped.

两套引擎在同一层，互不 import。规范文件 `docs/spec.md`（标题是 "Pico5 specification"，约 4700 行）没有拿 `Agent` 做比较，也没有计划把 pi 命令迁过来。它的来历写在 agent-core 1.0.0 的 CHANGELOG 里：那一版删掉了实验性的 `AgentHarness`、会话存储和 pico3，并写明 "Use `@earendil-works/pi-durable` for durable sessions"。

### 1.2 包的事实

| 项 | 值 |
|---|---|
| npm 包名 | `@earendil-works/pi-durable`，2026-09-19 首次发布（0.0.1），1.0.0 起附带 README 和 CHANGELOG，当前 1.0.3 |
| 运行时依赖 | `@earendil-works/pi-ai`、`@earendil-works/chord`、`diff`（edit 工具算差异）、`typebox`（工具参数 schema） |
| 导出入口 | 10 个：`.`、`./env`、`./env/node`、`./tools`、`./storage/memory`、`./storage/jsonl`、`./storage/jsonl/node`、`./storage/sqlite`、`./storage/sqlite/node`、`./testing` |
| 体积约束 | 仓库根目录的 `check:entry-graphs` 限制 `.` 入口最多引入 62 个源文件，且不能引入 pi-ai 的根入口；`check:browser-smoke` 用 esbuild 按浏览器平台打包 `.`、`/env`、`/storage/jsonl`、`/storage/sqlite` 四个入口，结果里不能出现 env、jsonl、sqlite 的三个 Node 适配器 `node.ts` |
| 历史 | 2026-09-18 从 agent-core 的 `src/pico/` 移成独立包，到基线共 96 次非合并提交 |

从 pi-ai 只取用了很窄的一组子路径：模型调用的类型、`validateToolArguments`（按 schema 校验并转换工具参数）、`uuidv7`（生成供应商会话 ID）、`isContextOverflow`（判断"上下文过长"错误）、`isRetryableAssistantError`、`retryDelayMs`（判断可重试错误、计算退避）、`utils/transcript` 里比较和生成工具声明的函数，以及几个上下文估算函数。根入口只导入类型。

chord 是仓库里的通用库（`package.json` 的描述是 "Application composition runtime for services, replicated state, RPC, and plugins"），不依赖任何 Pi 包。pi-durable 用它的三样东西：`Context`（每个异步调用都要传，负责取消）、`chord/delta`（跟踪 JSON 的改动并生成操作列表）和 `replicatedState`（可订阅的只读状态）。

### 1.3 谁在用它

运行时的使用者只有 coding-agent 的实验目录（另外还有 coding-agent 的实验测试和仓库的浏览器打包检查脚本），三个使用者：

| 使用者 | 入口 |
|---|---|
| 终端 coding agent | `src/experimental/durable/main.ts` |
| 旅行规划示例 | `src/experimental/vacation/main.ts` |
| 会话 worker | `src/experimental/session-worker.ts` |

它们都没有 npm 脚本或 bin，要在仓库根目录用源码运行。终端 Agent 的 README 给的命令是：

```bash
node --import ./packages/coding-agent/src/experimental/source-resolver.ts packages/coding-agent/src/experimental/durable/main.ts
node --import ./packages/coding-agent/src/experimental/source-resolver.ts packages/coding-agent/src/experimental/durable/main.ts --continue
```

旅行规划示例换成 `vacation/main.ts`；会话 worker 由 `PI_EXPERIMENTAL=1 ./pi-test.sh server` 和 `client` 启动。

它们不会出现在用户安装的 pi 里：

- coding-agent 的 `tsconfig.build.json` 不编译 `src/experimental`；
- `package.json` 的 `files` 用 `!dist/experimental` 再排除一次；
- 发布前的 `coding-agent-consumer.mjs` 检查已安装的包里没有 `dist/experimental`。

pi-durable 在 coding-agent 的依赖列表里也没有出现，实验代码靠工作区和根 `tsconfig.json` 的路径映射找到它。

终端 Agent 的 README 写明了它演示什么："Kill the process in the middle of a tool call and start it again with `--continue`: the interrupted call gets an interrupted result and the turn finishes. Nothing in the TUI handles recovery; it only renders the conversation view." 它也列出了还没有的东西：会话列表和恢复选择器、分叉与树导航、扩展、提示词模板、图片、`/login`。

---

## 2 · 骨架

![图 2 · 包内四层：从存储到对话句柄](/images/harness/pi/ch11/durable-fig2-layers.png)

### 2.1 最小用法

README 的 Quick Start（原样摘录）：

```typescript
import { BACKGROUND_CONTEXT } from "@earendil-works/chord/context";
import { createModels } from "@earendil-works/pi-ai/models";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";
import { AssistantEntry, createRegistry, Harness, MemoryStorage } from "@earendil-works/pi-durable";

const context = BACKGROUND_CONTEXT;

const models = createModels();
models.setProvider(openaiProvider()); // reads OPENAI_API_KEY

const harness = await Harness.open(new MemoryStorage(), { models, registry: createRegistry() }, context);
const root = await harness.root(context, { agent: { model: { provider: "openai", modelId: "gpt-6-sol" } } });

const submission = await root.submit({ type: "input", content: "What is the capital of France?" }, context);
const settled = await submission.wait(context);
if (settled.status === "done" && settled.type === "input") {
	const answer = await root.commit((tx) => tx.entry(AssistantEntry, settled.answer), context);
	console.log(answer?.model?.[0]);
}
await harness.close(context);
```

和第三章的 `agent.prompt()` 对比，有三处不同：

- 每个异步调用都多一个 `context` 参数。取消一次 `wait` 只取消这次等待，不取消后台的工作。
- `submit()` 不等运行结束。它把输入持久地收下，返回一个 `Submission`；`wait()` 等到这条输入被回答（`done`）或确定没有回答（`unanswered`，带原因）。
- 结果不在返回值里，而在存储里：`settled.answer` 是一个条目 ID，要通过 `root.commit(tx => tx.entry(...))` 读出来。

### 2.2 四层

| 层 | 主要类型 | 做什么 |
|---|---|---|
| 对外句柄 | `Harness`、`Conversation` | `Harness.open(storage, options, context)` 打开；`root()`、`createConversation()`、`fork()` 取得对话；对话上有 `submit`（提交输入）、`configure`（改模型、工具、指令）、`abort`（中止）、`reset`（开始新上下文）、`compact`（手动压缩）、`viewState` / `watch`（观察） |
| 调度与扩展 | 调度器、`Registry`、内置任务 | 调度器负责派发任务、恢复、中止级联；注册表持有本进程安装的扩展；三个内置任务 `pi.generation`、`pi.tool`、`pi.compaction` |
| 唯一写入线 | `Session`、`Transaction` | 所有改动排队提交；条目、任务、submission 和文档在同一个事务里写 |
| 存储后端 | `Storage` | 16 个方法，`commit(writes)` 原子写入 8 种 `StorageWrite` |

存进存储的东西分四类：

- **条目**：内置 6 种：`pi.user`（用户消息）、`pi.assistant`（模型回复）、`pi.tool-result`（工具结果）、`pi.system`（系统提示词或工具列表的变化）、`pi.reset`（新上下文的起点）、`pi.compaction`（压缩摘要）。用 `defineEntry` 可以加自己的种类。
- **文档**：内置 5 个，每个对话一份：`pi.agent`（这个对话用的模型、扩展、工具、指令、工作目录）、`pi.provider`（发给供应商的会话 ID）、`pi.live`（正在进行的生成和工具调用）、`pi.inbox`（排队的输入）、`pi.usage`（用量和花费）。
- **任务**：每个任务存一条完整记录，包括状态和检查点。
- **submission**：每条收下的输入，带可选的 `requestId` 用来去重。

P1 把一次问答写到 JSONL 存储后，目录里有一个 `main.jsonl` 和 5 个 `doc-N.jsonl`（每个内置文档一个边车文件）。`main.jsonl` 里是 6 行 `{"type":"commit","seq":N,"writes":[…]}`，转录是 `pi.user → pi.system → pi.assistant`。

### 2.3 存储后端

| 后端 | 导入 | 说明 |
|---|---|---|
| 内存 | 包根的 `MemoryStorage` | 不持久化 |
| SQLite | `openNodeSqliteStorage(file)`，来自 `/storage/sqlite/node` | 一个数据库文件，用 Node 自带的 `node:sqlite`。WAL 模式、`synchronous = NORMAL`：提交能扛住进程崩溃，掉电或宿主机故障可能丢最新的一次 |
| JSONL | `openNodeJsonlStorage(directory, context)`，来自 `/storage/jsonl/node` | 一个目录里的只追加文件。每次提交先写边车文件，最后在 `main.jsonl` 追加提交标记；打开时截掉没有标记的尾巴。传 `{ fsync: true }` 会在写标记前刷边车文件；`main.jsonl` 本身在普通提交时不刷盘 |

同一时刻只能有一个进程打开同一个存储，包里没有跨进程锁。终端 Agent 自己在会话目录上加了锁文件，崩溃留下的锁 10 秒后失效。

`/storage/sqlite` 和 `/storage/jsonl` 是不依赖 Node API 的核心实现。给它一个异步的 SQLite 门面或一个 `FileSystem`，就能在 Bun 或 Cloudflare Durable Objects 里跑。自定义后端可以用 `/testing` 里的 `registerStorageConformance` 跑同一套一致性测试。

---

## 3 · 唯一的写路径

![图 3 · 唯一的写路径：Session 的一次提交](/images/harness/pi/ch11/durable-fig3-commit.png)

### 3.1 一次提交的六步

"先提交、再展示"由 `Session` 的一个私有方法 `#runCommit` 保证。所有调用方（`conversation.commit`、任务里的 `runtime.commit`、工具里的 `api.commit`，以及引擎自己写部分答案、工具输出、检查点）最终都走到这里：

1. **排队**：`#enqueue` 把这次提交挂到同一条 Promise 链的尾部，一次只跑一个。
2. **回调**：在一个新的 `Transaction` 上运行调用方的 `change(tx)`。回调里追加条目、通过 `tx.doc(...)` 拿到文档草稿直接改字段、建任务。
3. **收集**：`tx.settleSuccess()` 把这些改动收集成一批 `StorageWrite`。文档草稿由 chord 的 `track()` 跟踪，改动变成一组操作，存成增量；满足 `checkpointWhen` 条件时改存完整的基值（`pi.live` 在没有任何东西运行时存基值）。批次为空就直接返回。
4. **落盘**：`await this.#storage.commit(writes, …)`。一旦开始，调用方的取消信号不再打断它。
5. **合并**：`tx.adopt(seq)` 把改动并入内存里的文档跟踪器。
6. **发布**：`#publish` 同步调用所有提交监听者。

视图、`watch`、`watchEvents` 和任务图都只通过 `subscribeCommits` 拿到改动，所以它们看到的永远是已经落盘的状态。规范把这条写成必须满足的不变式："A document update is published only after its storage commit succeeds." 和 "All visible progress is durable. There is no volatile publication path."

失败分三种：

- 回调抛错：事务回滚，什么都不写，错误抛给调用方。
- 存储抛 `StorageRejected`：保证这批写入一条都没生效，Session 还能继续用。
- 存储抛其他错误，或者存储已提交但合并失败：Session 被"毒化"，之后的所有操作都失败，必须重新打开。规范的说法是 "An uncertain storage failure is fatal to the open Session."

### 3.2 探针：提交和视图的先后

P4 给 `MemoryStorage.commit` 包了一层，在开始和结束时各记一行，同时订阅 `viewState()` 和 `watchEvents()`。一次"模型调用工具 → 工具输出两段 → 模型回答"的运行，时间线的开头是这样的：

```text
storage.commit#2 start [entry,submission,task,document.change]
storage.commit#2 end seq=2
  view frame entries=1
  events: message_start message_end submission run_start turn_start
storage.commit#3 start [task]
storage.commit#3 end seq=3
storage.commit#4 start [entry,task]
storage.commit#4 end seq=4
  view frame entries=2
  events: message_start message_end
```

提交 #1 是 `harness.root()` 创建根对话和 5 个内置文档，运行本身是 #2 到 #17 共 16 次提交。脚本检查"有没有视图帧出现在某次 `start` 和它的 `end` 之间"，结果是 0。

### 3.3 流式输出也走同一条路

模型的流式事件不会直接交给界面。`streamResponse` 只在内存里留着最新的部分答案，用定时器节流，每隔 `progress.partialIntervalMs`（默认 100 ms）提交一次到 `pi.live.generation.message`，同一时刻最多有一次这样的提交在进行。流结束后，`classify` 在一次提交里追加最终的 `pi.assistant` 条目并清掉部分答案。工具运行中的输出用同样的方式按 `outputIntervalMs`（默认 100 ms）提交到 `pi.live` 的工具槽位。

所以界面上的流式文字其实是"已提交的部分答案"。崩溃时，最多丢掉最后一个节流窗口里的内容，而界面从来不会显示存储里没有的东西。P3 在模型流式输出到一半时杀掉进程：被杀前视图里显示的是 `"one two three four five six seve"`，重新打开后从存储读到的 `pi.live.generation.message` 也是这一串，一字不差。存储在远端的宿主可以把间隔调大，例如 `{ partialIntervalMs: 500, outputIntervalMs: 500 }`。

### 3.4 三种观察方式

| 方式 | 拿到什么 | 慢消费者 |
|---|---|---|
| `conversation.viewState(context)` | 只读的 chord 状态：`entries`（当前上下文的转录）和 `docs`（5 个内置文档） | 订阅者总是看到最新值 |
| `conversation.watch(context)` | 连接时的视图，加上每次提交的精确 chord 操作，一次一个回调 | 积压超过 100 帧时，换成一帧完整的最新视图 |
| `watchEvents(harness, conversationId, context)` | 一个 `snapshot`，加上从每次提交翻译出来的事件批次（实验性） | 落后超过 100 批时，重新发一个 `snapshot` |

`watchEvents` 是给习惯第十章 JSON 模式事件的消费者准备的适配层，共 22 种事件：

- 和 coding-agent 的会话事件同名的：`message_start` / `message_update` / `message_end`（消息开始、增量、结束）、`tool_execution_start` / `tool_execution_update` / `tool_execution_end`（工具开始、输出增量、结束）、`turn_start` / `turn_end`（一轮开始、结束）、`auto_retry_start` / `auto_retry_end`（自动重试）、`compaction_start` / `compaction_end`（压缩）。前 8 个也是第三章 `Agent` 的事件。
- 这里独有的：`snapshot`（完整初始状态）、`run_start` / `run_end`（一次运行的开始和结束，带输入 ID）、`inbox_update`（排队内容变化）、`submission`（submission 状态变化）、`deferred_poll`（延迟响应的下次轮询时间）、`entry_appended`（其他条目被追加）、`agent_changed`（对话配置变化）、`usage_changed`（用量变化）、`task_failed`（某个任务失败）。

这些事件都是从提交里推出来的，不是另一条通道。晚加入或断线重连的客户端从当前视图开始，不会补发历史事件。

### 3.5 "已提交"能扛住什么

| 故障 | 结果 |
|---|---|
| 进程崩溃、`SIGKILL` | 已提交的都在。没提交的是：最后一个节流窗口里的部分答案和工具输出、正在运行的副作用（见第 5 节） |
| 掉电、宿主机故障 | SQLite（`synchronous = NORMAL`）可能丢最新的提交；JSONL 不论是否开 `fsync` 都可能丢最新一次提交，`fsync` 只保证留下来的提交标记不会跑在边车数据前面 |
| 两个进程同时打开 | 不支持，没有跨进程锁 |

---

## 4 · 一次运行是一串任务

![图 4 · 一次运行拆成的任务与提交（探针 P4，一轮工具调用）](/images/harness/pi/ch11/durable-fig4-run.png)

### 4.1 从输入到回答

README 用一张小图概括一次被回答的输入：

```text
submit(input) → pi.user
  pi.generation → pi.system (only if the prompt or tools changed), pi.assistant (tool calls)
    pi.tool × n → pi.tool-result × n   (owned by the generation, which waits for them)
  pi.generation → pi.assistant (answer) → submission done
```

第三章的 `runLoop` 是外层处理后续消息、内层处理工具调用和插队消息的两层 `while` 循环，全部在一次函数调用里完成。这里没有这样的循环：每一轮模型调用是一个新的 `pi.generation` 任务，它为本轮的每个工具调用建一个 `pi.tool` 任务，等这些任务结束后再建下一个 `pi.generation` 任务交棒。任务之间的数据都在存储里：哪条输入在跑、谁在等谁，都记在 `pi.live.run` 和任务记录里。

### 4.2 任务是什么

任务由 `defineTask` 定义：一个名字、一个版本号、一个初始检查点，以及一组按名字区分的阶段处理器和一个 `abort` 处理器。处理器通过 `runtime.commit` 返回下一个状态：

- `running` + 新检查点：进入下一阶段；
- `waiting` + `on`（等哪些任务）+ `policy`（`allSettled` 或 `failFast`）：不运行任何代码，等它们结束；
- `terminal` + 结果：结束。

调度器每次从任务的当前检查点取出 `phase` 字段，调用同名的处理器。重新打开存储后也是这样，所以恢复不需要单独的代码路径：一个阶段的处理器必须能接受"我上次可能跑到一半"。

### 4.3 生成任务的五个阶段

| 阶段 | 做什么 | 结束时提交 |
|---|---|---|
| `prepare` | 解析这个对话的 agent（模型、工具、提示词片段），渲染系统提示词；只有片段或工具变了才追加 `pi.system` 条目；必要时先等一次阻塞压缩 | 检查点改为 `request`，固定模型、思考级别、请求选项和 `cutoff`（这次请求包含的最新条目） |
| `request` | 先把上次留下的部分答案转成条目（第 5 节），再设 `pi.live.generation = { attempt }`；运行 `beforeRequest` 钩子，流式请求模型，节流提交部分答案 | 由 `classify` 决定：回答、进入工具轮、重试、轮询或失败 |
| `retry` | 睡到退避时间 | 检查点改回 `prepare`，`attempt + 1` |
| `poll` | 供应商返回"延迟响应"时，按句柄轮询结果 | 同 `request` |
| `tools` | 等本轮的工具任务；顺序执行的工具在这里一个一个启动 | 交棒：新建下一个生成任务，或结束运行 |

`classify` 里，"上下文过长"的错误会触发一次压缩后重试；可重试的错误按 `retry` 设置退避（默认最多 3 次，基础延迟 2000 ms）；带工具调用的回复由 `startToolRound` 在一次提交里追加 `pi.assistant` 条目、建好 `pi.tool` 任务并在 `pi.live` 里开好工具槽位：并行的一轮一次建好全部任务，顺序的一轮只建第一个，其余在 `tools` 阶段逐个启动；请求里没有提供的工具直接得到 `tool_unavailable` 结果。

`request` 阶段的检查点固定了这次请求要发什么，所以重新打开后重发的是同一组消息。供应商会话 ID 存在 `pi.provider` 文档里，每个对话一个 UUIDv7，跨重启、重试、重置和压缩都不变，作为 `sessionId` 传给 pi-ai，用于提示词缓存和会话亲和。

### 4.4 工具任务与效果夹心

`pi.tool` 只有两个阶段：

- `call`：读出工具调用，在当前 agent 的工具里找到实现，校验参数，运行 `beforeTool` 钩子，再校验一次。然后提交意图：检查点改为 `{ phase: "execute", arguments: final, replay: tool.replay ?? "unsafe" }`，工具槽位标为 running。接着调用 `execute(args, api, context)`；`finalResult` 运行 `afterTool` 钩子，随后 `settle` 在一次提交里追加 `pi.tool-result` 条目并把任务置为终态。
- `execute`：只有恢复时才会进入，见第 5 节。

规范把这种结构叫作效果夹心（effect sandwich）：

```text
commit intent phase
perform external effect
commit outcome or next phase
```

副作用发生在两次提交之间，不在任何事务里（规范的不变式 4："External effects do not run inside the Session mutation transaction."）。P4 的时间线里能看到这个顺序：`hook beforeTool deploy` 之后是提交 #8（`[task,document.change]`，意图加槽位），两段输出各一次提交（#9、#10），`hook afterTool` 之后是提交 #11（`[entry,task,document.change]`，结果条目、任务终态和槽位）。

工具失败都变成 `pi.tool-result` 条目，诊断信息以 `<harness>` 块附在内容后面：

| 诊断码 | 什么时候 |
|---|---|
| `tool_unavailable` | 调用的工具不在当前 agent 里 |
| `invalid_arguments` | 参数修复（`prepareArguments`）抛错或校验失败 |
| `blocked` | `beforeTool` 返回 `{ block }` 或抛错 |
| `tool_error` | `execute()` 或构建执行环境时抛错；任务以 failed 结束 |
| `interrupted` | 崩溃打断了一个不能重跑的调用 |
| `aborted` | 调用被中止 |

工具结果还能带 `usage`（计入对话用量）和 `control`：`{ terminate: true }`（本轮所有结果都要求时，不再请求模型）、`{ handoff: "…" }`（以交接说明开始新上下文）、`{ addTools: [...] }`（按名字给后续请求加工具）。

---

## 5 · 崩溃之后

![图 5 · 进程被杀之后：重新打开时每个任务怎么收尾](/images/harness/pi/ch11/durable-fig5-recovery.png)

### 5.1 重新打开

`Harness.open` 创建调度器时，调度器读出所有还活着的任务（pending、running、waiting、completing）。状态为 running 的任务被改回 pending，检查点保持不变，然后重新对齐中止级联。这时还不派发任何任务：要等 `harness.resume()`，或者调用 `submit()`、`wait()` 这类需要进展的方法。

恢复就是"从最后一个检查点重新调用那个阶段的处理器"。存储里没有事件日志，也不会重放历史事件。每个处理器自己决定上次跑到一半意味着什么。

提交时带 `requestId` 可以让重试变得安全：同一个 `requestId` 再提交一次，返回的是已有的 submission，不会重复提交。P2 的两个进程都提交了 `{ content: "Deploy it", requestId: "deploy-1" }`，拿到的都是 submission 8，第二次提交同时让调度器开始派发。

### 5.2 生成到一半

`request` 阶段重跑时，第一件事是 `convertPartial`：如果 `pi.live.generation.message` 里有上次提交的部分答案，就把它追加成一条 `stopReason: "aborted"` 的 `pi.assistant` 条目。推导模型上下文时，`stopReason` 为 `aborted`、`error`、`deferred` 的回复都会被排除，所以模型看不到这段残稿。然后用检查点里固定的 `cutoff` 重新发请求。

P3 的结果：

```text
--- killed by SIGKILL ; UI had shown: "one two three four five six seve"
[resume] pi.live.generation at reopen: {"attempt":1,"message":{… "text":"one two three four five six seve" … "stopReason":"pending"}}
[resume] request messages: [ 'user' ]
[resume] entry pi.user  "Count"
[resume] entry pi.assistant aborted ["one two three four five six seve"]
[resume] entry pi.assistant stop ["fresh answer"]
```

转录保留了用户看到过的残稿，重发的请求里只有那条用户消息。

### 5.3 工具执行到一半

工具任务崩溃时可能停在两个位置：

- **意图提交之前**（还在 `call` 阶段）：从头再跑 `call`。解析工具、校验参数、`beforeTool` 都会再执行一次，所以钩子也要能接受重跑。
- **意图提交之后**（检查点是 `execute`）：副作用可能已经发生了一部分。`execute` 处理器只在一种情况下重跑：存储的策略是 `"safe"`，并且当前注册的同名工具也声明了 `replay: "safe"`。重跑前先清掉上次发布的输出和细节，再用存储的参数调用 `execute`，不再经过 `beforeTool`。其他情况都写入一条错误结果："Tool X was interrupted and may have partially run"，内容是崩溃前已提交的输出；任务以 failed 结束，它拥有的子对话被中止。

P2 的工具在执行时先往文件里记一行"在哪个进程里跑过"，输出 `attempt in run`，然后一直挂起，直到被 `SIGKILL`：

| replay | 重新打开后的工具结果 | 工具实际执行过的进程 |
|---|---|---|
| 不声明（`unsafe`） | `isError=true`：`attempt in run` + `[error] Tool deploy was interrupted and may have partially run` | `run` |
| `"safe"` | `isError=false`：`deployed in resume` | `run`、`resume` |

两种情况下，模型都在拿到结果后回答了 `finished after restart`，submission 以 `done` 结束。

内置的 `read`、`write`、`edit`、`bash` 都没有声明 `replay`，连只读的 `read` 也是 `unsafe`。README 里声明了 `"safe"` 的例子是子 Agent 工具：它在 `api.commit` 里先按 `ownerTaskId` 查已有的子对话，有就复用，并用 `subagent:${api.taskId}` 作 `requestId` 提交任务，所以重跑会找到同一个子对话和同一条 submission。要让工具能安全重跑，常用的就是这两样：存储里的去重键，以及 `api.memo(name, candidate, context)`（先写者胜的小值，候选值的写入和读出赢家是同一次提交，跨检查点保留）。

### 5.4 中止和关闭

`conversation.abort(context)` 和崩溃不同，它会写下结果：

- 排队的输入被撤回（排队的写入保留），当前工作的每个任务都被中止，等对话空闲后才返回。
- 中止自下而上进行：先中止任务拥有的工作，等它们结束后，才调用这个任务自己的 `abort` 处理器，让每个任务撤销自己的效果。
- 工具任务的 `abort` 处理器写入 `aborted` 结果，带上已提交的输出。

P5 的场景 B 在工具运行时插入一条 steer 和一条 follow-up，然后调用 `abort()`：三条 submission 都以 `unanswered` / `aborted` 结束，工具结果是 `working\n` 加上 `[error] Tool slow was aborted`，模型只被请求过一次。

`harness.close()` 不写任何结果。没做完的工作留在存储里，下次打开时继续。

---

## 6 · 忙碌的对话

一次运行进行中，对话处于忙碌状态。这时 `submit` 的输入进入 `pi.inbox` 文档排队，`whenBusy` 决定怎么处理：

- `"followUp"`（默认）：这次运行回答后放入，开始下一次运行。
- `"steer"`：放在当前工具轮之后，加入正在进行的运行。
- `"reject"`：抛出 `ConversationBusy`。

另外，`{ type: "write", entry }` 这种 submission 只追加一条条目、不问模型，忙碌时同样进 inbox 排队。

P5 的场景 A 在工具运行时先后提交 steer 和 follow-up，三次模型请求里的用户消息依次是：

```text
'go'
'go | steer: use staging'
'go | steer: use staging | follow: then report'
```

设置 `steeringMode: "all"` 或 `followUpMode: "all"` 时，每次把排队的全部放入，而不是每轮一条。运行失败时，排队的内容留在 inbox 里，下一次提交时按先后放入。

---

## 7 · 扩展点

### 7.1 扩展与注册表

扩展是一个有名字的包，由 `defineExtension` 创建，可以带五样东西：

- `tools`：工具，`defineTool` 按 TypeBox schema 推出参数类型；
- `sections`：系统提示词片段，`section(name, render)`，每次请求前按顺序渲染；
- `hooks`：钩子，`hook(GenerationTask | ToolTask | CompactionTask, {...})`；
- `wraps`：按名字包装工具或片段，`wrapTool`、`wrapSection`；
- `tasks`：自定义的持久任务。

注册表（`createRegistry()`）属于进程，不进存储；对话只在 `pi.agent` 里存扩展和工具的名字，每次使用时再到注册表里解析。因此：

- 用同名扩展再 `install` 一次就是原地替换。已经开始的工具调用用旧代码跑完；每个任务阶段开始时解析一次钩子和 agent，下一阶段用新代码。
- 扩展被卸载后，选了它的对话只是暂时用不到它。重启后要重新安装同样的扩展，它的待办任务才会继续。

后安装的扩展里的同名工具会替换先安装的，`wrapTool` 包装最终胜出的那个。

### 7.2 七个钩子

| 任务 | 钩子 | 作用 |
|---|---|---|
| 生成 | `beforeRequest` | 替换这一次请求的消息 |
| 生成 | `afterResponse` | 观察模型回复 |
| 生成 | `onYield` | 模型给出最终回答时，追加一条用户消息让运行继续 |
| 生成 | `afterTools` | 一轮工具全部结束后调用 |
| 工具 | `beforeTool` | 拦截（`{ block }`）或改写参数 |
| 工具 | `afterTool` | 替换结果 |
| 压缩 | `beforeCompact` | 拒绝这次压缩，或自己提供摘要 |

钩子只在选了它所在扩展的对话里生效。规范说明钩子在崩溃后可能再跑一次，并且 "There is no public semantic event channel; current UI status is document state."

### 7.3 宿主提供的东西

`Harness.open` 的选项：

- `models`：pi-ai 的模型注册表；
- `registry`：扩展注册表；
- `settings`：运行策略，每次使用时读取、从不存储，可以用 getter 让它随设置文件变化（`extensions`、`stream`、`retry`、`compaction`、`progress`、`toolExecution`、`steeringMode`、`followUpMode`）；
- `env`：按对话 ID、工作目录和已提交的读数据构建执行环境，所以可以一个对话一个目录或一个容器；
- `conversationCreated`：在每次创建或分叉对话的同一次提交里运行，用来给对话写入自己的文档；
- `now`、`onReport`：时钟和错误上报。

工具只通过调用时拿到的执行环境接触文件和进程。没有配置 `env` 时，内置工具直接返回错误结果。

自己的状态用 `defineDoc` 定义成文档，在提交里和条目一起写；`history: "rewindable"` 的文档可以用 `snapshotAsOf()` 读历史值，`fork` 字段决定分叉时复制初始值、当前值还是分叉点的值。

---

## 8 · 与第三章的 Agent 对照

| 问题 | `Agent` + `agentLoop`（pi 命令在用） | pi-durable |
|---|---|---|
| 状态放在哪里 | 内存。`AgentSession` 在 `message_end` 时写会话文件 | 存储。每次改动都是一次提交，界面只看提交后的状态 |
| 循环结构 | 一次函数调用里的两层 `while`：外层后续消息，内层工具调用和插队 | 每轮一个 `pi.generation` 任务，每个调用一个 `pi.tool` 任务，阶段之间提交检查点 |
| 钩子 | `AgentLoopConfig` 的字段，例如 `transformContext`、`beforeToolCall`、`afterToolCall`、`getSteeringMessages`、`getFollowUpMessages` | 扩展里的 7 个钩子，按对话选择的扩展生效；系统提示词由片段拼成 |
| 工具失败 | 在内存里变成错误结果，随事件发出 | 变成带诊断码的 `pi.tool-result` 条目 |
| 中止 | `abort()` 触发 `AbortController`，被中止的回复结束运行 | 先提交中止标记，自下而上中止拥有的工作，每个任务的 `abort` 处理器提交终态；排队输入被撤回 |
| 崩溃恢复 | 引擎不负责。会话文件里只有写过的消息 | 重新打开后从检查点继续：生成重发请求，工具按 `replay` 重跑或记为中断 |
| 事件出口 | `subscribe` 的 10 种事件，在内存里同步派发 | 视图、操作流或 22 种派生事件，全部来自提交 |
| 插队与后续 | 两个内存队列 | `pi.inbox` 文档，`whenBusy` 选择方式 |
| 子 Agent | 不提供 | 对话可以被任务拥有，中止与空闲沿拥有关系传递；`background: true` 的任务是边界 |

---

## 9 · 对 coding-agent 意味着什么

在这个基线上，pi-durable 对 pi 命令的用户没有影响：它不在 pi 的依赖树里，实验目录也不会被安装。

实验目录里能看到 coding-agent 用这套引擎时的接法：

- **终端 Agent**：
  - 存储：一个会话一个 SQLite 文件，在 `~/.pi/agent/experimental/durable-sessions/<cwd-hash>/<session>/session.sqlite`。
  - 工具和环境：注册 `CodingTools`，`env` 用 `NodeExecutionEnv` 跟随对话的工作目录。
  - 界面：只渲染 `viewState()` 和 `taskGraph()`。
  - 复用 pi 的部分：模型运行时、认证、设置、系统提示词、快捷键、主题和交互组件。
- **会话 worker**：把 coding-agent 的 `AgentController`（prompt、steer、followUp、abort）映射到 `conversation.submit` 和 `abort`，把 `Transcript` 映射到 `viewState()`。它的 README 列出了从旧 `AgentHarness` 迁过来时暂时去掉的能力：
  - **树导航**：在 durable 里分支要分叉出新对话，还需要一条开始新上下文的摘要条目和一个"当前对话"指针。
  - **下一次运行队列**：durable 只排队插队消息和后续消息。
  - **子 Agent 服务**：目前的服务只覆盖根对话。
  - **历史分页**：视图只含最近一次重置或压缩之后的条目。

和第六章的会话树相比，最明显的差别在分支。`SessionManager` 在同一个文件里用 `parentId` 组成树，`/tree` 只移动叶子指针；pi-durable 的转录是一条只追加的线，分支靠 `fork(entryId)` 创建一个新对话，它能看到父对话到 `entryId` 为止的条目，之后各自独立。

---

## 10 · 关键数字

| 数字 | 值 |
|---|---|
| 源码 | 约 1.95 万行（`harness/` 6.7 千、`storage/` 2.9 千、`testing/` 2.7 千、`env/` 2.2 千、`session/` 2.0 千） |
| 导出入口 | 10 个 |
| 内置条目 / 文档 / 任务 | 6 / 5 / 3 |
| `StorageWrite` 种类 | 8 |
| `Storage` 方法 | 16 |
| 钩子 | 7 |
| `watchEvents` 事件 | 22 |
| 进度提交间隔 | 部分答案、工具输出都是 100 ms |
| `watch` / 事件流积压上限 | 100 帧 / 100 批 |
| 默认重试 | 最多 3 次，基础延迟 2000 ms |
| 默认压缩 | `reserveTokens` 16384、`keepRecentTokens` 20000、`backgroundTokens` 32768 |
| P4 一次工具运行的提交数 | 16（另有 1 次创建根对话） |
| `.` 入口的文件预算 | 62 |

---

## 11 · 术语表

| 术语 | 定义 | 别和它混淆 |
|---|---|---|
| Session | 打开的存储 + 一条提交队列 | 第六章的会话文件（`SessionManager`） |
| Harness | Session + 调度器 + 注册表 + 内置任务 | 已删除的 `AgentHarness` |
| 对话 | 一份只追加的转录，可以被任务拥有 | 第六章会话树里的一个分支 |
| 文档 | 和转录一起提交的带类型 JSON | 条目（不可变的转录记录） |
| 任务 | 带检查点的持久状态机 | 第三章的一"轮"（turn） |
| 运行（run） | 从一条输入到最终回答的若干轮 | 一次 `submit` |
| 意图 | 工具执行前提交的最终参数和 replay 策略 | 模型发出的原始工具调用 |
| replay | 工具的 `"safe"` / `"unsafe"`，决定意图提交后崩溃时是否重跑 | 重放事件（这里没有事件日志） |
| 毒化 | 存储结果不确定时 Session 拒绝后续操作 | `StorageRejected`（确定没写入，可继续） |

---

## 12 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 总体说明和示例 | `packages/durable/README.md`；`test/examples/14-chat.ts`（问答）、`13-recovery.ts`（恢复）、`19-json.ts`（事件流）、`22-subagent-foreground.ts`（replay-safe 子 Agent） |
| 规范 | `packages/durable/docs/spec.md`：§1 不变式、§5.2 效果夹心、§8 内置任务、§9 观察、§11 存储后端、§12 易错点；`docs/pico-v5-handoff.md`（实现计划） |
| 提交 | `src/session/session.ts` → `#runCommit`、`#enqueue`、`#publish`、`subscribeCommits`；`src/session/transaction.ts` |
| 存储 | `src/types.ts` → `Storage`、`StorageWrite`；`src/storage/memory.ts`；`src/storage/jsonl/storage.ts` → `commit`、`open`；`src/storage/sqlite/storage.ts`、`migrations.ts`、`node.ts` |
| Harness 与调度 | `src/harness/harness.ts` → `Harness.open`；`src/harness/scheduler.ts` → `open`、`resume`；`src/tasks.ts` → `defineTask` |
| 生成 | `src/harness/generation.ts` → `GenerationTask`、`streamResponse`、`classify`、`convertPartial`、`startToolRound`、`finishToolRound`；`src/harness/context.ts` → `EXCLUDED_STOP_REASONS` |
| 工具 | `src/harness/tool.ts` → `ToolTask`、`run`、`finalResult`、`settle`、`publishProgress`；`src/harness/types.ts` → `ToolRegistration`、`ToolExecutionApi` |
| 文档与视图 | `src/harness/live.ts`、`inbox.ts`、`agent.ts`、`provider.ts`、`usage.ts`；`src/harness/view.ts`；`src/harness/events.ts` → `watchEvents`、`AgentEvent` |
| 扩展 | `src/harness/define.ts`、`registry.ts`、`prompt.ts` |
| 环境与工具 | `src/env/index.ts`、`src/env/node.ts`；`src/tools/index.ts` |
| 使用者 | `packages/coding-agent/src/experimental/durable/`（`runtime.ts`、`harness-setup.ts`、`README.md`）、`vacation/`、`session-worker.ts`、`services/README.md` |
| 构建约束 | `scripts/check-entry-graphs.mjs`、`scripts/check-browser-smoke.mjs`；`packages/coding-agent/tsconfig.build.json`、`scripts/coding-agent-consumer.mjs` |
