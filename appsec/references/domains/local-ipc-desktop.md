# Domain: desktop, mobile (flows), local IPC

Use when the target is a desktop/mobile app, privileged helper, updater, local daemon, webview host, deeplink handler, browser native-messaging host, or local IPC client/server. Platform-specific code patterns for mobile (Manifest, Info.plist, Keystore, Keychain, WebView settings): [mobile/](../mobile/); this file hunts **flow and boundary bugs** that pattern grep does not see. Browser side of the webview: [client-side.md](client-side.md). Memory, loaders: [native-memory.md](native-memory.md). Update authenticity: [ci-release.md](ci-release.md).

## Discipline

- Identify the realistic attacker and starting capability: another app, a different OS user, a sandboxed child process, a downloaded document, a remote web origin. Self-harm within the same account and the same privilege is not a boundary crossing.
- Paths, process names, bundle/package IDs, and self-declared sender fields are **not** peer authentication. Use OS peer credentials, code identity (signature), capability handles, protected channel state.
- A native bridge or helper must authorize **each operation and the final resource** after parsing. A trusted UI or broker does not make attacker-influenced parameters trusted.
- OS sandbox, code signing, entitlements, permissions, keychain ACLs, exported-component policy, and confirmation dialogs are real controls when seen and pinned.
- Signing, post-merge manifest, OS version, device policy, installer ACLs, packaging not visible: Needs validation.

## Deeplinks, callbacks, navigation

| Layer | What to hunt |
|---|---|
| Deeplink, ambiguous custom scheme | Another app/page invokes a route that changes state, imports data, completes login, or selects an account without binding to the current session and a one-time callback. Check URI normalization, duplicate query parameters, scheme/host/path matching, stale/replayed links |
| App and account handoff | OAuth, SSO, magic link, invitation, device pairing, passwordless, payment callback returned to the wrong app, profile, tenant, or pending transaction. Bind state to the initiating app, session, account, provider, operation, expiry |
| File open, intent, share | Associated files, share intents, drag and drop, clipboard, notification actions, file-open events trigger a privileged operation without confirming content type, sender, current intent, final destination |

## Webview and native bridge

| Layer | What to hunt |
|---|---|
| Origin navigated to the bridge | A remote or attacker-controlled frame can call a JS/native bridge meant only for packaged content. Check origin **at call time** and after every navigation, redirect, subframe, popup, error page. A URL prefix check or checking only the first load is not enough |
| Over-broad bridge | Web content can choose arbitrary files, commands, IPC methods, credentials, system actions through a generic bridge. Method allowlist, normalized parameters, user/tenant permission, user-gesture requirement, return-value exposure |
| File and universal access | Remote content reads the app's local files, privileged custom scheme, internal origin created by file access, universal access, mixed content, debug, custom protocol handler merging origins. A missing hardening setting with no protected content reachable: hardening |

## Local IPC and exported components

| Layer | What to hunt |
|---|---|
| Missing IPC peer authentication | Unix socket, named pipe, XPC, Binder, D-Bus, native messaging, RPC, shared memory, loopback listener accepts a low-trust peer without checking OS credentials (`SO_PEERCRED`, audit token), code identity, sandbox token. Requires a meaningful method or data behind the channel |
| Self-declared principal differs from channel identity | The channel belongs to this app/user, but a request field selects another user, tenant, profile, or capability. Bind methods and resources to peer credentials |
| Exported component overreach | Activity, service, receiver, provider, automation endpoint callable from outside, doing work meant only for the app itself. Read the **post-merge manifest**, intent filters, permission/signature level, path grants, aliases |
| IPC lifecycle, correlation | Guessable request ID, reused handle, stale channel, inherited descriptor, world-writable socket path, a restart that lets one peer answer/cancel/reuse another peer's operation. Permissions for creating and cleaning up socket files, locks, ports, shared memory regions |

## Privileged helper and local files

| Layer | What to hunt |
|---|---|
| Privileged helper as confused deputy | A low-privilege caller can choose the command, file, service, user, or system setting without per-operation authz. sudo/polkit/UAC/XPC helper rules; the helper validates normalized parameters itself |
| Install, update, repair path | A privileged installer/helper reads manifests, scripts, packages, symlinks, working directories, or repair state that a low-trust actor can write **after** authz. Bind authz to immutable content and a safe destination |
| File ownership, TOCTOU | Path checked, then a replaced file, symlink, mount, or case/normalization difference is followed during privileged read/write. Use descriptor-based operations, check ownership at the end |
| Credential store crossing a boundary | Keychain/keystore items, token files, backups, logs, clipboard, notification previews, local config readable by a lower-privilege app/profile/user. Plaintext readable only by that same OS account is not automatically a vulnerability; name the low-privilege reader and the credential's power |

## App state and device lifecycle

| Layer | What to hunt |
|---|---|
| Leaks on account switch, logout, device restore | Caches, background tasks, widgets, notifications, local DB, webview storage, biometric approvals that survive logout/account switch and show up under the next account. Backup/restore and multi-profile |
| Pending operations, user presence | Notifications, widgets, shortcuts, share sheets, biometric prompts, deferred operations authorize an action different from the one displayed, run after expiry, or use another profile's state. Bind confirmation to the operation, resource, account, foreground state |

## Common moves

- Enumerate every local process, component, endpoint, URI scheme, file association, webview origin, helper. Record OS identity, runtime privileges, who can call it, which operations they can call.
- Read the final packaged input: post-merge manifest, entitlements, installer rules, native messaging registration, protocol handlers, ACL creation. Declarations in source can be overridden at a later step.

## Verification before reporting

1. State the attacker's starting capability, the OS/app principal crossed, the entry channel, the parameter or state accepted, and the unauthorized operation or exposed data.
2. Confirm whether OS sandbox, peer credentials, code signing, entitlements, permissions, user consent, or installer controls apply. Unclear packaging/runtime: Needs validation.
3. Webview bridge: quote both the origin/navigation check and the privileged native sink. IPC: quote peer authentication and per-resource authz. Helper: check the final normalized destination.
