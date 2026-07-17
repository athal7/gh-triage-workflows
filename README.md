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
- **`jules-assign.yml`** — a `workflow_call` reusable workflow that auto-applies a
  `jules` label to newly opened/edited/reopened issues and runs a daily self-healing
  sweep over all open issues. Skips issues that are already labeled `jules` or
  `no-jules`, are blocked on an open issue (body contains `blocked on #N` / `blocked
  by #N`), or have a linked open/merged PR (`closedByPullRequestsReferences`).
  Permissions: `issues: write`, `pull-requests: read`.
- **`issue-escalation.yml`** — a `workflow_call` reusable workflow that escalates
  issues to `athal7` on two signals:
  - **`jules-failed`**: fires when `google-labs-jules[bot]` posts an issue comment
    containing `"failed to create a task"` — immediately assigns `athal7`.
  - **`stalled-discussion`**: fires on every `issue_comment` created event; if the
    issue has no assignees and exactly 4 non-bot, non-athal7 human comments exist,
    assigns `athal7`. The `=== 4` exact-match fires once when the threshold is first
    crossed, not on every subsequent comment.
- **`review-bridge.yml`** — a `workflow_call` reusable workflow that requests `athal7`
  as a PR reviewer on two CodeRabbit signals, and flags external contributor PRs for
  CI approval:
  - **`coderabbit-approved`**: fires when CodeRabbit submits an `approved` review —
    clean pass, ready for human merge decision.
  - **`coderabbit-stalled`**: fires when CodeRabbit submits a `changes_requested`
    review; counts total CodeRabbit `CHANGES_REQUESTED` reviews on the PR and
    requests `athal7` only when that count reaches 3 or more (PR is stuck in a
    revision loop). The unconditional Jules-opened-PR trigger has been removed.
  - **`external-contributor-flag`**: fires on `pull_request_target` opened events.
    If the PR author's `author_association` is not one of `OWNER`, `COLLABORATOR`,
    or `MEMBER` (i.e. `FIRST_TIME_CONTRIBUTOR`, `FIRST_TIMER`, `CONTRIBUTOR`, or
    `NONE`) and the author is not a bot, adds `athal7` as assignee and posts a
    one-time comment explaining that GitHub requires manual CI approval for
    first-time contributors (idempotent — skips the comment if already posted).
    This surfaces externally-contributed PRs that would otherwise go silent because
    CI never runs and no reviewer is notified.
- **`dependabot-automerge.yml`** — a `workflow_call` reusable workflow that auto-merges
  Dependabot PRs for minor/patch bumps once required CI checks pass. When auto-merge
  cannot be enabled (repo has no required status checks) or the bump is a major version
  upgrade, the workflow requests `athal7` as a reviewer instead of silently leaving
  the PR open. **Note:** GitHub's auto-merge feature requires at least one required
  status check configured in branch protection — repos without any CI workflow cannot
  use auto-merge.

## Using this from another repo

Add `.github/workflows/triage.yml` to the consuming repo:

```yaml
name: Triage

on:
  issues:
    types: [opened, edited, reopened]
  issue_comment:
    types: [created]
  pull_request_target:
    types: [opened, edited, reopened]
  pull_request_review:
    types: [submitted]
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

  review-bridge:
    if: github.event_name == 'pull_request_review' || github.event_name == 'pull_request_target'
    uses: athal7/gh-triage-workflows/.github/workflows/review-bridge.yml@main
    secrets: inherit

  dependabot-automerge:
    if: github.event_name == 'pull_request_target' && github.event.pull_request.user.login == 'dependabot[bot]'
    uses: athal7/gh-triage-workflows/.github/workflows/dependabot-automerge.yml@main
    secrets: inherit

  jules-assign:
    if: github.event_name == 'issues' || github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    uses: athal7/gh-triage-workflows/.github/workflows/jules-assign.yml@main
    secrets: inherit

  issue-escalation:
    if: github.event_name == 'issue_comment'
    uses: athal7/gh-triage-workflows/.github/workflows/issue-escalation.yml@main
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
| `jules` | jules-assign.yml | Auto-applied: let Google Jules attempt a fix |
| `no-jules` | manual | Opt-out: skip this issue in jules-assign sweep |

## CodeRabbit prerequisite for review-bridge

For the CodeRabbit halves of `review-bridge.yml` to fire (requesting `athal7` when
CodeRabbit approves or repeatedly flags issues), each consuming repo must have a
`.coderabbit.yaml` at its root with at minimum:

```yaml
reviews:
  request_changes_workflow: true
```

Without this, CodeRabbit will post comments but won't emit formal review events, so
the `pull_request_review` trigger won't fire.
