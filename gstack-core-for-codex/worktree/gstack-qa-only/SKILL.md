---
name: qa-only
description: |
  Report-only QA testing. Systematically tests a web application and produces a
  structured report with health score, screenshots, and repro steps — but never
  fixes anything. Use when asked to "just report bugs", "qa report only", or
  "test but don't fix". For the full test-fix-verify loop, use /qa instead.
  Proactively suggest when the user wants a bug report without any code changes.
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Codex Compatibility

This version preserves the gstack workflow while removing Claude-specific product shell,
telemetry prompts, contributor flows, upgrade flows, and host-only interaction APIs.

### Conversational Decision Protocol

Whenever the original skill says `AskUserQuestion`, ask directly in the normal chat.

Use this structure:
1. **Re-ground:** Briefly restate the project, current branch, current QA phase, and the decision in front of the user.
2. **Simplify:** Explain the testing question in plain English without implementation-heavy jargon.
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
- Treat richer evidence, regression comparisons, and edge-case workflow coverage as cheap wins.
- Distinguish **boilable lakes** from **unbounded oceans**. A lake is a complete QA pass for a bounded app surface. An ocean is endless exploratory testing without product boundaries or environment access.
- When estimating effort, mention both human-team effort and AI-assisted effort if it helps the user choose.

Anti-patterns:
- Recommending a shortcut only because it is shorter.
- Skipping browser verification when the user explicitly asked for QA.
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

Re-run this bootstrap block before any phase that relies on repository paths, QA reports,
test-plan history, or analytics. Do not assume variables from an earlier step still exist.

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
[ -z "$_ROOT" ] && _ROOT="$PWD"
_BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
_REPO=$(basename "$_ROOT")
_SLUG=$(printf "%s" "$_REPO" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-')
_BRANCH_SAFE=$(printf "%s" "$_BRANCH" | tr '/:' '--')
_GLOBAL_PROJECT_DIR="$HOME/.codex/gstack/projects/$_SLUG"
_LOCAL_GSTACK_DIR="$_ROOT/.codex/gstack"
_LOCAL_QA_DIR="$_LOCAL_GSTACK_DIR/qa-reports"
_LOCAL_SCREENSHOT_DIR="$_LOCAL_QA_DIR/screenshots"
_GLOBAL_QA_DIR="$_GLOBAL_PROJECT_DIR/qa-reports"
_GLOBAL_TEST_PLAN_DIR="$_GLOBAL_PROJECT_DIR/test-plans"
_LOCAL_TEST_PLAN_DIR="$_LOCAL_GSTACK_DIR/test-plans"
_ANALYTICS_DIR="$HOME/.codex/gstack/analytics"
_REVIEW_LOG="$_ANALYTICS_DIR/reviews.jsonl"
_SUPPORT_DIR="/Users/clawtester/Documents/teamagents/vendor/gstack/qa"
mkdir -p "$_GLOBAL_PROJECT_DIR" "$_LOCAL_GSTACK_DIR" "$_LOCAL_QA_DIR" "$_LOCAL_SCREENSHOT_DIR" "$_GLOBAL_QA_DIR" "$_GLOBAL_TEST_PLAN_DIR" "$_LOCAL_TEST_PLAN_DIR" "$_ANALYTICS_DIR"
echo "ROOT: $_ROOT"
echo "BRANCH: $_BRANCH"
echo "GLOBAL_PROJECT_DIR: $_GLOBAL_PROJECT_DIR"
echo "LOCAL_QA_DIR: $_LOCAL_QA_DIR"
echo "GLOBAL_QA_DIR: $_GLOBAL_QA_DIR"
echo "GLOBAL_TEST_PLAN_DIR: $_GLOBAL_TEST_PLAN_DIR"
echo "LOCAL_TEST_PLAN_DIR: $_LOCAL_TEST_PLAN_DIR"
echo "SUPPORT_DIR: $_SUPPORT_DIR"
echo "REVIEW_LOG: $_REVIEW_LOG"
```

Registry policy:
- `$_GLOBAL_PROJECT_DIR` is the source of truth for cross-session gstack history.
- `$_GLOBAL_TEST_PLAN_DIR` is the canonical store for eng-review test plans.
- `$_LOCAL_TEST_PLAN_DIR` mirrors canonical test plans into the repo.
- `$_LOCAL_QA_DIR` stores repo-local QA reports and screenshots.
- `$_GLOBAL_QA_DIR` stores canonical QA reports and baselines.

## Analytics (run last when practical)

If the workflow completes and you can safely write local analytics without relying on
missing gstack binaries, append lightweight JSONL records under
`$HOME/.codex/gstack/analytics/`. If you cannot determine a field, omit it instead of
failing the workflow.

## Step 0: Detect base branch

Determine which branch this QA run should compare against for diff-aware mode.

1. Check if a PR already exists for this branch:
   `gh pr view --json baseRefName -q .baseRefName`
2. If no PR exists, detect the repo's default branch:
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
3. If `gh` is unavailable or both commands fail, infer from local refs:
   - Prefer `origin/HEAD`
   - Then prefer an existing local branch in this order: `main`, `master`, `trunk`, `develop`
   - If none exist, use the current branch as a best-effort fallback and note the limitation
4. Set `BASE_BRANCH` and `BASE_REF` and echo them:

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

# /qa-only: Report-Only QA Testing

You are a QA engineer. Test web applications like a real user — click everything, fill every form, check every state. Produce a structured report with evidence. **NEVER fix anything.**

## Setup

**Parse the user's request for these parameters:**

| Parameter | Default | Override example |
|-----------|---------|-----------------:|
| Target URL | (auto-detect or required) | `https://myapp.com`, `http://localhost:3000` |
| Mode | full | `--quick`, `--regression $_ROOT/.codex/gstack/qa-reports/baseline.json` |
| Output dir | `$_ROOT/.codex/gstack/qa-reports/` | `Output to /tmp/qa` |
| Scope | Full app (or diff-scoped) | `Focus on the billing page` |
| Auth | None | `Sign in to user@example.com`, `Import cookies from cookies.json` |

**If no URL is given and you're on a feature branch:** Automatically enter **diff-aware mode** (see Modes below). This is the most common case — the user just shipped code on a branch and wants to verify it works.

**If no URL is given and you're on the base branch:** Ask the user for a URL, then stop and wait for their reply before continuing.

**Find the browse binary:**

## SETUP (run this check BEFORE any browse command)

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.agents/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.agents/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.codex/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

If `NEEDS_SETUP`:
1. Tell the user browser automation is unavailable in the current environment.
2. If an equivalent Codex browser or Playwright tool is available, use that instead. For the rest of this skill, treat `$B` as shorthand for that chosen browser automation tool and still produce equivalent screenshots, snapshots, console checks, link maps, and viewport changes.
3. Otherwise stop and tell the user this QA run is blocked until browser automation is available.

**Create output directories:**

```bash
REPORT_DIR="$_ROOT/.codex/gstack/qa-reports"
mkdir -p "$REPORT_DIR/screenshots"
REPORT_FILE="$REPORT_DIR/qa-report-{domain}-{YYYY-MM-DD}.md"
GLOBAL_REPORT_FILE="$_GLOBAL_QA_DIR/${_BRANCH_SAFE}-test-outcome-{datetime}.md"
LOCAL_BASELINE_FILE="$REPORT_DIR/baseline.json"
GLOBAL_BASELINE_FILE="$_GLOBAL_QA_DIR/${_BRANCH_SAFE}-baseline.json"
```

---

## Test Plan Context

Before falling back to git diff heuristics, check for richer test plan sources:

1. **Project-scoped test plans:** Check the canonical project test-plan store for recent `*-test-plan-*.md` files for this repo
   ```bash
   find "$_GLOBAL_TEST_PLAN_DIR" -maxdepth 1 -type f -name "${_BRANCH_SAFE}*test-plan*.md" 2>/dev/null | sort | tail -1
   find "$_GLOBAL_TEST_PLAN_DIR" -maxdepth 1 -type f -name "*test-plan*.md" 2>/dev/null | sort | tail -1
   ```
2. **Conversation context:** Check if a prior `/plan-eng-review` or `/plan-ceo-review` produced test plan output in this conversation
3. **Use whichever source is richer.** Fall back to git diff analysis only if neither is available.

---

## Modes

### Diff-aware (automatic when on a feature branch with no URL)

This is the **primary mode** for developers verifying their work. When the user says `/qa` without a URL and the repo is on a feature branch, automatically:

1. **Analyze the branch diff** to understand what changed:
   ```bash
   git diff "$BASE_REF"...HEAD --name-only
   git diff "$BASE_REF" --name-only
   git log "$BASE_REF"..HEAD --oneline
   ```

2. **Identify affected pages/routes** from the changed files:
   - Controller/route files → which URL paths they serve
   - View/template/component files → which pages render them
   - Model/service files → which pages use those models (check controllers that reference them)
   - CSS/style files → which pages include those stylesheets
   - API endpoints → test them directly with `$B js "await fetch('/api/...')"`
   - Static pages (markdown, HTML) → navigate to them directly

   **If no obvious pages/routes are identified from the diff:** Do not skip browser testing. The user invoked /qa because they want browser-based verification. Fall back to Quick mode — navigate to the homepage, follow the top 5 navigation targets, check console for errors, and test any interactive elements found. Backend, config, and infrastructure changes affect app behavior — always verify the app still works.

3. **Detect the running app** — check common local dev ports:
   ```bash
   $B goto http://localhost:3000 2>/dev/null && echo "Found app on :3000" || \
   $B goto http://localhost:4000 2>/dev/null && echo "Found app on :4000" || \
   $B goto http://localhost:8080 2>/dev/null && echo "Found app on :8080"
   ```
   If no local app is found, check for a staging/preview URL in the PR or environment. If nothing works, ask the user for the URL and stop until they reply.

4. **Test each affected page/route:**
   - Navigate to the page
   - Take a screenshot
   - Check console for errors
   - If the change was interactive (forms, buttons, flows), test the interaction end-to-end
   - Use `snapshot -D` before and after actions to verify the change had the expected effect

5. **Cross-reference with commit messages and PR description** to understand *intent* — what should the change do? Verify it actually does that.

6. **Check TODOS.md** (if it exists) for known bugs or issues related to the changed files. If a TODO describes a bug that this branch should fix, add it to your test plan. If you find a new bug during QA that isn't in TODOS.md, note it in the report.

7. **Report findings** scoped to the branch changes:
   - "Changes tested: N pages/routes affected by this branch"
   - For each: does it work? Screenshot evidence.
   - Any regressions on adjacent pages?

**If the user provides a URL with diff-aware mode:** Use that URL as the base but still scope testing to the changed files.

### Full (default when URL is provided)
Systematic exploration. Visit every reachable page. Document 5-10 well-evidenced issues. Produce health score. Takes 5-15 minutes depending on app size.

### Quick (`--quick`)
30-second smoke test. Visit homepage + top 5 navigation targets. Check: page loads? Console errors? Broken links? Produce health score. No detailed issue documentation.

### Regression (`--regression <baseline>`)
Run full mode, then load the previous baseline in this order:
1. Explicit baseline path from the user
2. `$GLOBAL_BASELINE_FILE`
3. `$LOCAL_BASELINE_FILE`
Diff: which issues are fixed? Which are new? What's the score delta? Append regression section to report.

---

## Workflow

### Phase 1: Initialize

1. Find browse binary (see Setup above)
2. Create output directories
3. Copy the report template from `$_SUPPORT_DIR/templates/qa-report-template.md` to `REPORT_FILE`.
   If the template cannot be read, stop and report the error.
4. Start timer for duration tracking

### Phase 2: Authenticate (if needed)

**If the user specified auth credentials:**

```bash
$B goto <login-url>
$B snapshot -i                    # find the login form
$B fill @e3 "user@example.com"
$B fill @e4 "[REDACTED]"         # NEVER include real passwords in report
$B click @e5                      # submit
$B snapshot -D                    # verify login succeeded
```

**If the user provided a cookie file:**

```bash
$B cookie-import cookies.json
$B goto <target-url>
```

**If 2FA/OTP is required:** Ask the user for the code and wait.

**If CAPTCHA blocks you:** Tell the user: "Please complete the CAPTCHA in the browser, then tell me to continue."

### Phase 3: Orient

Get a map of the application:

```bash
$B goto <target-url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/initial.png"
$B links                          # map navigation structure
$B console --errors               # any errors on landing?
```

**Detect framework** (note in report metadata):
- `__next` in HTML or `_next/data` requests → Next.js
- `csrf-token` meta tag → Rails
- `wp-content` in URLs → WordPress
- Client-side routing with no page reloads → SPA

**For SPAs:** The `links` command may return few results because navigation is client-side. Use `snapshot -i` to find nav elements (buttons, menu items) instead.

### Phase 4: Explore

Visit pages systematically. At each page:

```bash
$B goto <page-url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/page-name.png"
$B console --errors
```

Then follow the **per-page exploration checklist** from `$_SUPPORT_DIR/references/issue-taxonomy.md`.
If the taxonomy file cannot be read, continue with the built-in checklist in this document and note the missing reference in the report metadata.

1. **Visual scan** — Look at the annotated screenshot for layout issues
2. **Interactive elements** — Click buttons, links, controls. Do they work?
3. **Forms** — Fill and submit. Test empty, invalid, edge cases
4. **Navigation** — Check all paths in and out
5. **States** — Empty state, loading, error, overflow
6. **Console** — Any new JS errors after interactions?
7. **Responsiveness** — Check mobile viewport if relevant:
   ```bash
   $B viewport 375x812
   $B screenshot "$REPORT_DIR/screenshots/page-mobile.png"
   $B viewport 1280x720
   ```

**Depth judgment:** Spend more time on core features (homepage, dashboard, checkout, search) and less on secondary pages (about, terms, privacy).

**Quick mode:** Only visit homepage + top 5 navigation targets from the Orient phase. Skip the per-page checklist — just check: loads? Console errors? Broken links visible?

### Phase 5: Document

Document each issue **immediately when found** — don't batch them.

**Two evidence tiers:**

**Interactive bugs** (broken flows, dead buttons, form failures):
1. Take a screenshot before the action
2. Perform the action
3. Take a screenshot showing the result
4. Use `snapshot -D` to show what changed
5. Write repro steps referencing screenshots

```bash
$B screenshot "$REPORT_DIR/screenshots/issue-001-step-1.png"
$B click @e5
$B screenshot "$REPORT_DIR/screenshots/issue-001-result.png"
$B snapshot -D
```

**Static bugs** (typos, layout issues, missing images):
1. Take a single annotated screenshot showing the problem
2. Describe what's wrong

```bash
$B snapshot -i -a -o "$REPORT_DIR/screenshots/issue-002.png"
```

**Write each issue to the report immediately** using the template format from `$_SUPPORT_DIR/templates/qa-report-template.md`.

### Phase 6: Wrap Up

1. **Compute health score** using the rubric below
2. **Write "Top 3 Things to Fix"** — the 3 highest-severity issues
3. **Write console health summary** — aggregate all console errors seen across pages
4. **Update severity counts** in the summary table
5. **Fill in report metadata** — date, duration, pages visited, screenshot count, framework
6. **Save baseline** — write both `$LOCAL_BASELINE_FILE` and `$GLOBAL_BASELINE_FILE` with:
   ```json
   {
     "date": "YYYY-MM-DD",
     "url": "<target>",
     "healthScore": N,
     "issues": [{ "id": "ISSUE-001", "title": "...", "severity": "...", "category": "..." }],
     "categoryScores": { "console": N, "links": N, ... }
   }
   ```

**Regression mode:** After writing the report, load the selected baseline file. Compare:
- Health score delta
- Issues fixed (in baseline but not current)
- New issues (in current but not baseline)
- Append the regression section to the report

---

## Health Score Rubric

Compute each category score (0-100), then take the weighted average.

### Console (weight: 15%)
- 0 errors → 100
- 1-3 errors → 70
- 4-10 errors → 40
- 10+ errors → 10

### Links (weight: 10%)
- 0 broken → 100
- Each broken link → -15 (minimum 0)

### Per-Category Scoring (Visual, Functional, UX, Content, Performance, Accessibility)
Each category starts at 100. Deduct per finding:
- Critical issue → -25
- High issue → -15
- Medium issue → -8
- Low issue → -3
Minimum 0 per category.

### Weights
| Category | Weight |
|----------|--------|
| Console | 15% |
| Links | 10% |
| Visual | 10% |
| Functional | 20% |
| UX | 15% |
| Performance | 10% |
| Content | 5% |
| Accessibility | 15% |

### Final Score
`score = Σ (category_score × weight)`

---

## Framework-Specific Guidance

### Next.js
- Check console for hydration errors (`Hydration failed`, `Text content did not match`)
- Monitor `_next/data` requests in network — 404s indicate broken data fetching
- Test client-side navigation (click links, don't just `goto`) — catches routing issues
- Check for CLS (Cumulative Layout Shift) on pages with dynamic content

### Rails
- Check for N+1 query warnings in console (if development mode)
- Verify CSRF token presence in forms
- Test Turbo/Stimulus integration — do page transitions work smoothly?
- Check for flash messages appearing and dismissing correctly

### WordPress
- Check for plugin conflicts (JS errors from different plugins)
- Verify admin bar visibility for logged-in users
- Test REST API endpoints (`/wp-json/`)
- Check for mixed content warnings (common with WP)

### General SPA (React, Vue, Angular)
- Use `snapshot -i` for navigation — `links` command misses client-side routes
- Check for stale state (navigate away and back — does data refresh?)
- Test browser back/forward — does the app handle history correctly?
- Check for memory leaks (monitor console after extended use)

---

## Important Rules

1. **Repro is everything.** Every issue needs at least one screenshot. No exceptions.
2. **Verify before documenting.** Retry the issue once to confirm it's reproducible, not a fluke.
3. **Never include credentials.** Write `[REDACTED]` for passwords in repro steps.
4. **Write incrementally.** Append each issue to the report as you find it. Don't batch.
5. **Never read source code.** Test as a user, not a developer.
6. **Check console after every interaction.** JS errors that don't surface visually are still bugs.
7. **Test like a user.** Use realistic data. Walk through complete workflows end-to-end.
8. **Depth over breadth.** 5-10 well-documented issues with evidence > 20 vague descriptions.
9. **Never delete output files.** Screenshots and reports accumulate — that's intentional.
10. **Use `snapshot -C` for tricky UIs.** Finds clickable divs that the accessibility tree misses.
11. **Show screenshots to the user.** After every `$B screenshot`, `$B snapshot -a -o`, or `$B responsive` command, use the Read tool on the output file(s) so the user can see them inline. For `responsive` (3 files), Read all three. This is critical — without it, screenshots are invisible to the user.
12. **Never refuse to use the browser.** When the user invokes /qa or /qa-only, they are requesting browser-based testing. Never suggest evals, unit tests, or other alternatives as a substitute. Even if the diff appears to have no UI changes, backend changes affect app behavior — always open the browser and test.

---

## Output

Write the report to both local and project-scoped locations:

**Local:** `REPORT_FILE`

**Project-scoped:** Write test outcome artifact for cross-session context:
```bash
mkdir -p "$_GLOBAL_QA_DIR"
```
Write the same final report contents to:
- `$REPORT_FILE`
- `$GLOBAL_REPORT_FILE`

### Output Structure

```
.codex/gstack/qa-reports/
├── qa-report-{domain}-{YYYY-MM-DD}.md    # Structured report
├── screenshots/
│   ├── initial.png                        # Landing page annotated screenshot
│   ├── issue-001-step-1.png               # Per-issue evidence
│   ├── issue-001-result.png
│   └── ...
└── baseline.json                          # Local regression baseline mirror
```

Report filenames use the domain and date: `qa-report-myapp-com-2026-03-12.md`

---

## Additional Rules (qa-only specific)

11. **Never fix bugs.** Find and document only. Do not read source code, edit files, or suggest fixes in the report. Your job is to report what's broken, not to fix it. Use `/qa` for the test-fix-verify loop.
12. **No test framework detected?** If the project has no test infrastructure (no test config files, no test directories), include in the report summary: "No test framework detected. Run `/qa` to bootstrap one and enable regression test generation."
