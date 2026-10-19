# Secure coding patterns (backend, web, API)

Each section: **Wrong** (a common pattern), **Bypass** (why a half-fix is not enough), **Right** (the standard fix). Use during review (workflows/review.md) to identify what is wrong and to write the Remediation part. Severity, CWE: [severity-cwe.md](severity-cwe.md).

General principles:
- Every deciding control (authn, authz, limits, prices, state) lives on the **server**. Client, gateway, and mobile app are only an extra layer.
- **Allowlist** instead of blocklist. A blocklist of characters, tags, or fields always misses something.
- Identity, tenant, and role come from the **session or a verified JWT**, not from headers, body, or query.
- Fix the root cause in code or configuration; a WAF, changing a port, or hiding a URL is only a temporary cover.
- AI-generated code is untrusted code: review it like code from a stranger.

## 1. Injection (CWE-89, 943, 78, 88, 94, 1336)

| Wrong | Bypass | Right |
|---|---|---|
| String concatenation into SQL | `mysqli_real_escape_string` does not protect numeric parameters outside quotes (`WHERE id = $id`); a SELECT-only DB account can still be read | Prepared statements / ORM binding. The ORM's raw, native queries are **not** protected |
| `ORDER BY $sort`, table name, column name from input | Identifiers cannot be bound with placeholders; escaping is useless | Compare against an **allowlist** of column names, use a default for unknown values; fixed `ASC/DESC` |
| `IN (...)` built by concatenating values | — | Generate the exact number of `?` (`nCopies(n,"?")`), limit the element count, bind each value. This pattern is a SAST false positive |
| Stored procedure that `EXEC`s a string built from parameters | Binding at the call layer does not save the dynamic SQL inside the proc | Use `sp_executesql` with parameters or drop dynamic SQL |
| `os.system("ping " + host)`, `exec`, `shell=True` | Blocking `; \| &` still misses `$()`, backticks, newlines | `subprocess.run([...], shell=False)` with a list of arguments + format validation (IP/hostname regex) + `--` before arguments when needed |
| `eval`, `render_template_string(user)`, `new Function` | — | Do not feed input into code/templates; use a fixed template and pass data through the context |
| NoSQL: `find(req.body)` | `{"$ne": null}`, `$where` | Coerce to scalar types, schema validation, forbid keys starting with `$` |
| LLM output into `subprocess`, SQL, `innerHTML` | A prompt is not a control | Treat LLM output as untrusted input; let the model pick an action from an allowlist, validate parameters |

## 2. Broken Access Control (CWE-639, 862, 863, 284, 22, 601, 352)

| Wrong | Bypass | Right |
|---|---|---|
| `GET /orders/{id}` reads by id, the gateway already checked the role | Customer A changes the id to view B's order (IDOR/BOLA) | The service checks `order.customer_id == user_from_token`; or query `WHERE id=? AND owner_id=?` |
| Checks `inv.org_id == user.org_id` when the spec says "only the creator or an accountant in the same org" | A colleague in the same org can view it | Implement the **spec exactly**: owner **or** (role=accountant **and** same org). Write a test for each branch |
| Switch ids to UUID to "prevent IDOR" | Hard to guess is not authorization | UUIDs are fine, but you must still check owner/tenant |
| A state-changing action (approve, delete, update) guarded only by view permission | — | Check a separate permission for each action |
| New route has no authz; middleware is opt-out | — | Global auth middleware; public routes must be declared explicitly and reviewed |
| Path: `new File(base + name)` then `getPath().startsWith(base)` | `getPath()` does not normalize `../`; `replace("../","")` is beaten by `....//` | Compare `startsWith` on `getCanonicalPath()` / `Path.normalize().toRealPath()` + base with a trailing `/`; better: map id → a server-generated file name |
| Open redirect: `if next.startswith("/")` | `//evil.com`, `/\evil.com` are interpreted by browsers as another host | Reject anything with a scheme/host, starting with `//` or `/\`; compare against a path or host allowlist |
| CSRF: `GET /account/delete` + `SameSite=Lax` cookie | Lax still sends the cookie on top-level navigation via GET | State-changing actions use POST/PUT/DELETE + CSRF token (or double-submit); Lax is only defense-in-depth |
| Multi-tenant: query, cache key, or job lacks `tenant_id` | Cross-tenant read | `tenant_id` comes from the token and is in every query, cache key, and message; RAG/vector search also filters by tenant/ACL |

## 3. Client-declared identity, auth, session (CWE-807, 290, 287, 306, 347, 384, 640)

- `user_id`, `tenant_id`, `role`, `org_id` from header/body/query: **wrong**. A `/internal/*` route guarded by a header with no credential: **wrong**. Service-to-service calls must carry a credential (mTLS, OIDC token, HMAC) and least-privilege authz, even on the internal network.
- JWT: pin a single algorithm. `jwt.verify(token, publicKey, {algorithms:['RS256','HS256']})` allows algorithm confusion (signing HS256 with the public key itself). Right: `{algorithms:['RS256']}`, check `iss`, `aud`, `exp`; do not accept `alg:none`; use a patched library (jsonwebtoken > 8.5.1).
- Session fixation: **issue a new session ID immediately after login**; do not accept a session ID via URL. Cookie: `HttpOnly`, `Secure`, `SameSite`.
- Reset, invite, and verify tokens: crypto-random ≥ 128 bits, short expiry, single-use, stored as a hash. Compare secrets in constant time.
- Rate limit per **account** on the server for login, OTP, PIN, reset (a 4-digit PIN is brute-forced in 2 minutes). A limit in the app is bypassed by calling the API directly.
- Passwords: Argon2id / bcrypt / scrypt with an appropriate cost. `sha256(salt+pw)`, SHA-512, double hashing are all wrong (too fast).
- Insecure design: easy-to-look-up security questions and no limit on attempts are spec bugs, fixed at the design level.

## 4. Insecure-by-default (CWE-1188, 1392, 489)

- `AUTH_DISABLED`, `SKIP_*`, `DEBUG` flags read from ENV; default values when ENV is missing (`dev-token`, `changeme`); admin port bound to `0.0.0.0`; `isLocalEnv` treating `dev`, `staging` as local. Right: **fail-closed**. If a required ENV is missing, do not start.
- Spring Boot `management.endpoints.web.exposure.include=*` in prod exposes `/actuator/env`, `/heapdump`. Right: prod exposes only `health`, verified in the pipeline baseline. Apply the same to debug, metrics, pprof, swagger in prod.

## 5. Sensitive data, logging (CWE-200, 359, 532, 312, 778)

- Do not return PII, hashes, or the whole user list; response DTOs use a field allowlist.
- Do not log request bodies, `Authorization` headers, cookies, passwords, tokens, email/phone/national ID numbers; no `%+v`, `JSON.stringify(req)`. Mask before writing.
- Sensitive actions (login, permission change, approve, delete, export) must have an audit log with the actor from the token; do not delete or lower the audit level. Logs go to the SIEM, with threshold alerts (e.g. failed logins).
- Verbose errors and stack traces do not go to the client.

## 6. SSRF (CWE-918)

| Wrong | Bypass | Right |
|---|---|---|
| Resolve DNS, check the IP, then let the HTTP client resolve again | DNS rebinding | Connect directly to **the exact IP that was checked** (keep Host header/SNI) or go through an egress proxy with an allowlist |
| `url.startswith("https://api.partner.com/")` + `allow_redirects=True` | The partner returns a 302 to an internal address | Disable redirects, or re-validate every hop |
| Block `169.254.169.254` as a string | `[::ffff:169.254.169.254]`, decimal, octal, IPv6 | Parse the IP properly, block private, loopback, link-local, IPv4-mapped IPv6; `validate_host` must strip scheme and port |
| Only enable IMDSv2 | Only an additional layer | Fix it in the app; if the IAM credential endpoint was already called, treat it as exposed and revoke the credential |

## 7. XSS (CWE-79)

- Encode by context on output (HTML entity, attribute, JS, URL). A `<script>` blacklist is beaten by `<img onerror>`, `<svg onload>`. `X-XSS-Protection` is obsolete.
- React, Vue, Angular auto-escaping is known-safe, except `dangerouslySetInnerHTML`, `v-html`, `[innerHTML]`, `bypassSecurityTrust*`. If HTML is needed, sanitize with DOMPurify.
- Image and link URLs from users: validate the `https:` scheme (block `javascript:`, `data:`).
- LLM output, RAG content: render with `textContent` or encode.
- CSP: run `Content-Security-Policy-Report-Only` to tune it, then enforce. For HSTS, raise `max-age` gradually before preloading.

## 8. Deserialization, XML, mass assignment (CWE-502, 611, 915)

- `pickle.loads`, `yaml.load`, `ObjectInputStream`, `unserialize`, `node-serialize` on untrusted data: code runs at deserialization time, and an `isinstance` check afterward is too late. Right: JSON + schema validation; `yaml.safe_load`. ML models as pickle `.bin`: prefer `safetensors`.
- XXE: `setExpandEntityReferences(false)` is **not enough**. Enable `http://apache.org/xml/features/disallow-doctype-decl`, disable external general and parameter entities, `XMLConstants.FEATURE_SECURE_PROCESSING`.
- Mass assignment: `body.pop('isAdmin'); user.update(body)` is a blocklist and misses `role`, `tenant_id`, `balance`. Right: a DTO/serializer **allowlist** of fields that may be updated.

## 9. Crypto (CWE-327, 328, 330, 916, 295, 319)

- No MD5 or SHA1 for security purposes; no ECB (`Cipher.getInstance("AES")` defaults to ECB); no static IV or salt; no `Math.random`, `rand` for tokens, keys, IVs.
- Data encryption: AES-GCM, a fresh random nonce every time, keys in KMS/Vault/Keystore, never hardcoded.
- TLS: do not disable verification, no trust-all; enable TLS + SASL/auth for Kafka, OpenSearch, DB, Redis.
- CORS: no `*` with credentials, do not reflect Origin; use an origin allowlist.

## 10. DoS (CWE-400, 770, 1333)

Limit body size, upload size and file type, page size, number of `$in`/`IN` elements, JSON/XML depth, import row count; use non-backtracking regex on long input; do no heavy processing before auth; LLM endpoints have auth, rate limit/quota per key, input limit, and `max_tokens`.

## 11. Race, TOCTOU (CWE-367, 362)

Check-then-act on balances, limits, vouchers, approval state, unique-by-select: use a transaction + `SELECT ... FOR UPDATE`, conditional update (`UPDATE ... WHERE balance >= ?` and check `rowsAffected`), a unique index, an idempotency key.

## 12. File upload

Check file type with magic bytes + an extension allowlist, limit size, use server-generated file names, store outside the webroot or in object storage, never execute, check authorization on download, scan for malware if available.

## 13. Business logic (deep review, large features)

Price manipulation, negative quantities, voucher reuse; skipped workflow steps; a state machine for state jumps (check valid transitions on the server); jobs and consumers processing across tenants; second-order (stored data reused in an unsafe context: stored XSS, SQL from a DB field, SSRF from a stored URL).

## Known-safe (do not flag without contrary evidence)

Parameterized queries or ORM binding; output through framework auto-escaping; ENV values, CLI flags, CI variables (except defaults when missing, and security-disabling flags); dependencies and base images through a controlled internal registry; UUID v4, crypto random; secrets in `*.example`, test fixtures; logging non-sensitive URLs and IDs; global auth middleware (flag only opt-out routes after checking that the middleware does not fail open); capability tokens with HMAC or verified JWT.
