# Domain: resource exhaustion and availability

Use when a request, message, file, tenant state, or agent job can consume CPU, memory, disk, connections, workers, paid APIs, queue capacity, or deadlock/crash a shared service. This section separates availability vulnerabilities from ordinary performance issues (performance: [code-review.md](../code-review.md) group 5). Baseline rules: [secure-coding-patterns.md](../secure-coding-patterns.md) section 10. **Never** verify by generating load against a real or shared service.

## Discipline

- Three things are required: a **path from input to cost**, a **missing effective limit**, and **impact on others, a shared service, safety functions, or the operator's costs**. Self-limiting within the requester's own process is not a service vulnerability.
- A missing rate limit is not enough. Check body/message/file limits, concurrency, queues, deadlines, DB constraints, front gateways, per-tenant quotas before calling something "unbounded".
- State the attacker's cost, the work the service must do, whether it persists, the scope, and recoverability. A small input causing a superlinear or persistent effect is very different from sustained load.
- Upstream limits, topology, autoscaling, paid quotas, or recovery behavior outside the repo: Needs validation.

## Computational amplification

| Layer | What to hunt |
|---|---|
| Superlinear parse, match, eval | Backtracking regex (ReDoS), deeply nested parse, recursive validation, template expansion, graph traversal, adversarial hash/sort. Identify the accepted depth/element count and the complexity |
| Decompression, representation bloat | Zip bomb, XML entities, YAML aliases, images/fonts/documents, protocol compression tables that expand beyond the checked size. Check limits **after each decompression** and across every parser layer |
| Query/downstream amplification | A small request causes a wide scan, pathological join, fan-out, unbounded sort/aggregate, or many downstream calls through unbounded query depth, filter count, pagination, expanded fields (GraphQL depth/complexity) |

## Resource accumulation

| Layer | What to hunt |
|---|---|
| Unbounded buffers and cardinality | Bodies, slow-read/slow-send connections (slowloris), out-of-order streams, HTTP/2 stream resets or CONTINUATION frames, uploads, sessions, unique cache keys, metric labels, log fields, subscriptions, pending jobs accumulate with no per-item **and** total limit. Is there cleanup on disconnect, timeout, cancel, half-finished parse |
| Handle and temp-resource leaks | Failed or aborted requests skip cleanup of sockets, files, DB cursors, timers, subprocesses, temp files |
| Work continuing after cancel | Client timeout, disconnect, cancelled job, or failed authz returns, but DB/model/network/worker work keeps running. Trace how cancellation and deadlines propagate through every layer |

## Quota and scheduling

| Layer | What to hunt |
|---|---|
| Heavy work before authentication | Parse, key lookup, crypto, decompression, external calls run before auth and before the earliest size/rate gate |
| Quota counted in the wrong scope | Counting per IP, route, tenant, key prefix, attacker-influenced task ID lets an attacker escape their budget or burn someone else's. Check overflow, distributed races, retries, reconnects, account switching |
| Worker, pool, priority starvation | An attacker's job holds locks, workers, DB pool slots, event-loop turns, or scheduling priority that others need; holds a slot past the deadline |
| Agent/LLM cost | One request pushes many paid model/tool calls with no per-request budget, no cancellation, not idempotent. See also [ai-llm-security.md](../ai-llm-security.md) |

## Failure and recovery

| Layer | What to hunt |
|---|---|
| Fatal errors, deadlock | Untrusted input reaches `panic`, abort, fatal assert, uncaught exception, `exit`, lock cycle, infinite loop in a shared process. Determine the supervisor's scope: one worker or the whole service. An auto-restarting worker reduces impact but does not remove the bug |
| Retry storm | Timeouts, dependency errors, half-processed messages, failing health checks trigger synchronized retries with no jitter, no cap, no circuit breaker, no dedup |
| Poison message, head-of-line blocking | One corrupt record repeats its failure at the head of a queue, partition, startup scan, migration, recovery loop. Is there skip/quarantine, an offset; does another tenant share the blocked unit |
| Unsafe recovery | Restart, restore, fallback, cleanup rebuilds unbounded state, drops the active quota, or reloads the very input that caused the failure |

## Common moves

- Build an input → resource table: the earliest accepted size/element count, work before auth, fan-out, persistence, shared pools, who limits and who cleans up, recovery.
- Compare total limits against per-object limits: ten thousand valid one-byte items can evade a per-message limit yet exhaust per-tenant or per-process state.
- Verify only through complexity analysis, small fixtures with CPU/memory/time limits, mocked paid calls.

## Verification before reporting

1. State the untrusted input, the requester's effort, the amplification or held resource, the scope of shared impact, and recovery. A missing limit with no concrete shared impact: hardening.
2. Confirm there is no limit upstream, in the parser, queue, tenant, or framework visible in the repo.
3. Severity rises with: low effort, no auth, cross-tenant, persistent, poor recovery ([severity-cwe.md](../severity-cwe.md)).
