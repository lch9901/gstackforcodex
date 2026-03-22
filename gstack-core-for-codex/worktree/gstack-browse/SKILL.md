---
name: browse
description: |
  Fast headless browser for QA testing and site dogfooding. Navigate any URL, interact with
  elements, verify page state, diff before/after actions, take annotated screenshots, check
  responsive layouts, test forms and uploads, handle dialogs, and assert element states.
  ~100ms per command. Use when you need to test a feature, verify a deployment, dogfood a
  user flow, or file a bug with evidence. Use when asked to "open in browser", "test the
  site", "take a screenshot", or "dogfood this".
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Codex Compatibility

This version preserves the gstack workflow while removing Claude-specific product shell,
telemetry prompts, contributor flows, upgrade flows, and host-only interaction APIs.

### Conversational Decision Protocol

Whenever the original skill says `AskUserQuestion`, ask directly in the normal chat.

Use this structure:
1. **Re-ground:** Briefly restate the project, current branch, current browser-testing phase, and the decision in front of the user.
2. **Simplify:** Explain the browser or QA question in plain English without implementation-heavy jargon.
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
- Treat real browser verification, screenshots, responsive passes, console checks, and evidence capture as cheap wins.
- Distinguish **boilable lakes** from **unbounded oceans**. A lake is a bounded QA or browser-debugging pass on a specific workflow or page. An ocean is indefinite site exploration without success criteria or environment access.
- When estimating effort, mention both human-team effort and AI-assisted effort if it helps the user choose.

Anti-patterns:
- Recommending a shortcut only because it is shorter.
- Skipping verification the user explicitly asked for.
- Pretending the environment supports browser automation when it does not.

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
artifacts, cookie state, or analytics. Do not assume variables from an earlier step still exist.

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
_LOCAL_SCREENSHOT_DIR="$_LOCAL_BROWSE_DIR/screenshots"
_GLOBAL_BROWSE_DIR="$_GLOBAL_PROJECT_DIR/browse"
_ANALYTICS_DIR="$HOME/.codex/gstack/analytics"
_REVIEW_LOG="$_ANALYTICS_DIR/reviews.jsonl"
mkdir -p "$_GLOBAL_PROJECT_DIR" "$_LOCAL_GSTACK_DIR" "$_LOCAL_BROWSE_DIR" "$_LOCAL_SCREENSHOT_DIR" "$_GLOBAL_BROWSE_DIR" "$_ANALYTICS_DIR"
echo "ROOT: $_ROOT"
echo "BRANCH: $_BRANCH"
echo "GLOBAL_PROJECT_DIR: $_GLOBAL_PROJECT_DIR"
echo "LOCAL_BROWSE_DIR: $_LOCAL_BROWSE_DIR"
echo "LOCAL_SCREENSHOT_DIR: $_LOCAL_SCREENSHOT_DIR"
echo "GLOBAL_BROWSE_DIR: $_GLOBAL_BROWSE_DIR"
echo "REVIEW_LOG: $_REVIEW_LOG"
```

Registry policy:
- `$_GLOBAL_PROJECT_DIR` is the source of truth for cross-session gstack history.
- `$_LOCAL_BROWSE_DIR` stores repo-local browser artifacts, notes, and session outputs.
- `$_GLOBAL_BROWSE_DIR` stores canonical browser artifacts worth preserving across sessions.

## Analytics (run last when practical)

If the workflow completes and you can safely write local analytics without relying on
missing gstack binaries, append lightweight JSONL records under
`$HOME/.codex/gstack/analytics/`. If you cannot determine a field, omit it instead of
failing the workflow.

# browse: QA Testing & Dogfooding

Persistent headless Chromium when the bundled gstack backend is available. First call
auto-starts (~3s), then ~100ms per command. Equivalent Codex browser backends should
preserve as much session state as they support; if persistence is weaker, say so before
continuing with a multi-step flow.

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
1. Tell the user browser automation is unavailable through the bundled gstack binary in the current environment.
2. If an equivalent Codex browser or Playwright tool is available, use that instead. For the rest of this skill, treat `$B` as shorthand for that chosen browser automation backend and still produce equivalent navigation, snapshots, screenshots, console checks, network checks, cookie reads, and viewport changes.
3. If the gstack browse binary already exists elsewhere and can be invoked safely, point `B` at that binary and continue.
4. Otherwise stop and tell the user this browse workflow is blocked until browser automation is available.

**Backend translation rule:**
- If you are using the bundled gstack browse binary, the examples below can be executed literally as shell commands.
- If you are using Codex browser tools or Playwright instead, translate each `$B ...` example into the equivalent browser action sequence rather than trying to execute the literal shell syntax.
- Preserve the workflow intent even if the command surface differs: `goto` means navigate, `snapshot` means produce an inspectable DOM/accessibility view, `console` means inspect browser console output, `network` means inspect requests, and `responsive` means capture mobile/tablet/desktop views.

## Core QA Patterns

### 1. Verify a page loads correctly
```bash
$B goto https://yourapp.com
$B text                          # content loads?
$B console                       # JS errors?
$B network                       # failed requests?
$B is visible ".main-content"    # key elements present?
```

### 2. Test a user flow
```bash
$B goto https://app.com/login
$B snapshot -i                   # see all interactive elements
$B fill @e3 "user@test.com"
$B fill @e4 "password"
$B click @e5                     # submit
$B snapshot -D                   # diff: what changed after submit?
$B is visible ".dashboard"       # success state present?
```

### 3. Verify an action worked
```bash
$B snapshot                      # baseline
$B click @e3                     # do something
$B snapshot -D                   # unified diff shows exactly what changed
```

### 4. Visual evidence for bug reports
```bash
$B snapshot -i -a -o /tmp/annotated.png   # labeled screenshot
$B screenshot /tmp/bug.png                # plain screenshot
$B console                                # error log
```

### 5. Find all clickable elements (including non-ARIA)
```bash
$B snapshot -C                   # finds divs with cursor:pointer, onclick, tabindex
$B click @c1                     # interact with them
```

### 6. Assert element states
```bash
$B is visible ".modal"
$B is enabled "#submit-btn"
$B is disabled "#submit-btn"
$B is checked "#agree-checkbox"
$B is editable "#name-field"
$B is focused "#search-input"
$B js "document.body.textContent.includes('Success')"
```

### 7. Test responsive layouts
```bash
$B responsive /tmp/layout        # mobile + tablet + desktop screenshots
$B viewport 375x812              # or set specific viewport
$B screenshot /tmp/mobile.png
```

### 8. Test file uploads
```bash
$B upload "#file-input" /path/to/file.pdf
$B is visible ".upload-success"
```

### 9. Test dialogs
```bash
$B dialog-accept "yes"           # set up handler
$B click "#delete-button"        # trigger dialog
$B dialog                        # see what appeared
$B snapshot -D                   # verify deletion happened
```

### 10. Compare environments
```bash
$B diff https://staging.app.com https://prod.app.com
```

### 11. Show screenshots to the user
After `$B screenshot`, `$B snapshot -a -o`, or `$B responsive`, always surface the saved PNG paths to the user and display the images inline when the host supports local image rendering. Without this, the screenshots are easy to miss.

## User Handoff

When you hit something you can't handle in headless mode (CAPTCHA, complex auth, multi-factor
login), hand off to the user:

```bash
# 1. Open a visible Chrome at the current page
$B handoff "Stuck on CAPTCHA at login page"

# 2. Tell the user what happened directly in chat
#    "I've opened Chrome at the login page. Please solve the CAPTCHA
#     and let me know when you're done."

# 3. When user says "done", re-snapshot and continue
$B resume
```

**When to use handoff:**
- CAPTCHAs or bot detection
- Multi-factor authentication (SMS, authenticator app)
- OAuth flows that require user interaction
- Complex interactions the AI can't handle after 3 attempts

On the bundled gstack backend, the browser preserves all state (cookies, localStorage,
tabs) across the handoff. After `resume`, you get a fresh snapshot of wherever the user
left off.

If the active backend does not support a native `handoff`/`resume` flow:
- Pause and tell the user exactly what manual browser step is needed.
- If session export or cookie import is supported, ask the user to complete the step in their own browser and then import or restore session state before continuing.
- If the backend cannot restore state after manual takeover, stop and report `BLOCKED` rather than pretending the original gstack handoff guarantee still holds.

## Snapshot Flags

The snapshot is your primary tool for understanding and interacting with pages.

```
-i        --interactive           Interactive elements only (buttons, links, inputs) with @e refs
-c        --compact               Compact (no empty structural nodes)
-d <N>    --depth                 Limit tree depth (0 = root only, default: unlimited)
-s <sel>  --selector              Scope to CSS selector
-D        --diff                  Unified diff against previous snapshot (first call stores baseline)
-a        --annotate              Annotated screenshot with red overlay boxes and ref labels
-o <path> --output                Output path for annotated screenshot (default: <temp>/browse-annotated.png)
-C        --cursor-interactive    Cursor-interactive elements (@c refs — divs with pointer, onclick)
```

All flags can be combined freely. `-o` only applies when `-a` is also used.
Example: `$B snapshot -i -a -C -o /tmp/annotated.png`

**Ref numbering:** @e refs are assigned sequentially (@e1, @e2, ...) in tree order.
@c refs from `-C` are numbered separately (@c1, @c2, ...).

After snapshot, use @refs as selectors in any command:
```bash
$B click @e3       $B fill @e4 "value"     $B hover @e1
$B html @e2        $B css @e5 "color"      $B attrs @e6
$B click @c1       # cursor-interactive ref (from -C)
```

**Output format:** indented accessibility tree with @ref IDs, one element per line.
```
  @e1 [heading] "Welcome" [level=1]
  @e2 [textbox] "Email"
  @e3 [button] "Submit"
```

Refs are invalidated on navigation — run `snapshot` again after `goto`.

## Full Command List

### Navigation
| Command | Description |
|---------|-------------|
| `back` | History back |
| `forward` | History forward |
| `goto <url>` | Navigate to URL |
| `reload` | Reload page |
| `url` | Print current URL |

### Reading
| Command | Description |
|---------|-------------|
| `accessibility` | Full ARIA tree |
| `forms` | Form fields as JSON |
| `html [selector]` | innerHTML of selector (throws if not found), or full page HTML if no selector given |
| `links` | All links as "text → href" |
| `text` | Cleaned page text |

### Interaction
| Command | Description |
|---------|-------------|
| `click <sel>` | Click element |
| `cookie <name>=<value>` | Set cookie on current page domain |
| `cookie-import <json>` | Import cookies from JSON file |
| `cookie-import-browser [browser] [--domain d]` | Import cookies from Comet, Chrome, Arc, Brave, or Edge (opens picker, or use --domain for direct import). If the active backend does not support browser-cookie import, use `/setup-browser-cookies` or stop and report the limitation. |
| `dialog-accept [text]` | Auto-accept next alert/confirm/prompt. Optional text is sent as the prompt response |
| `dialog-dismiss` | Auto-dismiss next dialog |
| `fill <sel> <val>` | Fill input |
| `header <name>:<value>` | Set custom request header (colon-separated, sensitive values auto-redacted) |
| `hover <sel>` | Hover element |
| `press <key>` | Press key — Enter, Tab, Escape, ArrowUp/Down/Left/Right, Backspace, Delete, Home, End, PageUp, PageDown, or modifiers like Shift+Enter |
| `scroll [sel]` | Scroll element into view, or scroll to page bottom if no selector |
| `select <sel> <val>` | Select dropdown option by value, label, or visible text |
| `type <text>` | Type into focused element |
| `upload <sel> <file> [file2...]` | Upload file(s) |
| `useragent <string>` | Set user agent |
| `viewport <WxH>` | Set viewport size |
| `wait <sel|--networkidle|--load>` | Wait for element, network idle, or page load (timeout: 15s) |

### Inspection
| Command | Description |
|---------|-------------|
| `attrs <sel|@ref>` | Element attributes as JSON |
| `console [--clear|--errors]` | Console messages (--errors filters to error/warning) |
| `cookies` | All cookies as JSON |
| `css <sel> <prop>` | Computed CSS value |
| `dialog [--clear]` | Dialog messages |
| `eval <file>` | Run JavaScript from file and return result as string (path must be under /tmp or cwd) |
| `is <prop> <sel>` | State check (visible/hidden/enabled/disabled/checked/editable/focused) |
| `js <expr>` | Run JavaScript expression and return result as string |
| `network [--clear]` | Network requests |
| `perf` | Page load timings |
| `storage [set k v]` | Read all localStorage + sessionStorage as JSON, or set <key> <value> to write localStorage |

### Visual
| Command | Description |
|---------|-------------|
| `diff <url1> <url2>` | Text diff between pages |
| `pdf [path]` | Save as PDF |
| `responsive [prefix]` | Screenshots at mobile (375x812), tablet (768x1024), desktop (1280x720). Saves as {prefix}-mobile.png etc. |
| `screenshot [--viewport] [--clip x,y,w,h] [selector|@ref] [path]` | Save screenshot (supports element crop via CSS/@ref, --clip region, --viewport) |

### Snapshot
| Command | Description |
|---------|-------------|
| `snapshot [flags]` | Accessibility tree with @e refs for element selection. Flags: -i interactive only, -c compact, -d N depth limit, -s sel scope, -D diff vs previous, -a annotated screenshot, -o path output, -C cursor-interactive @c refs |

### Meta
| Command | Description |
|---------|-------------|
| `chain` | Run commands from JSON stdin. Format: [["cmd","arg1",...],...] |

### Tabs
| Command | Description |
|---------|-------------|
| `closetab [id]` | Close tab |
| `newtab [url]` | Open new tab |
| `tab <id>` | Switch to tab |
| `tabs` | List open tabs |

### Server
| Command | Description |
|---------|-------------|
| `handoff [message]` | Open visible Chrome at current page for user takeover |
| `restart` | Restart server |
| `resume` | Re-snapshot after user takeover, return control to AI |
| `status` | Health check |
| `stop` | Shutdown server |
