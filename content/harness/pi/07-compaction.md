---
title: "Pi 源码分析 07 · 上下文压缩：压缩条目是怎么产生的"
date: 2026-10-04T00:30:00+08:00
description: "接着第六章往下讲：coding-agent 在一次运行里的哪几个时机检查上下文、阈值怎么算，切点怎样从最新的消息往回选，摘要请求由什么组成、增量摘要和文件列表怎么带下去，失败、中止和重试会发出哪些事件，扩展能在哪里取消或接管压缩，以及 /tree 的分支摘要和压缩有什么异同。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.1-1-g4c6fb7cfe（2026-10-03），commit `4c6fb7cfe`。这个提交在 v1.0.1 之上只加了 CHANGELOG 的空标题，代码与 v1.0.1 相同。文中所有行为、数字和代码均以该版本为准。

第六章讲了压缩条目写进会话后怎样参与投影，本章讲压缩条目是怎么产生的：pi 什么时候决定压缩，切在哪里，摘要怎么生成，失败了怎么办，扩展能在哪里插手，以及 `/tree` 的分支摘要和它有什么不同。

<!--more-->

本章要点：

- **一个阈值，四个检查点**：上下文超过 `contextWindow − reserveTokens`（默认留 16384）就压缩。一次 prompt 里有四个地方会检查：发送新提示词之前、虚拟模型路由之后、两轮之间、运行结束之后。上下文溢出由运行结束后的检查处理，做法是压缩后重试一次；发送前的补查也会识别溢出，但只压缩、不重试。
- **从新往旧数，够了就切**：从最新的条目往回累加估算的 token，累加到 `keepRecentTokens`（默认 20000）就停下，切点只能落在 user、assistant 这类消息上，不会落在工具结果上。切点落在一个回合中间时，回合的前半段单独总结。
- **摘要是一次独立的请求**：一条 system 消息加一条 user 消息，对话被序列化成纯文本放进 `<conversation>`，不带工具，不写提示词缓存。上一次的摘要放进 `<previous-summary>`，换成"更新摘要"的指令。读过和改过的文件列表附在摘要末尾，并跨压缩累积。
- **失败不写压缩条目**：摘要请求按 `retry` 设置重试。失败、被取消或被中止时，只发一个带原因的 `compaction_end`，不写压缩条目；两轮之间的压缩失败时，运行带着原来的上下文继续。
- **扩展有三种插手方式**：`session_before_compact` 可以取消或替换摘要；`ctx.compact()` 发起一次手动压缩；回合边界返回的 `compaction` 草稿直接写入条目，绕过整条流水线和所有压缩事件。

---

## 0 · 阅读说明

- 本章讲已发布的 `pi` 用的压缩实现：pi-coding-agent 的 `core/compaction/` 三个文件（`compaction.ts`、`branch-summarization.ts`、`utils.ts`）和 `AgentSession` 里接线的部分。
- 引用前几章的结论，不再重复：
  - 第二章 3.2 节：`compactionSummary`、`branchSummary` 两种消息怎样变成 user 消息。
  - 第三章：`prepareNextTurn`（下一轮开始前的钩子）、`prepareRequest`（每次请求前的钩子）在一轮里的位置。
  - 第四章 5.3 节：pi-ai 的三层重试，其中消息级的 `retryAssistantCall` 在本章用于摘要请求。
  - 第六章 4.2 节：投影怎样按最新的压缩条目裁剪、检查点 `systemMessage` 的作用；5.3 节：`_omitRecoveryAttempt` 怎样用 `context_edit` 隐藏失败的尝试。
- 文中的行为都用探针实际跑过。探针用 npm 上的 1.0.1，用 `createAgentSession` 加 pi-ai 的 faux 供应商（从 `@earendil-works/pi-ai/compat` 导入 `registerFauxProvider`、`fauxAssistantMessage`），把模型的 `contextWindow` 设成 8000，`reserveTokens` 设成 2000，`keepRecentTokens` 设成 600，阈值就是 6000，几轮对话就能稳定触发压缩。faux 按请求文本长度的 1/4 估算 usage，探针里的 token 数只用于比较大小。探针记录了每次发给模型的请求和压缩前后的会话文件，编号 P1–P7。
- 术语：
  - **切点**：保留区间的第一个条目，存在压缩条目的 `firstKeptEntryId` 里。
  - **保留区间**：切点及之后的条目，压缩后仍以原文进入上下文。
  - **回合前缀**：切点落在一个回合中间时，这个回合从 user 消息到切点之前的部分。
  - **检查点**：压缩条目里保存的完整 system 消息（第六章 4.2 节）。

---

## 1 · 什么时候压缩

![图 1 · 一次 prompt 里的四个自动检查点](/images/harness/pi/ch07/fig1-triggers.png)

### 1.1 阈值和设置

判断是否超过阈值只有一个函数：

```typescript
export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
	if (!settings.enabled) return false;
	return contextTokens > contextWindow - settings.reserveTokens;
}
```

三项设置都在 `settings.json` 的 `compaction` 下：

| 设置 | 默认值 | 作用 |
|---|---|---|
| `enabled` | `true` | 是否自动压缩；关掉后仍可以 `/compact` |
| `reserveTokens` | 16384 | 给回复留的空间；也决定摘要请求的输出上限（3.4 节） |
| `keepRecentTokens` | 20000 | 压缩后至少保留多少最近的上下文 |

`reserveTokens` 和 `keepRecentTokens` 可以按模型覆盖：`compaction.modelOverrides` 的键是精确的 `provider/modelId`，每一项分别按"模型覆盖 → 普通设置 → 内置默认值"取值。`enabled` 只有全局一份。以 200K 窗口的模型为例，默认阈值是 200000 − 16384 = 183616。

① 和 ④ 用产生那条回复的模型的 `contextWindow`；回复来自别的模型（用户中途换过模型）时，改用当前模型的窗口，而且不做溢出处理（1.4 节）。

### 1.2 四个检查点

图 1 左列是 `session.prompt()` 的执行顺序，右列是四个检查点：

| 检查点 | 位置 | 能触发的原因 | 探针 |
|---|---|---|---|
| ① 发送前补查 | `prompt()` 检查完模型和凭据、发出新提示词之前，用上一条 assistant 回复判断 | threshold、overflow（都不接着重试） | P4 |
| ② 路由之后 | `prepareRequest` 里，只对虚拟模型：路由到具体模型后，按那个模型的窗口判断 | threshold | 代码 |
| ③ 两轮之间 | `prepareNextTurnWithContext`，第 2 轮起每轮开始前，只对实体模型 | threshold | P2 |
| ④ 运行结束后 | `agent.prompt()` 返回后的 `_handlePostAgentRun`，用最后一条 assistant 回复判断 | overflow（可重试）、overflow（不重试）、threshold | P1、P3 |

压缩事件的 `reason` 只有三个值：`manual`、`threshold`、`overflow`。图 1 里的四个检查点决定的是"什么时候看一眼"，`reason` 说明"为什么压缩"。

几个细节：

- **第 1 轮之前没有检查**。`agentLoop` 只在已经完成一轮之后才调用 `prepareNextTurn`。实体模型在第 1 轮之前只有 ① 一次检查，用的是上一条回复的 usage，不算新提示词本身的长度。
- **③ 在工具结果写进会话树之后、下一次模型回复之前**。探针 P2 让模型连续读三个大文件，时间线节选如下：

  ```text
  event turn_end
  event compaction_start reason=threshold
  REQUEST #5 [summary] msgs=system,user maxTokens=1600
  REQUEST #6 [summary] msgs=system,user maxTokens=1000
  event compaction_end reason=threshold aborted=false willRetry=false tokensBefore=7524 …
  event turn_start
  REQUEST #7 msgs=system,user,assistant,toolResult
  ```

  压缩发生在两轮之间，下一次请求直接用压缩后的上下文。准备工作可能很慢，所以 `agentLoop` 在 `prepareNextTurn` 返回后会再取一次插队消息（第三章）。
- **② 只对虚拟模型**。实体模型的 `prepareRequest` 在投影之后直接返回。虚拟模型要等路由之后才知道这次请求落到哪个具体模型，所以在这里按那个模型的窗口再判断一次，压缩后重新投影，这次请求就用压缩后的上下文。
- **① 能补上运行结束时漏掉的情况**。运行结束时，用户中止的回复不参与判断；发送下一条提示词之前会把它也算进去。探针 P4 里，一条被中止的回复 usage 是 7963，运行结束时没有压缩，下一次 `prompt()` 在发出新消息之前先压缩，然后才写入新的 user 消息。
- **④ 压缩完如果有排队的消息**，会再调一次 `agent.continue()` 把它们送出去。

### 1.3 用什么数 token

两类检查点用不同的计数方式：

- **③ 和 ②** 用 `estimateProjectedContextTokens`（估算当前投影的大小）：取投影里最后一条有效 assistant 回复的 usage，加上它之后各条消息的估算。如果这条回复之后会话里出现过 `context_edit` 或压缩条目，usage 已经不能代表当前上下文，就整份按估算重新数：当前完整的 system 消息加上其余所有消息。
- **① 和 ④** 在 `_checkCompaction` 里，按情况选一种：
  - 投影里有 `context_edit`：同上，用 `estimateProjectedContextTokens`
  - 回复出错，或 usage 全为 0：用 `estimateContextTokens`（最后一次有效 usage 加之后的估算）
  - 其他情况：直接用这条回复的 usage

usage 用 `calculateContextTokens` 换算：有 `totalTokens` 就用它，否则把 input、output、cacheRead、cacheWrite 相加。出错、被中止、全为 0 的 usage 都不算有效。

另有一道防线：如果用来判断的回复比最新的压缩条目还早，① 和 ④ 都直接跳过。压缩之后保留区间里的旧回复带着压缩前的大 usage，不跳过的话，压缩刚结束就会再压一次。

### 1.4 溢出：压缩后重试一次

④ 先判断溢出，再判断阈值。以下三种情况算溢出：

- **报错里写着超长**：pi-ai 的 `isContextOverflow` 用 25 个正则匹配各家的报错（`prompt is too long`、`exceeds the context window`、`maximum context length is N tokens` 等），先排除限流类报错，Cerebras 另有一条专门的规则。
- **静默溢出**：回复成功，但 `input + cacheRead` 超过了窗口。
- **可恢复的截断**：`stopReason` 是 `length`，而输出 token 还没达到模型原本的 `maxTokens`。说明是上下文挤占了输出空间，而不是回复本身太长。

处理方式取决于回复是否成功：

| 情况 | 做法 | `willRetry` |
|---|---|---|
| 报错或可恢复的截断 | 用 `context_edit` 隐藏失败的回复和它的工具结果（第六章 5.3 节），压缩，然后 `agent.continue()` 重试 | `true` |
| 成功回复但 usage 超过窗口 | 只压缩，不重试，保留这条回复 | `false` |

探针 P3 让模型在第三轮返回 `400 prompt is too long: 9100 tokens > 8000 maximum`，时间线节选如下：

```text
event message_end assistant() stop=error
event agent_end
event entry_appended context_edit
event compaction_start reason=overflow
REQUEST #4 [summary] msgs=system,user maxTokens=1600
event compaction_end reason=overflow aborted=false willRetry=true …
REQUEST #5 msgs=system,user,user,assistant,user
event message_end assistant(recovered) stop=stop
```

重试只有一次。`_overflowRecoveryAttempted` 标记在压缩前置上，等到出现新的 user 消息，或者某条回复的 `stopReason` 不是 `error` 也不是 `length` 时才清除。如果重试后再次溢出，pi 不再压缩，只发一个 `compaction_end`（扩展另外收到 `session_compact_failed`），`errorMessage` 是：

```text
Context overflow recovery failed after one compact-and-retry attempt. Try reducing context or switching to a larger-context model.
```

截断的情况对应另一句：`Truncated response recovery failed after one compact-and-retry attempt.`。探针 P3 第二组确认，这时事件流里只有 `compaction_end`，前面没有 `compaction_start`。

溢出不走普通的自动重试：`_isRetryableError` 先排除溢出，再判断是不是过载、限流这类错误。

### 1.5 手动压缩

手动入口有三个，最后都调用 `AgentSession.compact(customInstructions?)`：

- 交互界面的 `/compact [说明]`，空格后面的文字作为摘要的额外要求
- RPC 命令 `{ "type": "compact", "customInstructions": "…" }`
- 扩展的 `ctx.compact()`（5.3 节）

和自动压缩相比：

- 先调用 `abort()` 中止当前运行，再开始压缩。
- `reason` 是 `manual`，不重试，也不会接着运行。
- 没法压缩时抛错：最后一个条目已经是压缩条目时是 `Already compacted`，没有可总结的内容时是 `Nothing to compact (session too small)`。这两种情况也会发出 `compaction_start` 和带 `errorMessage` 的 `compaction_end`。
- 手动压缩进行中调用 `prompt()` 会被拒绝：`Cannot submit a prompt while compaction is in progress. Wait for compaction to finish and retry.`（探针 P4 确认）。这条检查只看手动压缩，自动压缩进行时不会因此拒绝。

自动压缩可以在设置里关掉（`compaction.enabled`，RPC 命令 `set_auto_compaction`）。关掉后四个检查点都不再压缩，溢出也不再压缩重试，手动压缩不受影响。

---

## 2 · 切在哪里

![图 2 · 切点的选择](/images/harness/pi/ch07/fig2-cutpoint.png)

### 2.1 估算 token

切点选择和大部分阈值判断都靠 `estimateTokens`（按字符数估算一条消息的 token）：字符数除以 4，向上取整。各种消息数的内容不同：

| 消息 | 计入的字符 |
|---|---|
| user、toolResult、custom | 文本；每张图片按 4800 个字符算 |
| assistant | 文本、思考内容、工具名加参数的 JSON |
| system | 文本、各个 `sections`、`toolsAdded` 的 JSON |
| bashExecution | 命令加输出 |
| branchSummary、compactionSummary | 摘要文本 |

源码注释说这种估算偏保守，倾向于高估。

### 2.2 选切点

`prepareCompaction`（算出这次压缩要用的全部输入）先对当前分支做一次投影（第六章 4.3 节），在投影出的条目上选切点。所以被 `context_edit` 隐藏的条目不占预算，被替换的条目按替换后的内容估算。

**范围**：投影以上一个压缩条目开头时，范围从它后面开始。上一次压缩保留的原文排在投影里压缩条目之后，所以它们这一次会进入总结。

**往回累加**：`findProjectedCutPoint` 从最后一个条目往前，逐条累加估算的 token。累加值第一次达到 `keepRecentTokens` 时停下，切点取停下位置或之后的第一个合法切点；之后没有合法切点，就取最后一个合法切点。

**合法切点**由消息的角色决定：

- 可以：user、assistant、bashExecution、custom、branchSummary（`compactionSummary` 也在名单里，但它只来自压缩条目，而压缩条目在这里一律跳过）
- 不可以：toolResult

工具结果总跟在发起调用的 assistant 后面，切点只能落在 assistant 上，所以一次工具调用和它的结果总在切点同一侧。图 2 是探针 P2 第一次压缩时的情形：

- 最后一条 `read b.txt` 的结果单独就有 2000 token，超过 600，在这里停下。
- 它后面没有合法切点，退回到发起这次调用的 assistant。
- 保留区间是 `assistant(read b)` 和它的结果，压缩条目的 `firstKeptEntryId` 指向这条 assistant。

**两处微调**：

- 切点前面紧挨着的、不产生消息的条目（比如 `label`、`model_change`）一起划进保留区间。
- 溢出恢复时，如果切点后面只剩下被隐藏的失败回复，切点再往后挪一格。这时最后那条用户输入也进入总结。探针 P3e 用一条约 5000 token 的 user 消息触发溢出：这条消息进了回合前缀摘要，重试请求里只剩 system 和摘要两条消息。

### 2.3 切在回合中间

切点不是一条"回合开始"的消息（user、bashExecution、custom 或 branchSummary）时，这个回合就被切成两半。`prepareCompaction` 往回找到这个回合开头的消息，把输入分成两份：

- `messagesToSummarize`（历史）：从范围起点到这个回合开头之前
- `turnPrefixMessages`（回合前缀）：从回合开头到切点之前

两份分别发请求总结（3.3 节），再拼成一份摘要。图 2 里，u2 这一回合被切开：u1 那一回合是历史，`u2 → read a.txt → 结果` 是回合前缀。

### 2.4 写进条目的几个数

- `tokensBefore`：压缩前投影的大小，用 `estimateProjectedContextTokens` 算出。P2 里是上一条回复的 usage 5524 加上之后那条工具结果的估算 2000，共 7524。
- 压缩完成后，`compaction_end` 的结果里还有 `estimatedTokensAfter`：对压缩后的投影逐条估算再相加。这个值只出现在事件里，不写进条目。

### 2.5 什么时候切不出来

`prepareCompaction` 在以下情况返回空，这次不压缩：

- 当前分支的最后一个条目已经是压缩条目
- 切点之前没有任何要总结的消息，比如整段上下文都在 `keepRecentTokens` 以内，或者上一次压缩之后只多了很少的内容

自动压缩遇到这种情况时什么事件都不发；手动压缩报 1.5 节的两种错误。

### 2.6 一条不留

压缩条目的 `firstKeptEntryId` 等于它自己的 id 时，压缩之前的条目一条也不保留（第六章 4.2 节）。pi 自己的压缩不会产生这种条目，因为切点总有一个。它只来自扩展在回合边界提交的 `compaction` 草稿：草稿的 `firstKeptEntryId` 可以是 `null`，`appendCompaction` 把 `null` 换成新条目自己的 id（5.4 节）。

另外，如果 `firstKeptEntryId` 指向一个不在当前分支上的条目，投影时找不到它，效果和一条不留一样。`appendCompaction` 不检查这个 id。

---

## 3 · 摘要怎么生成

![图 3 · 摘要请求的组成](/images/harness/pi/ch07/fig3-request.png)

### 3.1 请求的组成

一次摘要请求只有两条消息。system 消息是固定的 `SUMMARIZATION_SYSTEM_PROMPT`：

```text
You are a context summarization assistant. Your task is to read a conversation between a user and an AI assistant, then produce a structured summary following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary.
```

user 消息由三部分拼成：

1. **`<conversation>`**：要总结的消息先经过 `convertToLlm`（把自定义消息转成 user 消息），再由 `serializeConversation`（把消息写成纯文本）写成下面这种格式。写成纯文本是为了不让模型把它当成一段要接着往下说的对话。工具结果只保留前 2000 个字符，后面加一句 `[... N more characters truncated]`。
   ```text
   [User]: u1 please write notes.md

   [Assistant tool calls]: write(path="notes.md", content="hello")

   [Tool result]: Successfully wrote to notes.md

   [Assistant]: wrote notes
   ```
   以上取自探针 P2。另有 `[Assistant thinking]:` 一种，用来放思考内容。
2. **`<previous-summary>`**：上一次压缩的摘要，有才加（3.2 节）。
3. **指令**：第一次压缩用 `SUMMARIZATION_PROMPT`，原文是：

```text
The messages above are a conversation to summarize. Create a structured context checkpoint summary that another LLM will use to continue the work.

Use this EXACT format:

## Goal
[What is the user trying to accomplish? Can be multiple items if the session covers different tasks.]

## Constraints & Preferences
- [Any constraints, preferences, or requirements mentioned by user]
- [Or "(none)" if none were mentioned]

## Progress
### Done
- [x] [Completed tasks/changes]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues preventing progress, if any]

## Key Decisions
- **[Decision]**: [Brief rationale]

## Next Steps
1. [Ordered list of what should happen next]

## Critical Context
- [Any data, examples, or references needed to continue]
- [Or "(none)" if not applicable]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```

`/compact` 后面带的说明追加在指令末尾，形式是 `Additional focus: <说明>`。探针 P4 用 `/compact focus on the x characters` 的等价调用，请求末尾是 `…error messages.\n\nAdditional focus: focus on the x characters`。自动压缩没有这一段。

### 3.2 增量摘要

上一个压缩条目存在时，它的 `summary` 作为 `previousSummary` 带进这一次：放在 `<previous-summary>` 里，指令换成 `UPDATE_SUMMARIZATION_PROMPT`。开头两段是：

```text
The messages above are NEW conversation messages to incorporate into the existing summary provided in <previous-summary> tags.

Update the existing structured summary with new information. RULES:
- PRESERVE all existing information from the previous summary
- ADD new progress, decisions, and context from the new messages
- UPDATE the Progress section: move items from "In Progress" to "Done" when completed
- UPDATE "Next Steps" based on what was accomplished
- PRESERVE exact file paths, function names, and error messages
- If something is no longer relevant, you may remove it
```

后面是同样的六个小节，大多数小节的占位说明改成了"保留原有的，再补上新的"；In Progress、Blocked、Next Steps 改成按进展更新、已解决的删掉。

探针 P1 连续压缩了三次。第二次请求的 `<conversation>` 里只有 `u3` 和 `a3`，正是第一次压缩保留下来的那段原文，`<previous-summary>` 里是第一次的摘要。每次压缩都在上一次的摘要之上更新，而不是把整个历史重新总结一遍；上一次保留的原文在这一次进入总结。

### 3.3 回合前缀

切在回合中间时（2.3 节），历史和回合前缀各发一次请求：

- 历史部分照常用 3.1、3.2 节的格式。历史为空时不发请求，用上一次的摘要代替；连上一次的摘要也没有时，写 `No prior history.`。
- 回合前缀用另一种格式：`# Conversation` 下面是序列化的消息，`# Instructions` 下面是 `TURN_PREFIX_SUMMARIZATION_PROMPT`：

```text
The messages above are earlier context from an ongoing conversation. Later messages are stored separately and do not need to be reconstructed.

Create a concise checkpoint of the user's request and the progress shown above. This checkpoint will be placed before the later messages so the conversation can continue with the necessary context.

## Original Request
[What did the user ask for?]

## Progress So Far
- [Key decisions and work completed in these messages]

## Context Needed to Continue
- [Information from these messages needed to understand the later work]

Only summarize information explicitly present above. Do not infer or recreate later messages.
```

两份结果用一条分隔线拼起来：`<历史摘要>\n\n---\n\n**Turn Context (split turn):**\n\n<回合前缀摘要>`。探针 P2 的第一个压缩条目就是这种形状。

### 3.4 模型和请求选项

- **模型**：当前选中的模型。选中的是虚拟模型时，先以 `reason: "direct"` 路由一次，用路由到的具体模型。思考级别沿用当前设置，模型支持推理且级别不是 `off` 时才传。
- **输出上限**：`min(⌊0.8 × reserveTokens⌋, model.maxTokens)`；回合前缀用 0.5。探针里 `reserveTokens` 是 2000，两种请求的 `maxTokens` 分别是 1600 和 1000。
- **缓存**：`cacheRetention: "none"`，`sessionId` 换成一个新的 uuidv7。一次性的摘要请求不值得写缓存，也不应该占用会话的缓存路由。coding-agent 的缓存预热只认会话自己的 `sessionId`，所以摘要请求不会触发预热。
- **发送方式**：`completeSummarization`（所有摘要请求共用的发送函数）优先用传进来的 `streamFn` 发请求再取最终结果，没有时才用 pi-ai 的 `completeSimple`（发一次请求、等完整回复）。`AgentSession` 总是传 `Agent` 的 `streamFunction`，和普通请求走同一个发送函数；API key 和请求头由 `_getSummarizationRequestAuth`（为摘要请求取模型和凭据）单独取一次。
- **不带工具**：请求里没有工具声明。
- **不经过 `agentLoop`**：不发消息事件，不写进会话，扩展的 `context` 事件也不参与。探针 P7 跑了三次普通请求和一次摘要请求，`context` 事件只触发了三次。

### 3.5 检查回复

回复有以下任一情况都算失败，不写入任何条目：

- `stopReason` 是 `error`
- `stopReason` 是 `length`：摘要被截断，不完整
- 回复里有工具调用

失败时报错：前两种的错误信息以 `Summarization failed:` 或 `Turn prefix summarization failed:` 开头，含工具调用时是 `Summarization attempted to call a tool`（回合前缀请求是 `Turn prefix summarization attempted to call a tool`），交给第 4 节的流程处理。

### 3.6 文件列表

摘要文本后面还要附上两份文件列表，同样的内容也存进条目的 `details`：

- **从工具调用里收集**：`read`、`write`、`edit` 三个工具，取参数里的 `path`。codemode 脚本里嵌套的调用记在工具结果的 `nestedCalls` 上（第五章 5.4 节），也一起收集。工具名必须正好是这三个，其他工具（比如 `bash`）里碰到的文件不算。
- **从上一个压缩条目继承**：上一个条目的 `details.readFiles`、`details.modifiedFiles` 并进来。只继承 pi 自己生成的条目，`fromHook` 为真的不继承。
- **分类**：写过或改过的进 `modifiedFiles`；只读过、没改过的进 `readFiles`。两份都按字母排序。
- **附在摘要后面**：

  ```text
  <read-files>
  a.txt
  </read-files>

  <modified-files>
  notes.md
  </modified-files>
  ```

只统计要总结的消息，保留区间里的不算。探针 P2 里：

- 第一次压缩时，`b.txt` 的读取还在保留区间里，`readFiles` 只有 `a.txt`。
- 第二次压缩时，`b.txt` 进入总结，列表变成 `a.txt`、`b.txt`。`a.txt` 是从上一个条目继承来的。

### 3.7 写入

摘要生成后：

1. 再检查一次中止信号。
2. `appendCompaction` 写入条目：`summary`（摘要文本）、`firstKeptEntryId`（切点）、`tokensBefore`（压缩前的大小）、`details`（文件列表）、`usage`（摘要请求的用量）、`fromHook`（是否来自扩展），以及当时完整的 system 消息作为检查点（第六章 4.2 节）。
3. `_refreshFinalizedContext` 用新的投影覆盖 `agent.state.messages`。
4. 发 `session_compact` 给扩展，再发 `compaction_end` 给会话的订阅者。

下一次请求时，摘要变成一条 user 消息：开头是 `COMPACTION_SUMMARY_PREFIX`，即 `The conversation history before this point was compacted into the following summary:` 加上换行和 `<summary>`，然后是摘要，最后以 `</summary>` 结尾（第二章 3.2 节）。

---

## 4 · 失败、中止与重试

### 4.1 事件

| 事件 | 发给 | 字段 | 什么时候 |
|---|---|---|---|
| `compaction_start` | 会话订阅者 | `reason` | 开始压缩 |
| `compaction_end` | 会话订阅者 | `reason`、`result`、`aborted`、`willRetry`、`errorMessage?` | 结束，成功或失败 |
| `summarization_retry_scheduled` | 会话订阅者 | `attempt`、`maxAttempts`、`delayMs`、`errorMessage` | 摘要请求失败，排了一次重试 |
| `summarization_retry_attempt_start` | 会话订阅者 | `source`（`compaction` 时还有 `reason`） | 等待结束，重发 |
| `summarization_retry_finished` | 会话订阅者 | 无 | 重试序列结束 |
| `session_compact` | 扩展 | `compactionEntry`、`fromExtension`、`reason`、`willRetry` | 条目写入之后 |
| `session_compact_failed` | 扩展 | `reason`、`errorMessage?`、`aborted`、`willRetry`、`fromExtension` | 失败、取消或中止 |

`compaction_end` 的几种结果：

- **成功**：`result` 是 `CompactionResult`，包含 `summary`（摘要）、`firstKeptEntryId`（切点）、`tokensBefore`（压缩前大小）、`estimatedTokensAfter`（压缩后估算）、`usage`（用量）、`details`（文件列表），`aborted` 为假。
- **失败**：`result` 为空，`errorMessage` 按来源加前缀：自动的阈值压缩是 `Auto-compaction failed: …`，溢出恢复是 `Context overflow recovery failed: …`，手动是 `Compaction failed: …`。
- **取消或中止**：`aborted` 为真，没有 `errorMessage`。扩展取消（5.1 节）也算这一种。
- **`willRetry`**：溢出压缩时，失败的回复是报错或截断就为真，表示这条回复已被隐藏、按计划要重试；任何失败时都为假。在 ④ 里为真时，运行确实会重试；在 ① 里同样为真，但新提示词会取代这次重试，不会接着重发。

### 4.2 失败之后

- 自动压缩失败只发事件，不抛错。② 和 ③ 失败时，运行带着原来的上下文继续，下一个检查点还会再试。④ 发生在运行结束之后：溢出恢复失败时不会重试，被隐藏的失败回复也保持隐藏。
- 手动压缩失败在发完事件后把错误抛给调用者。交互界面的 `/compact` 吞掉这个错误，只靠事件显示。

### 4.3 摘要请求的重试

摘要请求用 pi-ai 的 `retryAssistantCall`（第四章 5.3 节的消息级重试），复用 coding-agent 的 `retry` 设置，和普通回复的自动重试共用同一套参数：

| 设置 | 默认值 |
|---|---|
| `retry.enabled` | `true` |
| `retry.maxRetries` | 3 |
| `retry.baseDelayMs` | 2000，每次翻倍 |
| `retry.maxAgentDelayMs` | 60000，单次等待的上限 |

只有过载、5xx 这类可重试的错误才会重试；被中止的回复和不可重试的错误立即返回。探针 P4 让第一次摘要请求返回 `529 overloaded_error: Overloaded`，时间线节选如下：

```text
event compaction_start reason=threshold
REQUEST #3 [summary]
event summarization_retry_scheduled {"attempt":1,"maxAttempts":2,"delayMs":10,"errorMessage":"529 overloaded_error: Overloaded"}
event summarization_retry_attempt_start {"source":"compaction","reason":"threshold"}
REQUEST #4 [summary]
event summarization_retry_finished {}
event compaction_end reason=threshold aborted=false willRetry=false …
```

（探针把 `maxRetries` 设成 2，`baseDelayMs` 设成 10。）

### 4.4 中止

- `abortCompaction()`（中止正在进行的压缩）同时中止手动和自动压缩。交互界面里，压缩进行中按 Esc 调用的就是它。
- `session.abort()`（中止当前运行）和 `dispose()`（释放会话）也会顺带调用它。
- 中止后，摘要请求以 `aborted` 结束，pi 在写入前检查到信号，按"取消"处理：不写条目，`compaction_end` 的 `aborted` 为真。

探针 P4 在一次手动压缩进行中调用 `abortCompaction()`：`compact()` 以 `Compaction cancelled` 拒绝，事件是 `compaction_end reason=manual aborted=true`。

---

## 5 · 扩展怎么插手

![图 4 · 扩展能插手的地方](/images/harness/pi/ch07/fig4-extensions.png)

手动压缩和自动压缩共用一条流水线：`prepareCompaction` → `session_before_compact` → 默认摘要 → `appendCompaction` → `session_compact`。扩展有三种方式插手。

### 5.1 session_before_compact：取消或接管

事件字段：

- `preparation`：`prepareCompaction` 的结果，包括 `firstKeptEntryId`（切点）、`messagesToSummarize`（历史）、`turnPrefixMessages`（回合前缀）、`isSplitTurn`（是否切在回合中间）、`tokensBefore`（压缩前大小）、`previousSummary`（上一份摘要）、`fileOps`（三个集合：read、written、edited）、`settings`（这次用的压缩设置）
- `branchEntries`：当前分支的全部条目
- `customInstructions`：`/compact` 的说明，自动压缩时为空
- `reason`、`willRetry`：同第 4 节
- `signal`：中止信号

返回值有两种用法：

- `{ cancel: true }`：取消这次压缩，按"取消"处理（4.1 节）。探针 P5 里，扩展取消了一次阈值压缩，事件是 `compaction_end reason=threshold aborted=true`，接着扩展收到 `session_compact_failed`，其中 `aborted` 为真。
- `{ compaction: { summary, firstKeptEntryId, tokensBefore, details?, usage? } }`：跳过 pi 自己的摘要，直接用这份内容写条目，条目的 `fromHook` 为真。

几条规则：

- **多个扩展都处理这个事件时**，第一个返回 `cancel` 的立刻生效；否则以最后一个返回值为真的为准。返回 `{}` 也算，会把前面扩展给的 `compaction` 覆盖掉，结果是 pi 自己生成摘要。处理函数抛出的异常只报告，不中断，继续交给下一个扩展。
- **pi 不校验扩展给的内容**：`firstKeptEntryId` 不检查是否在当前分支上（不在时效果是一条不留，2.6 节），`tokensBefore` 原样写入。最稳妥的做法是直接沿用 `preparation` 里的这两个值。
- **`fromHook` 的条目不继承文件列表**：下一次由 pi 生成摘要时，不会从它的 `details` 里取文件列表（3.6 节）。探针 P5 里，扩展接管了两次自动压缩之后，再用 `/compact` 让 pi 自己总结，新条目的 `details` 是两个空列表。扩展的摘要仍会作为 `previousSummary` 带进去，所以如果扩展在摘要文本里写了文件，这些信息还能保留下来。

### 5.2 session_compact 和 session_compact_failed

两者都只是通知，返回值不起作用：

- `session_compact` 带着刚写入的条目，`fromExtension` 表示摘要是否来自扩展。
- `session_compact_failed` 在压缩失败、被取消或被中止时发出。自动压缩在 `prepareCompaction` 返回空时没有开始，不发这个事件。

### 5.3 ctx.compact()：从扩展发起

`ctx.compact({ customInstructions, onComplete, onError })` 发起一次手动压缩，调用后立即返回，结果通过回调交回。它走的是 `AgentSession.compact()`，所以会先中止当前运行。示例扩展 `trigger-compact.ts` 在 `turn_end` 里检查用量，用量从 100000 token 以下涨过这条线的那一轮调用它。在 `turn_end` 里调用时，当前运行同样会被中止。

### 5.4 回合边界的 compaction 草稿

第六章 7.2 节讲过，`turn_end` 和 `agent_before_settle` 的处理函数可以返回条目草稿，其中一种是 `compaction`：

```typescript
{ type: "compaction", summary: string, firstKeptEntryId: string | null, details?: unknown, usage?: Usage }
```

它和上面的流水线完全分开：

- 草稿在内存里预览合法后，直接调用 `appendCompaction` 写入，`fromHook` 固定为真。
- `tokensBefore` 由 pi 用 `estimateProjectedContextTokens` 估算，草稿里不能指定。
- `firstKeptEntryId` 可以是 `null`，表示一条不留（2.6 节）。
- 不发 `compaction_start` / `compaction_end`，也不触发 `session_before_compact` / `session_compact`，只发 `entry_appended`。

探针 P5 在第二轮的 `turn_end` 返回一个 `firstKeptEntryId: null` 的草稿。写入的条目 `firstKeptEntryId` 等于自己的 id，下一次请求只剩 system、摘要和新的 user 消息三条；整个过程中只出现了一次 `entry_appended compaction`。

### 5.5 只影响一次请求的改写

`context` 和 `context_with_system` 事件（第六章 5.1 节）只改写一次请求要发出的消息，不写会话。压缩读的是会话树，所以这两个事件改不了压缩的输入，摘要请求本身也不经过它们（3.4 节）。

---

## 6 · 分支摘要：/tree 的另一种摘要

![图 5 · 两种摘要在会话树上的位置](/images/harness/pi/ch07/fig5-branch.png)

### 6.1 流程

在 `/tree` 里选一个位置时，交互界面会问 `Summarize branch?`，选项有 `No summary`、`Summarize`、`Summarize with custom prompt`。设置 `branchSummary.skipPrompt` 为真时不问，默认不总结。之后 `navigateTree` 依次：

1. **收集**：`collectEntriesForBranchSummary` 找出旧 leaf 和目标位置最深的共同祖先，收集从旧 leaf 往回走到共同祖先之前的全部条目。
2. **扩展**：发 `session_before_tree`。扩展可以取消导航，可以提供自己的摘要（用户选了总结时才用），也可以改写 `customInstructions`（额外说明）、`replaceInstructions`（是否整个替换默认指令）、`label`（给条目加的标签）。
3. **生成**：`generateBranchSummary` 发一次摘要请求。
4. **写入**：`branchWithSummary` 在新 leaf 的位置写一条 `branch_summary`，`fromId` 记下离开时的 leaf（第六章 3.1 节）。
5. **通知**：发 `session_tree`。

图 5 右侧是探针 P6：

- 选中的是 user 消息 `u2`，所以共同祖先就是 `u2`，收集的是它后面那几个条目，`u2` 本身不在其中。
- 新 leaf 是 `u2` 的父节点 `a1`，`u2` 的原文回到输入框。
- 摘要条目挂在 `a1` 下面，下一次请求是 `u1 · a1 · 分支摘要 · u3`。

### 6.2 和压缩的异同

相同点：

- system 消息同是 `SUMMARIZATION_SYSTEM_PROMPT`，对话同样用 `<conversation>` 包住序列化后的纯文本
- 同样走 `completeSummarization`：`cacheRetention: "none"`、新的 `sessionId`、不带工具，同样复用 `retry` 设置
- 文件列表的格式相同，同样从 assistant 的 `read`、`write`、`edit` 调用里收集
- 回复是 `error`、`length` 或含工具调用时同样不写入

不同点：

| | 压缩 | 分支摘要 |
|---|---|---|
| 总结什么 | 当前分支切点之前的部分 | 离开的那段分支 |
| 消息来源 | 投影：`context_edit` 生效 | 原始条目：`context_edit` 不生效 |
| 工具结果 | 序列化时截到 2000 字符 | 整条丢掉，只留 assistant 里的调用 |
| 预算 | 往回保留 `keepRecentTokens`，其余全部总结 | 从最新的往回装，最多装 `contextWindow − branchSummary.reserveTokens`（默认 16384） |
| 指令 | `SUMMARIZATION_PROMPT` 或 `UPDATE_…`，六个小节 | `BRANCH_SUMMARY_PROMPT`，五个小节，没有 Critical Context |
| 增量 | 有 `<previous-summary>` | 没有；路径上的旧摘要作为普通消息放进 `<conversation>` |
| 自定义说明 | 只能追加 | 可以追加，也可以 `replaceInstructions` 整个替换 |
| 思考级别 | 模型支持推理时沿用当前级别 | 不传 |
| 文件列表 | 也收集 codemode 的 `nestedCalls`；继承上一个压缩条目 | 工具结果整条丢掉，收不到 `nestedCalls`；只继承路径上 `fromHook` 为假的旧分支摘要 |
| 输出上限 | `min(⌊0.8 × reserveTokens⌋, model.maxTokens)` | `min(4096, model.maxTokens)` |
| 写在哪 | 当前 leaf 下 | 新 leaf（目标位置）下 |
| 事件 | `compaction_start` / `end` | 没有开始和结束事件；重试时同样发 `summarization_retry_*` |

几点补充：

- 探针 P6 在离开前给 `a2` 追加了一条 `context_edit`，把内容换成 `a2-EDITED`；分支摘要请求里出现的仍是原文 `a2-original`。
- 预算装不下时，从最新的往旧的装，装满即停。停下的那一条如果是旧的压缩条目或分支摘要，只要已装的还不到预算的 90%，就把它也装进去。
- 分支摘要的文本前面会加一段固定的开头：`The user explored a different conversation branch before returning here.\nSummary of that exploration:`。投影时它再被包一层：前面是 `BRANCH_SUMMARY_PREFIX`（`The following is a summary of a branch that this conversation came back from:` 加上换行和 `<summary>`），后面是 `</summary>`。

---

## 7 · 用法：一个不调用模型的压缩扩展

下面这个扩展接管自动压缩：不调用模型，把每次被总结掉的用户请求和文件操作记成一份清单，跨压缩累积。`/compact` 仍交给 pi 用模型总结。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

type Notes = { requests: string[]; readFiles: string[]; modifiedFiles: string[] };

const textOf = (content: unknown): string =>
	typeof content === "string"
		? content
		: Array.isArray(content)
			? content.map((block) => (block?.type === "text" ? block.text : "")).join("")
			: "";

export default function (pi: ExtensionAPI) {
	pi.on("session_before_compact", (event, ctx) => {
		const { preparation, reason, branchEntries } = event;
		// /compact keeps Pi's model-written summary; only automatic compaction is replaced.
		if (reason === "manual") return;

		// Carry the previous notes forward: a fromHook compaction is skipped by Pi's own file tracking.
		const previous = branchEntries.findLast((entry) => entry.type === "compaction");
		const notes: Notes = { requests: [], readFiles: [], modifiedFiles: [] };
		if (previous?.fromHook && previous.details) Object.assign(notes, previous.details as Notes);

		for (const message of [...preparation.messagesToSummarize, ...preparation.turnPrefixMessages]) {
			if (message.role === "user") notes.requests.push(textOf(message.content).slice(0, 200));
		}
		const modified = new Set([...notes.modifiedFiles, ...preparation.fileOps.edited, ...preparation.fileOps.written]);
		const read = new Set([...notes.readFiles, ...preparation.fileOps.read]);
		notes.modifiedFiles = [...modified].sort();
		notes.readFiles = [...read].filter((path) => !modified.has(path)).sort();

		const summary = [
			"## User requests so far",
			...notes.requests.map((request) => `- ${request}`),
			"## Files read",
			...notes.readFiles.map((path) => `- ${path}`),
			"## Files modified",
			...notes.modifiedFiles.map((path) => `- ${path}`),
		].join("\n");

		ctx.ui.notify(`compacted ${preparation.tokensBefore} tokens without a model call (${reason})`, "info");
		return {
			compaction: {
				summary,
				firstKeptEntryId: preparation.firstKeptEntryId,
				tokensBefore: preparation.tokensBefore,
				details: notes,
			},
		};
	});
}
```

这段代码对应前面的几条规则：

- **`firstKeptEntryId` 和 `tokensBefore` 直接用 `preparation` 里的**：pi 不校验扩展给的值（5.1 节），沿用它算好的切点最稳妥。
- **自己继承上一份清单**：`fromHook` 的条目不会被 pi 的文件追踪继承（3.6 节），所以扩展从 `branchEntries` 里找到上一个压缩条目，从它的 `details` 接着往下记。
- **`turnPrefixMessages` 也要算进去**：切在回合中间时，回合开头那条 user 消息在回合前缀里（2.3 节）。
- **`reason === "manual"` 时返回空**：不返回结果，pi 照常走默认流程。

这段代码用 `tsc --strict` 检查过类型（`--skipLibCheck`，target ES2023）。探针 P5 把它编译成 JavaScript 后，用 faux 供应商跑了十轮对话，第一轮读 `a.txt`，第三轮改 `b.txt`：

- 第五轮和第九轮之后各触发一次阈值压缩，扩展都接管了。两个条目的 `fromHook` 为真，第二个条目的清单列出了 u1 到 u8 八条请求，`Files read` 是 `a.txt`，`Files modified` 是 `b.txt`。
- 这两次压缩都没有发出摘要请求。
- 最后用 `/compact` 手动压缩时，扩展让出，pi 发了一次摘要请求，新条目 `fromHook` 为假，`details` 是两个空列表（5.1 节）。

---

## 8 · 三个可以带走的方法

1. **只在固定的位置检查**。上下文大小只在四个确定的时机检查：发送前、路由后、两轮之间、运行结束后。每个检查点都处在两次模型请求之间，这时上下文已经稳定，压缩完可以直接换掉下一次请求的输入，不用打断进行中的流。
2. **在投影上切，在原文上留**。切点在投影上选，所以编辑过、隐藏过的内容按它们在上下文里的样子计数。条目本身只追加不删，压缩失败时什么都不写，压缩成功也只是多了一个改变投影的条目。
3. **摘要是滚动的检查点**。每次只总结新增的部分，把上一份摘要作为输入交给模型更新；文件列表这类确定性的信息不交给模型去记，由代码从工具调用里收集、累积，再附在摘要后面。

---

## 9 · 关键数字

| 项 | 数量 |
|---|---|
| `compaction.ts` / `branch-summarization.ts` / `utils.ts` | 1119 / 382 / 163 行 |
| `reserveTokens` / `keepRecentTokens` 默认值 | 16384 / 20000 |
| `branchSummary.reserveTokens` 默认值 | 16384 |
| 压缩的 `reason` | 3 种（manual、threshold、overflow） |
| 自动检查点 | 4 个 |
| 溢出后压缩重试 | 最多 1 次 |
| 识别溢出报错的正则 | 25 个（另有 Cerebras 一条专门规则、3 个排除限流的正则） |
| token 估算 | 字符数 ÷ 4；图片按 4800 字符 |
| 摘要里工具结果保留的字符 | 2000 |
| 摘要输出上限 | 0.8 × reserveTokens；回合前缀 0.5 ×；分支摘要 4096 |
| 摘要请求重试 | 最多 3 次，2 秒起翻倍，单次最长 60 秒 |
| 压缩摘要的小节 / 分支摘要的小节 | 6 / 5 |
| `core/compaction/` 的提交（北京时间 2026-08-01 至 2026-10-01，不含合并提交） | 17 次 |

---

## 10 · 术语表

| 术语 | 含义 | 别和它混淆 |
|---|---|---|
| **阈值压缩**（threshold） | 上下文超过 `contextWindow − reserveTokens` 时的压缩 | 溢出压缩：供应商已经报错或截断之后的补救 |
| **溢出压缩**（overflow） | 回复报超长、被截断或 usage 超过窗口之后的压缩，报错和截断时会重试一次 | 普通的自动重试：只处理过载、限流这类错误 |
| **切点** | 保留区间的第一个条目，即 `firstKeptEntryId` | 回合开头：切点可以落在回合中间 |
| **回合前缀** | 切在回合中间时，这个回合在切点之前的部分，单独总结 | 历史：回合开头之前的全部内容 |
| **`previousSummary`** | 上一个压缩条目的摘要，作为增量摘要的输入 | 分支摘要：不走增量，旧摘要只是普通消息 |
| **`fromHook`** | 条目的摘要来自扩展 | `fromExtension`：`session_compact` 事件里的同一个意思 |
| **`tokensBefore`** | 压缩前投影的估算大小 | `estimatedTokensAfter`：只在事件里，不写进条目 |
| **一条不留**（retain-none） | `firstKeptEntryId` 等于压缩条目自己的 id | 投影找不到切点：效果相同，但通常是 id 写错了 |

---

## 11 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 阈值、token 估算 | `packages/coding-agent/src/core/compaction/compaction.ts` → `shouldCompact`、`estimateTokens`、`estimateContextTokens`、`estimateProjectedContextTokens` |
| 切点 | `compaction.ts` → `prepareCompaction`、`findProjectedCutPoint`（包里另外导出的 `findCutPoint` 在原始条目上切，`AgentSession` 不用它） |
| 提示词、摘要请求 | `compaction.ts` → `SUMMARIZATION_PROMPT`、`UPDATE_SUMMARIZATION_PROMPT`、`TURN_PREFIX_SUMMARIZATION_PROMPT`、`compact`、`completeSummarization` |
| 序列化、文件列表 | `packages/coding-agent/src/core/compaction/utils.ts` → `serializeConversation`、`extractFileOpsFromMessage`、`computeFileLists`、`SUMMARIZATION_SYSTEM_PROMPT` |
| 分支摘要 | `packages/coding-agent/src/core/compaction/branch-summarization.ts` |
| 四个检查点 | `packages/coding-agent/src/core/agent-session.ts` → `prompt`、`_installAgentRequestProjection`、`_compactBeforeNextAssistantResponse`、`_handlePostAgentRun`、`_checkCompaction` |
| 自动和手动压缩 | `agent-session.ts` → `_runAutoCompaction`、`compact`、`_runDefaultCompaction`、`_getSummarizationRequestAuth`、`abortCompaction` |
| 回合边界草稿 | `agent-session.ts` → `_applyBoundaryDrafts` |
| /tree | `agent-session.ts` → `navigateTree` |
| 设置 | `packages/coding-agent/src/core/settings-manager.ts` → `getCompactionSettings`、`getBranchSummarySettings`、`getRetrySettings` |
| 溢出识别、摘要重试 | `packages/ai/src/utils/overflow.ts` → `isContextOverflow`、`isRecoverableLength`；`packages/ai/src/utils/retry.ts` → `retryAssistantCall` |
| 扩展事件类型 | `packages/coding-agent/src/core/extensions/types.ts` → `SessionBeforeCompactEvent`、`SessionCompactEvent`、`SessionCompactFailedEvent`、`CompactionEntryDraft`、`SessionBeforeTreeEvent` |
| 示例 | `packages/coding-agent/examples/extensions/custom-compaction.ts`、`trigger-compact.ts` |
| 官方说明 | `packages/coding-agent/docs/compaction.md` |

**下一章**：上下文工程。看系统提示词由哪些部分拼成、上下文文件和技能怎样进入请求，以及扩展能在请求前改写哪些内容。
