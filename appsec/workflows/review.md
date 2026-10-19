# Workflow: Review

A single flow that makes every choice itself. The review has two equal tracks: **Security** (Phase 3A) and **Code** (Phase 3B: logic, business rules, data, reliability, performance, design and over-engineering, tests, maintainability, conventions). Respond in the user's language (default English); keep code and identifiers in English. Standards live in `../references/` (abbreviated `R/`). Find code with Grep/Glob/Read; `bash` commands are Unix examples. Do not install tools on your own. Always run every bug class and every severity level.

## Phase 0. Understand the request, get the diff (no questions)

### Read the parameters

Everything after `/appsec` is plain language, not flags: it may be empty, one of the items below, or a free sentence. Infer everything and **do not ask** about scope, change group, platform, deep mode or review track:

| Seen in the request | Effect |
|---|---|
| Nothing | **Auto**: work in progress on the current branch against the target branch (commits on the branch + uncommitted + new files) |
| `!123`, `#123`, `MR 123`, `PR 123`, a bare number, a URL containing `/merge_requests/123` or `/pull/123` | **MR/PR** by number. Several numbers: multiple MRs ([R/multi-mr.md](../references/multi-mr.md)) |
| "whole repo", "audit", "pre-pentest sweep", a directory ("src/billing") | **Whole repo / directory**: no diff needed, go straight to [review-deep.md](review-deep.md) (skip Phase 1 and Gate 4) |
| A branch name ("into develop", "against release/1.2") | Auto, `TARGET` = that branch |
| "uncommitted", "just changed" | Uncommitted part only: `MODE=local`, `BASE=HEAD` |
| "staged" | `git diff --cached` |
| "security only" / "code only" | Only one review track (both by default) |
| Business description, "Spec: ..." | Intent source for the business-logic review |

Stop and ask only when you cannot continue: not a git repo; the MR/PR cannot be fetched by any method below; the target branch cannot be determined and there is no remote.

### Work in progress (auto)

```bash
TARGET="${TARGET:-$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@^origin/@@')}"
TARGET="${TARGET:-main}"
git fetch -q origin "$TARGET" 2>/dev/null || true
REF="origin/$TARGET"; git rev-parse -q --verify "$REF" >/dev/null || REF="$TARGET"
git rev-parse -q --verify "$REF" >/dev/null || REF=""
BASE="${BASE:-$([ -n "$REF" ] && git merge-base "$REF" HEAD 2>/dev/null)}"   # no remote/target branch yet: BASE=HEAD
[ -n "$(git status --porcelain)" ] && MODE="${MODE:-local}" || MODE="${MODE:-branch}"
echo "branch=$(git branch --show-current) TARGET=$TARGET MODE=$MODE BASE=$BASE"; git status --short
if [ "$MODE" = local ]; then
  git diff --stat "$BASE"; git diff "$BASE"                      # commits on the branch + uncommitted changes
  git ls-files --others --exclude-standard                       # new untracked files: Read each file, treat all of it as added lines
else
  git diff --stat "$BASE...HEAD"; git diff "$BASE...HEAD"
fi
```

Empty diff (standing on the target branch with no changes): say one line "No changes compared to `{TARGET}`. Type `/appsec <MR number>` to review an MR." and stop.

### MR/PR by number

Detect the host from `git remote get-url origin`. Do not check out, do not create a worktree, do not change the user's branch.

```bash
N=123
# GitHub
gh pr view "$N" --json baseRefName,headRefOid,title,body 2>/dev/null
git fetch -q origin "pull/$N/head" && HEAD_MR=$(git rev-parse FETCH_HEAD)
# GitLab: glab / API / refs/merge-requests/$N/head per R/multi-mr.md
BASE=$(git merge-base "origin/<baseRef>" "$HEAD_MR")
git diff --stat "$BASE...$HEAD_MR"; git diff "$BASE...$HEAD_MR"
```

Read files at the MR head with `git show "$HEAD_MR:$FILE"`, search with `git grep -n "<pattern>" "$HEAD_MR" -- .`. The MR title and description are an intent source. If the current branch is the MR's source branch and has uncommitted changes: review against the MR head, and note in one line that the uncommitted part is out of scope.

### Conventions

`DIFF` is `"$BASE"` when `MODE=local`, `"$BASE...HEAD"` when `MODE=branch`, `"$BASE...$HEAD_MR"` for an MR. With `MODE=local`, the changed file list = `git diff --name-only "$BASE"` + untracked files. Gate 4 compares against `git show "$BASE:$FILE"`.

The review is **read-only**: do not edit files, do not `git add`, `commit`, `stash`, `checkout`, `reset`. The working tree may be under edit by a person or another session. Skip generated files (`*.pb.go`, `*_generated.*`) and vendor; see Phase 2 for lockfiles.

Collect intent: the spec the user gave, the repo's review rules (`REVIEW.md`, `CLAUDE.md`, `CONTRIBUTING`), MR/issue description, commit messages (`git log --format=%B $BASE..HEAD`), tests, `docs/`. Order of trust and how to handle gaps: [R/code-review.md](../references/code-review.md) section "Intent sources".

Inventory: new attack surface (route, handler, gRPC, upload, deserialization, webhook, consumer, external system call, LLM tool, dependency); sensitive code touched (auth, session, crypto, access control, validation, query, config, CORS, migration, CI, Docker, k8s); controls removed or loosened on `-` lines.

### Decide how to run

| Decision | Rule |
|---|---|
| Independent reviewer | This session wrote or modified code in the diff (in the current conversation): run the **entire** review in one general-purpose subagent with a clean context (no fork). The prompt contains only: "Use the appsec skill, review per workflows/review.md", the repo path, the scope (`MODE`, `BASE`/`TARGET` or MR number), the business spec if the user gave one, and the read-only constraint. **Do not** send design rationale or "already handled X". Return the report to the user verbatim. No Agent tool: review yourself, re-read the diff with git, do not rely on memory, and write "Self-review in the development session" in Scope |
| Deep review | Diff over 400 changed lines, over 15 files, touches several services, touches 2 or more domains in the domain selection table, touches auth/payment/authorization by adding a route or changing a mechanism, whole-repo/directory scope, or the user says "thorough": the Security track runs [review-deep.md](review-deep.md) (recon, coverage table, parallel hunters, coverage critic, independent verifier, cumulative across rounds), and the Code track runs in parallel per the "Deep mode" section below |
| Domain | Choose from the inventory per [R/hunting-method.md](../references/hunting-method.md) section 6 |
| Mobile | Diff touches Android, iOS, React Native, Flutter code/config: add the Mobile part in Phase 3A |

Tell the user **one line** before reviewing: scope (branch → target, MR, or directory), number of files/lines, deep review yes/no (with an estimate of the number of subagents), domains chosen. Then continue, do not wait.

## Phase 1. Change classification

Per [R/change-classification.md](../references/change-classification.md): run the signal-collecting commands, infer the observed group. If the author declared a group (in the MR description), compare; if not, propose one. Match: write Match with the signals. Mismatch: the violated condition + `file:line` + one sentence. Mobile adds sensitive areas `AndroidManifest.xml`, `Info.plist`, `network_security_config`, `build.gradle`, `Podfile`, `pubspec`, deeplink, WebView, storage, keystore, keychain. Group 9: write "Needs re-review after release".

## Phase 2. Scanner

Probe `gitleaks version`, `trivy --version`, `semgrep --version`. Run what exists, do not install what is missing:

1. `gitleaks detect --redact --log-opts="$BASE..HEAD"` (MR: `$BASE..$HEAD_MR`). `MODE=local` adds the uncommitted part: `gitleaks git --pre-commit --redact` (older versions: `gitleaks protect --redact`), untracked files: `gitleaks dir --redact <file>` (older versions: `gitleaks detect --no-git --source <file> --redact`). No gitleaks: `git log -p "$BASE..HEAD" -G 'BEGIN [A-Z ]*PRIVATE KEY'` and grep for key/token patterns.
2. Diff touches manifest, lockfile, Dockerfile, CI: follow [R/supply-chain-devsecops.md](../references/supply-chain-devsecops.md) section "Checks".
3. `semgrep` scoped to the changed files. Mobile: `p/mobsf`, `p/react-native`. Only a hint to locations.
4. Trail of Bits if installed: [R/tob.md](../references/tob.md).

Every scanner hit goes through the 5 gates like a manual finding.

## Phase 3A. Security

For each hunk: where the input comes from, what it passes through, which sink it reaches; trace callers/callees when needed. Correct fixes and common bypasses: [R/secure-coding-patterns.md](../references/secure-coding-patterns.md). How to hunt (invariant, sibling paths, bad paths, variants), the three kinds of conclusion, and safe verification: [R/hunting-method.md](../references/hunting-method.md) sections 1 to 5.

For each chosen domain, open the file in `R/domains/` and hunt the classes in it too, beyond the table below. Diff touches export/import, search, preview, webhook, or chains several components: add [R/hunting-method.md](../references/hunting-method.md) section 7.

| Group | What to hunt |
|---|---|
| Injection (SQL, NoSQL, SPL, KQL, LDAP, CMD, SSTI) | String concatenation into a query or command; dynamic identifier without allowlist; input into `exec`, `system`, `subprocess`, `eval`, `render_template_string`; ORM raw query |
| Broken Access Control | New route missing authz; state-changing action guarded only by view permission; IDOR without owner/tenant check; authz diverging from the spec; path traversal; open redirect; CSRF on a state-changing GET; removed check |
| Client-declared identity | `user_id`, `tenant_id`, `role`, `org_id` from header, body, query; `/internal/*` guarded by a header with no credential |
| Auth, session | Admin/internal endpoint without auth; JWT `alg:none`, no verification, multiple algorithms; session not regenerated; guessable, non-expiring, or non-single-use reset/invite token; secret comparison not constant-time; missing rate limit on login/OTP |
| Insecure defaults that disable security | `AUTH_DISABLED`, `SKIP_*`, `DEBUG` from ENV; default when ENV is missing; admin port bound to `0.0.0.0`; actuator/debug in prod |
| Sensitive data | Secret added to code/config; response returning PII, hash, user list; PII in cache, cleartext logs |
| SSRF | Client-controlled URL into an HTTP client/webhook without allowlist; redirects enabled; validate then re-resolve |
| XSS | User data or LLM output into HTML/DOM without escaping; `dangerouslySetInnerHTML`, `innerHTML`, `v-html`; URL without scheme validation |
| Deserialization, XML | `pickle`, `yaml.load`, `ObjectInputStream`, `unserialize`, `node-serialize`; XML parser with DOCTYPE not disabled |
| Mass assignment | Model update from body with a blocklist or no filtering |
| Crypto | MD5, SHA1 for token/password; fast hash for password; ECB; static IV, salt; `Math.random` for token |
| DoS | No limit on body, `$in`, page size, JSON/XML depth, import row count; regex on long input; heavy processing before auth; unbounded LLM |
| Race, TOCTOU | Check-then-act outside a transaction (balance, quota, state, unique-by-select); UPDATE without checking `rowsAffected` |
| Audit, logging | Sensitive action not audited; empty actor; audit removed or downgraded; logging PII, credentials, body |
| Misconfig | Verbose errors; CORS `*` with credentials; TLS verify off; Kafka, OpenSearch, Redis without TLS/SASL |
| LLM, agent, MCP | Per the checklist and section C of [R/ai-llm-security.md](../references/ai-llm-security.md) |
| Supply chain | New dependency, registry, `FROM` bypassing the controlled registry (if the organization has one); missing or drifting lockfile; new unverified package; secret via `ARG` |
| CI/CD | `allow_failure: true` / `continue-on-error` on a security job; `*_DISABLED`; `--exit-code 0`; `.trivyignore`, `nosemgrep` added with no reason |
| New or modified CI workflow | Trigger runs untrusted code with secrets, expression injected into shell, unpinned action, cache/artifact mixing trust levels: [R/domains/ci-release.md](../references/domains/ci-release.md) |
| Obvious on added lines | Secret, TODO/FIXME mentioning security, debug that can be enabled in prod, unprotected `/debug` `/admin` `/metrics` `/actuator` endpoint, `eval`/`exec` with dynamic input: [R/hunting-method.md](../references/hunting-method.md) section 7 "The obvious stuff" |
| Removed controls | `-` lines that delete or loosen validation, sanitizer, guard, limit, security test |

Mobile: client-side threat model; review per MASVS (STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE) with [R/mobile/lessons.md](../references/mobile/lessons.md) and the platform file encountered (`android.md`, `ios.md`, `cross-platform.md`). Resilience is INFO or LOW only. Findings add a MASVS field.

### 5 verification gates

A finding must state all six trust-boundary-outcome points ([R/hunting-method.md](../references/hunting-method.md) section 1) and pass all 5 gates. If it does not pass: move it to Notes or drop it. **Exception**: the trace is real but one gate can only be decided by facts outside the repo (whether the proxy strips the header, whether the ingress is open, account IAM, broker ACL, browser defaults): move it to the **Needs validation** section, assign no severity, state the missing fact and a safe way to check. Do not guess that fact either way. The Verification field records concrete evidence (file, command, result). One root cause is one finding.

| # | Gate | How to check |
|---|---|---|
| 1 | Attacker-controlled | Trace the source to the handler. If only admin, operator, DB seed, ENV, backfill script, or CI variable can write it, it is not a finding. Mobile: another app on the same device, MITM, deeplink, WebView content |
| 2 | Reachable | `grep -rn "<func>("` excluding `_test`, `mock`, `testdata`; check wiring in `main`, DI, router. `/internal`, `/admin`, S2S: look for exposure evidence (ingress, compose `ports`, gateway route, network policy); if none, confidence is at most 7 and state the condition, or Needs validation if exposure is the deciding factor. Mobile: release build, not a debug flavor |
| 3 | No defense at another layer | Middleware chain, model constraint, capability token, unique index, ORM bind, auto-escape, allowlist. Check for middleware that fails open or takes identity from a header. Compare **every parallel path** to the same effect; if another layer already blocks it, the missing layer here is hardening, not a finding |
| 4 | Caused by this change | `git show "$BASE:$FILE" \| sed -n 'a,bp'`; `git blame -L a,b HEAD -- $FILE`. Already present at base: Context section |
| 5 | Sink has a security consequence | Who reads or executes the output. Browser `console.log`, non-sensitive errors, a CSV nobody opens are not sinks |

### Severity, CWE, confidence

Look up severity and CWE in [R/severity-cwe.md](../references/severity-cwe.md), deviate by at most one level and state the reason; for a class not in the table, use the anchor scale at the top of that file. Severity does not exceed the proven consequence. Confidence 1 to 10: 9, 10 have a verified exploit path; 8 a clear pattern with a known exploit; 7 needs a specific condition, state the condition.

| Level | It is a finding when | Otherwise |
|---|---|---|
| CRITICAL, HIGH | confidence 7 or higher and an Exploit line (input, sink, payload) | cannot sketch an exploit: downgrade to MEDIUM |
| MEDIUM | confidence 7 or higher | Notes |
| LOW, INFO | confidence 9 or higher | Notes |

The "Not a security finding" section in severity-cwe.md goes only to Notes or is dropped. Known-safe: end of [R/secure-coding-patterns.md](../references/secure-coding-patterns.md); mobile: EncryptedSharedPreferences, Android Keystore, Keychain `WhenUnlockedThisDeviceOnly`, `flutter_secure_storage`, pinning present, `exported=false` or `signature` permission.

## Phase 3B. Code review

Per [R/code-review.md](../references/code-review.md): its 9 groups, severity scale and 4 verification gates.

How to read: read the whole touched file (not only the hunk) and the main callers/callees; Grep for existing helpers before accepting newly hand-written code; compare conventions with the surrounding code in the repo (cite a sample `file:line`), not with taste; for each new abstraction ask "what is the simplest version". An issue that is both a vulnerability and a bug: record it once, in the Security track. Group 10 MR or a new integration: cross-check against the security design notes in the MR description per [R/secure-design.md](../references/secure-design.md).

## Deep mode (parallel)

Enabled when "Deep review" is on. Requires the Agent tool; without it, run sequentially (review-deep.md still applies the coverage table and adversarial challenge, just not in parallel).

1. Run Phase 0 to 2 in the main session; collect the diff, inventory, intent, scanner results.
2. In parallel:
   - **Security**: [review-deep.md](review-deep.md) D0 to D5.
   - **Code** (diff scope; whole repo only when the user asks, on the high-risk modules recon points to): 2 clean-context subagents (general-purpose), each receiving the repo path, scope, intent/spec, file list, read-only constraint, and exactly one part of the work: Logic and business rules (R/code-review.md groups 1 to 4 and 7); Design, performance, convention (groups 5, 6, 8, 9). Return raw findings in the report-template finding format, with evidence.
3. The main session merges, deduplicates (same file, same root cause), runs Phase 3C for code findings, then reports per review-deep.md D6.

## Phase 3C. Adversarial challenge

For each CRITICAL, HIGH, BLOCKER, MAJOR finding: try to refute it. Look for a guard at another layer, a test that already covers it, a spec that allows it, a caller that never passes that value. Refuted: downgrade or move to Notes/Questions, state the reason. Depends on facts outside the repo: move to Needs validation. Not refuted: keep it, write "Challenged: {what was checked}". Also check: is the consequence stated more strongly than observed (a crash turned into RCE, ordinary work turned into shared DoS, a principal's own inherent permission turned into privilege escalation).

Deep review: security findings have already passed the independent verifier at [review-deep.md](review-deep.md) D5. Code BLOCKER/MAJOR findings in deep mode: hand the challenge to a fresh subagent, **different** from the one that found it, which receives only the finding and the relevant code, not the original reasoning. Prompt:

```text
You did not write this finding. Try to refute it from source. Read-only, no network, do not probe real environments.
1. Check every file:line in the evidence. The first source is real input from a less-trusted principal; the endpoint is the stated sink or effect.
2. Rebuild the strongest control on the path: validation, identity, authz, normalization, framework, constraint, test; compare every parallel path to the same effect.
3. Check that severity, CWE, confidence, and fix match only what the evidence establishes; whether the fix enforces the invariant or merely moves the trust.
4. If it still depends on facts outside the repo, propose Needs validation, state that fact and a safe way to check.
Return: Keep / Downgrade (new level) / Needs validation / Drop, with reason and evidence.
```

## Phase 4. Report

Per [R/report-template.md](../references/report-template.md). Scope (base, head) is obtained by command, not filled in by hand. Do not record who ran it, machine, hostname, email, time, or model name.

Verdict: a Security CRITICAL/HIGH or a Code BLOCKER/MAJOR that passed all verification gates gives "Changes required before merge"; only MEDIUM/LOW/MINOR/NIT, Questions or Needs validation remaining gives "Approve with comments"; nothing gives "Approve". A Needs validation item does not by itself block the merge, but must go in the MR description so the system owner can answer. Security remediation follows the "Correct" column of secure-coding-patterns, as the smallest change that enforces the invariant at the final trust decision point, with a regression test; design remediation comes with a simpler version.

Start the report with the finding count; when nothing needs fixing, say so plainly.

Diff touches manifest, Dockerfile, CI: the SCA and DevSecOps section is mandatory.

Multiple MRs: run each unit, then merge. Deduplicate per [R/multi-mr.md](../references/multi-mr.md) "Deduplication across MRs". Block only for issues caused by at least one MR in the set. The overall verdict is the worst level; add a per-MR classification table.

Scanner false positives: record in Notes with tool, rule, location, reason dropped. Suppressions in code must have a reason: `// nosemgrep: <rule-id> -- <reason>`.

The report is returned in the conversation. At the end of the report add **one line** suggesting the appropriate next step (work in progress: "Want me to fix the findings?"; MR: "Want me to post the findings as comments on the MR?"). Do not fix code or post comments on your own. Do not write files on your own, except in deep review: save the report and coverage table outside the repo so the next round is cumulative ([review-deep.md](review-deep.md) D0). If the user asks to save a standard review: write outside the repo, to `~/.claude/appsec-reviews/<repo name>/`.
