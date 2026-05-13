# QB Security

A reusable workflow that runs supply-chain security checks on every pull request.

## What it checks

### Invisible Unicode detection

Scans every source file in the PR for invisible Unicode characters used in two known supply-chain attacks:

- **GlassWorm** — embeds Unicode Variation Selectors (U+FE00–U+FE0F, U+E0100–U+E01EF) inside commits. The characters are invisible in code editors, terminals, and GitHub's diff view, allowing payloads to hide inside what appear to be legitimate changes.
- **Trojan Source** — uses bidirectional control characters (U+202A–U+202E, U+2066–U+2069) to visually reorder code during review so that what a reviewer sees differs from what the compiler executes.

Binary files are skipped automatically. Findings are emitted as inline PR annotations pointing to the exact file and line.

### JS supply-chain hardening (npm / yarn / pnpm)

Walks the repo, identifies every JS project by its lockfile (`pnpm-lock.yaml` / `yarn.lock` / `package-lock.json`), and verifies each project enforces the three protections described in [Gajus' write-up](https://gajus.com/blog/3-pnpm-settings-to-protect-yourself-from-supply-chain-attacks):

1. **`minimumReleaseAge`** — quarantines newly published package versions (default: 7 days). Most malicious package versions are caught and yanked within 24–72 hours; quarantining new versions neutralises the window in which a compromised release reaches your CI. Natively enforced for pnpm; for yarn/npm this surfaces as an informational warning since there is no native equivalent.
2. **`blockExoticSubdeps`** — rejects subdependencies pulled from anything other than the npm registry (git URLs, tarball URLs, github: shortcuts, file paths). For pnpm this is a setting; for all managers the action also scans the lockfile and reports each exotic resolution.
3. **`allowBuilds`** (install-script allowlist) — install scripts run by default and have been the delivery mechanism for ua-parser-js, event-stream, nx, and many other attacks. The check requires `pnpm.onlyBuiltDependencies` (pnpm) / `ignore-scripts=true` (npm, yarn classic) / `enableScripts: false` (yarn berry). Specific packages can be whitelisted via the `js-allow-builds` input.

Every finding emits both a PR annotation and a detailed log block with **what** was found, **why** it matters, and a manager-specific **how to fix** snippet. At the bottom of the log an **ACTION REQUIRED** section aggregates every concrete edit, grouped by target file, so a developer can scroll to the bottom and see the exact list of changes needed to clear the check. If no JS lockfiles are present, the job logs that and exits cleanly.

#### Package manager version requirements

The recommended pnpm settings (`minimumReleaseAge`, `blockExoticSubdeps`) only take effect on **pnpm ≥ 10.0**. Older pnpm versions silently ignore these keys — the .npmrc / pnpm-workspace.yaml file looks correct but the protection isn't active at install time. The action reads each project's `packageManager` field in `package.json` (also `.tool-versions`, `engines.pnpm`) and emits a top-level error when a project pins a pnpm version below the threshold, with instructions to bump it. The minimum version is configurable via `js-pnpm-min-version`.

There is no version gate for npm or yarn — `ignore-scripts` / `enableScripts: false` have been supported for the lifetime of those tools.

#### Enforcing minimumReleaseAge for npm / yarn (no native setting)

npm and yarn have no native `minimumReleaseAge` equivalent. By default the action reports an informational warning for npm/yarn projects, explaining that the policy cannot be enforced at install time.

To **actually** enforce it on npm/yarn (and as a backstop for pnpm), enable the opt-in registry-scan mode:

```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      js-enforce-release-age-via-registry: true
```

When enabled, the action parses every lockfile, looks up each (package, version) pair against `registry.npmjs.org`, and fails the PR for any package version published less than the threshold ago. This is opt-in because it makes network calls (one per resolved dependency) and adds noticeable CI time on large lockfiles.

A minimal pnpm project complies with the defaults by adding:

```ini
# .npmrc
minimum-release-age=10080
block-exotic-subdeps=true
```

```jsonc
// package.json
{
  "pnpm": {
    "onlyBuiltDependencies": []
  }
}
```

### Secret scanning (TruffleHog)

Scans the commits introduced by the PR for verified secrets using [TruffleHog OSS](https://github.com/trufflesecurity/trufflehog). Only **verified** secrets (credentials that TruffleHog can confirm are active against the real service) are reported, which eliminates false positives from example keys or already-rotated credentials. Findings are emitted as inline PR annotations.

## Usage

```yaml
name: Security

on:
  pull_request:

jobs:
  # Invisible Unicode + JS supply-chain — no secrets needed
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main

  # Secret scanning — separate job, passes only the token it needs
  trufflehog-scan:
    runs-on: default-k8s-runner
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: QuickBirdEng/actions/trufflehog-scan@main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The unicode and JS supply-chain scans require no secrets. TruffleHog runs as a separate job and receives only `GITHUB_TOKEN` — no `secrets: inherit` needed.

## Inputs

All inputs are optional. Defaults are intentionally broad so most repos need no configuration.

| Input | Type | Default | Description |
|---|---|---|---|
| `runs-on` | string | `default-k8s-runner` | Runner label for the scan jobs |
| `search-directory` | string | `.` | Root directory for both scans |
| `exclude` | string | `''` | Newline- or comma-separated glob patterns to skip (in addition to the always-excluded `.git/**`, `node_modules/**`, `.idea/**`, `build/**`, `dist/**`, common binary types) |
| `fail-on-found` | boolean | `true` | Fail the check when any scan reports findings |
| `js-minimum-release-age-minutes` | number | `10080` | Required quarantine for new package versions, in minutes (7 days). Set to `0` to disable. |
| `js-allow-builds` | string | `''` | Newline- or comma-separated list of packages allowed to run install scripts. Empty = none allowed. |
| `js-require-block-exotic-subdeps` | boolean | `true` | Require pnpm `blockExoticSubdeps` AND scan every lockfile for non-registry resolutions |
| `js-enforce-release-age-via-registry` | boolean | `false` | Opt-in: enforce `minimumReleaseAge` for npm/yarn/pnpm by looking up each lockfile entry's publish date against the npm registry. Adds network calls per dependency. |
| `js-pnpm-min-version` | string | `10.0.0` | Minimum pnpm version where the recommended settings actually take effect. Projects pinning an older pnpm get a top-level error. |

## Typical overrides

**Exclude an additional generated directory:**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      exclude: |
        generated/**
        vendored/**
```

**Run on a self-hosted runner:**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      runs-on: 'self-hosted'
```

**Audit mode (report but do not fail):**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      fail-on-found: false
```

**Whitelist install-script packages and tighten the quarantine:**
```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      js-minimum-release-age-minutes: 20160  # 14 days
      js-allow-builds: |
        esbuild
        sharp
```
