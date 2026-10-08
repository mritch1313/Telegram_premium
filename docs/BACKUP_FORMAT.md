# Account backup formats — source investigation and implementation status

## Status

**No account backup/import/export code exists in this Android checkout.** No `.tdata` or `.session` output is generated, and no selected file is treated as a logged-in account. This document is a source-based compatibility investigation, not a claim of import/export support. No real account secrets or user fixtures were used.

## Official Telegram Desktop source audited

The official Telegram Desktop repository was inspected directly at `telegramdesktop/tdesktop` branch `dev`, commit `fc002c6112e6794db3229ed83feb7e5406165281` (2026-10-08). Relevant implementation files:

- [`storage_domain.cpp`](https://github.com/telegramdesktop/tdesktop/blob/fc002c6112e6794db3229ed83feb7e5406165281/Telegram/SourceFiles/storage/storage_domain.cpp)
- [`storage_account.cpp`](https://github.com/telegramdesktop/tdesktop/blob/fc002c6112e6794db3229ed83feb7e5406165281/Telegram/SourceFiles/storage/storage_account.cpp)
- [`storage_file_utilities.cpp`](https://github.com/telegramdesktop/tdesktop/blob/fc002c6112e6794db3229ed83feb7e5406165281/Telegram/SourceFiles/storage/details/storage_file_utilities.cpp)
- [`storage_file_utilities.h`](https://github.com/telegramdesktop/tdesktop/blob/fc002c6112e6794db3229ed83feb7e5406165281/Telegram/SourceFiles/storage/details/storage_file_utilities.h)
- [`localstorage.cpp`](https://github.com/telegramdesktop/tdesktop/blob/fc002c6112e6794db3229ed83feb7e5406165281/Telegram/SourceFiles/storage/localstorage.cpp)
- Desktop storage `AppVersion` in `core/version.h`: `7002010` / `7.2.10` for this source snapshot. This is a Desktop storage/app version, not the Android version `12.10.6`.

These files implement an internal, versioned application storage system. A TDesktop `tdata` is a directory tree. It is **not** a JSON document, arbitrary ZIP, generic `.session` file, or a copy of Android's SQLite cache.

## Verified format properties in the audited Desktop revision

1. The global data directory is `cWorkingDir() + "tdata/"` (`BaseGlobalPath()` in `storage_domain.cpp`). Account metadata is associated with `key_<dataName>` files; the file utility layer rotates generations with suffixes `0`, `1`, and `s`.
2. `key_<dataName>` content includes the local-key wrapping salt/encrypted local key and encrypted account-list information. The encrypted account manifest contains a count, account indices and active index. Current source appends a trailer identified by magic `0x4B443200`, format version `2`, committed generation and passcode-wrap metadata. The code also explicitly parses a legacy three-blob shape.
3. Desktop file descriptors use the actual `TDF$` magic, a stored `AppVersion`, serialized data and a 16-byte MD5 integrity trailer. Reads validate magic, reject versions newer than the reader, and verify the checksum before exposing the payload. Individual encrypted payloads are separately decrypted and length-checked. A suffix/file name alone is not a format check.
4. Desktop derives a local encryption key and encrypts its storage records. The source contains legacy PBKDF2-HMAC-SHA512 derivation and current passcode-wrapping paths (Argon2id or scrypt KDF metadata, depending on build/runtime support); the current open/passcode wraps, random salts, generations and compatibility paths are part of the implementation. Telegram's account 2FA password is not the same thing as a Desktop local app-lock passcode.
5. `storage_account.cpp` stores a per-account map and account files under an account hash-derived directory; the database path is `tdata/user_<dataName>/`. Names such as `map0`, `map1`, `maps`, and `config` are part of that account storage implementation. Auth/protocol state is handled by Desktop's `MTP::Config`/authorization serialization and encrypted with the local storage key; `writeMtpData()` writes `dbiMtpAuthorization` plus serialized MTP authorization data. This is not a flat, platform-neutral record.
6. Desktop supports legacy storage/migrations as well as the current path. The format is internal and its serialization/crypto/layout may change as TDesktop evolves. Compatibility must be pinned to a source revision and maintained alongside upstream.

The source links above are the reference implementation. This audit did not implement or independently reimplement the cryptographic format, nor did it generate or validate a real user `tdata` directory.

## Android format is different

At the Android revision in this fork:

- `ConnectionsManager.java` supplies a per-account files-directory path (`accountN` for nonzero slots) to native tgnet.
- Native `ConnectionsManager.cpp` persists its config as `tgnet.dat`; its serialized datacenter records contain DC/session configuration and MTProto key state (`Datacenter::serializeToStream`).
- `UserConfig` stores Android-side account identity/preferences separately (account 0 uses the historical `userconfing` SharedPreferences file; later slots use `userconfigN`).
- `MessagesStorage` uses per-account `cache4.db` and schema version 179.

Those structures are not TDesktop `tdata`. A real adapter needs to parse/validate the official Desktop envelope and cryptography, translate the relevant MTP authorization/DC state into the Android native tgnet representation, restore app-side account identity atomically, initialize the normal network stack, connect to Telegram, and verify that the server accepts the authorization before making the account visible. The direction from Android to Desktop must perform the inverse conversion and then be validated by an actual compatible Desktop build. Copying SQLite, writing a profile object or renaming a file does none of this.

## `.session` status and detection rules

There is no generic Telegram-wide `.session` file format established by this Android tree or by the TDesktop directory storage code inspected above. Client-specific session files must be named and versioned by a real parser/adapter. This fork currently has no such adapter. A future `FormatDetector` must use validated structure, signatures and serialization—not a filename extension—and return `Unsupported/corrupt` for unknown or malformed data.

## Import/export security requirements before implementation

- Treat any file carrying MTProto authorization as credential-equivalent. Never log keys, hashes that expose secrets, passwords or token material.
- Use app-private temporary storage, restrictive permissions and cleanup on both success and failure. Never stage raw secrets in public Downloads while importing.
- Do not store Telegram 2FA passwords in an export. If Desktop's local passcode wrapper is retained, document that separately; an additional wrapper must not be called TDesktop `tdata` unless the result is still an actual TDesktop-compatible directory.
- Preserve the source backup. Validate every length/count/version before allocation; reject unsupported versions and corrupted integrity; do not modify user-selected input.
- Add test fixtures without live credentials, then separately run cross-device tests with controlled test accounts only. Serializer/unit tests are not proof of Telegram authorization.

## Required validation gate

No format will be advertised as supported until all relevant checks pass: valid/corrupt/incomplete/unsupported fixtures; archive safety if implemented; export→import round trip; compatible Desktop→Android and Android→Desktop tests; first-account and additional-account import; live Telegram connection; server-authorized account check; normal chat-list operation; both authorization-screen and Settings → Devices entry points; and confirmation that phone/code/2FA login remains unchanged.

**Current result:** TDesktop source inspected; Android/Desktop conversion, import, export, session adapters, file picker, auth restoration, cross-device proof and APK tests are all **not implemented / not tested**.