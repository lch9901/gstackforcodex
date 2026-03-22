# gstack-core-for-codex

This directory is our Codex compatibility workspace for Garry Tan's `gstack`.

## Layout

- `vendor/gstack` lives outside this directory at `/Users/clawtester/Documents/teamagents/vendor/gstack`
- `upstream-codex-baseline/` contains untouched copies of the upstream Codex-generated skills
- `worktree/` contains the copies we will adapt for Codex-native use

## Upstream Baseline

- Repo: `https://github.com/garrytan/gstack`
- Commit: `1f4b6fd`

## Included Skills

- `gstack-office-hours`
- `gstack-plan-ceo-review`
- `gstack-plan-eng-review`
- `gstack-plan-design-review`
- `gstack-design-consultation`
- `gstack-review`
- `gstack-investigate`
- `gstack-design-review`
- `gstack-qa`
- `gstack-qa-only`
- `gstack-ship`
- `gstack-document-release`
- `gstack-retro`
- `gstack-browse`
- `gstack-setup-browser-cookies`

## Working Rule

We do not edit `upstream-codex-baseline/`.
All Codex compatibility changes happen only in `worktree/`.
