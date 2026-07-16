# gh-triage-workflows

Reusable GitHub Actions workflows for lightweight issue/PR triage, shared across
[athal7](https://github.com/athal7)'s public repositories. Centralizing here means
tuning a threshold or regex once propagates to every repo that calls in, instead of
maintaining near-identical YAML in 13+ places.

## What's here

- **`stale.yml`** — a `workflow_call` reusable workflow wrapping [`actions/stale`](https://github.com/actions/stale).
  Issues get a `stale` label after 90 days of inactivity but are **never auto-closed**
  (an old feature request isn't neglect). PRs get `stale` after 45 days and are
  auto-closed after a further 14 days of inactivity (abandoned PRs against a moving
  codebase are real debt). `dependencies`/`pinned`/`security` labels are exempt.
- **`label.yml`** — a `workflow_call` reusable workflow wrapping [`github/issue-labeler`](https://github.com/github/issue-labeler),
  pointed at this repo's canonical `labeler.yml` regex config. Applies `bug`,
  `enhancement`, or `question` labels to new/edited issues and PRs by keyword match
  in title + body. This is a first-pass sorter, not a classifier — expect occasional
  misfires on repos with unusual vocabulary; corrections stick (`sync-labels: 0`).
- **`labeler.yml`** — the canonical regex-to-label map consumed by `label.yml`.

## Using this from another repo

Add `.github/workflows/triage.yml` to the consuming repo:

```yaml
name: Triage

on:
  issues:
    types: [opened, edited, reopened]
  pull_request_target:
    types: [opened, edited, reopened]
  schedule:
    - cron: '30 1 * * *'
  workflow_dispatch:

jobs:
  label:
    if: github.event_name == 'issues' || github.event_name == 'pull_request_target'
    uses: athal7/gh-triage-workflows/.github/workflows/label.yml@main
    secrets: inherit

  stale:
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    uses: athal7/gh-triage-workflows/.github/workflows/stale.yml@main
    secrets: inherit
```

Callers pin to `@main` intentionally — this is a solo hobby fleet, auto-propagation
of tuning changes is preferred over a stability gate. Override any `stale.yml` input
via the caller's `with:` block if a specific repo needs different thresholds.

## Label taxonomy

| Label | Owner | Meaning |
|---|---|---|
| `bug` | label.yml (regex) | Title/body suggests broken/erroring behavior |
| `enhancement` | label.yml (regex) | Title/body suggests a feature request |
| `question` | label.yml (regex) | Title/body suggests a question |
| `dependencies` | Dependabot/Renovate (self-labeled) | Automated dependency-bump PR |
| `stale` | stale.yml | No activity past the configured threshold |
