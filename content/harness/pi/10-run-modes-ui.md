---
title: "Pi 源码分析 10 · 运行模式与界面：四种入口怎样驱动同一个 AgentSession"
date: 2026-10-05T10:30:00+08:00
description: "接着第九章往下讲：pi 命令怎样解析参数、选出交互、print、JSON、RPC 四种模式之一并建好 AgentSessionRuntime；一次性运行往 stdout 和 stderr 写什么、退出码是多少；RPC 的 33 条命令、事件流和扩展 UI 怎样在 stdin/stdout 上往返；交互模式的界面结构和输入分派；pi-tui 怎样只重写变化的行；以及用 SDK 自己驱动时要补哪些接线。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.2-18-gb2b5c42f6（2026-10-05），commit `b2b5c42f6`。v1.0.2 之后的 18 个提交里，与本章有关的只有交互模式的两处修复：终端断开时的 `EIO` 不再报告为崩溃；pi 的安装被更新或删除后，界面给出重启提示。它们不影响文中的探针结果。文中所有行为、数字和代码均以该版本为准。

前九章讲的都是同一个 `AgentSession` 内部怎么运转；本章讲外面的四种入口（交互、print、JSON、RPC）和 SDK 怎样启动、驱动和呈现它，以及终端界面是怎么画出来的。

<!--more-->

本章要点：

- **一条启动流水线，末尾才分叉**：`main()` 先分流子命令、解析参数，再由 `resolveAppMode` 选模式：`--mode rpc` / `json` 优先，`-p` 或 stdin、stdout 任一端不是终端就进 print，否则进交互。之后建会话管理器和 `AgentSessionRuntime`、读管道输入、做启动检查，最后才交给 `InteractiveMode`、`runPrintMode` 或 `runRpcMode`。每种模式拿到运行时都做同样三件事：注册重新绑定的回调、`bindExtensions`、订阅事件。
- **一次性运行只看最后一条回复**：非交互模式会把 `process.stdout.write` 转到 stderr，stdout 只留给结果。text 模式打印最后一条 assistant 消息的文字，失败时写 stderr 并以 1 退出；JSON 模式先写一行会话头，再把每个会话事件写成一行，`message_update` 去掉累积的消息快照，只留增量和用量。JSON 模式里模型出错也以 0 退出。
- **RPC 是并发的 JSONL 协议**：33 条命令（类型定义和 `switch` 分支两种方法核对，与文档一致），每行都立即异步处理，响应靠 `id` 配对。`prompt` 在预检通过时就回 `started` / `queued` / `handled`，不等运行结束。扩展的 `select`、`confirm`、`input`、`editor` 变成 `extension_ui_request`，超时或中止时 pi 自己给出默认值，不通知客户端。`abort` 不清空排队的消息，它们会跟着下一次 prompt 发出。
- **交互模式是一组固定容器加一条分派链**：聊天区、排队消息、状态、上方组件、编辑器、下方组件、页脚从上到下挂在 TUI 上，全屏时上半部分放进 `ScrollView`。Enter 先过内置命令的 `if` 链，再看是在压缩、运行还是空闲；扩展命令、`input` 事件、技能和模板在 `AgentSession.prompt` 里处理，所以 RPC 和 print 也走这一段，内置命令却只有交互模式有。
- **pi-tui 按行做差分**：组件只实现 `render(width): string[]`；渲染器把整棵树渲染成行，与上一帧逐行比较，只重写变化的行。常规模式用相对光标移动，全屏模式用绝对定位，都包在同步输出（`CSI ?2026h … l`）里。探针里流式输出每来一段文字，终端只收到一行约 130 字节的写入。pi-tui 只依赖两个第三方包，可以单独使用。

---

## 0 · 阅读说明

- 本章只讲已发布的 `pi` 命令走的路径：pi-coding-agent 的 `main()`、三个运行模式、`AgentSessionRuntime`，以及 pi-tui。
- 引用前几章的结论，不再重复：
  - 第一章 6.3 节：四种运行模式的启动方式。
  - 第三章 2.2 节：插队消息（steer）和后续消息（followUp）两个队列分别在什么时候被取出。
  - 第四章 5.3 节：请求失败后的自动重试。
  - 第六章 6 节：命令行里选会话的参数（`--continue`、`--resume`、`--session`、`--fork`、`--no-session`）。
  - 第九章 2.2 节：`bindExtensions` 接上什么、为什么 `session_start` 在绑定时才发；2.4 节：`/new`、`/resume`、`/fork` 怎样替换会话；5 节：`ctx.ui` 在各模式下的行为；6.1 节：扩展出错在各模式下怎样呈现。
- 行为都用探针实际跑过。探针 P1–P6 运行 npm 上 1.0.2 版的 `pi` 命令和包：
  - `HOME` 指向临时目录，`models.json` 配一个自定义供应商 `mock`（`api: "anthropic-messages"`，`baseUrl` 指向本地 mock 服务，模型 `mock-1`），设置 `PI_OFFLINE=1` 关掉联网刷新。
  - mock 服务按用户消息里的关键字回复：含 `TOOL` 时先调用工具 `deploy`，含 `SLOW` 时每 150 ms 流出一段文字、共 40 段，含 `ERR` 时返回 HTTP 500，其余回复 `Hello from mock.`。
  - 用 `-e` 加载一个探针扩展：工具 `deploy` 在执行时调用 `ctx.ui.confirm("Deploy?", …, { timeout: 30000 })`，再用 `ctx.ui.notify` 报告结果（第 7 节有源码）。
  - 交互模式（P4）用 node-pty 开伪终端，用 `@xterm/headless` 当终端：它负责回答 pi 发出的终端查询，也能读回屏幕内容。探针记录每一帧屏幕和每一次原始写入。
- 术语：
  - **运行时**：`AgentSessionRuntime`，持有当前的 `AgentSession` 和它的服务，负责替换会话。
  - **重新绑定**：会话被替换后，模式对新会话再做一次 `bindExtensions` 和事件订阅。
  - **CSI**：终端控制序列的前缀 `ESC [`。`CSI 2K` 清除整行，`CSI 5A` 光标上移 5 行，`CSI 17;1H` 光标移到第 17 行第 1 列。

---

## 1 · 从命令行到会话

![图 1 · 从命令行到会话：四种入口共用的启动流程与分叉点](/images/harness/pi/ch10/fig1-modes-startup.png)

### 1.1 main() 的步骤

`pi` 命令的入口 `cli.ts` 先调用 `setupCli`（设置进程名、`PI_CODING_AGENT=true` 和 `AI_AGENT=pi` 两个环境标记、HTTP 代理），再调用 `main(argv)`。`main` 按下面的顺序走：

| 阶段 | 做什么 | 模式 |
|---|---|---|
| 子命令 | `pi auth`、包管理（install / remove / update / list）、`pi config`、`pi mcp` 在解析参数之前就分流出去，处理完退出 | — |
| 解析参数 | `parseArgs` 认识 40 个参数；未知的 `--x` 先存起来，等扩展注册的参数来认领；`--version`、`--export` 立即退出 | 全部 |
| 选模式 | `resolveAppMode`（1.2 节） | 全部 |
| 接管 stdout | `takeOverStdout`：把 `process.stdout.write` 改写到 stderr，只有 `writeRawStdout` 能写真正的 stdout | 交互以外 |
| 会话 | 迁移旧数据、读设置、`createSessionManager` 按会话参数打开或新建会话（第六章 6 节） | 全部 |
| 运行时 | `createAgentSessionRuntime(createRuntime, …)`：工厂函数里建设置、资源加载器、扩展、模型运行时和 `AgentSession`。`--help`（要列出扩展注册的参数）和 `--list-models` 在这一步之后才退出 | 全部 |
| 初始消息 | `readPipedStdin` 读管道输入，`prepareInitialMessage` 处理 `@文件`（1.3 节） | RPC 以外 |
| 启动检查 | 打印诊断；有 error 级诊断或没有可用模型就退出（1.4 节） | 见 1.4 节 |
| 分叉 | `new InteractiveMode(runtime, …).run()`、`runPrintMode(runtime, …)` 或 `runRpcMode(runtime)` | — |

工厂函数 `createRuntime` 存在运行时里，`/new`、`/resume`、`/fork` 替换会话时会再调用它（第九章 2.4 节）。

### 1.2 选模式

```typescript
function resolveAppMode(parsed: Args, stdinIsTTY: boolean, stdoutIsTTY: boolean): AppMode {
	if (parsed.mode === "rpc") {
		return "rpc";
	}
	if (parsed.mode === "json") {
		return "json";
	}
	if (parsed.print || !stdinIsTTY || !stdoutIsTTY) {
		return "print";
	}
	return "interactive";
}
```

- `--mode` 只接受 `text`、`json`、`rpc`。`--mode text` 不带 `-p` 时在终端里仍是交互模式；`text` 只决定 print 模式的输出格式。
- 没有环境变量参与选择。stdin 或 stdout 被重定向（管道、文件）就自动进 print 模式，所以 `echo 问题 | pi` 和 `pi > out.txt` 都是一次性运行。
- RPC 模式里 stdin 留给命令：启动时不读管道输入，位置参数里的消息被忽略，带 `@文件` 参数直接以 1 退出。

### 1.3 初始消息：管道、@文件和位置参数

print 和交互模式把三段内容拼成第一条消息：管道输入（去掉首尾空白）、`@文件` 展开的内容、第一个位置参数。三段直接相连，中间不加分隔符，所以 `echo foo | pi -p bar` 发出去的是 `foobar`。`@文件` 展开成 `<file name="绝对路径">…</file>` 块，块尾有换行，不会和后面粘在一起；图片文件按内容识别后作为图片附上，文件不存在时打印错误并以 1 退出，空文件直接跳过。

其余位置参数各成一条消息。print 模式逐条 `await session.prompt()`；P1 里 `pi -p first second`，mock 依次收到 `first` 和 `second` 两次请求，stdout 只有最后一次的回复。

> 📌 **管道输入要等到 EOF**。`readPipedStdin` 在 stdin 不是终端时一直读到流结束。父进程用管道启动 pi 却不关闭它的 stdin，pi 会停在这一步。P1 第一次运行就卡在这里：探针脚本自己的 stdin 是一个不会关闭的管道，`pi -p` 继承了它。给子进程 `< /dev/null` 或关掉它的 stdin 即可。

### 1.4 启动检查和退出码

| 情形 | 交互 | print / JSON / RPC |
|---|---|---|
| 参数错误（未知选项、`--mode` 取值不对、`--name` 为空等） | stderr 一行 `Error: …`，退出码 1 | 同左 |
| 启动诊断（设置、信任、扩展包的警告和错误） | 没有错误时显示在界面里 | 打印到 stderr |
| 有 error 级诊断（扩展加载失败、未知的扩展参数等） | 打印后退出码 1；扩展加载失败时多一行 `pi -ne` 的提示 | 同左 |
| 没有任何可用模型 | 照常启动，在界面里处理 | `No models available.` 加登录说明，退出码 1 |
| 有模型但没有密钥 | 到 prompt 时才报错 | prompt 时抛错：print / JSON 写 stderr、退出码 1；RPC 返回失败的响应 |
| `--model` 指定的模型不存在 | 启动诊断，退出码 1 | 同左 |

P1 里 `HOME` 指向一个空目录时，内置的模型目录仍然有模型可选，所以走的是"没有密钥"这一行：stderr 是 `No API key found for the selected model.` 加两份文档路径，退出码 1。`--model nope/none` 得到 `Error: Model "nope/none" not found. Use --list-models to see available models.`，退出码 1。

### 1.5 四种模式共同做的三件事

拿到运行时之后，三个模式函数的开头几乎一样：

1. `runtime.setRebindSession(rebind)`：注册一个回调。会话被 `/new`、`/resume`、`/fork` 替换后，运行时调用它。
2. `rebind()` 里调用 `session.bindExtensions({ … })`，发出 `session_start`（第九章 2.2 节）。
3. 同在 `rebind()` 里，退订旧会话、`session.subscribe()` 订阅新会话的事件，交给各自的呈现方式。

不同之处在于绑定时传什么：

| 绑定项 | 交互 | print / JSON | RPC |
|---|---|---|---|
| `mode` | `tui` | `print` / `json` | `rpc` |
| `uiContext`（`ctx.ui` 的实现） | 终端界面（4.6 节） | 不传，扩展拿到空操作的实现 | 转成协议消息（3.5 节） |
| `commandContextActions`（命令里的 `newSession`、`fork`、`reload` 等） | 有，替换后重画界面 | 有 | 有 |
| `abortHandler`（`ctx.abort()` 时额外做的事） | 把排队消息退回编辑器 | 不传 | 不传 |
| `shutdownHandler`（`ctx.shutdown()`） | 空闲时退出，否则等运行结束 | 不传 | 记下请求，命令处理完或 `agent_settled` 时退出 |
| `onError`（扩展出错，第九章 6.1 节） | 对话区显示错误和调用栈 | stderr 一行 | `extension_error` 记录 |
| 订阅事件后做什么 | 更新界面组件 | JSON 模式写一行；text 模式不订阅输出 | 写一行 |

---

## 2 · print 与 JSON 模式

### 2.1 一次性运行的流程

`runPrintMode(runtime, { mode, messages, initialMessage, initialImages })` 的全部工作：

1. 注册 `SIGTERM`（Windows 以外还有 `SIGHUP`）的处理函数：结束跟踪的子进程，`runtime.dispose()`，再以 143（`SIGHUP` 是 129）退出。
2. JSON 模式先写一行会话头（`sessionManager.getHeader()`）。
3. `rebind()`：绑定扩展，订阅事件。
4. `await session.prompt(initialMessage)`，然后逐条 `await session.prompt(其余消息)`。
5. text 模式读会话里的最后一条消息，按 2.2 节的规则输出。
6. 无论成败，`finally` 里先退订事件，再 `runtime.dispose()`（发 `session_shutdown`，`reason` 为 `quit`），最后等 stdout 写完。

`main` 拿到返回值后只设置 `process.exitCode`，不调用 `process.exit()`，进程在事件循环空了之后自然结束。

print 模式默认也写会话文件，探针的临时 `HOME` 下每次运行都多一个 `.jsonl`；不想留记录加 `--no-session`。

### 2.2 stdout、stderr 和退出码

text 模式只输出最后一条消息：它是 assistant 消息、`stopReason` 不是 `error` 或 `aborted` 时，把其中每个文字块写一行到 stdout，思考和工具调用块不输出；是 `error` 或 `aborted` 时，把 `errorMessage` 写到 stderr，退出码 1。最后一条不是 assistant 消息时什么也不输出，退出码 0，例如新会话里这次输入被扩展命令处理掉了。用 `--continue` 接着旧会话时，最后一条可能是上一次的回复，它会被打印出来。

P1 的几种情况：

```text
运行                                  stdout                      stderr                    退出码
pi -p "say hi"                        Hello from mock.            —                         0
echo 'piped question' | pi            Hello from mock.            —                         0
pi -p @note.txt summarize             Hello from mock.            —                         0
pi -p "ERR please"                    —                           500 {"type":"error",…}    1
pi -e ext-deploy.ts -p "TOOL go"      Deployed (tool said ok).    [ext] session_start …     0
pi -p /model                          Hello from mock.            —                         0
```

`TOOL go` 那一行的 stderr 是探针扩展自己打的两行日志：`[ext] session_start reason=startup mode=print hasUI=false` 和 `[ext] session_shutdown reason=quit`。

最后两行值得注意：

- 没有界面时 `ctx.ui.confirm` 直接返回 `false`，工具的结果是 `user declined`，模型照常收到工具结果、给出回复（第九章 5 节）。
- `/model` 是交互模式的内置命令，print 模式里不存在。它被当作普通文字发给了模型（mock 的日志里 `last="/model"`）。扩展命令、技能和模板在 print 模式里照常可用（4.3 节）。

扩展、工具在运行中调用 `console.log` 也不会弄脏 stdout：`takeOverStdout` 已经把 `process.stdout.write` 转到 stderr，只有模式自己通过 `writeRawStdout` 写的内容进 stdout。这一点对 text 模式同样成立。

### 2.3 JSON 事件流

JSON 模式先写会话头，再把 `session.subscribe` 收到的每个事件经 `toJsonEvent` 转换后写成一行。P2 一次普通问答的完整事件流（每行只保留类型和关键字段）：

```text
session version=3 id=… cwd=<ws>
agent_start
turn_start
message_start role=system
message_end role=system
message_start role=user content=["say hi"]
message_end role=user content=["say hi"]
message_start role=assistant stopReason=pending content=[""]
message_update ame=text_start keys=[type,usage,assistantMessageEvent]
message_update ame=text_delta delta="Hello" keys=[type,usage,assistantMessageEvent]
message_update ame=text_delta delta=" from" keys=[type,usage,assistantMessageEvent]
message_update ame=text_delta delta=" mock." keys=[type,usage,assistantMessageEvent]
message_update ame=text_end content="Hello from mock." keys=[type,usage,assistantMessageEvent]
message_end role=assistant stopReason=stop content=["Hello from mock."]
turn_end role=assistant stopReason=stop content=["Hello from mock."]
agent_end messages=3 willRetry=false
agent_settled
```

开头的 system 消息是第八章讲的提示词补丁，它和 user 消息一样作为消息事件发出。

**"message_update 只带增量"的具体含义**：会话内部的 `message_update` 事件带着 `message`（到目前为止累积的整条消息）和 `assistantMessageEvent`（pi-ai 的流式事件，里面还有一份累积快照 `partial`）。`toJsonEvent` 只对这一种事件做转换：

- 去掉 `message`，只保留它的 `usage`（当前的用量）；
- 去掉 `assistantMessageEvent.partial`，留下 `type`、`contentIndex`，以及 `delta`（新增的文字或参数片段）或块结束时的完整 `content` / `toolCall`；
- `toolcall_start` 去掉 `partial` 后补上 `id` 和 `toolName`，客户端不必等参数流完就知道调用的是哪个工具。

所以每行的大小只和这一段增量有关。要拿到完整的消息，看 `message_end`，或者自己把 `delta` 按 `contentIndex` 拼起来。其他事件原样输出。

JSON 模式输出的事件就是 `AgentSessionEvent` 的全部 23 种：

| 组 | 事件（字段） |
|---|---|
| 运行（5） | `agent_start`；`agent_end`（`messages`、`willRetry`：是否还会自动重试）；`agent_settled`（不会再自动继续）；`turn_start`；`turn_end`（`message`、`toolResults`） |
| 消息（3） | `message_start`、`message_update`、`message_end` |
| 工具（3） | `tool_execution_start` / `_update` / `_end`；工具里经 `ctx.executeTool()` 嵌套调用时带 `parentToolCallId` |
| 队列与状态（4） | `queue_update`（`steering`、`followUp` 两个队列的文字）；`entry_appended`（新写入的会话条目）；`session_info_changed`（会话名）；`thinking_level_changed` |
| 压缩（2） | `compaction_start`（`reason`）、`compaction_end`（`result`、`aborted`、`willRetry`、`errorMessage`） |
| 重试（5） | `auto_retry_start`（`attempt`、`maxAttempts`、`delayMs`、`errorMessage`）、`auto_retry_end`；`summarization_retry_scheduled`、`summarization_retry_attempt_start`、`summarization_retry_finished`（摘要请求的重试，第七章 4.3 节） |
| shell（1） | `bash_execution_update`（`delta`：用户 `!` 命令的输出片段） |

其中 10 种来自 pi-agent-core 的 `AgentEvent`（`agent_end` 被换成多了 `willRetry` 的版本），13 种是 `AgentSession` 自己加的。

### 2.4 出错、重试和中止

P2 让 mock 返回 HTTP 500，JSON 模式下的事件（省略 `message_update`）：

```text
message_start role=assistant stopReason=error content=[] err=500 {"type":"error",…}
message_end   role=assistant stopReason=error …
turn_end … 
agent_end messages=3 willRetry=true
auto_retry_start {"attempt":1,"maxAttempts":3,"delayMs":2000,…}
entry_appended entry=context_edit
agent_start
…（第 2、3 次：delayMs 4000、8000）
agent_end messages=1 willRetry=false
auto_retry_end {"success":false,"attempt":3,"finalError":"500 …"}
agent_settled
```

- 失败的回复也是一条完整的 assistant 消息，`stopReason` 为 `error`，没有 `message_update`。
- 每次重试前写一条 `context_edit` 条目，把失败的尝试从上下文里编辑掉（第六章 5.3 节）。
- **JSON 模式的退出码是 0**，只有 text 模式检查最后一条消息的 `stopReason`。脚本用 JSON 模式时，要自己看 `message_end` 或 `auto_retry_end` 判断成败（官方的 `docs/cli-integration.md` 也这样说明）。

退出码汇总（P1、P2）：

| 情形 | text 模式 | JSON 模式 |
|---|---|---|
| 正常结束 | 0 | 0 |
| 最后一条回复 `error` / `aborted` | 1 | 0 |
| `prompt` 抛错（没有密钥、压缩进行中等） | 1 | 1（会话头已经写出） |
| `SIGTERM` / `SIGHUP` | 143 / 129，先 `dispose`，扩展收到 `session_shutdown` | 同左；`dispose` 之前已退订，关闭阶段的事件不再输出 |
| `SIGINT`（Ctrl+C） | 没有处理函数，Node 默认终止，shell 看到 130；扩展收不到 `session_shutdown` | 同左 |

### 2.5 排队消息和扩展的动作在一次性运行里会怎样

`session.prompt()` 不只等这一次请求：它内部的循环在还有排队消息时继续调用 `agent.continue()`，结束时发 `agent_settled`，再执行 `agent_settled` 处理函数里调用的 `prompt()` 或 `sendMessage({ triggerTurn: true })`（这些调用会被推迟到 settle 之后执行）。所以一次性运行会把下面这些都跑完再输出：

- 运行中由扩展 `sendMessage` 插入的消息、`deliverAs: "followUp"` 的后续消息（第九章 3.3 节）；
- 自动重试、溢出后的压缩重试；
- `agent_settled` 处理函数里发起的新一轮。

之后才 `dispose`。扩展在 settle 之后用定时器异步发起的工作会被截断：print 模式在最后一条 prompt 返回后不再调用 `waitForIdle()`。

---

## 3 · RPC 模式

![图 2 · RPC 的一次往返：prompt、扩展 confirm 和 abort](/images/harness/pi/ch10/fig2-rpc-sequence.png)

### 3.1 记录和帧格式

RPC 模式下 stdin 和 stdout 上都是一行一个 JSON 对象：

| 方向 | 记录 | 作用 |
|---|---|---|
| stdin | 命令（33 种，3.2 节） | 发起 prompt、查询状态、改设置、管理会话 |
| stdin | `extension_ui_response` | 回答扩展的对话框，不产生响应 |
| stdout | `response` | 一条命令的结果 |
| stdout | 会话事件 | 与 JSON 模式相同的 23 种（2.3 节） |
| stdout | `extension_ui_request` | 扩展要显示对话框或通知（3.5 节） |
| stdout | `extension_error` | 扩展处理函数出错（`extensionPath`、`event`、`error`，不带调用栈） |

和 JSON 模式相比，RPC 不输出会话头；会话 ID 和文件路径用 `get_state` 查询。

帧格式由 `jsonl.ts` 实现：输出是 `JSON.stringify(值) + "\n"`；读入只按 LF 切分，去掉行尾的一个 `\r`，用 `StringDecoder` 解码，跨块的多字节字符不会被切坏。它故意不用 Node 的 `readline`，因为 `readline` 还会在 U+2028、U+2029 处断行，而这两个字符可以合法地出现在 JSON 字符串里。客户端读 stdout 也应该这样做（第 7 节）。

无法解析的一行（包括空行）得到一条没有 `id` 的响应：

```json
{"type":"response","command":"parse","success":false,"error":"Failed to parse command: Unexpected token 'h', \"this is not json\" is not valid JSON"}
```

未知的命令得到 `Unknown command: warp_drive`（P3）。

### 3.2 33 条命令

命令的联合类型 `RpcCommand` 有 33 个成员；`runRpcMode` 里处理命令的 `switch` 也恰好有 33 个 `case`（另有 `default` 处理未知命令），两份名单一致；`docs/rpc-commands.md` 的 33 个小节也一一对应。每条命令都可以带一个可选的字符串 `id`。

| 组 | 数 | 命令 |
|---|---|---|
| 发起与中止 | 6 | `prompt`（发消息，可带 `images`、`streamingBehavior`）、`steer`（插队）、`follow_up`（排后续）、`abort`（中止并等到空闲）、`clear_queue`（清空两个队列，返回被清掉的文字）、`new_session`（新会话） |
| 状态 | 2 | `get_state`（模型、思考级别、是否在运行、两种队列模式、会话文件和 ID、消息数、排队数）、`get_messages`（当前上下文里的消息） |
| 模型 | 3 | `set_model`、`cycle_model`、`get_available_models` |
| 思考级别 | 3 | `set_thinking_level`、`cycle_thinking_level`、`get_available_thinking_levels` |
| 队列模式 | 2 | `set_steering_mode`、`set_follow_up_mode`（`all` 或 `one-at-a-time`，默认后者） |
| 压缩 | 2 | `compact`（手动压缩，等压缩完才回）、`set_auto_compaction` |
| 重试 | 2 | `set_auto_retry`、`abort_retry` |
| shell | 2 | `bash`（执行命令，等它结束才回；输出片段以 `bash_execution_update` 发出，带这条命令的 `id`）、`abort_bash` |
| 会话 | 10 | `get_session_stats`、`export_html`、`switch_session`、`fork`、`clone`、`get_fork_messages`（可以 fork 的用户消息）、`get_entries`（会话条目，可从某条之后开始）、`get_tree`、`get_last_assistant_text`、`set_session_name` |
| 命令发现 | 1 | `get_commands`（扩展命令、模板和技能；不含交互模式的内置命令） |

交互模式的 `/tree` 导航、`/reload`、`/login` 这类操作在 RPC 里没有对应的命令（第六章 6 节已经提到没有分支导航命令）。

### 3.3 请求与响应

响应的格式：

```json
{"id":"req-1","type":"response","command":"prompt","success":true,"data":{"disposition":"started"}}
{"id":"6","type":"response","command":"prompt","success":false,"error":"Agent is already processing. Specify streamingBehavior ('steer' or 'followUp') to queue the message."}
```

- `id` 原样回填；命令没有 `id`，响应里也没有。没有数据时省略 `data`；`cycle_model`、`cycle_thinking_level` 没东西可切换时返回 `"data":null`。
- **命令并发处理**。每读到一行就 `void handleInputLine(line)`，不等上一条处理完。一条要等很久的命令（`compact`、`bash`、`abort`）不会挡住后面的 `get_state`，所以客户端必须按 `id` 配对，不能按顺序。
- **`prompt` 不等运行结束**。它把 `preflightResult` 回调交给 `session.prompt()`，在三个时刻之一回响应：
  - `handled`：被扩展命令或 `input` 事件处理掉了，不会开始运行；
  - `queued`：正在运行，按 `streamingBehavior` 排进了队列；
  - `started`：通过了模型和密钥检查、`before_agent_start` 也跑完了，马上开始运行。
  预检之前的失败（流式中没给 `streamingBehavior`、压缩进行中、没有模型或密钥）变成失败的响应；`started` 之后的失败只出现在事件流里。所以每个 `id` 恰好一条响应，运行有没有结束要看 `agent_settled`。
- **扩展命令的 `handled` 要等命令执行完才回**。`session.prompt()` 先 `await` 命令的处理函数，再调用 `preflightResult("handled")`。P3b 的 `/ask timeout` 里，命令弹出一个 1 秒超时的 confirm，响应在 1213 ms 时才到，和命令最后发的 `notify` 同时。

P3 一次会话里的几条（`<<` 是 pi 写出的，`>>` 是客户端写入的）：

```text
>> {"id":"3","type":"prompt","message":"say hi"}
<< {"id":"3","type":"response","command":"prompt","success":true,"data":{"disposition":"started"}}
<< {"type":"agent_start"}
…
<< {"type":"agent_settled"}
>> {"id":"9","type":"get_entries"}
<< {"id":"9","type":"response","command":"get_entries","success":true,"data":{"entries":[…],"leafId":"…"}}
```

### 3.4 扩展 UI 的往返

RPC 模式绑定扩展时传入自己的 `uiContext`，`ctx.ui` 的方法分三类（完整的空操作清单见第九章 5 节）：

| 类别 | 方法 | 线上的记录 |
|---|---|---|
| 要等回复的对话框 | `select`、`confirm`、`input`、`editor` | `extension_ui_request`，`method` 为方法名，带一个随机 UUID 作 `id`，以及 `title`、`options` / `message` / `placeholder` / `prefill` 和 `timeout` |
| 单向通知 | `notify`、`setStatus`、`setWidget`（只转发字符串数组）、`setTitle`、`setEditorText` / `pasteToEditor`（都发 `set_editor_text`） | `extension_ui_request`，不需要回复 |
| 空操作 | `custom`、`setFooter`、`setHeader`、`setEditorComponent` 等 | 不发任何记录 |

客户端用三种形状之一回复，`id` 取自请求：

```json
{"type":"extension_ui_response","id":"<uuid>","value":"b"}
{"type":"extension_ui_response","id":"<uuid>","confirmed":true}
{"type":"extension_ui_response","id":"<uuid>","cancelled":true}
```

回复不产生 `response`；`id` 对不上的回复被静默丢弃。对话框的等待由 `createDialogPromise` 管理，P3b 验证了三种结束方式：

```text
/ask timeout   confirm(…, { timeout: 1000 }) 无人回答   → 1 秒后 confirm 返回 false
/ask select    客户端回 { cancelled: true }             → select 返回 undefined
/ask abort     input(…, { signal })，300 ms 后 signal 中止 → input 返回 undefined
```

超时和中止都由 pi 这边给出默认值（`select` / `input` 为 `undefined`，`confirm` 为 `false`），**不会通知客户端**：客户端显示的对话框不会自动关闭，之后再回复也会被丢弃。`editor` 的签名里没有选项参数，既没有超时也不能中止，没有回复就一直等。

`setWidget` 不传 `placement` 时，记录里就没有 `widgetPlacement` 字段（P3b：`{"method":"setWidget","widgetKey":"probe","widgetLines":["line 1","line 2"]}`），`rpc-extension-ui.md` 说明缺省时按 `aboveEditor` 处理，这个默认值要由客户端实现。

### 3.5 运行中的输入和中止

P3、P3b 在 mock 慢速流出文字时依次发送：

```text
>> {"id":"q1","type":"prompt","message":"SLOW one"}            → started
>> {"id":"q2","type":"prompt","message":"after that","streamingBehavior":"followUp"}
<< {"type":"queue_update","steering":[],"followUp":["after that"]}
<< {"id":"q2",…,"data":{"disposition":"queued"}}
>> {"id":"q3","type":"steer","message":"be brief"}
<< {"type":"queue_update","steering":["be brief"],"followUp":["after that"]}
<< {"id":"q3",…,"data":{"disposition":"queued"}}
>> {"id":"q4","type":"abort"}
<< {"type":"agent_end",…}
<< {"type":"agent_settled"}
<< {"id":"q4","type":"response","command":"abort","success":true}
>> {"id":"q5","type":"get_state"}                              → pendingMessageCount: 2
>> {"id":"q6","type":"prompt","message":"say hi"}
<< message_start user "say hi" → queue_update → message_start user "be brief" → … assistant …
<< queue_update(空) → message_start user "after that" → … assistant … → agent_settled
```

- 流式中的 `prompt` 必须带 `streamingBehavior`，否则失败；带了就和 `steer` / `follow_up` 一样排队，回 `queued`。
- `abort` 等到会话空闲才回响应，所以它的响应排在 `agent_settled` 之后。
- **`abort` 不清空队列**。中止后 `pendingMessageCount` 仍是 2；下一次 `prompt` 时，插队消息紧跟在新的 user 消息后面进入第一轮，后续消息在这一轮结束后进入第二轮。交互模式按 Escape 中止时会先把排队消息退回编辑器（4.2 节）；RPC 客户端想要同样的效果，先发 `clear_queue` 取回文字，再发 `abort`。

### 3.6 会话替换和退出

`new_session`、`switch_session`、`fork`、`clone` 由运行时替换整个 `AgentSession`。替换完成时运行时会调用 `setRebindSession` 注册的回调，RPC 模式的这几个 `case` 在返回前又自己调用了一次 `rebindSession()`，于是新会话被绑定两次。P3 里 `new_session` 之后探针扩展的日志：

```text
[ext] session_shutdown reason=new
[ext] session_start reason=new mode=rpc hasUI=true
[ext] session_start reason=new mode=rpc hasUI=true
```

交互和 print 模式只靠回调，扩展只收到一次。在 RPC 模式下写扩展时，`session_start` 处理函数要能重复执行。

退出：

| 情形 | 结果 |
|---|---|
| 关闭 pi 的 stdin | `shutdown()`：`runtime.dispose()`（扩展收到 `session_shutdown`，`reason` 为 `quit`），写完 stdout，退出码 0（P3） |
| `SIGTERM` / `SIGHUP` | 结束跟踪的子进程，dispose，退出码 143 / 129；`SIGTERM` 时不等 stdout 写完 |
| 扩展调用 `ctx.shutdown()` | 记下请求，当前命令处理完或下一次 `agent_settled` 时执行 `shutdown()` |
| `SIGINT` | 没有处理函数 |

不在离线模式时，RPC 模式启动后还会在后台刷新一次模型目录，最多 15 秒，不阻塞命令。

### 3.7 与官方文档对照

`docs/rpc.md`、`rpc-commands.md`、`rpc-extension-ui.md` 和 `json.md` 的命令名、字段、帧格式、`disposition` 的含义、关闭 stdin 即退出等说法都与代码一致。有出入或没写到的：

- `rpc-extension-ui.md` 说带 `timeout` 的对话框会自动结束，这对 `select`、`confirm`、`input` 成立；`editor` 没有超时，也不能中止。超时和中止都不通知客户端，文档没有提。
- `rpc-commands.md` 的 `get_state` 示例里 `steeringMode` 是 `"all"`，实际默认值是 `"one-at-a-time"`（P3 的 `get_state`），同一份文档后文也说默认是后者。
- `json.md` 的示例把 user 消息的 `content` 写成字符串，实际是内容块数组（P2：`[{"type":"text","text":"say hi"}]`）。工具执行事件的 `parentToolCallId` 写在 `extensions.md` 里，`json.md` 没有列出。
- `fork` 响应的类型把 `text` 声明为 `string`，被扩展取消时实际不带这个字段。
- 文档推荐 TypeScript 用户使用导出的 `RpcClient`。它的 33 个方法和 `promptAndWait()` 等辅助方法都在，但没有回答 `extension_ui_request` 的接口，这类记录只会交给事件监听函数；每个请求还有 30 秒的固定超时，`compact`、`bash` 这类长命令也一样。
- `new_session` 等命令让扩展收到两次 `session_start`（3.6 节），文档没有提。

---

## 4 · 交互模式

![图 3 · 交互模式：界面结构与输入分派](/images/harness/pi/ch10/fig3-interactive.png)

### 4.1 结构：一组固定的容器

`InteractiveMode` 的构造函数先按 `--tui-mode` 或设置 `tuiMode` 选渲染器（默认全屏；只有明确设成 `regular` 才用常规模式），再建好一组容器；`init()` 把它们从上到下挂到 TUI 上：


- `documentContainer`：包含 `headerContainer`（版本和快捷键提示）、`loadedResourcesContainer`（加载的扩展、技能等）、`chatContainer`（用户消息、回复、工具调用）。
- `pendingMessagesContainer`：排队中的 `Steering: …`、`Follow-up: …`。
- `statusContainer`：状态消息。Working 动画默认嵌在编辑器的上边框里，自定义编辑器不支持时才放到这里。
- `widgetContainerAbove`：扩展 `setWidget` 的组件（默认位置）。
- `editorContainer`：编辑器；扩展的对话框也显示在这里。
- `widgetContainerBelow`：`setWidget` 指定 `placement: "belowEditor"` 的组件。
- `footerContainer`：工作目录、上下文用量、模型名，以及扩展 `setStatus` 的状态文字。

界面没有单独的状态栏：扩展的状态显示在页脚里。

两种渲染器的区别：

- **全屏**（`TuiAltScreen`）：进入终端的备用屏幕（`CSI ?1049h`），用 `VStack` 布局：`documentContainer` 放进一个跟随底部的 `ScrollView`，其余容器固定在底部。滚动、搜索、鼠标选择都由 pi-tui 自己实现。退出时按设置 `fullscreenExitOutput` 处理：默认的 `transcript` 先切回常规模式，把整个对话输出到主屏，退出后还能在终端里翻看；另一个取值是 `resume-hint`。
- **常规**（`TuiMainScreen`）：同一组容器依次输出在主屏上，历史内容留在终端自己的回滚区里。
- 运行中可以在 `/settings` 里切换：停掉旧渲染器，把同一组组件挂到新渲染器上。组件拿到的 `ui` 是一个代理对象，换渲染器不影响它们。

P4 抓到的启动帧（80×24 的终端，全屏模式；常规模式内容相同，只是编辑器和页脚紧跟在内容下方，下面留空）：

```text
 2| ▀▀█  v1.0.2
 3| █▀ █ escape interrupt · ctrl+c/ctrl+d clear/exit · / commands · ! bash ·
 4| ctrl+o more
 …
10|[Extensions]
11|  ext-deploy.ts
 …
20|────────────────────────────────────────────────────────────────────────────────
21|
22|────────────────────────────────────────────────────────────────────────────────
23|/private/tmp/ch10.Dbpr/ws
24|0.0%/100k (auto)                                                          mock-1
```

第 20–22 行是编辑器，23–24 行是页脚。运行时 Working 动画就画在第 20 行编辑器的上边框上（4.2 节的帧）。

### 4.2 输入分派

Enter 由 `setupEditorSubmitHandler` 处理，按顺序判断：

1. 文字是某个内置命令（4.3 节；`/model`、`/name`、`/compact` 等 8 个还可以带参数）：执行它，结束。
2. 以 `!` 开头：执行 shell 命令；`!!` 的输出不进上下文。已经有命令在跑时拒绝。
3. 正在压缩：扩展命令立即执行，其余文字排成插队消息，压缩完再发。
4. 正在运行：`session.prompt(text, { streamingBehavior: "steer" })`，排成插队消息。
5. 空闲：把文字交给 `run()` 里等待输入的循环，由循环调用 `session.prompt(text)`。

其他按键（默认绑定，可在快捷键设置里改）：

| 按键 | 运行中 | 空闲 |
|---|---|---|
| Alt+Enter | 排成后续消息（`streamingBehavior: "followUp"`） | 和 Enter 一样 |
| Escape | 把排队的消息退回编辑器，再 `abort()` | 正在执行 `!` 命令时中止它；编辑器为空时连按两次打开 `/tree` 或 `/fork`（设置 `doubleEscapeAction`） |
| Alt+Up | 把排队的消息全部取回编辑器 | — |
| Ctrl+C | 清空编辑器，500 ms 内连按两次退出；**不中止运行** | 同左 |
| Ctrl+D | 编辑器为空时退出 | 同左 |

P4 在 mock 慢速输出时依次输入 `be brief` + Enter、`later please` + Alt+Enter，再按 Escape：

```text
帧 C（Enter 之后）              帧 D（Alt+Enter 之后）            帧 E（Escape 之后）
15| w0 w1 w2 w3 w4             14| w0 w1 w2 w3 w4               14| w0 w1 w2 w3 w4
16|                            15|                              15|
17| Steering: be brief         16| Steering: be brief           16| Operation aborted
18| ↳ Option+Up to edit all…   17| Follow-up: later please      17|
19|                            18| ↳ Option+Up to edit all…     18|──────────────────
20|── ⠋ Working ─────────      20|── ⠋ Working ─────────        19|be brief
                                                                20|
                                                                21|later please
                                                                22|──────────────────
```

排队消息显示在聊天区和编辑器之间；Escape 之后两条消息回到了编辑器（中间隔一个空行），回复停在 `w4`，下面是 `Operation aborted`。这和 RPC 的 `abort` 不同：RPC 不动队列（3.5 节）。

### 4.3 斜杠命令

斜杠命令分两段处理：

- **内置命令**在 `InteractiveMode` 的 Enter 处理函数里，用一串 `if (text === "/settings")`（带参数的用 `startsWith("/model ")`）判断。`BUILTIN_SLASH_COMMANDS` 列出 24 个（`settings`、`model`、`tree`、`thinking`、`scoped-models`、`export`、`import`、`share`、`bug`、`copy`、`name`、`session`、`changelog`、`hotkeys`、`fork`、`clone`、`trust`、`login`、`logout`、`new`、`compact`、`resume`、`reload`、`quit`），这份列表只用于自动补全和冲突警告；`if` 链里另外还有 3 个不列出的命令（`/debug` 和两个彩蛋）。
- **其余的**交给 `session.prompt()`，在 `AgentSession` 里依次处理：扩展命令（找到就执行，即使正在运行）→ `input` 事件（第八章 5 节）→ `/skill:名字` 展开 → 提示词模板展开 → 排队或开始运行。

所以内置命令只存在于交互模式；扩展命令、技能和模板在四种模式里都能用（RPC 用 `get_commands` 列出它们）。扩展命令和内置命令同名时，内置命令先匹配，扩展命令从自动补全里隐去。

### 4.4 事件怎样变成画面

`subscribeToAgent` 用 `session.subscribe(handleEvent)` 订阅会话事件。`handleEvent` 对每个事件都让页脚失效重算，`switch` 里有 22 个 `case`（23 种里只少 `turn_end`，`bash_execution_update` 是空分支），大多数分支改完组件后请求重绘：

- `message_start`（assistant）：建一个 `AssistantMessageComponent` 加进 `chatContainer`。
- `message_update`：更新这个组件的内容；每遇到一个工具调用块，建或更新一个 `ToolExecutionComponent`。
- `message_end`：定稿。中止或出错时，把还在等待的工具组件都标成错误。
- `tool_execution_*`：更新工具组件的参数、进度和结果。
- `queue_update`：重画排队消息区。
- `entry_appended` 是压缩条目时：清空聊天区，按压缩后保留的条目重新渲染，加上摘要。
- `agent_start` / `agent_end`、`compaction_*`、`auto_retry_*`：显示和收起 Working 动画、状态提示。

组件只改自己的状态然后调用 `ui.requestRender()`，什么时候画、画哪些行由 pi-tui 决定（第 5 节）。

### 4.5 会话替换和 /reload 时界面怎样重建

- **`/new`、`/resume`、`/fork`、`/clone`、`/import`**：运行时替换 `AgentSession`。拆旧会话时先调用 `setBeforeSessionInvalidate` 注册的 `resetExtensionUI`，撤掉扩展留下的对话框、组件、页眉页脚、状态、快捷键和自定义编辑器；建好新会话后调用 `setRebindSession` 注册的回调 `rebindCurrentSession`：退订、应用设置、清空资源区、聊天区和排队区，按新会话重新渲染消息，再订阅、再 `bindExtensions`。
- **`/tree`**：不替换会话，只在同一个会话里 `navigateTree`，然后清空聊天区、按新分支重新渲染。
- **`/reload`**：运行中或压缩中拒绝执行。它先 `resetExtensionUI`，在编辑器位置显示 Reloading…，再调用 `session.reload()`。`AgentSession` 用上次保存的绑定重新接上新的扩展运行器（第九章 2.3 节），不经过 `rebindCurrentSession`；之后界面重新加载快捷键、主题、自动补全和资源清单。

### 4.6 ctx.ui 在交互模式里落到哪里

交互模式的 `uiContext` 由 `createExtensionUIContext` 创建，28 个成员（第九章 5 节）。它们落到 4.1 节的容器上：

| 方法 | 落点 |
|---|---|
| `select`、`confirm`、`input`、`editor` | 替换 `editorContainer` 的内容并取得焦点，回答后换回编辑器 |
| `custom(factory, { overlay })` | 默认同上；`overlay: true` 时作为叠加层浮在界面上 |
| `setWidget(key, …, { placement })` | 按 key 放进编辑器上方或下方的组件区 |
| `setFooter` / `setHeader` | 替换页脚 / 页眉组件 |
| `setStatus` | 页脚里的状态文字 |
| `notify` | 聊天区里的一条提示 |
| `setEditorComponent` | 换掉默认的编辑器 |

P4 让模型调用 `deploy`，确认框显示在编辑器的位置，带倒计时：

```text
12|────────────────────────────────────────────────────────────────────────────────
13|
14| Deploy?
15| Deploy to prod? (29s)
16|
17| → Yes
18|   No
19|
20| ↑↓ navigate  enter select  escape/ctrl+c cancel
21|
22|────────────────────────────────────────────────────────────────────────────────
```

按 Enter 选 Yes 之后，聊天区依次出现工具结果 `deployed to prod`、通知 `deploying` 和回复 `Deployed (tool said ok).`。倒计时的数字每秒重画一次，这一行就是那一秒里唯一被重写的内容。

---

## 5 · pi-tui：终端界面怎样画出来

![图 4 · pi-tui 的渲染管线：从 requestRender() 到终端](/images/harness/pi/ch10/fig4-tui-pipeline.png)

### 5.1 组件接口

pi-tui 里没有叫 `TUI` 的类，`TUI` 是一个接口；抽象基类 `TuiBase` 继承自 `Container`（容器组件），两个具体的渲染器是 `TuiMainScreen` 和 `TuiAltScreen`。组件只需要实现：

```typescript
export interface Component {
	render(width: number): string[];
	handleInput?(data: string): void;
	handleMouse?(event: TuiMouseEvent): TuiMouseEventResult | undefined;
	wantsKeyRelease?: boolean;
	invalidate(): void;
}
```

- `render(width)`（按给定宽度返回若干行字符串，可以带颜色序列）：每行的显示宽度不能超过 `width`。超过时 `TuiMainScreen` 写一份崩溃日志 `pi-tui-crash.log` 并抛错，`TuiAltScreen` 直接按列截断。
- `handleInput`（获得焦点时收到按键数据）、`handleMouse`（全屏模式的鼠标事件）、`wantsKeyRelease`（是否要按键松开事件）。
- `invalidate()`（主题变化等情况下丢弃缓存，下次从头渲染）。
- 实现 `Focusable`（`focused` 属性）的组件可以在输出里放一个 `CURSOR_MARKER`，渲染器据此摆放真实的终端光标，输入法的候选框就会跟着它。

组件之间没有"脏区域"的约定。组件改了状态就调用 `requestRender()`，每一帧都从根容器把整棵树渲染一遍，差分在渲染器里做。

### 5.2 渲染调度

- `requestRender()` 只置一个标志，在 `process.nextTick` 里调度；距上一帧不足 16 ms（`MIN_RENDER_INTERVAL_MS`）就用定时器推迟。一个 tick 里多少次请求都只画一帧。
- `requestRender(true)` 先清掉渲染器记住的上一帧，再在下一个 tick 立即渲染，跳过节流。
- `renderNow()` 同步渲染。

### 5.3 两种差分

两个渲染器共用"渲染整棵树 → 合成叠加层 → 找出光标标记"这几步，差别在比较和写法：

- **`TuiMainScreen`**：把新的行和上一帧逐行比较，找出第一行和最后一行变化的位置，只重写这一段。主屏上无法直接寻址已经输出的行，所以用相对移动（`CSI n A` 上移、`CSI n B` 下移）回到变化的行，`\r CSI 2K` 清行后重写。新增了行时，从第一处变化一直写到末尾。
- **`TuiAltScreen`**：画面固定为终端高度，按行和上一帧比较，只写不同的行，用绝对定位 `CSI 行;1H` 跳过去。
- 两者都把一帧的行更新包在 `CSI ?2026h … CSI ?2026l`（同步输出）里，支持它的终端会等这一段写完再一起显示，不会闪。光标归位的序列可能跟在后面。

P4 记录了每一次原始写入。全屏模式下 mock 每流出一段文字：

```text
 658 submit-slow   bytes=130 sync=2026h el2K=1 cupRows=[17,21]  "w0"
 720 submit-slow   bytes=338 sync=2026h el2K=1 cupRows=[20,21]  "── ⠸ Working ───…"
 816 submit-slow   bytes=130 sync=2026h el2K=1 cupRows=[17,21]  "w0 w1"
 960 submit-slow   bytes=130 sync=2026h el2K=1 cupRows=[17,21]  "w0 w1 w2"
```

每段文字只重写第 17 行（130 字节，跳到第 21 行是为了把光标放回编辑器）；Working 动画大约每 80 ms 换一帧，只重写第 20 行。常规模式下同样的更新是 145 字节，用的是 `CSI 3A` / `CSI 3B` 这样的相对移动；但第一段文字让回复多出一行，那一次从第一处变化写到了末尾（约 1 KB）。

P6 写了一个只用 pi-tui 的小程序：十行文字，每 100 ms 改第 5 行。终端收到的完整输出：

```text
+  83ms "\e[>7u\e[?u\e[c\e[?25l\e[?2026hrow 0: static…\r\r\nrow 1: static…"   ← 第一帧：整屏输出
+  85ms "\e[>4;2m"                                                          ← 终端不支持 Kitty 键盘协议，改用 modifyOtherKeys
+ 184ms "\e[?2026h\e[5A\r\e[2Krow 4: tick=1\e[0m…\e[?2026l"                  ← 上移 5 行，清行，重写
+ 285ms "\e[?2026h\r\e[2Krow 4: tick=2\e[0m…\e[?2026l"                      ← 光标已在这一行，不用移动
+ 388ms "\e[?2026h\r\e[2Krow 4: tick=3\e[0m…\e[?2026l"
```

（`\e` 即 ESC。）

### 5.4 什么时候整屏重画

常规模式在这些情况下放弃差分，写 `CSI 2J CSI H CSI 3J` 后从头输出全部行：终端宽度变化、高度变化（Termux 除外）、开启 `clearOnShrink` 且内容变短、第一处变化在已经滚出视口的行上、删掉的行超过一屏。`CSI 3J` 会连终端的回滚区一起清掉，所以常规模式下改变窗口宽度，之前的输出会被整体重新打印一遍。第一帧不清屏，直接输出。

全屏模式在第一帧和终端尺寸变化时写 `CSI 2J`，再写满每一行。

设置环境变量 `PI_TUI_DEBUG_REDRAW=1` 后，常规模式每次整屏重画都会把原因追加到日志文件 `pi-tui-debug.log`。

### 5.5 按键和输入

- `ProcessTerminal`（基于 `process.stdin` / `stdout` 的终端实现）启动时发出 `CSI >7u`（请求 Kitty 键盘协议）、`CSI ?u`（查询是否支持）和 `CSI c`（设备属性）。终端回了设备属性却没回 Kitty 查询，就改发 `CSI >4;2m` 打开 modifyOtherKeys。P6 里 xterm.js 走的就是这条退路。
- `StdinBuffer`（输入缓冲）把被拆开的转义序列重新拼好，单独的 ESC 等 10 ms 才判定为 Escape 键，并处理括号粘贴（`CSI ?2004h`）。
- `keys.ts` 同时解析旧式序列和 Kitty 协议，`matchesKey(data, "ctrl+c")` 这类函数供组件判断按键；Kitty 协议下还能区分按下、重复和松开。

### 5.6 主题和暗色适配

pi-tui 不带主题文件，只提供颜色工具和终端颜色查询：用 OSC 10 / 11 查询前景色和背景色、OSC 4 查询 16 色调色板，用 `CSI ?2031h` 订阅终端的明暗切换通知。

主题在 pi-coding-agent 里：`dark.json`、`light.json` 两套内置主题，加上 `system-theme.ts` 根据终端背景色和调色板生成的 system 主题。设置成一对明暗主题或 system 主题时，`theme-controller.ts` 在终端明暗切换后自动换主题；查询终端颜色时最多等 100 ms。

### 5.7 单独使用

pi-tui 的 `dependencies` 只有 `get-east-asian-width`（计算东亚字符宽度）和 `marked`（Markdown 解析），不依赖任何 `@earendil-works` 包。导出的组件有 `Box`、`Editor`、`HStack`、`VStack`、`Image`、`Input`、`Loader`、`Markdown`、`ScrollView`、`SelectList`、`SettingsList`、`Text` 等，渲染器 `TuiMainScreen`、`TuiAltScreen` 和终端实现 `ProcessTerminal` 也都导出。

P6 只装了 `@earendil-works/pi-tui`，程序的全部代码：

```javascript
import { ProcessTerminal, TuiMainScreen } from "@earendil-works/pi-tui";
class Board {
	constructor() { this.tick = 0; }
	render(width) { return Array.from({ length: 10 }, (_, i) => (i === 4 ? `row ${i}: tick=${this.tick}` : `row ${i}: static`).slice(0, width)); }
	invalidate() {}
}
const tui = new TuiMainScreen(new ProcessTerminal());
const board = new Board();
tui.addChild(board);
tui.start();
const iv = setInterval(() => { board.tick++; tui.requestRender(); if (board.tick === 3) { clearInterval(iv); setTimeout(() => { tui.stop(); process.exit(0); }, 50); } }, 100);
```

`packages/tui/README.md` 的快速上手示例从 `./test/test-themes.ts` 导入编辑器主题，这个目录不在发布的包里；照着写时要自己提供一个 `EditorTheme`。

---

## 6 · SDK 与远程

![图 5 · SDK 自己驱动 AgentSessionRuntime：模式替你做的接线](/images/harness/pi/ch10/fig5-sdk-wiring.png)

### 6.1 自己驱动时要补的接线

`createAgentSession()` 适合一次性的嵌入；要支持 `/new`、`/resume`、`/fork` 这类替换会话的操作，用 `createAgentSessionRuntime(createRuntime, { cwd, agentDir, sessionManager })`，和 `main()` 的做法一样。两者都不会替你绑定扩展。1.5 节的三件事要自己做，P5 逐项验证了漏掉的后果（扩展里有一个调用 `ctx.ui.confirm` 的工具和一个会抛错的 `turn_end` 处理函数）：

```text
--- A: no bindExtensions
  tool result: declined
--- B: bindExtensions with a custom uiContext and onError, via setRebindSession
  [ext] session_start reason=startup mode=rpc hasUI=true
  [ui] confirm "Deploy?" -> true
  [onError] turn_end: turn_end handler failed
  [onError] turn_end: turn_end handler failed
  [event] agent_settled
  tool result: deployed
--- C: runtime.newSession() (rebind happens inside)
  [ext] session_start reason=new mode=rpc hasUI=true
--- E: second runtime, no setRebindSession; newSession() then manual bind
  [ext] session_start reason=startup mode=print hasUI=false
  newSession() …
  manual bindExtensions …
  [ext] session_start reason=new mode=print hasUI=false
--- F: setRebindSession AND manual bind after newSession()
  [ext] session_start reason=new mode=print hasUI=false
  [ext] session_start reason=new mode=print hasUI=false
```

- **A**：不绑定时扩展收不到 `session_start`，`ctx.ui.confirm` 返回 `false`，`turn_end` 里的两次抛错没有任何人收到。
- **B**：`bindExtensions` 可以传宿主自己实现的 `uiContext`。P5 用一个代理对象，`confirm` 一律回答 `true`，其余方法都是空函数；`mode` 传什么，`ctx.mode` 就是什么。
- **C、E、F**：注册了 `setRebindSession`，`runtime.newSession()` 会在内部重新绑定；没注册就要在替换之后自己再绑一次（官方示例 `examples/sdk/13-session-runtime.ts` 用的就是这种写法）；两种都做，扩展会收到两次 `session_start`，和 RPC 模式现在的情况一样（3.6 节）。

### 6.2 直接复用模式

不想自己接线时，包的入口直接导出了这几个模式：

- `runPrintMode(runtime, options)`、`runRpcMode(runtime)`、`new InteractiveMode(runtime, options)`：把自己建好的运行时交给它们，得到和 `pi` 命令一样的输出和接线。
- `main(args, { extensionFactories })`：整个 CLI，可以带上自己的扩展工厂，相当于一个加了内置扩展的 `pi`。
- `RpcClient`：用 `node` 启动 `cliPath` 指向的 CLI 脚本并加上 `--mode rpc`（不传时是相对当前目录的 `dist/cli.js`，用 npm 安装的 pi 时要指向包里的 CLI 文件），每条命令一个方法，外加 `promptAndWait()`、`waitForIdle()`。它没有回答扩展对话框的接口（3.7 节），需要时自己读 stdout（第 7 节）。
- RPC 的类型 `RpcCommand`、`RpcResponse`、`RpcExtensionUIRequest`、`RpcExtensionUIResponse` 和 JSON 事件的类型 `JsonAgentSessionEvent` 也都导出。

### 6.3 client、server、protocol 的边界

仓库里另有三个包和远程会话有关：`@earendil-works/pi-protocol`（CBOR 编码的消息信封和分帧）、`pi-client`（通过这套协议连接远程会话的客户端）、`pi-server`（自称 "experimental server package"，把客户端路由到持久会话）。它们和其他包一起以 1.0.2 发布在 npm 上，README 都说明服务于实验性的 Pi 服务协议。

它们和本章讲的路径没有交集：

- pi-coding-agent 只把这三个包列在 `devDependencies` 里。用到它们的代码在 `src/experimental/`、`src/cli/experimental/` 和 `src/client/` 下，三处编译出的目录都被 `files` 排除，不进发布的包。
- 发布的 `pi` 命令不引用它们；实验命令要从 `src/experimental/cli.ts` 启动，并设置 `PI_EXPERIMENTAL=1`。
- 远程客户端的终端界面复用了交互模式的 `createChatViewport` 和 `CustomEditor`。

持久会话这一侧的实现留给番外 pi-durable。

---

## 7 · 用法：一个最小 RPC 客户端

下面的客户端用 Node 写，覆盖协议的四件事：发 prompt、流式打印回复、回答一次扩展的 confirm、中途 abort。它只用 Node 自带的模块；从 pi 包里导入的都是类型，运行时不需要这个包，类型检查时需要。

```typescript
// Minimal RPC client: prompt, stream the reply, answer an extension confirm, abort mid-stream.
// Run: node rpc-client.ts [extra pi args...]   (Node 22.18+ runs .ts directly)
import { spawn } from "node:child_process";
import type {
	JsonAgentSessionEvent,
	RpcCommand,
	RpcExtensionUIRequest,
	RpcExtensionUIResponse,
	RpcResponse,
} from "@earendil-works/pi-coding-agent";

type ExtensionError = { type: "extension_error"; extensionPath: string; event: string; error: string };
type Incoming = RpcResponse | RpcExtensionUIRequest | ExtensionError | JsonAgentSessionEvent;
type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never;

const pi = spawn(process.env.PI_BIN ?? "pi", ["--mode", "rpc", ...process.argv.slice(2)], {
	stdio: ["pipe", "pipe", "inherit"], // stderr carries diagnostics, never protocol records
});

// Framing: split stdout on LF only (not readline: it also splits on U+2028/U+2029).
const listeners = new Set<(record: Incoming) => void>();
let buffer = "";
pi.stdout.setEncoding("utf8"); // decodes multi-byte characters split across chunks
pi.stdout.on("data", (chunk: string) => {
	buffer += chunk;
	for (let i = buffer.indexOf("\n"); i >= 0; i = buffer.indexOf("\n")) {
		const line = buffer.slice(0, i).replace(/\r$/, "");
		buffer = buffer.slice(i + 1);
		if (line) for (const listener of [...listeners]) listener(JSON.parse(line) as Incoming);
	}
});

const write = (record: RpcCommand | RpcExtensionUIResponse) => pi.stdin.write(`${JSON.stringify(record)}\n`);

function next(match: (record: Incoming) => boolean): Promise<Incoming> {
	return new Promise((resolve) => {
		const listener = (record: Incoming) => {
			if (!match(record)) return;
			listeners.delete(listener);
			resolve(record);
		};
		listeners.add(listener);
	});
}

let seq = 0;
async function request(command: DistributiveOmit<RpcCommand, "id">): Promise<RpcResponse> {
	const id = `req-${++seq}`;
	const reply = next((r) => r.type === "response" && r.id === id); // subscribe before writing
	write({ ...command, id } as RpcCommand);
	const response = (await reply) as RpcResponse;
	if (!response.success) throw new Error(`${response.command} failed: ${response.error}`);
	return response;
}

// Extension UI: answer dialogs; fire-and-forget methods need no reply.
function onExtensionUi(req: RpcExtensionUIRequest): void {
	switch (req.method) {
		case "confirm":
			console.log(`\n[confirm] ${req.title} ${req.message} -> yes`);
			write({ type: "extension_ui_response", id: req.id, confirmed: true });
			break;
		case "select":
		case "input":
		case "editor":
			write({ type: "extension_ui_response", id: req.id, cancelled: true });
			break;
		case "notify":
			console.log(`[notify] ${req.message}`);
			break;
		default: // setStatus, setWidget, setTitle, set_editor_text
			break;
	}
}

let lastStopReason = "";
listeners.add((record) => {
	if (record.type === "extension_ui_request") onExtensionUi(record);
	else if (record.type === "extension_error") console.error(`[extension error] ${record.extensionPath}: ${record.error}`);
	else if (record.type === "message_update" && record.assistantMessageEvent.type === "text_delta")
		process.stdout.write(record.assistantMessageEvent.delta);
	else if (record.type === "message_end" && record.message.role === "assistant") lastStopReason = record.message.stopReason;
});

// One prompt; optionally abort after N text deltas.
async function run(message: string, abortAfterDeltas?: number): Promise<void> {
	console.log(`\n> ${message}`);
	const settled = next((r) => r.type === "agent_settled");
	let deltas = 0;
	const aborter = (record: Incoming) => {
		if (record.type !== "message_update" || record.assistantMessageEvent.type !== "text_delta") return;
		if (++deltas === abortAfterDeltas) {
			listeners.delete(aborter);
			console.log("\n[abort]");
			void request({ type: "abort" });
		}
	};
	if (abortAfterDeltas !== undefined) listeners.add(aborter);
	const response = await request({ type: "prompt", message });
	if (response.command === "prompt" && response.success && response.data?.disposition === "handled") return;
	await settled;
	listeners.delete(aborter);
	console.log(`\n[settled] stopReason=${lastStopReason}`);
}

await run("TOOL: deploy the app to prod");
await run("SLOW: tell me a long story", 5);
pi.stdin.end(); // orderly shutdown: pi disposes the session and exits
pi.on("exit", (code) => console.log(`[pi exited] code=${code}`));
```

几处写法对应前面的结论：

- **只按 LF 切分**，`setEncoding("utf8")` 处理跨块的多字节字符（3.1 节）。
- **先订阅再写入**：`request()` 先挂上等响应的监听函数，`run()` 在发 prompt 之前就开始等 `agent_settled`，快速完成的运行不会漏掉事件。
- **按 `id` 配对**：`abort` 的响应要等到空闲才来，`agent_settled` 在它前面（3.5 节）。
- **`disposition` 为 `handled` 时不等 `agent_settled`**：这次输入被扩展命令或 `input` 事件处理掉了，不会有运行（3.3 节）。
- 回答 `confirm` 时用 `confirmed`，其他对话框一律取消；`notify` 这类单向消息不回复（3.4 节）。

用 `tsc --strict` 检查没有错误。环境：从 npm 装 `@earendil-works/pi-coding-agent@1.0.2` 和 `@types/node`，`package.json` 写 `"type": "module"`（文件用了顶层 `await`），tsconfig 用 `module: nodenext` 并打开 `skipLibCheck`（不打开时，pi-ai 自带的 `.d.ts` 在 nodenext 下有 JSON 导入的报错，和这份代码无关）。服务端用的是 §0 的探针扩展：

```typescript
// Probe extension: a `deploy` tool that asks for confirmation through ctx.ui.
import { Type } from "@earendil-works/pi-ai";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
export default function (pi: ExtensionAPI) {
	pi.registerTool({
		name: "deploy",
		label: "Deploy",
		description: "Deploy to an environment",
		parameters: Type.Object({ env: Type.String() }),
		async execute(_id, params, _signal, _onUpdate, ctx) {
			const ok = await ctx.ui.confirm("Deploy?", `Deploy to ${params.env}?`, { timeout: 30000 });
			ctx.ui.notify(ok ? "deploying" : "cancelled", "info");
			return { content: [{ type: "text", text: ok ? `deployed to ${params.env}` : "user declined" }], details: { ok } };
		},
	});
	// The [ext] lines in sections 2 and 3; silenced with PROBE_LOG=0 in the TUI probe.
	if (process.env.PROBE_LOG !== "0") pi.on("session_start", (e, ctx) => console.error(`[ext] session_start reason=${e.reason} mode=${ctx.mode} hasUI=${ctx.hasUI}`));
	if (process.env.PROBE_LOG !== "0") pi.on("session_shutdown", (e) => console.error(`[ext] session_shutdown reason=${e.reason}`));
}
```

运行 `PROBE_LOG=0 node rpc-client.ts --no-session -e ext-deploy.ts`，终端输出：

```text
> TOOL: deploy the app to prod

[confirm] Deploy? Deploy to prod? -> yes
[notify] deploying
Deployed (tool said ok).
[settled] stopReason=stop

> SLOW: tell me a long story
w0 w1 w2 w3 w4 
[abort]

[settled] stopReason=aborted
[pi exited] code=0
```

客户端和 pi 之间真实的 JSONL（用一个透明的包装进程按到达顺序记录两个方向，共 50 行；`message` 只保留角色、内容和 `stopReason`，`usage` 和 system 消息的内容换成 `…`，UUID 截短）：

```text
>> {"type":"prompt","message":"TOOL: deploy the app to prod","id":"req-1"}
<< {"id":"req-1","type":"response","command":"prompt","success":true,"data":{"disposition":"started"}}
<< {"type":"agent_start"}
<< {"type":"turn_start"}
<< {"type":"message_start","message":{"role":"system","content":"…"}}
<< {"type":"message_end","message":{"role":"system","content":"…"}}
<< {"type":"message_start","message":{"role":"user","content":[{"type":"text","text":"TOOL: deploy the app to prod"}]}}
<< {"type":"message_end","message":{"role":"user","content":[{"type":"text","text":"TOOL: deploy the app to prod"}]}}
<< {"type":"message_start","message":{"role":"assistant","content":[{"type":"toolCall","name":"deploy","arguments":{}}],"stopReason":"pending"}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"toolcall_start","contentIndex":0,"id":"toolu_1","toolName":"deploy"}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"toolcall_delta","contentIndex":0,"delta":"{\"env\":\"prod\"}"}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"toolcall_end","contentIndex":0,"toolCall":{"name":"deploy","arguments":{"env":"prod"}}}}
<< {"type":"message_end","message":{"role":"assistant","content":[{"type":"toolCall","name":"deploy","arguments":{"env":"prod"}}],"stopReason":"toolUse"}}
<< {"type":"tool_execution_start","toolCallId":"toolu_1","toolName":"deploy","args":{"env":"prod"}}
<< {"type":"extension_ui_request","id":"0e6d77e7…","method":"confirm","title":"Deploy?","message":"Deploy to prod?","timeout":30000}
>> {"type":"extension_ui_response","id":"0e6d77e7…","confirmed":true}
<< {"type":"extension_ui_request","id":"2170b073…","method":"notify","message":"deploying","notifyType":"info"}
<< {"type":"tool_execution_end","toolCallId":"toolu_1","toolName":"deploy","result":{"content":[{"type":"text","text":"deployed to prod"}]},"isError":false}
<< {"type":"message_start","message":{"role":"toolResult","content":[{"type":"text","text":"deployed to prod"}],"toolName":"deploy"}}
<< {"type":"message_end","message":{"role":"toolResult","content":[{"type":"text","text":"deployed to prod"}],"toolName":"deploy"}}
<< {"type":"turn_end","message":{"role":"assistant","content":[{"type":"toolCall","name":"deploy","arguments":{"env":"prod"}}],"stopReason":"toolUse"},"toolResults":"[1]"}
<< {"type":"turn_start"}
<< {"type":"message_start","message":{"role":"assistant","content":[{"type":"text","text":""}],"stopReason":"pending"}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_start","contentIndex":0}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"Deployed"}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":" (tool said ok)."}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_end","contentIndex":0,"content":"Deployed (tool said ok)."}}
<< {"type":"message_end","message":{"role":"assistant","content":[{"type":"text","text":"Deployed (tool said ok)."}],"stopReason":"stop"}}
<< {"type":"turn_end","message":{"role":"assistant","content":[{"type":"text","text":"Deployed (tool said ok)."}],"stopReason":"stop"},"toolResults":"[0]"}
<< {"type":"agent_end","messages":"[5 messages]","willRetry":false}
<< {"type":"agent_settled"}
>> {"type":"prompt","message":"SLOW: tell me a long story","id":"req-2"}
<< {"id":"req-2","type":"response","command":"prompt","success":true,"data":{"disposition":"started"}}
<< {"type":"agent_start"}
<< {"type":"turn_start"}
<< {"type":"message_start","message":{"role":"user","content":[{"type":"text","text":"SLOW: tell me a long story"}]}}
<< {"type":"message_end","message":{"role":"user","content":[{"type":"text","text":"SLOW: tell me a long story"}]}}
<< {"type":"message_start","message":{"role":"assistant","content":[{"type":"text","text":""}],"stopReason":"pending"}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_start","contentIndex":0}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"w0 "}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"w1 "}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"w2 "}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"w3 "}}
<< {"type":"message_update","usage":"…","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"w4 "}}
>> {"type":"abort","id":"req-3"}
<< {"type":"message_end","message":{"role":"assistant","content":[{"type":"text","text":"w0 w1 w2 w3 w4 "}],"stopReason":"aborted","errorMessage":"This operation was aborted"}}
<< {"type":"turn_end","message":{"role":"assistant","content":[{"type":"text","text":"w0 w1 w2 w3 w4 "}],"stopReason":"aborted","errorMessage":"This operation was aborted"},"toolResults":"[0]"}
<< {"type":"agent_end","messages":"[2 messages]","willRetry":false}
<< {"type":"agent_settled"}
<< {"id":"req-3","type":"response","command":"abort","success":true}
```

这份记录和图 2 的 15 步一一对应。第二次运行没有 system 消息：提示词补丁只在提示词变化时才写（第八章 4 节）。

---

## 8 · 三个可以带走的方法

1. **把呈现做成会话的订阅者**。四种模式都不碰 `AgentSession` 的内部，只做"绑定扩展 + 订阅事件 + 调用 `prompt` / `abort`"。终端界面、一行一个 JSON、双向协议只是三种订阅者；SDK 用户用同样的三步就能做出第五种呈现。
2. **协议按"受理"回复，用事件报告"完成"**。RPC 的 `prompt` 在预检通过时就回 `started` / `queued` / `handled`，运行的结果全在事件流里，以 `agent_settled` 收尾；命令并发处理、靠 `id` 配对。慢操作不会挡住查询，客户端也不用猜一个响应到底代表什么。
3. **组件只管输出行，差分交给渲染器**。组件的唯一义务是 `render(width): string[]`，每帧从根重新渲染；渲染器和上一帧逐行比较，只重写变化的行，再用同步输出包起来。组件写起来简单，终端收到的字节又少。

---

## 9 · 关键数字

| 项 | 数量 |
|---|---|
| 运行模式 | 4 种（交互、print、JSON、RPC），由 `resolveAppMode` 选出 |
| 命令行参数 | 40 个（不含 `--` 和 `@文件`） |
| RPC 命令 | 33 条（类型定义、`switch` 分支、`rpc-commands.md` 三处一致） |
| 会话事件（JSON / RPC 输出） | 23 种（agent-core 10 种 + AgentSession 13 种） |
| RPC 额外的 stdout 记录 | 3 种（`response`、`extension_ui_request`、`extension_error`） |
| `extension_ui_request` 的 `method` | 9 种（4 种对话框 + 5 种单向通知） |
| 交互模式 `switch` 里的事件 | 22 种（少 `turn_end`；`bash_execution_update` 为空分支） |
| 内置斜杠命令 | 列表 24 个，`if` 链另有 3 个 |
| 自动重试 | 3 次，间隔 2 s、4 s、8 s（探针中的默认值） |
| 退出码 | 0 正常；1 失败（text 模式含 `error` / `aborted`）；143 `SIGTERM`；129 `SIGHUP` |
| 渲染节流 | 16 ms |
| 流式文字的一次屏幕更新（P4） | 全屏 130 字节 / 常规 145 字节，只重写 1 行 |
| pi-tui 的运行时依赖 | 2 个（`get-east-asian-width`、`marked`） |
| 源码体量 | `interactive-mode.ts` 7072 行、`rpc-mode.ts` 819 行、`print-mode.ts` 169 行、`main.ts` 998 行；pi-tui 的 `src` 约 1.9 万行 |

---

## 10 · 术语表

| 术语 | 含义 | 别和它混淆 |
|---|---|---|
| **运行模式** | `main()` 末尾选择的呈现方式：交互、print、JSON、RPC | `ctx.mode`：扩展看到的 `tui` / `print` / `json` / `rpc` |
| **运行时** | `AgentSessionRuntime`，持有当前会话并负责替换 | 扩展运行器 `ExtensionRunner`（第九章） |
| **重新绑定** | 会话被替换后，模式对新会话再做 `bindExtensions` 和订阅 | `/reload`：同一个会话换扩展运行器，复用已保存的绑定 |
| **disposition** | RPC `prompt` 响应里的受理结果：`started`、`queued`、`handled` | 运行结果：看 `message_end` 的 `stopReason` 和 `agent_settled` |
| **增量事件** | JSON / RPC 里去掉累积快照的 `message_update` | 会话内部的 `message_update`：带着整条消息 |
| **扩展 UI 子协议** | `extension_ui_request` / `extension_ui_response` 两种记录 | 命令和响应：带 `id` 的 `type: "response"` |
| **差分渲染** | 与上一帧逐行比较，只重写变化的行 | 整屏重画：首帧、尺寸变化等情况下清屏重写 |
| **同步输出** | `CSI ?2026h … CSI ?2026l` 包起来的一段输出，终端写完再显示 | 备用屏幕 `CSI ?1049h`：全屏模式切换到的另一块画面 |

---

## 11 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 启动流程、选模式 | `packages/coding-agent/src/main.ts` → `main`、`resolveAppMode`、`readPipedStdin`、`prepareInitialMessage`、`createSessionManager`；`src/cli.ts`、`src/cli/setup.ts` |
| 参数和初始消息 | `src/cli/args.ts` → `parseArgs`；`src/cli/file-processor.ts`、`src/cli/initial-message.ts` |
| stdout 接管 | `src/core/output-guard.ts` → `takeOverStdout`、`writeRawStdout`、`flushRawStdout` |
| print / JSON | `src/modes/print-mode.ts` → `runPrintMode`；`src/modes/json-event.ts` → `toJsonEvent` |
| RPC | `src/modes/rpc/rpc-mode.ts` → `runRpcMode`、`createDialogPromise`、`createExtensionUIContext`、`handleCommand`；`rpc-types.ts`、`jsonl.ts`、`rpc-client.ts` |
| prompt 的受理 | `src/core/agent-session.ts` → `prompt`（`preflightResult`）、`steer`、`followUp`、`abort`、`bindExtensions`；`AgentSessionEvent` 类型 |
| 运行时与替换 | `src/core/agent-session-runtime.ts` → `createAgentSessionRuntime`、`setRebindSession`、`setBeforeSessionInvalidate`、`finishSessionReplacement`、`dispose` |
| 交互模式 | `src/modes/interactive/interactive-mode.ts` → 构造函数、`init`、`run`、`setupEditorSubmitHandler`、`handleFollowUp`、`subscribeToAgent`、`handleEvent`、`rebindCurrentSession`、`bindCurrentSessionExtensions`、`resetExtensionUI`、`createExtensionUIContext`；`tui-renderer.ts`、`chat-viewport.ts`；`src/core/slash-commands.ts`、`src/core/keybindings.ts` |
| 主题 | `src/modes/interactive/theme/` → `theme.ts`、`theme-controller.ts`、`system-theme.ts`、`dark.json`、`light.json` |
| pi-tui | `packages/tui/src/tui.ts` → `Component`、`TuiBase`、`requestRender`；`tui-main-screen.ts` → `doRender`；`tui-alt-screen.ts` → `doRender`；`terminal.ts`、`stdin-buffer.ts`、`keys.ts`、`terminal-colors.ts`；`packages/tui/README.md` |
| SDK | `src/core/sdk.ts` → `createAgentSession`；`examples/sdk/13-session-runtime.ts`；`src/index.ts` 的模式导出 |
| 官方说明 | `packages/coding-agent/docs/cli.md`、`cli-integration.md`、`json.md`、`rpc.md`、`rpc-commands.md`、`rpc-extension-ui.md`、`sdk.md`、`tui.md` |

这一章讲完了 pi 的四种入口。到这里，主干的十章从模型调用、循环、工具、会话、压缩、上下文、扩展一直讲到了最外层的界面。
