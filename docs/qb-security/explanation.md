# QB Security

A reusable workflow that runs supply-chain security checks on every pull request.

## What it checks

### Invisible Unicode detection

Scans every source file in the PR for invisible Unicode characters used in two known supply-chain attacks:

- **GlassWorm** — embeds Unicode Variation Selectors (U+FE00–U+FE0F, U+E0100–U+E01EF) inside commits. The characters are invisible in code editors, terminals, and GitHub's diff view, allowing payloads to hide inside what appear to be legitimate changes.
- **Trojan Source** — uses bidirectional control characters (U+202A–U+202E, U+2066–U+2069) to visually reorder code during review so that what a reviewer sees differs from what the compiler executes.

Binary files are skipped automatically. Findings are emitted as inline PR annotations pointing to the exact file and line.

### Secret scanning (TruffleHog)

Scans the commits introduced by the PR for verified secrets using [TruffleHog OSS](https://github.com/trufflesecurity/trufflehog). Only **verified** secrets (credentials that TruffleHog can confirm are active against the real service) are reported, which eliminates false positives from example keys or already-rotated credentials. Findings are emitted as inline PR annotations.

## Usage

```yaml
name: Security

on:
  pull_request:

jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    secrets: inherit
```

That is the minimal setup. `secrets: inherit` is required for the TruffleHog job to access `GITHUB_TOKEN`.

## Inputs

All inputs are optional. Defaults are intentionally broad so most repos need no configuration.

| Input | Type | Default | Description |
|---|---|---|---|
| `runs-on` | string | `default-k8s-runner` | Runner label |
| `search-directory` | string | `.` | Root directory for the invisible Unicode scan |
| `exclude-dirs` | string | `.git,node_modules,.idea,build,dist` | Directory names to skip |
| `exclude-patterns` | string | `*.png,*.jpg,*.jpeg,*.gif,*.ico,*.pdf,*.zip,*.tar,*.gz,*.bin,*.dill` | File glob patterns to skip |
| `fail-on-found` | boolean | `true` | Fail the check when invisible Unicode is detected |

## Typical overrides

**Exclude an additional generated directory:**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      exclude-dirs: '.git,node_modules,.idea,build,dist,generated'
    secrets: inherit
```

**Run on a self-hosted runner:**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      runs-on: 'self-hosted'
    secrets: inherit
```

**Audit mode (report but do not fail):**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      fail-on-found: false
    secrets: inherit
```
