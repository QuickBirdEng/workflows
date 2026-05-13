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

#### Enforcing minimumReleaseAge — yarn/npm have no native setting

Only pnpm 10+ enforces `minimumReleaseAge` at install time. yarn (1.x and Berry) and npm have **no equivalent**: there is no `.yarnrc` / `.npmrc` flag that makes those package managers refuse a too-recent install. Anyone cloning the repo and running `yarn install` / `npm install` can pull a freshly-published — and possibly compromised — version of any dependency.

So the action **hard-fails** yarn/npm projects that set `js-minimum-release-age-minutes > 0`, with a message pointing at the only real fix: migrate the project to pnpm 10+.

Migration is usually one command:

```bash
npx @pnpm/exe@latest import   # imports yarn.lock / package-lock.json → pnpm-lock.yaml
```

then update `package.json`:

```jsonc
{
  "packageManager": "pnpm@10.x.y",
  "pnpm": {
    "onlyBuiltDependencies": []
  }
}
```

and `.npmrc`:

```ini
minimum-release-age=10080
block-exotic-subdeps=true
```

Update the CI install step to `pnpm install --frozen-lockfile` and delete the old lockfile.

##### Escape hatch: CI-only registry scan (not recommended)

For projects that cannot migrate immediately, an opt-in CI band-aid is available:

```yaml
jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
    with:
      js-enforce-release-age-via-registry: true
```

When enabled, the action scans every lockfile entry against `registry.npmjs.org` at PR time and fails on any version published less than the threshold ago. This gates the merged code so anyone cloning `main` is safe — but it **does NOT** protect a developer who runs `yarn install` on a local branch before opening the PR. For that, only pnpm 10+ works. Use this as a stopgap during a migration, not as the permanent answer.

Setting `js-minimum-release-age-minutes: 0` disables the policy entirely.

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
| `js-enforce-release-age-via-registry` | boolean | `false` | Opt-in CI-time band-aid for yarn/npm projects that cannot migrate to pnpm 10+ immediately. When false (default), yarn/npm projects with `minimum-release-age > 0` hard-fail with a migration prompt. |
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
