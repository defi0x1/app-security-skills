# Secure design and threat modeling

Use when reviewing an MR in change group 10, a new integration, or an infrastructure change: if the MR description has security design notes, check the implementation against the authz matrix and acceptance criteria; if they are missing and the change is large, add it to Questions for the author (not a finding).

## Design documents need a security section

| Section | Must include |
|---|---|
| Connection protection | Cloud: a table of services with the security measures for each. On-premise: topology, VLANs, CIDRs, segmentation |
| Account management | An **RBAC table** and a **privileged account table** (including service accounts): provider, consumer, permissions |
| Business logic | Each feature: name, flowchart **plus** sequence diagram, default security control for each step |
| Acceptance criteria | Quantified and verifiable. "Ensure it is secure" and "has authentication" fail: a pentest has no baseline to verify against, and it cannot be audited after go-live |
| Integrations | A table: source, destination, protocol, port, auth, **where the secret is stored**, **whether the data contains PII**, **assessment/data processing agreement with the partner**, whether the flow is **internal or external** |

A common design flaw: service-to-service calls use a hardcoded API key in config, with a note saying "internal, so no authz needed". Correct: the key goes in a Secret Manager/Vault; internal flows still get least-privilege authz (zero trust); the service account is listed in the privileged account table.

Examples of measurable acceptance criteria:
- "Every `/api/v1/orders/*` endpoint returns 403 when `order.customer_id` differs from the user in the JWT; there is an integration test for cross-user access."
- "5 failed logins in 15 minutes per account lock the account for 15 minutes, with increasing backoff; the event is sent to the SIEM."
- "Access token TTL 15 minutes, refresh token 7 days, single-use rotation, revoked on logout."
- "Upload accepts only PNG/JPEG ≤ 5 MB checked by magic bytes; the file name is server-generated."

## Quick threat modeling: STRIDE-per-element

1. Draw a DFD: external entity, process, data store, data flow, **trust boundary** (Internet, partner, internal, admin).
2. Apply STRIDE per element:

| Element | Consider |
|---|---|
| External entity | S (Spoofing), R (Repudiation) |
| Process | S, T, R, I, D, E (all 6) |
| Data flow | T (Tampering), I (Information disclosure), D (DoS) |
| Data store | T, I, D (+R if it is a log/audit store) |

3. For each threat: the matching control + measurable acceptance criteria + test.

| Threat | Common controls |
|---|---|
| Spoofing | Strong AuthN (SSO/OIDC, MFA), mTLS/OIDC for S2S, correct JWT verification |
| Tampering | TLS, signatures/HMAC, write authz, input validation, integrity checks |
| Repudiation | Audit log with actor taken from the token, tamper-proof, sent to the SIEM |
| Information disclosure | Read authz (owner/tenant), encryption at-rest/in-transit, DTO allowlist, log masking |
| DoS | Rate limit, quota, size limits, timeout, queue |
| Elevation of privilege | Server-side RBAC, least privilege, no mass assignment, never trust a role from the client |

## "Security design notes" template (paste into the design document or MR description)

```markdown
### Security design notes
- Trust boundary and principal: {public / tenant user / partner / internal service / admin}
- New attack surface: {route, upload, webhook, consumer, LLM tool...}
- Sensitive data touched: {PII, payment, credential / none}
- AuthN: {mechanism}. AuthZ: {owner/tenant/role rule for each action}
- Integrations: {destination, auth, where the secret is stored, PII, internal/external}
- Main threats (STRIDE) and controls: {...}
- Security acceptance criteria (measurable): {...}
- Expected change group (1-10): {N} because {...}
```
