---
title: "Codex Security 03 · 找漏洞：从入口追到汇点"
date: 2026-11-12T09:20:00+08:00
description: "Codex Security 怎样在源码里找漏洞：标准扫描同时跑一个不带任何假设的独立基线审计和一组按威胁模型分组的专项调查，候选项要写清 source、control、sink 和反证才能交给验证。本章逐条讲基线审计的检查清单、专项调查员的追踪技巧、发现阶段的证据标准和源码密钥的判断方法，用 notes-api 里的五个候选漏洞做示例，最后讲怎样把 diff 扫描的规则用到日常 Code Review。"
tags:
  - Harness
  - Codex
  - 安全开发
  - 代码审计
---

> **版本基线**：同[第一章](../01-overview/)：`openai/codex-security` main `2911786`（2026-11-12）。

[上一章](../02-policy-threat-model/)为 notes-api 写好了 `SECURITY.md` 和威胁模型，列出了六个威胁场景。这一章拿着这些场景进入代码：从接受不可信输入的入口出发，或者从敏感操作倒推回去，找出控制缺失、可被绕过、或者检查了错误对象的地方。

<!--more-->

本章要点：

- **两条路同时走**：标准扫描先启动一个独立基线审计员，它不拿父级的任何假设，自己从入口追到敏感操作；同时父级按威胁模型把问题分组成"调查包"，交给专项调查员深挖。前者防止被威胁模型带偏，后者保证重要边界被认真查过。
- **按代码位置决定查什么**：对象读取和状态变更处查鉴权、归属、租户隔离和兄弟操作之间的差异；查询构造器、渲染器、解释器处查注入；文件和网络边界查穿越、上传、重定向、SSRF。
- **核心技巧是比对"检查的值"和"使用的值"**：找出敏感操作实际使用的对象、值或目标，和守卫检查的东西逐一比较；追踪中间的解码、重定向、对象选择和状态转换；确认检查发生在第一个敏感副作用之前。
- **候选项要带证据和反证**：来源、最接近的控制、汇点、影响、反证、仍需验证的内容。危险 API、崩溃、缺失的检查本身只是线索。证据不全的合理候选项交给验证阶段，不编造证据，也不因为没有复现就丢掉。
- **源码里的密钥是一条单独的泄露路径**：攻击者是能读源码的人，不需要代码被执行。按格式和用法判断用途，测试文件名不是反证，发现的凭据绝不试用。
- **diff 审查的边界**：每个变更文件都审，包括被删除的文件；共享控制改了要查所有调用方；但不要扩展成整仓库审计。

---

## 0 · 阅读说明

- 涉及的技能：`security-scan`（标准扫描入口）、`finding-discovery`（发现阶段）、`security-diff-scan`、`deep-security-scan`。涉及的共享文档：`core-scan.md`、`baseline-auditor.md`、`focused-investigator.md`。
- 本章的 Go 代码都是 notes-api 的示例，故意写成有漏洞的版本，代码里用注释标出 source、control、sink。修复放在[第五章](../05-fix-verify/)，这里只指出问题。
- 示例里的 `Principal` 是鉴权中间件从令牌里解析出的调用者身份，`principalFrom(r)` 从请求上下文取出它。路由用 Go 1.22 起支持的 `r.PathValue`。

```go
// 示例：notes-api 里几个示例共用的类型
type Note struct {
	ID       string `json:"id"`
	TenantID string `json:"tenant_id"`
	OwnerID  string `json:"owner_id"`
	Title    string `json:"title"`
	Body     string `json:"body"`
}

// Principal 是鉴权中间件从令牌里解析出的调用者身份。
type Principal struct {
	TenantID string
	UserID   string
}

type NoteStore interface {
	Get(ctx context.Context, id string) (*Note, error)
	Delete(ctx context.Context, id string) error
}

type QuotaService interface {
	Charge(ctx context.Context, tenantID string, bytes int64) error
}

type Server struct {
	store          NoteStore
	quota          QuotaService
	attachmentsDir string
	httpClient     *http.Client
}
```

---

## 1 · 出发点：不可信输入和行使权限的代码

`security-scan` 的第一段规定了从哪里开始：

> 查找攻击者真正能够利用的漏洞（vulnerability）。从接受不可信输入或行使权限（authority）的代码入手。沿着它追踪到其所控制的操作。检查真实的强制执行点（enforcement point）、它的调用方，以及能够证明可疑攻击被阻止的最有力证据。对于暴露的凭据（credential），相关路径是谁能读取它以及它授予了何种访问权限。

几个关键词：

- **"攻击者真正能够利用的"**：目标不是列出所有不规范的写法，而是能被利用的漏洞。
- **"接受不可信输入或行使权限的代码"**：两类起点。HTTP handler、消息消费者、文件解析器是前者；删除、转账、发邮件、调用云 API 是后者。
- **"能够证明可疑攻击被阻止的最有力证据"**：审查者要主动去找反证，而不是只收集支持漏洞的证据。
- **凭据是个例外**：它的"路径"不是数据流，而是"谁能读到它、读到之后能干什么"。第 6 节单独讲。

`core-scan.md` 的开头说法类似，多了一句方法上的提示：

> Start at an exposed entry point or a sensitive operation, trace who controls the data, and check whether the expected control holds. Use the actual code to decide which leads to pursue.

（从暴露的入口或敏感操作开始，追踪谁控制着数据，检查预期的控制是否成立。根据实际代码决定追哪条线索。）

---

## 2 · 标准扫描怎样组织：独立基线 + 专项调查

### 2.1 八个步骤

`core-scan.md` 的 Core Workflow 有八步，见图 1。

![图 1 · 标准扫描的八步：独立基线和威胁模型驱动的调查并行，在检查点汇合](/images/harness/codex/ch03/fig1-core-scan.png)

第 2 步对基线审计员的要求很严格：

> start an independent baseline audit with `fork_turns: "none"`. ... Keep parent hypotheses and other findings out of the assignment.

（用 `fork_turns: "none"` 启动独立基线审计。……不要把父级的假设和其他发现项放进任务里。）

给它的只有：目标、范围和文件清单、用户上下文、威胁模型、继承的策略、严重性标准、离线搜索命令。父级已经怀疑哪里有问题，它不知道。没有条件委派时，父级自己按基线说明做一遍，并"record that the audit lacked an independent reviewer"（记录这次审计缺少独立审查者）。

第 4 步的调查包是按"谁攻击、保护什么、从哪进、本该被什么挡住、最终做了什么"来组织的，"Keep distinct boundaries and mechanisms visible"——不同的边界和机制不要混在一起。

第 6 步要求候选项在合并或验证**之前**先保存下来，被拒绝的候选项也要保留"the candidate identity, original evidence, and source-backed counterevidence"（候选项标识、原始证据和有源码支撑的反证）。这样以后有人问"这里你们查过没有"，能拿出记录。

### 2.2 为什么两条路都要走

| | 独立基线审计 | 威胁模型驱动的专项调查 |
|---|---|---|
| 出发点 | 代码本身：入口、边界、敏感操作 | 威胁模型列出的场景 |
| 优势 | 不受预设影响，能发现建模时没想到的攻击面 | 对重要边界查得深，能追到兄弟操作和具体实现 |
| 弱点 | 广而浅，可能在重要边界上停得太早 | 被模型框住，模型漏了的地方就不会去看 |

两者的结果在第 6 步汇合。如果基线审计员报了一个威胁模型里没有的问题，往往说明模型漏掉了一条边界。

### 2.3 合并的规则

多个审查者可能报了同一个问题，也可能报了看起来相似、其实不同的问题。`core-scan.md` 规定：

> Group observations only when they share the broken control and effective remediation, preserving affected routes, operations, and locations; a shared CWE alone is not a reason to merge.

（只有共享同一个失效控制、同一个有效修复方式的观察才合并，并保留受影响的路由、操作和位置；仅仅 CWE 相同不是合并的理由。）

notes-api 里 C1（`DeleteNote` 没查归属）和一个假想的"`UpdateNote` 没查归属"，如果两者都是因为各自的 handler 漏写了同一段检查，修法是在存储层统一加租户条件，可以合并成一个发现项、列两个位置；如果 `UpdateNote` 是因为调用了另一个绕过存储层的批量更新函数，修法不同，就要分开。

---

## 3 · 基线审计员的检查清单

`baseline-auditor.md` 按代码位置列出了要查的漏洞类别：

| 代码位置 | 要查的问题 |
|---|---|
| 对象读取和状态变更 | 身份认证、归属、租户隔离，以及**兄弟操作之间的授权差异** |
| 查询构造器、渲染器、解释器 | SQL / NoSQL 注入、XSS、模板注入、命令注入、不安全的反序列化 |
| 文件和网络边界 | 路径穿越、不安全的上传、开放重定向、SSRF、敏感数据暴露 |
| XML 解析器 | 外部实体解析（XXE） |
| HTTP 中间层 | 请求走私 |
| 对象合并 | 原型污染 |
| 攻击者控制的大小或生命周期 | 资源耗尽、内存安全错误 |

紧接着一句：

> Use these examples as starting points. Establish the reachable path, effective control, and consequence for each finding.

（把这些例子当作起点。对每个发现项确立可达路径、有效的控制和后果。）

清单的价值在于"看到什么代码就想到查什么"，但它不是穷举。另外，基线审计员必须显式检查嵌入的凭据和私钥，范围包括配置、文档、测试、示例和不活跃的代码。

---

## 4 · 专项调查员的追踪技巧

`focused-investigator.md` 只有一段核心方法，但几乎每句话都是一个技巧：

> Trace forward from attacker-controlled input or backward from a sensitive operation. Identify the exact object, value, or destination the operation uses and, for an interpreter, the grammar it applies. Compare that with what the guard checked. Follow decoding, redirects, object selection, or state transitions when they can change the result, and verify the check occurs before the first sensitive effect.

（从攻击者可控的输入正向追踪，或从敏感操作反向追踪。找出操作实际使用的对象、值或目标，如果是解释器，还要找出它应用的语法。拿它和守卫检查的东西比较。当解码、重定向、对象选择或状态转换可能改变结果时，追踪它们，并确认检查发生在第一个敏感副作用之前。）

前面还有两句关于调查范围的：

> Check sibling operations and concrete implementations that share the suspected control, including differences in authorization or state transitions. Preserve independently vulnerable paths and continue beyond the first issue when assigned questions remain.

（检查共享可疑控制的兄弟操作和具体实现，包括授权或状态转换上的差异。保留各自独立存在漏洞的路径，分配的问题还没查完时，找到第一个问题后继续。）

以及：如果原假设被排除了，但路径上的证据指向另一个失效的控制，就顺着追下去，"Record the disposition of the original question separately from any new finding"（原问题的结论和新发现分开记录）。

下面用 notes-api 的四个候选项，逐个对应这些技巧。

### 4.1 C1 兄弟操作：同一个资源，不同的检查

对应技巧："Check sibling operations ... including differences in authorization"。

```go
// 示例：C1 —— GET 查了归属，DELETE 没查

// GET /notes/{id}
func (s *Server) GetNote(w http.ResponseWriter, r *http.Request) {
	p := principalFrom(r)
	n, err := s.store.Get(r.Context(), r.PathValue("id"))
	if err != nil {
		http.NotFound(w, r)
		return
	}
	// 控制：租户 + 归属，两项都查
	if n.TenantID != p.TenantID || n.OwnerID != p.UserID {
		http.NotFound(w, r)
		return
	}
	json.NewEncoder(w).Encode(n)
}

// DELETE /notes/{id}
func (s *Server) DeleteNote(w http.ResponseWriter, r *http.Request) {
	// source：路径里的 id，任何登录用户都能填
	// control：无。GetNote 里的租户/归属检查在这里不存在
	// sink：store.Delete 按 id 删除，不带租户条件
	if err := s.store.Delete(r.Context(), r.PathValue("id")); err != nil {
		http.Error(w, "delete failed", http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}
```

只审 `GetNote` 的人会得出"笔记接口做了租户隔离"的结论。把同一资源的所有操作（读、改、删、导出、分享、挂附件）列成一张表，逐个看它们的检查，差异一眼就出来。

审查时还要沿 `store.Delete` 往下看一层：如果 SQL 是 `DELETE FROM notes WHERE id = $1 AND tenant_id = $2`，那 handler 层虽然没检查，存储层兜住了租户隔离，剩下的问题只是"同租户内能删别人的笔记"，严重性不同。这就是"检查真实的强制执行点"的意思：控制不一定在你以为的那一层。

### 4.2 C2 检查的值 ≠ 使用的值

对应技巧："Identify the exact object, value, or destination the operation uses ... Compare that with what the guard checked. Follow decoding ..."。

```go
// 示例：C2 —— 校验原始参数，使用解码并拼接后的路径

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

图 2 把检查的值和使用的值按顺序排了出来。守卫看的是 `raw`，`ServeFile` 打开的是 `full`，中间隔着一次解码和一次路径规整。这些值都来自我写的测试：第一步的值能通过检查，最后一步落在另一个租户的目录下。

![图 2 · C2：守卫检查的是 raw，ServeFile 打开的是两步之后的 full](/images/harness/codex/ch03/fig2-c2-values.png)

这里还有一个容易被当成反证的东西：`http.ServeFile` 的文档说它会拒绝 `r.URL.Path` 含 `..` 的请求。但它检查的是**请求路径**（这里是 `/attachments`），不是传给它的 `name` 参数。一个真实存在的控制，检查的却是另一个值，不能算反证。这正是 `finding-discovery` 证据标准里说的"被应用到了错误的值或边界上"。

### 4.3 C3 检查的目标 ≠ 实际连接的目标

对应技巧："Identify the exact ... destination the operation uses ... Follow ... redirects"。

```go
// 示例：C3 —— 只检查了第一次解析的地址

func checkPublicHost(ctx context.Context, host string) error {
	ips, err := net.DefaultResolver.LookupIPAddr(ctx, host)
	if err != nil {
		return err
	}
	for _, ip := range ips {
		if ip.IP.IsLoopback() || ip.IP.IsPrivate() ||
			ip.IP.IsLinkLocalUnicast() || ip.IP.IsUnspecified() {
			return errBlockedHost
		}
	}
	return nil
}

// POST /previews  {"url": "..."}
func (s *Server) PreviewURL(w http.ResponseWriter, r *http.Request) {
	var req struct {
		URL string `json:"url"`
	}
	if err := json.NewDecoder(io.LimitReader(r.Body, 4096)).Decode(&req); err != nil {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	u, err := url.Parse(req.URL)
	if err != nil || (u.Scheme != "http" && u.Scheme != "https") {
		http.Error(w, "bad url", http.StatusBadRequest)
		return
	}
	// control：在发请求之前解析一次主机名，检查解析结果
	if err := checkPublicHost(r.Context(), u.Hostname()); err != nil {
		http.Error(w, "forbidden", http.StatusForbidden)
		return
	}
	// sink：Client 拨号时会重新解析 DNS；遇到 3xx 会自动跟随（默认最多 10 次），
	// 跳转后的地址完全没有经过 checkPublicHost
	resp, err := s.httpClient.Get(u.String())
	if err != nil {
		http.Error(w, "fetch failed", http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
	json.NewEncoder(w).Encode(map[string]string{"title": extractTitle(body)})
}
```

`checkPublicHost` 本身写得不差，覆盖了回环、私有网段、链路本地（`169.254.0.0/16`，云元数据服务就在这里）。问题在于它检查的不是最终连接的地址：

1. **重定向**：攻击者提交 `https://attacker.example/a`，它解析到公网地址，通过检查；这个页面返回 `302 Location: http://169.254.169.254/latest/meta-data/`。`http.Client` 在 `CheckRedirect` 为空时默认跟随最多 10 次重定向，第二跳的地址从没被检查过。
2. **再次解析**：`checkPublicHost` 调了一次 `LookupIPAddr`，`httpClient.Get` 拨号时又解析一次。攻击者控制的域名可以第一次返回公网地址、第二次返回内网地址（DNS rebinding）。检查和使用之间隔着一次独立的解析。

攻击者能不能拿到响应内容，取决于 `extractTitle` 返回什么：这里会把 `<title>` 的内容回显给调用者，元数据服务的响应没有 `<title>`，所以拿不到正文。但按 notes-api 的 `SECURITY.md`，"从服务端发起到内网或元数据服务的请求可报告，即使拿不到响应内容"。

第五章的修法会把检查放到实际拨号的地方（`net.Dialer.Control` 检查真正要连接的 IP），并对每一跳重定向重新检查。

### 4.4 C4 副作用先于检查

对应技巧："verify the check occurs before the first sensitive effect"。

```go
// 示例：C4 —— 先写盘，再查配额，再查归属

// POST /notes/{id}/attachments
func (s *Server) UploadAttachment(w http.ResponseWriter, r *http.Request) {
	p := principalFrom(r)
	noteID := r.PathValue("id")

	dst := filepath.Join(s.attachmentsDir, p.TenantID, randomName())
	f, err := os.Create(dst)
	if err != nil {
		http.Error(w, "storage error", http.StatusInternalServerError)
		return
	}
	// 第一个敏感副作用：把请求体整个写进磁盘，没有大小上限
	n, err := io.Copy(f, r.Body)
	f.Close()
	if err != nil {
		os.Remove(dst)
		http.Error(w, "upload failed", http.StatusBadRequest)
		return
	}

	// 写完之后才检查配额
	if err := s.quota.Charge(r.Context(), p.TenantID, n); err != nil {
		os.Remove(dst)
		http.Error(w, "quota exceeded", http.StatusRequestEntityTooLarge)
		return
	}
	// 再之后才检查这条笔记是不是调用者的
	note, err := s.store.Get(r.Context(), noteID)
	if err != nil || note.TenantID != p.TenantID || note.OwnerID != p.UserID {
		os.Remove(dst)
		http.NotFound(w, r)
		return
	}
	w.WriteHeader(http.StatusCreated)
}
```

三个检查都在，顺序错了。审查时把 handler 里的语句分成两类：有副作用的（写盘、写库、发消息、扣费）和检查。只要有一个副作用出现在某个检查之前，就要问：检查失败时，这个副作用能不能完全撤销？

这里的回答是不能：

- `io.Copy` 没有上限，一个请求可以写满磁盘，配额检查要等写完才执行。这影响所有租户（`SECURITY.md` 的"有界解析"不变式）。
- 失败时的 `os.Remove` 是事后补救。删除失败、进程在中间崩溃、并发请求同时写入，都会留下文件。

修法是把顺序倒过来：先查归属，再用 `http.MaxBytesReader` 限制大小、按 `Content-Length` 或上限预扣配额，最后才写盘。

### 4.5 四个候选项放在一起看

![图 3 · 四个候选项的共同点：守卫检查的，不是汇点实际使用的](/images/harness/codex/ch03/fig3-checked-vs-used.png)

---

## 5 · 发现阶段的证据标准

`finding-discovery` 的"证据标准"一节有六条，每条都在回答"一个候选项至少要有什么"。

**第一条：从输入经过控制追到汇点和影响，并说明控制为什么不起作用。**

> 将攻击者可控的输入或特权操作，经由最接近的相关控制，追踪到汇点（sink）和安全影响。解释为什么该控制缺失、可被绕过，或被应用到了错误的值或边界上。

控制失效有三种形式：缺失（C1）、可被绕过（C3 的重定向）、检查了错的值或边界（C2）。

**第二条：查运行时可达性和部署假设，记录反证。**

> 检查运行时可达性和部署假设。将有效的防护和隔离措施记录为反证（counterevidence）；某个同级实现是安全的，并不能证明候选路径是安全的。

C1 的反证候选是存储层的 SQL 有没有租户条件；C3 的反证候选是部署环境有没有出站防火墙。查到了就记下来，查不到就作为验证阶段要确立的事项。

**第三条：保留理解主张需要的每个位置。**

> 保留根控制位置，以及理解该主张所需的入口点、包装层、具体实现和汇点。当主张涵盖多个实现时，检查这些实现，而不是断言它们全部受影响。

**第四条：独立的实例分开，重复的观察合并。**

> 保持各自可独立到达的漏洞实例可单独定位。对于同一来源/控制/汇点/影响的重复观察结果进行合并，而不是拆分出仅表面不同的变体；合并时保留所有受影响位置和原始候选项 ID。

**第五条：用户给的线索要追到底。**

> 将用户或告知中明确给出的线索（seed）追踪到其确切的来源和控制。记录支持、否定或使该线索仍未解决的证据；同一类别中的邻近问题并不能关闭该线索。

有人报告"附件下载可能有路径穿越"，你在上传接口找到了一个路径问题，这不能关闭关于下载接口的线索。

**第六条：线索不是发现项，但也别丢掉。**

> 区分具体的安全影响与可维护性或健壮性问题。崩溃、危险 API、可选功能或缺失的检查本身只是线索，而不是发现项（finding）。将带有明确证明缺口的合理候选项交给验证阶段，而不是编造证据，或因缺少可运行的复现而丢弃它们。

最后一句是发现阶段和验证阶段的分工：发现阶段只负责"技术上合理"，证据不全没关系，写清楚缺什么，交给验证阶段去补。

### 5.1 候选项要记录什么

`finding-discovery` 规定每个候选项保留：

- 稳定的候选项 ID 和标题。
- 受影响的文件和行，需要时标注 `entrypoint/wrapper`、`root_control`、`sink`、`concrete_implementation`。
- 攻击者可控的来源、最接近的控制、汇点或失效的控制、可能的影响，以及串起它们的源码证据。
- 反证、部署前提条件、未解决的问题，以及验证阶段需要确立的内容。
- 已知的 CWE ID，否则留空列表。

用 C1 写一份候选项记录（示例，行号是示意）：

```yaml
candidate_id: notes-delete-idor
title: DeleteNote 缺少租户与归属检查，可删除其他租户的笔记
cwe: [CWE-639]
locations:
  entrypoint: internal/http/routes.go:41      # DELETE /notes/{id} → DeleteNote
  root_control: internal/http/notes.go:58-66  # DeleteNote，没有任何归属检查
  sink: internal/store/notes.go:112           # Delete：DELETE FROM notes WHERE id = $1
  sibling_reference: internal/http/notes.go:30-36  # GetNote 的检查，作为对照
source: 路径参数 id，任意已登录用户可控
control: 无（GetNote 有 TenantID/OwnerID 比对，DeleteNote 没有）
sink: store.Delete 按 id 删除，SQL 不含 tenant_id 条件
impact: 任意注册用户可删除其他租户的笔记（违反 SECURITY.md 不变式 1）
counterevidence:
  - 笔记 id 为 UUIDv4，无法枚举；但 id 会出现在分享链接和导出文件里
proof_gaps:
  - 未确认 store.Delete 之外是否还有软删除或审计恢复机制
  - 未确认网关层是否对 DELETE 方法另有限制
validation_needs:
  - 确认攻击者获取他人笔记 id 的现实途径
  - 确认删除是否可恢复（影响严重性）
```

注意"id 不可枚举"被记成了反证，同时写明它为什么不能推翻主张。第四章会讲验证阶段怎样处理这条反证。

---

## 6 · 源码里的密钥

`core-scan.md` 专门有一节"Secrets Exposed In Source"：

> Review embedded credentials and private keys in code, configuration, documentation, tests, examples, and inactive code. A source reader obtaining a credential with authority beyond reading the source is a disclosure path; the containing code need not execute or accept attacker input.

（审查嵌入在代码、配置、文档、测试、示例和不活跃代码里的凭据和私钥。能读源码的人拿到一个权限超出"读源码"的凭据，就是一条泄露路径；包含它的代码不需要被执行，也不需要接受攻击者输入。）

这里攻击者的起始能力是"能读源码"（对应第二章的场景 S5），新能力是凭据授予的权限。所以测试代码、注释掉的代码、文档里的凭据都在范围内。

怎样判断一串字符是不是真凭据：

> Establish purpose from format and usage. Distinguish credential material from public keys, certificates, identifiers, secret-store references, and demonstrable dummy values. An opaque string is not sufficient evidence, and a test filename is not sufficient counterevidence.

（根据格式和用法确定用途。把凭据和公钥、证书、标识符、密钥存储引用、可证明的占位值区分开。一串看不懂的字符串不足以作为证据，测试文件名也不足以作为反证。）

以及绝对不能做的事：

> Never use discovered credentials or contact services to validate them.

（绝不使用发现的凭据，也不联系服务去验证它们。）

用 C5 演示这套判断。notes-api 的 `internal/storage/testdata/` 里有一个夹具：

```go
// 示例：C5 —— internal/storage/testdata 里的凭据
var docExampleCreds = struct {
	AccessKeyID, SecretAccessKey string
}{
	AccessKeyID:     "AKIAIOSFODNN7EXAMPLE",
	SecretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
}
```

这两个值是 AWS 官方文档里的示例占位符，以 `EXAMPLE` 结尾，属于"demonstrable dummy values"，不是发现项。

换一个情况：同一个目录下的 `s3_integration_test.go` 里，`AccessKeyID` 是一串 `AKIA` 开头、20 个字符、没有 `EXAMPLE` 字样的值，配套的 `Bucket` 字段是 `notes-api-prod-attachments`，测试在设置了 `INTEGRATION=1` 时会真的连到这个桶。判断过程：

| 问题 | 证据 | 结论 |
|---|---|---|
| 格式像什么 | `AKIA` 前缀 + 20 字符，配一个 40 字符的密钥 | 长期访问密钥的格式 |
| 怎么被使用 | 传给 S3 客户端，连接名字带 `prod` 的桶 | 用途是访问生产附件存储 |
| 是不是占位值 | 不是文档示例值，没有 `EXAMPLE`、`dummy` 之类的标记 | 不能证明是占位值 |
| 在测试文件里算不算反证 | "a test filename is not sufficient counterevidence" | 不算 |
| 现在还有效吗 | 不知道，也不去试 | 作为证明缺口，影响置信度，不影响"已暴露"这个事实 |

`core-scan.md` 对最后一行的态度很明确："Unknown validity or rotation limits confidence and impact claims."（有效性或轮换情况未知，会限制置信度和影响的主张。）`baseline-auditor.md` 的说法是 "Unknown live validity limits confidence, not the evidence of exposure."（不知道是否仍有效，限制的是置信度，不是暴露的证据。）

处理方式：报告位置、凭据类型、相关上下文和不确定性，然后走轮换流程。不要为了确认严重性去调一次 `ListBuckets`。

---

## 7 · 把 diff 扫描用到日常 Code Review

`security-diff-scan` 针对的是 PR、提交、分支差异和本地改动，这正是大多数开发者每天做的事。它的开头两句话就定下了范围：

> 审查每一个已变更的文本文件，包括已删除的文件。沿着已变更的行为追踪到支持代码中，但不要扩展成与之无关的整仓库审计。

### 7.1 范围：哪些文件要看

> 清单（inventory）包含所有已变更的文本文件，与文件名、扩展名或目录无关。已跟踪的变更即使被忽略也仍在范围内；本地补丁还包括未被忽略的未跟踪文件。在基线修订版本中检查已删除的文件。根据行为和可达性评估安全相关性，包括构建、部署和 CI 中的使用。

拆成可以直接照做的规则：

- **不按文件名挑**。`.yaml`、`.sh`、`Dockerfile`、`.github/workflows/*.yml` 都要看。CI 配置的变更可能把密钥暴露给 fork 来的 PR，Dockerfile 的变更可能让容器以 root 运行。
- **被删除的文件要看它原来的内容**。删掉一个文件可能就是删掉了一个控制。
- **被 `.gitignore` 忽略但已跟踪的文件**，改了也要看。

### 7.2 深度：追到哪里为止

`finding-discovery` 对 diff 的要求：

> 对于 diff，检查实际变更的行为，包括基线修订版本中已删除的文件。沿着支持代码追踪到足以理解受影响边界的程度；当已变更的共享控制影响到同级调用方时，扩展到这些调用方。不要扩大成与之无关的整仓库审计。

规则是：**改了什么行为，就追到这个行为的边界为止**。如果改的是一个被多处调用的共享控制，边界就包括所有调用方。

**例子一：改了共享的跳转校验。** PR 的描述是"支持登录后跳转到子域名"：

```diff
 func isAllowedRedirect(target string) bool {
 	u, err := url.Parse(target)
 	if err != nil {
 		return false
 	}
-	return u.Host == "" && strings.HasPrefix(u.Path, "/") && !strings.HasPrefix(u.Path, "//")
+	if u.Host == "" {
+		return strings.HasPrefix(u.Path, "/") && !strings.HasPrefix(u.Path, "//")
+	}
+	return strings.HasSuffix(u.Host, "notes.example.com")
 }
```

只看 diff 本身，能看出 `HasSuffix` 会接受 `evilnotes.example.com`（我把这个也写进了测试）。但 diff 审查的要求是继续追：这个函数被谁调用？在 notes-api 里搜一下，调用方有三个。同一个改动，在前两个调用方是中低危的开放重定向，在 OAuth 回调里可能是账户接管，见图 4。

![图 4 · 改一个共享控制，要按调用方逐个评估：同一行改动在 OAuth 回调里可能是账户接管](/images/harness/codex/ch03/fig4-diff-callers.png)

**例子二：删掉了一个中间件文件。** PR 的描述是"清理不用的代码"，删除了 `internal/http/middleware/tenant_scope.go`，同时在 `routes.go` 里去掉了一行 `r.Use(middleware.TenantScope)`。

diff 里只有删除，没有新增的逻辑，很容易一扫而过。按规则要去基线版本看被删的文件原来做什么：

```go
// 示例：被删除的中间件（基线版本里的内容）
func TenantScope(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		p := principalFrom(r)
		if h := r.Header.Get("X-Tenant-ID"); h != "" && h != p.TenantID {
			http.Error(w, "tenant mismatch", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

它拒绝 `X-Tenant-ID` 头和令牌里的租户不一致的请求。接下来的问题是：现在还有没有 handler 读 `X-Tenant-ID`？如果某个旧接口仍然用这个头来选租户，删掉中间件就打开了跨租户访问。如果没有任何代码再读这个头，这个删除就是安全的清理，审查记录里写"已检查，`X-Tenant-ID` 已无消费者"。

### 7.3 不能把没看完说成没问题

`security-diff-scan` 最后一句：

> 每个变更文件和候选项都必须被说明为已审查、已解决或明确未完成；不要把不完整的审查说成"没有发现项"。

一个 80 个文件的 PR，你认真看了其中 20 个，审查意见里就写清楚看了哪 20 个，而不是一个 LGTM。

---

## 8 · deep-security-scan：什么时候需要多轮

`deep-security-scan` 不是一种新的审查方法，而是把标准扫描跑多次：

> 宿主运行多次 Standard 审计并合并其发现项（finding）。同一扫描中的 worker 可以交换调查线索，并互相核实对方的证据。后续轮次会收到已经报告过的标题和位置。当连续多次审计都没有新增内容，或达到配置的扫描次数或时间上限时，宿主停止。

两个设计值得注意：后续轮次知道已经报告了什么，所以会往别处找；停止条件是"连续几轮没有新东西"，而不是"跑完固定次数"。

它的代价也写得很直白：启动前如果发现另一个深度扫描在跑，要向用户说明额外的 CPU、内存和 token 消耗，默认推荐的选项是取消。结束时，"要区分'已完成审计但发现项为零'与'在任何审计完成之前已达到截止时间'"。

对人工审查的启发：重要的代码（鉴权、支付、多租户隔离）值得让不同的人各审一遍，第二个人拿到的是"已经发现了什么"，而不是"第一个人怎么想的"；当连续两三个人都没有新发现时再停。

---

## 自查清单：PR 安全审查

- [ ] 列出了 PR 里所有变更的文件，包括删除的文件、配置、CI、Dockerfile，没有按文件名跳过
- [ ] 对删除的文件，看了它在基线版本里的内容和它原来承担的控制
- [ ] 对每个新增或修改的入口，找到了不可信输入从哪进、经过哪些检查、最终到哪个敏感操作
- [ ] 对每个改动的共享函数（校验、鉴权、路径拼接、URL 处理），列出了所有调用方并逐个评估
- [ ] 对同一资源的兄弟操作（读、改、删、导出、分享），比较过它们的授权检查是否一致
- [ ] 对每个检查，确认它检查的值就是后面实际使用的值，中间没有解码、规整、重定向、重新解析
- [ ] 确认所有检查都在第一个副作用（写库、写盘、发请求、扣费）之前
- [ ] 检查了新增的字面量里有没有凭据，判断依据是格式和用法，不是文件名
- [ ] 对每个疑点写下了来源、控制、汇点、反证和还缺什么证据
- [ ] 审查意见里写明了看了哪些、没看哪些

## 练习

1. 在你的项目里挑一个资源（比如订单、文档、项目），把它的所有操作列成表，每行写出这个操作做了哪些授权检查、检查在哪一层。看看有没有像 C1 那样的差异。
2. 找一个接受路径、URL 或文件名参数的接口，把参数从进入 handler 到被使用的每一步变换写下来（解码、拼接、规整、重定向），确认守卫检查的是最后那一步的值。
3. 找一个最近合并的、改动了共享工具函数的 PR，补做一次调用方分析：这个函数有几个调用方？改动对每个调用方意味着什么？
