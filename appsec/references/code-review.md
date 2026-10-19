# Code review: logic, business rules, design, over-engineering

The Code track runs alongside Security. Goal: catch defects that break behavior, break business rules, or make the code more expensive than it needs to be. This is not a place to apply personal preference.

## Severity

| Level | Meaning | Example |
|---|---|---|
| BLOCKER | Certainly wrong on the main flow, data loss or corruption, spec violation, breaks compatibility with no plan | Wrong money calculation; migration drops a column in use; API changes a field and breaks old clients; loop that never terminates |
| MAJOR | Wrong in a realistic situation; missing tests for an important flow; significant unnecessary complexity; bad performance at real scale | Empty list/null from an external API not handled; non-idempotent retry; interface + factory + strategy for exactly one implementation; N+1 on a list endpoint |
| MINOR | Has consequences but small, local, easy to fix | Misleading name; small duplication of an existing helper; log missing context on error |
| NIT | Optional, no effect on behavior | Simplify an expression. At most 5 items; drop entirely if a linter/formatter already covers it |

## 4 verification gates (apply to every code review finding)

| # | Gate | Requirement |
|---|---|---|
| 1 | Concrete consequence | Write a **failure scenario** (specific input/state → wrong result) or a **concrete cost** (N more files/layers, X is hard to change, Y is slow at scale Z). If you cannot write one: drop it |
| 2 | Checked in code | Read the relevant callers, callees, tests, config; confirm nothing elsewhere already handles it (validation at a higher layer, DB constraint, covering test). Record evidence as `file:line` |
| 3 | In scope | On a changed line, or a changed line makes old code wrong. Pre-existing defects: the Context section |
| 4 | Not a matter of taste | Based on a consequence, the spec, or a convention **already in the repo** (cite an example `file:line`). "I prefer to write it differently" is not a finding |

**Class-level reporting.** After a finding is confirmed, search the diff and the repo for other instances of the same pattern and list them all under that one finding, so the author fixes the class in a single pass instead of through a fix, re-review, new-instance loop.

Confidence 1 to 10 as in Security. BLOCKER and MAJOR need 7 or higher and a scenario; MINOR 8 or higher; NIT 9 or higher. Below the threshold: move to "Questions for the author" if worth asking, otherwise drop.

## Intent sources (for business-rule review)

Collect before reviewing, in order of trust: the spec the user provides; the repo's own review rules (`REVIEW.md`, `CLAUDE.md`, `CONTRIBUTING`, per-directory variants), whose documented failure modes, skip paths and nit limits are treated as review rules and whose violated or stale convention lines are reportable (they never lower the evidence gates); MR description, issue; docs in the repo (`docs/`, ADR, module README); test names and contents; commit messages; code comments. If there is no source: infer the intent from the code, **state the assumption explicitly** at the top of the code review part, and put uncertain business points under "Questions for the author" instead of concluding they are defects.

## What to hunt

### 1. Logic correctness

- Boundary conditions: empty, null/undefined/nil, 0, negative, very large, duplicates, unicode, time zones, DST, month end, leap year, money rounding (use decimal, not float).
- Off-by-one, `<` vs `<=`, double negation, wrong short-circuit, `==` vs `===`, comparing objects by reference.
- Shadowed variables, mutating shared parameters/objects, mutable default values (Python), closures in loops.
- Async: missing `await`, promises without error handling, `forEach` with async, goroutine leak, context not passed/not cancelled.
- Forgotten `else`/`default` branch; `switch` missing a case when a new enum value is added.
- Logic duplicated in two places that have already drifted apart.

### 2. Business-rule correctness

- Does the code do what the spec says, and does it cover **all** branches of the spec (e.g. "creator or accountant": are both handled).
- State machine: valid transitions, no skipping, terminal state does not return; repeated actions (double submit) give the correct result.
- Money, quantities, limits: rounding, units, currency, order of applying discounts/taxes/fees, upper/lower bounds.
- Time: expiry computed by server or client time, user time zone, effective time.
- Permissions and data scope by role (overlaps with Security: record once, in the Security track if there is a security consequence).
- Behavior with old data: are records created before the change still valid, is a backfill needed.

### 3. Data, migrations, API contracts

- Migration: runs on a large table (lock, duration), has rollback, code and schema deploy order (expand → migrate → contract), default for a new NOT NULL column, index for new queries.
- API/event/schema: renaming/removing a field, changing a type, changing meaning, changing a status code is **breaking**; needs versioning or backward compatibility. Check consumers in the repo.
- Transaction: correct boundaries, no network calls inside a long transaction, rollback on error, no partial writes.
- Cache: invalidation when data changes, key has all dimensions (user, tenant, locale), TTL.

### 4. Reliability, error handling

- Swallowed exceptions, `catch` then return success, log then continue with a corrupted state.
- Errors from external systems: timeout, retry with backoff and **idempotent**, circuit breaker when needed; do not retry 4xx errors.
- Resources: close file/connection/stream, `defer`/`finally`/`using`; pool not exhausted.
- Concurrency: unlocked shared state, race between read and write, double processing of messages (consumers must be idempotent).
- Error messages returned to the client are useful but do not leak internals.

### 5. Performance (only when there is a consequence at real scale)

N+1 queries; queries in a loop; loading a whole table into memory; missing pagination; missing index for a new filter/sort; O(n²) algorithm on a set that can be large; sequential network calls that could be parallel; repeated serialize/deserialize; missing cache for read-heavy data. No premature optimization of code that runs once or of small bounded data sets.

### 6. Design and over-engineering

Root question: **what is the simplest version that meets the current requirement, and how much more complex is this code than that?** Over-engineering is a finding when the extra complexity does not serve the current requirement or a requirement known for certain.

| Over-engineering sign | Ask | Common suggestion |
|---|---|---|
| Interface/abstract class with a single implementation, no need to mock | Is there a second implementation in a concrete plan? | Use a concrete class/function; extract an interface when there is a real need |
| Factory, builder, strategy, plugin registry for 1 to 2 cases | Would one `if`/`switch` or a map be enough? | `switch` or a constant map |
| Pass-through layers: controller → service → manager → repository where the middle layers only call down | What behavior does this layer add? | Remove layers that add no behavior |
| Generics/templates, making configurable something that has never changed | Who will change this value, and when? | A constant; make it configurable when there is a need |
| Parameters, flags, hooks "for later" | Does any caller pass a value other than the default? | Remove unused parameters |
| New framework/library for something the stdlib or an existing helper can do | What does the repo already have that is similar? (Grep) | Use what exists |
| Rewriting something already in the codebase or stdlib (retry, date utils, validation) | Grep for a similar name/behavior | Reuse |
| Event/queue/microservice for a simple synchronous flow | Is there a requirement for latency, separate deploys, load tolerance? | Call the function directly |
| Premature optimization: cache, pool, batch with no measurements | Is there a measurement or SLA? | Simple version, measure first |
| DTO/mapper repeating every field between identical layers | Are the models actually different? | Merge or share |

Conversely, **under-engineering** is also a finding: copy-pasted business logic in several places; long functions doing many things that are hard to test; unnamed business magic numbers; mixing I/O with pure logic so it cannot be tested; boolean parameters that give a function two behaviors.

A design finding must include a **simpler version** (code or a concrete description) and an estimate of what is removed (files, layers, lines). Do not propose large refactors outside the MR scope; record them in Context if worthwhile.

### 7. Tests

- Tests exist for new behavior and for fixed bugs (a reproducing test).
- Tests cover error branches, boundaries, permissions; not only the happy path.
- Tests check real results, not just "does not throw"; do not mock the thing under test; assertions are not too loose.
- Tests do not depend on real time, ordering, or network; not flaky.
- Tests deleted, `skip`ped, or expectations loosened in the MR: ask why.

### 8. Readability, maintainability

Misleading names (a `get` function with side effects); comments that disagree with the code; dead code, commented-out code, TODOs with no owner; unused imports/variables; one file changed for too many concerns; violations of conventions already in the repo (cite an example).

### 9. Operability

Logs with enough context to investigate (ids, no PII); metrics for important new flows; feature flags with a kill switch; new config has a value for every environment; docs/README/environment variables updated when config is added.

## Not a code review finding

Formatting, whitespace, import order (linters handle it); renaming by preference when the current name is not wrong; rewriting the same logic in a different style; suggesting "more modern" patterns/libraries with no consequence; unmeasured micro-optimizations; refactors outside the MR scope.
