# gstack Codex Audit Matrix

This file records what we found in the first compatibility audit pass over the 15 selected skills in `worktree/`.

## Shared Findings Across All 15 Skills

Every audited skill currently contains the same host-coupled shell:

- A `## Preamble (run first)` block
- `gstack-update-check`
- `gstack-config` reads and writes
- telemetry prompts
- `Boil the Lake` intro flow
- contributor mode logging
- analytics/session file writes under `~/.gstack`
- `AskUserQuestion` instructions in the shared format section

### Shared Rewrite Policy

- `Preamble`: delete and replace with a short Codex-native context rule when needed
- telemetry, upgrade, analytics, contributor mode: delete
- `AskUserQuestion Format`: rewrite into a plain conversational decision protocol
- `Completeness Principle`: keep, but strip references to gstack product onboarding and `CC+gstack`
- proactive suggestion toggles: delete

## Rewrite Buckets

### Bucket A: Light Rewrite

These skills mainly need the shared shell removed plus a small number of `AskUserQuestion` rewrites.

- `gstack-browse`
- `gstack-qa-only`
- `gstack-review`
- `gstack-setup-browser-cookies`
- `gstack-retro`
- `gstack-investigate`
- `gstack-design-consultation`

### Bucket B: Medium Rewrite

These skills depend more heavily on staged user decisions or on runtime/project-state branching.

- `gstack-office-hours`
- `gstack-design-review`
- `gstack-document-release`
- `gstack-qa`

### Bucket C: Heavy Rewrite

These are the most Claude-shaped skills. They have many `AskUserQuestion` gates and more host-specific workflow assumptions.

- `gstack-plan-ceo-review`
- `gstack-plan-design-review`
- `gstack-plan-eng-review`
- `gstack-ship`

## Skill-by-Skill Matrix

| Skill | AskUserQuestion Count | Rewrite Level | Keep | Rewrite | Delete |
|---|---:|---|---|---|---|
| `gstack-office-hours` | 14 | Medium | Core forcing questions, reframing flow, output structure | Every decision gate and confirmation prompt | Full preamble, telemetry, contributor mode |
| `gstack-plan-ceo-review` | 30 | Heavy | Expansion/reduction modes, vision-first review structure, score/routing logic | All question gates, review toggles, next-step prompts | Full preamble, telemetry, contributor mode, config toggles |
| `gstack-plan-eng-review` | 23 | Heavy | Architecture/data-flow/edge-case review structure | Question gates, next-step prompts, review toggle references | Full preamble, telemetry, contributor mode, config toggles |
| `gstack-plan-design-review` | 20 | Heavy | Design-dimension scoring, 10/10 framing, issue-by-issue critique | All question gates, TODO presentation flow, review toggle references | Full preamble, telemetry, contributor mode, config toggles |
| `gstack-design-consultation` | 9 | Light | Design system workflow, SAFE/RISK framing, final synthesis | Proposal checkpoints and final confirmation prompts | Full preamble, telemetry, contributor mode |
| `gstack-review` | 9 | Light | Review methodology, findings-first discipline | ASK-item and false-positive flows | Full preamble, telemetry, contributor mode |
| `gstack-investigate` | 9 | Light | Root-cause discipline, 3-strike rule, blast-radius reasoning | Stop points that currently require AskUserQuestion | Full preamble, telemetry, contributor mode |
| `gstack-design-review` | 10 | Medium | Visual audit rubric, fix loop logic, before/after mindset | Dirty tree, runtime detection, auth prompts, fix confirmation gates | Full preamble, telemetry, contributor mode |
| `gstack-qa` | 9 | Medium | QA methodology, bug-fix-reverify loop, regression-test expectations | Dirty tree, runtime detection, user decision prompts | Full preamble, telemetry, contributor mode |
| `gstack-qa-only` | 5 | Light | Report-only QA methodology | Small number of decision prompts | Full preamble, telemetry, contributor mode |
| `gstack-ship` | 15 | Heavy | Release sequencing, readiness logic, preflight structure | All branching prompts, runtime decisions, ASK flows | Full preamble, telemetry, contributor mode, config toggles |
| `gstack-document-release` | 14 | Medium | Docs synchronization logic, stale-doc detection | Contradiction confirmation and narrative decision points | Full preamble, telemetry, contributor mode |
| `gstack-retro` | 5 | Light | Retro structure, trend categories, summary format | Any remaining user-choice prompts | Full preamble, telemetry, contributor mode |
| `gstack-browse` | 6 | Light | Browser command reference and operating model | Small decision prompts only | Full preamble, telemetry, contributor mode |
| `gstack-setup-browser-cookies` | 5 | Light | Cookie import procedure | Small decision prompts only | Full preamble, telemetry, contributor mode |

## Special Notes

### `gstack-browse`

This skill has low dialog complexity, but it is runtime-sensitive. We should preserve the browser usage model while removing the product shell. It can be adapted early.

### `gstack-plan-ceo-review`

This is the most interaction-heavy skill in the set. It should be treated as a flagship workflow port, not a quick cleanup.

### `gstack-plan-design-review`

This skill depends strongly on one-question-per-issue pacing. We need to preserve that pacing in prose without relying on Claude's `AskUserQuestion`.

### `gstack-ship`

This skill is both host-coupled and environment-coupled. It should be migrated after the planning/review skills are stable.

## Recommended Rewrite Order

### Phase 1

- `gstack-office-hours`
- `gstack-plan-ceo-review`
- `gstack-browse`

### Phase 2

- `gstack-plan-eng-review`
- `gstack-plan-design-review`
- `gstack-design-consultation`
- `gstack-review`
- `gstack-investigate`
- `gstack-qa-only`

### Phase 3

- `gstack-design-review`
- `gstack-qa`
- `gstack-ship`
- `gstack-document-release`
- `gstack-retro`
- `gstack-setup-browser-cookies`
