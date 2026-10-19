# Change risk classification

Every MR belongs to one of 10 groups. The MR author declares the group (if any); the reviewer verifies it against the diff. Groups 1 to 8 are low-risk and may get a light review; 9 and 10 need a full one. The group is a label for the approver: it never lowers review depth, severity, or the verdict. The skill proposes a group; it does not decide the approval flow.

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

**Observed group**: the lowest-numbered group among 1 to 8 whose conditions all hold on the diff; otherwise **10**. Group 9 is never inferred, only declared. If the declared group's conditions fail, report Mismatch even when the finding itself (SCA, CWE-532 for logged PII) is reported separately.

## Sensitive code areas (shared by all groups)

Touching **one** line in these areas means groups 1 to 8 fail their "does not touch ..." condition. The regex in command 2 below is the single source: authentication/session, authorization, encryption/secrets, network security, sensitive data (PII, payment, export), system integration (webhook, gRPC, SDK), infrastructure/pipeline (Dockerfile, CI, k8s, terraform, migration, SQL), critical business logic (order, price, transfer, limit, state machine).

A path match is a **signal to read closely**, not a conclusion: `auth_test.go` that only changes a fixture can still be group 6. Always open the hunk.

## Signal-gathering commands (BASE/HEAD come from Phase 0 of workflows/review.md)

The examples below are written for `MODE=branch` (`"$BASE...HEAD"`). `MODE=local`: replace `"$BASE...HEAD"` with `"$BASE"` and add untracked files (`git ls-files --others --exclude-standard`).

```bash
# 1) Files touched + stats
git diff --stat "$BASE...HEAD"
git diff --name-status "$BASE...HEAD"

# 2) Files touching sensitive areas (paths). Case-insensitive; one fragment per area.
SENS='auth|login|session|jwt|token|oauth|sso|passw|otp|mfa|2fa|credential|cookie'
SENS+='|authz|permission|role|rbac|acl|policy|guard|middleware|tenant|owner|scope'
SENS+='|crypt|cipher|hash|hmac|sign|kms|vault|secret|\.env|keystore|keychain|\.pem|\.p12'
SENS+='|cors|csrf|tls|ssl|cert|pinning|helmet|csp|ingress|network.?policy|nginx|proxy'
SENS+='|pii|kyc|ssn|national.?id|identity|payment|card|bank|wallet|balance|payout|invoice|export|report'
SENS+='|webhook|callback|client|sdk|integration|gateway|adapter|grpc|\.proto|openapi|swagger'
SENS+='|Dockerfile|\.gitlab-ci|helm|k8s|kustomize|terraform|migration|schema|\.sql'
SENS+='|order|checkout|price|discount|voucher|loyalty|approve|reject|transfer|limit|quota|state.?machine|workflow'
git diff --name-only "$BASE...HEAD" | grep -iE "$SENS"

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

Fields are in [report-template.md](report-template.md).

- **Match**: all conditions met; the Signals line shows why.
- **Mismatch**: the specific violated condition + `file:line` + one sentence. Example: *"Declared group 6 (Refactor) but `internal/auth/middleware.go:77` changes the user id source from the JWT to the `x-user-id` header → changes security behavior → proposed group 10."*
- Group 9 (emergency fix): do not verify the conditions; record **"Needs re-review after release"** in the verdict even if the review is clean.
- Never write "safe" or "no review needed". Write "meets the conditions of group N" / "does not meet".
