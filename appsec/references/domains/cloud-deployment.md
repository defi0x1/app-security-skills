# Domain: cloud and deployment

Use when the repo defines IAM, IaC (Terraform, Helm, Kustomize, CloudFormation), containers, Kubernetes, service mesh, serverless/edge, ingress, object storage, managed services, or per-environment configuration. Ask: does the workload get the right identity, isolation, network reachability, secrets, and policy. Source usually shows **intent**, not running reality: separate flaws confirmable from the repo from what the system owner must validate. Build and promotion: [ci-release.md](ci-release.md). HTTP proxy: [web-auth.md](web-auth.md). Tenancy in data stores: [data-isolation.md](data-isolation.md).

## Discipline

- Do not infer running exposure from a single manifest. Determine which environment uses it, which default or overlay modifies it, and whether that source path is actually in use.
- Map each workload's identity to specific operations and resources. A broad policy is a finding only when low-trust input can reach an unauthorized operation.
- Ingress, proxy, mesh, metadata service, and admission policy are real boundaries, but count only when both the configuration **and** its attachment are visible.
- A secret reference is not a secret leak. It needs a low-privilege reader, an output, an artifact, a log path, or an unsafe fallback.
- Account policy, network attachment, runtime admission, hosted metadata, drift: Needs validation.

## Workload identity and IAM

| Layer | What to hunt |
|---|---|
| Over-privileged workload identity | A workload, pod, function, edge worker, or node identity can affect tenants/accounts/resources/APIs beyond its role, and request/job input can choose the target. Check policy conditions, resource patterns (`*`), service account attachment, namespace mapping, fallback credentials |
| Cross-account/tenant role confusion | AssumeRole, external ID, token exchange, workload federation (GitHub OIDC → cloud), or resource policies accept claims not bound to the correct account, audience, repo, branch, namespace, or source workload |
| App authz relying on cloud metadata | The app trusts identity headers, tags, labels, account IDs, or resource metadata supplied by the caller without verifying they come from the control plane or a trusted proxy. Cloud IAM and application authz are two separate checks |

## Ingress, network, control plane

| Layer | What to hunt |
|---|---|
| Exposed service or management plane | Ingress, `LoadBalancer`/`NodePort` Services, listeners, `0.0.0.0/0` security groups, LB annotations, port mappings, or a `0.0.0.0` bind expose admin, debug, metrics, pprof, actuator, or internal APIs. A public route in the repo to a sensitive handler: confirmable. Only a missing network control: Needs validation |
| Bypassing the trusted proxy/mesh | The backend accepts forwarded identity, mTLS subject, or authz metadata from peers outside the ingress/sidecar; secondary ports, health, and legacy paths go around the mesh. Check header stripping, who can connect, fail-open when the proxy is missing |
| Reaching metadata and internal services | An untrusted URL/destination/protocol reaches the instance metadata service (169.254.169.254, IMDSv1), control plane sockets, or internal APIs carrying workload credentials. URL parsing: SSRF in [secure-coding-patterns.md](../secure-coding-patterns.md) section 6 |

## Container and orchestration

| Layer | What to hunt |
|---|---|
| Host/control plane privilege exposure | A workload can choose `privileged`, capabilities (`SYS_ADMIN`, `NET_ADMIN`), `hostNetwork/hostPID/hostIPC`, `hostPath`, devices, `docker.sock`/containerd socket, or an automounted service account token with broad RBAC. Missing seccomp or a read-only FS alone: hardening, unless a boundary-crossing operation is reachable |
| Inconsistent admission/policy | One deploy path enforces policy for images, namespaces, resources, secrets, and permissions; controllers, jobs, upgrades, restore, and compatibility paths do not |
| Trusting namespaces and labels | NetworkPolicy, admission, secrets, or workload identity depend on labels/annotations/names/namespaces that a low-trust principal can set |

## Configuration and secret lifecycle

| Layer | What to hunt |
|---|---|
| Control precedence drift | Dev values, chart defaults, ENV, CLI flags, feature gates, sidecar injection, or per-region overlays turn off auth, TLS, tenant scoping, or audit in one environment. **Render the final configuration** for each environment (`helm template`, `kustomize build`), not just the base file |
| Secrets leaking across workload boundaries | Secrets enter logs, crash reports, process arguments, shared ENV, broad volumes, build output, service discovery, or read APIs that another workload/tenant can access. A public endpoint or key ID is not a credential |
| Credential renewal and incident fallback | A failed mount, refresh, rotation, or revocation leaves old credentials valid or drops the app into a lower-trust identity mode. Check startup, readiness, reconnect, client caches |

## Storage, event, edge

| Layer | What to hunt |
|---|---|
| Object policy, signed URL | Bucket policies, object keys, CDN origins, or signed URLs not bound to principal, operation, namespace, audience, expiry. Check list/version and write as well |
| Event source identity confusion | A function/worker trusts a body field as the source identity without checking the provider-signed envelope, subscription/topic, account, region, replay |
| Edge and origin mismatch | The edge/serverless runtime assumes different secrets, APIs, FS, isolation, or tenant policy than the origin; fallback to origin changes permissions or caching |

## Common moves

- Render every maintained environment and build a matrix: external ports, workload identity, network peers, secret mounts, cloud resources. Differences need an owner's explanation.
- Follow untrusted requests, objects, labels, and events into cloud policy. Show which workload credential performs the final operation and which condition should have limited it.
- Compare normal deploy, migration, restore, node maintenance, failover, and local/emulator. Observe behavior when the mesh, admission, identity, secrets, or policy service is absent.

## Verification before reporting

1. Identify the source path in use and the effective deployed object; if that is not possible, Needs validation, stating which rendered manifest or attachment is missing.
2. State the low-trust caller/workload, the cloud or application identity, the attacker-controlled selector, the affected resource, and the unauthorized operation.
3. Check provider and orchestrator defaults by version. Do not assume a public IP, reachable metadata, an open firewall, or absent admission.
4. Verify only through template rendering, policy evaluation, or an emulator with a fake identity. Do not probe real endpoints and do not modify shared cloud resources.
