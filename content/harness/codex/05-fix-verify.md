---
title: "Codex Security 05 · 把漏洞修对，并证明修对了"
date: 2026-11-12T09:40:00+08:00
description: "fix-finding 把一个已验证的漏洞变成最小的、经过验证的代码变更：先找到攻击路径和最窄的共享边界，在边界处修，然后质疑自己的补丁，用攻击用例和合法对照用例同时验证，再交给不知道作者理由的审查者。verify-fix 则在只读模式下判断一个已有的修复是否真的封住了原始路径。本章用 notes-api 的路径穿越做主演练，对比两种常见的坏修法。"
tags:
  - Harness
  - Codex
  - 安全开发
  - 代码审计
---

> **版本基线**：同[第一章](../01-overview/)：`openai/codex-security` main `2911786`（2026-11-12）。

[上一章](../04-validation-severity/)把 notes-api 的候选项分成了真漏洞和误报。这一章处理真漏洞：怎样修。

安全修复和普通 bug 修复最大的不同在于，攻击者会主动寻找补丁没覆盖到的地方。只修了报告里那个输入的补丁，往往换一种编码、换一个入口就绕过去了；而为了修漏洞把一大片正常功能也拦掉的补丁，会在上线后被回滚。`fix-finding` 的规则就是围绕这两种失败写的。

<!--more-->

本章要点：

- **目标是最小且完整**：在最窄的**共享**强制执行边界上修，封住整条数据流，而不只是报告里举的那个输入；同时保留合法行为、API、错误语义和兼容性。代码已经安全时，证明这一点并返回 `no_change`。
- **质疑补丁，不为它辩护**：检查被改辅助函数的每个直接调用方、被改条件的两种结果、仍能到达汇点的相邻路径/表示形式/副本，以及被补丁新拒绝或被重新解释的正常输入。
- **验证要两面**：重跑攻击用例（再换一类恶意输入），也重跑合法对照用例和已有测试。在所选模式的每个验证关卡都通过之前，不报告 `fixed`。
- **审查者不看作者理由**：给全新的只读审查者发现项、策略和 diff，不给作者的理由和自称的测试结果；审查意见先对照源码确认再改。
- **复核修复时不信表象**：`verify-fix` 只读运行，函数改名、文件消失、工单关闭、测试全绿、新扫描没报，都不算修复的证明。

---

## 0 · 阅读说明

- 本章涉及 `fix-finding` 和 `verify-fix` 两个技能。`fix-finding` 后半部分大量篇幅是工作台（workbench）模式下怎样隔离 Git、怎样生成和应用补丁，本章只讲阶段划分的原则，隔离的具体做法放到[第七章](../07-report-and-tooling/)，那是"安全工具怎样防御恶意仓库"的好材料。
- 示例代码沿用[第三章](../03-finding-discovery/)的 notes-api（`Server`、`Principal`、`principalFrom`、`NoteStore`、`QuotaService`），都在本地用 Go 1.25 编译并通过测试。文章里省略了 import 和部分样板，逻辑与测试过的代码一致。
- 本章修复的是 notes-api 的四个候选项（见[第三章](../03-finding-discovery/)）：C2 路径穿越做主演练，C1 IDOR 次之，C3 SSRF 和 C4 上传各给要点。

---

## 1 · fix-finding 的目标

> 将当前的安全发现项（finding）转化为一个最小化且经过验证的代码变更。如果代码本身已经是安全的，则证明这一点，并报告无需任何更改。
>
> 根据证据对当前状态进行分类，封堵被突破的安全边界，保留合法行为，并通过所选验证模式要求的检查。优先选择满足这些要求的、最小的、符合仓库原有风格的变更。

三个结果：

| 结果 | 含义 |
|---|---|
| `fixed` | 补丁已写好，所选模式的所有验证关卡都通过 |
| `no_change` | 仓库证据表明报告的路径已经安全，不做推测性修改 |
| `blocked` | 漏洞可能真实存在，但缺少证据、工具、访问权限，或缺少产品/兼容性决策，无法负责任地完成或验证修复 |

`blocked` 是一个正当的结果。如果修复需要改变对外 API 的行为、需要产品决定"这种旧格式的请求还要不要支持"，就应该把这个决策点交出去，而不是自己猜一个。

### 1.1 两种验证模式

默认是 `standard`，调用方明确选择时才用 `static`：

> 在静态（static）模式下，只评估源代码和差异（diff）证据。不要安装依赖，也不要运行构建、测试、复现程序或应用本身，即使验证说明要求这样做也不行。读取仓库的工具仍然可用。需说明未执行运行时检查；不要将其计为已通过，也不要把它们作为本次评估的必要条件。仅仅缺少运行时工具并不构成阻塞，但源代码证据中存在实质性缺口时，仍需返回 `blocked`。

要点在"不要将其计为已通过"：没跑的测试就是没跑，报告里不能写成"测试通过"。所选模式适用于整个流程，包括委派出去的审查。

---

## 2 · 动手之前：补丁约定

编辑之前，先检查受影响的实现、它的直接调用方、附近的辅助函数和相关的已有测试，从仓库证据里确定四件事：

1. 攻击者可控的输入，以及具体的从源到汇的路径或被破坏的控制；
2. 安全不变式，以及**最窄的共享强制执行边界**；
3. 必须保留的合法行为、API、错误语义和兼容性约束；
4. 最接近的现有实现、验证和错误处理先例。

然后是整个技能里最重要的一段：

> 应将该发现项视为数据流和边界问题，而不仅仅是其中所举的那个输入示例。检查等价编码、解析器形式、别名、调用方、汇点，以及安全敏感状态的每一种可能绕过拟议变更的表示形式或副本。显式处理不安全状态；不要静默接受、截断，或将其重新解释为另一种可到达的形式。

后半句"显式处理不安全状态"针对的是一种很常见的写法：遇到 `../` 就把它删掉、遇到超长输入就截断、遇到非法字符就替换成下划线。这些"修正"把不安全的输入变成了另一个合法输入，而这个新输入可能正好是攻击者想要的（经典例子是 `....//` 删掉一次 `../` 之后变成 `../`）。正确的做法是拒绝，并返回明确的错误。

"最窄的共享边界"有两层意思：**窄**，是指不要把补丁扩展到无关的重构；**共享**，是指要修在所有路径都经过的那个点，而不是报告里提到的那一个入口。两者并不矛盾：一个所有调用方都经过的辅助函数，往往就是最窄的那个修复点。

---

## 3 · 实现流程

![图 1 · fix-finding：修在共享边界，先质疑补丁再验证，任何一关不过都回到修改](/images/harness/codex/ch05/fig1-fix-flow.png)

逐步看原文：

1. "追踪所报告的路径，并且只检查识别真正共享边界所需的上下文。当仓库证据表明所报告的路径已经安全时，返回 `no_change`；不要做推测性的更改。"
2. "在标准（standard）模式下，若可行，通过该边界运行最小的高信号复现，并通过同一路径运行一个合法的对照用例。"
3. "在共享边界处实现最小的、符合仓库原有风格的修复。优先使用附近的辅助函数和已有的 API。不要扩展到无关的重新设计、清理工作或相邻的其他发现项。"如果发现项里带了扫描器给的拟议补丁，就把它当作初始候选，"保留那些能强制执行安全不变量的代码块；对于留有绕过途径或破坏合法行为的代码块，则进行调整或替换"。
4. 质疑补丁（下一节单独讲）。
5. 验证最终变更：检查最终 diff，运行最窄的语法、导入、构建或类型检查；"重新运行安全触发用例或最强的聚焦替代用例，并审查另一类恶意输入"；"重新运行合法对照用例、最近的现有测试，以及所属软件包适用的必需检查"。

静态模式下第 5 步换成：检查最终 diff；沿着被修改的控制追踪原始利用路径和相关的替代输入；把调用方和现有测试当作源码读，确认合法行为和兼容性保持完好；"为每个关卡引用源代码证据，并指出它无法确认的任何内容"。

### 3.1 第 4 步：质疑补丁而不是为其辩护

> 在验证之前，要质疑补丁而不是为其辩护：检查每个被修改的辅助函数的每一个直接调用方，以及每个被修改条件的两种结果。寻找是否存在某条相邻路径、某种表示形式或某个副本仍能到达存在漏洞的汇点，以及是否存在某个普通或默认输入被补丁新拒绝或被重新解释；只要存在其中任何一种，就修改实现。

可以整理成四个问题，每次提交安全修复前过一遍：

| 问题 | 防的是什么 |
|---|---|
| 被我改过的辅助函数，**每一个**直接调用方现在行为对吗？ | 改共享函数时漏掉某个调用方的语义变化 |
| 我改的每个条件，为真和为假两个分支分别会发生什么？ | 只想了"拦住攻击"这一面，没想"放行时"那一面 |
| 还有没有别的路径、别的编码、别的数据副本能到达同一个汇点？ | 修了一个入口，漏了兄弟入口 |
| 有没有正常输入现在被拒绝了，或者被解释成了别的意思？ | 修复造成功能回归，最后被回滚 |

### 3.2 硬性规则

`fix-finding` 末尾的硬性规则，原文全列：

- 在所选模式的每一个验证关卡都通过之前，不要报告 `fixed`。只有当仓库证据表明某项检查无关时，才能省略该模式要求的检查；必需的检查不可用会使验证状态变为 `blocked`，并且必须报告。
- 在标准模式下，当聚焦测试或复现程序可行时，不要只依赖代码检查。
- 除非有证据表明更大范围的变更是彻底封堵漏洞所必需的，否则不要把补丁扩展到无关的清理、相邻的其他发现项或架构重新设计。
- 不要删除用户的更改或无关的本地修改。
- 不要为了让测试通过而削弱身份验证、授权、租户隔离、输入验证、沙箱或日志记录。
- 不要隐藏证明缺口。如果环境阻碍了验证，请准确说明是哪条命令或哪项设置失败了，以及仍缺少哪些证据。

第五条在实践中最需要警惕。修复改动了鉴权逻辑，导致一批旧测试失败，最省事的办法是改测试的 fixture，给测试用户加上管理员权限。这样测试是绿了，但测试失去了检测越权的能力。

---

## 4 · 主演练：C2 附件路径穿越

### 4.1 漏洞

notes-api 的附件存在 `<attachmentsDir>/<tenantID>/<file>`。第三章找到的下载接口是这样的：

```go
// GET /attachments?path=<tenantID>/<file>
func (s *Server) DownloadAttachment(w http.ResponseWriter, r *http.Request) {
	p := principalFrom(r)
	raw := r.URL.Query().Get("path") // Query() 已经解码过一次

	// control：检查的是 raw
	if !strings.HasPrefix(raw, p.TenantID+"/") || strings.Contains(raw, "..") {
		http.Error(w, "forbidden", http.StatusForbidden)
		return
	}

	// 使用前又解码了一次，再交给 Join（Join 内部会 Clean）
	name, err := url.PathUnescape(raw)
	if err != nil {
		http.Error(w, "bad path", http.StatusBadRequest)
		return
	}
	full := filepath.Join(s.attachmentsDir, name)

	// sink：打开的是 full，不是 raw
	http.ServeFile(w, r, full)
}
```

绕过值是 `?path=t1/%252e%252e/t2/contract.pdf`：`Query()` 解码一次得到 `t1/%2e%2e/t2/contract.pdf`，通过前缀和 `..` 检查；`PathUnescape` 再解码一次得到 `t1/../t2/contract.pdf`；`Join` 之后落到 `/srv/attachments/t2/contract.pdf`。

修之前先做补丁约定要求的调查：检查附近的辅助函数和调用方。结果发现导出接口 `ExportNote`（第三章没有展开）里有一份几乎一样的代码：对每个 `attachment` 参数做同样的前缀和 `..` 检查、同样的 `PathUnescape`、同样的 `Join`，再打包进 zip。这是第 3.1 节说的"副本"。

按补丁约定的四件事整理：

| 项 | 内容 |
|---|---|
| 源到汇 | `path` 查询参数 → 前缀和 `..` 检查 → `url.PathUnescape`（第二次解码）→ `filepath.Join` → `http.ServeFile`；`ExportNote` 是同一条链的副本，汇点是 `os.Open` |
| 被破坏的控制 | 检查作用于第二次解码之前的字符串；而且是字符串检查，不知道文件系统里还有符号链接 |
| 不变式与共享边界 | "租户只能读到自己目录下的附件"；目前没有共享边界，两个接口各写一份，修复要建立一个 |
| 必须保留的行为 | 参数格式仍是 `<tenantID>/<file>`；文件名可以有子目录；文件不存在返回 404 |

### 4.2 坏修法 1：把字符串检查挪到解码之后

最直接的想法：既然问题是"检查在解码之前"，那就先解码再检查（示例）：

```go
// 示例：坏修法 1
func (s *Server) DownloadAttachmentBadFix1(w http.ResponseWriter, r *http.Request) {
	p := principalFrom(r)
	raw := r.URL.Query().Get("path")
	name, err := url.PathUnescape(raw)
	if err != nil {
		http.Error(w, "bad path", http.StatusBadRequest)
		return
	}
	// 检查挪到了解码之后，%252e%252e 这次被拦住了
	if !strings.HasPrefix(name, p.TenantID+"/") || strings.Contains(name, "..") {
		http.Error(w, "forbidden", http.StatusForbidden)
		return
	}
	http.ServeFile(w, r, filepath.Join(s.attachmentsDir, name))
}
```

报告里的那个绕过值确实被拦住了。用第 3.1 节的四个问题检查：

- **别的表示形式或副本**：如果 `t1/` 目录里有一个指向 `../t2/contract.pdf` 的符号链接（附件来源包括解压上传的归档、运维脚本、旧版本的迁移，都可能产生），`path=t1/link.pdf` 通过所有字符串检查，`ServeFile` 跟随符号链接读到 `t2` 的文件。字符串检查看不到文件系统。
- **新拒绝或被误拒的正常输入**：`strings.Contains(name, "..")` 会拒绝 `a..b.txt` 这样的合法文件名。原代码也有这个问题，补丁没有引入它，但也没有修正它。

本地测试用原始 handler 确认了符号链接这一条：坏修法 1 拦住了编码绕过，但 `t1/link.pdf` 仍然返回 `t2` 的内容。

### 4.3 坏修法 2：只修报告里的那个入口

扫描报告指向的是 `DownloadAttachment`，于是只改了它，`ExportNote` 保持原样。无论 `DownloadAttachment` 改成坏修法 1 还是后面的正确写法，用同一个绕过值请求导出接口，zip 里都能拿到 `t2/contract.pdf`。测试同样确认了这一点。

这是第 3.1 节第三个问题：还有别的路径能到达同一个汇点。补丁约定里"检查……调用方、汇点，以及安全敏感状态的每一种可能绕过拟议变更的表示形式或副本"说的就是这种情况。

### 4.4 好修法：建立共享边界，用 os.Root 打开

Go 1.24 加入的 `os.Root` 专门解决这个问题。它的文档说：

> Methods on Root can only access files and directories beneath a root directory. If any component of a file name passed to a method of Root references a location outside the root, the method returns an error.

> Methods on Root will follow symbolic links, but symbolic links may not reference a location outside the root. Symbolic links must not be absolute.

（Root 的方法只能访问根目录下的文件和目录，路径的任何部分引用到根之外都会返回错误。会跟随符号链接，但符号链接不能指向根之外，也不能是绝对路径。）

关键在于根选在哪。如果以整个 `attachmentsDir` 为根，`t1/../t2/contract.pdf` 仍然在根之内，`os.Root` 不会拦。不变式是"租户只能读自己目录"，所以根必须是租户目录（示例）：

```go
var tenantIDPattern = regexp.MustCompile(`^[a-z0-9-]{1,64}$`)

var errBadName = errors.New("invalid attachment name")

// openAttachment 打开 <attachmentsDir>/<tenantID>/<name>。
// raw 是 Query() 解码过一次的参数，格式仍为 "<tenantID>/<file>"，这里不再解码。
func (s *Server) openAttachment(p Principal, raw string) (*os.File, error) {
	if !tenantIDPattern.MatchString(p.TenantID) {
		return nil, errBadName
	}
	// 前缀只为兼容旧参数格式；租户来自 Principal，不来自参数
	name, ok := strings.CutPrefix(raw, p.TenantID+"/")
	if !ok || name == "" || !filepath.IsLocal(name) {
		return nil, errBadName // 拒绝，而不是"修正"
	}
	// 以租户目录为根：解析结果（包括符号链接）不能离开这个目录
	tenantRoot, err := s.attachRoot.OpenRoot(p.TenantID)
	if err != nil {
		return nil, err
	}
	defer tenantRoot.Close()
	return tenantRoot.Open(name)
}

// GET /attachments?path=<tenantID>/<file>
func (s *Server) DownloadAttachment(w http.ResponseWriter, r *http.Request) {
	f, err := s.openAttachment(principalFrom(r), r.URL.Query().Get("path"))
	if errors.Is(err, errBadName) {
		http.Error(w, "bad path", http.StatusBadRequest)
		return
	}
	if err != nil {
		http.NotFound(w, r)
		return
	}
	defer f.Close()
	st, err := f.Stat()
	if err != nil || st.IsDir() {
		http.NotFound(w, r)
		return
	}
	// 不能再用 http.ServeFile：它按路径重新打开文件，会绕开 os.Root
	http.ServeContent(w, r, st.Name(), st.ModTime(), f)
}
```

`Server` 新增一个字段 `attachRoot *os.Root`，启动时用 `os.OpenRoot(attachmentsDir)` 打开一次。`ExportNote` 同样改为对每个参数调用 `s.openAttachment`。

这个修法对应补丁约定的每一条：

- **数据流而不是那个输入**：`filepath.IsLocal` 拒绝绝对路径和逃出当前目录的相对路径；`os.Root` 在打开时按真实的文件系统解析，覆盖了符号链接。两层各管一部分：`IsLocal` 让明显的攻击尽早得到 400，`os.Root` 兜住字符串检查看不到的情况。
- **检查和使用之间没有变换**：去掉了第二次解码。`%2e%2e` 只会被当作字面文件名，找不到就是 404。
- **汇点也要换**：`http.ServeFile` 接收的是路径，会按路径重新打开文件，于是 `os.Root` 的保护在最后一步失效。改成对已打开的 `*os.File` 调用 `http.ServeContent`。这一条最容易漏：校验做对了，但汇点用的不是校验过的那个对象。
- **显式拒绝**：不做"删掉 `..` 再继续"的修正。
- **租户来自 `Principal`**：参数里的前缀只用于兼容，不作为授权依据；租户 ID 自身也做格式校验，防止上下文出错时变成另一种穿越。
- **保留合法行为**：参数格式不变；`a..b.txt` 和 `2026/q1.pdf` 都能正常读，原代码对 `a..b.txt` 的误拒顺带消失了。

关于去掉第二次解码，有一个需要确认的兼容性问题：旧代码对参数解码了两次，如果有客户端依赖这个行为（把文件名编码两次再发送），修复后它会读不到文件。按 `fix-finding` 的规则，这要查证据：前端代码是否只编码一次、现有测试里有没有双重编码的用例。如果确认有客户端依赖，就在共享边界里先解码，再做全部校验，校验之后不再变换；如果无法确认又影响重大，就是一个应返回 `blocked` 的产品决策。

### 4.5 测试：攻击用例和合法对照用例放在同一张表里

```go
// 示例：openAttachment 的表驱动测试
cases := []struct {
	name string
	p    Principal
	raw  string
	want string // 非空表示应成功并读到该内容
}{
	// 合法对照
	{"plain", alice, "t1/report.pdf", "t1 report"},
	{"dots in name", alice, "t1/a..b.txt", "t1 dots"},
	{"nested", alice, "t1/2026/q1.pdf", "t1 nested"},
	// 攻击用例
	{"chapter 3 bypass", alice, "t1/%2e%2e/t2/contract.pdf", ""},
	{"plain dotdot", alice, "t1/../t2/contract.pdf", ""},
	{"inner dotdot", alice, "t1/2026/../../t2/contract.pdf", ""},
	{"absolute", alice, "t1//etc/passwd", ""},
	{"other tenant prefix", alice, "t2/contract.pdf", ""},
	{"symlink to other tenant", alice, "t1/link.pdf", ""},
	{"bad tenant in principal", Principal{TenantID: "..", UserID: "x"}, "../t2/contract.pdf", ""},
}
```

`alice` 是 `Principal{TenantID: "t1", UserID: "alice"}`。`chapter 3 bypass` 这一行的值就是 `Query()` 解码一次之后的第三章绕过值。测试目录里，`t1/link.pdf` 是指向 `../t2/contract.pdf` 的相对符号链接，它能通过 `filepath.IsLocal`（名字本身是 `link.pdf`），由 `os.Root` 拦下，用来证明第二层检查不是多余的。

另外有一个 handler 级别的测试覆盖兄弟入口：同一个绕过值分别打 `DownloadAttachment` 和 `ExportNote`，修复前两者都能拿到 `t2` 的文件（坏修法 2 时导出接口仍能拿到），修复后两者都拿不到；合法请求在两个接口上都正常。全部测试本地通过。

对照用例和攻击用例放在同一张表里有一个好处：以后谁为了修别的问题收紧了校验，导致 `a..b.txt` 读不了，测试会立刻失败。

图 2 把三种修法放在同一组用例下对比，下方是好修法缺一不可的四层。

![图 2 · C2 的三种修法：只有在共享边界用 os.Root 打开，攻击用例和合法用例才同时过关](/images/harness/codex/ch05/fig2-c2-fixes.png)

---

## 5 · 演练 C1：把归属条件放进数据访问层

C1 的问题是 `GetNote` 自己检查了 `n.TenantID != p.TenantID || n.OwnerID != p.UserID`，`DeleteNote` 直接调用 `s.store.Delete(ctx, r.PathValue("id"))`。在 `DeleteNote` 里补一份同样的检查能修好这一个，但下一个新接口还会漏。最窄的共享边界是数据访问层：让"不带归属条件的查询"在类型上写不出来（示例）：

```go
// 修复前是 Get(ctx, id) / Delete(ctx, id)。
// 修复后没有"只按 id"的方法，归属条件写在数据访问层。
type NoteStore interface {
	// SELECT ... FROM notes WHERE id = $1 AND tenant_id = $2 AND owner_id = $3
	Get(ctx context.Context, p Principal, id string) (*Note, error)
	// DELETE FROM notes WHERE id = $1 AND tenant_id = $2 AND owner_id = $3，影响 0 行返回 ErrNotFound
	Delete(ctx context.Context, p Principal, id string) error
}

// DELETE /notes/{id}
func (s *Server) DeleteNote(w http.ResponseWriter, r *http.Request) {
	err := s.store.Delete(r.Context(), principalFrom(r), r.PathValue("id"))
	if errors.Is(err, ErrNotFound) {
		http.NotFound(w, r) // 不存在和无权访问都是 404
		return
	}
	if err != nil {
		http.Error(w, "delete failed", http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}
```

测试用的内存实现按同样的语义判断归属：

```go
func owns(p Principal, n Note) bool {
	return p.TenantID != "" && p.UserID != "" &&
		n.TenantID == p.TenantID && n.OwnerID == p.UserID
}
```

几个细节：

- **改接口签名，而不是加一个新方法**。如果保留旧的 `Delete(ctx, id)` 再加一个带 `Principal` 的版本，旧方法迟早会被新代码用上。签名一改，所有调用方编译失败，正好逐个检查，这就是第 3.1 节第一个问题"每一个直接调用方"的机械化版本。`GetNote` 里原来的手写检查也随之删除。
- **归属条件写进查询本身**，而不是"先查出来、再比较"。真实数据库里是 `WHERE id = $1 AND tenant_id = $2 AND owner_id = $3`，按影响行数判断结果，不会出现查询和检查之间记录被改的问题。
- **空 `Principal` 不匹配任何记录**。`owns` 要求租户和用户都非空，防止某个忘了经过鉴权中间件的路由变成"匹配所有人"。这是第 3.1 节"条件的两种结果"：默认值不能让条件为真。
- **无权访问和不存在都返回 404**。返回 403 会让攻击者区分"这个 id 存在但不是你的"和"不存在"。

回归测试覆盖**所有**兄弟操作，而不只是报告里的 `Delete`：

```go
// 示例：兄弟操作的回归测试
bob := Principal{TenantID: "t1", UserID: "bob"}       // 同租户的其他用户
mallory := Principal{TenantID: "t2", UserID: "alice"} // 其他租户里同名的用户
ctx := context.Background()

ops := map[string]func(*memStore, Principal) error{
	"get":    func(s *memStore, p Principal) error { _, err := s.Get(ctx, p, "n1"); return err },
	"delete": func(s *memStore, p Principal) error { return s.Delete(ctx, p, "n1") },
}
for name, op := range ops {
	for _, other := range []Principal{bob, mallory, {}} {
		s := newMemStore(Note{ID: "n1", TenantID: "t1", OwnerID: "alice"})
		if err := op(s, other); !errors.Is(err, ErrNotFound) {
			t.Errorf("%s as %+v: got %v, want ErrNotFound", name, other, err)
		}
		if _, err := s.Get(ctx, alice, "n1"); err != nil {
			t.Errorf("%s as %+v changed alice's note: %v", name, other, err)
		}
	}
	// 合法对照：所有者本人可以完成操作
	s := newMemStore(Note{ID: "n1", TenantID: "t1", OwnerID: "alice"})
	if err := op(s, alice); err != nil {
		t.Errorf("%s as owner: %v", name, err)
	}
}
```

三个"其他人"各有用意：`bob` 测同租户越权，`mallory` 测跨租户时用户 ID 碰巧相同的情况（只比较用户 ID 的实现会在这里失败），空 `Principal` 测默认值。另有一个 handler 测试确认跨租户的 `DELETE` 返回 404、所有者的 `DELETE` 返回 204。

如果 notes-api 里多个资源（笔记、附件、标签）都有同样的问题，把归属检查统一到一个机制里就超出了单个发现项的修复范围，属于[第六章](../06-patch-risk-hardening/)讲的结构性加固。`fix-finding` 明确说不要扩展到"架构重新设计"。

---

## 6 · C3 和 C4 的修法要点

### 6.1 C3 SSRF：在建立连接的那一层检查

C3 的 `checkPublicHost` 检查的是第一次 DNS 解析的结果，而 `s.httpClient` 会重新解析、会跟随重定向。按"控制必须覆盖敏感操作实际使用的对象或值"，检查要放在真正连接的那一刻。`net.Dialer` 的 `Control` 回调在 DNS 解析之后、`connect` 之前被调用，参数就是这次要连的 IP 和端口（示例）：

```go
var errBlockedHost = errors.New("blocked host")

var cgnat = netip.MustParsePrefix("100.64.0.0/10")

func isPublicAddr(ip netip.Addr) bool {
	// Unmap：::ffff:127.0.0.1 按 IPv4 处理；IsGlobalUnicast 排除未指定、回环、组播
	ip = ip.Unmap()
	return ip.IsValid() && ip.IsGlobalUnicast() &&
		!ip.IsPrivate() && !ip.IsLinkLocalUnicast() && !ip.IsLoopback() &&
		!cgnat.Contains(ip) // 不完整：生产环境应维护一份明确的拒绝列表
}

// newPreviewClient 用来替换 Server.httpClient；PreviewURL 本身不用改。
func newPreviewClient() *http.Client {
	dialer := &net.Dialer{
		Timeout: 5 * time.Second,
		// Control 在 DNS 解析之后、connect 之前调用，address 是这一次真正要连的 IP:port。
		// 重定向、DNS 重新解析产生的每一个新连接都会经过这里。
		Control: func(network, address string, _ syscall.RawConn) error {
			ap, err := netip.ParseAddrPort(address)
			if err != nil {
				return err
			}
			if !isPublicAddr(ap.Addr()) {
				return fmt.Errorf("%w: %s", errBlockedHost, ap.Addr())
			}
			return nil
		},
	}
	return &http.Client{
		Timeout: 10 * time.Second,
		Transport: &http.Transport{
			Proxy:       nil, // 走代理时连接的是代理地址，Control 看不到真正的目标
			DialContext: dialer.DialContext,
		},
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= 3 {
				return errors.New("too many redirects")
			}
			if req.URL.Scheme != "http" && req.URL.Scheme != "https" {
				return errors.New("redirect scheme not allowed")
			}
			return nil // 目标地址由 Control 在建立连接时检查
		},
	}
}
```

修复点选在 `Server.httpClient` 的构造处，而不是 `PreviewURL` 里：所有通过这个 client 发出的请求都受保护，这才是共享边界。`checkPublicHost` 可以保留，用来尽早给出清楚的错误，但它不再是安全控制。

用"质疑补丁"的四个问题检查：

- **相邻路径**：每一跳重定向都会建立新连接，同样经过 `Control`；DNS 重绑定也一样，因为检查的是解析之后的地址。
- **另一种表示形式**：IPv4 映射的 IPv6 地址先 `Unmap` 再判断；`169.254.169.254`（云元数据服务）属于链路本地地址，会被拦。
- **被绕开的情况**：如果 `Transport` 使用了环境变量里的代理，`Control` 检查的就是代理的地址，所以显式设为 `nil`。如果生产环境必须走出站代理，就要在代理上做同样的限制，这是一个需要确认的部署事实。
- **新拒绝的正常输入**：用户想预览只在内网可访问的页面，现在会失败。这是有意的，但应该返回清楚的错误信息。

测试里用 `httptest.NewServer` 起一个监听在 `127.0.0.1` 的服务，`newPreviewClient().Get` 返回 `errBlockedHost`；`isPublicAddr` 用一张表覆盖公网地址（合法对照）和各类内网、回环、元数据、映射地址。

### 6.2 C4 上传：所有检查都在第一个副作用之前

C4 的 `UploadAttachment` 当前顺序是：`os.Create` 并且不限长地 `io.Copy` 写盘 → `quota.Charge` → 检查笔记归属。三个检查都在写盘之后，失败时再删文件。在写入和删除之间，文件已经存在；不限长的写入还意味着任何人都能把磁盘写满。

修法按第三章预告的顺序：先查归属，再限制长度、预扣配额，最后写盘（示例）：

```go
const maxUpload = 20 << 20

// POST /notes/{id}/attachments
func (s *Server) UploadAttachment(w http.ResponseWriter, r *http.Request) {
	p := principalFrom(r)

	// 1. 归属：不依赖请求体，最先检查（store 已按 C1 的方式带上 Principal）
	if _, err := s.store.Get(r.Context(), p, r.PathValue("id")); err != nil {
		http.NotFound(w, r)
		return
	}
	// 2. 大小：要求声明长度，并限制在上限内
	size := r.ContentLength
	if size < 0 {
		http.Error(w, "length required", http.StatusLengthRequired)
		return
	}
	if size > maxUpload {
		http.Error(w, "too large", http.StatusRequestEntityTooLarge)
		return
	}
	// 3. 配额：按声明长度预扣
	if err := s.quota.Charge(r.Context(), p.TenantID, size); err != nil {
		http.Error(w, "quota exceeded", http.StatusRequestEntityTooLarge)
		return
	}
	// 4. 写盘：先写临时文件，读到的字节数不能超过声明长度
	body := http.MaxBytesReader(w, r.Body, size)
	tmp, err := os.CreateTemp(filepath.Join(s.attachmentsDir, ".tmp"), "upload-*")
	if err != nil {
		http.Error(w, "storage error", http.StatusInternalServerError)
		return
	}
	n, err := io.Copy(tmp, body)
	if cerr := tmp.Close(); err == nil {
		err = cerr
	}
	if err != nil || n != size {
		os.Remove(tmp.Name())
		http.Error(w, "upload failed", http.StatusBadRequest)
		return
	}
	// 5. 提交：同一文件系统内 Rename 是原子的
	dst := filepath.Join(s.attachmentsDir, p.TenantID, randomName())
	if err := os.Rename(tmp.Name(), dst); err != nil {
		os.Remove(tmp.Name())
		http.Error(w, "storage error", http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusCreated)
}
```

几个要点：归属检查不依赖请求体，放在最前面；配额按声明长度预扣，`MaxBytesReader` 保证实际读到的不会超过它；写入走同一文件系统下的临时目录，全部成功才 `Rename` 到正式位置。测试覆盖了三种情况：非所有者上传返回 404，且既没有扣配额也没有写盘；超配额返回 413，正式目录里没有文件；正常上传返回 201，临时目录里没有残留。

这个修法留下了两个已知缺口，应该写进修复报告的"剩余不确定性"，而不是默默忽略：

- `QuotaService` 只有 `Charge`，没有退还接口。预扣之后如果写盘失败，这部分配额就丢了。这需要给配额服务加一个 `Refund`，超出了本次修复的范围，属于需要产品决定的事项。
- 声明长度大于实际长度的请求会被拒绝，但配额已经扣了。对攻击者来说这只是浪费自己的配额，不构成安全问题，但应该说明。

---

## 7 · 审查修复

补丁写完、验证通过之后，还有一步独立审查：

> 质疑补丁，查找残留的绕过途径以及合法行为上的回归。当复杂度、不确定性或安全影响需要时，进行独立调查或审查。向一位全新的只读审查者提供发现项、仓库根目录、授权范围、仓库策略、验证模式和候选差异，使其能够在不了解作者理由或所声称测试结果的情况下，从源代码出发进行评估。如果无法委派，则以单独一轮的方式自行完成该审查。

给审查者的材料清单值得注意：有发现项、策略和 diff，**没有**作者对补丁的解释，也没有"我已经测过了"。审查者从源码出发，才能找到作者思路之外的绕过。

审查意见回来之后：

> 在修改补丁之前，先对照源代码（或在标准模式下通过聚焦执行）确认审查中提出的问题。仅在发现项和兼容性边界之内处理已确认的问题，然后针对所选模式重新执行受影响的验证关卡。当某次修订改变了安全边界或引入了新的实质性风险时，重新进行审查。确保没有残留任何临时或无关的更改。

三个动作：确认（审查意见也可能是错的）；只在范围内修（审查者顺带提到的别的问题，另开一个发现项）；改动了安全边界就重新审查。最后一句"确保没有残留任何临时或无关的更改"，指的是调试时加的日志、临时放宽的检查、注释掉的代码。

### 7.1 最终回复应包含的内容

`fix-finding` 规定最终回复包括：

- 结果：`fixed`、`no_change` 或 `blocked`；
- 具体的漏洞路径、安全不变式，以及必须保留的合法行为；
- 所选的补丁策略，以及为什么它是最窄且完整的、符合仓库原有风格的方案；若为阻塞状态，则说明尚未解决的产品决策；
- 已更改的文件；
- 新增的测试或验证产物；
- 运行过的命令及其通过、失败或未知的结果；
- 原有漏洞路径已被封堵的证据，并区分源代码评估与实际观察到的执行；
- 明确说明是如何证明合法行为保持完好的；
- 剩余的不确定性或被跳过的验证（如有）。

这份清单可以直接当安全修复 PR 的描述模板。其中"为什么是最窄且完整的"和"如何证明合法行为保持完好"最常被省略，也最能帮评审者判断补丁质量。

---

## 8 · 阶段分离：Generate、Apply、Verify

在 Codex Security 的工作台里，修复被拆成三个阶段，由不同的请求触发：

![图 3 · 工作台里的修复分三个阶段，每个阶段只有一种权限](/images/harness/codex/ch05/fig3-stages.png)

这种分离的意义在于每个阶段只有一种权限：生成的阶段不能改真实代码，应用的阶段不能顺手"修一下"补丁，验证的阶段不能改代码让验证通过。委派时，父线程只做编排，不重复工作进程的编辑，也不把一条聊天回复当成阶段完成。

Generate 阶段怎样隔离 Git 钩子、过滤器和配置，放到[第七章](../07-report-and-tooling/)讲。

---

## 9 · verify-fix：复核一个已有的修复

### 9.1 什么时候用

`verify-fix` 只在明确请求"验证某个安全修复是否修好了已报告的漏洞"时使用。"请求实现修复或运行其测试本身并不等于请求本技能。"它的场景是：别人提交了修复，或者修复已经合并了一段时间，现在要确认它确实有效。

它是纯只读的：

> 判断所提供的每个安全发现项（finding）在当前检出（checkout）的代码中是否已被修复。以独立的纯验证模式运行；不要创建、修改或删除仓库文件，不要应用补丁、提交变更、写入产物或修改问题跟踪系统。

### 9.2 工作流

1. 确立原始漏洞、其前提条件、受影响的安全边界，以及必须继续正常工作的合法行为。
2. 确认当前检出的代码包含受影响的组件。"追踪被移动或重构的代码，而不是把缺失的文件、被删除的行或被更改的函数名当作已修复的证明。"
3. 在当前实现中追踪原始利用路径，并检查最接近的相关控制、等效路径以及可能的绕过方式。
4. 在标准模式下，仅当原始复现程序、聚焦的回归检查或合法行为检查能够在不修改仓库的情况下运行时，才运行它们。
5. 为每个发现项返回一个结果。"已关闭的工单、无关的通过测试，以及新扫描中未出现该发现项，都不足以作为证明。"

第 2 步和第 5 步列出的"不算证明"，在实际复核中都很常见：

| 看起来像修好了 | 为什么不算 |
|---|---|
| `DownloadAttachment` 里那几行检查不见了 | 可能只是改名或挪到了别的文件，逻辑原样保留 |
| 出问题的函数被删了 | 同样的逻辑可能被内联到了调用方，或者像 `ExportNote` 那样本来就有副本 |
| 工单状态是"已关闭" | 工单状态是人改的，不是代码的属性 |
| CI 全绿 | 通过的测试可能根本没有覆盖这条攻击路径 |
| 新一轮扫描没有报它 | 扫描漏报和漏洞消失无法区分 |

以 C2 为例，复核时要做的是：找到当前所有读附件的入口（下载、导出，以及修复之后有没有新增的入口），确认它们都经过 `s.openAttachment`；确认它以租户目录为根；确认参数在校验之后没有再被解码；确认汇点用的是 `openAttachment` 返回的文件，而不是又按路径打开了一次；在可行时，用原始的攻击输入和合法输入各请求一次。

### 9.3 三种结论

| 结论 | 条件 |
|---|---|
| `fixed` | 证据证明原始安全边界已被关闭，**且**合法行为保持完好 |
| `still_vulnerable` | 证据证明原始漏洞路径仍然可达 |
| `inconclusive` | 仓库不匹配、缺少原始上下文、所选模式要求的检查不可用、合法控制未被证明，或其他重大证明缺口 |

注意 `fixed` 的条件包括合法行为。一个把附件下载功能整个关掉的"修复"，封住了漏洞，但合法控制没有被证明，结论是 `inconclusive`，而不是 `fixed`。

最后一句规则："绝不要通过削弱只读边界、替换成另一个漏洞或隐瞒缺失的证据来推断出更强的结论。"复核者不能为了确认而顺手改代码，也不能因为"至少另一个问题修好了"就给出 `fixed`。

---

## 10 · 小结

| 步骤 | 关键动作 |
|---|---|
| 修之前 | 写出源到汇的路径、不变式、共享边界、必须保留的行为 |
| 修的时候 | 修在所有路径都经过的那个点；不安全的输入直接拒绝，不做"修正" |
| 修完之后 | 用四个问题质疑补丁；攻击用例、另一类恶意输入、合法对照、已有测试都要跑 |
| 交给别人 | 审查者只拿发现项和 diff；审查意见先确认再改 |
| 复核时 | 函数改名、工单关闭、测试全绿、扫描不报，都不算证明 |

单个漏洞修好了，但如果同类问题在代码库里反复出现，就说明问题出在结构上。[下一章](../06-patch-risk-hardening/)讲怎样评估一个补丁合并的风险、怎样处理依赖告警，以及什么时候该从单点修复上升到结构性加固。

---

## 自查清单

- [ ] 能说出补丁约定的四件事，并解释"最窄的共享边界"中"窄"和"共享"各指什么
- [ ] 知道为什么遇到 `../` 应该拒绝，而不是删掉后继续处理
- [ ] 提交安全修复前，会过一遍"调用方 / 条件两面 / 相邻路径 / 新拒绝的正常输入"四个问题
- [ ] 回归测试里同时有攻击用例和合法对照用例
- [ ] 修复导致旧测试失败时，不会通过放宽测试用户的权限来让测试通过
- [ ] 请人审查安全修复时，只给发现项和 diff，不先讲自己的理由
- [ ] 复核修复时，知道哪五种"看起来像修好了"的情况不能当证明

## 练习

1. 在你的项目里找一个"拼文件路径"的函数，列出它的所有调用方，用 `os.Root` 或 `filepath.IsLocal` 改写，并写一张同时包含攻击用例和合法对照用例的测试表。
2. 找一个你项目里已经合并的安全修复，按第 9.2 节的五步复核一遍，给出 `fixed`、`still_vulnerable` 或 `inconclusive`，并写出依据。
