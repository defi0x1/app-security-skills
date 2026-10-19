# Domain: HTTP protocol, session, identity

Use when the diff or repo has: a reverse proxy, CDN, gateway, hand-written HTTP parser, cache; session, CSRF, JWT, OAuth/OIDC, SAML, MFA, passkey, account recovery, account linking, API key, mTLS. Supplements the Broken Access Control and Auth sections of [secure-coding-patterns.md](../secure-coding-patterns.md): access control asks "is this principal allowed to do it"; this file asks "does the HTTP/identity layer confuse the request, token, principal or authentication level".

## Discipline

- Framing and cache bugs need **two interpretations** of the same request/response/key. Name the two components and the normalized value on each side.
- For each credential: find the line that verifies the signature/secret and **every binding** of its role: issuer, audience, origin, RP, client, session, principal, resource, authentication level, expiry, one-time-use state.
- `Host`, `Forwarded`, `X-Forwarded-*`, `Origin`, `Referer`, redirect target, callback state, and URLs built from the request are trust decisions. Trace each one to the identity or response it affects.
- A missing header, cookie flag, MFA prompt, or rate limit is hardening. A finding needs: a wrong request accepted, cross-principal impact, an authentication-level downgrade, or credential exposure.
- Proxy, IdP, browser, cert, or deploy config outside the repo that decides the outcome: Needs validation.

## Framing and cache

| Layer | What to hunt | It is a finding when |
|---|---|---|
| Request smuggling / desync | Multiple `Content-Length`, `Transfer-Encoding`, HTTP/2→1 downgrade (H2.CL, H2.TE), header-name normalization, CR/LF, obsolete line folding | You can point to the bytes the front end assigns to this request and the back end assigns to the next one. A single server behind a managed proxy not visible in the repo: Needs validation |
| Cache poisoning via unkeyed input | Forwarded host/scheme, selective cookies, language/device headers, auth state that changes content but is not in the cache key | A response cached for others carries attacker-chosen content |
| Cache deception | Private dynamic paths treated as static (`/account/x.css`, `;` path parameter), personal responses missing `Cache-Control: private/no-store` | The victim's response is cached publicly |
| Trusting Host / forwarded headers | Absolute URLs, tenant routing, password reset links, callbacks, client IP for authz/rate limit taken from client-sent headers | The trusted ingress does not strip the client-sent copy (check proxy config in the repo) |
| Header injection | Untrusted data in `Location`, `Set-Cookie`, CSP | The framework does not block CR/LF and the response changes in a security-relevant way |

## Browser sessions

| Layer | What to hunt | It is a finding when |
|---|---|---|
| CSRF | Every cookie-authenticated mutation: form, JSON, multipart, method-override, legacy routes; login CSRF | Ambient credential + state-changing route + cross-site request accepted + no token / no effective Origin check. Routes that need a non-ambient bearer token do not count |
| Session fixation, non-invalidation | No rotation on login, account switch, MFA completion, impersonation; still valid after logout, password change, account lock | Check refresh tokens, signed cookies, WebSocket, caches, fallback endpoints too |
| Cookie scope | `Domain`/`Path` too broad, same-name sibling cookies chosen by another component | A less-trusted origin or a realistic network position can read/overwrite the credential. Missing flags only: hardening |

## Federated identity

Establish the role first: redirect allowlisting and code issuance belong to the authorization server, not the relying party. Verification and binding bugs belong to the component that consumes the token.

| Layer | Check |
|---|---|
| JWT | Signature is verified; algorithm and key source fixed by the server; `exp`, `nbf`, `aud`, `iss`; `kid`/`jku`/`x5u` are untrusted key selectors; `none` and HS/RS key confusion; decode-without-verify paths. A token valid for another service is the wrong token here |
| OAuth/OIDC | `redirect_uri` exact match (if AS); `state` bound to the session; PKCE (code injection); ID token issuer/audience/signature/nonce; with multiple IdPs, bind the selected IdP (mix-up). Compare first callback, retry, mobile/deeplink, account-link |
| SAML | The signature-verified element is the same element used for identity (signature wrapping); unsigned/fallback paths; safe XML parser; canonicalization; audience, recipient, `InResponseTo`, replay protection |

## MFA, passkey, account state transitions

| Layer | Check |
|---|---|
| Enrolling/downgrading MFA | Can a valid factor 1 self-enroll, replace, or disable factor 2, generate recovery codes, or create a trusted device without re-authentication |
| Step-up | The challenge is bound to the right principal, session, target, operation/resource, expiry, one-time use. Compare UI, API, batch, recovery, resume flows |
| WebAuthn | Registration: challenge, RP ID, origin, credential, userHandle, algorithm, UV bound to the initiating session. Authentication: challenge, RP/origin, credential belongs to the user, signature, UP/UV |
| Account linking | Adding an IdP, passkey, email, phone, device needs the current session, proof of ownership of the new identity, step-up per policy, callback state bound to the initiating account. Unlink/relink, accepting an invitation with a duplicate identifier/tenant |
| Recovery | Random token, bound to user/action, expiry, one-time, rate limit, the URL sent out is not built from Host, old tokens/sessions invalidated. A response difference that only reveals "account exists" publicly: not automatically a finding |

## API key and mTLS

| Layer | Check |
|---|---|
| API key scope | The key reaches a wider tenant/resource/action/environment than the server-side record; request parameters override the binding; publishable and secret keys confused; comparing a prefix instead of the whole secret; revocation cache; bulk endpoints |
| API key exposure | In the client bundle, URLs, redirects, logs, errors, build artifacts. A public identifier called a "key" is not a secret |
| mTLS | Trusting a cert-identity header from any peer; a subject the attacker influences mapped to an account; accepting a cert from the wrong trust domain/EKU. When a proxy terminates mTLS: only the proxy can connect, it strips incoming identity headers, the backend attaches the sanitized identity |
| Cert fail-open | An expired/revoked/missing cert falls back to bearer-only or anonymous; a connection pool keeps privileges after revocation. A fail-open branch in the repo: confirmable |

## Common moves

- Walk the full credential lifecycle: issue → store → transmit → use → refresh → revoke. Compare the normal, error, retry, migration, legacy, and account-switch branches.
- List every entry point to the same identity and every route to the same sensitive operation. The real policy is the **weakest parallel path**.
- Put side by side how the parser, proxy, router, cache, and app normalize the same value.
- Recovery and linking: draw the account graph before/after; for each edge state the current principal, proof of ownership, authentication level, callback/session binding, revocation effect.

## Verification before reporting

1. Framing/cache: name the two components and the differing delimiter/key values; impact across requests, across users, or on a private response.
2. Token, MFA, passkey, linking, recovery, API key, mTLS: quote the verify line and the missing binding; prove the server accepts the wrong state transition or credential.
3. CSRF: state the ambient credential, the state-changing route, the accepted cross-site request form, the cookie policy, and the missing effective control.
4. Check framework/library defaults against the exact version; if version or config is unclear, Needs validation (no severity).
