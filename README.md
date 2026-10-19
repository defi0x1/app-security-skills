<div align="center">

# appsec

**A code review skill for Claude Code that reviews security and code quality in one pass, and reports only what it can prove.**

One command. Two equal review tracks. Evidence-gated findings. Read-only by design.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.4.0-informational.svg)](CHANGELOG.md)
[![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-D97757.svg)](https://docs.claude.com/en/docs/claude-code/skills)

</div>

---

Most automated reviews fail in one of two ways: they bury you in plausible-sounding noise, or they miss the bug that lives in the *gap between two components*. `appsec` is built around fixing both. Every finding must pass explicit verification gates, facts the repository cannot prove are labelled **Needs validation** instead of guessed, and hunting is driven by *invariants and trust boundaries* instead of keyword lists.

```text
/appsec
```

That is the whole interface. It detects the scope (your branch, an MR/PR, or the whole repo), decides how deep to go, picks the relevant domain playbooks, and returns a structured report with a merge verdict. It does not ask questions unless it cannot continue.

## Why it is different

| | Typical AI review | `appsec` |
|---|---|---|
| **Coverage** | One generic pass | **Two equal tracks**: Security (OWASP/CWE, 10 domain playbooks) and Code (logic, spec, data, performance, design, tests) |
| **Findings** | Anything that sounds risky | Must pass **5 security gates / 4 code gates** with `file:line` evidence, a named trust boundary and a concrete outcome |
| **Unknowns** | Guesses either way | **Needs validation** verdict for facts outside the repo (proxy, IdP, IAM, ingress) with a safe way to check |
| **Hunting** | Pattern matching | **Invariant-based**: compares sibling code paths, checks handoff assumptions between components, looks for variants of a confirmed root cause |
| **Big changes** | Same single pass | **Deep engine**: recon, countable coverage table, parallel hunters, coverage critic, independent verifiers, cumulative across rounds |
| **Noise control** | Reports hardening as bugs | Separates findings from hardening; ships a long "not a security finding" list; blocks a merge only for problems *this change* introduced |
| **Safety** | May run things | **Read-only**: no checkout, no stash, no files in your project, no probing of deployed systems |

## What it reviews

### Security track

- **Core classes**: injection, access control, authentication and session, SSRF, XSS, deserialization, cryptography, DoS, races, secrets, supply chain, CI/CD.
- **Ten domain playbooks**, selected automatically from the trust boundaries your diff touches:

| Domain | Hunts for |
|---|---|
| [`web-auth`](appsec/references/domains/web-auth.md) | HTTP desync and request smuggling, cache poisoning and deception, trusted-Host bugs, CSRF, session fixation, JWT/OAuth/SAML binding, MFA, passkey and account-recovery state transitions, API-key scope, mTLS fail-open |
| [`client-side`](appsec/references/domains/client-side.md) | DOM XSS, DOM clobbering, prototype pollution with a real gadget, `postMessage`, cross-site WebSocket hijacking, CORS, service-worker scope, XS-Leaks, clickjacking |
| [`data-isolation`](appsec/references/domains/data-isolation.md) | Tenant and object isolation, stale ACLs in search, cache and embeddings, export/backup/restore/migration leaks, deletion and revocation lifecycle |
| [`resource-exhaustion`](appsec/references/domains/resource-exhaustion.md) | Input-to-cost paths, ReDoS, decompression bombs, GraphQL depth, retry storms, poison messages, unbounded paid LLM/agent cost |
| [`protocols-messaging`](appsec/references/domains/protocols-messaging.md) | RPC, queues, webhooks, the gap between channel identity (mTLS) and the application principal |
| [`cloud-deployment`](appsec/references/domains/cloud-deployment.md) | Cloud IAM drift, Kubernetes (`hostPath`, `docker.sock`), IMDSv1, final-render checks for Helm and Kustomize |
| [`ci-release`](appsec/references/domains/ci-release.md) | `pull_request_target` "pwn requests", expression injection, unpinned actions, dependency confusion, update-metadata rollback, release integrity |
| [`local-ipc-desktop`](appsec/references/domains/local-ipc-desktop.md) | Peer authentication, deeplink and OAuth handoff, webview bridges, XPC/Binder/D-Bus/named pipes, privileged installers, TOCTOU |
| [`native-memory`](appsec/references/domains/native-memory.md) | Bounds and integer representation, lifetime and type confusion, FFI/ABI contracts, loader search order, JIT, kernel double-fetch |
| [`ai-llm-security`](appsec/references/ai-llm-security.md) | OWASP LLM Top 10 (2025), the "lethal trifecta", RAG/memory/agent/MCP hunting, confused deputy, MCP metadata used as policy |

- **Mobile** ([Android](appsec/references/mobile/android.md), [iOS](appsec/references/mobile/ios.md), [React Native and Flutter](appsec/references/mobile/cross-platform.md)): OWASP MASVS, with ready-to-run greps and a [table of common misconceptions](appsec/references/mobile/lessons.md).
- **Supply chain and DevSecOps**: lockfiles, registry sources, CVE triage by exploitability and reachability, SBOM, CI token hygiene, disabled or downgraded scanners.

### Code track

Correctness and edge cases, business logic against the spec, data migrations and API compatibility, error handling, performance, design and **over-engineering** (every smell comes with a diagnostic question and a simpler alternative), tests, maintainability and conventions, operability. Severity is `BLOCKER / MAJOR / MINOR / NIT` with a hard cap on nits. Uncertain business rules become **Questions for the author**, never accusations. It reads your repo's own `REVIEW.md` / `CLAUDE.md` rules, and reports every instance of a confirmed pattern under a single finding so the author can fix the whole class in one pass.

## How it works

```mermaid
flowchart TD
    A["/appsec"] --> B{"Detect scope"}
    B -->|"no args"| C["Work in progress<br/>commits + uncommitted + new files"]
    B -->|"123 or URL"| D["MR / PR"]
    B -->|"whole repo / dir"| E["Audit"]
    C --> F["Intent, inventory, change classification, scanners"]
    D --> F
    F --> G{"Large or risky?"}
    G -->|"no"| H["Standard review"]
    G -->|"yes, or you say 'thorough'"| I["Deep engine"]
    E --> I
    H --> J["Security track<br/>5 gates"]
    H --> K["Code track<br/>4 gates"]
    I --> L["Recon, coverage table,<br/>parallel hunters, coverage critic"]
    L --> M["Independent verifiers<br/>try to refute each finding"]
    M --> J
    J --> N["Report + merge verdict"]
    K --> N
```

**Standard review** runs both tracks over the diff with a clean-context adversarial challenge for serious findings.

**Deep review** triggers automatically on diffs over 400 lines or 15 files, changes spanning services or multiple risk domains, auth/payment/authorization mechanism changes, whole-repo scope, or when you ask for it. It works like this:

1. **Recon** maps the blast radius, including parallel code paths *outside* the diff.
2. A **coverage table** gives every unit a stable ID, a status and evidence. "I reviewed auth" does not count as coverage.
3. **Parallel hunters** run per domain, each receiving the domain's discipline rules verbatim.
4. A **coverage critic** (and a closing critic) proposes missed coverage, never findings. "Sufficiently covered" is only claimed when both are clean.
5. Every candidate goes to a **fresh independent verifier** whose job is to refute it. It may promote or demote between *Finding*, *Needs validation* and *Rejected*. HIGH and CRITICAL get a second verifier. Unverified candidates are never reported.
6. Results are saved per round, so re-reviewing the same MR after fixes is **cumulative**: fixed findings are recognised, changed source is re-checked, earlier gaps are closed. From round 3 on it reviews only the delta and says so when the approach itself may need rework.

If the session that wrote the code is also the one asked to review it, `appsec` hands the review to a clean-context subagent to avoid author bias.

## Usage

```text
/appsec                        work in progress: branch commits + uncommitted + new files, against the target branch
/appsec 123                    MR/PR 123 (also !123, #123, or a pasted URL)
/appsec 123 125                several MRs
/appsec into develop           set the target branch
/appsec uncommitted only       only what is not committed yet
/appsec thorough               force the deep engine
/appsec whole repo             audit everything (or: /appsec src/billing)
/appsec security only          skip the code track (or: code only)
/appsec 123. Spec: invoices are visible only to their creator or an accountant of the same org
```

Everything after `/appsec` is optional and written in plain language. Before it starts, it prints one line: scope, files and lines, standard or deep, and the domains it chose.

## The report

Every review ends in a fixed structure: verdict, security and code finding counts, change classification, findings, **Needs validation**, **Questions for the author**, pre-existing issues, coverage, and SCA/DevSecOps. See the full [report template](appsec/references/report-template.md).

| Verdict | When |
|---|---|
| **Changes required before merge** | HIGH/CRITICAL security finding or BLOCKER/MAJOR code finding caused by this change |
| **Approve with comments** | Only MEDIUM/LOW/MINOR/NIT items, questions, or *Needs validation* entries |
| **Approve** | Nothing to report |

A security finding carries severity, CWE, confidence, the low-trust principal, the control that should have blocked it, the boundary crossed, the attack path, the verification performed, the smallest fix and a regression test. A *Needs validation* entry carries the trace, the missing fact, a local check and which system owner can answer it, with no severity attached.

The report is about the code only: no usernames, hostnames, timestamps or local paths.

## Design principles

1. **Evidence or silence.** No `file:line`, no finding. Severity is capped at the proven consequence.
2. **Few and right beats many and noisy.** Unsure about business intent: ask. Depends on a fact outside the repo: *Needs validation*. A missing secondary layer when another layer already blocks the issue is hardening, not a finding.
3. **Server-side decisions.** Identity, tenant and role come from verified tokens. Allowlists over blocklists. Fix the root cause.
4. **Simple by default.** Every abstraction must serve a current requirement; every design suggestion ships with a simpler version.
5. **Block only on what this change caused.** Pre-existing issues go to *Context*.
6. **Read-only and traceless.** No code edits, no files, hooks or worktrees in your project, no checkout, no stash. Verification is source-based; no probing of deployed environments, third-party services or real credentials.

## Install

Requires [Claude Code](https://docs.claude.com/en/docs/claude-code). Clone anywhere and link the `appsec` folder into your skills directory:

```bash
git clone https://github.com/defi0x1/app-security-skills.git ~/.claude/app-security-skills
mkdir -p ~/.claude/skills
ln -sfn ~/.claude/app-security-skills/appsec ~/.claude/skills/appsec
```

Or install per project:

```bash
mkdir -p .claude/skills
cp -R /path/to/app-security-skills/appsec .claude/skills/appsec
```

Update with `git pull`. Uninstall with `rm ~/.claude/skills/appsec`.

### Optional tools

`appsec` works without any of these and reports which scanners were missing.

| Tool | Used for |
|---|---|
| [`gitleaks`](https://github.com/gitleaks/gitleaks) | Secret scanning |
| [`trivy`](https://github.com/aquasecurity/trivy) | Dependency CVEs, IaC, images |
| [`semgrep`](https://github.com/semgrep/semgrep) | SAST rules |
| `gh` / `glab` | Fetching GitHub PRs / GitLab MRs (MRs can also be fetched via `git fetch` refs) |
| [Trail of Bits plugins](appsec/references/tob.md) | Optional second opinions (false-positive check, differential review, variant analysis) |

## Repository layout

```text
appsec/
├── SKILL.md                         single entry point and principles
├── workflows/
│   ├── review.md                    standard flow: scope, diff, scanners, security, code, challenge, report
│   └── review-deep.md               deep engine: recon, coverage, hunters, critic, verifiers, rounds
└── references/
    ├── code-review.md               code track: severity, gates, intent sources, over-engineering table
    ├── secure-coding-patterns.md    wrong / bypass / right tables, known-safe patterns
    ├── hunting-method.md            invariants, boundary test, domain selection, feature abuse, chaining
    ├── domains/                     nine domain playbooks (table above)
    ├── severity-cwe.md              severity anchors and ~100 finding types mapped to CWE
    ├── change-classification.md     10 change groups, sensitive-area recipes
    ├── supply-chain-devsecops.md    lockfiles, registries, CVE triage, CI tokens
    ├── secure-design.md             design notes, STRIDE per element, acceptance criteria
    ├── ai-llm-security.md           OWASP LLM Top 10 2025 and agent/MCP hunting
    ├── mobile/                      Android, iOS, cross-platform, lessons
    ├── report-template.md           report format
    ├── multi-mr.md                  multiple MRs and repos
    └── tob.md                       optional Trail of Bits integration
```

## FAQ

**Does it change my code or post comments?** No. It is read-only. At the end it suggests a next step (apply a fix, post MR comments) and does it only if you agree.

**Does it need a GitHub or GitLab remote?** No. It works on a plain local repo. Remotes only matter for fetching MRs/PRs by number.

**Will it flag everything?** Deliberately not. A rule that cannot say when something is *not* a finding does not belong here.

**Which language does it answer in?** Yours. Code and identifiers stay in English.

## Contributing

Issues and pull requests are welcome, especially false negatives, false positives, and new domain playbooks. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Acknowledgements

The invariant-based hunting method, domain playbooks, deep-review engine and the agent/MCP hunting section were adapted from [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill) (MIT), rewritten and condensed for Claude Code. See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Disclaimer

`appsec` is an aid for reviewers, not a replacement for a security audit or penetration test. Findings are produced by a language model and must be reviewed by a person.

## License

[MIT](LICENSE)
