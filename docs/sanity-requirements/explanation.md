# Sanity Requirements
---

This reusable workflow wraps two lightweight checks that are commonly used on feature branches:

- a LoC delta check against the closest configured base branch
- a branch-ticket check that fails when the ticket from the branch name still appears in the repository

The base branch resolution is implemented inside the `check-loc` action itself, so the action can also be used standalone after a normal checkout step.

The defaults are intentionally conservative so most repos only need to set the ticket prefix, and optionally the runner label.

## Trigger: pull_request vs push

Prefer triggering the caller on `pull_request` instead of `push`. A `push` event carries no base ref, so the LoC check falls back to guessing the closest of `base-branches` by merge-base. That guess is wrong for a stacked PR, a branch made off another feature branch instead of `main`, because the real base is not in `base-branches` at all.

On a `pull_request` event, this workflow reads `github.event.pull_request.base.ref` automatically and uses it as the exact base. Stacked PRs then get the correct diff without extra input. `push`-triggered callers keep working unchanged through the `base-branches` fallback.

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:
```

## Common Inputs

- `runs-on`: defaults to `default-k8s-runner`
- `base-branches`: defaults to `main`; fallback used only when `base-ref` is empty (push-triggered callers, or `pull_request` callers that want to override the detected base)
- `base-ref`: explicit base ref; normally left empty so it is picked up from `github.event.pull_request.base.ref`
- `loc-limit`: defaults to `800`
- `ticket-prefixes`: optional, but recommended to avoid overly broad matching
- `loc-ignore-patterns`: optional extra git pathspec excludes for repo-specific generated files

## Typical Overrides

- Swiss Cannabis style setup:
  `base-branches: "main,zuerich"`
  `ticket-prefixes: "SCC"`
- Mindnet style setup:
  `runs-on: "self-hosted"`
  `ticket-prefixes: "MNET"`
- FDS Copilot style setup:
  `loc-limit: 1200`
  `ticket-prefixes: "FDS,SEL"`
  add repo-specific values to `loc-ignore-patterns`
