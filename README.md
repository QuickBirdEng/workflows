# workflows
A set of workflows used throughout the organization

## Usage Examples
---

The examples in [The docs folder](docs/) are provided as examples of calling workflows only. Please verify any values/inputs before use in other projects.

From the root of your project repo, place the calling workflow in `.github/workflows`.

## Workflows

| Workflow | Description | Docs |
|---|---|---|
| `qb-security` | Invisible Unicode detection + verified secret scanning on pull requests | [docs/qb-security](docs/qb-security/explanation.md) |
| `sanity-requirements` | LoC delta check and branch-ticket check for feature branches | [docs/sanity-requirements](docs/sanity-requirements/explanation.md) |
| `flutter-package-branch` / `flutter-package-release` | Flutter lint, test, and Slack notification for package repos | [docs/flutter-packages](docs/flutter-packages/explanation.md) |

