# Architecture audit — Telegram Android

> Snapshot audited: 2026-10-08. This is a source-tree audit, not a claim that an APK has been built or run.

## Repository and upstream baseline

- Working branch: `arena/8fb9cb13-telegram-premium`.
- Official Android source base: `f2908b14133bbffbf7ab04f641ecb5bfaf533242` (`update to 12.10.6 (7112)`, 2026-09-30); fetched `upstream/master` resolves to this SHA.
- Downstream-only changes are audit documentation and `.github/workflows/build.yml`; Telegram Android application sources remain identical to the official base.
- `upstream` is configured as `https://github.com/DrKLO/Telegram.git`; this fork is built directly on the official source, not a clean-room Telegram imitation.
- Application version in `gradle.properties`: `12.10.6`, version code `7112`.
- The checkout is shallow (`f2908b1` is shown as grafted), so full-history merge analysis still requires unshallow fetch.

The baseline build was **not established**. `./gradlew --version` stops before Gradle starts with `JAVA_HOME is not set and no 'java' command could be found in your PATH`. This environment also has no Android SDK/NDK, CMake, `sdkmanager`, or emulator. Per the requested sequencing, feature code should not be changed until a clean upstream APK build can be performed in a correctly provisioned Android environment.

## Gradle, variants and native toolchain

- Wrapper: Gradle `8.13` (`gradle/wrapper/gradle-wrapper.properties`).
- Android Gradle Plugin: `8.13.2`; Kotlin Gradle plugin: `2.1.0` (`build.gradle`). Main application code is predominantly Java; Kotlin is used in generated/instrumentation tests and `buildSrc` plugins/tasks.
- SDK/build-tools: compile SDK `36`, build-tools `36.0.0`, target SDK `36`; main library minimum SDK `21` (some flavors use `23`; test app minimum SDK `26`).
- NDK: `27.2.12479018`; CMake: `3.22.1`; native C++ standard is C++17. JNI target is `tmessages.49` (`TMessagesProj/jni/CMakeLists.txt`).
- Modules: `TMessagesProj` is the Android library containing the application UI, controllers, protocol models and native libraries; application shells are `TMessagesProj_App`, `TMessagesProj_AppHuawei`, `TMessagesProj_AppHockeyApp`, `TMessagesProj_AppStandalone`; `TMessagesProj_AppTests` is the instrumentation-test app. `TMessagesProj_Modules/media` supplies Media3 modules; `jlatexmath` is included from a submodule.
- The primary app has `bundleAfat`, `bundleAfat_SDK23`, and `afat` flavors plus debug/release/standalone build types. Huawei, HockeyApp, standalone, and test modules have their own variant definitions. The README asks for Android Studio 2025.1.4, NDK 27.2.12479018, and SDK 36.
- There are 15 Git submodules in `.gitmodules`. In this checkout all 15 show a leading `-` in `git submodule status`, meaning they are not initialized. A full native build therefore also needs `git submodule update --init --recursive`.
- The working branch now contains `.github/workflows/build.yml`, a hosted clean-build/instrumentation workflow scaffold. It has not run yet. Upstream-sync and release workflows do not exist.

## Runtime architecture

### Application entry and navigation

- `org.telegram.ui.LaunchActivity` is the Android entry activity, intent router and top-level navigation owner. It chooses an authenticated app fragment or the onboarding/login path based on the selected account and persisted login state.
- `IntroActivity` is the onboarding/intro surface; its start button opens `LoginActivity`.
- `LoginActivity` is a stateful, multi-page authorization flow. Its `PhoneView` (`VIEW_PHONE_INPUT`) submits `auth.sendCode`; code-entry pages handle the server-selected code delivery type; `auth.signIn` handles code sign-in; `SESSION_PASSWORD_NEEDED` leads to the password/2FA page and `auth.checkPassword`; registration, recovery, email, and reset-wait stages are also present. Successful authorization funnels through `onAuthSuccess`, persists the user/account state, and hands off to the normal app navigation.
- Add-account is not a separate auth implementation: callers create `new LoginActivity(account)` for a free account slot and reuse the same flow. Account switching and add-account affordances are in `MainTabsActivity`, `SettingsActivity`, `UserInfoActivity`, `LogoutActivity`, and `LaunchActivity`.
- The authorization phone page currently has no three-dot overflow menu. `LoginActivity.createView()` installs back/done handling and a separate proxy control; there is no tdata/session login item or picker. The ordinary phone → code → 2FA flow is a mature, intertwined state machine and must remain intact if another entry point is added.
- `SettingsActivity` routes “Devices” to `SessionsActivity`, which manages Telegram server sessions/devices. It is not currently an account-import screen.

### Accounts and lifecycle

- `UserConfig` stores per-account client identity and settings. `AccountInstance` exposes account-scoped controllers (`MessagesController`, `MessagesStorage`, `ContactsController`, `MediaDataController`, `ConnectionsManager`, `NotificationsController`, `SecretChatHelper`, etc.).
- Important capacity mismatch found in the current source:
  - Java `UserConfig.MAX_ACCOUNT_DEFAULT_COUNT = 3` and `MAX_ACCOUNT_COUNT = 4`.
  - `UserConfig.getMaxAccountCount()` reports a default/premium-dependent UI limit (3/5), which is distinct from the number of Java account slots.
  - Numerous Java singleton arrays and arrays in UI/controller code are allocated with `UserConfig.MAX_ACCOUNT_COUNT`.
  - Native `TMessagesProj/jni/tgnet/Defines.h` defines `MAX_ACCOUNT_COUNT` as `5` and JNI networking state is indexed by account.
  - Changing one constant would leave Java arrays, UI affordances, storage paths and JNI/native arrays inconsistent. The requested 15-account change needs a centralized capacity design and an audit of every fixed-size per-account structure; it is not a safe one-line limit change.
- `ConnectionsManager` is instantiated per account and receives a private config path under the app files directory (`accountN` for nonzero accounts). Native tgnet persists its configuration as `tgnet.dat`; `Datacenter` serialization includes permanent and temporary MTProto auth-key state. This is credential material, not a disposable cache.

### Telegram protocol, models and networking

- Generated Telegram API types and methods are in `org.telegram.tgnet.TLRPC` and `org.telegram.tgnet.tl.*`; model generation/validation tasks are in `buildSrc` and the checked-in schema sources.
- `MessagesController`, `ContactsController`, `MediaDataController`, `SendMessagesHelper`, `SecretChatHelper`, `StoriesController`, and related controllers coordinate server requests, updates and account state.
- `ConnectionsManager.java` is the Java/JNI boundary. Native `TMessagesProj/jni/tgnet` owns MTProto connections, datacenters, handshake/key material, request scheduling, and `tgnet.dat` persistence. `TMessagesProj/jni/CMakeLists.txt` also builds media, SQLite, crash reporting and other native pieces.
- `TLRPC.Message`/`Chat`/`User` and associated generated types are protocol models. `MessageObject`, `ChatObject`, `UserObject`, and UI adapters are client/view helpers; UI model state is not proof that a server-side RPC succeeded.

### Local persistence

- `MessagesStorage` is a per-account SQLite store, accessed through a storage queue. `cache4.db` is under the app-private files directory; nonzero accounts use `accountN/cache4.db`. The current schema constant is `LAST_DB_VERSION = 179` and migrations are in `MessagesStorage`.
- Account preferences are separate: `UserConfig` uses the historical `userconfing` preference file for account 0 and `userconfigN` for additional slots. Per-account main/notification/emoji preferences are in `MessagesController` (`mainconfigN`, `NotificationsN`, `emojiN`).
- The message DB is Telegram client state/cache, not a portable cross-client session format. Custom aliases, categories, retention metadata, and custom settings should not be mixed into cloud message tables; a separately versioned custom-data store is required by the requested design.

### Notifications, background work and push

- FCM delivery enters `GcmPushListenerService`; `PushListenerController` decrypts/decodes the Telegram push payload, associates it with an account, dispatches updates and participates in per-account push registration. Huawei delivery is wired by the Huawei app variant/manifest.
- `NotificationsController` is per-account and builds local Android notifications; `NotificationsService`, `KeepAliveJob`, activity lifecycle calls to `ConnectionsManager.setAppPaused`, and `ConnectionsManager.setPushConnectionEnabled` participate in background behavior.
- Existing push/background settings are not the requested four-mode, independent per-account policy. Android Doze, OEM battery management, process death and force-stop remain platform constraints. Force-stop cannot be overridden or promised away by an app.

### Settings, localization and tests

- Settings use Telegram’s fragment/cell/theme infrastructure in `SettingsActivity` and the individual `*Activity`/fragment screens; strings live in Android resources under `TMessagesProj/src/main/res/values*` (including `values-ru`).
- `TMessagesProj_AppTests` contains Android instrumentation tests, including generated TL scheme tests plus TLS/database tests. These are not tests for any requested fork features. No build or test workflow currently exists in GitHub Actions.

## Authorization/session implications

A correct account restore must coordinate at least the native tgnet per-DC authorization/config (`tgnet.dat`), the app-side account identity/preferences, startup initialization, and a real Telegram connection/authorization check. Creating a local `UserConfig`/profile object alone would be a fake login. There is no importer/exporter or file-picker restore path in this source tree at the audited revision.

See [CUSTOM_FEATURES.md](CUSTOM_FEATURES.md) for the feature-to-code map and explicit implementation status, and [BACKUP_FORMAT.md](BACKUP_FORMAT.md) for the source-based Telegram Desktop format investigation.