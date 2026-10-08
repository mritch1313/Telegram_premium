# Upstream relationship and update procedure

## Current relationship

- Official Android source: `https://github.com/DrKLO/Telegram.git` (`master`).
- Downstream repository: `https://github.com/mritch1313/Telegram_premium.git` (`origin`).
- The local Git remote `upstream` has been added and `master` fetched.
- At the initial 2026-10-08 source audit, the source base and `upstream/master` were both `f2908b14133bbffbf7ab04f641ecb5bfaf533242` (`update to 12.10.6 (7112)`).
- The current session branch has downstream-only audit documentation and a build-workflow scaffold above that source base. The Telegram Android application source has not been modified; there are no custom app-feature commits yet.
- The checkout is shallow and all 15 submodules are uninitialized. An actual update job must use a full-history checkout (`fetch-depth: 0`) and initialize recursive submodules at the gitlink revisions before building.

## Required update policy

1. Fetch official Android `master` (and record the fetched SHA and version metadata). Do not infer a source update from a version label alone.
2. Start a unique update branch from the selected downstream base. Do not rewrite an existing branch or force-push over user history.
3. Merge the official upstream commit as a merge commit (or use a documented non-rewriting strategy). Preserve every downstream custom commit. Do not squash the complete downstream history into one commit.
4. Record the upstream file diff and compare it with the feature ownership map in [CUSTOM_FEATURES.md](CUSTOM_FEATURES.md). Include direct conflicts and logical/API changes even where Git reports no conflict.
5. Treat these as high-risk areas: `LoginActivity`/`LaunchActivity`, `UserConfig` and account-indexed arrays, JNI/native tgnet and CMake, `ConnectionsManager`/`tgnet.dat`, push/background lifecycle, `MessagesStorage` migrations, generated TLRPC schema, TDesktop adapters if added, and the media submodule.
6. On a conflict, do not drop custom changes or push a conflict-marked build. Save `git status`, unmerged paths, conflict hunks, upstream commit, base commit and affected feature rows in an artifact/report; open or update a review item for human resolution.
7. On a clean merge, run the clean baseline APK build, unit/instrumentation/static checks that the environment supports, and each affected feature's behavioral tests. For session/import work, rerun real authorization/import tests; parser success or a clean merge is not proof of a usable Telegram session.
8. Review changed files and the diff for semantic regressions, then build again after any fix. Create a PR only when checks pass; do not automatically merge it.
9. Attach the commit/version diff, feature-impact report, tests/build results, APK artifact and known limitations to the PR. Never include secrets, account data or logs containing credentials.

## Workflow status

There is no `.github/workflows/upstream-sync.yml` or `release.yml` in the checkout yet. `.github/workflows/build.yml` is a hosted baseline-build/instrumentation workflow. First run `37826201297` failed during Android SDK setup. Rerun `37826983540` passed toolchain setup and release/test APK assembly, uploading a 62,835,297-byte baseline APK artifact, but its instrumentation step was canceled at the 120-minute job limit; no passing test result is established. The workflow timeout is now 240 minutes for a retry. The APK artifact is available on the [Actions run](https://github.com/mritch1313/Telegram_premium/actions/runs/37826983540), but this sandbox could not download it from GitHub’s external artifact-storage host. The upstream workflow should open a PR only after clean build and test checks pass. This sandbox has no JDK/Android SDK/NDK, so the workflow cannot be validated locally.

## Suggested reproducible local commands

```bash
git remote add upstream https://github.com/DrKLO/Telegram.git # only if absent
git fetch --prune upstream master
git rev-parse HEAD upstream/master
git diff --stat <downstream-base> upstream/master
git submodule update --init --recursive
```

Use a new update branch and record its base/upstream SHA before merging. For an actual PR, test it in a full Android environment; a successful `git merge` alone is insufficient.