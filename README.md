# xview — Adversarial review skills for Claude Code

Two Claude Code skills for **dual-reviewer convergence** on plans and PRs:

- **`/xplanview <issue#>`** — review an implementation plan **before** writing code, persisted to a GitHub issue
- **`/xprview <PR#>`** — review an open PR's diff, auto-fix via Sonnet, persisted to PR comments + commits

Both enforce the same convergence rule:

> **Two distinct agents must PASS the same artifact for the review to converge. A single PASS doesn't count — self-review can't pass itself.**

Inspired by [`superpowers:santa-method`](https://github.com/anthropics/claude-skills) (dual-reviewer pattern), extended with GitHub persistence and per-version pass tracking.

## Why two reviewers

A single reviewer that PASSes a plan/diff is doing self-review on its own analysis. That's not enough signal. Requiring two **different model families** (Anthropic Claude Opus + OpenAI Codex CLI) to both PASS the same revision adds independent judgment — they catch different classes of bugs, and a defect that survives both is much more likely to be a real non-defect.

## What's in each skill

### `xplanview` — plan review

- **Reviewers**: Opus (subagent) + Codex (`codex exec`), strictly alternating R1/R2/R3/R4
- **Fixer**: main Claude session writes revisions itself (text only, no code)
- **Convergence**: both reviewers must PASS the same `v{N}` plan version
- **Persistence**: GitHub issue comments with HTML markers (`<!-- xplanview:plan:v1 -->`, `<!-- xplanview:review:r{N}:{reviewer}:v{V} -->`, etc.)
- **Max 4 rounds**, then ESCALATE to user

### `xprview` — PR review + auto-fix

- **Reviewers**: Codex + Opus, strictly alternating
- **Fixer**: Sonnet 4.6 subagent (separate role — never reviews, only modifies code, commits, pushes)
- **Convergence**: both reviewers must PASS the same HEAD SHA
- **Persistence**: PR comments + git commits ("xprview r{N} fix: ..."), PR description trail
- **Same-repo PRs only** (fork PRs early-escalate — no origin push access)
- **Max 4 review rounds + 3 sonnet fix rounds**

## Installation

### Requirements

- [Claude Code](https://docs.claude.com/claude-code) (skills auto-register from `~/.claude/skills/<name>/SKILL.md`)
- [`gh` CLI](https://cli.github.com/) authenticated (`gh auth status`)
- [`codex` CLI](https://github.com/openai/codex) installed (`which codex`) — used as the second reviewer
- `jq` installed
- For `xprview` only: an open same-repo PR with `npm run lint` / `npm test` available (or skip the gate)

### Install

```bash
git clone https://github.com/byron-dowsure/xview.git
mkdir -p ~/.claude/skills
cp -r xview/xplanview xview/xprview ~/.claude/skills/
```

Verify in a new Claude Code session: type `/xplan` — autocomplete should show `/xplanview`.

## Usage

### `/xplanview` — review a plan before coding

After Claude (the main session) proposes an implementation plan, run:

```
/xplanview              # auto-detect issue from current branch (e.g. cc/384-foo → #384)
/xplanview 431          # use a specific issue
/xplanview new          # create a new issue to anchor the review on
```

Claude will:

1. Post the plan to the issue as `<!-- xplanview:plan:v1 -->`
2. Run R1 Opus review → if PASS but Codex hasn't reviewed v1 yet, advance to R2 Codex on **same v1**
3. If both PASS v1 → CONVERGED, exit
4. If any FAIL → write v2 (revision) → R3 Opus on v2 → R4 Codex on v2 → ...
5. Max 4 rounds; if no 2-of-2 PASS by R4 → ESCALATE

### `/xprview` — review a PR + auto-fix

```
/xprview                # auto-detect PR from current branch
/xprview 123            # use PR #123
```

Claude will:

1. Verify PR is OPEN, same-repo, worktree clean → checkout the PR branch
2. Post baseline marker
3. Run alternating reviews:
   - R1 Codex on baseline SHA → if PASS, R2 Opus on same SHA → both PASS → CONVERGED
   - On any FAIL → Sonnet subagent reads the FAIL review, edits files, returns
   - Orchestrator runs `npm run lint && npm test` (if `package.json` declares them)
   - Commits "xprview r{N} fix: ..." and pushes
   - Loop back to next review
4. Max 4 review rounds; if no 2-of-2 PASS on the same HEAD SHA → ESCALATE

## Marker schemes (how state survives across runs)

Every comment posted by the skills starts with an HTML marker so future runs (or humans) can find specific artifacts via grep.

### xplanview (issue comments)

| Marker | Meaning |
|---|---|
| `<!-- xplanview:plan:v1 -->` | Original plan |
| `<!-- xplanview:review:r{N}:{opus\|codex}:v{V} -->` | Reviewer R reviewed plan v{V} at round N |
| `<!-- xplanview:revision:v{V+1} -->` | Revised plan after FAIL on v{V} |
| `<!-- xplanview:verdict:CONVERGED:r{N}:v{V} -->` | Terminal: both Opus and Codex PASSed v{V} |
| `<!-- xplanview:verdict:ESCALATE -->` | Terminal: 4 rounds, no 2-of-2 PASS |

### xprview (PR comments)

| Marker | Meaning |
|---|---|
| `<!-- xprview:baseline:{sha} -->` | Skill startup |
| `<!-- xprview:review:r{N}:{codex\|opus}:{sha} -->` | Reviewer R reviewed HEAD={sha} at round N |
| `<!-- xprview:fix:r{N}:{new_sha} -->` | Sonnet fix → new commit at {new_sha} |
| `<!-- xprview:verdict:CONVERGED:r{N}:{sha} -->` | Terminal: both reviewers PASSed {sha} |
| `<!-- xprview:verdict:ESCALATE -->` | Terminal failure |

## Cost & duration

Worst case per run:

| Skill | Worst-case calls |
|---|---|
| `xplanview` | 4 reviews (2 Opus + 2 Codex) + 3 main-session revisions |
| `xprview` | 4 reviews (2 Codex + 2 Opus) + 3 Sonnet fixes + lint/test |

Best case: 2 reviews (R1 + R2 both PASS same artifact), no revisions/fixes.

## License

MIT (see [LICENSE](./LICENSE))

## Acknowledgments

- [`superpowers:santa-method`](https://github.com/anthropics/claude-skills) — dual-reviewer convergence pattern
- [`everything-claude-code:codex`](https://github.com/anthropics/claude-skills) — Codex CLI integration patterns

---

# xview — Claude Code 多 agent 对抗评审 skill 集

两个 Claude Code skill，强制**双 reviewer 收敛**做计划评审和 PR 评审：

- **`/xplanview <issue#>`** — 在写代码**之前**评审实施方案，状态持久化到 GitHub issue
- **`/xprview <PR#>`** — 评审已开 PR 的 diff，由 Sonnet 自动改代码，状态持久化到 PR 评论 + commit

两者执行同一条收敛规则：

> **必须有两个不同的 agent 都对同一份 artifact 给出 PASS 才算收敛。1 个 PASS 不够 —— 自己审自己不算评审。**

灵感来自 [`superpowers:santa-method`](https://github.com/anthropics/claude-skills) 的双 reviewer 模式，扩展了 GitHub 持久化和按版本/SHA 跟踪 PASS 状态。

## 为什么要两个 reviewer

单 reviewer 给出 PASS 等于在用自己的分析评审自己的分析 —— 信号不够。要求**两个不同模型族**（Anthropic Claude Opus + OpenAI Codex CLI）对同一份修订都 PASS，才能加入独立判断 —— 它们能抓不同类别的 bug，而能同时通过两边的 defect 大概率是真的没问题。

## 两个 skill 各自做什么

### `xplanview` — 计划评审

- **Reviewer**：Opus（subagent）+ Codex（`codex exec`），严格交替 R1/R2/R3/R4
- **Fixer**：主 Claude 会话直接写修订（只动文本，不动代码）
- **收敛条件**：两 reviewer 都对同一 `v{N}` 计划版本 PASS
- **持久化**：GitHub issue 评论，HTML marker（`<!-- xplanview:plan:v1 -->`、`<!-- xplanview:review:r{N}:{reviewer}:v{V} -->` 等）
- **最多 4 轮**，未收敛则升级给用户决策

### `xprview` — PR 评审 + 自动修复

- **Reviewer**：Codex + Opus 严格交替
- **Fixer**：Sonnet 4.6 subagent（角色分离 —— 只改代码、commit、push，不参与评审）
- **收敛条件**：两 reviewer 都对同一 HEAD SHA PASS
- **持久化**：PR 评论 + git commit（"xprview r{N} fix: ..."）+ PR description 收敛轨迹
- **仅支持同仓库 PR**（fork PR 因无 origin push 权限会早期升级）
- **最多 4 轮 review + 3 轮 sonnet fix**

## 安装

### 前置要求

- [Claude Code](https://docs.claude.com/claude-code)（skill 从 `~/.claude/skills/<name>/SKILL.md` 自动注册为 slash command）
- [`gh` CLI](https://cli.github.com/) 已认证（`gh auth status`）
- [`codex` CLI](https://github.com/openai/codex) 已安装（`which codex`）—— 用作第二 reviewer
- `jq` 已安装
- 仅 `xprview` 需要：一个开着的同仓库 PR + `npm run lint` / `npm test`（没有也可，会自动跳过门禁）

### 安装命令

```bash
git clone https://github.com/byron-dowsure/xview.git
mkdir -p ~/.claude/skills
cp -r xview/xplanview xview/xprview ~/.claude/skills/
```

新开 Claude Code 会话验证：输入 `/xplan` 应能自动补全 `/xplanview`。

## 使用

### `/xplanview` —— 写代码前评审方案

主会话 Claude 提出实施方案后，运行：

```
/xplanview              # 从当前分支推断 issue（如 cc/384-foo → #384）
/xplanview 431          # 指定 issue
/xplanview new          # 新建 issue 承载评审
```

Claude 会：

1. 把方案以 `<!-- xplanview:plan:v1 -->` post 到 issue
2. R1 Opus 评审 → 若 PASS 但 Codex 还没评 v1，进 R2 让 Codex 评**同一 v1**
3. 两个都 PASS v1 → 收敛，退出
4. 任一 FAIL → 写 v2（修订）→ R3 Opus 评 v2 → R4 Codex 评 v2 → ...
5. 最多 4 轮；R4 仍未达到 2-of-2 PASS → 升级给用户决策

### `/xprview` —— 评审 PR + 自动修复

```
/xprview                # 从当前分支推断 PR
/xprview 123            # 指定 PR #123
```

Claude 会：

1. 验证 PR 是 OPEN、同仓库、本地 worktree 干净 → checkout PR 分支
2. Post baseline marker
3. 交替评审：
   - R1 Codex 评 baseline SHA → 若 PASS，R2 Opus 评同一 SHA → 双 PASS → 收敛
   - 任一 FAIL → Sonnet subagent 读 FAIL review、改文件、返回
   - 编排器跑 `npm run lint && npm test`（如 `package.json` 声明）
   - Commit "xprview r{N} fix: ..." 并 push
   - 进入下一轮评审
4. 最多 4 轮；同一 HEAD SHA 未达到 2-of-2 PASS → 升级

## Marker 协议（跨次运行如何追溯状态）

skill 发的每条评论都以 HTML marker 开头，后续运行（或人）可以用 grep 找到具体 artifact。

### xplanview（issue 评论）

| Marker | 含义 |
|---|---|
| `<!-- xplanview:plan:v1 -->` | 原始方案 |
| `<!-- xplanview:review:r{N}:{opus\|codex}:v{V} -->` | reviewer R 在第 N 轮评审了方案 v{V} |
| `<!-- xplanview:revision:v{V+1} -->` | v{V} FAIL 后的修订 |
| `<!-- xplanview:verdict:CONVERGED:r{N}:v{V} -->` | 终止：双 reviewer 都 PASS v{V} |
| `<!-- xplanview:verdict:ESCALATE -->` | 终止：4 轮内未达成 2-of-2 PASS |

### xprview（PR 评论）

| Marker | 含义 |
|---|---|
| `<!-- xprview:baseline:{sha} -->` | skill 启动 |
| `<!-- xprview:review:r{N}:{codex\|opus}:{sha} -->` | reviewer R 在第 N 轮评审了 HEAD={sha} |
| `<!-- xprview:fix:r{N}:{new_sha} -->` | Sonnet 修复 → 新 commit {new_sha} |
| `<!-- xprview:verdict:CONVERGED:r{N}:{sha} -->` | 终止：双 reviewer 都 PASS {sha} |
| `<!-- xprview:verdict:ESCALATE -->` | 终止：失败 |

## 成本与时长

每次最坏情况：

| Skill | 最坏调用次数 |
|---|---|
| `xplanview` | 4 次评审（2 Opus + 2 Codex）+ 3 次主会话修订 |
| `xprview` | 4 次评审（2 Codex + 2 Opus）+ 3 次 Sonnet 修复 + lint/test |

最佳情况：2 次评审（R1 + R2 都对同一 artifact PASS），无修订/修复。

## License

MIT（详见 [LICENSE](./LICENSE)）

## 致谢

- [`superpowers:santa-method`](https://github.com/anthropics/claude-skills) —— 双 reviewer 收敛模式
- [`everything-claude-code:codex`](https://github.com/anthropics/claude-skills) —— Codex CLI 集成参考
