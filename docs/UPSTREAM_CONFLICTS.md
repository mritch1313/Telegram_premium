# Upstream conflict and compatibility report

## Snapshot report (2026-10-08)

- Downstream branch: `arena/8fb9cb13-telegram-premium`.
- Downstream `HEAD`: `f2908b14133bbffbf7ab04f641ecb5bfaf533242`.
- Official Android `upstream/master`: `f2908b14133bbffbf7ab04f641ecb5bfaf533242`.
- Tracked source diff between the two SHAs: none.
- Custom commits/features to preserve at this snapshot: none; the checkout is upstream-only.
- Conflict result: **no divergence to merge at this snapshot**. This is not evidence that any requested custom feature works: none is implemented.
- History caveat: the checkout is shallow (`grafted`). A future conflict analysis must fetch full history before computing merge bases or presenting a complete downstream/upstream change range.
- Validation result: Android build/test not run locally. Gradle cannot start because no Java runtime is installed; Android SDK/NDK/CMake are also absent. A GitHub Actions baseline build scaffold exists at `.github/workflows/build.yml`, but no Actions run has verified it yet.

## Required report format for later updates

Every upstream update should attach a report with:

```text
UPSTREAM_BASE_SHA:
UPSTREAM_TARGET_SHA:
DOWNSTREAM_BASE_SHA:
MERGE_COMMIT:
CONFLICTS: none | list
BUILD: pass | fail | blocked (include exact task and first root cause)
TESTS: pass | fail | blocked (include commands and coverage)
FEATURES_TOUCHED:
  - feature:
    upstream files:
    downstream files/custom owner:
    logical/API changes:
    compatibility evidence:
    remaining risk:
HIGH_RISK_REVIEW:
  authorization/login:
  account/session persistence:
  tgnet/JNI/CMake:
  push/background:
  local DB/migrations:
  generated protocol/schema:
  tdata/session adapter:
ARTIFACTS:
KNOWN_LIMITATIONS:
```

## Semantic conflict checklist

A textually clean merge still needs review of:

- account capacity constants and every static array/index boundary across Java and C++/JNI;
- authorization start/resume routes and all `LoginActivity` transitions (phone, code type, 2FA, recovery, registration and already-authorized/add-account paths);
- `ConnectionsManager` native init/config/auth persistence and `tgnet.dat` schema/meaning;
- push token registration, per-account routing, app pause/resume, notification service and OS restrictions;
- `MessagesStorage` schema version and each migration branch;
- read acknowledgements, typing/send-action RPCs and story view updates;
- server-side protected-content errors versus client UI gating;
- TDesktop internal storage format revisions, cryptographic derivation and account authorization validation;
- submodule SHAs, NDK/CMake/ABI and generated TL schema.

A change to a candidate file listed in [CUSTOM_FEATURES.md](CUSTOM_FEATURES.md) must identify the affected feature even if `git merge` reports no conflict. The adapter/authorization paths, if implemented later, require real Telegram server authorization confirmation and two-entry-point regression tests, not only serializer or UI tests.