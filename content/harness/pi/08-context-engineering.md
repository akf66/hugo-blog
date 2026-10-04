---
title: "Pi 源码分析 08 · 上下文工程：系统提示词和上下文是怎么拼出来的"
date: 2026-10-04T01:30:00+08:00
description: "接着第七章往下讲：每次请求前，coding-agent 怎样把默认文字、工具说明、AGENTS.md、技能索引和工作目录拼成按名字分段的系统提示词，上下文文件和技能从哪些目录发现，/skill 和模板怎样展开，提示词的变化怎样写成 system 补丁，扩展的 input、before_agent_start、context、context_with_system、before_provider_request 各能改什么，以及这些安排和提示词缓存、缓存预热的关系。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.1-2-g83692682f（2026-10-03），commit `83692682f`。这个提交在 v1.0.1 之上只改了 CHANGELOG 和 Nix 工作流，代码与 v1.0.1 相同。文中所有行为、数字和代码均以该版本为准。

前两章讲了消息怎么存进会话、怎么裁剪，本章讲每次请求前，系统提示词和上下文是怎么拼出来的。

<!--more-->

本章要点：

- **系统提示词是一组有名字的段**：`buildSystemPromptSections` 产出 `preamble`、`tools`、`rules`、`docs`、`addendum`、`project_context`、`skills`、`cwd` 八段，扩展还可以在末尾追加自己的段。除 `preamble` 外，每段包在同名的 XML 标签里。默认提示词里没有日期、操作系统和 git 信息。
- **上下文文件全加载，技能只给索引**：AGENTS.md 一类的文件从全局目录和从根到工作目录的每一层祖先收集，全文写进 `project_context`，没有大小上限，也不需要信任项目。技能只把名字、描述和文件路径写进 `skills`，模型要用时自己用 `read` 读。
- **提示词的变化写成 system 补丁**：每次 prompt 和每轮开始前，pi 用当前选项重新算一遍各段，与会话里重放出的段比较，把变了的段（删除的段写 `null`）写成一条 system 消息，存进会话。
- **扩展的改写分两类**：`input` 和 `before_agent_start` 的改动会写进会话；`context`、`context_with_system`、`before_provider_request` 只影响这一次请求。`before_agent_start` 返回的 `systemPrompt` 在整次运行里替换提示词全文，但不写进会话。
- **段的顺序服务于缓存**：稳定的段在前，扩展的段在后。支持对话中途 system 消息的模型，提示词变化以补丁的形式排在对话后面，开头的提示词和缓存前缀保持不变。缓存预热只对带 `promptCache` 的模型生效，默认只在一次运行进行中时预热。

---

## 0 · 阅读说明

- 本章只讲已发布的 `pi` 走的路径：pi-coding-agent 的 `AgentSession`、`system-prompt.ts`、资源加载和扩展运行器，以及 pi-ai 里与 system 消息、缓存相关的部分。
- 引用前几章的结论，不再重复：
  - 第三章：`prepareNextTurnWithContext`（每轮开始前的钩子）、`prepareRequest`（每次请求前的钩子）、`transformContext`（只改这一次请求的上下文）在一轮里的位置。
  - 第五章 3 节：system 消息上的 `sections`、`toolsAdded`、`toolsRemoved`，工具声明的三种发法；7.3 节的 `tool_search`。
  - 第六章 2.1 节：system 消息作为普通 `message` 条目存进会话；4.2 节：压缩条目里的检查点代替之前所有 system 消息；5.1 节：每次请求前从会话树重新投影。
  - 第七章 3.4 节：摘要请求用 `cacheRetention: "none"`，不触发缓存预热。
- 文中的行为都用探针实际跑过。探针用 npm 上的 1.0.1，用 `createAgentSession` 加 pi-ai 的 faux 供应商（从 `@earendil-works/pi-ai/compat` 导入 `registerFauxProvider`、`fauxAssistantMessage`），在临时目录里放几层 AGENTS.md、技能和提示词模板，把 `HOME` 也指向临时目录，记录每次发给模型的消息列表和会话文件。faux 的每个回复都用工厂函数在请求到达时才创建。faux 不触发 `before_provider_request`（`before_provider_headers` 和 `after_provider_response` 照常触发），所以涉及请求体的探针（P4）改用本地 mock 服务模拟 Anthropic 的流式接口。探针编号 P1–P8。
- 术语：
  - **段**：`SystemMessage.sections` 里的一项，键是段名，值是包好标签的文字。
  - **补丁**：对话中途的 system 消息，只带变了的段和工具增减。
  - **强制提示词**：`before_agent_start` 返回的 `systemPrompt`，请求时替换全部段。
  - **基础选项 / 运行级选项**：`_baseSystemPromptOptions`（由资源和激活工具算出）和 `_runSystemPromptOptions`（本次运行里扩展改过的那份）。

---

## 1 · 一次请求前的拼装流水线

![图 1 · 一次请求前的拼装流水线](/images/harness/pi/ch08/fig1-pipeline.png)

用户输入的一段文字，到供应商收到的请求体之间要经过 12 步。第 1–7 步发生在 `AgentSession.prompt()` 和循环的第一轮里，结果写进会话；第 8–12 步在每次请求前运行，只影响这一次请求。

| 步 | 发生了什么 | 扩展能做什么 |
|---|---|---|
| 01 | 以 `/` 开头、且名字是扩展注册的命令：直接执行，不发请求 | `pi.registerCommand` |
| 02 | `input` 事件，拿到展开之前的原文 | 改写文字和图片，或拦截 |
| 03 | `_expandSkillCommand`（展开 `/skill:name`），再 `expandPromptTemplate`（展开 `/模板名`） | — |
| 04 | `before_agent_start` 事件，拿到展开后的文字和一份提示词选项 | 改选项；追加一条 custom 消息；强制替换提示词 |
| 05 | 组装新消息：user 消息、排队的 nextTurn 消息、扩展的 custom 消息 | — |
| 06 | `_preparePromptAndToolLoadout`（按选项设好工具，算出提示词补丁） | — |
| 07 | 循环的 `declareToolChanges`（把工具增减写到同一条 system 消息上），随后 `message_end` 把这些消息写进会话 | — |
| 08 | `prepareRequest`：从会话树投影出这次请求的消息 | — |
| 09 | `transformContext`：`context` → `context_with_system` → 去掉隐藏的工具声明 → 套用强制提示词 | 改这次请求的消息 |
| 10 | `convertToLlm`（自定义消息转成 user），`streamFn` 带上 `sessionId` 并登记缓存预热 | — |
| 11 | 协议实现把 system 补丁渲染进对话或合并回开头，打缓存断点 | `before_provider_request` 改请求体 |
| 12 | 发出 HTTP 请求 | `before_provider_headers` 改请求头 |

同一次运行里，模型回复、工具执行完之后的每一轮，由 `prepareNextTurnWithContext` 重新算一次选项，回到第 06 步：有变化就再写一条补丁，然后照常走 08–12。

后面几节按这个顺序展开：第 2 节讲第 06 步算出的那组段，第 3 节讲段里的上下文文件和技能从哪来、第 03 步怎么展开，第 4 节讲补丁，第 5 节讲第 02、04、09、11 步的扩展事件，第 6 节讲第 10、11 步与缓存的关系。

---

## 2 · 系统提示词怎么拼

![图 2 · 系统提示词由哪些段组成](/images/harness/pi/ch08/fig2-sections.png)

### 2.1 输入：BuildSystemPromptOptions

拼提示词的函数都在 `core/system-prompt.ts`（216 行），输入是一个选项对象 `BuildSystemPromptOptions`：

- `cwd`（工作目录，唯一的必填项）
- `customPrompt`（自定义提示词，替换默认的开头）
- `appendSystemPrompt`（追加的文字）
- `contextFiles`（已读好的上下文文件，`{ path, content }` 数组）
- `skills`（已加载的技能）
- `selectedTools`（要写进提示词的工具名，默认 `["read", "bash", "edit", "write"]`）
- `toolSnippets`（工具名 → 一行说明）
- `toolGuidelines`（工具名 → 守则列表）
- `promptGuidelines`（额外的守则）
- `sections`（额外的段，段名 → 文字）
- `forceSystemPrompt`（强制提示词）

`normalizeBuildSystemPromptOptions`（补齐默认值，并把每个数组和对象复制一份）把它变成字段齐全的 `NormalizedBuildSystemPromptOptions`。扩展在 `before_agent_start` 里拿到、直接修改的就是这个对象。

`AgentSession` 的 `_rebuildSystemPrompt`（算出基础选项）负责填这些字段，它只算选项，不渲染提示词：

- `customPrompt`、`appendSystemPrompt`、`contextFiles`、`skills` 来自资源加载器（第 3 节）
- `selectedTools` 是当前激活的工具
- `toolSnippets` 和 `toolGuidelines` 来自每个工具定义的 `promptSnippet`、`promptGuidelines`；没有 `promptSnippet` 的工具、以及被 `prepareLoadout` 隐藏声明的工具不进列表
- `promptGuidelines`、`sections`、`forceSystemPrompt` 留空，只有扩展会填

它在这些时候运行：会话创建时、激活的工具变了（`setActiveToolsByName`、恢复会话时从记录里还原工具）、扩展在运行中注册了新工具、`/reload` 重建运行时之后、扩展通过 `resources_discover` 补充了技能或模板路径之后。

### 2.2 八个段

`buildSystemPromptSections` 按固定顺序产出这些段：

| 段 | 内容 | 什么时候有 |
|---|---|---|
| `preamble` | 默认是一句身份说明；有自定义提示词时就是那段文字 | 总有 |
| `tools` | 每个带说明的激活工具一行 `- 名字: 说明`，末尾一句固定的话 | 没有自定义提示词时 |
| `rules` | 一条按工具组合决定的文件操作建议、各工具的守则、扩展加的守则、两条固定守则 | 没有自定义提示词时 |
| `docs` | pi 自带的 README、docs、examples 的绝对路径和阅读指引 | 没有自定义提示词时 |
| `addendum` | `APPEND_SYSTEM.md` 或 `--append-system-prompt` 的文字 | 有追加文字时 |
| `project_context` | 所有上下文文件的全文 | 有上下文文件时 |
| `skills` | 技能索引 | 有可由模型调用的技能，且 `read` 或 `bash` 处于激活状态 |
| `cwd` | 工作目录的绝对路径，反斜杠换成 `/` | 总有 |

扩展通过 `sections` 加的段排在这八段之后。段名必须匹配 `^[a-z][a-z0-9_-]*$` 且不能是 `preamble`，否则直接抛错；值为空字符串的段被跳过。和内置段同名时，内容被替换，位置不变。

除 `preamble` 外，每段包在同名标签里，例如 `<cwd>\n…\n</cwd>`。源码注释说明了用意：模型可以按名字把后来的更新对应到这一段上。pi-ai 的 `getSystemMessageText`（把 system 消息渲染成文字）按顺序把各段用空行连起来，`buildSystemPrompt` 返回的字符串就是这样渲染出来的。

探针 P1 在默认设置下的第一条 system 消息（`<T>` 是临时目录）。`preamble` 原文：

```text
You are an expert coding assistant operating inside pi, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files.
```

`tools` 和 `rules`：

```text
<tools>
- read: Read file contents
- bash: Execute bash commands (ls, grep, find, etc.)
- edit: Make precise file edits with exact text replacement, including multiple disjoint edits in one call
- write: Create or overwrite files

In addition to the tools above, you may have access to other custom tools depending on the project.
</tools>

<rules>
- Use bash for file operations like ls, rg, find
- Use read to examine files instead of cat or sed.
- You can inspect PI_* environment variables for current model and session details.
- Use edit for precise changes (edits[].oldText must match exactly)
- When changing multiple separate locations in one file, use one edit call with multiple entries in edits[] instead of multiple edit calls
- Each edits[].oldText is matched against the original file, not after earlier edits are applied. Do not emit overlapping or nested edits. Merge nearby changes into one edit.
- Keep edits[].oldText as small as possible while still being unique in the file. Do not pad with large unchanged regions.
- Use write only for new files or complete rewrites.
- Be concise in your responses
- Show file paths clearly when working with files
</rules>
```

`rules` 的组成：

- 第一条只在激活了 `bash` 或 `powershell`、而 `grep`、`find`、`ls` 一个都没激活时出现，让模型用 shell 做文件操作。三种工具组合各有一句。
- 接着按工具顺序放各工具的 `promptGuidelines`。`PI_*` 环境变量那条来自 `bash`。
- 然后是扩展加的 `promptGuidelines`。
- 最后两条固定不变。
- 所有条目去掉首尾空白后去重。

`docs` 段开头一句是 `Pi documentation (read only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):`，后面列出 npm 包里 README、docs、examples 的绝对路径，以及"问到扩展就读 docs/extensions.md"这类主题对照表。它让模型在被问到 pi 本身时能找到文档，平时不必读。

默认设置、4 个默认工具、没有上下文文件和技能时，整段提示词渲染出来是 371 个英文词、约 2800 个字符（P1 的环境是 2803 个），字符数随 `docs` 和 `cwd` 里路径的长度变化。

### 2.3 自定义提示词和追加文字

整段换掉开头有三种来源，前一种存在时后面的不看：

1. 命令行 `--system-prompt <文字或文件路径>`
2. `<cwd>/.pi/SYSTEM.md`（项目被信任时）
3. `~/.pi/agent/SYSTEM.md`

`resolvePromptInput`（解析提示词参数）的规则是：参数是一个存在的文件路径就读文件，否则把参数本身当作文字。

追加文字的来源是 `--append-system-prompt`（可以写多次），没有这个参数时用 `<cwd>/.pi/APPEND_SYSTEM.md`（需信任）或 `~/.pi/agent/APPEND_SYSTEM.md`，只取找到的第一个。多段追加文字用空行连接，进 `addendum` 段。

探针 P6 用同一组文件（全局和项目各一份 `SYSTEM.md`、`APPEND_SYSTEM.md`，工作目录里一份 `AGENTS.md`，一个技能）跑了几种组合，只看第一条 system 消息：

```text
默认（项目已信任）      sections=[preamble,addendum,project_context,skills,cwd]  preamble="PROJECT SYSTEM.md persona"  addendum="project append"
项目未信任              同上                                                    preamble="GLOBAL SYSTEM.md persona"   addendum="global append"
--system-prompt 文字    同上                                                    preamble="You are a terse reviewer."
--system-prompt 文件    同上                                                    preamble="persona from a file path\n"
--append-system-prompt ×2                                                       addendum="first append\n\nsecond append"
--no-context-files      sections=[preamble,addendum,skills,cwd]
--no-skills             sections=[preamble,addendum,project_context,cwd]
--tools ls              sections=[preamble,addendum,project_context,cwd]        ← 没有 read 和 bash，技能段也没了
```

自定义提示词只替换 `preamble`，同时去掉 `tools`、`rules`、`docs` 三段；上下文文件、技能、工作目录照常保留。所以写 `SYSTEM.md` 时，工具清单和守则要自己写；工具和扩展提供的 `promptGuidelines` 只进 `rules`，有自定义提示词时都不会出现。

第四种来源是扩展的强制提示词，它替换的是全部段，第 4.4 节讲。

---

## 3 · 上下文文件、技能与提示词模板

![图 3 · 上下文文件与技能的发现顺序](/images/harness/pi/ch08/fig3-discovery.png)

### 3.1 上下文文件：全文进 project_context

`loadProjectContextFiles`（收集上下文文件）在每个目录里依次找这 5 个文件名，只取第一个存在的：

`AGENTS.override.md` → `AGENTS.md` → `AGENTS.MD` → `CLAUDE.md` → `CLAUDE.MD`

目录的顺序：

1. 全局目录 `~/.pi/agent/`
2. 从文件系统根一层层往下，直到工作目录。不在 git 根停下，工作目录的每一层祖先都看。

同一个路径只取一次。在嵌套于主仓库内部的 git worktree 里，worktree 根目录有自己的上下文文件时，主仓库根目录的同名文件被跳过，避免同一份约定加载两次。

探针 P1 的目录树和结果：

```text
<T>/agent/AGENTS.md                  global: answer in English
<T>/agent/CLAUDE.md                  （同目录有 AGENTS.md，不取）
<T>/ws/CLAUDE.md                     ws: uses pnpm
<T>/ws/proj/AGENTS.md                （同目录有 AGENTS.override.md，不取）
<T>/ws/proj/AGENTS.override.md       proj: override wins
<T>/ws/proj/app/CLAUDE.md            app: tests live in test/      ← 工作目录

context files: [<T>/agent/AGENTS.md, <T>/ws/CLAUDE.md, <T>/ws/proj/AGENTS.override.md, <T>/ws/proj/app/CLAUDE.md]
```

它们按这个顺序写进 `project_context` 段，原样放入全文：

```text
<project_context>
Project-specific instructions and guidelines:

<project_instructions path="<T>/agent/AGENTS.md">
global: answer in English
</project_instructions>

<project_instructions path="<T>/ws/CLAUDE.md">
ws: uses pnpm
</project_instructions>
…
</project_context>
```

几个容易忽略的地方：

- **没有大小上限**：文件多大就放多大，源码里没有截断。
- **不受项目信任控制**：P1 把项目设成未信任再跑，4 个文件照样全部加载。需要信任的只有 `.pi/` 下的 `SYSTEM.md`、`APPEND_SYSTEM.md`、技能、模板、扩展这一类资源。
- **只在加载资源时读**：探针 P7 在两次 prompt 之间把 `AGENTS.md` 从 `v1: use npm` 改成 `v2: use pnpm`，第二次请求里还是 v1；调用 `session.reload()`（交互界面里是 `/reload`）之后，下一次 prompt 前写进会话的是一条只含 `project_context` 段的补丁，内容是 v2。
- **`--no-context-files`**（SDK 里是 `noContextFiles: true`）：整段 `project_context` 都没有，全局的 `AGENTS.md` 也不加载。`SYSTEM.md`、技能不受影响。

### 3.2 技能：发现顺序

技能的候选路径由包管理器 `PackageManager.resolve` 收集，再按 `resourcePrecedenceRank`（资源优先级）排序，排名小的在前：

| 排名 | 来源 | 需要信任项目 |
|---|---|---|
| 0 | 项目 `settings.json` 里 `skills` 列出的路径 | 是 |
| 1 | `<cwd>/.pi/skills`；从工作目录往上每层的 `.agents/skills`（有 git 仓库时到 git 根为止，没有时一直到文件系统根） | 是 |
| 2 | 用户 `settings.json` 里 `skills` 列出的路径 | — |
| 3 | `~/.pi/agent/skills`、`~/.agents/skills` | — |
| 4 | pi 包提供的技能 | 项目 settings 里列的包需信任，用户 settings 里的不需要 |

命令行 `--skill <路径>` 和扩展在 `resources_discover` 事件里返回的路径追加在最后。`--no-skills` 关掉上表的来源，但命令行显式给出的技能照样加载：`--skill` 的路径，以及命令行 `-e` 指定的扩展包带的技能（P6 里 `--no-skills --skill <dir>` 的技能段只有那一个技能）。

目录里怎样算一个技能：

- 含 `SKILL.md` 的目录就是一个技能，不再往下找。
- `.pi/skills` 这类目录：顶层的 `.md` 文件也各算一个技能，子目录里只认 `SKILL.md`。`.agents/skills`：顶层的 `.md` 不算，子目录里任何带描述的 `.md` 都算。
- frontmatter 只读 `name`、`description`、`disable-model-invocation` 三个字段。`name` 缺省时用文件所在目录的名字（顶层 `.md` 文件就会叫 `skills`，两个这样的文件会重名）。
- 名字超过 64 个字符、含小写字母数字和连字符以外的字符、描述超过 1024 个字符，名字以连字符开头或结尾、含连续连字符，都只给警告，技能照样加载。被跳过的是：缺少 `description` 的、frontmatter 解析失败或文件读不出来的，以及通过符号链接重复指向同一个文件的。
- 同名技能先到先得，后来的记一条 `name "…" collision` 诊断。

探针 P1 在工作目录、它的 git 根、git 根的上一层、`~/.pi/agent/skills`、`~/.agents/skills` 各放了一个技能：

```text
项目已信任：proj-skill（.pi/skills）、team（git 根的 .agents/skills）、release、secret（~/.pi/agent/skills）、home-skill（~/.agents/skills）
项目未信任：release、secret、home-skill
```

P1 的 `ws/proj` 是 git 仓库，所以它上一层的 `above-git` 两种情况下都没有加载。`~/.agents/skills` 在未信任时照样加载。

### 3.3 技能怎样声明给模型：只给索引

`formatSkillsForPrompt`（把技能列表格式化成提示词）只写每个技能的名字、描述和文件路径。P1 的 `skills` 段（截取）：

```text
<skills>
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.

<available_skills>
  <skill>
    <name>release</name>
    <description>Cut a release of this repo</description>
    <location><T>/agent/skills/release/SKILL.md</location>
  </skill>
  …
</available_skills>
</skills>
```

- 技能正文不进提示词。模型判断任务和描述相符时，用工具读 `location` 指向的文件，读到的内容作为工具结果进入对话。
- 所以技能段要求能读文件的工具：`read` 激活时第二句写 "Use the read tool…"；只有 `bash` 时改成 "Use bash to load a skill's file…"；两者都没有，整段不出现。
- 设了 `disable-model-invocation: true` 的技能不进索引（P1 的 `secret`），只能由用户用 `/skill:secret` 调用。
- 描述里的 `&`、`<`、`'` 等字符做了 XML 转义。

这和第五章的 `tool_search` 是同一个思路：平时只放一份目录，用到时再加载。两者的代码没有关系。`tool_search` 加载的是工具声明，写进 system 消息的 `toolsAdded`；技能加载靠模型调用 `read`，内容是一条普通的工具结果。技能也不会被 `tool_search` 搜到。

### 3.4 /skill:name 和 /模板：发送前展开

第 03 步先展开技能命令，再展开模板，两者都在 `input` 事件之后。

**`/skill:name 参数`**：

- 按名字在已加载的技能里找，找不到就原样发出。
- 读 `SKILL.md`，去掉 frontmatter，包成一个 `<skill>` 块。
- 参数不做替换，空一行接在块后面。
- `disable-model-invocation` 不影响这个命令。

P1 发送 `/skill:secret ship it now`，模型收到的 user 消息是：

```text
<skill name="secret" location="<T>/agent/skills/secret/SKILL.md">
References are relative to <T>/agent/skills/secret.

Only via /skill:secret
</skill>

ship it now
```

**提示词模板**：

- 来自 `<cwd>/.pi/prompts`（需信任）、`~/.pi/agent/prompts`、settings 和 pi 包，只看目录顶层的 `.md` 文件。
- 文件名就是命令名。
- frontmatter 有 `description`（缺省时取正文第一个非空行，超过 60 个字符就截断并加 `...`）和 `argument-hint`。
- 参数按 shell 的引号规则切分，模板里可以用：
  - `$1`、`$2`…（第几个参数）
  - `$@` 或 `$ARGUMENTS`（全部参数）
  - `${2:-默认值}`（缺省值）
  - `${@:2}` 和 `${@:2:1}`（从第几个参数开始取几个）

P1 的模板 `review.md` 是 `Review $1 with focus on ${2:-correctness}. All args: $@`。发送 `/review "src/a b.ts" perf extra`，得到：

```text
Review src/a b.ts with focus on perf. All args: src/a b.ts perf extra
```

几点规则：

- 会话里存的是展开后的文字，不是 `/review …`。
- 名字没有对应的模板时原样发出（P3 的 `/missing-template arg` 就是这样进了对话）。
- 扩展命令优先于技能和模板。
- RPC 和 print 模式同样调用 `session.prompt()`，所以也会展开。
- 扩展的 `pi.sendUserMessage` 默认不展开。

---

## 4 · 提示词怎么变：system 补丁

![图 4 · 提示词变化怎样落成 system 补丁](/images/harness/pi/ch08/fig4-patches.png)

### 4.1 两个时机

提示词在两个地方重新计算，都调用 `_preparePromptAndToolLoadout`：

1. **每次 prompt**：`before_agent_start` 处理完之后，用它改过的选项计算。这份选项存为运行级选项 `_runSystemPromptOptions`，补丁插在本次新消息的最前面。
2. **同一次运行的后续每一轮**：`AgentSession` 包在 `prepareNextTurnWithContext` 外面的那层，取运行级选项，把 `selectedTools` 换成当前激活的工具，再算一次，补丁作为这一轮的新消息交给循环。

> ⚠️ 扩展用 `pi.sendMessage(message, { triggerTurn: true })` 在空闲时启动的运行不走 `prompt()`：不触发 `input` 和 `before_agent_start`，也不经过第一个时机，消息直接交给 `Agent`。会话里已经有提示词时，这次请求照常从会话树投影出提示词；在全新的会话上第一个动作就是它时，这次请求只带一条文字为空的 system 消息（只有工具声明），没有系统提示词，要等之后一次正常的 prompt 才把完整的段作为补丁写进来（第九章 3.3 节的探针 P6b）。需要提示词和 `before_agent_start` 的场景，用 `pi.sendUserMessage`。

`_preparePromptAndToolLoadout` 做三件事：

- 按 `selectedTools` 设好可执行的工具，这一步会运行各工具的 `prepareLoadout`（第五章）。
- 从工具说明里去掉被隐藏声明的工具。
- 用 `buildSystemPromptSections` 算出目标段，再与会话里重放出的当前段比较。

比较用 `diffSystemPromptSections`（逐段比较）：内容变了的段写进补丁，当前有、目标里没有的段写 `null`，什么都没变就不产生补丁。补丁是 `{ role: "system", content: "", sections: {…} }`，随后由循环的 `declareToolChanges` 把工具增减也写到这条消息上。所以一条 system 消息可以同时带段的变化和工具的变化。

第一次 prompt 时会话里还没有任何段，比较的对象是空集，补丁就是完整的提示词。第六章探针里"第一条 system 消息带全部段和 `toolsAdded`"就是这样来的。

运行结束时（`_runAgentPrompt` 的 `finally`）运行级选项被清空。下一次 prompt 又从基础选项开始，`before_agent_start` 要重新加一遍。扩展这次没有加的段，会被补丁以 `null` 删除。

### 4.2 探针 P2：三条补丁

P2 的扩展做了三件事：

- 注册一个带 `promptSnippet` 的工具 `enable_grep`，执行时调用 `pi.setActiveTools([...当前, "grep"])`。
- 在 `before_agent_start` 里，prompt 含 `#ticket` 时加一个 `ticket` 段和一条守则，并返回一条 custom 消息。
- prompt 含 `#force` 时返回 `systemPrompt`。

依次发送 `first #ticket`（模型先调 `enable_grep` 再回答）、`second plain`、`third #force`、`fourth plain`。会话文件：

```text
message system sections={preamble,tools,rules,docs,cwd,ticket} toolsAdded=[read,bash,edit,write,enable_grep]
message user("first #ticket")
custom_message ticket-note "Loaded ticket PI-42"
message assistant("enable_grep({})")
message toolResult("grep enabled")
message system sections={tools,rules} toolsAdded=[grep]
message assistant("done 1")
message system sections={rules,ticket:null}
message user("second plain")
message assistant("done 2")
message user("third #force")
message assistant("done 3")
message user("fourth plain")
message assistant("done 4")
```

- **第二条 system**：工具执行时改了激活集合，下一轮开始前重算。`tools` 段多了一行 `- grep: Search file contents for patterns (respects .gitignore)`；`rules` 段少了 `Use bash for file operations like ls, rg, find`，因为现在有了 `grep`。这条补丁和 `toolsAdded=[grep]` 在同一条消息上。
- **第三条 system**：第二次 prompt 时运行级选项已清空，扩展没加 `ticket`，于是 `ticket: null`；它加的守则也没了，`rules` 跟着变。
- **第三、四次 prompt 之间没有补丁**：强制提示词不改变结构化的段，见 4.4 节。

### 4.3 补丁写进会话

补丁和其他新消息一样，在循环发出 `message_end` 时由 `AgentSession` 写进会话文件，时间早于这次请求发出。之后它就是会话树上的普通节点：

- 投影时按顺序参与重放（第六章）。
- 压缩时被检查点代替（第六、七章）。
- 在 `/tree` 上切到别的分支时，那条分支上没有这条补丁，提示词自然回到那条分支的样子。

`session.systemPrompt` 和扩展的 `ctx.getSystemPrompt()` 返回的是"当前生效的提示词"：运行中用运行级选项渲染，运行结束后用基础选项渲染。P2 在第一次运行结束后读 `session.systemPrompt`，里面已经没有 `<ticket>`，而会话里最后一次写进去的段仍然包含它，要等下一次 prompt 的补丁才删掉。

### 4.4 强制提示词：替换请求，不写会话

`before_agent_start` 返回 `{ systemPrompt }` 时，`emitBeforeAgentStart` 把它记在选项的 `forceSystemPrompt` 上。`buildSystemPromptState` 遇到它就只返回这段文字，不带段。它的效果：

- **会话里不记**：补丁仍按结构化的段计算，所以 P2 的第三次 prompt 没有写任何 system 消息。
- **请求里替换**：`_installAgentForcedPromptProjection` 装在 `transformContext` 的最外层，把所有 system 消息合并成一条开头的 system 消息，内容是强制文字，工具取当前的完整集合。
- **整次运行有效**：探针 P2b 让强制提示词所在的运行调用一次工具，两次请求的开头都是 `FORCED PROMPT`；下一次 prompt 恢复成结构化的段。类型注释里写的是 "for this turn"，实际生效范围是整次运行，因为运行级选项要到运行结束才清空。
- **`context_with_system` 看不到它**：P2b 里三次 `context_with_system` 拿到的开头都是 `sections=[preamble,tools,rules,docs,cwd]`，强制文字在它们之后才套上。

同一次 prompt 里有多个扩展返回 `systemPrompt` 时，后面的覆盖前面的；后面的处理函数从 `event.systemPrompt` 读到的就是前面那段强制文字。

---

## 5 · 请求前的改写链

![图 5 · 扩展改写链](/images/harness/pi/ch08/fig5-chain.png)

### 5.1 五个事件

**`input`**

- 拿到：`text`、`images`、`source`（`interactive`、`rpc` 或 `extension`）、`streamingBehavior`。拿不到提示词和对话。
- 返回：`{ action: "transform", text, images? }` 改写；`{ action: "handled" }` 拦截，后面的扩展和整个 prompt 都不再执行；`continue` 或不返回表示不改。
- 多个扩展：依次执行，后一个拿到前一个改写后的文字。
- 结果：改写后的文字经过展开，成为写进会话的 user 消息。排队的 steer 和 follow-up 消息也经过 `input`。

**`before_agent_start`**

- 拿到：`prompt`（展开后的文字）、`images`、`systemPrompt`（按当前选项渲染的提示词，只读）、`systemPromptOptions`（可修改的选项对象）。
- 改法：直接修改 `systemPromptOptions` 的字段，例如 `sections`、`promptGuidelines`、`appendSystemPrompt`、`contextFiles`、`skills`、`selectedTools`。
- 返回：`{ message }` 追加一条 custom 消息，`{ systemPrompt }` 设强制提示词。
- 多个扩展：所有处理函数共用同一个选项对象，后面的能看到前面的修改（P3 里扩展 B 读到的 `sectionsSoFar` 已经有扩展 A 加的 `ext_a`，`event.systemPrompt` 里也有 `<ext_a>`）。
- 结果：
  - 段的变化按第 4 节写成补丁。
  - custom 消息写成 `custom_message` 条目，排在 user 消息之后，发给模型时转成 user 消息。
  - 改了 `selectedTools` 的，以修改为准；没改的，以当前激活的工具为准，处理函数里调用 `setActiveTools()` 的效果不会被冲掉。

**`context`**

- 拿到：这次请求的消息副本，**去掉了所有 system 消息**。
- 返回：`{ messages }`，或者直接修改 `event.messages`（包括替换其中的元素）。
- 多个扩展：依次接力。
- 结果：只用于这一次请求，不写会话（第六章 5.1 节）。
- 对话没改时，system 消息留在原位。对话改了时，`restoreSystemMessages`（把 system 状态放回去）把所有 system 消息重放成一条，放在最前面：中途的补丁合并进开头，这一次请求的开头提示词因此变化。
- 这样设计是为了让裁剪、开窗、从压缩摘要处切片这类改动不会把提示词和工具声明弄丢。

**`context_with_system`**

- 拿到：完整的消息列表，包括开头的 system 消息和中途的补丁。
- 返回：`{ messages }`，原样使用；也可以直接修改 `event.messages`，它就是要发出的那个数组。
- 多个扩展：依次接力。
- 结果：只影响这一次请求。
- 返回的列表开头不再是 system 消息时，运行器报一条错误（`Handler removed the leading system message; …`），但仍然使用这个列表，此时请求里没有提示词和初始工具声明。有强制提示词时例外：之后那一层会补回一条只含强制文字、不带工具声明的开头。

**`before_provider_request`**

- 拿到：`payload`，即协议实现拼好的供应商原生请求体。此时提示词已经渲染进去，例如 Anthropic 的 `system` 数组。
- 返回：返回值替换请求体；返回 `undefined` 表示不改。
- 多个扩展：依次接力。
- 结果：只影响这一次 HTTP 请求。

请求头另有 `before_provider_headers`：直接修改 `event.headers`，设为 `null` 表示删除这个头。

### 5.2 探针 P3：两个扩展接力

P3 装了两个扩展 A、B，各自在四个事件上记录收到的内容：

- A 在 `input` 里给以 `hi` 开头的文字加上 ` +A`，第一次 prompt 时还在 `before_agent_start` 里加一个 `ext_a` 段。
- B 在 `context_with_system` 里往末尾追加一条 user 消息 `[B: request-only reminder]`。

依次发送 `hi 1`、`hi 2`、`/noop`（A 返回 `handled`）、`/missing-template arg`：

```text
[A] input text="hi 1" source=interactive
[B] input text="hi 1 +A" source=interactive
[A] before_agent_start prompt="hi 1 +A" sectionsSoFar=[] systemPrompt has <ext_a>? false
[B] before_agent_start prompt="hi 1 +A" sectionsSoFar=[ext_a] systemPrompt has <ext_a>? true
[A] context      sees: user
[A] context_with_system sees: system{preamble,tools,rules,docs,cwd,ext_a} user
---- run 2
[A] context      sees: user assistant user
[A] context_with_system sees: system{preamble,tools,rules,docs,cwd,ext_a} user assistant system{ext_a} user
---- /noop
[A] input text="/noop" source=interactive          ← B 没有收到，也没有请求
```

第二次请求实际发出的消息，和会话文件对照：

```text
请求 #2                                        会话文件
system {preamble,tools,rules,docs,cwd,ext_a}   message system {preamble,tools,rules,docs,cwd,ext_a}
user "hi 1 +A"                                 message user "hi 1 +A"
assistant "r1"                                 message assistant "r1"
system {ext_a: null}                           message system {ext_a: null}
user "hi 2 +A"                                 message user "hi 2 +A"
user "[B: request-only reminder]"              （没有）
```

再加一个开关，让 A 在 `context` 里删掉所有 assistant 消息：第二次请求变成 `system{preamble,tools,rules,docs,cwd} user user [B 的提醒]`，那条 `{ext_a: null}` 补丁已经合并进开头，开头那条消息里没有 `ext_a` 段了。

### 5.3 选哪个事件

| 想做的事 | 用哪个 | 留在会话里吗 |
|---|---|---|
| 把用户的缩写展开、拦截某些输入 | `input` | 改后的文字留下 |
| 给提示词加一段随会话保存的上下文 | `before_agent_start` 改 `sections` | 补丁留下 |
| 给这次 prompt 附一条可见的说明 | `before_agent_start` 返回 `message` | `custom_message` 留下 |
| 每次请求临时裁剪对话 | `context` | 不留 |
| 临时调整提示词或在对话里插 system 消息 | `context_with_system` | 不留 |
| 改供应商参数（metadata、采样参数等） | `before_provider_request` | 不留 |

`input` 和 `before_agent_start` 的改动会被压缩、分支、恢复会话继承下去；`context`、`context_with_system`、`before_provider_request` 的改动每次请求都要重新做，压缩读的是会话树，也看不到它们（第七章 5.5 节）。

---

## 6 · 和提示词缓存的关系

### 6.1 段的顺序与补丁的位置

供应商的提示词缓存按前缀命中：请求开头有多长一段和上次相同，就有多少能读缓存。Pi 的安排是：

- **稳定的段在前**：`preamble`、`tools`、`rules`、`docs` 几乎不变；`addendum`、`project_context`、`skills` 只在 `/reload` 时变；`cwd` 在一个会话里不变；扩展的段排在最后。
- **变化以补丁的形式排在后面**：开头那条 system 消息写进会话之后就不再改动，之后的变化都是追加的补丁。

补丁最后能不能保住缓存，取决于协议实现怎么发它（第五章 3.3 节讲过工具声明的三种发法，段的补丁走同一套判断）：

- 模型声明了 `supportsMidConvoSystemMessages` 时，补丁由 `renderSystemMessageUpdate` 渲染成文字，作为对话中途的 system 消息发出，格式是 `Updated system prompt section "名字":\n\n<新内容>` 或 `Removed system prompt section "名字".`。
- 否则所有 system 消息合并回开头，开头的提示词跟着变，缓存从开头失效。

探针 P4 用 mock 的 Anthropic 接口跑两次 prompt，第一次 `before_agent_start` 加了 `ticket` 段，第二次没加：

```text
claude-fable-5（支持中途 system 消息）
  req1 system[0] cache_control=ephemeral len=3068
       tools=[read,bash,edit,write*,__pi_deferred_placeholder__(deferred)]
       user: text*:"first"
  req2 system[0] cache_control=ephemeral len=3068          ← 开头不变
       user: "first" | assistant: "one" | user: "second"
       system: text*:"Removed system prompt section \"ticket\"."

claude-haiku-4-5（不支持）
  req2 system[0] cache_control=ephemeral len=3035          ← 开头变短，缓存从这里失效
       user: "first" | assistant: "one" | user: text*:"second"
```

（`*` 表示带 `cache_control` 的块。）在 Anthropic 的实现里，中途的 system 消息挪到下一条 assistant 消息之前，所以补丁出现在 `user "second"` 后面。

内置模型目录里显式声明 `supportsMidConvoSystemMessages` 的有 85 个，分布在 Anthropic、OpenAI Responses 一系和部分走 OpenAI Completions 的模型上；这个标记只能在模型的 compat 里显式打开，没有自动判断。其中 12 个还能在对话中途改工具：第五章 3.3 节的 6 个 Anthropic 模型（原生增删），和 6 个走 OpenAI Completions 的 kimi-k3（只能新增）。用其他模型时，要尽量少让提示词变化，一次变化就是一次从开头开始的缓存失效。

### 6.2 缓存断点、cacheRetention 和 sessionId

**断点**：Anthropic 的协议实现打三个缓存断点：

- 整段提示词作为一个文本块，块上一个断点。没有按段分别打断点。
- 最后一个初始工具上一个断点（除非模型的 compat 把 `supportsCacheControlOnTools` 关掉，默认打开）。
- 最后一条 user 或 system 消息的最后一个块上一个断点。

用 OAuth 登录时，提示词前面多一个固定的身份块，两块各一个断点。

**`cacheRetention`**：coding-agent 没有这个设置项，正常请求不传它，由协议实现决定：

- 环境变量 `PI_CACHE_RETENTION=long` 时用 `long`，否则用 `short`。
- Anthropic 的 `long` 对应 `ttl: "1h"`（除非模型的 compat 把 `supportsLongCacheRetention` 关掉，默认打开）。P4 设了这个环境变量后，断点变成 `{"type":"ephemeral","ttl":"1h"}`。
- 显式传值的只有 `completeSummarization`（摘要请求共用的发送函数），传 `"none"`。压缩、分支摘要（第七章）和错误报告的会话摘要都经过它。

**`sessionId`**：`createAgentSession` 把会话 id 交给 `Agent`，每次请求都带上（P5 记录的三次请求都是会话自己的 id）：

- Anthropic 在 `cacheRetention` 不为 `none`、且模型的 compat 打开了 `sendSessionAffinityHeaders`（`baseUrl` 是 OpenRouter 时默认打开）时，用它设置会话亲和的请求头：OpenRouter 是 `x-session-id`，其他是 `x-session-affinity`。GitHub Copilot 和 OAuth 登录走另外的分支，不发这个头。
- OpenAI Responses 用它作 `prompt_cache_key`。
- faux 供应商按 `sessionId` 记住上一次的请求文本，按公共前缀估算 `cacheRead`。P5 第二次请求在开头之后多了一条 `{late}` 补丁，usage 是 `input=16 cacheRead=1454 cacheWrite=17`，第一次请求的 1454 个估算 token 全部命中。

### 6.3 缓存预热

`CacheWarmer`（缓存预热器）在缓存快过期时，用 `maxTokens: 1` 原样重发上一次请求，让缓存续期。

**什么时候启动**：`streamFn` 收到 `sessionId` 等于会话 id 的请求时，就把这次请求（模型、上下文、选项）交给预热器。压缩和分支摘要用的是新的 `sessionId`，不会启动它。错误报告的会话摘要用的是会话自己的 id，它会启动预热器，但因为带着 `cacheRetention: "none"`，预热器随即停止。

**什么时候不预热**：

- 设置 `cacheWarming` 为 `off`（只能写在全局设置里，源码注释说明的理由是每次续期都要花钱）。
- 模型没有 `promptCache`（各档缓存的有效期）。内置目录 1536 个对话模型里只有 16 个声明了它，全部是 `anthropic` 供应商的 Claude 模型，`short` 300 秒、`long` 3600 秒。
- 请求用了 `cacheRetention: "none"`。
- 有效期不超过 10 秒。
- 请求不能安全重放：开了思考的请求发给 Anthropic 按预算思考的模型（没有 `forceAdaptiveThinking`）时，`budget_tokens` 从 `max_tokens` 算出来，重放时这个值会变，缓存键也跟着变。
- 会话的模型换了，或者当前消息不再以那次请求的消息为前缀。

**什么时候发**：

- 在有效期的 90% 处发送，并且至少提前 10 秒。默认 300 秒的缓存在第 270 秒续期。
- 定时器晚到、迟到的时间超过原定提前量的一半（300 秒的缓存是 15 秒）时，放弃这一次，因为这时多半已经过期，发出去也是全价写缓存。
- 都从那次真实请求算起：运行进行中最多续到 60 分钟，进入空闲阶段后最多续到 30 分钟。

**值不值得发**：每次续期前算一笔账：

- `warmCost`：读一次缓存加 1 个输出 token 的价格。
- `missCost`：缓存丢了以后，下一次请求多付的钱，即写缓存（或全价输入）减去读缓存。
- 下一次真实请求在过期前到来的概率：运行进行中取 1，空闲时取 0.15（源码注释说这个常数来自他们自己的使用统计）。
- 期望节省 = 概率 × `missCost` − `warmCost`，不低于 0.05 美元才发。

以 `claude-opus-5`（每百万 token：读缓存 0.5 美元、写缓存 6.25 美元）、10 万 token 的提示词为例：`warmCost` ≈ 0.050，`missCost` = 0.625 − 0.05 = 0.575。运行进行中，期望节省 0.525，发；空闲时 0.15 × 0.575 − 0.050 ≈ 0.036，不发。

**三种模式**：设置 `cacheWarming` 取 `off`、`streaming`、`idle`，默认 `streaming`。

- `streaming`：运行一结束（`agent_settled`）就停。实际起作用的场景是一次运行里有很久没有新请求的时候，例如一个工具执行了五分钟。
- `idle`：运行结束后按空闲时的概率继续算账。

扩展可以在 `cache_warming_decision` 事件里看到 pi 的决定和这几个数，返回 `{ action: "warm" | "stop" }` 推翻它。

探针 P5 给 faux 模型加上 `promptCache: { short: 12 }` 和一组很高的价格，让续期在 2 秒后发生：

```text
idle 模式
  request #1: sessionId=session cacheRetention=(unset) maxTokens=-  msgs=system,user
  request #2: sessionId=session cacheRetention=(unset) maxTokens=-  msgs=system,user,assistant,system,user
  request #3: sessionId=session cacheRetention=(unset) maxTokens=1  msgs=system,user,assistant,system,user   ← 续期
  cache_warming_decision action=warm warmCost=0.4611 missCost=5.1302 p=0.15
  会话文件末尾：usage {"kind":"cache_warm","provider":"faux",…,"cacheRead":1470,…}
streaming 模式  运行结束时状态 {"state":"inactive","reason":"agent run settled"}，没有第三次请求
off 模式        {"state":"inactive","reason":"cache warming disabled"}
```

续期请求的消息和第二次请求完全一样，结果写成一条 `usage` 条目（`kind: "cache_warm"`），计入会话成本，不产生消息。faux 不执行 `maxTokens` 限制，所以这里的 output 不是 1。

---

## 7 · 用法：按条件注入一段 git 上下文

下面这个扩展在每次 prompt 前查一下当前分支和未提交的文件数，作为一个 `git` 段放进提示词：

```typescript
import { execFileSync } from "node:child_process";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

function git(cwd: string, args: string[]): string | undefined {
	try {
		return execFileSync("git", args, { cwd, encoding: "utf8", stdio: ["ignore", "pipe", "ignore"] }).trim();
	} catch {
		return undefined;
	}
}

export default function (pi: ExtensionAPI) {
	pi.on("before_agent_start", (event, ctx) => {
		const branch = git(ctx.cwd, ["branch", "--show-current"]);
		// Not a git repo: add nothing. A section from an earlier prompt is then removed by a null patch.
		if (!branch) return;
		const dirty = (git(ctx.cwd, ["status", "--porcelain"]) ?? "").split("\n").filter(Boolean).length;
		// Keep the text stable: every change becomes a new system patch in the session.
		event.systemPromptOptions.sections.git = [
			`Current branch: ${branch}`,
			dirty > 0 ? `Uncommitted changes in ${dirty} file(s)` : "Working tree clean",
		].join("\n");
	});
}
```

这段代码里的几个选择：

- **用 `sections` 而不是 `appendSystemPrompt`**：独立的段可以单独更新、单独删除，排在内置段之后，变化只影响这一段的补丁。
- **不用返回值**：直接改 `event.systemPromptOptions`，同一个 prompt 里的其他扩展能看到这一段。
- **不在仓库里就什么都不加**：运行级选项每次 prompt 都从基础选项重新开始，这一次不加，上一次加的段就会被 `null` 补丁删掉，不需要自己清理。
- **只放变化少的事实**：段的内容每变一次，会话里就多一条补丁，不支持中途 system 消息的模型就从开头失效一次缓存。所以这里只写"有几个文件没提交"，不写文件列表或时间。

这段代码在 `tsc --strict`（加 `skipLibCheck`）下通过。探针 P8 把它放进 `DefaultResourceLoader` 的 `extensionFactories`，再把加载器交给 `createAgentSession`，在一个刚初始化的 git 仓库里依次发五次 prompt：

```text
p1 clean main          → 第一条 system 消息末尾：<cwd>…</cwd> <git>Current branch: main / Working tree clean</git>
p2 nothing changed     → 没有补丁
p3 （新建 new.txt）     → patch: {"git":"<git>\nCurrent branch: main\nUncommitted changes in 1 file(s)\n</git>"}
p4 （git switch -c feature）→ patch: {"git":"<git>\nCurrent branch: feature\nUncommitted changes in 1 file(s)\n</git>"}
p5 （删掉 .git）        → patch: {"git":null}
```

会话文件里一共 4 条 system 消息：开头的完整提示词，加三条只含 `git` 段的补丁。没有 custom 消息，user 消息保持原样。恢复这个会话、或者压缩之后，检查点里的 `git` 段就是最后一次补丁后的值。

如果只想让模型在这一次请求里看到，不想留在会话里，可以改用 `context_with_system`，在开头那条 system 消息的 `sections` 里加一段。代价是每次请求都要重新加，而且开头的提示词每次都可能和上一次不同，前缀缓存从开头失效。

---

## 8 · 三个可以带走的方法

1. **把提示词拆成有名字的段**。每段有固定位置和标签，变化时只发变了的那一段，删除也有明确的表示。拼提示词的代码、扩展、会话记录、协议实现说的是同一份结构。
2. **提示词的变化作为对话的一部分记录下来**。开头写一次完整的，之后每次变化都是追加的补丁。重放会话就能还原任何时刻模型看到的提示词，也不需要修改已经发给模型的前缀。
3. **区分"写进会话"和"只改这一次"**。前者被压缩、分支、恢复继承，后者每次请求重做。扩展作者按需要选择事件，不必自己处理持久化。

---

## 9 · 关键数字

| 项 | 数量 |
|---|---|
| 内置段 | 8 个（另有扩展段，排在最后） |
| 自定义提示词去掉的段 | 3 个（`tools`、`rules`、`docs`） |
| 默认提示词（4 个默认工具，无上下文文件和技能） | 371 个英文词，约 2800 个字符 |
| 每个目录的上下文文件候选名 | 5 个，只取第一个 |
| 上下文文件大小上限 | 无 |
| 技能名最长 / 描述最长（超出只警告） | 64 / 1024 个字符 |
| 技能 frontmatter 读取的字段 | 3 个 |
| 请求前的扩展改写事件 | 5 个（另有 `before_provider_headers`） |
| 内置目录中声明 `promptCache` 的模型 | 16 个（全部是 `anthropic` 供应商），`short` 300 秒、`long` 3600 秒 |
| 缓存续期时机 | 有效期的 90%，至少提前 10 秒 |
| 续期的期望节省门槛 | 0.05 美元 |
| 空闲时的续用概率 | 0.15 |
| 续期上限 | 运行中 60 分钟，空闲 30 分钟 |
| 本章相关文件的提交（北京时间 2026-08-01 至 2026-10-01，不含合并提交） | `system-prompt.ts`、`resource-loader.ts`、`skills.ts`、`prompt-templates.ts`、`cache-warmer.ts` 合计 16 次 |

---

## 10 · 术语表

| 术语 | 含义 | 别和它混淆 |
|---|---|---|
| **段（section）** | 系统提示词里有名字的一部分，`SystemMessage.sections` 的一项 | 第七章摘要模板里的 `## Goal` 这类标题 |
| **补丁** | 对话中途的 system 消息，只带变了的段和工具增减 | 压缩条目里的检查点（完整的 system 消息） |
| **基础选项** | 由资源和激活工具算出的提示词选项，`_baseSystemPromptOptions` | 运行级选项：本次运行里扩展改过的那份 |
| **强制提示词** | `before_agent_start` 返回的 `systemPrompt`，请求时替换全部段 | 自定义提示词：`SYSTEM.md` / `--system-prompt`，只替换 `preamble` |
| **上下文文件** | AGENTS.md / CLAUDE.md 一类的文件，全文进 `project_context` | 技能：只进索引，用时再读 |
| **技能索引** | `skills` 段里的名字、描述、路径列表 | `/skill:name` 展开的 `<skill>` 块（全文，进 user 消息） |
| **缓存预热** | 快过期时用 `maxTokens: 1` 重发上一次请求 | `cacheRetention`：缓存要保留多久 |

---

## 11 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 段的组成与渲染 | `packages/coding-agent/src/core/system-prompt.ts` → `buildSystemPromptSections`、`buildSystemPromptState`、`diffSystemPromptSections`；`packages/ai/src/utils/text.ts` → `getSystemMessageText`、`renderSystemMessageUpdate` |
| 选项从哪来、何时重算 | `packages/coding-agent/src/core/agent-session.ts` → `_rebuildSystemPrompt`、`_preparePromptAndToolLoadout`、`_installAgentNextTurnRefresh`、`prompt` |
| 强制提示词 | `agent-session.ts` → `_installAgentForcedPromptProjection`；`core/extensions/runner.ts` → `emitBeforeAgentStart` |
| 上下文文件、SYSTEM.md | `packages/coding-agent/src/core/resource-loader.ts` → `loadContextFileFromDir`、`loadProjectContextFiles`、`discoverSystemPromptFile`、`resolvePromptInput` |
| 技能 | `packages/coding-agent/src/core/skills.ts` → `loadSkills`、`formatSkillsForPrompt`；`core/package-manager.ts` → `resourcePrecedenceRank`、`collectAncestorAgentsSkillDirs`；`agent-session.ts` → `_expandSkillCommand` |
| 提示词模板 | `packages/coding-agent/src/core/prompt-templates.ts` → `substituteArgs`、`expandPromptTemplate` |
| 扩展事件 | `packages/coding-agent/src/core/extensions/runner.ts` → `emitInput`、`emitBeforeAgentStart`、`emitContext`、`restoreSystemMessages`、`emitBeforeProviderRequest`；`core/extensions/types.ts` |
| 工具增减与补丁合在一条消息 | `packages/agent/src/agent-loop.ts` → `declareToolChanges` |
| 补丁怎样发给 Anthropic、缓存断点 | `packages/ai/src/api/anthropic-messages.ts` → `buildParams`、`convertMessages` |
| 缓存预热 | `packages/coding-agent/src/core/cache-warmer.ts`；`core/sdk.ts` 里 `streamFn` 的接线 |
| 官方说明 | `packages/coding-agent/docs/skills.md`、`prompt-templates.md`、`extensions.md`、`environment-variables.md` |

**下一章**：扩展与事件。看扩展从哪里加载、能注册哪些东西，以及本章之外的其他事件。
