# JS Sanity Requirements
---

## Why this check exists

Prisma applies pending migrations in filename-sorted order, and that order is set by the
timestamp prefix in the migration folder name. A new migration folder timestamped earlier
than what is already on the target branch still gets applied, since Prisma only tracks
what ran before, not whether the name sorts after everything else. It just runs before
migrations that were already written, and possibly deployed, assuming it did not exist yet.
Harmless for an isolated migration, a real ordering bug if a later migration ever touches
something the backdated one also touches.

## How it works

The `migration-order-check` job fails a PR that adds a migration folder not timestamped
after the latest migration already on the PR's target branch. It resolves the base and
head from `github.event.pull_request.base.ref` and `github.event.pull_request.head.sha`,
the same stacked-PR-safe resolution used by
[sanity-requirements](../sanity-requirements/explanation.md), so trigger the caller on
`pull_request`.

The actual comparison lives in the `check-migration-order` action, so it can also be used
standalone after a normal checkout and base-branch fetch.

## Common Inputs

- `runs-on`: defaults to `default-k8s-runner`
- `migrations-dir`: defaults to `prisma/migrations`; set this to wherever the repo actually
  keeps its migrations, e.g. `web/prisma/migrations`
- `base-ref`: optional override; normally left empty so it is picked up from
  `github.event.pull_request.base.ref`

## Typical Overrides

- Swiss Cannabis style setup:
  `migrations-dir: "web/prisma/migrations"`
  trigger on `branches: [main, zuerich]` with `paths: ["web/prisma/migrations/**"]`
  caller job id: `PrismaJS`
