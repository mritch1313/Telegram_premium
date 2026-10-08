# Protected content and forwarding

## Source audit

The Android source carries Telegram protocol flags such as message/chat/story `noforwards` and checks them in client UI paths. Examples include `MessageObject`/`ChatObject`, `MessagesController.isPeerNoForwards`, `ChatActivity`, and `PhotoViewer`; the latter suppresses share/forward actions for protected items. TLRPC models preserve server-supplied protection state. The repository also includes normal Telegram forwarding RPC types/paths.

This distinguishes two things:

- **Client UI policy:** hide or disable copying, forwarding, selection, sharing or other actions when upstream marks content protected.
- **Server authority:** forwarding/copying by protocol operation is subject to server-side rules. A fork cannot turn an error response into a successful forward or honestly claim a server-side restriction was lifted by changing the UI.

## Safe behavior

No protected-content bypass was added. Any future implementation must preserve the actual error/result returned by Telegram and must not spoof success, silently clear protocol flags, or misrepresent an unavailable server operation. The fact that text/media has already been delivered to a client does not make a server-side forward permitted. UI-only access to already-rendered data must remain distinct from forwarding and must respect Telegram's content policy and user expectations.

The requested copy/forward behavior is **not implemented** and has not been tested. Before changing it, review the current protocol layer and client checks for protected channels, groups, protected private chats, stories and Secret Chats separately. Record which operation is a local UI action and which issues an RPC. Test a genuine server rejection and confirm the client reports it honestly.

## Secret Chats

Secret Chats are not ordinary cloud chats. Their E2E encryption, local keys, re-key flows and device-bound state need a separate compatibility review. See [SECRET_CHATS.md](SECRET_CHATS.md); no protected-content change should weaken those invariants.