# Vulnerability hunting method

Used for the Security track of a review (Phase 3A in [review.md](../workflows/review.md)) and for deep review ([review-deep.md](../workflows/review-deep.md)). Complements the 5 verification gates; does not replace them. Adapted from cloudflare/security-audit-skill (MIT).

## 1. Boundary and outcome

Every candidate must state all six points. If one is missing, it is not a finding yet:

| # | Point | Example |
|---|---|---|
| 1 | Low-trust principal and initial capability | Logged-in user of tenant A |
| 2 | Input, action, state transition, or resource selector it can supply | `PATCH /invoices/{id}` with an `id` from tenant B |
| 3 | Control that should block, constrain, isolate, limit, or revoke | `InvoiceService.update` should check `org_id` |
| 4 | Boundary crossed | Tenant |
| 5 | Principal or resource affected | Tenant B's invoice |
| 6 | Concrete outcome, observable by the attacker or by the system owner | Invoice B's amount is modified |

Do not promote a missing best practice, a guess about deployment behavior, a generic parser crash, or self-impact to a security finding.

## 2. Invariant-based hunting

For each assigned input (a hunk in the diff, a coverage unit in deep review):

1. **Read deeply**: follow the input through parsing, identity, authz, normalization, state, derived copies, to the final sink.
2. **Read sibling paths**: legacy, batch, bulk, retry, cancel, migration, error branches, import/export, admin, background jobs that produce the same effect. **The real policy is the weakest parallel path.** Compare sibling controls for **equivalence**, not just "present or absent".
3. **Compare handoffs**: what component A guarantees versus what component B assumes (truncation, type coercion, normalization, tenant scope, plugin permissions).
4. **Walk the bad paths**: missing, empty, 0, negative, max, over the limit, duplicate, mixed encoding, stale, revoked, reordered, concurrent, half-finished migration, failing dependency, rollback. Try only what the interface actually accepts.
5. **Limit depth**: trace only paths that reach the boundary under review or that the boundary relies on. Once the invariant is clear (holds or broken), stop and record the result.
6. **Look for variants**: when a HIGH/CRITICAL finding exposes a reusable root cause, look for lexical, structural, and logical variants in scope (Grep; `variant-analysis` if available, see [tob.md](tob.md)). Same root cause: one finding with multiple locations; each variant must still prove its own conditions and consequence.
7. **Multi-step bugs**: each output is a prerequisite; do not assume the downstream boundary. If one premise is missing, record it as a blocker.

Fix: find the **invariant** the code must hold and the **smallest** change that enforces it **at the final trust decision point**, with a regression test. Prefer a change specific to repo paths over generic hardening advice.

## 3. Three kinds of conclusion

| Conclusion | When | Severity | Where it goes in the report |
|---|---|---|---|
| **Finding** | Full source trace, passed all gates, concrete outcome | Yes | Security findings |
| **Needs validation** | The source trace to a boundary is real, but **a decisive fact lies outside the repo**: proxy/CDN/IdP configuration, account IAM policy, broker ACL, browser defaults, ingress, admission, branch protection, runtime version | **No** severity assigned | "Needs validation" section, with the missing fact and a safe way to check |
| **Rejected** | The source or a visible control is refutable, the consequence is missing, or a premise is impossible | No | Note (one line) when it helps the reader understand why it was not reported |

Rules:

- **Do not guess facts outside the repo in either direction.** Do not assume the proxy stripped the header, and do not assume it did not.
- "Needs validation" is **not** a place for vague ideas or low-confidence findings. It requires: a real `file:line` trace, a concrete boundary, a possible outcome, **exactly one or a few missing facts**, and a check plan: local (fixture, test, rendered manifest) and/or observed by the system owner (look at config, policy, routes). Never "send a test request to the live environment".
- If the source is refutable, it is Rejected, not Needs validation.
- Needs validation does not affect the merge verdict, but is listed in the report with the missing fact so the system owner can answer.

## 4. Hardening is not a finding

When layer A already blocks the attack, the absence of layer B is a **hardening note**, not a finding. A missing header, missing cookie flag, missing MFA prompt, missing rate limit, broad IAM permission, or missing seccomp becomes a finding only when there is a reachable path and a concrete consequence. Group hardening into its own section of the report, one line per item, with `file:line`.

## 5. Safe verification

- By default, only read source and run scanners. Do not probe deployed endpoints, external services, shared infrastructure, production identities, other people's data, or real control planes.
- Run the repo's tests, builds, and scripts only when needed to settle a candidate, with fake data, no external network, no dependency installs, no writes outside a temp directory, and with a time limit. If that cannot be done, **do not run it**: keep the candidate in "Needs validation" with the blocker "no isolated execution environment available" and state the test that would be run.
- Stop at the smallest effect that proves the point (wrong returned value, fake record read across tenants, sanitizer warning). Do not extend into exploitation, persistence, or concealment; do not load-test real services; do not burn paid quota.
- Credentials, secrets, and principals used in verification are all fake. Do not copy the machine's ENV or login state into the report.

## 6. Domain selection

From the inventory (Phase 0) or recon (deep review), pick domain files by the **visible trust boundary**, not by language name or dependency. Do not skip a visible boundary just because you think another part will cover it.

| Signal in the diff/repo | Read |
|---|---|
| Proxy, CDN, cache, gateway, hand-written HTTP parser; session, CSRF, JWT, OAuth/OIDC, SAML, MFA, passkey, reset/recovery, account linking, API key, mTLS | [domains/web-auth.md](domains/web-auth.md) |
| SPA, extension, webview, service worker, `postMessage`, WebSocket, CORS, browser storage, DOM rendering | [domains/client-side.md](domains/client-side.md) |
| Multi-tenant, cache/search/index/embedding, signed URL, export/import/backup/restore, migration, soft-delete, revocation | [domains/data-isolation.md](domains/data-isolation.md) |
| Input that consumes CPU/memory/disk/connections/workers/quota/paid APIs; regex, decompression, GraphQL, queue | [domains/resource-exhaustion.md](domains/resource-exhaustion.md) |
| gRPC, GraphQL transport, Protobuf, hand-written protocol, Kafka/RabbitMQ/SQS/pub-sub, webhook, streaming | [domains/protocols-messaging.md](domains/protocols-messaging.md) |
| IAM, Terraform/Helm/Kustomize, Kubernetes, mesh, serverless/edge, ingress, bucket, per-environment configuration | [domains/cloud-deployment.md](domains/cloud-deployment.md) |
| CI workflow, build, release, signing, promotion, updater, plugin/extension | [domains/ci-release.md](domains/ci-release.md) + [supply-chain-devsecops.md](supply-chain-devsecops.md) |
| LLM, RAG, agent memory, tool calling, MCP | [ai-llm-security.md](ai-llm-security.md) |
| Desktop app, privileged helper, daemon, local IPC, deeplink, webview bridge | [domains/local-ipc-desktop.md](domains/local-ipc-desktop.md) + [mobile/](mobile/) if mobile |
| C/C++/Rust `unsafe`/FFI, binary parser, kernel, loader, JIT | [domains/native-memory.md](domains/native-memory.md) |

Each domain file has: Discipline (pass verbatim into the subagent prompt), attack-class table, Common moves, Verification before reporting.

## 7. Additional attack classes

Beyond the hunting table in Phase 3A. Standard review: use when the diff touches the relevant spot; "The obvious stuff" always applies to added lines. Deep review: put them into the coverage table; for whole-repo scope they are mandatory in full.

### Feature abuse and data leaks

- **Export/backup to extract data**: a low-privilege user triggers an export, snapshot, or backup containing data beyond their permissions, other users' data, drafts, deleted items, history that should have been purged.
- **Import/restore to inject**: overwrite existing data, create records that bypass validation, insert into collections without write permission, not following the same permission model as the UI.
- **Search/filter/sort as an oracle**: a query reveals the existence of content the user may not view; filters probe state, role, hidden fields; sorting by a hidden field leaks values through ordering.
- **Enumeration through side effects**: error messages, timing, size, or status that differ between "does not exist" and "no permission"; user enumeration through password reset, invites, sign-up.
- **Preview/draft/staging leaks**: a preview token that covers more than one item; drafts exposed through search, RSS, sitemap, list APIs; cache headers that make the CDN serve private content to others.
- **Notification/webhook becomes SSRF**: a user sets a notify/webhook/callback URL that the server fetches; check the internal network, including after redirects.

### Vulnerability chaining and trust boundaries

- **Multi-step bugs**: list what the low-privilege principal can read, write, call, and hold, then connect only concrete outputs into later trust decisions. Prove each premise.
- **Trust gap between components**: A validates then passes to B; compare A's exact guarantee with B's assumption.
- **Second-order**: data that is safe when stored becomes dangerous later: field names become JSON paths, slugs become file paths, escaped text goes into raw rendering, stored strings become URLs, regexes, templates, policy expressions.
- **Scope and capability creep**: tokens, API keys, plugins, OAuth, MCP, AI permissions widen after delegation, refresh, caching, role changes, composition.
- **Timing and ordering**: setup windows, migration, soft-delete, revocation/cache expiry, check/use, validate/consume.
- **Rollback, recovery**: undelete, restore, revert, cancel must apply **current** ownership, validation, and authz.

### Wildcard (deep review)

A subagent that is not assigned a bug class looks outside the standard classes:

- What is the strangest code in the repo, why does it exist, and what happens if it is abused. Half-finished, experimental, bolted-on features (the least reviewed).
- APIs that can be called but the frontend never calls; hidden endpoints, parameters, or headers not in the docs (read route registration, middleware, config).
- Feature mixes not designed to go together: localization + preview + cache; import + plugin + webhook; OAuth + impersonation + API key.
- Git history: a reverted security patch, a commented-out auth check, a secret committed then deleted (still in history).
- Actions by a legitimate account that affect others, shared integrity, availability, operator cost. Irreversible operations or those needing elevated confirmation: authz and approval bound to the principal, action, and final resource.
- Assumptions about the environment: local DB, correct clock, trustworthy DNS, case-sensitive FS.
- What tests do **not** check: compare the boundaries the author thought of (has tests) with the ones they did not.
- Comments explaining why something is safe: verify the explanation. Variables named `temp`, `hack`, `legacy`: read carefully.

### The obvious stuff

One pass checking each item, literally, reporting the result of each; every hit must still trace its consequence before becoming a finding:

- Hardcoded secrets (`password`, `secret`, `apikey`, `token`, `Bearer`, `-----BEGIN`, default passwords); committed `.env`, `credentials.json`, `*.pem`, `*.key` files; whether `.gitignore` covers secrets, uploads, local config.
- TODO/FIXME/HACK/XXX mentioning security (`TODO: add auth`, `HACK: skip permission check`).
- Debug/dev mode that can be enabled in prod via ENV, query, header; seed/test/example credentials that work in prod.
- Unprotected `/debug`, `/admin`, `/test`, `/status`, `/health`, `/metrics`, `/env`, `/.env`, `/config`, `/actuator`, `/swagger`.
- `eval`, `exec`, `child_process`, `Function()`, `vm.runInContext`, `import()` with dynamic input.
- CORS `*` or reflected origin with credentials; cookies missing `HttpOnly`/`Secure`/`SameSite` (check whether the cookie is sensitive and whether JS needs to read it); open redirect via `redirect`, `return`, `next`, `url`, `goto`, `continue`; plain HTTP endpoints; prod errors returning stack traces, internal paths, SQL errors.
- Whether dependencies are pinned, and whether the lockfile contains CVEs.

## 8. Anti-patterns

Beyond the rules above (six points, Needs validation, hardening, safe verification):

1. Reporting a deviation from a checklist as if it were a vulnerability.
2. Guessing vendor, proxy, browser, IdP, or deployment behavior that is not in the source.
3. Treating the same principal's inherent permissions, or self-impact, as crossing a boundary.
4. Reporting a stronger consequence than the one observed (crash into RCE, ordinary work into shared DoS).
5. Writing the report before the adversarial challenge is done, or letting the prose and the findings table disagree.
6. Using an old confirmed finding as a template that anchors the hunt, or re-reporting an unchanged old finding as new.
