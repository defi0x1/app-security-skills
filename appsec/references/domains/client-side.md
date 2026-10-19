# Domain: client-side, browser

Use when a trust decision or the rendering of untrusted data happens in the browser: SPA, browser extension, embedded webview, service worker, offline app, code receiving `postMessage`, code using browser storage. Some sources are never seen by the server: URL fragment, `window.name`, `postMessage`, cached content. Server-side XSS and basic CSP: [secure-coding-patterns.md](../secure-coding-patterns.md) section 7. Native side of a webview bridge: [local-ipc-desktop.md](local-ipc-desktop.md) and [mobile/](../mobile/). CSRF, server-side sessions: [web-auth.md](web-auth.md).

## Discipline

- Needs a **controllable source** and a **sink that executes or exposes data**. Name both and show the attacker's data reaching the sink.
- The consequence must touch the victim's session, another origin, or a shared storage area. Self-injection into oneself, or exposing the attacker's own data: not a finding.
- Framework escaping, same-origin policy, CSP, COOP/CORP, service worker scope, and default `noopener` are real controls. Check them before assigning severity.
- Storage and caches are shared per origin and can outlive the login session. Establish who writes, who reads, and which lifecycle (account, tenant, worker) clears each record.
- Renderer behavior, extension permissions, deploy headers, or browser policy you cannot see: Needs validation.

## DOM and object state

| Layer | What to hunt | It is a finding when |
|---|---|---|
| DOM XSS | `location.*`, `document.referrer`, `window.name`, message data, storage → `innerHTML`, `outerHTML`, `document.write`, `eval`/`setTimeout(string)`/`new Function`, `javascript:` URLs, jQuery `.html()`/`$(html)`, framework escape hatches | Attacker data reaches the sink without encoding. Interpolation already escaped by the framework: not a finding |
| DOM clobbering | Attacker-injected `id`/`name` shadowing globals, form properties, config objects, init flags | A markup path keeps the attribute **and** the shadowed value is used for a security decision |
| Prototype pollution + gadget | Attacker keys entering a deep merge, path assignment (`__proto__`, `constructor.prototype`) | There is a recursive write **and** a gadget that reads the polluted property and changes authz, execution, navigation, or rendering. `JSON.parse` alone or a shallow copy: not enough |

## Messaging and cross-origin networking

| Layer | What to hunt | It is a finding when |
|---|---|---|
| `postMessage` | A handler doing sensitive work with `event.data` without an exact origin allowlist (and `event.source` when multiple frames share an origin); sending sensitive data to `*` | The origin check is absent, or by substring, prefix, suffix, or unanchored regex (`indexOf`, `endsWith('.example.com')`), which is not an origin check |
| Cross-site WebSocket hijacking | The upgrade accepts ambient cookies from a foreign origin, no `Origin` check, no per-channel token | Both a bad upgrade **and** a message handler with security consequences |
| CORS with credentials | Reflecting or loosely comparing `Origin` + `Access-Control-Allow-Credentials: true` + a sensitive response | `*` + credentials is blocked by the browser itself; report only paths that truly reflect/allow the origin and the cross-origin data or mutation |

## Service worker and browser storage

| Layer | What to hunt | It is a finding when |
|---|---|---|
| Service worker scope takeover | Attacker content becomes the worker script; `Service-Worker-Allowed` too broad; `importScripts` without integrity | Check the final script URL, MIME, origin, scope, and who controls every imported script. A same-origin worker with the correct scope: no bug |
| Worker cache mixing identities | Caching personal responses not keyed by account, tenant, auth state; serving them after account switch/logout; offline fallback returning the previous user's response | There is a path to another user's data or data after logout |
| Storage exposure, stale authz | Tokens, private responses, authz decisions in `localStorage`, `sessionStorage`, IndexedDB, Cache Storage, extension storage | A less-privileged reader exists in practice, or the data is still used after revocation/logout. Merely storing a token: not yet a finding |
| Context confusion across tabs | `storage` events, `BroadcastChannel`, shared workers carrying identity/commands not bound to the current session | A stale tab overwrites new auth state, or an account/tenant switch mixes data |

## Cross-site leaks, UI redress, navigation

| Layer | What to hunt | It is a finding when |
|---|---|---|
| XS-Leaks | An attacker page distinguishes cross-origin state via load/error, frame count, redirect, timing, cache, size | There is **one concrete secret predicate** (a private object exists, a role, an account). Generic timing: no |
| Window/opener | Cross-origin window metadata leaks state; a retained `window.opener` lets an attacker page navigate a privileged page (reverse tabnabbing) | Check COOP, `noopener` (implicit for `target=_blank` in current browsers), the exact origin |
| Clickjacking | A state-changing action missing `frame-ancestors`/`X-Frame-Options` | A sensitive action can be completed inside a frame. A read-only page missing the header: hardening |
| Client-side navigation | A client-controlled source drives redirect/navigation without checking scheme or destination (`javascript:`, `data:`) | The attacker picks the destination or scheme and a victim session or origin is affected |

## Common moves

- Walk from the sink (DOM, navigation, worker, message, storage) back to browser-only sources and server-provided sources. Note the browser policy that should have blocked it.
- Try account switch, logout, worker update, offline fallback, and stale tabs with a test origin and dummy accounts.
- XS-Leak: list only provable predicates; name the header or rendering approach that removes the oracle.

## Verification before reporting

1. State the source, sink, browser policy, affected origin/session, and the observable mutation or data exposed.
2. Prototype pollution: prove the recursive write and a gadget with consequences. DOM clobbering: prove the markup survives and the shadowed value is used.
3. Worker, storage: prove a record written by the attacker reaches another account, tenant, or authz state.
4. Messaging, CORS, WebSocket, XS-Leaks: show the exact origin/source check and the state/action exposed; confirm CSP, COOP/CORP, cookies, SameSite do not already block it.
