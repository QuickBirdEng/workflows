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

Two managers now have native install-time enforcement of the full policy:

| Manager | Minimum version | Settings honored |
|---|---|---|
| pnpm | **10.0+** | `minimumReleaseAge`, `blockExoticSubdeps`, `onlyBuiltDependencies` |
| yarn | **4.14+** | `npmMinimalAgeGate` (4.10+), `approvedGitRepositories` (4.14+), `enableScripts: false` |

Older versions parse the settings but silently ignore them — the config file looks correct, but the install-time behaviour is unchanged. The action reads each project's `packageManager` field in `package.json` and emits a top-level error when a project pins a version below the threshold. The minimum versions are configurable via `js-pnpm-min-version` and `js-yarn-min-version`.

yarn 1.x (classic) and npm < 11.10 have no native equivalent. Projects on those managers hard-fail when `js-minimum-release-age-minutes > 0` with a migration prompt offering yarn 4.14+ OR pnpm 10+ as the two viable paths.

#### Enforcing minimumReleaseAge — two native paths

The two managers that enforce minimum-release-age at install time:

**Yarn 4.14+** (least disruptive for yarn projects):

```bash
corepack enable
yarn set version 4.14.0
```

then in `.yarnrc.yml`:

```yaml
npmMinimalAgeGate: 10080         # 7 days, in minutes
approvedGitRepositories: []      # empty = block all git/tarball subdeps
enableScripts: false             # disable install scripts
```

**Pnpm 10+** (full pnpm parity):

```bash
npx @pnpm/exe@latest import      # imports yarn.lock / package-lock.json → pnpm-lock.yaml
```

then in `package.json`:

```jsonc
{
  "packageManager": "pnpm@10.x.y",
  "pnpm": { "onlyBuiltDependencies": [] }
}
```

and `.npmrc`:

```ini
minimum-release-age=10080
block-exotic-subdeps=true
```

Update the CI install step to `pnpm install --frozen-lockfile` and delete the old lockfile.

Projects on yarn 1.x, yarn-berry below 4.14, or npm (which currently has no widely-supported native setting) **hard-fail** when `js-minimum-release-age-minutes > 0` — the action emits an error pointing at the two migration paths above.

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
| `js-yarn-min-version` | string | `4.14.0` | Minimum yarn version with native install-time enforcement (`npmMinimalAgeGate` + `approvedGitRepositories`). Yarn 1.x and yarn-berry below this hard-fail with a migration prompt. |

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
