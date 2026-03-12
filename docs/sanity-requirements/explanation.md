# Sanity Requirements
---

This reusable workflow wraps two lightweight checks that are commonly used on feature branches:

- a LoC delta check against the closest configured base branch
- a branch-ticket check that fails when the ticket from the branch name still appears in the repository

The base branch resolution is implemented inside the `check-loc` action itself, so the action can also be used standalone after a normal checkout step.

The defaults are intentionally conservative so most repos only need to set the ticket prefix, and optionally the runner label.

## Common Inputs

- `runs-on`: defaults to `default-k8s-runner`
- `base-branches`: defaults to `main`
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
