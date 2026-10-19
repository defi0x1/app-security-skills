# Report template

Applies to every review mode. Style: short, assertive sentences; no emoji; no narration of the process; respond in the user's language (default English); keep code and identifiers in English. Every section appears; an empty section says "None". Do not add sections beyond the template.

**Do not include** information about who ran the review, the machine, hostname, email, time, model name, absolute local paths, or internal remote URLs. The report is about the code only.

Take Scope values from commands; do not fill them in by hand:

```bash
echo "head=$(git rev-parse --short HEAD 2>/dev/null || echo 'unknown')"
echo "base=$(git rev-parse --short "$BASE" 2>/dev/null || echo 'not applicable')"
for t in gitleaks trivy semgrep; do command -v $t >/dev/null && echo "have $t" || echo "missing $t"; done
```

## Verdict by mode

| Mode | Verdict values |
|---|---|
| Work in progress, MR, multiple MRs | Changes required before merge / Approve with comments / Approve |
| Whole repo, directory | Remediation required / Approve with comments / Approve |

Whole repo, directory: the Change classification section says "Not applicable (no diff)".

## Findings

- One root cause is one finding. The same misconfiguration behind several endpoints is one finding with all locations listed.
- Dependency CVEs: one finding per package, listing the CVEs, severity by the highest CVE, CWE-1395.
- Locations always use `file:line` relative to the repo root.

## Template

```markdown
# Review report

| Field | Value |
|---|---|
| Scope | {Work in progress: feature/x → develop, base abc1234, head def5678, with/without uncommitted changes} {MR !123: …; one line per MR} {Whole repo/directory: path, HEAD commit} |
| Review tracks | Security + Code / Security only / Code only. Deep review: {yes, round N / no}. Mobile: {yes/no}. {Self-review in the development session, if applicable} |
| Intent | {Source: spec / MR description / commits / tests. No source: "Assumption: {…}"} |
| Scanners | Ran: {…}. Missing: {…}. |

## Verdict

{per the Verdict by mode table}
{One sentence of reasoning, at most 30 words.}
{Group 9: Needs re-review after release.}

| Security | CRITICAL | HIGH | MEDIUM | LOW | INFO | Needs validation |
|---|---:|---:|---:|---:|---:|---:|
| Count | 0 | 0 | 0 | 0 | 0 | 0 |

| Code | BLOCKER | MAJOR | MINOR | NIT | Questions |
|---|---:|---:|---:|---:|---:|
| Count | 0 | 0 | 0 | 0 | 0 |

## Change classification

| Field | Value |
|---|---|
| Declared | {Group N: name} / Unclassified |
| Observed from diff | {Group N: name} |
| Verdict | Match. / Mismatch. Condition "{quote}" violated at `{file}:{line}`: {one sentence}. |
| Signals | {N} files. Sensitive areas: {…/none}. New routes: {yes/no}. Dependencies: {yes/no}. CI, Docker, k8s: {yes/no}. |

{Multiple MRs: add a table}
| MR | Declared | Observed | Match | Verdict | C/H/M/L |
|---|---|---|---|---|---|

## Security findings

| ID | Issue | Severity | CWE | Confidence | Location |
|---|---|---|---|---:|---|

### SEC-001: {Title}

| Field | Value |
|---|---|
| Severity | HIGH |
| CWE | CWE-89 SQL Injection |
| MASVS | {mobile only} |
| Confidence | 9/10 |
| Location | `{file}:{line}`, one line per location when there are several {multiple MRs: add !IID} |

**Description.** {What the vulnerability is. Why it is dangerous in this context.}

**Exploit.** {Required for HIGH, CRITICAL. Input, sink, sample payload.}

**Trust boundary.** Low-trust principal: {…}. Control that should block it: `{file}:{line}`. Boundary crossed: {…}. Affected: {…}.

**Verification.** Source: {…}. Reachable via: {…}. Existing defenses: {none / present, why insufficient}. Parallel paths compared: {…}. Caused by this change: {yes, new lines / yes, modified lines}. {Ran local check: command, input, result / Source only.}

**Evidence.**
\```{lang}
{at most 10 lines}
\```

**Impact.** {…}

**Remediation.** {The invariant to enforce; the smallest change at the final trust decision point.}
\```{lang}
{fixed code}
\```
{Regression test: the input/scenario that must be rejected.}

## Code findings

| ID | Category | Issue | Severity | Confidence | Location |
|---|---|---|---|---:|---|

### CODE-001: {Title}

| Field | Value |
|---|---|
| Category | Logic / Business rules / Data-API / Reliability / Performance / Design / Tests / Maintainability / Operability |
| Severity | MAJOR |
| Confidence | 8/10 |
| Location | `{file}:{line}` |

**Issue.** {One to three sentences.}

**Scenario.** {Concrete input/state → wrong result. Design: concrete cost (number of layers, files, redundant lines; what becomes hard to change).}

**Verification.** {Which callers/tests/config were read; why nothing else handles it. Adversarial challenge: {how refutation was attempted}.}

**Suggestion.**
\```{lang}
{fixed code, or the simpler version}
\```

## Needs validation

{The trace is real but a deciding fact lies outside the repo. No severity, not counted as a finding. / None.}

| ID | Issue | Boundary | Trace | Missing fact | Local check | System owner to check |
|---|---|---|---|---|---|---|
| NV-001 | {…} | {e.g. tenant, authentication} | `{file}:{line}` → `{file}:{line}` | {e.g. does the ingress strip a client-sent `X-User-Id`} | {test/fixture/rendered manifest, or "not applicable"} | {which config/policy/route to inspect; do not send probe requests to a real environment} |

## Questions for the author

{Business or design points without enough basis for a conclusion, each with `file:line` and the consequence if the answer is "yes". / None.}

## Context: pre-existing issues, not caused by this change

{`file:line`, issue type, severity, one line per item. / None.}

## Notes (not counted as findings)

{One line per item, including scanner false positives with the reason they were dismissed, and hardening (a missing secondary layer of defense when another layer already blocks the issue) with `file:line`. Whole repo: add good patterns worth keeping. / None.}

## Coverage

{Deep review only; standard review says "Not applicable".}

| Field | Value |
|---|---|
| Profile | {quick / standard / deep}. {quick, narrow scope, or stopped early: "Partial coverage"} |
| Domains selected | {domain files + the boundary that triggered each} |
| Units | {N covered · N with candidates · N blocked · N deferred · N out of scope} |
| Subagents | {used / planned} |
| Final critic | {clean / open proposals: …} |
| Previous rounds | {none / round-N: findings carried over, fixed, re-checked because source changed, open work} |
| Incomplete | {no / reason, unverified candidates: IDs} |
| Saved | `~/.claude/appsec-reviews/<repo>/<key>/round-<N>/` |

Never imply that a round exhausted the repo.

## SCA and DevSecOps

{Repo audit, or a diff that touches manifests, Dockerfiles, or CI. Otherwise: Not applicable.}

| Item | Result |
|---|---|
| Lockfile | {complete / missing: ecosystem} |
| Dependency CVEs | {N HIGH, N CRITICAL / none} |
| Dependency sources | {through a controlled registry / pulled from public at `file:line` / no policy found} |
| Pipeline | {security jobs per MR / schedule only / none}. {allow_failure, disabled scanners, suppressions without a reason / none} |

| Package | Version | CVE | Severity | Fixed version | Direct/Transitive |
|---|---|---|---|---|---|
```
