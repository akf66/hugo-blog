---
title: "Pi 源码分析 06 · 会话：一条消息怎么存进会话树，又怎么变回上下文"
date: 2026-10-02T18:00:00+08:00
description: "接着第五章往下讲：coding-agent 的会话文件里有哪些条目、什么时候写盘，条目怎样用 parentId 连成一棵树，/tree、/fork、/clone 分别动了什么，每次请求模型前上下文怎样从当前分支重新投影出来，压缩和 context_edit 怎样参与投影，以及 AgentSession 怎样让会话树成为上下文的唯一来源。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.1（2026-10-03），commit `a7229ddc2`。文中所有行为、数字和代码均以该版本为准。

前五章讲的都是一次运行内部的事：循环怎么转、模型怎么调、工具怎么跑。运行结束以后，这些消息去了哪里？下一次请求时，模型看到的历史又是从哪来的？本章接着往下讲 coding-agent 的会话：一条消息怎么写进会话树，又怎么在下一次请求前变回上下文。

<!--more-->

本章要点：

- **会话是一棵只增不改的树**：一个会话就是一个 JSONL 文件。第一行是文件头，之后每行一个条目，用 `parentId` 连成树，`leaf` 指向当前位置。新条目总是挂在 leaf 下面。切换分支只移动 leaf，已有条目不删也不改。
- **11 种条目分四组**：会变成消息的 4 种，改写别的条目的 1 种（`context_edit`），决定模型和思考级别的 2 种，模型完全看不到的 4 种。
- **文件按需创建，之后逐行追加**：只有模型和思考级别这类设置时，条目只留在内存；第一条 user 或 assistant 消息出现时才建文件，之后每个条目追加一行。leaf 本身不写进文件，重新打开会话时落在最后一行。
- **上下文每次请求前重新投影**：`AgentSession` 在每次请求前，从 leaf 走到根，按最新的压缩条目裁剪，再套用 `context_edit`，用结果替换要发给模型的消息。`agent.state.messages` 只是这份投影的一个副本。
- **编辑上下文也是追加条目**：失败的重试、扩展想隐藏的旧输出，都通过追加一条 `context_edit` 从后续上下文里去掉或替换。原始条目留在文件里，换到别的分支时也不受影响。

---

## 0 · 阅读说明

- 本章讲已发布的 `pi` 用的会话实现：pi-coding-agent 的 `SessionManager`（会话的读写和树操作）和 `AgentSession` 里接线的部分。源码里 `src/experimental/` 下还有一套基于 pi-durable 和 SQLite 的会话，不随 npm 包发布，不在本章范围内。
- 压缩怎么挑选要总结的消息、摘要怎么生成，留给第七章。本章只讲压缩条目写进会话以后，怎样参与上下文的投影。
- 第二章 3.2 节讲过 4 种自定义消息（`bashExecution`、`custom`、`branchSummary`、`compactionSummary`）和 `convertToLlm`，第五章讲过 system 消息上的 `sections`、`toolsAdded`、`toolsRemoved`。本章引用这些结论，不再重复。
- 文中的行为都用探针实际跑过。探针用 npm 上的 1.0.1，一部分直接调用 `SessionManager`，另一部分用 `createAgentSession` 加 pi-ai 自带的 faux 供应商（按脚本返回预先写好的回复，并能拿到每次请求的上下文），不需要真实的 API key。
- 术语：
  - **条目**（entry）：会话文件里除文件头以外的一行。
  - **leaf**：当前位置，下一个条目会挂在它下面。
  - **当前分支**：从 leaf 沿 `parentId` 走到根经过的条目。
  - **投影**：从当前分支算出发给模型的消息列表。

---

## 1 · 一条消息的全程

![图 1 · 会话的两个方向](/images/harness/pi/ch06/fig1-roundtrip.png)

会话在一次运行里有两个方向，图 1 左列是写，右列是读：

1. **写**：`Agent` 每完成一条消息就发出 `message_end`（第三章）。`AgentSession` 在事件处理里把它交给 `SessionManager`：`custom` 消息走 `appendCustomMessageEntry`（写一条扩展消息条目），`system`、`user`、`assistant`、`toolResult` 走 `appendMessage`（写一条普通消息条目）。新条目的 `parentId` 是当前 leaf，写完后 leaf 移到新条目上，再追加到文件末尾。
2. **读**：下一次请求模型之前，`AgentSession` 装在 `Agent` 上的 `prepareRequest`（每次请求前调用的钩子，第三章）调用 `buildSessionProjection`（从当前分支投影出消息），用它的结果替换循环里的消息。之后再经过 `transformContext` 和 `convertToLlm`，交给 pi-ai。

两边的交汇点是 `SessionManager` 在内存里保存的全部条目。请求时读的是内存，不读文件；文件是内存的持久副本，重新打开会话时，内存从文件重建。

`Agent` 的事件处理是逐个等待的：`message_end` 的处理函数返回之后，循环才继续往下走。所以下一次请求做投影时，上一条消息一定已经在树上。

---

## 2 · 会话文件里有什么

### 2.1 文件和位置

默认位置是 `~/.pi/agent/sessions/--<工作目录>--/<时间戳>_<会话 id>.jsonl`。工作目录去掉开头的路径分隔符，再把 `/`、`\`、`:` 换成 `-`。会话 id 是 UUIDv7，条目 id 是 8 位十六进制，与本会话已有的 id 冲突就重新生成，100 次都冲突就改用完整 UUID。

探针用 `createAgentSession` 跑了一次带工具调用的对话（读一个文件，再回答一句），文件里依次是：

```text
session                   ← 文件头：version 3、id、timestamp、cwd
model_change              ← 新会话先记下模型
thinking_level_change     ← 和思考级别
message  system           ← 第一次请求时写下完整的提示词和工具声明
message  user
message  assistant        ← 调用 read
message  toolResult
message  assistant
```

系统提示词和工具声明也是普通的 `message` 条目，role 为 `system`。第一条带上全部 `sections` 和 `toolsAdded`，之后的变化写成补丁（第五章），除了压缩条目里的检查点（4.2 节），会话里没有另外存一份"当前提示词"。

### 2.2 11 种条目

![图 2 · 11 种条目分成四组](/images/harness/pi/ch06/fig2-entries.png)

文件头之外，`SessionEntry` 有 11 种。图 2 按它们对模型上下文的作用分组：

| 组 | 条目 | 投影时变成 |
|---|---|---|
| 变成消息 | `message` | 存进去的 `AgentMessage` 原样 |
| | `custom_message` | 一条 `custom` 消息；`display` 和 `details` 不发给模型 |
| | `branch_summary` | 一条 `branchSummary` 消息 |
| | `compaction` | 一条 system 检查点，加一条 `compactionSummary` 消息 |
| 改写别的条目 | `context_edit` | 自己不产生消息；替换或去掉目标条目的内容 |
| 决定设置 | `model_change`、`thinking_level_change` | 不产生消息；决定恢复会话时用的模型和思考级别 |
| 不进上下文 | `usage`、`custom`、`label`、`session_info` | 什么都不产生 |

几个容易混的地方：

- `custom` 和 `custom_message` 都由扩展写入。`custom` 用来保存扩展自己的状态（`pi.appendEntry`），模型看不到；`custom_message` 是扩展插进对话的消息，模型看得到。
- `label` 和 `session_info` 也是树上的节点，写入时同样挂在 leaf 下面并移动 leaf。它们不产生消息，所以不影响上下文。
- `SessionManager.appendMessage` 的参数类型里没有 `branchSummary` 和 `compactionSummary` 两种消息（运行时不做检查）。这两种摘要应该分别通过 `branchWithSummary` 和 `appendCompaction` 写成独立的条目，方便以后查找。

### 2.3 什么时候写盘

**第一条对话消息出现时才建文件**。`newSession`（新建一个空会话）只在内存里准备文件头。`_persist`（把条目写进文件）在会话里出现 user 或 assistant 消息之前什么都不做，所以打开 pi 什么都不说就退出，不会留下空文件。第一次写盘时，用 `wx` 方式（文件已存在就失败）新建文件，把内存里攒下的条目一次写进去；之后每个条目用 `appendFileSync` 追加一行。探针确认：写完模型、思考级别和 system 消息后文件还不存在，写入第一条 user 消息后文件出现，里面有 5 行。

**有三种情况会整体重写文件**：

- 打开的文件版本低于 3，迁移后整个重写（2.4 节）
- 打开一个 0 字节的文件，补上文件头
- `/fork`、`/clone` 截取出的新文件（3.3 节）

**不是所有条目都在 `message_end` 时写入**：

- 用 `!` 执行的命令由 `recordBashResult`（记录一次 `!` 命令的结果）写成 `bashExecution` 消息。如果模型正在回复，它先排队，等这次运行结束再写
- 压缩摘要和分支摘要由各自的流程写成独立条目
- 插队消息、后续消息和扩展排进下一轮的消息，在真正进入循环、发出 `message_end` 之前，都只在队列里

### 2.4 版本与容错

文件头的 `version` 当前是 3：

- 第 1 版没有 `id` 和 `parentId`，迁移时按行序给每个条目生成 id，把前一条设为父节点，连成一条链；压缩条目里的 `firstKeptEntryIndex` 换成 `firstKeptEntryId`。
- 第 2 版到第 3 版只把消息角色 `hookMessage` 改名为 `custom`。

加载时自动迁移并重写文件。解析不了的行直接跳过；文件非空、第一条能解析的记录却不是文件头，就拒绝打开，不改动原文件。最后一行缺换行符时，加载时会补上。

---

## 3 · 树与分支

![图 3 · 会话树](/images/harness/pi/ch06/fig3-tree.png)

### 3.1 leaf 决定新条目挂在哪

所有 `append*` 方法都走同一个私有函数 `_appendEntry`：

```typescript
private _appendEntry(entry: SessionEntry): void {
	this.fileEntries.push(entry);
	this.byId.set(entry.id, entry);
	this.leafId = entry.id;
	this._persist(entry);
}
```

新条目在创建时已经把 `parentId` 设成当前 leaf。所以只要把 leaf 移到更早的条目，下一个条目就会成为它的又一个子节点，树上就多出一个分支。改变 leaf 的方法有三个：

- `branch(id)`（把 leaf 移到指定条目）：只改指针，不写任何东西。
- `resetLeaf()`（把 leaf 设为空）：下一个条目会成为新的根，`parentId` 为 `null`。用于重新编辑第一条用户消息。
- `branchWithSummary(id, summary)`（移动 leaf，并在新位置挂一条摘要）：写一条 `branch_summary`，`parentId` 是目标位置，`fromId` 记下离开时的 leaf。

所以一个文件里可以有多个根。`getTree()`（把条目组装成树）把 `parentId` 为空的条目、父节点找不到的孤儿条目都当作根返回；同一父节点下的子节点按时间戳排序。

### 3.2 leaf 不写进文件

leaf 只存在内存里。重新打开会话时，`_buildIndex`（按文件重建索引）把 leaf 设成文件的最后一个条目。探针确认：调用 `branch()` 把 leaf 移回第一条用户消息、之后不再写任何条目，重新打开文件后，leaf 仍是最后一行。

这意味着"上次停在哪条分支"取决于最后写入的是哪条分支，而不是最后看的是哪条。`/tree` 只移动 leaf、没有接着发消息就退出，下次打开会回到最后写入的那条分支。

### 3.3 /tree、/fork、/clone

| 操作 | 文件 | 新的 leaf | 选中消息的原文 |
|---|---|---|---|
| `/tree` | 同一个文件 | 选中的条目；选中 user 或 custom 消息时是它的父节点 | 回到输入框 |
| `/fork` | 新文件 | 所选 user 消息的父节点 | 回到输入框 |
| `/clone` | 新文件 | 当前 leaf | — |

- `/tree` 由 `navigateTree`（在树上换位置）完成。交互界面先问要不要给离开的分支生成摘要（设置 `branchSummary.skipPrompt` 可以跳过这一问），`navigateTree` 再按情况调用 `branchWithSummary`、`resetLeaf` 或 `branch`；用户给了标签时，还会追加一条 `label`。
- `/fork` 选的若是第一条用户消息（没有父节点），直接新建一个空会话，`parentSession` 指回原文件。
- `/fork` 和 `/clone` 都调用 `createBranchedSession`，区别只在截到哪里：`/fork` 截到所选 user 消息之前，`/clone` 截到当前 leaf（含）。

`createBranchedSession`（把一条路径截取成新会话）做了几件事：

- 跳过路径上的 `label` 条目，把它们的子节点改接到前一个条目上
- 按保留下来的条目重建标签，追加在路径末尾
- 修正压缩条目的 `firstKeptEntryId`
- 新文件头的 `parentSession` 指回原文件

原文件不受影响，别的分支也不会带到新文件里。如果截取出的路径里还没有对话消息，新文件同样等到第一条对话消息出现时才建。

另外还有两种跨文件的复制：

- `SessionManager.forkFrom`（命令行 `--fork`）：把另一个会话的全部条目复制到当前目录的新会话里，整棵树都带过来
- `/import`：直接把 JSONL 文件复制进会话目录，再打开

---

## 4 · 从树到上下文

![图 4 · 投影](/images/harness/pi/ch06/fig4-projection.png)

`buildSessionContext()`（算出发给模型的消息和设置）背后是三步，图 4 用一个例子画出每一步的结果。

### 4.1 第一步：走出当前分支

`buildSessionPath`（从 leaf 走到根）沿 `parentId` 往上走，再反转成根在前的顺序。leaf 为空时得到空列表。传入的 leaf 找不到时，从最后一个条目开始走。父节点缺失时，路径在缺口处停下。

### 4.2 第二步：按最新的压缩条目裁剪

`buildContextEntries`（得到参与上下文的条目）只看路径上**最后一个**压缩条目：

- 输出以这个压缩条目开头
- 接着是它之前、从 `firstKeptEntryId` 开始的条目，其中 role 为 `system` 的消息被丢掉
- 然后是它之后的所有条目

`firstKeptEntryId` 等于压缩条目自己的 id 时，表示之前的条目一条不留。

丢掉 system 消息是因为压缩条目自带一份检查点。`appendCompaction`（写入压缩条目）在写入时调用 pi-ai 的 `getCurrentSystemMessage`（把一串 system 消息重放成一条完整的），把当时的完整提示词和工具集存进 `systemMessage` 字段。投影时，这份检查点代替了被裁掉的、以及保留区间里的所有 system 消息。

探针按图 4 的压缩部分写了条目：一条带 `tools` 段的 system 消息，后面跟一条只补 `skills` 段的 system 补丁，再压缩。投影出来的第一条消息是 `system`，`sections` 同时含 `tools` 和 `skills`，后面依次是 `compactionSummary`、`user u2`、`assistant a2`、`user u3`。

更早的压缩条目如果落在保留区间里，也会出现在条目列表中，但投影时不产生任何消息。只有排在第一位的那个压缩条目贡献检查点和摘要。

### 4.3 第三步：逐条转消息，套用 context_edit

`buildSessionProjection`（投影出带来源的消息）先收集上一步列表里的 `context_edit`，按 `targetId` 建一张表。同一个目标有多条编辑时，后写入的覆盖先写入的。然后把每个条目转成消息：

- `replacement` 为 `null`：目标条目不产生消息
- 否则只替换内容，角色和其他字段保持原样。assistant 和 toolResult 收到字符串时，自动包成一个文本块

返回值除了消息列表，还有 `entries`：每个条目和它投影出的消息一一对应。`AgentSession` 用它记住"这条消息来自哪个条目"，后面要编辑某条消息时就知道目标是谁。

`appendContextEdit`（写入一条编辑）在写入前检查三件事：

- 目标存在
- 目标在当前分支上
- 目标是 user、assistant、toolResult 消息或 `custom_message`

编辑也是树上的节点，所以它**只对所在的分支生效**。探针在一个分支上去掉了第一条用户消息，再用 `branchWithSummary` 回到更早的位置，投影里那条用户消息又出现了：新分支的路径上没有那条编辑。

### 4.4 模型和思考级别

同一趟遍历还从**完整路径**（不经过压缩裁剪）里算出两项设置：

- 思考级别：最后一条 `thinking_level_change`，没有就是 `off`
- 模型：最后一条 `model_change`，或者最后一条 assistant 消息上记录的 `provider` 和 `model`，两者取后出现的

这两项只在打开会话时使用（5.4 节）。

---

## 5 · AgentSession：会话树是上下文的唯一来源

### 5.1 每次请求前重新投影

`_installAgentRequestProjection`（安装请求前的投影）包住 `agent.prepareRequest`：

```typescript
const projection = this.sessionManager.buildSessionProjection();
const canonicalContext = {
	...request.context,
	messages: projection.messages,
	// Messages declare the provider-visible loadout; context.tools keeps executable implementations.
	tools: this.agent.state.tools.slice(),
};
```

每次请求模型之前，循环里的消息都被换成从当前分支新投影出来的那份。模型和思考级别则取 `agent.state` 里当前选中的值，不从树上算。

投影之后还有几步，所以模型实际收到的和投影结果不完全相同：

1. `transformContext` 依次经过三层：扩展的上下文事件、去掉工具的 `prepareLoadout` 要求隐藏的声明（第五章）、强制系统提示词。扩展的上下文事件分两段：先是 `context`，处理函数拿到的是去掉 system 消息的副本，返回后 pi 把 system 消息放回原位；然后是 `context_with_system`，能看到完整的消息列表。
2. `convertToLlm` 把 4 种自定义消息转成 user 消息（第二章）。
3. pi-ai 的协议实现按供应商能力处理 system 消息（第五章）。

这几步都只影响这一次请求，不写回会话。

### 5.2 agent.state.messages 是副本

运行过程中，`Agent` 自己也把每条 `message_end` 的消息推进 `agent.state.messages`（第三章），所以这里有两份消息：会话树和 `Agent` 的状态。以会话树为准，`agent.state.messages` 在这些时机被 `_refreshFinalizedContext`（用投影覆盖 `agent.state.messages`）覆盖：

- 打开会话时，`createAgentSession` 用 `buildSessionContext().messages` 作为 `Agent` 的初始消息
- 写入 `bashExecution` 或扩展消息之后
- 压缩之后、`/tree` 切换分支之后
- 扩展在回合边界提交条目之后（7.2 节）
- 隐藏失败的尝试之后（5.3 节）

SDK 文档也写明了这一点：直接给 `session.agent.state.messages` 赋值，不会替换持久化的上下文。

一次普通的运行结束时，不会额外刷新。不过下一次请求前，`prepareRequest` 照样会重新投影，所以模型看到的始终是会话树里的内容。

### 5.3 失败的尝试被编辑掉，而不是删掉

供应商返回可重试的错误时（过载、限流之类），出错的 assistant 消息已经在 `message_end` 时写进了会话。`_omitRecoveryAttempt`（隐藏一次失败的尝试）给这条消息追加一条 `replacement: null` 的 `context_edit`，再刷新状态。上下文超长、需要压缩后再试时也这样处理，这时连同这条回复产生的工具结果一起隐藏。

结果是：

- 文件里保留着失败的记录
- 界面、导出和用量统计照常能看到这次失败
- 后续请求里不再出现这次失败

用户主动中止的回复、不可重试的错误，以及重试次数用完后的最后一次失败，都不走这条路，会留在上下文里。

### 5.4 打开会话和切换分支

`createAgentSession` 打开已有会话时：

- **模型**：调用方没有指定模型时，从当前分支恢复。`getBranchSelection`（取分支上最后一次选择的模型）沿分支往回找，取最后出现的 `model_change` 或 assistant 消息上记录的模型；之前选的是虚拟模型时，以 `model_change` 为准。恢复不了，比如找不到这个模型或没有凭据，就按设置里的默认值选，并给出 `modelFallbackMessage`。
- **思考级别**：从 `thinking_level_change` 恢复。老会话里没有这类条目时，补写一条。
- **新会话**：先写一条 `thinking_level_change`，选到了模型还会写一条 `model_change`。这就是探针文件第 2、3 行的来源。

`/tree` 切换分支时，只重新投影消息、从投影里的 system 消息恢复工具集，模型和思考级别保持当前的选择，不切回那条分支当时用的。

---

## 6 · 会话从哪里被操作

所有入口最终都落到 `SessionManager` 的方法上：

| 入口 | 操作 | 落到 |
|---|---|---|
| 命令行 | `--continue` / `-c` | `continueRecent`：打开最近修改的会话，没有就新建 |
| | `--resume` / `-r` | 列出会话让用户选，再 `open` |
| | `--session <路径或 id 前缀>` | 当前项目里找到就 `open`；在别的项目里找到，先问要不要复制过来（`forkFrom`） |
| | `--session-id <id>` | 按精确 id 打开，不存在就用这个 id 新建 |
| | `--fork <路径或 id 前缀>` | `forkFrom`：复制成当前目录下的新会话 |
| | `--no-session` | `inMemory`：不写文件 |
| | `--session-dir <目录>` | 会话目录；也可以用环境变量 `PI_CODING_AGENT_SESSION_DIR` 或设置项 `sessionDir` |
| | `--name` / `-n` | `appendSessionInfo` |
| 斜杠命令 | `/tree`、`/fork`、`/clone` | 见 3.3 节 |
| | `/resume`、`/new` | 换一个 `SessionManager`，重建整个运行时 |
| | `/name` | `appendSessionInfo` |
| | `/export`、`/share` | 导出。JSONL 只导出当前分支，HTML 导出整棵树 |
| SDK | `createAgentSession({ sessionManager })` | 不传时默认 `SessionManager.create` |
| 扩展 | `pi.appendEntry`、`pi.setLabel`、`pi.setSessionName` | 写 `custom`、`label`、`session_info` 条目 |
| | `ctx.sessionManager` | 类型是 `ReadonlySessionManager`，只暴露读取方法 |
| | `turn_end`、`agent_before_settle` 的返回值 | 提交条目草稿（7.2 节） |
| | `session_before_switch` / `_fork` / `_compact` / `_tree` | 可以取消切换、复制、压缩和分支导航。`_compact` 和 `_tree` 还能代替默认流程提供摘要 |

几点补充：

- `ReadonlySessionManager` 只在类型上只读，运行时拿到的就是同一个 `SessionManager` 对象。扩展要写会话，应该通过上面列出的 API。
- 会话列表（`SessionManager.list`）里的名字、消息数、第一条消息，统计的是整个文件，不区分分支。
- RPC 模式提供 `get_entries`、`get_tree`、`fork`、`clone`、`switch_session`、`new_session` 等命令，但没有直接的分支导航命令。

---

## 7 · 用法

### 7.1 用 SDK 选择会话的保存方式

`examples/sdk/11-sessions.ts` 里的写法：

```typescript
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

// In-memory (no persistence)
const { session: inMemory } = await createAgentSession({
	sessionManager: SessionManager.inMemory(),
});
console.log("In-memory session:", inMemory.sessionFile ?? "(none)");
inMemory.dispose();

// New persistent session
const { session: newSession } = await createAgentSession({
	sessionManager: SessionManager.create(process.cwd()),
});
console.log("New session file:", newSession.sessionFile);
newSession.dispose();

// Continue most recent session (or create new if none)
const { session: continued, modelFallbackMessage } = await createAgentSession({
	sessionManager: SessionManager.continueRecent(process.cwd()),
});
if (modelFallbackMessage) console.log("Note:", modelFallbackMessage);
console.log("Continued session:", continued.sessionFile);
continued.dispose();
```

`SessionManager.inMemory` 还可以接收一组现成的条目，用来从外部恢复历史。

### 7.2 扩展：只给模型看最近一轮的 read 输出

`turn_end` 和 `agent_before_settle` 是两个可以写会话的回合边界。

**工作方式**：

- 处理函数收到 `event.entries`（前面的扩展已经提出的条目草稿），返回新的 `entries` 就替换这个列表。返回 `continue: true` 要求再请求一次模型；预览出的上下文不能继续时（比如最后一条是 assistant 回复、又没有排队的消息），会报告扩展错误。
- 草稿只有 4 种：`custom`、`custom_message`、`context_edit`、`compaction`。

**提交方式**：

1. `AgentSession` 用当前分支建一个内存里的临时会话，把草稿写进去，预览出新的上下文。
2. 草稿不合法（比如编辑的目标不在当前分支上）时，这一批全部作废，并报告扩展错误。
3. 合法的草稿才写进真正的会话，然后刷新 `agent.state.messages`。

下面这个扩展在每一轮结束时，把上一轮 `read` 工具的输出从后续上下文里换成一句提示，只留最近一轮的：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
	let previousReads: string[] = [];

	pi.on("turn_end", (event, ctx) => {
		const onBranch = new Set(ctx.sessionManager.getBranch().map((entry) => entry.id));
		const edits = previousReads
			.filter((id) => onBranch.has(id))
			.map((targetId) => ({
				type: "context_edit" as const,
				targetId,
				replacement: { content: "(earlier read output omitted; read the file again if needed)" },
			}));
		previousReads = event.toolResultEntryIds.filter((id) => {
			const entry = ctx.sessionManager.getEntry(id);
			return entry?.type === "message" && entry.message.role === "toolResult" && entry.message.toolName === "read";
		});
		if (edits.length === 0) return;
		return { entries: [...event.entries, ...edits] };
	});
}
```

这段代码用 `tsc --strict` 检查过类型。探针用 faux 供应商跑了它的 JavaScript 等价版本：模型先读 `a.txt`，再读 `b.txt`，最后回答。

- 第三次请求里，`a.txt` 的结果已经换成提示文字，`b.txt` 的结果还是原文。
- 会话文件里两次 `read` 的原始输出都在，后面各跟一条 `context_edit`。

过滤 `onBranch` 是因为用户可能在两轮之间用 `/tree` 换了分支，编辑不在当前分支上的条目会让整批草稿作废。`toolResultEntryIds` 只包含找得到条目的工具结果，不一定和 `toolResults` 一一对齐，所以这里按条目 id 回查工具名，而不是按下标配对。

### 7.3 离线读一个会话

`SessionManager.open` 可以直接打开任意一个会话文件。注意打开不是只读的：版本低于 3 会迁移并重写，0 字节的文件会补上文件头，缺换行的最后一行会补上换行。下面三个方法拿到的是 pi 下一次请求会用的上下文、整棵树和当前分支：

- `buildSessionContext()`（要发给模型的消息和设置）
- `getTree()`（整棵树）
- `getBranch()`（当前分支上的条目）

```typescript
import { SessionManager } from "@earendil-works/pi-coding-agent";

const sm = SessionManager.open("/path/to/session.jsonl");
const { messages, model, thinkingLevel } = sm.buildSessionContext();
console.log(model, thinkingLevel, messages.map((m) => m.role));
```

打开文件后，leaf 是最后一行（3.2 节）。

---

## 8 · 三个可以带走的方法

1. **历史只追加，位置用指针表示**。条目一旦写下就不再改动，分支靠移动 leaf 产生。写盘只需要追加一行，写到一半的最后一行在下次加载时会被跳过；换个分支也不会丢失任何东西。
2. **上下文从历史投影出来，不单独维护**。每次请求前都从当前分支重新算出消息，压缩、编辑、分支切换都只是改变投影的输入。内存里的消息列表只是投影的副本，两者不一致时以历史为准。
3. **修改也写成条目**。`context_edit` 和压缩条目都是追加进去的。它们只对所在的分支生效，原始内容一直留在文件里，界面、导出和用量统计看到的仍是真实发生过的事。

---

## 9 · 关键数字

| 项 | 数量 |
|---|---|
| `session-manager.ts` | 2013 行 |
| 条目类型 | 11 种（另有 1 种文件头） |
| 会产生消息的条目类型 | 4 种 |
| 当前会话版本 | 3 |
| 条目 id | 8 位十六进制，最多重试 100 次 |
| 会话 id | UUIDv7 |
| 读取文件头的上限 | 1 MB |
| `turn_end` 草稿类型 | 4 种 |
| 选择或命名会话的命令行选项（不含 `--export`） | 8 个 |
| `session-manager.ts` 的提交（北京时间 2026-08-01 至 2026-10-01，不含合并提交） | 14 次 |

---

## 10 · 术语表

| 术语 | 含义 | 别和它混淆 |
|---|---|---|
| **条目**（entry） | 会话文件里除文件头以外的一行，有 `id`、`parentId`、`timestamp` | 消息：只有 `message` 条目直接装着消息 |
| **leaf** | 当前位置，新条目挂在它下面 | 文件最后一行：移动 leaf 之后、再写入之前两者不同；重新打开时 leaf 总是最后一行 |
| **当前分支** | 从 leaf 沿 `parentId` 走到根经过的条目 | `getEntries()`：整个文件的所有条目 |
| **投影** | 从当前分支算出发给模型的消息 | `transformContext`：投影之后、只影响一次请求的改写 |
| **`context_edit`** | 追加的一条编辑，替换或去掉早先条目的内容 | 删除条目：会话里没有删除操作 |
| **检查点** | 压缩条目里存的完整 system 消息 | 系统提示词补丁：普通 system 消息只写变化的部分 |
| **`custom`** / **`custom_message`** | 扩展的状态 / 扩展插入的消息 | 前者模型看不到，后者看得到 |
| **`parentSession`** | 文件头里指向来源会话文件的路径 | `parentId`：条目之间的父子关系 |

---

## 11 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 条目类型、文件头 | `packages/coding-agent/src/core/session-manager.ts` → `SessionEntry`、`SessionHeader` |
| 写盘 | `session-manager.ts` → `_appendEntry`、`_persist`、`_rewriteFile`、`_hasConversation` |
| 树和分支 | `session-manager.ts` → `branch`、`resetLeaf`、`branchWithSummary`、`getTree`、`createBranchedSession` |
| 投影 | `session-manager.ts` → `buildSessionPath`、`buildContextEntries`、`buildSessionProjection`、`sessionEntryToContextMessages` |
| 迁移 | `session-manager.ts` → `migrateV1ToV2`、`migrateV2ToV3` |
| system 消息的重放 | `packages/ai/src/utils/transcript.ts` → `getCurrentSystemMessage` |
| 消息落盘 | `packages/coding-agent/src/core/agent-session.ts` → `_handleAgentEvent` |
| 请求前投影 | `agent-session.ts` → `_installAgentRequestProjection`、`_refreshFinalizedContext` |
| 失败的尝试 | `agent-session.ts` → `_omitRecoveryAttempt` |
| 回合边界草稿 | `agent-session.ts` → `_applyBoundaryDrafts`、`_buildBoundaryContext`；`packages/coding-agent/src/core/extensions/runner.ts` → `emitBoundary`；`extensions/types.ts` → `SessionBoundaryDraft` |
| /tree | `agent-session.ts` → `navigateTree` |
| /fork、/clone、/import | `packages/coding-agent/src/core/agent-session-runtime.ts` → `fork`、`importFromJsonl` |
| 打开会话时恢复模型 | `packages/coding-agent/src/core/sdk.ts` → `createAgentSession` |
| 自定义消息转换 | `packages/coding-agent/src/core/messages.ts` → `convertToLlm` |
| 命令行选项 | `packages/coding-agent/src/cli/args.ts`；`packages/coding-agent/src/main.ts` → `resolveSessionPath` |
| 导出 | `packages/coding-agent/src/core/session-export.ts`；`packages/coding-agent/src/core/export-html/index.ts` |
| 示例 | `packages/coding-agent/examples/sdk/11-sessions.ts`、`13-session-runtime.ts` |
| 官方说明 | `packages/coding-agent/docs/session-format.md`、`docs/sessions.md`、`docs/sdk.md` |

**下一章**：上下文压缩。看 pi 什么时候决定压缩、怎么挑出要总结的消息、摘要怎么生成，以及 `/tree` 的分支摘要和它的异同。
