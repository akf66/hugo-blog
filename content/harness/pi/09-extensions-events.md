---
title: "Pi 源码分析 09 · 扩展与事件：扩展从加载到派发的全过程"
date: 2026-10-04T19:30:00+08:00
description: "接着第八章往下讲：把扩展当成一个整体，看它从哪八类来源按什么顺序加载、TypeScript 怎样被 jiti 转译、怎样装进 AgentSession、/reload 和 /new 时会不会重新执行、ExtensionAPI 的 33 个成员各落到哪个子系统、41 个事件在一次 prompt 里的先后和多个扩展的返回值怎样合并、三种 ctx 有什么不同，以及处理函数出错时会怎样。"
tags:
  - Harness
  - Pi
  - 源码分析
---

> **版本基线**：`earendil-works/pi` v1.0.2-1-g200387122（2026-10-04），commit `200387122`。这个提交在 v1.0.2 之上只给各包的 CHANGELOG 加了 `[Unreleased]` 段，代码与 v1.0.2 相同。文中所有行为、数字和代码均以该版本为准。

前几章分别讲了扩展在工具、会话、压缩、请求前改写里的那几个插口；本章把扩展当成一个整体来讲，包括它从哪来、怎么装进会话、能注册什么、事件怎么派发、出错和重载时会怎样。

<!--more-->

本章要点：

- **八类来源，一个固定顺序**：命令行 `-e`、项目 settings、`<cwd>/.pi/extensions`、用户 settings、`~/.pi/agent/extensions`、pi 包、内置扩展、SDK 的 `extensionFactories`，按这个顺序加载，事件处理函数也按这个顺序执行。项目里的来源要先信任项目。TypeScript 由 jiti 转译后加载，不需要编译。
- **同名冲突在启动时就拦下**：两个扩展注册同名工具或参数时，第一个生效，并记一条加载错误；CLI 遇到任何加载错误都会退出。同名命令都保留，改名为 `name:1`、`name:2`。
- **运行器住在 AgentSession 里**：`AgentSession` 构造时建 `ExtensionRunner` 并接上动作方法，运行模式调用 `bindExtensions` 时才发 `session_start`。`/reload` 会重新导入模块、重跑工厂，旧的注册随旧运行器整体丢弃；`/new`、`/resume`、`/fork` 只重跑工厂，模块取自缓存。之前留下的 `pi` 和 `ctx` 一律失效。
- **33 个 API 成员、41 个事件**：注册类方法加载时就能调用，动作类方法要等绑定之后。41 个事件里 19 个会采用处理函数的返回值或修改，按接力、拦截即停、后者覆盖、先到先得、收集五种规则合并；其余 22 个只通知。
- **出错只报告，不中断**：处理函数抛错时报告给 `onError`，跳过它继续执行；`tool_call` 例外，抛错就阻止这次工具调用。扩展和 pi 跑在同一个进程里，没有权限边界，项目信任只决定加载不加载。

---

## 0 · 阅读说明

- 本章只讲已发布的 `pi` 走的路径：pi-coding-agent 的 `DefaultResourceLoader`（资源加载器）、`PackageManager`（包与路径解析）、扩展加载器、`ExtensionRunner`（扩展运行器）、`AgentSession` 和 `AgentSessionRuntime`（会话替换）。
- 引用前几章的结论，不再重复：
  - 第二章 2.1 节：扩展里 `import "@earendil-works/pi-agent-core"` 拿到的是宿主自带的那一份；第 4 节：`AgentSession` 怎样把扩展事件接到 `Agent` 的钩子上。
  - 第三章 6.2 节：`tool_call` 返回 `{ block, reason }` 拦截工具；`turn_end` 返回 `continue`。
  - 第五章 7 节：`registerTool` 之后工具怎样进注册表、`exposure` 的五种取值、MCP 工具的命名和暴露方式。
  - 第六章 6 节和 7.2 节：`pi.appendEntry`、`setLabel`、`setSessionName` 写的条目；`turn_end` 和 `agent_before_settle` 的条目草稿；`session_before_switch / _fork / _tree`。
  - 第七章 5 节：`session_before_compact`、`session_compact`、`session_compact_failed`、`ctx.compact()`。
  - 第八章 5 节：`input`、`before_agent_start`、`context`、`context_with_system`、`before_provider_request`、`before_provider_headers` 能改什么；6.3 节：`cache_warming_decision`。
- 文中的行为都用探针实际跑过。探针用 npm 上的 1.0.2，用 `createAgentSession` 加 pi-ai 的 faux 供应商（从 `@earendil-works/pi-ai/compat` 导入 `registerFauxProvider`、`fauxAssistantMessage`）。临时目录里放全局、项目两层扩展目录、settings 里列出的路径和两个本地 pi 包，`HOME` 也指向临时目录。faux 的每个回复都用工厂函数在请求到达时才创建。faux 不调用 `before_provider_request` 和 `provider_stream_event`，涉及它们的探针（P2b）改用本地 mock 服务模拟 Anthropic 的流式接口。涉及启动退出码和项目信任的探针（P1b、P1c）直接运行 npm 包里的 `pi` 命令。探针编号 P1–P7。
- 术语：
  - **工厂**：扩展模块默认导出的函数 `(pi: ExtensionAPI) => void | Promise<void>`，加载时调用一次。
  - **注册表**：每个扩展一个 `Extension` 对象，里面是它注册的处理函数、工具、命令、快捷键、参数、渲染器。
  - **运行器**：`ExtensionRunner`，持有本次加载的全部 `Extension`，负责派发事件、汇总注册。
  - **绑定**：运行模式调用 `session.bindExtensions()`，接上界面、命令上下文和错误监听，并发出 `session_start`。

---

## 1 · 扩展从哪来

![图 1 · 扩展从哪来：八类来源与加载顺序](/images/harness/pi/ch09/fig1-ext-sources.png)

### 1.1 八类来源和加载顺序

扩展的路径由 `PackageManager.resolve`（收集 settings、包和自动发现的路径）汇总，按 `resourcePrecedenceRank`（资源优先级）排序，再由 `DefaultResourceLoader.reload` 把命令行路径放在最前、SDK 工厂放在最后：

| 顺序 | 来源 | 需要信任项目 |
|---|---|---|
| 1 | 命令行 `-e` / `--extension`（文件、目录、`npm:`、`git:`） | — |
| 2 | 项目 `.pi/settings.json` 的 `extensions` 列出的路径 | 是 |
| 3 | `<cwd>/.pi/extensions/` 自动发现 | 是 |
| 4 | 用户 `~/.pi/agent/settings.json` 的 `extensions` 列出的路径 | — |
| 5 | `~/.pi/agent/extensions/` 自动发现 | — |
| 6 | pi 包：settings 里 `packages` 列出的包，项目的排在用户的前面 | 项目 settings 里的包需要 |
| 7 | 内置扩展 `builtin:llama.cpp`、`codemode`、`tool-search`、`mcp` | — |
| 8 | SDK 的 `extensionFactories`，路径记为 `<inline:N>` | — |

这个顺序就是之后每个事件处理函数的执行顺序；同一个扩展里的多个处理函数按注册顺序执行。探针 P1 在每一类来源各放一个扩展，每个扩展在工厂里和 `input` 事件里各记一行：

```text
项目已信任
  factory: cli-e → proj-setting → p-file → user-setting → g-dir → g-early → g-file → g-replace → proj-pkg → user-pkg → builtin:fake-builtin → <inline:SDK>
  input:   cli-e → proj-setting → p-file → user-setting → g-dir → g-early → g-file → g-replace → proj-pkg → user-pkg → <inline:SDK>
项目未信任
  factory: cli-e → user-setting → g-dir → g-early → g-file → g-replace → user-pkg → builtin:fake-builtin → <inline:SDK>
```

`g-*` 都在 `~/.pi/agent/extensions/` 里，同一个目录内按 `readdirSync` 返回的顺序，代码里没有再排序（macOS 的 APFS 上恰好是文件名顺序）。未信任时，项目 settings、`.pi/extensions` 和项目 settings 里的包都消失了，命令行、用户和 SDK 的扩展照常加载。

P1 用 SDK，信任事先给定。CLI 需要询问项目信任时会先跑一遍预信任加载（1.5 节），这时命令行、用户和 SDK 扩展的工厂比项目扩展先执行；加载完成后仍按上表重新排列，处理函数的顺序不变。

几点补充：

- **内置扩展只有 CLI 会装上**：`main.ts` 把 `builtInExtensions` 放在用户给的 `extensionFactories` 前面，带 `builtin: true` 的项被资源加载器当作 `builtin:<名字>` 路径处理。直接用 SDK 的 `createAgentSession` 时没有这四个扩展，要自己加，例如第五章的 `createToolSearchExtension()`。
- **关掉内置扩展**：在 `extensions` 设置里写 `-builtin:mcp`；项目设置里的 `+`、`-`、`!` 条目优先于用户设置。`-e builtin:mcp` 可以强制加载。
- **`-ne`（`--no-extensions`）**：去掉第 2–7 类，命令行 `-e` 的扩展和 SDK 工厂照样加载（P1b 里 `-ne -e a.ts` 时 `a.ts` 的工厂照常执行）。
- **settings 里的 `extensions` 条目**：普通条目是路径；带 `!`、`+`、`-` 前缀或含 `*`、`?` 的是过滤规则，用来排除或强制包含已收集到的文件，也能开关自动发现的文件。

### 1.2 目录里什么算一个扩展

自动发现只看一层：

- 顶层的 `*.ts`、`*.js` 文件各算一个扩展。
- 子目录先看 `package.json` 里的 `pi.extensions`（列出入口文件），没有就找 `index.ts`，再找 `index.js`。
- 跳过以 `.` 开头的项和 `node_modules`，遵守目录里的 `.gitignore`、`.ignore`、`.fdignore`。
- 例外：扩展目录本身就有 `package.json` 的 `pi.extensions` 或 `index.ts` / `index.js` 时，只加载这个入口，旁边的文件都不看。
- `*.d.ts` 也以 `.ts` 结尾，会被当成扩展加载，因为没有默认导出而报加载错误，CLI 因此启动失败。

P1 的全局目录里放了 `g-mjs.mjs` 和 `.g-hidden.ts`，两个都没有加载。`.mjs`、`.mts` 这类扩展名不会被自动发现，但用 `-e` 或 settings 显式给出路径时照样交给加载器。

pi 包（第一章 6 节的 Pi Packages）用同一套规则：`package.json` 的 `pi.extensions` 列出入口，没有时用约定的 `extensions/` 目录。

### 1.3 TypeScript 怎么加载

加载器用 [jiti](https://github.com/unjs/jiti)（2.7）导入扩展，`.ts` 文件经 jiti 内置的 Babel 转译后执行（Node 下 jiti 的 `tryNative` 默认关闭，pi 也没有打开它；Babel 在第一次需要转译时才加载）。所以扩展可以直接写 `.ts`，不需要编译，也不限于"可擦除"的类型语法。P1 的扩展里用了 `enum` 和 `satisfies`，照常运行，输出 `g-file enum=2`。

加载器的几个设置：

- **`moduleCache: false`**：每次导入都重新执行模块，不复用 Node 的模块缓存。另有一层按路径缓存工厂函数的表，第 2 节讲它和 `/reload` 的关系。
- **宿主提供的包**：扩展里导入 `@earendil-works/pi-coding-agent`、`pi-agent-core`、`pi-ai`、`pi-tui`、`typebox` 时，拿到的是 pi 自带的那一份。npm 安装的版本用别名表（`getAliases`）指过去，编译成单文件的二进制和自带 Node 的发行版用 `VIRTUAL_MODULES`（虚拟模块表）。`@earendil-works/pi-ai` 的根入口被指到 `compat` 入口，旧的全局 API 在扩展里还能用。
- **"重复的运行时"**：如果某个 pi 包把上面这些宿主包写进了 `dependencies`，`npm install` 会在包里装一份自己的副本，导入时可能绕过别名，变成第二份 pi 模块，类型检查和 `instanceof` 都会出问题。加载器为此给出警告，要求改写到 `peerDependencies` 并用 `"*"` 版本范围。P1 的 `user-pkg` 故意这样写，得到：

```text
Host-provided extension packages must be declared in peerDependencies with a "*" range, not dependencies: @earendil-works/pi-ai. Installed copies can bypass the extension loader and create duplicate runtime modules.
```

### 1.4 同名与重复

| 情形 | 结果 |
|---|---|
| 同一个文件被列了两次（例如 settings 和自动发现都指向它） | 按规范路径去重，只加载一次 |
| 全局和项目各放一份同样的扩展 | 是两个不同的文件，都加载 |
| 两个扩展注册同名工具或同名参数 | 第一个生效；后一个记一条加载错误 `Tool "dup" conflicts with <先注册的扩展>` |
| 两个扩展注册同名命令 | 都保留，调用名改成 `/hello:1`、`/hello:2` |
| 两个扩展注册同一个快捷键 | 后一个生效，记一条警告；与保留的内置快捷键冲突时跳过扩展的 |
| 两个扩展注册同名 MCP 服务器（只差 `-` 和 `_` 的也算） | 后注册的那次在工厂里抛错，这个扩展整个加载失败 |
| 第三方扩展注册了与可替换内置扩展同名的工具、命令或参数（例如 `tool_search`、`/mcp`） | 对应的内置扩展被替换，不加载 |
| SDK `customTools` 和扩展工具同名 | `customTools` 的生效 |

P1 里项目扩展 `p-file` 和全局扩展 `g-file` 都注册了工具 `dup` 和命令 `/hello`：工具 `dup` 的描述是 `from p-file`，命令变成 `/hello:1(p-file hello)`、`/hello:2(g-file hello)`；项目未信任时 `p-file` 不加载，`dup` 就是 `g-file` 的，命令也恢复成 `/hello`。

内置扩展被替换时给出的警告（P1 的 `g-replace` 注册了 `tool_search`）：

```text
Extension <T>/home/.pi/agent/extensions/g-replace.ts registers tool `tool_search`, so built-in extension `tool-search` was not loaded. To use `tool-search`, run `pi config` and make sure it is enabled under Built-in extensions, then disable or remove the existing extension. We recommend only having one or the other loaded at a time.
```

替换发生在加载之后：内置扩展的工厂已经执行过，只是它的 `Extension` 没有交给运行器。

### 1.5 项目信任

`.pi/` 里有 `settings.json`、`mcp.json`、`extensions`、`skills`、`prompts`、`themes`、`SYSTEM.md`、`APPEND_SYSTEM.md` 之一，或者工作目录及其祖先目录里有 `.agents/skills`（`~/.agents/skills` 除外）时，pi 要先决定是否信任这个项目。决定的顺序（`resolveProjectTrusted`）：

1. 命令行 `--approve` / `--no-approve`。
2. `project_trust` 事件：第一个返回 `yes` 或 `no` 的处理函数做决定，`undecided` 交给下一个。
3. `~/.pi/agent/trust.json` 里保存的决定，从当前目录往上找最近的一条。
4. 设置 `defaultProjectTrust`（`always`、`never`，默认 `ask`）。
5. 交互模式弹出选择；print、JSON、RPC 模式不能弹窗，按不信任处理。

`project_trust` 要在项目扩展加载之前回答，所以需要询问时（没有 `--approve` / `--no-approve`、这次进程里还没有决定过、项目里确实有需要信任的资源），CLI 先做一遍**预信任加载**：把项目当成未信任，加载命令行、用户和 SDK 的扩展（不含内置扩展），让它们处理 `project_trust`；决定之后再加载其余的，已经加载过的扩展直接复用，不会执行第二次。探针 P1c 在用户目录放了三个扩展，依次返回 `undecided`、`yes`、`no`：

```text
factory a-undecided
factory b-yes
factory c-no
a project_trust cwd=ws/proj mode=print hasUI=false
b project_trust -> yes
factory PROJECT extension
```

`c` 没有被调用，项目扩展随后加载。`b` 返回的 `remember: false` 使 `trust.json` 没有写入。

SDK 里由调用方决定：`SettingsManager.create` 的 `projectTrusted` 默认是 `true`，`createAgentSession` 不经过上面的流程。

### 1.6 加载失败

单个扩展失败不影响其他扩展加载，错误收集到 `getExtensions().errors`：

- 模块没有默认导出函数：`Extension does not export a valid factory function: <路径>`
- 导入或工厂抛错：文件扩展记为 `Failed to load extension: <错误信息>`，SDK 工厂和内置扩展只记原始错误信息。工厂里已经做的注册全部作废，之后再用这个扩展的 `pi` 会抛 `failed to load and its API is no longer active`。
- 工厂里调用动作方法（`sendMessage` 等）：抛 `Extension runtime not initialized. Action methods cannot be called during extension loading.`（P1 的 `g-early`）

CLI 对加载错误的处理更严格：启动时只要有一条错误，就打印错误和提示后以退出码 1 结束，四种模式都一样。1.4 节的同名工具冲突也记在错误里，同样会让 CLI 退出。探针 P1b 直接运行 npm 包里的 `pi`：

```text
--- tool conflict, project trusted (--approve)
factory b
factory a
Error: Failed to load extension "<T>/home/.pi/agent/extensions/a.ts": Tool "dup" conflicts with <T>/ws/proj/.pi/extensions/b.ts
Hint: Start without extensions using "pi -ne".
exit=1
--- same, project not trusted (--no-approve)
factory a
No API key found for the selected model.        ← 项目扩展没加载，没有冲突，启动继续
```

SDK 不会退出，要自己检查 `resourceLoader.getExtensions().errors`。`/reload` 之后的加载错误在交互界面里列出来，不会退出。

---

## 2 · 扩展怎么装进会话

![图 2 · 扩展的生命周期](/images/harness/pi/ch09/fig2-ext-lifecycle.png)

### 2.1 三个对象

- **`Extension`**（一个扩展的注册表）：加载器为每个扩展建一个，里面是 `handlers`（事件名 → 处理函数列表）、`tools`、`commands`、`shortcuts`、`flags`、`messageRenderers` 等。`pi.on`、`pi.registerTool` 这类注册方法只往这里写。
- **`ExtensionRuntime`**（共享运行时）：同一次加载的所有扩展共用一个。它持有动作方法（`sendMessage`、`appendEntry`、`setActiveTools`……）、参数值、排队中的供应商和虚拟模型注册、MCP 服务器表。刚建出来时动作方法都是会抛错的占位。
- **`ExtensionRunner`**（运行器）：持有本次加载的全部 `Extension` 和共享运行时，负责派发事件、汇总工具和命令、创建 `ctx`。

`ExtensionRunner` 由 `AgentSession` 在构造函数里创建（`_buildRuntime`），SDK 和运行模式都不直接创建它。创建之后立刻：

1. `bindCore`（接上核心动作）：把共享运行时里的占位换成 `AgentSession` 的真实方法，把加载期间排队的 `registerProvider`、`registerVirtualModel` 交给模型运行时，之后这类注册立即生效。
2. 用运行器包装扩展工具，刷新工具注册表（第五章 7.1 节）。

### 2.2 绑定：session_start 从这里发出

运行模式拿到会话后调用 `session.bindExtensions(bindings)`（绑定界面与错误监听）：

- `uiContext` 和 `mode`：交互模式传终端界面，RPC 模式传转发给客户端的实现，print 和 JSON 模式不传（第 5 节）。
- `commandContextActions`：命令上下文里 `newSession`、`fork`、`reload` 等方法的实现。
- `onError`：处理函数出错时的回调（第 6 节）。

接上之后它依次发出 `session_start` 和 `resources_discover`。所以 **`session_start` 不在会话创建时发出，而在绑定时发出**。用 SDK 时，`createAgentSession` 不会替你绑定；不调用 `session.bindExtensions(…)`，扩展就收不到 `session_start`，也收不到 `resources_discover`，处理函数出错时也没有人接收。本章的探针都在创建会话后调用了它，并传了 `onError`。

> ⚠️ `session.reload()` 只在至少有一项绑定（`uiContext`、`commandContextActions`、`shutdownHandler`、`onError` 之一）时才在重载后发 `session_start` 和 `resources_discover`。SDK 里只调用 `bindExtensions({})` 的话，重载后的扩展收不到 `session_start`。

### 2.3 /reload：重新导入模块，整体换掉运行器

`session.reload()`（交互界面里是 `/reload`，扩展命令里是 `ctx.reload()`）的步骤：

1. 向旧扩展发 `session_shutdown`，`reason` 为 `reload`。
2. 旧运行器 `invalidate()`：之后旧的 `pi`、`ctx` 一用就抛错，`pi.events` 上的订阅全部退订。
3. 重新读设置，`DefaultResourceLoader.reload()`：先 `clearExtensionCache()` 清掉工厂缓存，加上 `moduleCache: false`，每个扩展的模块重新导入、工厂重新执行。
4. `_buildRuntime` 建一个新的运行器，参数值沿用，之前激活的工具名留作待定，新运行器里有同名工具就恢复激活。
5. 有绑定时，发 `session_start`（`reason` 为 `reload`），然后发 `resources_discover`。

旧的注册不是逐项注销的，旧的 `Extension` 对象随旧运行器一起丢弃。探针 P4 在两次操作之间改写磁盘上的扩展文件，让新版本注册另一个工具：

```text
=== startup
  module body ran (version v1), total module runs=1
  factory ran (version v1), factory calls in this module instance=1
  session_start reason=startup (version v1)
  [registry after startup] tools=tool_v1 active=tool_v1 commands=/ver(version v1)
=== session.reload() after editing the file to v2
  session_shutdown reason=reload (version v1)
  module body ran (version v2), total module runs=2
  factory ran (version v2), factory calls in this module instance=1
  session_start reason=reload (version v2)
  [registry after reload] tools=tool_v2 active=tool_v2 commands=/ver(version v2)
  old pi.getActiveTools() threw: This extension ctx is stale after session replacement or reload. …
  old ctx.cwd threw: This extension ctx is stale after session replacement or reload. …
```

`tool_v1` 和旧的 `/ver` 都没了。pi 只清理它自己记录的东西：扩展在模块或工厂里启动的定时器、子进程、文件监听不会被自动停止，要在 `session_shutdown` 里自己关掉。官方文档的建议是不要在工厂里启动这类资源，放到 `session_start` 或用到它的命令、工具里。

### 2.4 /new、/resume、/fork：换一个会话，模块不重新导入

这三个操作由 `AgentSessionRuntime`（持有当前会话及其服务）完成，换掉的是整个 `AgentSession`：

1. 发 `session_before_switch`（`/new`、`/resume`）或 `session_before_fork`，处理函数可以取消（第六章）。
2. 发 `session_shutdown`（`reason` 为 `new`、`resume` 或 `fork`），`dispose()` 旧会话，旧运行器失效。
3. 用新的资源加载器和新的 `AgentSession` 重建，运行模式重新绑定，发 `session_start`（同样的 `reason`，并带上 `previousSessionFile`）。

新的资源加载器第一次加载时不清工厂缓存，所以扩展的工厂会重新执行，模块却不会重新导入（新会话的工作目录变了时缓存会被清掉，模块重新导入）。SDK 里用 `AgentSessionRuntime` 时，要用 `setRebindSession` 注册一个调用 `bindExtensions` 的函数，新会话才会收到 `session_start`。P4 接着把文件改成 v3，再调用 `runtime.newSession()`：

```text
=== runtime.newSession() after editing the file to v3
  session_shutdown reason=new (version v2)
  factory ran (version v2), factory calls in this module instance=2
  session_start reason=new (version v2)
=== runtime.dispose()
  session_shutdown reason=quit (version v2)
```

工厂第二次在同一个模块实例里执行，模块级变量还在，磁盘上的 v3 没有生效。所以模块顶层的状态会跨 `/new` 保留，要在新会话里重新开始的状态应放在工厂函数内部。

退出时 `AgentSessionRuntime.dispose()` 发 `session_shutdown`（`reason=quit`）。SDK 里直接调用 `session.dispose()` 只让运行器失效，不发 `session_shutdown`（P2 里 `dispose()` 之后没有任何事件）。

### 2.5 ctx 什么时候失效

运行器、共享运行时和每个扩展的 `pi` 对象都带一个失效标记。`ctx` 上的每个属性和方法都是取值时才检查（getter），失效后访问任何一个都抛同一段错误：

```text
This extension ctx is stale after session replacement or reload. Do not use a captured pi or command ctx after ctx.newSession(), ctx.fork(), ctx.switchSession(), or ctx.reload(). For newSession, fork, and switchSession, move post-replacement work into withSession and use the ctx passed to withSession. For reload, do not use the old ctx after await ctx.reload().
```

失效发生在 `/reload` 发完 `session_shutdown` 之后，以及会话被替换或 `dispose()` 时。命令里调用 `ctx.newSession({ withSession })` 这类方法后，要继续操作新会话，用 `withSession` 回调里拿到的新 `ctx`。

---

## 3 · ExtensionAPI 能注册什么

![图 3 · ExtensionAPI 能力地图](/images/harness/pi/ch09/fig3-ext-api.png)

### 3.1 33 个成员

`ExtensionAPI` 有 33 个成员（用 TypeScript 类型检查器数出来的属性数）：

- **订阅**（2）：`on`（订阅事件，返回退订函数）、`events`（扩展之间互发消息的事件总线，`emit` / `on`）。
- **注册**（11）：
  - `registerTool`（给模型用的工具）、`registerCommand`（`/` 命令）、`registerShortcut`（快捷键）、`registerFlag`（命令行参数）
  - `registerMessageRenderer`（`custom_message` 的显示）、`registerEntryRenderer`（`custom` 条目的显示）、`registerToolRenderer`（工具调用的显示）、`registerMarkdownTransformer`（显示前改写 Markdown）
  - `registerProvider`（模型供应商）、`registerVirtualModel`（按请求路由到具体模型的虚拟模型）、`registerMcpServer`（MCP 服务器）
- **注销**（3）：`unregisterProvider`、`unregisterVirtualModel`、`unregisterMcpServer`，分别撤销上面最后三种注册。
- **动作**（10）：
  - 写会话或发消息：`sendMessage`（发一条自定义消息）、`sendUserMessage`（以用户身份发消息）、`appendEntry`（写一条不进上下文的条目）、`setSessionName`（会话名）、`setLabel`（给条目加标签）
  - 改会话状态：`setActiveTools`（激活的工具）、`setModel`（模型）、`setThinkingLevel`（思考级别）
  - 其他：`exec`（在工作目录里执行命令）、`getFlag`（读命令行参数的值）
- **查询**（7）：`getSessionName`（会话名）、`getActiveTools`（激活的工具名）、`getAllTools`（全部工具）、`getSettings`（合并后的设置）、`getCommands`（扩展命令、模板和技能命令）、`getThinkingLevel`（思考级别）、`getMcpServers`（已注册的 MCP 服务器）。

能在工厂里调用的是注册、注销、`on`、`events`、`getFlag`、`exec` 和 `getMcpServers`。其余动作和查询在加载期间都是占位，调用就抛 1.6 节的错误；`setModel` 返回一个被拒绝的 Promise。

### 3.2 每项注册落到哪里

| 方法 | 落到哪里 | 能否撤销 | 同名时 |
|---|---|---|---|
| `on` | 本扩展 `handlers` 表，运行器派发时读取 | 返回的函数退订；不影响正在进行的派发 | 各自执行 |
| `registerTool` | `AgentSession` 的工具注册表，再到 `agent.state.tools`（第五章） | 不能注销 | 先注册的生效 |
| `registerCommand` | 命令表。`prompt()` 遇到 `/名字` 先查这里，查到就执行，不发请求 | 不能 | 都保留，改名 `name:N` |
| `registerShortcut` | 交互模式的按键表 | 不能 | 后注册的生效 |
| `registerFlag` | 命令行参数，`pi --help` 里列出；`getFlag` 读值 | 不能 | 先注册的生效 |
| `registerMessageRenderer` / `registerEntryRenderer` | 交互界面里 `custom_message` / `custom` 条目的显示 | 不能 | 先注册的生效 |
| `registerToolRenderer` | 工具调用的显示，多个解析器按加载顺序串成一条链，每个可以交给下一个 | 不能 | 链式 |
| `registerMarkdownTransformer` | 交互界面渲染 user、assistant 的 Markdown 之前改写文字 | 不能 | 全部依次执行 |
| `registerProvider` / `registerVirtualModel` | 模型运行时。加载期间排队，`bindCore` 时生效；之后立即生效 | `unregister*` | — |
| `registerMcpServer` | MCP 服务器表。变化时发 `mcp_servers_change`，由 `builtin:mcp` 去连接；加载期间注册的在 `session_start` 时读取 | `unregisterMcpServer` | 抛错 |

两点说明：

- 渲染器只在交互模式起作用。print、JSON、RPC 模式里注册不会报错，只是不用。
- 没有扩展处理 `mcp_servers_change` 时（例如 `builtin:mcp` 被关掉或替换了），注册的 MCP 服务器会被报告为错误：`MCP server "<name>" is registered, but no loaded extension connects MCP servers; …`。

### 3.3 动作对会话的影响

探针 P6 在空闲的会话上逐个调用动作方法，记录新增的请求数和会话条目：

```text
sendMessage(display:true)                    idle
    requests +0; entries +[custom_message(note)]
sendMessage(triggerTurn:true)                idle
    requests +1; entries +[message:system, custom_message(kick), message:assistant]
sendMessage(deliverAs:nextTurn)              idle
    requests +0; entries +[]
sendUserMessage("hello")                     idle
    requests +1; entries +[message:system, message:user, custom_message(next), message:assistant]
  input source=extension text="hello"
appendEntry("todo-state", {...})             idle
    requests +0; entries +[custom(todo-state)]
setLabel / setSessionName                    idle
    requests +0; entries +[label, session_info]
setActiveTools([read, work])                 idle
    requests +0; entries +[]
```

再让一次运行里的工具在执行期间调用三个动作：

```text
prompt("do work") — tool sends 3 messages while streaming
    requests +2; entries +[…, message:toolResult, custom_message(later-note), custom_message(steer-note), message:assistant]
!! emitError <runtime> event=send_user_message error=Agent is already processing. Specify streamingBehavior ('steer' or 'followUp') to queue the message.
```

整理成规则：

| 动作 | 空闲时 | 运行中 |
|---|---|---|
| `sendMessage(msg)` | 写一条 `custom_message`，不发请求 | 默认作为插队消息（steer）交给当前运行 |
| `sendMessage(msg, { triggerTurn: true })` | 写入并立即开始一次运行 | 同上 |
| `sendMessage(msg, { triggerTurn: false })` | 只写入 | 不插队，等这一轮的工具结果写完再写入；这次运行如果还有下一次请求，会带上它 |
| `sendMessage(msg, { deliverAs: "followUp" })` | 只写入 | 排进后续消息队列（第三章 2.2 节） |
| `sendMessage(msg, { deliverAs: "nextTurn" })` | 不写，等下一次 prompt 跟在 user 消息后面写入 | 同左 |
| `sendUserMessage(text)` | 走 `prompt()`：`input` 事件的 `source` 为 `extension`，默认不展开模板 | 必须给 `deliverAs: "steer"` 或 `"followUp"`，否则报错 |
| `appendEntry(type, data)` | 写一条 `custom` 条目，不进上下文 | 同左 |
| `setLabel` / `setSessionName` | 写 `label` / `session_info` 条目 | 同左 |
| `setActiveTools(names)` | 不写条目；下一次请求前以 system 补丁写入（第八章 4 节） | 下一轮生效 |

`custom_message` 在发给模型前转成 user 消息（第二章 3.2 节），`display` 只影响界面里是否显示。`sendMessage` 和 `sendUserMessage` 不返回 Promise，失败时通过 `onError` 报告，`extensionPath` 记为 `<runtime>`。

两个容易踩的点：

> 📌 **`triggerTurn` 启动的运行不经过 prompt 的前半段**。`sendMessage(..., { triggerTurn: true })` 直接把消息交给 `Agent`，不经过 `input`、`before_agent_start`，也不重算提示词补丁（第八章 4.1 节的第一个时机）。探针 P6b 在一个全新的会话上第一个动作就这样做，这次请求里只有一条文字为空的 system 消息，没有系统提示词；之后一次正常的 prompt 才把完整的段作为补丁写进来。需要提示词和 `before_agent_start` 的场景，用 `sendUserMessage`。
>
> 📌 **`appendEntry` 存的是引用**。条目在内存里直接保存传入的 `data`。会话文件要等第一条 assistant 消息出现才开始写（第六章 2.3 节），在那之前追加的条目写盘时才序列化，几条条目可能序列化出同样的最新内容（第 7 节的例子第一版就踩了这一点）；文件已经存在时，条目立即序列化进文件，但内存里的条目仍会跟着变，`getBranch()` 读到的是改过的值。传 `structuredClone(data)`。

---

## 4 · 事件全表

### 4.1 41 个事件，两种方法核对

`pi.on` 能订阅的事件由联合类型 `ExtensionEvent` 定义：32 个成员，其中 `SessionEvent` 又是 10 个 `session_*` 事件的联合，合计 41 个事件名。两种方法核对：

1. **类型**：用 TypeScript 类型检查器展开 `ExtensionEvent["type"]`，得到 41 个字符串字面量；`ExtensionAPI.on` 的重载也是 41 个，两份名单完全一致。
2. **派发点**：在源码里逐个查找每个事件的派发位置，包括通用的 `runner.emit({ type: … })`、专用的 `emitInput`、`emitToolCall` 这类方法，以及 `AgentSession` 转发引擎事件的 `_emitExtensionEvent`。41 个都至少有一处，没有联合类型以外的事件名。

`ExtensionRunner` 用通用的 `emit()` 派发其中 26 个：22 个只通知的，加上 4 个 `session_before_*`（`emit()` 里专门处理了它们的取消和返回值）。另外 15 个走专用路径：12 个专用方法（`emitBoundary` 兼管 `turn_end` 和 `agent_before_settle`，`emitContext` 兼管 `context` 和 `context_with_system`），以及独立函数 `emitProjectTrustEvent`。

### 4.2 一次 prompt 里的先后

![图 4 · 一次 prompt 的事件时间线](/images/harness/pi/ch09/fig4-ext-timeline.png)

探针 P2 注册一个订阅全部 41 个事件的扩展，用 faux 跑一次"用户 → 工具调用 → 工具结果 → 回答"；P2b 用本地 Anthropic mock 把供应商相关的事件补全。P2b 的记录（连续的 `message_update`、`provider_stream_event` 已合并）：

```text
  input text="go" source=interactive
  before_agent_start prompt="go"
  agent_start
  turn_start turnIndex=0
  message_start role=system
  message_end role=system
  message_start role=user
  message_end role=user
  context messages=[user]
  context_with_system messages=[system,user]
  before_provider_headers headers=0
  before_provider_request payload keys=[model,messages,max_tokens,stream,system,tools]
  after_provider_response status=200
  provider_stream_event …
  message_start role=assistant
  message_update toolcall_start / toolcall_delta / toolcall_end（与 provider_stream_event 交替）
  message_end role=assistant
  tool_execution_start echo
  tool_call echo input={"text":"hi"}
  tool_result echo isError=false
  tool_execution_end echo isError=false
  message_start role=toolResult
  message_end role=toolResult
  turn_end turnIndex=0 message=assistant toolResults=1 entries=0 continue=false
  turn_start turnIndex=1
  context … context_with_system … before_provider_headers … before_provider_request … after_provider_response
  message_start role=assistant … message_end role=assistant
  turn_end turnIndex=1 message=assistant toolResults=0 entries=0 continue=false
  agent_end messages=5
  agent_before_settle entries=0 continue=false
  agent_settled
```

从这份记录可以读出几件事：

- `message_start / end` 的第一对是 system 消息，即第八章的提示词补丁，它和 user 消息一起在第 0 轮开始后写入。
- `tool_execution_start` 在 `tool_call` 之前：引擎先宣布开始执行，再调用 `beforeToolCall` 钩子。`tool_call` 阻止了调用时，`tool_result` 不会触发（P3）。
- `before_provider_headers` 在 `before_provider_request` 之前。faux 会触发 `before_provider_headers` 和 `after_provider_response`，但不触发 `before_provider_request` 和 `provider_stream_event`。
- 工具执行期间调用了 `onUpdate` 才有 `tool_execution_update`（P2 的 faux 版本有一次）。

其他操作触发的事件（P2 后半段）：

```text
setSessionName("probe")        session_info_changed name=probe
setModel(faux-think)           model_select faux-ext→faux-think source=set
setThinkingLevel("high")       thinking_level_select off→high
compact()                      session_before_compact reason=manual → before_provider_headers（摘要请求）→ session_compact fromExtension=false
navigateTree(…)                session_before_tree → session_tree
```

压缩的摘要请求也经过 `before_provider_headers`，但不经过 `context` 一类事件（第七章 3.4 节）。

### 4.3 分组表

"结果"一列：**通知**表示返回值被忽略；其余写明能返回什么、多个扩展怎样合并（规则见 4.4 节）。最后一列是已经详细讲过的章节。

**启动与资源（3）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `project_trust` | 预信任加载之后、项目扩展加载之前 | `{ trusted: "yes" \| "no" \| "undecided", remember? }`，先到先得 | 1.5 节 |
| `resources_discover` | 每次绑定和重载发完 `session_start` 之后（`reason` 只有 `startup`、`reload` 两种，`/new` 等之后也是 `startup`）；没有处理函数时不发 | `{ skillPaths, promptPaths, themePaths }`，收集 | 第八章 3.2 节 |
| `mcp_servers_change` | 运行器构造（`bindCore`）之后 MCP 服务器表变化 | 通知，不等待 | 3.2 节 |

**会话（10）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `session_start` | 绑定时；`reason` 为 `startup`、`reload`、`new`、`resume`、`fork` | 通知 | 2.2 节 |
| `session_shutdown` | 重载、替换、退出前；`reason` 为 `reload`、`new`、`resume`、`fork`、`quit` | 通知 | 2.3 节 |
| `session_info_changed` | 会话改名 | 通知，不等待 | 第六章 |
| `session_before_switch` / `session_before_fork` | `/new`、`/resume` / `/fork` 之前 | `{ cancel }`，拦截即停 | 第六章 6 节 |
| `session_before_tree` / `session_tree` | `/tree` 导航前 / 后 | 前者 `{ cancel, summary… }`；后者通知 | 第六章 |
| `session_before_compact` / `session_compact` / `session_compact_failed` | 压缩前 / 后 / 失败 | 前者 `{ cancel, compaction }`；后两个通知 | 第七章 5 节 |

**运行（5）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `before_agent_start` | prompt 展开之后、运行开始之前 | 改 `systemPromptOptions`（接力）；`message` 收集；`systemPrompt` 接力，最后一个生效 | 第八章 5.1 节 |
| `agent_start` / `agent_end` | 一次运行开始 / 引擎结束 | 通知 | — |
| `agent_before_settle` | 运行即将结束 | `{ entries, continue }`，接力 | 第六章 7.2 节 |
| `agent_settled` | 不会再自动继续 | 通知 | — |

**轮与消息（5）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `turn_start` | 每轮开始，带 `turnIndex` | 通知 | — |
| `turn_end` | 每轮结束 | `{ entries, continue }`，接力 | 第六章 7.2 节 |
| `message_start` / `message_update` | 消息开始 / 流式更新 | 通知 | — |
| `message_end` | 消息完成、写会话之前 | `{ message }` 替换消息，接力；角色必须不变 | 4.4 节 |

**工具（5）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `tool_call` | 执行前 | `{ block, reason }`，拦截即停；原地改 `event.input` 接力 | 第三章 4.2 节 |
| `tool_result` | 执行后 | 改 `content`、`details`、`isError` 等，接力 | 第三章 4.2 节 |
| `tool_execution_start` / `_update` / `_end` | 引擎的执行进度 | 通知 | — |

**请求（6）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `context` / `context_with_system` | 每次请求前改消息 | `{ messages }`，接力 | 第八章 5.1 节 |
| `before_provider_headers` | 发请求前 | 原地改 `event.headers`，接力 | 第八章 5.1 节 |
| `before_provider_request` | 请求体拼好后 | 返回值替换请求体，接力 | 第八章 5.1 节 |
| `after_provider_response` | 收到响应头 | 通知 | — |
| `provider_stream_event` | 每个解析出的原始流事件 | 通知；逐个等待，处理慢会拖慢流 | — |

**输入（2）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `input` | 用户、RPC 或扩展的输入到达 | `transform` 接力，`handled` 拦截即停 | 第八章 5.1 节 |
| `user_bash` | 交互和 RPC 模式里用户用 `!` 执行命令 | `{ operations }` 或 `{ result }`，先到先得 | — |

**设置与界面（5）**

| 事件 | 何时 | 结果 | 详见 |
|---|---|---|---|
| `model_select` | 换模型，`source` 为 `set` 或 `cycle`（类型里还有 `restore`，这个版本不会发出） | 通知 | — |
| `thinking_level_select` | 换思考级别 | 通知，不等待 | — |
| `cache_warming_decision` | 缓存预热定时器到期 | `{ action: "warm" \| "stop" }`，后者覆盖 | 第八章 6.3 节 |
| `ui_prompt_start` / `ui_prompt_end` | 扩展弹出的对话框打开 / 关闭（只算最外层） | 通知，不等待 | 第 5 节 |

3 + 10 + 5 + 5 + 5 + 6 + 2 + 5 = 41。"不等待"的五个（`mcp_servers_change`、`session_info_changed`、`thinking_level_select`、`ui_prompt_start`、`ui_prompt_end`）是丢进微任务或不 `await` 的派发，处理函数拖不住调用方。其余事件的处理函数都被逐个 `await`，一个慢的处理函数会让整条流程等它。

### 4.4 返回值的五种合并规则

![图 5 · 返回值合并规则](/images/harness/pi/ch09/fig5-ext-merge.png)

41 个事件里，19 个会采用处理函数的返回值或修改，22 个只通知。会采用结果的事件按下面五种规则合并：

| 规则 | 做法 | 事件 |
|---|---|---|
| 接力 | 每个处理函数拿到前一个的输出，最后的结果生效 | `input`（transform）、`before_agent_start`（共用选项对象，`systemPrompt` 也写在上面）、`context`、`context_with_system`、`before_provider_request`、`before_provider_headers`、`message_end`、`tool_call`（原地改 input）、`tool_result`、`turn_end`、`agent_before_settle` |
| 拦截即停 | 某个处理函数给出终止信号，后面的不再执行 | `input` 的 `handled`、`tool_call` 的 `block`、四个 `session_before_*` 的 `cancel` |
| 后者覆盖 | 不接力，最后一个非空返回值生效 | `session_before_*`（非 cancel 的结果）、`cache_warming_decision` |
| 先到先得 | 第一个给出决定的生效，之后的不执行 | `project_trust`、`user_bash` |
| 收集 | 所有结果合并成一个列表 | `resources_discover`、`before_agent_start` 的 `message` |

探针 P3 装两个扩展 A、B，在同一次运行里观察几种规则（模型先后调用 `echo("hi")`、`echo("stop")`，再回答）：

```text
A message_end assistant="echo({\"text\":\"hi\"})"
B message_end assistant="echo({\"text\":\"hi\"}) <A>"            ← B 看到 A 追加的 " <A>"
!! emitError <inline:3> event=message_end error=message_end handlers must return a message with the same role
A tool_call input={"text":"hi"}
B tool_call input={"text":"hi+A"}                              ← A 原地改了 input
  echo.execute text=hi+A
A tool_result content="hi+A"
B tool_result content="hi+A [A]"                               ← B 看到 A 改过的结果
A turn_end entries_in=0
B turn_end entries_in=1                                        ← B 看到 A 提交的草稿
…
A tool_call input={"text":"stop"}
B tool_call input={"text":"stop+A"}                            ← B 返回 block，工具没有执行，也没有 tool_result
…
navigateTree -> {"cancelled":true}
A session_before_tree                                          ← A 返回 cancel，B 没有被调用
```

B 在 `message_end` 里返回了一条 role 为 `user` 的消息，被拒绝并报告，A 的修改保留。会话文件里的结果：

```text
   message assistant "echo({\"text\":\"hi\"}) <A>"
   message toolResult "hi+A [A]"
   custom a-note {"turn":0}
   custom b-note {}
   message assistant "echo({\"text\":\"stop\"}) <A>"
   message toolResult "B blocks stop"
   …
```

被阻止的调用，工具结果的内容就是 `reason`（`B blocks stop`），标为错误交回模型。

---

## 5 · ExtensionContext：三种 ctx

每次派发事件时，运行器新建一个 `ctx`，这次派发的所有处理函数共用它。按调用场合分三种（P5 列出各自的属性）：

| ctx | 谁拿到 | 在上一行基础上多出的成员 |
|---|---|---|
| `ExtensionContext`（18 项） | 事件处理函数、快捷键 | 见下面的列表 |
| `ExtensionCommandContext`（25 项） | 命令的 `handler(args, ctx)` | `waitForIdle()`（等运行结束）、`reload()`（重载）、`newSession()` / `fork()` / `switchSession()`（替换会话）、`navigateTree()`（在会话树上跳转）、`getSystemPromptOptions()`（基础提示词选项） |
| `ExtensionToolContext`（20 项） | 工具的 `execute(…, ctx)` | `tools`（能被调用的工具）、`executeTool()`（嵌套调用，第五章 5.4 节） |

`ExtensionContext` 的 18 项：

- 界面：`ui`（对话框、通知等）、`mode`（`tui` / `rpc` / `json` / `print`）、`hasUI`（能否和用户交互）
- 环境：`cwd`（工作目录）、`sessionManager`（会话，类型上只读）、`modelRegistry`（模型注册表）、`model`（当前模型）、`scopedModels`（`--models` 限定的模型）、`thinkingLevel`（思考级别）
- 运行状态：`signal`（当前运行的中止信号）、`isIdle()`（是否空闲）、`hasPendingMessages()`（是否有排队消息）、`getContextUsage()`（上下文用量）、`getSystemPrompt()`（当前提示词）、`isProjectTrusted()`（项目是否被信任）
- 控制：`abort()`（中止运行）、`compact()`（发起压缩，第七章）、`shutdown()`（请求退出进程）

- 会替换会话、重载、等待空闲的方法只放在命令上下文里。官方文档的解释是：在事件处理函数里调用它们可能让运行时死锁。
- `before_agent_start` 拿到的 `ctx.getSystemPrompt()` 返回正在被修改的那份提示词，其他事件返回当前生效的提示词（第八章 4.3 节）。
- 命令执行时不发请求，也不写任何条目（P5 里 `/probe a b` 之后会话里只有开头的两条设置条目）。

**`ctx.ui` 在各模式下的行为**：

- **交互模式**（`mode` 为 `tui`，`hasUI` 为 true）：完整的终端界面，对话框、通知、状态栏、组件都可用。
- **RPC 模式**（`mode` 为 `rpc`，`hasUI` 为 true）：
  - `select`、`confirm`、`input`、`editor` 作为 `extension_ui_request` 发给客户端，等客户端回复。
  - `notify`、`setStatus`、`setWidget`（只转发字符串数组）、`setTitle`、`setEditorText` / `pasteToEditor`（发 `set_editor_text`）单向发送。
  - `custom` 和自定义组件是空操作。
- **print / JSON 模式**（`mode` 为 `print` / `json`，`hasUI` 为 false）：交互和显示都是空操作。
  - `select`、`input`、`editor` 返回 `undefined`，`confirm` 返回 `false`，`getEditorText` 返回空串。
  - `setTheme` 返回 `{ success: false, error: "UI not available" }`；`theme` 仍返回当前主题。

SDK 不传 `uiContext` 时和 print 模式一样（P5：`hasUI=false mode=print select=undefined confirm=false`）。只在终端里有意义的功能用 `ctx.mode === "tui"` 判断，需要用户回答的交互用 `ctx.hasUI` 判断。

---

## 6 · 隔离与出错

### 6.1 处理函数抛错

运行器的派发方法逐个用 `try/catch` 包住处理函数：抛错时组装一个 `{ extensionPath, event, error, stack }` 交给 `emitError`，然后接着调用下一个，当前运行不受影响。`project_trust` 发生在绑定之前，它的错误不交给 `onError`，而是变成启动警告 `Extension "…" project_trust error: …`；处理函数返回 `undefined` 也算出错，要返回 `{ trusted: "undecided" }`。P3 让 A 在 `turn_end` 里抛错，三轮都是：

```text
A turn_end entries_in=0
!! emitError <inline:2> event=turn_end error=A exploded in turn_end
B turn_end entries_in=0
```

B 照常执行并提交了草稿，运行正常结束。

例外有两个：

- **`tool_call`**：派发时没有 `try/catch`，抛错直接传到引擎，后面的扩展不再执行，这次工具调用被阻止，错误信息作为工具结果交给模型。这是有意的：权限类扩展出错时宁可不执行。P3 让 A 在 `tool_call` 里抛错，会话里记的是 `message toolResult "A exploded in tool_call"`，没有 `emitError`。
- **`user_bash`**：先报告，再把错误继续抛出，这条命令不执行。

`emitError` 的错误交给绑定时给的 `onError`，各模式的呈现：

| 模式 | 呈现 |
|---|---|
| 交互 | 对话区一条 `Extension "<路径>" error: <信息>`，下面是灰色的调用栈 |
| print / JSON | stderr 一行 `Extension error (<路径>): <信息>` |
| RPC | 一条 `{ type: "extension_error", extensionPath, event, error }`，不带调用栈 |
| SDK | 自己在 `bindExtensions({ onError })` 里接；不绑定就没有人收到 |

命令的处理函数抛错时，`extensionPath` 记为 `command:<名字>`，命令照样算已处理，不会把文字发给模型。

### 6.2 有没有权限边界

没有。扩展是普通的 TypeScript 模块，在 pi 的进程里执行，拥有 pi 进程的全部权限：读写任意文件、执行命令、发网络请求、读环境变量和 `~/.pi/agent/auth.json`。`docs/security.md` 的说法是 "Extensions execute inside the Pi process"，建议加载前先审查扩展和包。

pi 提供的限制都只管"加载不加载"：

- **项目信任**：没有信任的项目，`.pi/` 里的扩展和项目 settings 里的包不加载（1.5 节）。用户目录、命令行和 SDK 的扩展不受它约束。
- **`-ne`**：一次性关掉自动发现的扩展。

扩展一旦加载，`ctx.sessionManager` 的只读只体现在类型上（第六章 6 节），`pi.exec` 也不经过任何审批。需要隔离时，按 `docs/security.md` 的建议把整个 pi 放进容器或虚拟机。

---

## 7 · 用法：一个 /todo 扩展

下面这个扩展同时用到命令、工具、事件、`appendEntry` 和消息渲染器：

- 用户用 `/todo add <文字>`、`/todo done <编号>` 维护一个待办列表，`/todo` 显示列表。
- 模型可以调用 `todo_done` 工具勾掉一项。
- 列表的快照存成会话里的 `custom` 条目，模型改动过的那一轮结束时，再写一条模型看得到的汇总。
- `session_start` 和 `session_tree` 时从当前分支的最后一个快照恢复，所以 `/reload`、恢复会话、`/tree` 换分支之后状态都对。

```typescript
import { Type } from "@earendil-works/pi-ai";
import type { ExtensionAPI, ExtensionContext } from "@earendil-works/pi-coding-agent";
import { Text } from "@earendil-works/pi-tui";

interface Todo {
	id: number;
	text: string;
	done: boolean;
}

const STATE = "todo-state"; // custom entry: a snapshot, not sent to the model
const SUMMARY = "todo-summary"; // custom message: sent to the model as a user message

export default function (pi: ExtensionAPI) {
	let todos: Todo[] = [];
	let dirty = false;

	const format = () =>
		todos.length === 0 ? "No todos." : todos.map((t) => `${t.done ? "[x]" : "[ ]"} #${t.id} ${t.text}`).join("\n");

	// Rebuild from the latest snapshot on the current branch: survives /reload, resume, and /tree.
	const restore = (ctx: ExtensionContext) => {
		const last = ctx.sessionManager
			.getBranch()
			.filter((e) => e.type === "custom" && e.customType === STATE)
			.at(-1);
		todos = last?.type === "custom" ? structuredClone(last.data as Todo[]) : [];
		dirty = false;
	};
	pi.on("session_start", (_event, ctx) => restore(ctx));
	pi.on("session_tree", (_event, ctx) => restore(ctx));

	pi.registerCommand("todo", {
		description: "Manage todos: /todo add <text> | /todo done <id> | /todo",
		handler: async (args, ctx) => {
			const [verb, ...rest] = args.trim().split(/\s+/);
			if (verb === "add" && rest.length > 0) {
				todos.push({ id: todos.length + 1, text: rest.join(" "), done: false });
			} else if (verb === "done") {
				const todo = todos.find((t) => t.id === Number(rest[0]));
				if (!todo) return ctx.ui.notify(`No todo #${rest[0]}`, "error");
				todo.done = true;
			} else {
				// Without triggerTurn this does not start a run; the list joins the context of the next request.
				return pi.sendMessage({ customType: SUMMARY, content: format(), display: true });
			}
			pi.appendEntry(STATE, structuredClone(todos)); // commands run outside a turn; copy, the entry keeps a reference
		},
	});

	pi.registerTool({
		name: "todo_done",
		label: "Todo done",
		description: "Mark a todo item as done by id",
		promptSnippet: "Mark items of the user's todo list as done",
		parameters: Type.Object({ id: Type.Number() }),
		async execute(_toolCallId, params) {
			const todo = todos.find((t) => t.id === params.id);
			if (!todo) throw new Error(`No todo #${params.id}`);
			todo.done = true;
			dirty = true;
			return { content: [{ type: "text", text: `#${todo.id} done` }], details: {} };
		},
	});

	// Changes made by the model are written once per turn, together with a summary the model sees next turn.
	pi.on("turn_end", (event) => {
		if (!dirty) return;
		dirty = false;
		const open = todos.filter((t) => !t.done).length;
		return {
			entries: [
				...event.entries,
				{ type: "custom", customType: STATE, data: structuredClone(todos) },
				{ type: "custom_message", customType: SUMMARY, content: `Todos (${open} open):\n${format()}`, display: true },
			],
		};
	});

	pi.registerMessageRenderer(SUMMARY, (message, { outputPad }, theme) => {
		const text = typeof message.content === "string" ? message.content : "";
		return new Text(theme.fg("accent", text), outputPad, 0);
	});
}
```

这段代码里的几个选择：

- **状态放在会话里，不放在文件里**：快照是 `custom` 条目，跟着会话树分支走。恢复时只看 `getBranch()`（当前分支），不扫整个文件，被放弃的分支不会混进来。
- **命令里直接写，工具里攒到 `turn_end`**：命令在运行之外执行，直接 `appendEntry`。工具在运行中改了状态，就把快照和汇总作为条目草稿交给 `turn_end`，和这一轮的其他扩展草稿一起校验、一起写入（第六章 7.2 节）。
- **快照不进上下文，汇总进上下文**：`custom` 条目模型看不到；`custom_message` 会转成 user 消息，同一次运行的下一次请求就能看到更新后的列表。
- **复制再存**：`appendEntry` 和草稿里的 `data` 都用 `structuredClone`，原因见 3.3 节。
- **渲染器只影响交互界面**：其他模式里 `custom_message` 照样写入、照样发给模型，只是不经过这个渲染器。

这段代码在 `tsc --strict --skipLibCheck --target es2022 --module nodenext` 下通过（用了 `Array.prototype.at`，需要 ES2022 的库）。探针 P7 把它原样放进 `~/.pi/agent/extensions/todo.ts`，用 faux 依次执行：两次 `/todo add`；一次 prompt，模型调用 `todo_done({ id: 1 })` 后回答；再一次 prompt；`session.reload()`；最后 `/todo`。

```text
after two /todo add: requests sent = 0
request 3 (after the turn that called todo_done) carried: system,user,assistant,toolResult,user,assistant,user
after reload, /todo wrote: custom_message(todo-summary) "[x] #1 write tests\n[ ] #2 update changelog"
errors: []
--- session file
   session (header)
   model_change {"provider":"faux","modelId":"faux-ext"}
   thinking_level_change {"thinkingLevel":"off"}
   custom todo-state [{"id":1,"text":"write tests","done":false}]
   custom todo-state [{"id":1,"text":"write tests","done":false},{"id":2,"text":"update changelog","done":false}]
   message system  sections={preamble,tools,rules,docs,cwd} toolsAdded=[read,bash,edit,write,todo_done]
   message user "Tests are written, tick it off."
   message assistant "todo_done({\"id\":1})"
   message toolResult "#1 done"
   custom todo-state [{"id":1,"text":"write tests","done":true},{"id":2,"text":"update changelog","done":false}]
   custom_message todo-summary display=true "Todos (1 open):\n[x] #1 write tests\n[ ] #2 update changelog"
   message assistant "Marked #1."
   message user "thanks"
   message assistant "ok"
   custom_message todo-summary display=true "[x] #1 write tests\n[ ] #2 update changelog"
```

- 两次 `/todo add` 没有发请求，各留下一条快照。
- 模型调用工具的那一轮结束时，`turn_end` 写入新快照和汇总。同一次运行的第二次请求就以这条汇总（转成 `user`）结尾，第三次请求里 `toolResult` 之后的那条 `user` 也是它。
- `/reload` 重新导入模块，内存里的 `todos` 从空数组开始，`session_start` 从最后一条快照恢复，所以重载后的 `/todo` 列出的是 `#1` 已完成。

---

## 8 · 三个可以带走的方法

1. **注册和动作分两个阶段**。工厂执行时只允许写自己的注册表，动作方法是会抛错的占位，供应商注册先排队；宿主准备好之后统一接上。扩展的加载顺序和副作用因此可控，加载失败的扩展可以整块丢弃。
2. **每个事件声明自己的合并规则**。只通知的事件走通用派发，会采用结果的事件按接力、拦截、覆盖、先到先得、收集之一合并，大多有自己的派发方法。扩展作者看事件的返回类型就知道多个扩展怎样相处，不必猜测。
3. **重载就是换一整套**。不逐项注销，旧运行器连同它的注册一起丢弃，旧的 `pi` 和 `ctx` 标记失效，用了就报清楚的错误。宿主不需要追踪每个扩展注册过什么。

---

## 9 · 关键数字

| 项 | 数量 |
|---|---|
| 扩展来源 | 8 类 |
| 内置扩展 | 4 个（`llama.cpp`，以及可替换的 `codemode`、`tool-search`、`mcp`），只在 CLI 里装上 |
| `ExtensionAPI` 成员 | 33 个，其中注册方法 11 个、注销方法 3 个 |
| 事件 | 41 个（`ExtensionEvent` 的 32 个成员，`SessionEvent` 展开为 10 个） |
| 会采用结果的事件 / 只通知的事件 | 19 / 22 |
| 有专用派发方法的事件 | 15 个 |
| 合并规则 | 5 种 |
| 不等待处理函数的事件 | 5 个 |
| `ctx` 成员：事件 / 命令 / 工具 | 18 / 25 / 20 |
| `ctx.ui` 成员 | 28 个 |
| `session_start` 的 `reason` | 5 种（`startup`、`reload`、`new`、`resume`、`fork`） |
| `session_shutdown` 的 `reason` | 5 种（`quit`、`reload`、`new`、`resume`、`fork`） |
| 扩展相关源码 | `extensions/types.ts` 2260 行、`runner.ts` 1560 行、`loader.ts` 876 行 |
| 官方示例 | `examples/extensions/` 下 70 个 `.ts` 文件、9 个目录 |
| 本章相关文件的提交（北京时间 2026-08-01 至 2026-10-04，不含合并提交） | `core/extensions/` 与 `resource-loader.ts` 合计 41 次 |

---

## 10 · 术语表

| 术语 | 含义 | 别和它混淆 |
|---|---|---|
| **工厂** | 扩展模块默认导出的函数，加载时调用一次 | 处理函数：`pi.on` 注册的、事件发生时调用的函数 |
| **`Extension`** | 一个扩展的注册表 | `ExtensionAPI`：交给工厂的 `pi` 对象，往注册表里写 |
| **运行器** | `ExtensionRunner`，派发事件、汇总注册 | `AgentSessionRuntime`：持有当前会话，负责替换会话 |
| **绑定** | `bindExtensions`：接上界面和错误监听，发 `session_start` | `bindCore`：构造时把动作方法接到 `AgentSession` |
| **失效** | 重载或替换后旧的 `pi`、`ctx` 一用就抛错 | 卸载：pi 不逐项卸载，旧注册随旧运行器丢弃 |
| **预信任加载** | 决定是否信任项目之前，先加载用户、命令行和 SDK 的扩展 | 项目信任本身：只决定 `.pi/` 下的资源加载不加载 |
| **可替换的内置扩展** | 第三方注册同名工具/命令/参数时让位的内置扩展 | 同名冲突：两个普通扩展同名时记加载错误 |

---

## 11 · 源码导航

| 想搞懂 | 从这里读 |
|---|---|
| 来源、顺序、信任 | `packages/coding-agent/src/core/package-manager.ts` → `resolve`、`resourcePrecedenceRank`、`collectAutoExtensionEntries`；`core/resource-loader.ts` → `reload`、`loadFinalExtensionSet`、`omitReplacedExtensions`、`detectExtensionConflicts`、`collectExtensionPackageWarnings`；`core/project-trust.ts`、`core/trust-manager.ts` |
| 模块怎么导入 | `packages/coding-agent/src/core/extensions/loader.ts` → `loadExtensionModule`、`createExtensionAPI`、`createExtensionRuntime`；`core/extensions/virtual-modules.ts` |
| 内置扩展 | `packages/coding-agent/src/extensions/index.ts`；`src/main.ts` 里的 `builtInExtensions` 和启动时的错误检查 |
| 运行器与派发 | `packages/coding-agent/src/core/extensions/runner.ts` → `bindCore`、`emit`、`emitBoundary`、`emitToolCall`、`emitToolResult`、`emitMessageEnd`、`emitInput`、`createContext`、`createCommandContext`、`invalidate` |
| 装进会话、重载 | `packages/coding-agent/src/core/agent-session.ts` → `_buildRuntime`、`_bindExtensionCore`、`bindExtensions`、`reload`、`_emitExtensionEvent`、`sendCustomMessage`、`sendUserMessage` |
| 替换会话 | `packages/coding-agent/src/core/agent-session-runtime.ts` → `newSession`、`switchSession`、`fork`、`teardownCurrent` |
| 类型 | `packages/coding-agent/src/core/extensions/types.ts` → `ExtensionAPI`、`ExtensionEvent`、`ExtensionContext`、`ExtensionCommandContext`、`ExtensionToolContext` |
| 各模式的 ui 和错误呈现 | `src/modes/rpc/rpc-mode.ts` → `createExtensionUIContext`；`src/modes/print-mode.ts`；`src/modes/interactive/interactive-mode.ts` |
| 官方说明 | `packages/coding-agent/docs/extensions.md`、`rpc-extension-ui.md`、`security.md`、`packages.md`；示例 `examples/extensions/` |

**下一章**：运行模式与界面。看交互、print、JSON、RPC 四种模式怎样驱动同一个 `AgentSession`。
