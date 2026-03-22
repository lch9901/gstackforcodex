# gstackforcodex

High-fidelity Codex adaptation of Garry Tan's [`gstack`](https://github.com/garrytan/gstack).

This repo keeps the original `gstack` workflow philosophy intact and ports it to Codex with
minimal behavioral drift. The goal is not to invent a new workflow system. The goal is to
preserve `gstack`'s product, planning, design, QA, and release flow while replacing
Claude-specific host assumptions with Codex-compatible behavior.

## What This Repo Is

This repository contains:

- a pinned upstream reference for `gstack`
- an untouched upstream Codex baseline
- a Codex-compatible worktree with adapted skills

The main deliverable is the adapted skill set in
[`gstack-core-for-codex/worktree`](./gstack-core-for-codex/worktree).

## Why This Exists

`gstack` already has a strong workflow design, but several of its skills rely on host-specific
behavior that does not transfer cleanly to Codex, including:

- `AskUserQuestion`
- hooks-based safety flows
- product-shell setup and telemetry helpers
- browser/runtime assumptions tied to the original host

This repo keeps the workflow brain and swaps out the host plumbing.

## Included Skill Set

The current migration covers 15 core skills.

### Product and Direction

- `gstack-office-hours`
- `gstack-plan-ceo-review`

### Architecture and Planning

- `gstack-plan-eng-review`
- `gstack-plan-design-review`

### Design

- `gstack-design-consultation`
- `gstack-design-review`

### Engineering Quality

- `gstack-review`
- `gstack-investigate`
- `gstack-qa`
- `gstack-qa-only`

### Release and Learning

- `gstack-ship`
- `gstack-document-release`
- `gstack-retro`

### Browser Foundation

- `gstack-browse`
- `gstack-setup-browser-cookies`

## Repository Layout

- [`gstack-core-for-codex`](./gstack-core-for-codex): migration workspace
- [`gstack-core-for-codex/upstream-codex-baseline`](./gstack-core-for-codex/upstream-codex-baseline): untouched upstream Codex-generated skill copies
- [`gstack-core-for-codex/worktree`](./gstack-core-for-codex/worktree): adapted Codex-compatible skills
- `vendor/gstack`: local upstream reference checkout

## Adaptation Rules

The migration follows a small set of strict rules:

1. Preserve workflow structure, role boundaries, and decision gates.
2. Replace Claude-only interaction APIs with normal Codex chat decisions.
3. Replace old `~/.gstack` assumptions with `~/.codex/gstack/...` plus repo-local `.codex/gstack/...`.
4. Prefer global canonical history plus repo-local mirrors for continuity across sessions.
5. Make each skill executable in Codex, not just descriptively similar.

## Current Status

All 15 target skills have been reviewed and adapted into the Codex worktree.

The migration currently includes:

- high-fidelity compatibility rewrites
- host-coupling cleanup
- path and artifact migration
- walkthrough-based breakpoint checks for each adapted skill

## Recommended First Workflow

For a first real run, start here:

1. `/office-hours`
2. `/plan-ceo-review`
3. `/plan-eng-review`

Then continue into:

- `/plan-design-review`
- `/design-consultation`
- `/review`
- `/qa-only` or `/qa`
- `/ship`
- `/document-release`
- `/retro`

## How To Use In A New Codex Session

Open a new Codex session in this repo and tell Codex to treat the adapted skill files as the
workflow source.

Suggested activation prompt:

```text
This session should use ./gstack-core-for-codex/worktree as the workflow source.

When I invoke a gstack skill name such as /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review, /design-consultation, /design-review, /review, /investigate, /qa-only, /qa, /ship, /document-release, /retro, /browse, or /setup-browser-cookies:

1. Read the corresponding SKILL.md first.
2. Follow it with high fidelity.
3. Do not replace it with your default workflow.
4. If the skill has decision gates, ask one question at a time.
5. Prefer ~/.codex/gstack/projects/<slug>/ and repo-local .codex/gstack/ history when the skill expects prior artifacts.
```

Then invoke the first skill directly, for example:

```text
/office-hours

I want to build a small tool that grabs podcast content, turns it into Chinese text, and then produces a social-media-ready Chinese post.
```

## Upstream Reference

- Upstream repo: [garrytan/gstack](https://github.com/garrytan/gstack)
- Internal migration notes: [`gstack-core-for-codex/README.md`](./gstack-core-for-codex/README.md)
- Skill inventory: [`gstack-core-for-codex/SKILLS.md`](./gstack-core-for-codex/SKILLS.md)

## Notes

- This repo intentionally favors workflow fidelity over a ground-up native rewrite.
- The main working artifacts are inside [`gstack-core-for-codex`](./gstack-core-for-codex).
- `vendor/gstack` is a local upstream reference checkout, not the primary deliverable.
