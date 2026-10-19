# Domain: data isolation and lifecycle

Use when there is multi-tenant data or ACLs, derived copies (search, index, cache, analytics, embeddings), links to objects, export/import/backup/restore, schema migration, or commitments about deletion, revocation, or retention. This file follows **one record** through every copy and state transition. Endpoint authz: [secure-coding-patterns.md](../secure-coding-patterns.md) section 2. Cloud storage policy: [cloud-deployment.md](cloud-deployment.md).

## Discipline

- A tenant/owner column on a record is **not** isolation. Find the query, key, path, policy, or row-level control that enforces it for every read and write path.
- Trace derived copies: source data that was filtered can become dangerous in search, cache, analytics, export, preview, logs, replicas, and backups with different ACLs and lifetimes.
- Deletion and revocation are lifecycle contracts: check the current copy, history, cache, index, export, restore, and queues.
- A generic privacy preference or retention concern is **not automatically** a vulnerability. It needs a clear data-access boundary or a deletion/revocation commitment, plus an unauthorized reader or operation.
- Storage policy outside the repo, retention, CDN, replica lag, backup access that is not visible: Needs validation.

## Tenant and object isolation

| Layer | What to hunt |
|---|---|
| Missing tenant/owner constraint | Read, update, delete, list, count, or bulk operations identify an object without binding the authenticated tenant/owner, or take identity from the body. Compare direct lookup, nested relations, background jobs, admin, import, legacy |
| Key/namespace collision | Cache keys, object paths, unique indexes, search document IDs, temp files, or dedup keys that lack a tenant or environment: two principals overwrite or obtain the same key |
| Policy and query mismatch | RLS, ORM default scope, authz filters, and raw/bypass clients use different predicates. Check joins, aggregates, aliases, views, transactions, `unscoped`/service clients, and error branches when context is missing |
| Overly broad blob and signed URL | Object keys, attachment IDs, version IDs, share links, or signed URLs allow an operation/namespace broader than the issuer's permission, or remain valid after the ACL changes. Bind operation, object/version, audience, expiry, tenant |

## Derived data and leakage

| Layer | What to hunt |
|---|---|
| Search/cache/index ACL drift | The source record's ACL or lifecycle changes but copies (search, cache, embeddings, thumbnails, RSS, previews) are not invalidated. The filter must apply at **retrieval time**, not only at ingest |
| Analytics, logs, traces, diagnostics | Private content or credentials enter a system with a wider readership, longer retention, or mixed tenants. Needs the actual data type and readers; a field name or public ID is not enough |
| Oracle via counts and aggregates | Counts, filters, sorting, errors, unique constraints, timing, messages, or existence checks reveal the state of a protected object/account. Needs a concrete secret predicate and an observable difference |

## Export, backup, restore, migration

| Layer | What to hunt |
|---|---|
| Export/backup scope expansion | Exports, snapshots, portability bundles, or reports include other tenants, fields the viewer may not see, soft-deleted data, secrets, or history beyond permission. Check authz **per item after selection** and the permission to download the final file |
| Import/restore privilege expansion | Skips owner, schema, ACL, unique, or validation checks; overwrites existing resources; creates records in a tenant without write permission. Treat archive contents as untrusted and authorize the resulting operation |
| Migration: defaults and ownership | Old records lack tenant/ACL/lifecycle fields; old and new IDs collide; a partial rollout makes old and new readers apply different defaults. Check backfill, dual read/write, rollback, re-run migrations |
| Backup and replica boundary drift | Encryption keys, storage accounts, cross-region replicas, restore environments, or support snapshots have wider scope than the source data. The repo can only confirm policy in the repo; the rest is Needs validation |

## Deletion, revocation, lifecycle

| Layer | What to hunt |
|---|---|
| Soft-delete bypass | Direct lookup, search, relation traversal, object links, background jobs, or restore drop the lifecycle predicate and return or operate on deleted/revoked records. Can a soft-deleted identifier be re-registered before all references disappear |
| Stale authz, stale copies | Removing a member, changing an ACL, withdrawing consent, revoking a secret, or demoting a role does not invalidate sessions, caches, subscriptions, jobs, or materialized data that still grant access |
| Queues outlive deletion commitments | Deletion completes in primary storage but jobs, retries, exports, analytics, or generated artifacts recreate or retain the data. Needs idempotent deletion and propagating tombstones |
| Restore reintroduces bad state | Backup, undo, undelete, or replica recovery brings back data, credentials, members, or permissions that current policy does not allow. Re-authorize the restored state; reapply lifecycle changes made after the snapshot |

## Common moves

- Pick one protected record and draw its paths: original write, query, cache, index, event, export, backup, delete, restore. Mark the principal and tenant on every edge.
- Compare two fake tenants through the same service method, then repeat after an ACL change, deletion, account change, and restore.
- Start from raw-client bypasses, background jobs, migrations, global uniques, and cache keys: the places that often lack the per-request identity an interactive endpoint has.

## Verification before reporting

1. State the attacker, the protected data/state, the affected owner/tenant, the substitute copy or operation, and the exposed data or unauthorized mutation.
2. Quote both the canonical source policy and the path that bypasses or diverges from it; confirm that no other layer enforces the same tenant/lifecycle condition.
3. If an external cache, object storage, replica, analytics, backup, or retention outside the repo determines the outcome: Needs validation, and state what the system owner needs to observe.
