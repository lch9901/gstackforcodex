---
name: setup-browser-cookies
description: |
  Import cookies from your real browser (Comet, Chrome, Arc, Brave, Edge) into the
  headless browse session. Opens an interactive picker UI where you select which
  cookie domains to import. Use before QA testing authenticated pages. Use when asked
  to "import cookies", "login to the site", or "authenticate the browser".
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Codex Compatibility

This version preserves the gstack workflow while removing Claude-specific product shell,
telemetry prompts, contributor flows, upgrade flows, and host-only interaction APIs.

### Conversational Decision Protocol

Whenever the original skill says `AskUserQuestion`, ask directly in the normal chat.

Use this structure:
1. **Re-ground:** Briefly restate the project, current branch, current authentication/browser phase, and the decision in front of the user.
2. **Simplify:** Explain the cookie-import or login-state question in plain English without implementation-heavy jargon.
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
- Treat working authenticated browser state, explicit verification, and session reuse as cheap wins.
- Distinguish **boilable lakes** from **unbounded oceans**. A lake is a bounded login-state import for a known browser and domain. An ocean is trying to invent a new session-sync system when the backend does not support browser-cookie import.
- When estimating effort, mention both human-team effort and AI-assisted effort if it helps the user choose.

Anti-patterns:
- Recommending manual login loops when cookie import is already available.
- Pretending browser-cookie import exists on a backend that cannot do it.
- Talking only in abstract setup terms instead of telling the user exactly what state can or cannot be imported.

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

Re-run this bootstrap block before any phase that relies on repository paths, browser
state, cookie-import artifacts, or analytics. Do not assume variables from an earlier step still exist.

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
[ -z "$_ROOT" ] && _ROOT="$PWD"
_BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
_REPO=$(basename "$_ROOT")
_SLUG=$(printf "%s" "$_REPO" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-')
_BRANCH_SAFE=$(printf "%s" "$_BRANCH" | tr '/:' '--')
_GLOBAL_PROJECT_DIR="$HOME/.codex/gstack/projects/$_SLUG"
_LOCAL_GSTACK_DIR="$_ROOT/.codex/gstack"
_LOCAL_BROWSE_DIR="$_LOCAL_GSTACK_DIR/browse"
_LOCAL_COOKIE_DIR="$_LOCAL_BROWSE_DIR/cookies"
_GLOBAL_COOKIE_DIR="$_GLOBAL_PROJECT_DIR/browse/cookies"
_ANALYTICS_DIR="$HOME/.codex/gstack/analytics"
_REVIEW_LOG="$_ANALYTICS_DIR/reviews.jsonl"
mkdir -p "$_GLOBAL_PROJECT_DIR" "$_LOCAL_GSTACK_DIR" "$_LOCAL_BROWSE_DIR" "$_LOCAL_COOKIE_DIR" "$_GLOBAL_COOKIE_DIR" "$_ANALYTICS_DIR"
echo "ROOT: $_ROOT"
echo "BRANCH: $_BRANCH"
echo "GLOBAL_PROJECT_DIR: $_GLOBAL_PROJECT_DIR"
echo "LOCAL_COOKIE_DIR: $_LOCAL_COOKIE_DIR"
echo "GLOBAL_COOKIE_DIR: $_GLOBAL_COOKIE_DIR"
echo "REVIEW_LOG: $_REVIEW_LOG"
```

Registry policy:
- `$_GLOBAL_PROJECT_DIR` is the source of truth for cross-session gstack history.
- `$_LOCAL_COOKIE_DIR` stores repo-local cookie-import notes and optional redacted summaries.
- `$_GLOBAL_COOKIE_DIR` stores canonical cookie-import metadata worth reusing across sessions.

## Analytics (run last when practical)

If the workflow completes and you can safely write local analytics without relying on
missing gstack binaries, append lightweight JSONL records under
`$HOME/.codex/gstack/analytics/`. If you cannot determine a field, omit it instead of
failing the workflow.

# Setup Browser Cookies

Import logged-in sessions from your real Chromium browser into the headless browse session.

## How it works

1. Find the browse binary
2. Run `cookie-import-browser` to detect installed browsers and open the picker UI
3. User selects which cookie domains to import in their browser
4. Cookies are decrypted and loaded into the active browser automation session

## Steps

### 1. Find the browse binary

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
1. Tell the user browser automation is unavailable through the bundled gstack browse binary in the current environment.
2. If an equivalent Codex browser or Playwright backend is available and supports browser-cookie import, use that instead. For the rest of this skill, treat `$B` as shorthand for that chosen backend.
3. If the gstack browse binary already exists elsewhere and can be invoked safely, point `B` at that binary and continue.
4. Otherwise stop and tell the user this workflow is blocked until a browser backend with cookie-import support is available.

**Backend compatibility rule:**
- If you are using the bundled gstack browse binary, the examples below can be executed literally.
- If you are using Codex browser tools or Playwright instead, only continue if that backend truly supports importing cookies from a real browser profile or an equivalent session restore path.
- If the active backend can automate pages but cannot import browser cookies, do not pretend this skill still works. Stop and report `BLOCKED`, or redirect the user to a manual login workflow.

### 2. Open the cookie picker

```bash
$B cookie-import-browser
```

This auto-detects installed Chromium browsers (Comet, Chrome, Arc, Brave, Edge) and opens
an interactive picker UI in your default browser where you can:
- Switch between installed browsers
- Search domains
- Click "+" to import a domain's cookies
- Click trash to remove imported cookies

Tell the user: **"Cookie picker opened — select the domains you want to import in your browser, then tell me when you're done."**

If the active backend does not support an interactive picker UI:
- Explain the limitation directly.
- If it supports direct import by browser and domain, switch to that path instead.
- If it supports only JSON cookie import, ask the user for a compatible export file and use `cookie-import <json>`.
- Otherwise stop and report `BLOCKED`.

### 3. Direct import (alternative)

If the user specifies a domain directly (e.g., `/setup-browser-cookies github.com`), skip the UI:

```bash
$B cookie-import-browser comet --domain github.com
```

Replace `comet` with the appropriate browser if specified.

If the backend supports only a subset of browsers, say so before continuing. Do not imply Chrome/Arc/Brave/Edge parity unless you have it.

### 4. Verify

After the user confirms they're done:

```bash
$B cookies
```

Show the user a summary of imported cookies (domain counts).

If the active backend cannot list cookies directly:
- Verify by navigating to an authenticated page and confirming the session is live.
- Tell the user that verification is indirect because raw cookie enumeration is unavailable on this backend.

If available, save a redacted import summary under `$_LOCAL_COOKIE_DIR` and optionally mirror it to `$_GLOBAL_COOKIE_DIR`. Never persist raw cookie values.

## Notes

- First import per browser may trigger a macOS Keychain dialog — click "Allow" / "Always Allow"
- On the bundled gstack backend, the cookie picker is served on the same port as the browse server (no extra process)
- Only domain names and cookie counts are shown in the UI — no cookie values are exposed
- On the bundled gstack backend, the browse session persists cookies between commands, so imported cookies work immediately. On other backends, state persistence may be weaker; say so before relying on multi-step authenticated flows.
