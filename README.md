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
