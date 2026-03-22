---
name: retro
description: |
  Weekly engineering retrospective. Analyzes commit history, work patterns,
  and code quality metrics with persistent history and trend tracking.
  Team-aware: breaks down per-person contributions with praise and growth areas.
  Use when asked to "weekly retro", "what did we ship", or "engineering retrospective".
  Proactively suggest at the end of a work week or sprint.
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Codex Compatibility

This version preserves the gstack workflow while removing Claude-specific product shell,
telemetry prompts, contributor flows, upgrade flows, and host-only interaction APIs.

### Conversational Decision Protocol

Whenever the original skill says `AskUserQuestion`, ask directly in the normal chat.

Use this structure:
1. **Re-ground:** Briefly restate the project, current branch, current retrospective phase, and the decision in front of the user.
2. **Simplify:** Explain the metric, tradeoff, or data gap in plain English without implementation-heavy jargon.
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
- Treat trend tracking, contributor breakdowns, and evidence-backed growth advice as cheap wins.
- Distinguish **boilable lakes** from **unbounded oceans**. A lake is a bounded retro window with available repo data. An ocean is trying to infer a perfect productivity system from missing analytics and incomplete history.
- When estimating effort, mention both human-team effort and AI-assisted effort if it helps the user choose.

Anti-patterns:
- Pretending missing analytics exist.
- Turning a data-limited retro into fake precision.
- Skipping available repo history just because some optional metrics are absent.

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

Re-run this bootstrap block before any phase that relies on repository paths, retro history,
review analytics, or Greptile history. Do not assume variables from an earlier step still exist.

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
[ -z "$_ROOT" ] && _ROOT="$PWD"
_BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
_REPO=$(basename "$_ROOT")
_SLUG=$(printf "%s" "$_REPO" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-')
_BRANCH_SAFE=$(printf "%s" "$_BRANCH" | tr '/:' '--')
_GLOBAL_PROJECT_DIR="$HOME/.codex/gstack/projects/$_SLUG"
_LOCAL_GSTACK_DIR="$_ROOT/.codex/gstack"
_LOCAL_RETRO_DIR="$_LOCAL_GSTACK_DIR/retros"
_GLOBAL_RETRO_DIR="$_GLOBAL_PROJECT_DIR/retros"
_LOCAL_REVIEW_DIR="$_LOCAL_GSTACK_DIR/reviews"
_GLOBAL_REVIEW_DIR="$_GLOBAL_PROJECT_DIR/reviews"
_LOCAL_GREPTILE_DIR="$_LOCAL_REVIEW_DIR/greptile-history"
_GLOBAL_GREPTILE_DIR="$_GLOBAL_REVIEW_DIR/greptile-history"
_ANALYTICS_DIR="$HOME/.codex/gstack/analytics"
_REVIEW_LOG="$_ANALYTICS_DIR/reviews.jsonl"
_SKILL_USAGE_LOG="$_ANALYTICS_DIR/skill-usage.jsonl"
mkdir -p "$_GLOBAL_PROJECT_DIR" "$_LOCAL_GSTACK_DIR" "$_LOCAL_RETRO_DIR" "$_GLOBAL_RETRO_DIR" "$_LOCAL_REVIEW_DIR" "$_GLOBAL_REVIEW_DIR" "$_LOCAL_GREPTILE_DIR" "$_GLOBAL_GREPTILE_DIR" "$_ANALYTICS_DIR"
echo "ROOT: $_ROOT"
echo "BRANCH: $_BRANCH"
echo "GLOBAL_PROJECT_DIR: $_GLOBAL_PROJECT_DIR"
echo "LOCAL_RETRO_DIR: $_LOCAL_RETRO_DIR"
echo "GLOBAL_RETRO_DIR: $_GLOBAL_RETRO_DIR"
echo "LOCAL_GREPTILE_DIR: $_LOCAL_GREPTILE_DIR"
echo "GLOBAL_GREPTILE_DIR: $_GLOBAL_GREPTILE_DIR"
echo "REVIEW_LOG: $_REVIEW_LOG"
echo "SKILL_USAGE_LOG: $_SKILL_USAGE_LOG"
```

Registry policy:
- `$_GLOBAL_PROJECT_DIR` is the source of truth for cross-session gstack history.
- `$_GLOBAL_RETRO_DIR` is the canonical store for saved retro snapshots.
- `$_LOCAL_RETRO_DIR` mirrors snapshots into the repo for local continuity and easy diffing.
- `$_LOCAL_GREPTILE_DIR` and `$_GLOBAL_GREPTILE_DIR` are the compatible Greptile history stores established by `/review` and `/ship`.

## Analytics (run last when practical)

If the workflow completes and you can safely write local analytics without relying on
missing gstack binaries, append lightweight JSONL records under
`$HOME/.codex/gstack/analytics/`. If you cannot determine a field, omit it instead of
failing the workflow.

## Detect default branch

Before gathering data, detect the repo's default branch name:
`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

If this fails, infer from local refs:
- Prefer `origin/HEAD`
- Then prefer an existing local branch in this order: `main`, `master`, `trunk`, `develop`
- If none exist, use the current branch as a best-effort fallback and say so clearly

Set both `DEFAULT_BRANCH` and `DEFAULT_REF` before continuing:

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || true)
[ -z "$DEFAULT_BRANCH" ] && DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's#^refs/remotes/origin/##')
[ -z "$DEFAULT_BRANCH" ] && for _CANDIDATE in main master trunk develop; do git show-ref --verify --quiet "refs/heads/$_CANDIDATE" && DEFAULT_BRANCH="$_CANDIDATE" && break; done
[ -z "$DEFAULT_BRANCH" ] && DEFAULT_BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
DEFAULT_REF="$DEFAULT_BRANCH"
[ -n "$DEFAULT_BRANCH" ] && git show-ref --verify --quiet "refs/remotes/origin/$DEFAULT_BRANCH" && DEFAULT_REF="origin/$DEFAULT_BRANCH"
echo "DEFAULT_BRANCH: $DEFAULT_BRANCH"
echo "DEFAULT_REF: $DEFAULT_REF"
```

---

# /retro — Weekly Engineering Retrospective

Generates a comprehensive engineering retrospective analyzing commit history, work patterns, and code quality metrics. Team-aware: identifies the user running the command, then analyzes every contributor with per-person praise and growth opportunities. Designed for a senior IC/CTO-level builder using Claude Code as a force multiplier.

## User-invocable
When the user types `/retro`, run this skill.

## Arguments
- `/retro` — default: last 7 days
- `/retro 24h` — last 24 hours
- `/retro 14d` — last 14 days
- `/retro 30d` — last 30 days
- `/retro compare` — compare current window vs prior same-length window
- `/retro compare 14d` — compare with explicit window

## Instructions

Parse the argument to determine the time window. Default to 7 days if no argument given. All times should be reported in the user's **local timezone** (use the system default — do NOT set `TZ`).

**Midnight-aligned windows:** For day (`d`) and week (`w`) units, compute an absolute start date at local midnight, not a relative string. For example, if today is 2026-03-18 and the window is 7 days: the start date is 2026-03-11. Use `--since="2026-03-11T00:00:00"` for git log queries — the explicit `T00:00:00` suffix ensures git starts from midnight. Without it, git uses the current wall-clock time (e.g., `--since="2026-03-11"` at 11pm means 11pm, not midnight). For week units, multiply by 7 to get days (e.g., `2w` = 14 days back). For hour (`h`) units, use `--since="N hours ago"` since midnight alignment does not apply to sub-day windows.

**Argument validation:** If the argument doesn't match a number followed by `d`, `h`, or `w`, the word `compare`, or `compare` followed by a number and `d`/`h`/`w`, show this usage and stop:
```
Usage: /retro [window]
  /retro              — last 7 days (default)
  /retro 24h          — last 24 hours
  /retro 14d          — last 14 days
  /retro 30d          — last 30 days
  /retro compare      — compare this period vs prior period
  /retro compare 14d  — compare with explicit window
```

### Step 1: Gather Raw Data

First, fetch origin and identify the current user:
```bash
git fetch origin "$DEFAULT_BRANCH" --quiet 2>/dev/null || true
# Identify who is running the retro
git config user.name
git config user.email
```

The name returned by `git config user.name` is **"you"** — the person reading this retro. All other authors are teammates. Use this to orient the narrative: "your" commits vs teammate contributions.

Run ALL of these git commands in parallel (they are independent):

```bash
# 1. All commits in window with timestamps, subject, hash, AUTHOR, files changed, insertions, deletions
git log "$DEFAULT_REF" --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# 2. Per-commit test vs total LOC breakdown with author
#    Each commit block starts with COMMIT:<hash>|<author>, followed by numstat lines.
#    Separate test files (matching test/|spec/|__tests__/) from production files.
git log "$DEFAULT_REF" --since="<window>" --format="COMMIT:%H|%aN" --numstat

# 3. Commit timestamps for session detection and hourly distribution (with author)
git log "$DEFAULT_REF" --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# 4. Files most frequently changed (hotspot analysis)
git log "$DEFAULT_REF" --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn

# 5. PR numbers from commit messages (extract #NNN patterns)
git log "$DEFAULT_REF" --since="<window>" --format="%s" | grep -oE '#[0-9]+' | sed 's/^#//' | sort -n | uniq | sed 's/^/#/'

# 6. Per-author file hotspots (who touches what)
git log "$DEFAULT_REF" --since="<window>" --format="AUTHOR:%aN" --name-only

# 7. Per-author commit counts (quick summary)
git shortlog "$DEFAULT_REF" --since="<window>" -sn --no-merges

# 8. Greptile triage history (if available)
if ls "$_GLOBAL_GREPTILE_DIR"/* >/dev/null 2>&1; then
  find "$_GLOBAL_GREPTILE_DIR" -type f 2>/dev/null | sort | while read -r f; do cat "$f"; done
else
  find "$_LOCAL_GREPTILE_DIR" -type f 2>/dev/null | sort | while read -r f; do cat "$f"; done
fi

# 9. TODOS.md backlog (if available)
cat TODOS.md 2>/dev/null || true

# 10. Test file count
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' 2>/dev/null | grep -v node_modules | wc -l

# 11. Regression test commits in window
git log "$DEFAULT_REF" --since="<window>" --oneline --grep="test(qa):" --grep="test(design):" --grep="test: coverage"

# 12. gstack skill usage telemetry (if available)
cat "$_SKILL_USAGE_LOG" 2>/dev/null || true

# 13. Review analytics (if available)
cat "$_REVIEW_LOG" 2>/dev/null || true

# 14. Test files changed in window
git log "$DEFAULT_REF" --since="<window>" --format="" --name-only | grep -E '\.(test|spec)\.' | sort -u | wc -l
```

### Step 2: Compute Metrics

Calculate and present these metrics in a summary table:

| Metric | Value |
|--------|-------|
| Commits to main | N |
| Contributors | N |
| PRs merged | N |
| Total insertions | N |
| Total deletions | N |
| Net LOC added | N |
| Test LOC (insertions) | N |
| Test LOC ratio | N% |
| Version range | vX.Y.Z.W → vX.Y.Z.W |
| Active days | N |
| Detected sessions | N |
| Avg LOC/session-hour | N |
| Greptile signal | N% (Y catches, Z FPs) |
| Test Health | N total tests · M added this period · K regression tests |

Then show a **per-author leaderboard** immediately below:

```
Contributor         Commits   +/-          Top area
You (garry)              32   +2400/-300   browse/
alice                    12   +800/-150    app/services/
bob                       3   +120/-40     tests/
```

Sort by commits descending. The current user (from `git config user.name`) always appears first, labeled "You (name)".

**Greptile signal (if history exists):** Read any entries found under `$_LOCAL_GREPTILE_DIR` and `$_GLOBAL_GREPTILE_DIR` (fetched in Step 1, command 8). Filter entries within the retro time window by date. Count entries by type: `fix`, `fp`, `already-fixed`. Compute signal ratio: `(fix + already-fixed) / (fix + already-fixed + fp)`. If no entries exist in the window, skip the Greptile metric row. Skip unparseable lines silently.

**Backlog Health (if TODOS.md exists):** Read `TODOS.md` (fetched in Step 1, command 9). Compute:
- Total open TODOs (exclude items in `## Completed` section)
- P0/P1 count (critical/urgent items)
- P2 count (important items)
- Items completed this period (items in Completed section with dates within the retro window)
- Items added this period (cross-reference git log for commits that modified TODOS.md within the window)

Include in the metrics table:
```
| Backlog Health | N open (X P0/P1, Y P2) · Z completed this period |
```

If TODOS.md doesn't exist, skip the Backlog Health row.

**Skill Usage (if analytics exist):** Read `$_SKILL_USAGE_LOG` if it exists. Filter entries within the retro time window by `ts` field. Separate skill activations (no `event` field) from hook fires (`event: "hook_fire"`). Aggregate by skill name. If `$_REVIEW_LOG` has richer review completions in the same window, use it to supplement ship/review/qa counts rather than inventing them. Present as:

```
| Skill Usage | /ship(12) /qa(8) /review(5) · 3 safety hook fires |
```

If the JSONL file doesn't exist or has no entries in the window, skip the Skill Usage row.

### Step 3: Commit Time Distribution

Show hourly histogram in local time using bar chart:

```
Hour  Commits  ████████████████
 00:    4      ████
 07:    5      █████
 ...
```

Identify and call out:
- Peak hours
- Dead zones
- Whether pattern is bimodal (morning/evening) or continuous
- Late-night coding clusters (after 10pm)

### Step 4: Work Session Detection

Detect sessions using **45-minute gap** threshold between consecutive commits. For each session report:
- Start/end time (local timezone)
- Number of commits
- Duration in minutes

Classify sessions:
- **Deep sessions** (50+ min)
- **Medium sessions** (20-50 min)
- **Micro sessions** (<20 min, typically single-commit fire-and-forget)

Calculate:
- Total active coding time (sum of session durations)
- Average session length
- LOC per hour of active time

### Step 5: Commit Type Breakdown

Categorize by conventional commit prefix (feat/fix/refactor/test/chore/docs). Show as percentage bar:

```
feat:     20  (40%)  ████████████████████
fix:      27  (54%)  ███████████████████████████
refactor:  2  ( 4%)  ██
```

Flag if fix ratio exceeds 50% — this signals a "ship fast, fix fast" pattern that may indicate review gaps.

### Step 6: Hotspot Analysis

Show top 10 most-changed files. Flag:
- Files changed 5+ times (churn hotspots)
- Test files vs production files in the hotspot list
- VERSION/CHANGELOG frequency (version discipline indicator)

### Step 7: PR Size Distribution

From commit diffs, estimate PR sizes and bucket them:
- **Small** (<100 LOC)
- **Medium** (100-500 LOC)
- **Large** (500-1500 LOC)
- **XL** (1500+ LOC) — flag these with file counts

### Step 8: Focus Score + Ship of the Week

**Focus score:** Calculate the percentage of commits touching the single most-changed top-level directory (e.g., `app/services/`, `app/views/`). Higher score = deeper focused work. Lower score = scattered context-switching. Report as: "Focus score: 62% (app/services/)"

**Ship of the week:** Auto-identify the single highest-LOC PR in the window. Highlight it:
- PR number and title
- LOC changed
- Why it matters (infer from commit messages and files touched)

### Step 9: Team Member Analysis

For each contributor (including the current user), compute:

1. **Commits and LOC** — total commits, insertions, deletions, net LOC
2. **Areas of focus** — which directories/files they touched most (top 3)
3. **Commit type mix** — their personal feat/fix/refactor/test breakdown
4. **Session patterns** — when they code (their peak hours), session count
5. **Test discipline** — their personal test LOC ratio
6. **Biggest ship** — their single highest-impact commit or PR in the window

**For the current user ("You"):** This section gets the deepest treatment. Include all the detail from the solo retro — session analysis, time patterns, focus score. Frame it in first person: "Your peak hours...", "Your biggest ship..."

**For each teammate:** Write 2-3 sentences covering what they worked on and their pattern. Then:

- **Praise** (1-2 specific things): Anchor in actual commits. Not "great work" — say exactly what was good. Examples: "Shipped the entire auth middleware rewrite in 3 focused sessions with 45% test coverage", "Every PR under 200 LOC — disciplined decomposition."
- **Opportunity for growth** (1 specific thing): Frame as a leveling-up suggestion, not criticism. Anchor in actual data. Examples: "Test ratio was 12% this week — adding test coverage to the payment module before it gets more complex would pay off", "5 fix commits on the same file suggest the original PR could have used a review pass."

**If only one contributor (solo repo):** Skip the team breakdown and proceed as before — the retro is personal.

**If there are Co-Authored-By trailers:** Parse `Co-Authored-By:` lines in commit messages. Credit those authors for the commit alongside the primary author. Note AI co-authors (e.g., `noreply@anthropic.com`) but do not include them as team members — instead, track "AI-assisted commits" as a separate metric.

### Step 10: Week-over-Week Trends (if window >= 14d)

If the time window is 14 days or more, split into weekly buckets and show trends:
- Commits per week (total and per-author)
- LOC per week
- Test ratio per week
- Fix ratio per week
- Session count per week

### Step 11: Streak Tracking

Count consecutive days with at least 1 commit to `DEFAULT_REF`, going back from today. Track both team streak and personal streak:

```bash
# Team streak: all unique commit dates (local time) — no hard cutoff
git log "$DEFAULT_REF" --format="%ad" --date=format:"%Y-%m-%d" | sort -u

# Personal streak: only the current user's commits
git log "$DEFAULT_REF" --author="<user_name>" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

Count backward from today — how many consecutive days have at least one commit? This queries the full history so streaks of any length are reported accurately. Display both:
- "Team shipping streak: 47 consecutive days"
- "Your shipping streak: 32 consecutive days"

### Step 12: Load History & Compare

Before saving the new snapshot, check for prior retro history:

```bash
LATEST_RETRO_PATH=$(ls -t "$_GLOBAL_RETRO_DIR"/*.json 2>/dev/null | head -1)
[ -z "$LATEST_RETRO_PATH" ] && LATEST_RETRO_PATH=$(ls -t "$_LOCAL_RETRO_DIR"/*.json 2>/dev/null | head -1)
printf '%s\n' "$LATEST_RETRO_PATH"
```

**If prior retros exist:** Load `LATEST_RETRO_PATH`, which prefers the global canonical store and falls back to the local mirror only when canonical history is absent. Calculate deltas for key metrics and include a **Trends vs Last Retro** section:
```
                    Last        Now         Delta
Test ratio:         22%    →    41%         ↑19pp
Sessions:           10     →    14          ↑4
LOC/hour:           200    →    350         ↑75%
Fix ratio:          54%    →    30%         ↓24pp (improving)
Commits:            32     →    47          ↑47%
Deep sessions:      3      →    5           ↑2
```

**If no prior retros exist:** Skip the comparison section and append: "First retro recorded — run again next week to see trends."

### Step 13: Save Retro History

After computing all metrics (including streak) and loading any prior history for comparison, save a JSON snapshot:

```bash
mkdir -p "$_GLOBAL_RETRO_DIR" "$_LOCAL_RETRO_DIR"
```

Determine the next sequence number for today (substitute the actual date for `$(date +%Y-%m-%d)`):
```bash
# Count existing retros for today to get next sequence number
today=$(date +%Y-%m-%d)
existing=$(ls "$_GLOBAL_RETRO_DIR"/${today}-*.json 2>/dev/null | wc -l | tr -d ' ')
next=$((existing + 1))
RETRO_BASENAME="${today}-${next}.json"
GLOBAL_RETRO_PATH="$_GLOBAL_RETRO_DIR/$RETRO_BASENAME"
LOCAL_RETRO_PATH="$_LOCAL_RETRO_DIR/$RETRO_BASENAME"
```

Write the JSON snapshot to `GLOBAL_RETRO_PATH`, then mirror the exact same file to `LOCAL_RETRO_PATH` with this schema:
```json
{
  "date": "2026-03-08",
  "window": "7d",
  "metrics": {
    "commits": 47,
    "contributors": 3,
    "prs_merged": 12,
    "insertions": 3200,
    "deletions": 800,
    "net_loc": 2400,
    "test_loc": 1300,
    "test_ratio": 0.41,
    "active_days": 6,
    "sessions": 14,
    "deep_sessions": 5,
    "avg_session_minutes": 42,
    "loc_per_session_hour": 350,
    "feat_pct": 0.40,
    "fix_pct": 0.30,
    "peak_hour": 22,
    "ai_assisted_commits": 32
  },
  "authors": {
    "Garry Tan": { "commits": 32, "insertions": 2400, "deletions": 300, "test_ratio": 0.41, "top_area": "browse/" },
    "Alice": { "commits": 12, "insertions": 800, "deletions": 150, "test_ratio": 0.35, "top_area": "app/services/" }
  },
  "version_range": ["1.16.0.0", "1.16.1.0"],
  "streak_days": 47,
  "tweetable": "Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm",
  "greptile": {
    "fixes": 3,
    "fps": 1,
    "already_fixed": 2,
    "signal_pct": 83
  }
}
```

**Note:** Only include the `greptile` field if Greptile history exists under `$_LOCAL_GREPTILE_DIR` or `$_GLOBAL_GREPTILE_DIR` and has entries within the time window. Only include the `backlog` field if `TODOS.md` exists. Only include the `test_health` field if test files were found (command 10 returns > 0). If any has no data, omit the field entirely.

Include test health data in the JSON when test files exist:
```json
  "test_health": {
    "total_test_files": 47,
    "tests_added_this_period": 5,
    "regression_test_commits": 3,
    "test_files_changed": 8
  }
```

Include backlog data in the JSON when TODOS.md exists:
```json
  "backlog": {
    "total_open": 28,
    "p0_p1": 2,
    "p2": 8,
    "completed_this_period": 3,
    "added_this_period": 1
  }
```

### Step 14: Write the Narrative

Structure the output as:

---

**Tweetable summary** (first line, before everything else):
```
Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm | Streak: 47d
```

## Engineering Retro: [date range]

### Summary Table
(from Step 2)

### Trends vs Last Retro
(from Step 12, loaded before save — skip if first retro)

### Time & Session Patterns
(from Steps 3-4)

Narrative interpreting what the team-wide patterns mean:
- When the most productive hours are and what drives them
- Whether sessions are getting longer or shorter over time
- Estimated hours per day of active coding (team aggregate)
- Notable patterns: do team members code at the same time or in shifts?

### Shipping Velocity
(from Steps 5-7)

Narrative covering:
- Commit type mix and what it reveals
- PR size discipline (are PRs staying small?)
- Fix-chain detection (sequences of fix commits on the same subsystem)
- Version bump discipline

### Code Quality Signals
- Test LOC ratio trend
- Hotspot analysis (are the same files churning?)
- Any XL PRs that should have been split
- Greptile signal ratio and trend (if history exists): "Greptile: X% signal (Y valid catches, Z false positives)"

### Test Health
- Total test files: N (from command 10)
- Tests added this period: M (from command 14 — test files changed)
- Regression test commits: list `test(qa):` and `test(design):` and `test: coverage` commits from command 11
- If prior retro exists and has `test_health`: show delta "Test count: {last} → {now} (+{delta})"
- If test ratio < 20%: flag as growth area — "100% test coverage is the goal. Tests make vibe coding safe."

### Focus & Highlights
(from Step 8)
- Focus score with interpretation
- Ship of the week callout

### Your Week (personal deep-dive)
(from Step 9, for the current user only)

This is the section the user cares most about. Include:
- Their personal commit count, LOC, test ratio
- Their session patterns and peak hours
- Their focus areas
- Their biggest ship
- **What you did well** (2-3 specific things anchored in commits)
- **Where to level up** (1-2 specific, actionable suggestions)

### Team Breakdown
(from Step 9, for each teammate — skip if solo repo)

For each teammate (sorted by commits descending), write a section:

#### [Name]
- **What they shipped**: 2-3 sentences on their contributions, areas of focus, and commit patterns
- **Praise**: 1-2 specific things they did well, anchored in actual commits. Be genuine — what would you actually say in a 1:1? Examples:
  - "Cleaned up the entire auth module in 3 small, reviewable PRs — textbook decomposition"
  - "Added integration tests for every new endpoint, not just happy paths"
  - "Fixed the N+1 query that was causing 2s load times on the dashboard"
- **Opportunity for growth**: 1 specific, constructive suggestion. Frame as investment, not criticism. Examples:
  - "Test coverage on the payment module is at 8% — worth investing in before the next feature lands on top of it"
  - "3 of the 5 PRs were 800+ LOC — breaking these up would catch issues earlier and make review easier"
  - "All commits land between 1-4am — sustainable pace matters for code quality long-term"

**AI collaboration note:** If many commits have `Co-Authored-By` AI trailers (e.g., Claude, Copilot), note the AI-assisted commit percentage as a team metric. Frame it neutrally — "N% of commits were AI-assisted" — without judgment.

### Top 3 Team Wins
Identify the 3 highest-impact things shipped in the window across the whole team. For each:
- What it was
- Who shipped it
- Why it matters (product/architecture impact)

### 3 Things to Improve
Specific, actionable, anchored in actual commits. Mix personal and team-level suggestions. Phrase as "to get even better, the team could..."

### 3 Habits for Next Week
Small, practical, realistic. Each must be something that takes <5 minutes to adopt. At least one should be team-oriented (e.g., "review each other's PRs same-day").

### Week-over-Week Trends
(if applicable, from Step 10)

---

## Compare Mode

When the user runs `/retro compare` (or `/retro compare 14d`):

1. Compute metrics for the current window (default 7d) using the midnight-aligned start date (same logic as the main retro — e.g., if today is 2026-03-18 and window is 7d, use `--since="2026-03-11T00:00:00"`)
2. Compute metrics for the immediately prior same-length window using both `--since` and `--until` with midnight-aligned dates to avoid overlap (e.g., for a 7d window starting 2026-03-11: prior window is `--since="2026-03-04T00:00:00" --until="2026-03-11T00:00:00"`)
3. Show a side-by-side comparison table with deltas and arrows
4. Write a brief narrative highlighting the biggest improvements and regressions
5. Save only the current-window snapshot to `$_GLOBAL_RETRO_DIR` and `$_LOCAL_RETRO_DIR` (same as a normal retro run); do **not** persist the prior-window metrics.

## Tone

- Encouraging but candid, no coddling
- Specific and concrete — always anchor in actual commits/code
- Skip generic praise ("great job!") — say exactly what was good and why
- Frame improvements as leveling up, not criticism
- **Praise should feel like something you'd actually say in a 1:1** — specific, earned, genuine
- **Growth suggestions should feel like investment advice** — "this is worth your time because..." not "you failed at..."
- Never compare teammates against each other negatively. Each person's section stands on its own.
- Keep total output around 3000-4500 words (slightly longer to accommodate team sections)
- Use markdown tables and code blocks for data, prose for narrative
- Output directly to the conversation — do NOT write to filesystem except the retro JSON snapshot and its local mirror

## Important Rules

- ALL narrative output goes directly to the user in the conversation. The ONLY files written are the canonical retro JSON snapshot and its repo-local mirror.
- Use `DEFAULT_REF` for all git queries (not a stale local default branch)
- Display all timestamps in the user's local timezone (do not override `TZ`)
- If the window has zero commits, say so and suggest a different window
- Round LOC/hour to nearest 50
- Treat merge commits as PR boundaries
- Do not read CLAUDE.md or other docs — this skill is self-contained
- On first run (no prior retros), skip comparison sections gracefully
