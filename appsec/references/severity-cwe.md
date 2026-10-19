# Severity and CWE by finding type

Look up the table first. Deviate by at most one level and state the reason in the finding (for example, an internal system behind SSO is lowered one level; payment data is raised one level). Severity does not exceed the **proven consequence**; Needs validation items have no severity.

## Anchor scale

Use when the finding type is not in the table, or to check whether the table severity fits the real consequence:

| Level | Anchor |
|---|---|
| CRITICAL | Unauthenticated actor achieves code execution, access to the entire data store, or arbitrary account takeover |
| HIGH | Actor **completely defeats** a clear security control with a real consequence: authentication bypass, cross-tenant read/write, stored script running on other users, post-auth code execution, remote unauthenticated stop of a shared service |
| MEDIUM | A real boundary violation but with narrow scope, rare prerequisites, or a consequence contained in a small resource set |
| LOW | Disclosure of non-secret internal information, or sustained effort needed for minimal gain |
| INFO | Confirmed but minimal consequence, mostly a precondition in a larger chain |

Distinguishing HIGH from MEDIUM: does the proven outcome **completely defeat** a clear control for an action with a real consequence, or only **weaken** it? If you cannot state concrete damage, go to the lower level.

## Lookup table

| Finding type | Severity | CWE |
|---|---|---|
| RCE, OS command injection, code injection (`eval`, `exec`, template) | CRITICAL | CWE-78, CWE-94, CWE-1336 |
| Insecure deserialization of untrusted data | CRITICAL | CWE-502 |
| Auth bypass; endpoint that should require auth but does not; JWT not verified, `alg:none` | CRITICAL | CWE-306, CWE-287, CWE-347 |
| Real secret hardcoded or committed | CRITICAL if it grants prod access, otherwise HIGH | CWE-798 |
| SQL, NoSQL, SPL, KQL, LDAP injection | HIGH, CRITICAL if pre-auth | CWE-89, CWE-943, CWE-90 |
| Broken access control: IDOR, missing owner or tenant check, missing role check on a state-changing action | HIGH | CWE-639, CWE-862, CWE-863, CWE-284 |
| Server trusts client-declared identity (header, body) | HIGH | CWE-807, CWE-290 |
| Flag or default that disables security (`AUTH_DISABLED`, default token when ENV is missing, `skip_verify`) | HIGH | CWE-1188, CWE-1392, CWE-489 |
| SSRF to internal network, metadata | HIGH | CWE-918 |
| Stored XSS | HIGH | CWE-79 |
| Path traversal | HIGH | CWE-22 |
| XXE | HIGH | CWE-611 |
| Mass assignment into sensitive fields | HIGH | CWE-915 |
| Reset, invite, verify token that is guessable, never expires, or is not single-use | HIGH | CWE-640, CWE-330, CWE-613 |
| Race, TOCTOU on balance, quota, approval state | HIGH if financial, otherwise MEDIUM | CWE-367, CWE-362 |
| Reflected, DOM XSS | MEDIUM | CWE-79 |
| CSRF on a state-changing action | MEDIUM | CWE-352 |
| Open redirect, deeplink without scheme and host validation | MEDIUM | CWE-601 |
| DoS: unbounded body, `$in`, pagination, depth; ReDoS, zip bomb, amplifying query | MEDIUM, HIGH if unauthenticated and it affects other users | CWE-400, CWE-770, CWE-1333, CWE-409 |
| Sensitive information disclosure through responses | MEDIUM, HIGH if bulk or unauth | CWE-200, CWE-359 |
| Server-side logging of PII, credentials, tokens, request bodies | MEDIUM | CWE-532, CWE-312 |
| Missing or broken audit log for sensitive actions | MEDIUM | CWE-778, CWE-223 |
| Weak crypto for security: MD5, SHA1, ECB, static IV, `Math.random` | MEDIUM, HIGH if password or session token | CWE-327, CWE-328, CWE-330, CWE-916 |
| TLS verify disabled, trust-all, cleartext to external services | MEDIUM, HIGH if carrying credentials or PII | CWE-295, CWE-319 |
| CORS `*` with credentials or reflected origin | MEDIUM | CWE-942 |
| Dependency or base image pulled straight from a public registry, bypassing the organization's internal registry/controlled proxy (only when one exists) | MEDIUM, HIGH if production | CWE-829, CWE-1357 |
| Dependency with a HIGH, CRITICAL CVE that has a fix available | CVSS band (9.0+ CRITICAL, 7.0 to 8.9 HIGH); lower one level if the vulnerable code is not reached | CWE-1395 |
| Pipeline: security job `allow_failure`, `--exit-code 0`, `schedule` only; scanner disabled | MEDIUM | CWE-1357 |
| Suppression (`.trivyignore`, `nosemgrep`, `nosec`) without a reason | LOW | CWE-1357 |
| Missing lockfile, floating versions | LOW | CWE-1357 |
| Third-party script without SRI | MEDIUM if it runs in a context holding tokens, otherwise LOW | CWE-353, CWE-829 |
| Missing security headers | LOW | CWE-693, CWE-1021 |
| Verbose error, stack trace to the client | LOW | CWE-209 |
| Non-constant-time secret comparison | LOW | CWE-208 |
| User enumeration | LOW | CWE-204 |
| Missing rate limit on login, OTP, reset | MEDIUM | CWE-307 |
| Missing rate limit on an ordinary read endpoint | LOW, INFO if no concrete abuse | CWE-770 |
| Mobile resilience: root detection, obfuscation, anti-tamper | INFO, LOW at most | none |

### Domain additions ([domains/](domains/))

| Finding type | Severity | CWE |
|---|---|---|
| Request smuggling / desync between proxy and backend | HIGH, CRITICAL if it captures other users' requests or bypasses auth at the proxy | CWE-444 |
| Web cache poisoning; cache deception exposing private responses | HIGH if many users are affected or private data is exposed, otherwise MEDIUM | CWE-349, CWE-524, CWE-525 |
| Untrusted Host/forwarded header in reset links, callbacks, tenant routing | HIGH if account takeover via reset link, otherwise MEDIUM | CWE-807, CWE-640 |
| OAuth/OIDC missing `state`, PKCE, nonce, audience check; IdP confusion | HIGH | CWE-352, CWE-345, CWE-287 |
| SAML signature wrapping, verified element differs from the element used | CRITICAL | CWE-347 |
| Enrolling/changing/disabling MFA, step-up without re-authentication; account linking without proof of ownership | HIGH | CWE-308, CWE-287, CWE-284 |
| Session not invalidated after logout, password change, revocation | MEDIUM, HIGH if long-lived token | CWE-613 |
| API key accessing records beyond its server-side scope | HIGH | CWE-863 |
| Trusting mTLS/proxy identity header from any peer | HIGH | CWE-290 |
| Prototype pollution with a gadget | HIGH if it reaches execution or authz, otherwise MEDIUM | CWE-1321 |
| `postMessage` without origin check triggering a sensitive action or leaking data | MEDIUM, HIGH if session takeover | CWE-346, CWE-345 |
| Cross-site WebSocket hijacking | HIGH | CWE-1385 |
| Clickjacking on a sensitive action | MEDIUM | CWE-1021 |
| Cache key, object path, document ID lacking tenant, causing cross-tenant read/write | HIGH | CWE-639, CWE-668 |
| Export/backup including data beyond permissions; import/restore bypassing authz | HIGH | CWE-200, CWE-863 |
| Soft-delete bypass, old permissions still effective after member removal/role downgrade | MEDIUM, HIGH if cross-tenant | CWE-672, CWE-613 |
| Signed URL, share link exceeding scope or never expiring | MEDIUM | CWE-284, CWE-613 |
| Enumeration oracle through count, filter, sort, errors | LOW, MEDIUM if it exposes sensitive data | CWE-203, CWE-204 |
| gRPC interceptor not covering streams, reflection, gateway | HIGH if authz bypass | CWE-862 |
| Webhook/event without source signature verification, accepting replay | HIGH if it triggers a privileged or financial operation | CWE-345, CWE-294 |
| Duplicate message repeating a financial or state side effect | HIGH if financial, otherwise MEDIUM | CWE-841, CWE-362 |
| Workload identity, IAM over-permission with a selectable target input | HIGH | CWE-269, CWE-250 |
| Container can select privileged, hostPath, docker.sock | HIGH, CRITICAL if from low-trust input | CWE-250 |
| CI runs untrusted code together with secrets or a write token (`pull_request_target` + PR checkout) | CRITICAL | CWE-829, CWE-94 |
| Attacker-controlled CI expression injected into a shell | HIGH, CRITICAL if the job has secrets | CWE-78, CWE-94 |
| Cache/artifact from a low-trust job consumed by a privileged job | HIGH | CWE-345, CWE-829 |
| CI action not pinned to SHA, `curl \| sh` tool in a build | LOW, MEDIUM if the job has secrets or release permission | CWE-494, CWE-829 |
| Updater not authenticating metadata, allowing rollback | HIGH | CWE-494, CWE-347 |
| Webview bridge callable from an untrusted origin | HIGH | CWE-749, CWE-940 |
| Local IPC not authenticating the peer, privileged helper becomes a confused deputy | HIGH | CWE-306, CWE-441 |
| Loading a library/plugin from a path writable by a low-privilege user | HIGH | CWE-427, CWE-426 |
| Out-of-bounds write from untrusted input | CRITICAL if remotely reachable, otherwise HIGH | CWE-787 |
| Out-of-bounds read, use-after-free, integer overflow before allocation | HIGH | CWE-125, CWE-416, CWE-190 |
| Panic/abort/deadlock from untrusted input in a shared process | MEDIUM, HIGH if unauthenticated and stops the whole service | CWE-248, CWE-833 |
| Agent: attacker content triggers an action under the victim's permissions that was not requested/approved | HIGH | CWE-441, CWE-863 |
| Agent/tool uses a shared credential, does not check the requester's permission on the resource | HIGH | CWE-441, CWE-862 |
| Agent memory/RAG written by one principal and read by another | HIGH | CWE-668, CWE-1427 |

## Not a security finding

Do not put these in the findings table; they do not affect the verdict. If they need mentioning, one line in the Notes section.

- Dockerfile missing `HEALTHCHECK`, not pinning a digest, not multi-stage, running as root when the image exposes nothing sensitive.
- `console.log`, `print` that does not contain secrets, PII, or tokens, including printing the query string in a browser.
- `.env.*` files containing only URLs and public environment names.
- `allow_failure: true` on a non-security job (deploy, cleanup, lint).
- Comments, docstrings, naming, formatting, performance, functional bugs without a security consequence.
- Generic defense-in-depth recommendations not tied to a line of code in scope.
- Missing type or length validation on a field that does not reach a dangerous sink and does not cause DoS.
- Test code, fixtures, mocks, generated code (`*.pb.go`, `*_test.go`, `__mocks__`, `testdata/`), unless it contains a real secret.
- Missing defense layer B when layer A already blocks the attack, or a missing header, cookie flag, MFA prompt, rate limit, seccomp, least privilege with no reachable path and no concrete consequence: hardening ([hunting-method.md](hunting-method.md) section 4).
- Prompt injection that does not lead to a boundary violation (does not touch another principal, does not use permissions the requester lacks, does not reach a sink the requester cannot reach).
- Self-injection, self-impact, or use of the principal's own inherent permissions.
- Generic parser crash with no real untrusted input reaching it; a panic in the requester's own process.
- Floating dependency or one with a CVE where no one can influence resolution and the vulnerable code is not reached (the CVE is still reported per supply-chain-devsecops.md, with a reachability note).
- Response difference that only exposes already-public information (an account exists when the user list is already public).
- Storing a token in browser storage or plaintext that only that same OS account can read, when there is no lower-privilege reader.
