---
name: xplanview
description: Use when the user types "xplanview" or "/xplanview" immediately after Claude has proposed an implementation plan that needs adversarial multi-model review with GitHub issue persistence before any code is written.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Agent
  - AskUserQuestion
---

# xplanview

Multi-round plan review. Inspired by `superpowers:santa-method` (dual-reviewer
convergence) but persists state to a GitHub issue and uses CLI-isolated reviewers
instead of in-process subagents.

## Hard rules (do not adapt)

- Round order: **R1=Opus → R2=Codex → R3=Opus → R4=Codex**. Strictly alternating.
- **Convergence requires BOTH Opus AND Codex to PASS the SAME plan version** (`v{V}`).
  A single PASS is NOT enough — one reviewer alone is "self-review" and doesn't prove
  correctness. Track which version each reviewer has passed.
- On any FAIL, revise to next version. Past PASSes on older versions no longer count;
  both reviewers must re-pass the new version.
- **Round counter ≠ version counter.** Version increments only on FAIL. If R1 Opus
  PASSes v1 (no convergence yet), R2 Codex still reviews v1.
- Max 4 review rounds total. If 2-of-2 PASS not achieved by R4 → ESCALATE. No R5.
- Reviewers must be context-isolated. Opus = `Agent` tool with `model: opus`,
  fresh subagent. Codex = `codex exec` subprocess. Never paste your own thinking
  into the reviewer prompt.
- Every comment posted to the issue starts with the exact HTML marker below.

## Marker scheme

| Marker | When |
|---|---|
| `<!-- xplanview:plan:v1 -->` | Original plan (pre-review) |
| `<!-- xplanview:review:r{N}:{opus\|codex}:v{V} -->` | Reviewer for round N reviewed plan version V |
| `<!-- xplanview:revision:v{V+1} -->` | Revised plan after FAIL on v{V} |
| `<!-- xplanview:verdict:CONVERGED:r{N}:v{V} -->` | Terminal: both Opus AND Codex PASSed v{V} (last review at R{N}) |
| `<!-- xplanview:verdict:ESCALATE -->` | Terminal: 4 rounds without 2-of-2 PASS |

Round number ≠ version number. A reviewer always reviews the **current** version
`v{V}`, regardless of `r{N}`. Version increments only when revising after FAIL.

## Phase 1 — Anchor

The user invocation may carry an argument: `/xplanview 428` (use #428),
`/xplanview new` (create new), or `/xplanview` (auto-detect). Resolve in this order:

1. **Numeric arg** → `ISSUE=<arg>`. Verify with `gh issue view "$ISSUE" --json state`.
   If state is `CLOSED`, ask user whether to reopen or pick another.
2. **`new` arg** → skip to issue creation (below).
3. **No arg** → try branch detection:
   ```bash
   BRANCH=$(git branch --show-current)
   ISSUE=$(echo "$BRANCH" | grep -oE '[0-9]+' | head -1)
   ```
   If branch yields a number AND `gh issue view "$ISSUE"` shows OPEN, use it. Else
   ask via `AskUserQuestion`: "Issue # or `new`?"

**Creating a new issue** (when arg is `new` or user picked `new`):

```bash
TITLE="[xplanview] $(head -1 /tmp/xplanview-plan-draft.md | sed 's/^#* *//' | head -c 60)"
ISSUE=$(gh issue create --title "$TITLE" --body "Plan review tracker created by xplanview." | grep -oE '[0-9]+$')
```

Write the plan body (no chat filler) to `/tmp/xplanview-${ISSUE}-plan-v1.md`, then:

```bash
{ echo '<!-- xplanview:plan:v1 -->'
  echo '**xplanview Plan v1** — awaiting Round 1.'; echo
  cat /tmp/xplanview-${ISSUE}-plan-v1.md
} | gh issue comment "$ISSUE" --body-file -
```

## Phase 2 — Review loop (round = 1..4)

**State variables** maintained across rounds (the orchestrator — you — tracks these):
- `current_v` — latest plan version (starts at 1, incremented only on FAIL)
- `passed_by[v]` — set of reviewers who PASSed version `v` (e.g. `passed_by[2] = {opus}`)

**Per round**:

1. **Pick reviewer**: `opus` if round ∈ {1,3}, else `codex`.
2. **Read current plan** from `/tmp/xplanview-${ISSUE}-plan-v${current_v}.md`.
   (Note: `${current_v}`, NOT `${round}`. Version stays the same when one reviewer
   PASSed and we're now waiting for the other reviewer to weigh in on the same plan.)
3. **Run reviewer** (see "Reviewer invocations" below). Capture stdout to
   `/tmp/xplanview-${ISSUE}-review-r${round}.md`.
4. **Parse verdict** — tolerant of `## VERDICT\nPASS`, `## VERDICT: PASS`, and
   blank-line-after-header forms:
   ```bash
   VERDICT=$(awk '
     /^## VERDICT/ {
       if (match($0, /(PASS|FAIL)/)) { print substr($0, RSTART, RLENGTH); exit }
       found=1; next
     }
     found && /^(PASS|FAIL)/ { print $1; exit }
   ' /tmp/xplanview-${ISSUE}-review-r${round}.md)
   ```
   Empty/malformed → treat as `FAIL` with reason "reviewer output malformed".
5. **Post review** (marker now includes the version reviewed):
   ```bash
   { echo "<!-- xplanview:review:r${round}:${reviewer}:v${current_v} -->"
     echo "**Round ${round} — ${reviewer^} reviewer (reviewed v${current_v})**"; echo
     cat /tmp/xplanview-${ISSUE}-review-r${round}.md
   } | gh issue comment "$ISSUE" --body-file -
   ```
6. **Branch on verdict**:

   **PASS path**:
   - Add `${reviewer}` to `passed_by[${current_v}]`.
   - **Convergence check**: if `passed_by[${current_v}]` contains BOTH `opus` AND
     `codex` → CONVERGED. Post terminal verdict:
     ```bash
     { echo "<!-- xplanview:verdict:CONVERGED:r${round}:v${current_v} -->"
       echo "✓ **xplanview converged** — both Opus AND Codex PASSed v${current_v} (last review at R${round})."
     } | gh issue comment "$ISSUE" --body-file -
     ```
     Tell user in chat (`✓ xplanview 收敛 — v{V} 双 reviewer PASS`). Exit.
   - **Otherwise** (only one reviewer has passed v${current_v} so far): do NOT
     revise. Increment `round`, loop to step 1. Next reviewer will review the same
     `current_v`.

   **FAIL path**:
   - If `round == 4` → post ESCALATE with summary of `passed_by` map:
     ```bash
     { echo '<!-- xplanview:verdict:ESCALATE -->'
       echo '⚠️ **xplanview 4 rounds without 2-of-2 PASS.**'
       echo "Latest plan: v${current_v}"
       echo "Pass history:"
       echo "  v1: ${passed_by[1]:-(none)}"
       echo "  v2: ${passed_by[2]:-(none)}"
       echo "  ..."
     } | gh issue comment "$ISSUE" --body-file -
     ```
     Tell user in chat the core disagreement (1-2 sentences) and ask via
     `AskUserQuestion`: (a) accept current_v (b) human-rewrite (c) abort. Exit.
   - If `round < 4` → proceed to Phase 3 (revise).

## Phase 3 — Revise (only on FAIL with round < 4)

Read the FAIL review. Write a revision addressing every CRITICAL and the warranted
HIGH items to `/tmp/xplanview-${ISSUE}-plan-v$((current_v+1)).md`.

**Reset the pass tracking for the new version** — past PASSes on the old
`current_v` don't carry over. Both reviewers must re-pass the new revision.

```bash
NEXT_V=$((current_v + 1))
{ echo "<!-- xplanview:revision:v${NEXT_V} -->"
  echo "**xplanview Plan v${NEXT_V}** — revised after R${round} ${reviewer} FAIL on v${current_v}."; echo
  echo "### Changes from v${current_v}"
  echo "<2-5 bullets, each citing which review issue it addresses>"; echo
  echo "### Revised plan"
  cat /tmp/xplanview-${ISSUE}-plan-v${NEXT_V}.md
} | gh issue comment "$ISSUE" --body-file -

current_v=$NEXT_V
# passed_by[$NEXT_V] starts empty — needs both opus + codex on this new version
```

Then increment `round` and loop back to Phase 2 with the new `current_v`.

## Reviewer invocations

**Reviewer prompt** lives in `~/.claude/skills/xplanview/REVIEWER_PROMPT.md`. Both
reviewers use it verbatim, then append `ROUND: $round`, `REVIEWER: {opus|codex}`,
and the plan content under `PLAN UNDER REVIEW:`.

**Opus** — via the `Agent` tool:

```
Agent(
  description="xplanview Opus R{round}",
  subagent_type="general-purpose",
  model="opus",
  prompt=<REVIEWER_PROMPT + context + plan>
)
```

The agent's text return IS the review.

**Codex** — via Bash. Use stdin (long arg substitution unreliable ~20KB+) and
clean codex's output (it echoes the input prompt + appends `tokens used` telemetry):

```bash
# 1. Build prompt to file
{ cat ~/.claude/skills/xplanview/REVIEWER_PROMPT.md
  echo
  echo "ROUND: $round"
  echo "REVIEWER: codex"
  echo
  echo "PLAN UNDER REVIEW:"
  cat /tmp/xplanview-${ISSUE}-plan-v${round}.md
} > /tmp/xplanview-${ISSUE}-codex-prompt-r${round}.txt

# 2. Run codex via stdin (- = read from stdin)
codex exec --skip-git-repo-check --color never - \
  < /tmp/xplanview-${ISSUE}-codex-prompt-r${round}.txt \
  > /tmp/xplanview-${ISSUE}-review-raw-r${round}.txt 2>&1

# 3. Extract clean review (capture from LAST '## VERDICT' through end, drop telemetry)
# Note: variable name 'inblock' (not 'in' — awk reserved word)
awk '
  /^## VERDICT/ { capture=$0 ORS; inblock=1; next }
  inblock && /^tokens used$/ { inblock=0; next }
  inblock { capture=capture $0 ORS }
  END { printf "%s", capture }
' /tmp/xplanview-${ISSUE}-review-raw-r${round}.txt \
  > /tmp/xplanview-${ISSUE}-review-r${round}.md
```

## Red flags — STOP

- About to skip a round because "the plan is fine" → no, alternation is mandatory
- About to run R5 → no, escalate at 4
- About to feed reviewer your own context/opinion → no, isolation is the whole point
- About to omit the marker on a comment → no, breaks resume
- After PASS, about to start writing code immediately → no, this skill ends at PASS;
  user decides when to implement
