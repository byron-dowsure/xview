---
name: xprview
description: Use when the user types "xprview" or "/xprview <PR#>" to launch a code-review-and-fix loop on an open same-repo PR. Two distinct reviewers (Codex + Opus) must BOTH PASS the same HEAD SHA to converge; Sonnet does the fixes. Max 4 rounds. Fork PRs early-escalated.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Agent
  - AskUserQuestion
---

# xprview

PR code review + auto-fix loop. **Three distinct roles, no overlap**:

- **Codex** = reviewer-only (`codex exec` subprocess), never touches code
- **Opus** = reviewer-only (Agent subagent with `model: opus`), never touches code
- **Sonnet** = fixer-only (Agent subagent with `model: sonnet`), never gives opinion
- **Main session** = orchestrator only, never reviews and never edits files

State persisted as PR comments + git commits. Inspired by `xplanview` (which reviews
plans before code) — `xprview` reviews code after a PR is open.

## When to use which

| Skill | Phase | Reviewers | Fixer | Persistence |
|---|---|---|---|---|
| `xplanview` | Before code | Opus + Codex (alternating) | Main session | issue comments |
| `xprview` | After PR open | Codex + Opus (alternating) | Sonnet subagent | PR comments + commits |

## Hard rules (do not adapt)

- Round order: **R1=Codex → R2=Opus → R3=Codex → R4=Opus**. Strictly alternating.
- **Convergence requires BOTH Codex AND Opus to PASS the SAME HEAD SHA.** A single
  PASS is NOT enough — one reviewer alone is "self-review" and doesn't prove
  correctness. Track which SHA each reviewer has passed.
- On any FAIL, Sonnet fixes + commits + pushes (HEAD SHA changes). Past PASSes on
  older SHAs no longer count — both reviewers must re-pass the new HEAD.
- **Round counter ≠ SHA changes.** SHA only changes after Sonnet fix. If R1 Codex
  PASSes baseline SHA (no convergence yet), R2 Opus reviews the same baseline SHA.
- Max 4 review rounds. After R4 without 2-of-2 PASS on same SHA → ESCALATE. No R5.
- **Same-repo PRs only.** Fork PRs (no origin push) early-escalate at Phase 1.5.
- Refuse to start if local worktree is dirty (Phase 1.6).
- Every PR comment starts with the exact HTML marker below.

## Marker scheme

| Marker | When |
|---|---|
| `<!-- xprview:baseline:{sha} -->` | Skill startup, captures PR HEAD SHA |
| `<!-- xprview:review:r{N}:{codex\|opus}:{sha} -->` | Reviewer R for round N reviewed HEAD={sha} |
| `<!-- xprview:fix:r{N}:{new_sha} -->` | Sonnet fix after round N FAIL → new HEAD SHA |
| `<!-- xprview:verdict:CONVERGED:r{N}:{sha} -->` | Terminal: both Codex AND Opus PASSed {sha} |
| `<!-- xprview:verdict:ESCALATE -->` | Terminal: 4 rounds no convergence, OR fork PR, OR fatal failure |

PR description gets a `<!-- xprview-trail -->` block with one line per round.

## Phase 0 — Probes

```bash
codex exec --help >/dev/null 2>&1 || { echo "❌ codex exec unavailable"; exit 1; }
gh auth status >/dev/null 2>&1 || { echo "❌ gh not authenticated"; exit 1; }
command -v jq >/dev/null || { echo "❌ jq required"; exit 1; }
```

**Note**: We use `codex exec` (not `codex exec review`). The `review` subcommand
expects to compute its own diff from git repo state and does NOT accept diff
content in the prompt. Since xprview pipes the PR diff in via the prompt, plain
`codex exec` is the correct invocation.

## Phase 1 — Anchor

The user invocation may carry an argument: `/xprview 123` or `/xprview` (auto-detect):

1. **Numeric arg** → `PR=<arg>`. Verify with `gh pr view "$PR" --json state`.
2. **No arg** → `PR=$(gh pr view --json number -q .number 2>/dev/null)`.
3. **Still empty** → ask via `AskUserQuestion`.

```bash
META=$(gh pr view "$PR" --json state,headRefOid,headRefName,isCrossRepository)
[ "$(jq -r .state <<<"$META")" = "OPEN" ] || { echo "PR not OPEN"; exit 1; }
IS_FORK=$(jq -r .isCrossRepository <<<"$META")
BASELINE_SHA=$(jq -r .headRefOid <<<"$META")
PR_BRANCH=$(jq -r .headRefName <<<"$META")
```

### Phase 1.5 — Fork PR early-escalate

```bash
if [ "$IS_FORK" = "true" ]; then
  cat <<EOF | gh pr comment "$PR" --body-file -
<!-- xprview:verdict:ESCALATE -->
⚠️ **xprview does not support fork PRs** (no origin push access).
Suggest: human review + ask the contributor to push fixes themselves.
EOF
  exit 0
fi
```

### Phase 1.6 — Dirty worktree early-escalate

```bash
if ! git diff --quiet || ! git diff --cached --quiet; then
  cat <<EOF | gh pr comment "$PR" --body-file -
<!-- xprview:verdict:ESCALATE -->
⚠️ **Local worktree has uncommitted changes**, xprview refuses to start.
Please commit or stash first.
EOF
  exit 0
fi
```

### Sync to PR branch + post baseline

```bash
git fetch origin "$PR_BRANCH"
git checkout "$PR_BRANCH"
git pull --ff-only origin "$PR_BRANCH"

cat <<EOF | gh pr comment "$PR" --body-file -
<!-- xprview:baseline:$BASELINE_SHA -->
🔍 **xprview started** — baseline @ \`$BASELINE_SHA\`.
Convergence rule: BOTH Codex AND Opus must PASS the same HEAD SHA.
Max 4 review rounds + 3 sonnet fix rounds.
EOF
```

## Phase 2 — Main loop

**State variables** maintained by orchestrator:
- `current_sha` — current PR HEAD SHA (starts at `$BASELINE_SHA`, changes after every Sonnet fix)
- `passed_by[sha]` — set of reviewers who PASSed `sha`

```
round = 1
current_sha = BASELINE_SHA
passed_by = {}

while round <= 4:
  reviewer = "codex" if round % 2 == 1 else "opus"
  
  # Run reviewer (sub-functions below)
  if reviewer == "codex":
    VERDICT = run_codex_review(round, current_sha)
  else:
    VERDICT = run_opus_review(round, current_sha)
  
  post_review(round, reviewer, current_sha, VERDICT)
  update_pr_trail(round, reviewer, VERDICT)
  
  if VERDICT == PASS:
    passed_by[current_sha].add(reviewer)
    if {"codex", "opus"} ⊆ passed_by[current_sha]:
      post_verdict_converged(round, current_sha); exit
    # only one reviewer passed; don't fix, advance round to let other reviewer weigh in
  else:  # FAIL
    if round == 4:
      post_verdict_escalate(passed_by); ask_three_options(); exit
    new_sha = run_sonnet_fix(round, current_sha)   # exits via post_escalate on internal failure
    current_sha = new_sha   # passed_by[new_sha] starts empty
  
  round += 1

# Loop exited at round=5 without converging (last action was a non-converging PASS)
post_verdict_escalate(passed_by, "ran out of rounds without 2-of-2 PASS")
```

## `run_codex_review` (Bash)

```bash
gh pr diff "$PR" > /tmp/xprview-${PR}-diff-r${round}.diff

# Pull changed files with size cap
MAX_LINES=2000
gh pr view "$PR" --json files -q '.files[].path' > /tmp/xprview-${PR}-files-r${round}.txt
> /tmp/xprview-${PR}-context-r${round}.md
while read -r f; do
  [ -f "$f" ] || { echo "=== $f === (deleted)" >> /tmp/xprview-${PR}-context-r${round}.md; continue; }
  L=$(wc -l < "$f")
  echo "=== $f ($L lines) ===" >> /tmp/xprview-${PR}-context-r${round}.md
  if [ "$L" -gt "$MAX_LINES" ]; then
    echo "(truncated to first $MAX_LINES lines)" >> /tmp/xprview-${PR}-context-r${round}.md
    head -n $MAX_LINES "$f" >> /tmp/xprview-${PR}-context-r${round}.md
  else
    cat "$f" >> /tmp/xprview-${PR}-context-r${round}.md
  fi
done < /tmp/xprview-${PR}-files-r${round}.txt

# Build prompt + run codex via stdin
{ cat ~/.claude/skills/xprview/REVIEWER_PROMPT.md
  echo; echo "ROUND: $round"; echo "REVIEWER: codex"; echo "HEAD SHA: $current_sha"; echo
  echo "PR DIFF:"
  cat /tmp/xprview-${PR}-diff-r${round}.diff
  echo
  echo "CHANGED FILES (truncated to $MAX_LINES lines each):"
  cat /tmp/xprview-${PR}-context-r${round}.md
} > /tmp/xprview-${PR}-prompt-r${round}.txt

codex exec --skip-git-repo-check --color never - \
  < /tmp/xprview-${PR}-prompt-r${round}.txt \
  > /tmp/xprview-${PR}-review-raw-r${round}.txt 2>&1

# Clean codex output (echoes input + appends "tokens used" telemetry)
# Note: variable name 'inblock' (not 'in' — awk reserved word)
awk '
  /^## VERDICT/ { capture=$0 ORS; inblock=1; next }
  inblock && /^tokens used$/ { inblock=0; next }
  inblock { capture=capture $0 ORS }
  END { printf "%s", capture }
' /tmp/xprview-${PR}-review-raw-r${round}.txt \
  > /tmp/xprview-${PR}-review-r${round}.md

VERDICT=$(parse_verdict /tmp/xprview-${PR}-review-r${round}.md)   # see helper below
```

## `run_opus_review` — orchestrated (NOT pure bash)

`Agent` is a Claude Code tool, not a shell command. This step mixes Bash (prep)
and direct Claude tool calls (Agent + Write).

### Step a — Build prompt file (Bash)

Same diff + file context fetching as `run_codex_review` (diff and context files
can be reused if same `current_sha`; but to keep code simple, refetch each round):

```bash
{ cat ~/.claude/skills/xprview/REVIEWER_PROMPT.md
  echo; echo "ROUND: $round"; echo "REVIEWER: opus"; echo "HEAD SHA: $current_sha"; echo
  echo "PR DIFF:"
  cat /tmp/xprview-${PR}-diff-r${round}.diff
  echo
  echo "CHANGED FILES:"
  cat /tmp/xprview-${PR}-context-r${round}.md
} > /tmp/xprview-${PR}-prompt-r${round}.txt
```

### Step b — Main session calls Agent tool

Read prompt with `Read` tool, then invoke:

```
Agent(
  description="xprview Opus review r{round}",
  subagent_type="general-purpose",
  model="opus",
  prompt=<contents of /tmp/xprview-${PR}-prompt-r${round}.txt>
)
```

### Step c — Main session writes return value to file

The Agent returns the review text. Use `Write` tool to save to
`/tmp/xprview-${PR}-review-r${round}.md`. Then parse:

```bash
VERDICT=$(parse_verdict /tmp/xprview-${PR}-review-r${round}.md)
```

## `parse_verdict` — shared helper

```bash
parse_verdict() {
  local file=$1
  # Tolerant: handles "## VERDICT\nPASS" and "## VERDICT: PASS"
  local v=$(awk '
    /^## VERDICT/ {
      if (match($0, /(PASS|FAIL)/)) { print substr($0, RSTART, RLENGTH); exit }
      found=1; next
    }
    found && /^(PASS|FAIL)/ { print $1; exit }
  ' "$file")
  
  # Malformed → synthesize FAIL
  if [ -z "$v" ] || ! grep -q '^## REASONING' "$file"; then
    cat > "$file" <<EOF
## VERDICT
FAIL

## CRITICAL
- (auto-synthesized) Reviewer returned malformed output.

## HIGH
- (none)

## MEDIUM
- (none)

## REASONING
Auto-synthesized FAIL because output did not parse.
EOF
    echo "FAIL"
  else
    echo "$v"
  fi
}
```

## `post_review` (Bash)

```bash
{ echo "<!-- xprview:review:r${round}:${reviewer}:${current_sha} -->"
  echo "**Round ${round} — ${reviewer^} reviewer (HEAD ${current_sha:0:7})**"; echo
  cat /tmp/xprview-${PR}-review-r${round}.md
} | gh pr comment "$PR" --body-file -
```

## `run_sonnet_fix` — orchestrated (NOT pure bash)

Same 4-step pattern as `run_opus_review`: Bash prep → Agent call → Write to file → Bash verify+commit+push.

### Step a — Build fixer input (Bash)

```bash
{ cat ~/.claude/skills/xprview/FIXER_PROMPT.md
  echo; echo "WORKING DIR: $(pwd)"; echo "PR BRANCH: $PR_BRANCH"; echo
  echo "LATEST FAIL REVIEW:"
  cat /tmp/xprview-${PR}-review-r${round}.md
  echo; echo "PR DIFF:"
  cat /tmp/xprview-${PR}-diff-r${round}.diff
  echo; echo "CHANGED FILES:"
  cat /tmp/xprview-${PR}-context-r${round}.md
} > /tmp/xprview-${PR}-fixer-in-r${round}.md
```

### Step b — Main session calls Agent tool

```
Agent(
  description="xprview Sonnet fix r{round}",
  subagent_type="general-purpose",
  model="sonnet",
  prompt=<contents of /tmp/xprview-${PR}-fixer-in-r${round}.md>
)
```

### Step c — Main session writes return value to file

`Write` tool → `/tmp/xprview-${PR}-fix-r${round}-result.md`.

### Step d — Verify + lint/test + commit + push (Bash)

```bash
if git diff --quiet && git diff --cached --quiet; then
  post_escalate "Sonnet made no changes in r${round} (disagrees with reviewer)"
  return 1
fi

if [ -f package.json ]; then
  HAS_LINT=$(jq -r '.scripts.lint // ""' package.json)
  HAS_TEST=$(jq -r '.scripts.test // ""' package.json)
  [ -n "$HAS_LINT" ] && (npm run lint || { post_escalate "lint failed"; return 1; })
  [ -n "$HAS_TEST" ] && (npm test || { post_escalate "test failed"; return 1; })
fi

xargs -a /tmp/xprview-${PR}-files-r${round}.txt git add --

SUMMARY=$(grep '^SUMMARY:' /tmp/xprview-${PR}-fix-r${round}-result.md | head -1 | sed 's/^SUMMARY: *//')
[ -z "$SUMMARY" ] && SUMMARY="(no summary)"
git commit -m "xprview r${round} fix: ${SUMMARY}"
NEW_SHA=$(git rev-parse HEAD)
git push origin "$PR_BRANCH" || { post_escalate "push failed (no auto-rebase)"; return 1; }

{ echo "<!-- xprview:fix:r${round}:${NEW_SHA} -->"
  echo "**Round ${round} fix** — commit \`${NEW_SHA}\` (was \`${current_sha:0:7}\`)"; echo
  echo "**Summary**: ${SUMMARY}"; echo
  awk '/^ADDRESSED:/,EOF' /tmp/xprview-${PR}-fix-r${round}-result.md
} | gh pr comment "$PR" --body-file -

# Return new SHA for orchestrator to update current_sha
echo "$NEW_SHA"
```

## `update_pr_trail`

```bash
update_pr_trail() {
  local round=$1 reviewer=$2 verdict=$3
  local summary=$(awk '/^## REASONING$/{flag=1; next} flag && NF{print; exit}' \
    /tmp/xprview-${PR}-review-r${round}.md | head -c 80)
  local body=$(gh pr view "$PR" --json body -q .body)
  if echo "$body" | grep -q '<!-- xprview-trail -->'; then
    printf '%s\n- xprview r%d (%s): %s — %s\n' "$body" "$round" "$reviewer" "$verdict" "$summary"
  else
    printf '%s\n\n<!-- xprview-trail -->\n- xprview r%d (%s): %s — %s\n' "$body" "$round" "$reviewer" "$verdict" "$summary"
  fi | gh pr edit "$PR" --body-file -
}
```

## ESCALATE three options (after R4 without convergence)

`AskUserQuestion`:
- `accept` — accept current PR state, merge as-is
- `human-fix` — I'll fix manually, xprview stops
- `abort` — close the PR

## Red flags — STOP

- About to run R5 → no, escalate at 4
- About to converge on 1 PASS alone → no, requires BOTH Codex AND Opus on SAME SHA
- About to use main-session as reviewer or fixer → no, role separation is mandatory
- About to skip Phase 1.5 (fork) or Phase 1.6 (dirty worktree) → no
- About to attempt fork PR fix → no, early-escalate
- About to omit a marker on a comment → no, breaks the trail
- About to `git add -A` → no, only stage PR files via the `xargs` line
- About to feed reviewer your own opinion → no, isolation is the whole point
- About to count a PASS on an old SHA toward the new SHA → no, fix invalidates past PASSes
