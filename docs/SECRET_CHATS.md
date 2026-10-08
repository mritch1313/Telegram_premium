# Secret Chats compatibility notes

## Existing implementation

Secret Chats use a separate encrypted-chat path, not the ordinary cloud-message flow. In this Android source, `SecretChatHelper` manages encrypted-chat handshakes, key exchange/re-keying, message encryption/decryption and key fingerprints. `TLRPC.EncryptedChat` carries encrypted-chat state; `MessagesStorage` persists encrypted-chat records and associated protocol state. `MessagesController.sendTyping()` has a distinct encrypted-chat branch and restricts which activity action it sends. UI components also have explicit Secret Chat branches and feature differences.

The account's cloud MTProto authorization and an individual Secret Chat's E2E key/state are different security objects. Copying the Android message database or importing a cloud account session does not establish that a Secret Chat can be restored safely on another device/client. An imported cloud session must not be reported as having restored Secret Chats unless their protocol state and E2E behavior are independently proven.

## Current feature status

No Secret Chat backup/import/export, retained-deletion system, privacy override, or content-protection bypass was implemented. No real E2E cross-device fixture/test was run. This fork does not claim Secret Chat compatibility with Desktop `tdata`.

## Compatibility rules for future work

- Do not treat Secret Chats as cloud chats or route them through generic message forwarding/restore code.
- Do not copy, log, export or expose `auth_key`, `future_auth_key`, fingerprints, DH material or other secret-chat key state as diagnostics.
- Preserve the existing encryption, key exchange, re-keying, random IDs, TTL/self-destruct and deletion semantics.
- Do not create a fake local “restored” encrypted chat. A real implementation needs a protocol-supported restore or a tested compatible encrypted-state migration, followed by cryptographic and server interaction checks.
- Test E2E send/receive, key rotation, reset/failure, TTL/deletion and app restart separately from cloud-session import. Do not use a real user's secrets as fixtures.

Until that evidence exists, Secret Chat backup/import and any extension that changes copy/forward/read behavior must be documented as unsupported or unverified, not “working.”