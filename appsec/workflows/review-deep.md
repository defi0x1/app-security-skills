# Workflow: Deep review (audit engine)

The engine runs inside [review.md](review.md); it is not a separate command. Review.md invokes it automatically when:

- The diff is large or risky (the "Auto-decide how to run" table of review.md), or the user says "thorough".
- The scope is the **whole repo or a directory** ("review the whole repo", "audit", "pre-pentest sweep", "review the src/billing directory").

Adapted from the process of cloudflare/security-audit-skill (MIT), trimmed for Claude Code. The core idea: **no single review round is enough**. Coverage must be countable (the coverage table), every candidate is challenged by a different subagent than the one that found it, the report says which parts were not covered, and later rounds (re-reviewing an MR after fixes) accumulate on earlier rounds. The standards live in `../references/` (abbreviated `R/`). Hunting method, the "Needs validation" verdict, and safe verification: [R/hunting-method.md](../references/hunting-method.md).

Phases 0 to 2 of review.md (scope, diff, intent, inventory, classification, scanner) have already run. The Code track (Phase 3B) still runs in parallel as in review.md deep mode; this file replaces Phase 3A and 3C with the process below.

## D0. Setup (decide automatically, do not ask)

| Scope | Profile | Differences |
|---|---|---|
| Diff (work in progress, MR) | `standard` | Coverage units come from the surfaces in the diff and from parallel paths outside the diff to the same effect. Repeat hunter batches until the critic is clean, at most 2 batches |
| Whole repo or directory | `standard`; the user says "quick" then `quick`, "thorough"/"pre-pentest" then `deep` | Units come from every surface in scope. Skip Phase 1 (classification) and gate 4 (there is no diff) |
| `quick` | | Coarse units (surface × boundary × class, merging subsystems). Exactly 1 hunter batch + 1 critic; critic suggestions are marked "deferred" |
| `deep` | | Split units by subsystem and lifecycle (create, update, delete, restore, migration). Critic until clean. HIGH/CRITICAL findings go through two independent verifiers |

The profile changes **breadth**, not the evidence standard: do not drop the 5 gates, do not drop independent verifiers, do not drop the "Needs validation" discipline.

**Storage** (automatic, outside the repo, never written into the repo): `~/.claude/appsec-reviews/<repo name>/<key>/round-<N>/`, where the key is `mr-<number>`, `branch-<branch name>`, or `repo` / `dir-<path>`. Save `report.md`, `coverage.md`, `architecture.md`. If the user says not to save, return results in the conversation only.

**Previous rounds**: read every `round-*` of the same key (and `repo` for a diff review, if present). Handle them per the section "Cumulative across rounds".

**Cost**: estimate = 4 recon (diff: 2) + number of hunters + 1 critic per batch + 1 closing critic + 1 to 2 verifiers per candidate. Put it in the announcement line of review.md. Diff: merge units on your own to stay around 10 subagents, do not ask. Whole repo above 15 subagents, or the user sets a cap: ask once (keep, narrow, lower the profile). Never quietly thin out the evidence.

## D1. Recon

Read-only subagents (`Explore` or general-purpose), in parallel, returning facts with `file:line`, writing no files, running no repo code, no network.

**Diff scope** (2 agents):

| Agent | Returns |
|---|---|
| Blast radius | For each surface in the inventory: callers, callees, final sinks, derived copies (cache, index, event, export); **parallel paths outside the diff** to the same effect (old route, batch, job, admin, import, retry) and the controls on each path |
| Principals, controls, deploy visibility | Low-trust principals and the actions they may perform around the changed area; authentication, per-resource authz, tenant/owner scope; escalation, revocation, fallback; which controls are visible in source and which depend on deploy facts that are not visible; whether existing tests check the boundary |

**Whole repo / directory** (4 agents):

| Agent | Returns |
|---|---|
| 1a. Product, stack | Product type, users, operators, sensitive actions; languages, frameworks, build, runtime, visible deploy model; entry points and subsystem boundaries; build/test commands that can run offline (not run); similar software for calibration (if grounded) |
| 1b. Principals, permissions, controls | Each low-trust principal and its permitted actions; authentication at each surface; per-resource authz, tenant/owner scope; process, workload, CI, plugin, model/tool, IPC permissions; escalation, confirmation, revocation, recovery, fallback; visible controls and controls that depend on deploy |
| 1c. Entry surfaces, copies, sinks | Every place external input enters: HTTP/browser, RPC/message, file/archive, CLI/ENV/config, plugin/dependency/CI, cloud events, model context/tool parameters, mobile/deeplink/webview, IPC. For each surface: main transformations, copies, sensitive sinks, limits, parallel paths |
| 1d. Test, build, deploy | Small tests/fixtures that can check a boundary with fake data; commands that would download dependencies, publish, or call paid APIs (forbidden); deploy controls that source cannot establish; final configuration per environment when the repo picks it deterministically |

Synthesize an **Architecture summary** (diff: ≤ 400 words; repo: ≤ 1000 words): principals and protected assets; stack and deploy path; entry surfaces and key source→sink paths; trust boundaries and the strongest control on each boundary; starting points by path; gaps from previous rounds; the domains selected and the boundary that led to choosing them ([R/hunting-method.md](../references/hunting-method.md) section 6).

## D2. Coverage table

One **unit** = surface × trust boundary × subsystem × attack class (`deep`: plus lifecycle). Attack classes come from the Phase 3A table of review.md, section 7 of hunting-method.md, and the classes in the selected domain files. Diff scope: a surface is a surface in the diff **and** the parallel paths that recon found. A visible surface outside scope is marked "out of scope", **not** "covered".

IDs are stable across rounds, built from source references, not from line numbers or batch numbers: `<surface path>#<route/handler> :: <boundary control> :: <subsystem> :: <standard file>#<class name>`. Example: `src/api/invoice.ts#PATCH /invoices/:id :: src/authz.ts#requireOrg :: packages/api :: secure-coding-patterns#Broken Access Control`.

`coverage.md`:

```markdown
| ID | Status | Batch | Owner | Files read | Checks done | Candidate | Still open |
|---|---|---|---|---|---|---|---|
```

| Status | Meaning | Required |
|---|---|---|
| not started | Not yet assigned | No evidence |
| in progress | Assigned to a hunter | Has an owner |
| covered | Read fully, no candidate | Files read + checks done non-empty |
| has candidate | Has a candidate with an ID attached | Files + checks + candidate ID |
| blocked | Partly read, missing a deciding fact | Files + checks + blocking reason |
| deferred / out of scope / not applicable | Not assigned in this round | Reason |

The coverage table **is** the coverage claim. A summary, an agent count, or the sentence "auth was reviewed" is not coverage evidence.

## D3. Hunter batch

Assign "not started" units to general-purpose subagents, in parallel. One hunter may take several related units in the same subsystem; do not merge unrelated boundaries. Priority when cuts are needed: (1) unauthenticated or lowest-trust surfaces; (2) boundaries protecting the most valuable assets (credentials, cross-tenant data, code execution, release rights); (3) newly added lines and gaps from previous rounds; (4) classes that often yield findings in this kind of code. Units that do not get a turn: "deferred" with a reason.

Hunter prompt, in order:

```text
You are a security hunter. Goal: find source-grounded bugs that violate security invariants in the assigned units, and return exactly the result format at the end.
Repo: {path}. Scope: {MODE/BASE/head or MR, or directory}. Read-only: do not edit files, no git add/commit/stash/checkout, no network, install nothing, do not run repo code unless it meets the conditions in the Safety section.

## Architecture summary
{verbatim}

## Assigned units
{ID, subsystem, boundary, starting point}

## Attack classes
{copy VERBATIM the Discipline section + the selected class rows + Common moves + Verification before reporting from each domain file; the matching row of the Phase 3A table. Do not send only the file name.}
Exclusions: {classes/blocks not selected + reason}. Units of other hunters, do not duplicate: {ID}. Findings from previous rounds still standing, do not report again: {ID, title, root cause}.

## Method
{copy verbatim sections 1 to 5 of hunting-method.md; for diff scope add gate 4 of review.md}

## Result
Each assigned unit appears exactly once:
- ID | status: covered / has candidate / blocked | files read | checks done (invariant, method: source/local, result) | candidate ID | still open
Candidate (each):
- ID stable by root cause (no line number, batch, or severity in it)
- proposal: Finding / Needs validation
- the six boundary-outcome points; trace `file:line` from entrypoint through propagation to sink; strongest control seen
- Finding: conditions, caused by this change or pre-existing, bounded local reproduction (if run: command, input, result), proposed severity, CWE, smallest fix + regression test
- Needs validation: the missing fact, local check plan and/or the system owner to observe; do NOT assign a severity
Hardening: {one line per item, file:line}
Not covered: {surfaces/boundaries discovered outside the assigned units, with starting point and reason}
If nothing passes the gates, return an empty candidate list.
```

The main session updates **only** the assigned units. Broken results, missing units, or conclusions without evidence: return the unit to "not started". Merge candidates by ID, then by root cause: one root cause exposed on several paths is one candidate with the strongest trace; missing controls that are independent of each other are separate candidates. "Not covered" becomes units for the next batch.

## D4. Coverage critic

Right after each batch, call a fresh read-only subagent that receives the architecture summary, the whole `coverage.md`, the candidate list and statuses, and the gaps from previous rounds. It **proposes coverage, not findings**. It looks for: unmapped surfaces, unchecked parallel paths, missing lifecycles, selected domain classes with no unit, exclusions with no reason, closed units missing files/checks, Needs validation items or source changes from the previous round that no unit addresses. It returns:

```text
Missing units: {surface, boundary, subsystem, class, starting point, source-grounded reason}
Reassign: {ID that was closed but not really closed, reason}
Stop: yes/no
```

The main session drops out-of-scope suggestions, builds IDs, and deduplicates. Reassigned units: move the old result to the history column, assign a **new** hunter, do not copy old evidence. `standard`/`deep`: when the critic has nothing left and no "not started" units remain, call **one more closing critic**, a different one; only if it is also clean may you say "sufficiently covered within scope". `quick`: exactly 1 batch + 1 critic, all suggestions marked "deferred". Maximum batches or budget exhausted: untouched units are marked "deferred", and the report says so explicitly.

## D5. Independent verification

Replaces Phase 3C of review.md for the Security track. Assign each unique candidate to a **fresh verifier** subagent, not the hunter that found it. The verifier receives: the candidate, the checks done for the related unit, the part of the architecture needed to understand the path, the "Verification before reporting" section of the related domain, sections 3 and 5 of hunting-method.md, the 5 verification gates and the severity table of review.md Phase 3A. It does not receive another verifier's conclusion.

```text
You did not write this candidate. Try to refute it from source and bounded local evidence. Read-only, no network, do not probe real environments.
1. Check every file:line in the trace and evidence. The first entry is a real entrypoint for a low-trust principal; the last entry is the stated sink/boundary effect.
2. Reconstruct the strongest control on the path: validation, identity, authz, normalization, lifecycle, framework, isolation; compare every parallel path. If the architecture summary has similar software, note whether it has the same pattern (for calibration, not for refutation).
3. Proposed Finding: reproduce the smallest result yourself if possible; check the input, interface, conditions, and the fake principal/resource affected. Do not infer a stronger result. Diff scope: re-check whether this change caused it or it was pre-existing.
4. Check that severity, CWE, confidence, and fix match only what the evidence establishes; whether the fix enforces the invariant or merely moves the trust point.
5. Proposed Needs validation: whether the blocker really lies outside source/local. If source can refute it, Reject. If it still decides the outcome, keep it, write a local check plan and the specific system owner to observe, non-destructive.
6. Keep the ID if the root cause is the same.
Return: Decision: Finding / Needs validation / Reject, with the corrected record in the finding template of report-template (Finding), or trace + missing fact + plan (Needs validation), or reason (Reject).
```

A verifier raises Needs validation to Finding only when it independently establishes the full path and outcome; it lowers a Finding to Needs validation when a deploy/runtime fact is still unknown. `deep`: a HIGH/CRITICAL Finding goes through a second verifier that receives only the final record; if that verifier changes it significantly (raises the verdict, changes the root cause, trace, outcome, severity), the replacement goes through yet another fresh verifier before being applied. Budget exhausted: verify in ID order as far as possible; unverified candidates do **not** enter the report as findings, the report states "verification incomplete" and lists them.

Code BLOCKER/MAJOR findings from the Code track are still challenged per Phase 3C of review.md.

Every final verdict is written to the related unit in `coverage.md`.

## D6. Report

Follow [R/report-template.md](../references/report-template.md), including the "Coverage" section. Write it only after D5 is done for every retained item. Prose must not change a verified verdict, severity, blocker, or consequence. If there are no findings, say so and state the remaining coverage limits; do not invent a LOW. Do not include instructions for probing real environments. End in one of two states: a complete report; or an explicit "incomplete" with the reason and gaps. Do not stop midway.

## Cumulative across rounds

One round typically finds only about half of the vulnerabilities that multiple rounds together find. When re-reviewing the same MR/branch after fixes, or re-running the whole repo:

1. Compare current source against each old finding and unit (`git diff <old head>..<new head> -- <relevant paths>`). The old head does not prove the path is unchanged.
2. Old finding, source unchanged: put it into the candidate set (keep the ID), remove it from the hunt list, and it **still** goes through the verifier in D5. Source changed (usually fixed): create a "re-check" unit; if the verifier confirms the fix, record it in Context as "fixed in round N".
3. Needs validation items, deferred, blocked, old out of scope, and units whose source changed: become work for this round. They never substitute for current coverage.
4. Units already covered, source unchanged: still shown in the table, may be lowered in priority; the critic decides whether another round is needed. An old Reject blocks only that exact claim, and only when the evidence is unchanged.
5. The previous round was `quick` or narrow scope: it contributes only the evidence and gaps it recorded, and never implies "the rest is fine".

6. Convergence (round 3 and later, both tracks): review only the delta since the last round, and do not repeat resolved or rejected findings. If new findings keep arriving, say that the approach may need rework instead of listing more nits.

No previous round: say so in the Coverage section. Never imply that one round exhausted everything.
