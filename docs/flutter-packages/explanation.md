# Flutter Packages
---

### Branch

The branch workflow runs flutter lint and flutter test. The intention is to have it run on every push to the repo, to uphold quality standards before and during merges.

### Release

The release workflow is the same as the branch workflow, but also notifies a slack channel of success or failure. The ID of the channel to use is given as a parameter.
