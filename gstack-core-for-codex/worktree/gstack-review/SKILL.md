---
name: review
description: |
  Pre-landing PR review. Analyzes diff against the base branch for SQL safety, LLM trust
  boundary violations, conditional side effects, and other structural issues. Use when
  asked to "review this PR", "code review", "pre-landing review", or "check my diff".
  Proactively suggest when the user is about to merge or land code changes.
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Codex Compatibility

This version preserves the gstack workflow while removing Claude-specific product shell,
telemetry prompts, contributor flows, upgrade flows, and host-only interaction APIs.

### Conversational Decision Protocol

Whenever the original skill says `AskUserQuestion`, ask directly in the normal chat.

Use this structure:
1. **Re-ground:** Briefly restate the project, current branch, current review phase, and the decision in front of the user.
2. **Simplify:** Explain the risk in plain English without implementation-heavy jargon.
3. **Recommend:** Give a clear recommendation, including completeness posture when relevant.
4. **Ask one question:** Offer 2-3 concrete options only when there is a meaningful tradeoff.

Rules:
- Ask one focused question at a time.
- Wait for the user's reply before moving to the next decision gate.
- If the answer is obvious and there is no meaningful tradeoff, state what you are doing and continue.
- When effort matters, prefer showing both human-team effort and AI-assisted effort.

### Completeness Principle

AI-assisted development makes the marginal cost of completeness much lower than human-only work.
When presenting options:

- Prefer the complete version when the extra effort is modest.
- Treat edge-case handling, trust-boundary validation, enum completeness, and documentation freshness as cheap wins.
- Distinguish **boilable lakes** from **unbounded oceans**. A lake is a complete review and fix set for a bounded diff. An ocean is an unscoped rewrite, speculative refactor, or undefined cleanup campaign.
- When estimating effort, mention both human-team effort and AI-assisted effort if it helps the user choose.

Anti-patterns:
- Recommending a shortcut only because it is shorter.
- Deferring obvious safety checks or test gaps without a real reason.
- Talking only in team-weeks when the practical AI-assisted cost is hours.

## Completion Status Protocol

When completing a skill workflow, report status using one of:
- **DONE** — All steps completed successfully. Evidence provided for each claim.
- **DONE_WITH_CONCERNS** — Completed, but with issues the user should know about. List each concern.
- **BLOCKED** — Cannot proceed. State what is blocking and what was tried.
- **NEEDS_CONTEXT** — Missing information required to continue. State exactly what you need.

### Escalation

It is always OK to stop and say "this is too hard for me" or "I'm not confident in this result."

Bad work is worse than no work. You will not be penalized for escalating.
- If you have attempted a task 3 times without success, STOP and escalate.
- If you are uncertain about a security-sensitive change, STOP and escalate.
- If the scope of work exceeds what you can verify, STOP and escalate.

Escalation format:
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Local Workspace Conventions

Re-run this bootstrap block before any phase that relies on repository paths, review history,
or analytics. Do not assume variables from an earlier step still exist.

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
[ -z "$_ROOT" ] && _ROOT="$PWD"
_BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
_REPO=$(basename "$_ROOT")
_SLUG=$(printf "%s" "$_REPO" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-')
_BRANCH_SAFE=$(printf "%s" "$_BRANCH" | tr '/:' '--')
_GLOBAL_PROJECT_DIR="$HOME/.codex/gstack/projects/$_SLUG"
_LOCAL_GSTACK_DIR="$_ROOT/.codex/gstack"
_LOCAL_REVIEW_DIR="$_LOCAL_GSTACK_DIR/reviews"
_GLOBAL_REVIEW_DIR="$_GLOBAL_PROJECT_DIR/reviews"
_LOCAL_GREPTILE_DIR="$_LOCAL_REVIEW_DIR/greptile-history"
_GLOBAL_GREPTILE_DIR="$_GLOBAL_REVIEW_DIR/greptile-history"
_ANALYTICS_DIR="$HOME/.codex/gstack/analytics"
_REVIEW_LOG="$_ANALYTICS_DIR/reviews.jsonl"
_SUPPORT_DIR="/Users/clawtester/Documents/teamagents/vendor/gstack/review"
mkdir -p "$_GLOBAL_PROJECT_DIR" "$_LOCAL_GSTACK_DIR" "$_LOCAL_REVIEW_DIR" "$_GLOBAL_REVIEW_DIR" "$_LOCAL_GREPTILE_DIR" "$_GLOBAL_GREPTILE_DIR" "$_ANALYTICS_DIR"
echo "ROOT: $_ROOT"
echo "BRANCH: $_BRANCH"
echo "GLOBAL_PROJECT_DIR: $_GLOBAL_PROJECT_DIR"
echo "LOCAL_REVIEW_DIR: $_LOCAL_REVIEW_DIR"
echo "GLOBAL_REVIEW_DIR: $_GLOBAL_REVIEW_DIR"
echo "LOCAL_GREPTILE_DIR: $_LOCAL_GREPTILE_DIR"
echo "GLOBAL_GREPTILE_DIR: $_GLOBAL_GREPTILE_DIR"
echo "SUPPORT_DIR: $_SUPPORT_DIR"
echo "REVIEW_LOG: $_REVIEW_LOG"
```

Registry policy:
- `$_GLOBAL_PROJECT_DIR` is the source of truth for cross-session gstack history.
- `$_LOCAL_REVIEW_DIR` stores repo-local review artifacts.
- `$_GLOBAL_REVIEW_DIR` stores canonical review reports and related cross-session metadata.

## Analytics (run last when practical)

If the workflow completes and you can safely write local analytics without relying on
missing gstack binaries, append lightweight JSONL records under
`$HOME/.codex/gstack/analytics/`. If you cannot determine a field, omit it instead of
failing the workflow.

## Step 0: Detect base branch

Determine which branch this review should compare against. Use the result as "the base branch"
in all subsequent steps.

1. Check if a PR already exists for this branch:
   `gh pr view --json baseRefName -q .baseRefName`
   If this succeeds, use the printed branch name as the base branch.

2. If no PR exists, detect the repo's default branch:
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. If `gh` is unavailable or both commands fail, infer from local refs:
   - Prefer `origin/HEAD`
   - Then prefer an existing local branch in this order: `main`, `master`, `trunk`, `develop`
   - If none exist, use the current branch's merge-base ancestor heuristically and say this is best-effort

4. Set `BASE_BRANCH` and `BASE_REF` to the chosen values and echo them before continuing:

```bash
BASE_BRANCH=$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || true)
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || true)
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's#^refs/remotes/origin/##')
[ -z "$BASE_BRANCH" ] && for _CANDIDATE in main master trunk develop; do git show-ref --verify --quiet "refs/heads/$_CANDIDATE" && BASE_BRANCH="$_CANDIDATE" && break; done
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
BASE_REF="$BASE_BRANCH"
[ -n "$BASE_BRANCH" ] && git show-ref --verify --quiet "refs/remotes/origin/$BASE_BRANCH" && BASE_REF="origin/$BASE_BRANCH"
echo "BASE_BRANCH: $BASE_BRANCH"
echo "BASE_REF: $BASE_REF"
```

---

# Pre-Landing PR Review

You are running the `/review` workflow. Analyze the current branch's diff against the base branch for structural issues that tests don't catch.

---

## Step 1: Check branch

1. Run `git branch --show-current` to get the current branch.
2. If on the base branch, output: **"Nothing to review — you're on the base branch or have no changes against it."** and stop.
3. If an `origin/$BASE_BRANCH` ref exists, run `git fetch origin "$BASE_BRANCH" --quiet` and compare against `origin/$BASE_BRANCH`.
4. Otherwise compare against `BASE_REF`, which may be the best available local ref for `$BASE_BRANCH`.
5. If there is no diff, output the same message and stop.

---

## Step 1.5: Scope Drift Detection

Before reviewing code quality, check: **did they build what was requested — nothing more, nothing less?**

1. Read `TODOS.md` (if it exists). Read PR description (`gh pr view --json body --jq .body 2>/dev/null || true`).
   Read commit messages (`git log "$BASE_REF"..HEAD --oneline`).
   **If no PR exists:** rely on commit messages and TODOS.md for stated intent — this is the common case since /review runs before /ship creates the PR.
2. Identify the **stated intent** — what was this branch supposed to accomplish?
3. Run `git diff` against the resolved base reference and compare the files changed against the stated intent.
4. Evaluate with skepticism:

   **SCOPE CREEP detection:**
   - Files changed that are unrelated to the stated intent
   - New features or refactors not mentioned in the plan
   - "While I was in there..." changes that expand blast radius

   **MISSING REQUIREMENTS detection:**
   - Requirements from TODOS.md/PR description not addressed in the diff
   - Test coverage gaps for stated requirements
   - Partial implementations (started but not finished)

5. Output (before the main review begins):
   ```
   Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
   Intent: <1-line summary of what was requested>
   Delivered: <1-line summary of what the diff actually does>
   [If drift: list each out-of-scope change]
   [If missing: list each unaddressed requirement]
   ```

6. This is **INFORMATIONAL** — does not block the review. Proceed to Step 2.

---

## Step 2: Read the checklist

Read `$_SUPPORT_DIR/checklist.md`.

**If the file cannot be read, STOP and report the error.** Do not proceed without the checklist.

---

## Step 2.5: Check for Greptile review comments

Read `$_SUPPORT_DIR/greptile-triage.md` and follow the fetch, filter, classify, and **escalation detection** steps.

**If no PR exists, `gh` fails, API returns an error, or there are zero Greptile comments:** Skip this step silently. Greptile integration is additive — the review works without it.

**If Greptile comments are found:** Store the classifications (VALID & ACTIONABLE, VALID BUT ALREADY FIXED, FALSE POSITIVE, SUPPRESSED) under both `$LOCAL_GREPTILE_DIR` and `$GLOBAL_GREPTILE_DIR` — you will need them in Step 5.

---

## Step 3: Get the diff

Fetch the latest base branch to avoid false positives from stale local state:

```bash
[ -n "$BASE_BRANCH" ] && git fetch origin "$BASE_BRANCH" --quiet 2>/dev/null || true
```

Run `git diff "$BASE_REF"...HEAD` to get the committed diff, then inspect `git diff "$BASE_REF"` if you also need uncommitted changes against the resolved base reference.

---

## Step 4: Two-pass review

Apply the checklist against the diff in two passes:

1. **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Enum & Value Completeness
2. **Pass 2 (INFORMATIONAL):** Conditional Side Effects, Magic Numbers & String Coupling, Dead Code & Consistency, LLM Prompt Issues, Test Gaps, View/Frontend

**Enum & Value Completeness requires reading code OUTSIDE the diff.** When the diff introduces a new enum value, status, tier, or type constant, use Grep to find all files that reference sibling values, then Read those files to check if the new value is handled. This is the one category where within-diff review is insufficient.

Follow the output format specified in the checklist. Respect the suppressions — do NOT flag items listed in the "DO NOT flag" section.

---

## Step 4.5: Design Review (conditional)

## Design Review (conditional, diff-scoped)

Check if the diff touches frontend files by inspecting changed paths in both the committed diff and the current working tree:

```bash
{
  git diff --name-only "$BASE_REF"...HEAD 2>/dev/null
  git diff --name-only "$BASE_REF" 2>/dev/null
} | sort -u | rg -E '\.(tsx?|jsx?|css|scss|sass|less|html|vue|svelte)$|(^|/)(components|pages|app|src/ui|src/components|src/routes)/' || true
```

**If no frontend files are touched:** Skip design review silently. No output.

**If frontend files are touched:**

1. **Check for DESIGN.md.** If `DESIGN.md` or `design-system.md` exists in the repo root, read it. All design findings are calibrated against it — patterns blessed in DESIGN.md are not flagged. If not found, use universal design principles.

2. **Read `$_SUPPORT_DIR/design-checklist.md`.** If the file cannot be read, skip design review with a note: "Design checklist not found — skipping design review."

3. **Read each changed frontend file** (full file, not just diff hunks). Frontend files are identified by the patterns listed in the checklist.

4. **Apply the design checklist** against the changed files. For each item:
   - **[HIGH] mechanical CSS fix** (`outline: none`, `!important`, `font-size < 16px`): classify as AUTO-FIX
   - **[HIGH/MEDIUM] design judgment needed**: classify as ASK
   - **[LOW] intent-based detection**: present as "Possible — verify visually or run /design-review"

5. **Include findings** in the review output under a "Design Review" header, following the output format in the checklist. Design findings merge with code review findings into the same Fix-First flow.

6. **Log the result** for the Review Readiness Dashboard:

```bash
printf '%s\n' '{"skill":"design-review-lite","timestamp":"TIMESTAMP","status":"STATUS","findings":N,"auto_fixed":M,"commit":"COMMIT"}' >> "$_REVIEW_LOG"
```

Substitute: TIMESTAMP = ISO 8601 datetime, STATUS = "clean" if 0 findings or "issues_found", N = total findings, M = auto-fixed count, COMMIT = output of `git rev-parse --short HEAD`.

Include any design findings alongside the findings from Step 4. They follow the same Fix-First flow in Step 5 — AUTO-FIX for mechanical CSS fixes, ASK for everything else.

---

## Step 5: Fix-First Review

**Every finding gets action — not just critical ones.**

Output a summary header: `Pre-Landing Review: N issues (X critical, Y informational)`

### Step 5a: Classify each finding

For each finding, classify as AUTO-FIX or ASK per the Fix-First Heuristic in
`$_SUPPORT_DIR/checklist.md`. Critical findings lean toward ASK; informational findings lean
toward AUTO-FIX.

### Step 5b: Auto-fix all AUTO-FIX items

Apply each fix directly. For each one, output a one-line summary:
`[AUTO-FIXED] [file:line] Problem → what you did`

### Step 5c: Batch-ask about ASK items

If there are ASK items remaining, present them in one direct chat decision gate:

- List each item with a number, the severity label, the problem, and a recommended fix
- For each item, provide options: A) Fix as recommended, B) Skip
- Include an overall RECOMMENDATION

Example format:
```
I auto-fixed 5 issues. 2 need your input:

1. [CRITICAL] app/models/post.rb:42 — Race condition in status transition
   Fix: Add `WHERE status = 'draft'` to the UPDATE
   → A) Fix  B) Skip

2. [INFORMATIONAL] app/services/generator.rb:88 — LLM output not type-checked before DB write
   Fix: Add JSON schema validation
   → A) Fix  B) Skip

RECOMMENDATION: Fix both — #1 is a real race condition, #2 prevents silent data corruption.
```

If 3 or fewer ASK items, you may use individual direct chat questions instead of batching.

### Step 5d: Apply user-approved fixes

Apply fixes for items where the user chose "Fix." Output what was fixed.

If no ASK items exist (everything was AUTO-FIX), skip the question entirely.

### Verification of claims

Before producing the final review output:
- If you claim "this pattern is safe" → cite the specific line proving safety
- If you claim "this is handled elsewhere" → read and cite the handling code
- If you claim "tests cover this" → name the test file and method
- Never say "likely handled" or "probably tested" — verify or flag as unknown

**Rationalization prevention:** "This looks fine" is not a finding. Either cite evidence it IS fine, or flag it as unverified.

### Greptile comment resolution

After outputting your own findings, if Greptile comments were classified in Step 2.5:

**Include a Greptile summary in your output header:** `+ N Greptile comments (X valid, Y fixed, Z FP)`

Before replying to any comment, run the **Escalation Detection** algorithm from `greptile-triage.md` to determine whether to use Tier 1 (friendly) or Tier 2 (firm) reply templates.

1. **VALID & ACTIONABLE comments:** These are included in your findings — they follow the Fix-First flow (auto-fixed if mechanical, batched into ASK if not) (A: Fix it now, B: Acknowledge, C: False positive). If the user chooses A (fix), reply using the **Fix reply template** from greptile-triage.md (include inline diff + explanation). If the user chooses C (false positive), reply using the **False Positive reply template** (include evidence + suggested re-rank), save to both `$LOCAL_GREPTILE_DIR` and `$GLOBAL_GREPTILE_DIR`.

2. **FALSE POSITIVE comments:** Present each one via a direct chat decision gate:
   - Show the Greptile comment: file:line (or [top-level]) + body summary + permalink URL
   - Explain concisely why it's a false positive
   - Options:
     - A) Reply to Greptile explaining why this is incorrect (recommended if clearly wrong)
     - B) Fix it anyway (if low-effort and harmless)
     - C) Ignore — don't reply, don't fix

   If the user chooses A, reply using the **False Positive reply template** from greptile-triage.md (include evidence + suggested re-rank), save to both `$LOCAL_GREPTILE_DIR` and `$GLOBAL_GREPTILE_DIR`.

3. **VALID BUT ALREADY FIXED comments:** Reply using the **Already Fixed reply template** from greptile-triage.md — no user question needed:
   - Include what was done and the fixing commit SHA
   - Save to both `$LOCAL_GREPTILE_DIR` and `$GLOBAL_GREPTILE_DIR`

4. **SUPPRESSED comments:** Skip silently — these are known false positives from previous triage.

---

## Step 5.5: TODOS cross-reference

Read `TODOS.md` in the repository root (if it exists). Cross-reference the PR against open TODOs:

- **Does this PR close any open TODOs?** If yes, note which items in your output: "This PR addresses TODO: <title>"
- **Does this PR create work that should become a TODO?** If yes, flag it as an informational finding.
- **Are there related TODOs that provide context for this review?** If yes, reference them when discussing related findings.

If TODOS.md doesn't exist, skip this step silently.

---

## Step 5.6: Documentation staleness check

Cross-reference the diff against documentation files. For each `.md` file in the repo root (README.md, ARCHITECTURE.md, CONTRIBUTING.md, CLAUDE.md, etc.):

1. Check if code changes in the diff affect features, components, or workflows described in that doc file.
2. If the doc file was NOT updated in this branch but the code it describes WAS changed, flag it as an INFORMATIONAL finding:
   "Documentation may be stale: [file] describes [feature/component] but code changed in this branch. Consider running `/document-release`."

This is informational only — never critical. The fix action is `/document-release`.

If no documentation files exist, skip this step silently.

---



## Important Rules

- **Read the FULL diff before commenting.** Do not flag issues already addressed in the diff.
- **Fix-first, not read-only.** AUTO-FIX items are applied directly. ASK items are only applied after user approval. Never commit, push, or create PRs — that's /ship's job.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Use Greptile reply templates from greptile-triage.md.** Every reply includes evidence. Never post vague replies.
