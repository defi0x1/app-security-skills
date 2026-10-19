# Domain: RPC, protocol, messaging

Use when the code has gRPC, GraphQL transport, Protobuf/Thrift/Cap'n Proto, custom binary protocols, streaming RPC, webhooks, Kafka/RabbitMQ/NATS/SQS/pub-sub, or an event bus. This section covers peer identity, message interpretation, routing, replay, ordering, and delivery semantics. Parser memory safety: [native-memory.md](native-memory.md). HTTP framing: [web-auth.md](web-auth.md). Availability: [resource-exhaustion.md](resource-exhaustion.md).

## Discipline

- **"Internal" is not authentication.** State the peer identity at each hop and how it becomes the application principal used for authz.
- Schema validation proves **shape**, not origin, resource permission, ordering, or safe values. Follow the decoded field to policy and side effects.
- Broker guarantees and application guarantees are different. Record retry, ordering, ack, dedup, and transaction behavior before assessing a state change.
- A parser mismatch needs two consumers, two schema versions, or two concrete wire representations and **one differing value with a security consequence**.
- Broker ACLs, mesh identity, topic bindings, or compatibility behavior outside the repo: Needs validation.

## Framing, schema, interpretation

| Layer | What to hunt |
|---|---|
| Boundary and normalization mismatch | Two components interpret length, compression, duplicate fields, unknown fields, encoding, number width, normalization, or envelope/body precedence differently. Compare generated and hand-written parsers, gateways, language bindings, version converters. After decoding, which principal/resource/operation differs |
| Union, enum, default | Unknown variants, missing discriminator, zero values, default permissions, compatibility mappings to code that assumes validation already happened. Does dispatch cover every case, what does the default branch do, how does an old consumer interpret a new field |
| Envelope and payload identity mismatch | Authz uses routing/envelope metadata while the handler uses the tenant/account/subject/object/sender in the body. Determine which source is canonical and that the client cannot overwrite it |

## RPC identity and authz

| Layer | What to hunt |
|---|---|
| Interceptor does not cover everything | An authn/authz interceptor applies to unary calls but not to streams, reflection, health, gateway transcode paths (grpc-gateway), compatibility services, or individual messages within a stream. Compare every registration and every route to the same operation |
| Peer identity differs from application principal | mTLS, workload identity, bearer metadata, or a credential broker authenticates the **channel**, but a caller-chosen field decides the user/tenant. The channel and the declared principal must be bound by deterministic policy |
| Per-item and per-stream authz | A stream, subscription, or batch is authorized once, and later items point to a different resource or continue after a role, membership, or token is revoked |
| Callback/reply correlation | Predictable, reused, or cross-tenant correlation IDs let a response, webhook, cancel, or ack satisfy another caller's pending operation |

## GraphQL

| Layer | What to hunt |
|---|---|
| Resolver-level authz | Authz only on top-level queries/mutations; nested fields, node/`id` lookups, connections, subscriptions, and union/interface resolvers return objects without an owner/tenant check |
| Batching and aliases | Aliases or array batching run many login/OTP/coupon/lookup attempts in one request and bypass per-request rate limits; query depth/complexity limits: see [resource-exhaustion.md](resource-exhaustion.md) |
| Transport | Mutations accepted over GET or `text/plain`/form content types (CSRF); introspection or field suggestions only as hardening unless they expose a private schema |

## Broker and queue isolation

| Layer | What to hunt |
|---|---|
| Topic, routing key, subscription scope | A publisher/subscriber can choose another tenant's topic, wildcard, consumer group, partition, reply queue, or DLQ. A tenant name in the payload is not isolation |
| DLQ, retry, diagnostics leakage | Messages in a DLQ, error topic, tracing, or operator screens contain secrets or cross-tenant payloads that a low-privilege consumer can read |
| Untrusted producer acting as control plane | A body that self-declares as an admin event, provider callback, replication record, or migration command without independently authenticating the producer and event type. Webhook: verify signature, timestamp, source/account/audience before privileged processing |

## Replay, ordering, transactions

| Layer | What to hunt |
|---|---|
| Duplicate delivery, missing idempotency | Retry/redelivery repeats side effects because there is no dedup, dedup happens after mutation, or the dedup key collides across tenants/operations |
| Stale and out-of-order messages | Old state, a revoked member, a cancelled job, or a pre-step-up authz arrives late and overwrites newer state. Check sequence/version, tombstones, partition changes, replay |
| Ack/commit ordering | Ack before durable commit loses security-relevant work; commit before ack may cause duplicate mutation. Check outbox/inbox and recovery |
| Partial state transition across consumers | Several consumers perform one authz/business state transition; retries and partial failures leave only part of it committed |

## Common moves

- Draw producer → broker/transport → gateway → consumer → storage for each message family. For every hop record the authenticated peer, the canonical tenant/resource field, validation, and side effects.
- Run the same small fixture through every schema version or binding in the repo. Try duplicate, missing, unknown, boundary, replayed, and reordered messages, without generating load.
- Compare the normal path, retry, DLQ, replay, migration, reflection, stream, and gateway transcode. Security policy must survive every transport.

## Verification before reporting

1. State the actual producer/peer, the message received, the authenticated channel identity, the affected principal/resource, and the unauthorized mutation or data exposure.
2. Parser mismatch: quote both parsers/consumers and the differing decoded values. If one side safely rejects, do not confirm.
3. Replay/ordering: determine the actual delivery guarantee; reproduce with a bounded in-memory transport.
4. Authz and isolation: check every interceptor, broker ACL, gateway, and consumer layer that is visible. Bindings outside the repo: Needs validation.
