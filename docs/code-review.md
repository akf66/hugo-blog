# OrcaCode Review

每个 Pull Request 都会由 [OrcaCode Review](https://github.com/Continuum-AI-Corp/orca-code-review)
自动审查,发现的问题以行内评论的形式贴在 PR 上。

## 严重级别

| 级别 | 含义 | 行为 |
| --- | --- | --- |
| `P0` | 严重 | ❌ `review` 检查失败,阻止合并 |
| `P1` | 高 | ❌ `review` 检查失败,阻止合并 |
| `P2` | 建议 | 💬 只贴评论,不阻止合并 |

## 配置在哪

审查设置(模型、审查模式、严重级别规则、rubric)都在
**OrcaRouter → Apps → OrcaCode Review** 控制台里改,不需要动 workflow 文件。

`.github/workflows/orca-code-review.yml` 里只写了一项覆盖:

- `auto-review-authors: "OWNER,MEMBER,COLLABORATOR,CONTRIBUTOR"`

本仓库是公开仓库,workflow 跑在 `pull_request_target` 上并带着 `ORCAROUTER_API_KEY`,
所以只有已知贡献者的 PR 会自动触发付费审查,陌生人的 PR 不会消耗额度。

## 手动触发

维护者(OWNER / MEMBER / COLLABORATOR)可以在 PR 里评论以下任一命令重新触发审查:

```
/orcacode-review
```

其他人评论无效——否则任何参与者都能不推新提交就烧掉付费额度。

## diff 太大

超过 `max-diff-kb`(默认 512)或 `max-diff-files`(默认 300)时,审查会跳过并让
`review` 检查失败(`on-oversized-diff` 默认 `fail`)。拆分 PR,或在控制台调高上限。
