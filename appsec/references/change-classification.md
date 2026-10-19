# Change risk classification

Every MR belongs to one of 10 groups. The MR author declares the group (if any); the reviewer verifies it against the diff. Groups 1 to 8 are low-risk changes that can get a light review; groups 9 and 10 need a full security review. The skill proposes a group; it does not decide the approval flow.

## 10 groups and verification conditions

| # | Group | Conditions that must hold on the diff | Signs of NOT qualifying (→ move group) |
|---|---|---|---|
| 1 | **Non-sensitive data/config change** — data/config parameters for ordinary display/operations | Only data/config values change; no contact with auth/authz/crypto/secret/network security/sensitive data/integrations/critical business logic | Changes keys named `*secret*`, `*token*`, `*password*`, `*key*`, `cors*`, `tls*`, `auth*`, `*_URL` pointing to external systems, security timeouts/limits, role/permission mapping, `.env*`, prod config |
| 2 | **UI or static content** — color, font, layout, label, icon, text, translation, static docs | Only presentation files (`.css/.scss`, i18n `.json/.yaml`, `.md`, images/svg, template markup with no data binding change) | `.js/.ts/.tsx/.go/.py/.java/.kt/.swift` with logic changes; changes `dangerouslySetInnerHTML`/`v-html`/`innerHTML`; changes a link/redirect/deeplink; changes a form action/API path |
| 3 | **Fix functional/display/business-calculation bug** — does not change authentication, authorization, access control, encryption, session management, or sensitive data handling | Diff does not touch sensitive areas (table below); adds no route/API; does not change a query in a way that drops the owner/tenant filter | Removes/changes a `WHERE user_id/tenant_id` condition; changes middleware/guard; changes how token/password is compared; adds a client-controlled parameter to a query/command |
| 4 | **Dependency/package version upgrade** — compatible, trusted source, no change to architecture/security API, no serious CVE in the new version | Only manifest + lockfile; version bump within compatible semver; registry unchanged; `trivy fs` shows no HIGH/CRITICAL on the new version | Adds a **new** dependency; major bump; changes `.npmrc`/`pip.conf`/`settings.xml`/`FROM` to a public registry; comes with code changes outside the manifest; auth/crypto/TLS/serialization library |
| 5 | **Toggle a feature flag** for a feature already security-assessed — no code/logic change, no expansion of permissions/audience/data scope | Only a flag/config value changes; the feature was previously security-reviewed (author provides a reference); audience/role/tenant/data scope unchanged | Flag comes with code changes; flag opens the feature to a new audience (public, another tenant, a lower role); flag turns off a security control (`AUTH_DISABLED`, `SKIP_VERIFY`, `DEBUG`) |
| 6 | **Refactor/optimize/cleanup** — no change to inputs/outputs, processing flow, business logic, access permissions, API, security behavior | Adds/removes no route, parameter, or returned field; does not change the order of auth check → processing; existing tests keep their expectations | Changes the signature of a public function/handler; moves auth/validate checks elsewhere or removes them; ORM → raw query; deletes security tests |
| 7 | **Add/change logs, metrics, tracing, alerts** — logs no additional PII/credential/token/secret; does not reduce existing audit | Every `+` line is a log/metric/trace; log arguments contain no request body, Authorization header, token, password, email/phone/national ID; no audit log line removed | Logs `%+v`/`JSON.stringify(req)`/a whole object; logs headers/cookies; removes or downgrades audit logs; changes the actor/user_id recorded in audit |
| 8 | **Other functionality on an internal-only system** — full firewall + 2FA/SSO already in place | Author names the system + confirms internal-only use; diff does not expose an endpoint externally, adds no public ingress/route/CORS `*` | Changes ingress/network policy/CORS; adds an endpoint that bypasses the SSO middleware; the system has a mobile/public client |
| 9 | **Emergency fix for an operational incident** — security review required again after release | Author confirms it is an incident hotfix (incident id) | — (no group change; the skill still reviews fully and states "Needs re-review after release") |
| 10 | **Large change affecting business flows** | Default for every diff that does not meet the conditions of groups 1–8 | — |

**Proposed group when conditions are not met**: usually **10**; a dependency bump with a CVE → still 4 but "conditions not met" with an SCA finding; logging PII → 7 not met + CWE-532 finding.

## Sensitive code areas (shared by all groups)

Touching **one** line here means groups 1–8 do not meet the "does not touch …" condition:

| Area | Path / identifier signs (regex, case-insensitive) |
|---|---|
| Authentication / session | `auth`, `login`, `session`, `jwt`, `token`, `oauth`, `sso`, `passw`, `otp`, `mfa`, `2fa`, `credential`, `cookie` |
| Authorization | `authz`, `permission`, `role`, `rbac`, `acl`, `policy`, `guard`, `middleware`, `tenant`, `owner`, `scope` |
| Encryption / secrets | `crypt`, `cipher`, `hash`, `hmac`, `sign`, `kms`, `vault`, `secret`, `\.env`, `keystore`, `keychain`, `\.pem`, `\.p12` |
| Network security | `cors`, `csrf`, `tls`, `ssl`, `cert`, `pinning`, `helmet`, `csp`, `ingress`, `network.?policy`, `nginx`, `proxy` |
| Sensitive data | `pii`, `kyc`, `ssn`, `national_id`, `identity`, `payment`, `card`, `bank`, `wallet`, `balance`, `payout`, `invoice`, `export`, `report` |
| System integration | `webhook`, `callback`, `client`, `sdk`, `integration`, `gateway`, `adapter`, `grpc`, `\.proto`, `openapi`, `swagger` |
| Infrastructure / pipeline | `Dockerfile`, `\.gitlab-ci`, `helm`, `k8s`, `kustomize`, `terraform`, `migration`, `schema`, `\.sql` |
| Critical business logic | `order`, `checkout`, `price`, `discount`, `voucher`, `loyalty`, `approve`, `reject`, `transfer`, `limit`, `quota`, `state.?machine`, `workflow` |

This table is a **signal to read closely**, not a conclusion: a file named `auth_test.go` that only changes a fixture can still be group 6. Always open the hunk.

## Signal-gathering commands (BASE/HEAD come from Phase 0 of workflows/review.md)

The examples below are written for `MODE=branch` (`"$BASE...HEAD"`). `MODE=local`: replace `"$BASE...HEAD"` with `"$BASE"` and add untracked files (`git ls-files --others --exclude-standard`).

```bash
# 1) Files touched + stats
git diff --stat "$BASE...HEAD"
git diff --name-status "$BASE...HEAD"

# 2) Files touching sensitive areas (paths)
git diff --name-only "$BASE...HEAD" | grep -iE \
 'auth|login|session|jwt|token|oauth|sso|passw|otp|mfa|2fa|credential|cookie|authz|permission|role|rbac|acl|policy|guard|middleware|tenant|owner|scope|crypt|cipher|hash|hmac|sign|kms|vault|secret|\.env|keystore|keychain|\.pem|\.p12|cors|csrf|tls|ssl|cert|pinning|helmet|csp|ingress|network.?policy|nginx|proxy|pii|kyc|ssn|national.?id|identity|payment|card|bank|wallet|balance|payout|invoice|export|report|webhook|callback|client|sdk|integration|gateway|adapter|grpc|\.proto|openapi|swagger|Dockerfile|\.gitlab-ci|helm|k8s|kustomize|terraform|migration|schema|\.sql|order|checkout|price|discount|voucher|loyalty|approve|reject|transfer|limit|quota|state.?machine|workflow'

# 3) Added/removed line contents touching sensitive identifiers (even in harmlessly named files)
git diff -U0 "$BASE...HEAD" | grep -E '^[+-][^+-]' | grep -iE \
 'auth|permission|role|token|secret|password|crypt|cors|csrf|verify|tls|ssl|where.*(user|tenant|owner)_id|isAdmin|hasRole|allowAll|skip|disable|bypass'

# 4) New route/API? (added handler/route/decorator)
git diff -U0 "$BASE...HEAD" | grep -E '^\+' | grep -iE \
 '(app|router|r|e|g)\.(get|post|put|delete|patch|handle|use)\(|@(app|router)\.(route|get|post)|@(Get|Post|Put|Delete|Request)Mapping|HandleFunc|\.route\(|proxy_pass|path:'

# 5) Dependency / registry / infrastructure
git diff --name-only "$BASE...HEAD" | grep -iE \
 'package(-lock)?\.json|yarn\.lock|pnpm-lock|requirements|pyproject|poetry\.lock|go\.(mod|sum)|pom\.xml|build\.gradle|Podfile|pubspec|composer|\.npmrc|\.yarnrc|pip\.conf|settings\.xml|Dockerfile|\.gitlab-ci|helm|k8s|terraform'

# 6) Group 7 — is any added line NOT a log/metric/trace?
git diff -U0 "$BASE...HEAD" | grep -E '^\+[^+]' | grep -viE \
 'log(ger)?\.|console\.|zap\.|slog\.|logrus|printf|println|metric|counter|gauge|histogram|span|trace|otel|sentry|^\+\s*$|^\+\s*(//|#|\*)'
#   → any output = diff is not purely logging → group 7 not met. Also inspect the log arguments:
git diff -U0 "$BASE...HEAD" | grep -E '^\+' | grep -iE 'log|print' | grep -iE \
 '%\+v|%#v|JSON\.stringify\((req|request|body|headers|user)|req\.body|Authorization|cookie|password|token|secret|email|phone|ssn|national.?id'

# 7) Group 6 (refactor) — check for removed controls
git diff -U0 "$BASE...HEAD" | grep -E '^-[^-]' | grep -iE \
 'auth|permission|role|verify|validate|sanitize|escape|check|guard|require|limit|throttle|rate'
```

## How to record the conclusion

- **Match**: declared group is correct, all conditions met. Still include a "Signals" line so the approver sees why it matches.
- **Mismatch**: state the **specific condition** violated + `file:line` + one descriptive sentence. Example: *"Declared group 6 (Refactor) but `internal/auth/middleware.go:77` changes the user id source from the JWT to the `x-user-id` header → changes security behavior → proposed group 10."*
- Group 9 (emergency fix): do not verify the conditions, but record **"Needs re-review after release"** in the verdict even if the review is clean.
- Do not use the words "safe"/"no review needed". Use "meets the conditions of group N" / "does not meet".
