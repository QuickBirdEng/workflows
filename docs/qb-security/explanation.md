# QB Security

A reusable workflow that runs three supply-chain security scans on every pull request.

## Table of Contents

- [Usage](#usage)
- [Secret Scanning (TruffleHog)](#secret-scanning-trufflehog)
- [Invisible Unicode Detection](#invisible-unicode-detection)
- [JS Supply-Chain Hardening](#js-supply-chain-hardening-npm--yarn--pnpm)
- [Inputs](#inputs)

---

## Usage

```yaml
name: Security

on:
  pull_request:
  push:
    branches: [main]

jobs:
  security:
    uses: QuickBirdEng/workflows/.github/workflows/qb-security.yml@main
```

No secrets are required. All three scans run as parallel jobs inside the workflow.

**Self-hosted runner:**
```yaml
    with:
      runs-on: 'self-hosted'
```

**Audit mode — report but do not fail:**
```yaml
    with:
      fail-on-found: false
```

---

## Secret Scanning (TruffleHog)

Scans git commits for verified secrets using [TruffleHog OSS](https://github.com/trufflesecurity/trufflehog). Only **verified** secrets (credentials that TruffleHog can confirm are active against the real service) are reported, eliminating false positives from example keys or already-rotated credentials. Findings are emitted as inline PR annotations.

By default the **full repository history** is scanned on every run. Pass `trufflehog-base` to limit the scan to commits reachable from HEAD but not from the base — equivalent to `git log <base>..HEAD`:

```yaml
    with:
      trufflehog-base: ${{ github.event.pull_request.base.sha || github.event.before }}
```

`github.event.pull_request.base.sha` is the tip of the target branch on a `pull_request` event; `github.event.before` is the SHA before the push on a `push` event.

**Exclude or restrict paths:**
```yaml
    with:
      trufflehog-exclude-paths: .trufflehog-exclude   # file listing paths to skip
      trufflehog-include-paths: .trufflehog-include   # file listing paths to scan
```

**Skip the scan entirely:**
```yaml
    with:
      trufflehog-enable: false
```

---

## Invisible Unicode Detection

Scans every source file for invisible Unicode characters used in two known supply-chain attacks:

- **GlassWorm** — embeds Unicode Variation Selectors (U+FE00–U+FE0E, U+E0100–U+E01EF) inside commits. The characters are invisible in code editors, terminals, and GitHub's diff view, allowing payloads to hide inside what appear to be legitimate changes.
- **Trojan Source** — uses bidirectional control characters (U+202A–U+202E, U+2066–U+2069) to visually reorder code during review so that what a reviewer sees differs from what the compiler executes.

Binary files are skipped automatically. Findings are emitted as inline PR annotations pointing to the exact file and line.

**Exclude paths:**
```yaml
    with:
      unicode-exclude: |
        generated/**
        vendored/**
```

**Skip the scan entirely:**
```yaml
    with:
      unicode-enable: false
```

---

## JS Supply-Chain Hardening (npm / yarn / pnpm)

Walks the repo, identifies every JS project by its lockfile (`pnpm-lock.yaml` / `yarn.lock` / `package-lock.json`), and verifies each project enforces the three protections described in [Gajus' write-up](https://gajus.com/blog/3-pnpm-settings-to-protect-yourself-from-supply-chain-attacks):

1. **`minimumReleaseAge`** — quarantines newly published package versions (default: 3 days). Most malicious package versions are caught and yanked within 24–72 hours; quarantining new versions neutralises the window in which a compromised release reaches your CI. Natively enforced for pnpm 10+ and yarn 4.14+; yarn 1.x, older yarn-berry, and npm hard-fail with a migration prompt since these managers have no equivalent install-time setting.
2. **`blockExoticSubdeps`** — rejects subdependencies pulled from anything other than the npm registry (git URLs, tarball URLs, github: shortcuts, file paths). For pnpm this is a setting; for all managers the action also scans the lockfile and reports each exotic resolution.
3. **`allowBuilds`** (install-script allowlist) — install scripts run by default and have been the delivery mechanism for ua-parser-js, event-stream, nx, and many other attacks. The check requires `pnpm.onlyBuiltDependencies` (pnpm) / `ignore-scripts=true` (npm, yarn classic) / `enableScripts: false` (yarn berry). Specific packages can be whitelisted via the `js-allow-builds` input.

Every finding emits both a PR annotation and a detailed log block with **what** was found, **why** it matters, and a manager-specific **how to fix** snippet. At the bottom of the log an **ACTION REQUIRED** section aggregates every concrete edit grouped by target file. If no JS lockfiles are present, the job logs that and exits cleanly.

### Package Manager Version Requirements

Two managers have native install-time enforcement of the full policy:

| Manager | Minimum version | Settings honored |
|---|---|---|
| pnpm | **10.0+** | `minimumReleaseAge`, `blockExoticSubdeps`, `onlyBuiltDependencies` |
| yarn | **4.14+** | `npmMinimalAgeGate` (4.10+), `approvedGitRepositories` (4.14+), `enableScripts: false` |

Older versions parse the settings but silently ignore them. The action reads each project's `packageManager` field in `package.json` and emits a top-level error when a project pins a version below the threshold. The minimums are configurable via `js-pnpm-min-version` and `js-yarn-min-version`.

yarn 1.x (classic) and npm < 11.10 have no native equivalent. Projects on those managers hard-fail when `js-minimum-release-age-minutes > 0` with a migration prompt offering yarn 4.14+ OR pnpm 10+ as the two viable paths.

### Enforcing minimumReleaseAge — Migration Paths

**Yarn 4.14+** (least disruptive for yarn projects):

```bash
corepack enable
yarn set version 4.14.0
```

`.yarnrc.yml`:
```yaml
npmMinimalAgeGate: 4320          # 3 days, in minutes
approvedGitRepositories: []      # empty = block all git/tarball subdeps
enableScripts: false             # disable install scripts
```

**Pnpm 10+**:

```bash
npx @pnpm/exe@latest import      # imports yarn.lock / package-lock.json → pnpm-lock.yaml
```

`package.json`:
```jsonc
{
  "packageManager": "pnpm@10.x.y",
  "pnpm": { "onlyBuiltDependencies": [] }
}
```

`.npmrc`:
```ini
minimum-release-age=4320
block-exotic-subdeps=true
```

Update CI to `pnpm install --frozen-lockfile` and delete the old lockfile.

#### Escape Hatch: CI-Only Registry Scan (not recommended)

For projects that cannot migrate immediately:

```yaml
    with:
      js-enforce-release-age-via-registry: true
```

This scans every lockfile entry against `registry.npmjs.org` at PR time and fails on any version published less than the threshold ago. It gates merged code but **does NOT** protect a developer who runs `yarn install` on a local branch before opening the PR. Use as a stopgap during migration only.

Setting `js-minimum-release-age-minutes: 0` disables the policy entirely.

#### Minimal Compliant pnpm Config

```ini
# .npmrc
minimum-release-age=4320
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

### Overrides

**Exclude paths:**
```yaml
    with:
      js-exclude: |
        generated/**
        vendored/**
```

**Tighten the quarantine and whitelist install-script packages:**
```yaml
    with:
      js-minimum-release-age-minutes: 20160  # 14 days
      js-allow-builds: |
        esbuild
        sharp
```

**Disable individual sub-checks:**
```yaml
    with:
      js-minimum-release-age-minutes: 0       # turn off the age gate
      js-require-block-exotic-subdeps: false  # turn off the exotic-subdeps check
      js-check-install-scripts: false         # turn off the install-scripts check
```

**Approve an urgent security patch newer than the quarantine threshold:**

The exemption lives in the project's native package-manager config — yarn / pnpm both honor it at install time, and the qb-security check trusts the project's decision. The exemption shows up in the PR diff and is reviewed alongside the lockfile change.

For yarn 4.10+:
```yaml
# .yarnrc.yml (project)
npmPreapprovedPackages:
  - lodash    # CVE-2026-XXXX, reviewed by @<reviewer>
```

For pnpm 10+:
```ini
# .npmrc (project)
minimum-release-age-exclude=lodash
```

No workflow-side change is needed. The optional registry-scan path (`js-enforce-release-age-via-registry: true`) honours the project's exempt list automatically.

**Skip the scan entirely:**
```yaml
    with:
      js-enable: false
```

---

## Inputs

All inputs are optional.

| Input | Type | Default | Description |
|---|---|---|---|
| `runs-on` | string | `default-k8s-runner` | Runner label for all scan jobs |
| `fail-on-found` | boolean | `true` | Fail the check when any scan reports findings |
| `trufflehog-enable` | boolean | `true` | Set to `false` to skip the TruffleHog scan entirely |
| `trufflehog-base` | string | `''` | Base commit SHA or ref. Leave empty to scan the full git history |
| `trufflehog-exclude-paths` | string | `''` | Path to a file listing paths to exclude from the TruffleHog scan |
| `trufflehog-include-paths` | string | `''` | Path to a file listing paths to include in the TruffleHog scan |
| `unicode-enable` | boolean | `true` | Set to `false` to skip the invisible-Unicode scan entirely |
| `unicode-exclude` | string | `''` | Newline- or comma-separated glob patterns to exclude from the Unicode scan (always excluded: `.git/**`, `node_modules/**`, `.idea/**`, `build/**`, `dist/**`, common binary types) |
| `js-enable` | boolean | `true` | Set to `false` to skip the JS supply-chain scan entirely |
| `js-exclude` | string | `''` | Newline- or comma-separated glob patterns to exclude from the JS supply-chain scan (always excluded: `.git/**`, `node_modules/**`, `.idea/**`, `build/**`, `dist/**`) |
| `js-minimum-release-age-minutes` | number | `4320` | Required quarantine for new package versions in minutes (3 days). Set to `0` to disable |
| `js-allow-builds` | string | `''` | Newline- or comma-separated list of packages allowed to run install scripts. Empty = none allowed |
| `js-check-install-scripts` | boolean | `true` | Set to `false` to skip the install-scripts sub-check |
| `js-require-block-exotic-subdeps` | boolean | `true` | Require `blockExoticSubdeps` and scan every lockfile for non-registry resolutions. Set to `false` to skip |
| `js-enforce-release-age-via-registry` | boolean | `false` | CI-time band-aid: scan lockfile entries against the npm registry to enforce minimum release age. Does not protect developer machines |
| `js-pnpm-min-version` | string | `10.0.0` | Minimum pnpm version where supply-chain settings take effect |
| `js-yarn-min-version` | string | `4.14.0` | Minimum yarn version with native install-time enforcement (`npmMinimalAgeGate` + `approvedGitRepositories`) |
| `search-directory` | string | `.` | Root directory for the Unicode and JS supply-chain scans |
