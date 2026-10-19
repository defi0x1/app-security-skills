# Domain: CI, build, release, updates

Use when the repo resolves dependencies, builds from untrusted contributions, runs CI, produces release artifacts, signs or promotes builds, loads plugins, or updates deployed software. This section is the chain of trust handoffs from source and dependencies to the artifact users run. Rules for lockfiles, registries, CVEs, SBOMs, CI tokens, pipelines: [supply-chain-devsecops.md](../supply-chain-devsecops.md); this file hunts **flow vulnerabilities** that checklist does not catch. Workload permissions at runtime: [cloud-deployment.md](cloud-deployment.md).

## Discipline

- A floating dependency or one with a CVE is not by itself a finding in the sense of this file (CVEs are still reported per supply-chain-devsecops.md). Here you must show: who can influence resolution, which build uses it, and which execution or release boundary is crossed.
- Follow integrity across every handoff: source identity, resolved inputs, build worker, artifact identity, test results, signature/attestation, promotion, updating consumers.
- **CI configuration is authorization code.** Determine which event triggers the workflow, whose code runs, which secrets and tokens it has, and what it can publish or modify.
- A checksum taken from the same untrusted place as the artifact does not prove integrity. Identify the root of trust and the behavior when verification fails.
- Branch protection, hosted runners, registries, signing services, production promotion not visible in the repo: Needs validation.

## Dependencies and build inputs

| Layer | What to hunt |
|---|---|
| Source and namespace confusion (dependency confusion) | Resolver configuration picks the wrong public/private namespace, registry fallback, mirror, or source URL. Unscoped internal package names; `--extra-index-url`; substitute build files; platform-dependent resolution; first install differing from update |
| Mutable build inputs | The build uses branches, tags, unverified submodules, downloaded tools (`curl \| sh`), generated assets, remote includes, CI actions not pinned to a SHA (`uses: x/y@v1`, `@main`), or mutable image tags. Needs a less-trusted writer and a path into trusted build output |
| Unchecked codegen and vendoring | Schemas, vendored archives, generated clients, translations, or binary blobs produce executable/shipped content without the review and integrity gates that source gets. Compare local regeneration against the committed output |
| Build context leaking data | Secrets, local config, `.git`, fixtures, or dev artifacts end up in the package/image because the build context and `.dockerignore`/`files` are broader than intended. Confirm that the final artifact exposes credentials, private data, or privileged configuration |

## CI and automation

| Layer | What to hunt |
|---|---|
| Untrusted code in a privileged workflow ("pwn request") | Fork PRs, issue comments, dependency updates, or external events run contributor-controlled code together with protected secrets, write tokens, deploy permissions, or trusted runners. GitHub: `pull_request_target` or `workflow_run` + checkout of `github.event.pull_request.head.sha`. GitLab: fork MR pipelines running with protected variables. Compare trigger type, checkout ref, approval gate, environment protection, narrowed `permissions` |
| Injection via workflow expressions | Branch names, commit messages, issue/PR titles and bodies, artifact names, matrix values, or generated outputs flow into `run:` shell, template expressions, or paths. GitHub: `run: echo "${{ github.event.issue.title }}"`. Correct: pass through `env:` and then use quoted `"$VAR"` |
| Mixed cache, artifact, workspace | A low-trust job writes a cache, artifact, workspace, or output that a later privileged job restores and then runs or releases. Check cache keys/namespaces, producer identity, digest binding, and whether promotion re-resolves by a mutable name |
| Over-privileged automation identity | A CI job has permissions beyond the operation, repo, environment, or duration it needs; untrusted job input can choose the affected resource. Missing least privilege alone: hardening; a reachable privileged operation is required |

## Release and updates

| Layer | What to hunt |
|---|---|
| Swap between build and promotion | Tests, review, signatures, or publishing reference a mutable tag, filename, channel, or artifact ID instead of the same immutable digest. Check every copy, repack, architecture merge, provenance step |
| Release authorization, signing policy | A release or signature is accepted from the wrong workflow, repo, branch, environment, key role, or threshold. Consumers must check the identity claims in the attestation, not only "signature valid". Rotation, expiry, and revocation must fail closed |
| Update metadata, rollback | The updater authenticates payload bytes but not version, product, platform, channel, target path, expiry, or rollback state, or accepts metadata and payload from two different transactions. Atomic install, recovery |
| Plugin and extension privilege expansion | An extension package has host permissions beyond its declared scope; a low-trust publisher replaces another publisher's identity; install/update hooks run before authenticity and capability checks. Installing an arbitrary plugin for oneself is intent, not a boundary |

## Common moves

- Walk backward from the released digest or installed update to every source, generated input, credential, worker, cache, test result, and authorization decision.
- Place untrusted and protected events side by side. Mark every persistent channel that passes between them; require an immutable identity and a trusted producer.
- Examine the path of a revoked key, failed download, missing attestation, partial platform release, rollback, and registry outage. Failure policy is part of release integrity.

```bash
# Quick probe hints (GitHub Actions; GitLab is similar with .gitlab-ci.yml)
grep -rnE 'pull_request_target|workflow_run|issue_comment' .github/workflows 2>/dev/null
grep -rnE '\$\{\{\s*github\.(event\.(issue|pull_request|comment|review|head_commit)|head_ref)' .github/workflows 2>/dev/null
grep -rnE 'uses:\s*[^@]+@(v?[0-9]+(\.[0-9]+)*|main|master)\s*$' .github/workflows 2>/dev/null   # action not pinned to SHA
grep -rnE 'permissions:|contents:\s*write|id-token:\s*write' .github/workflows 2>/dev/null
grep -rnE 'curl[^|]*\|\s*(ba)?sh|wget[^|]*\|\s*(ba)?sh' --include='*.yml' --include='*.yaml' --include='Dockerfile*' --include='*.sh' . 2>/dev/null
```

## Verification before reporting

1. State the low-trust actor, the source/cache/artifact/metadata it controls, the trusted job or updater that consumes it, and the result: unauthorized publish, code injection, secret exposure, privileged execution.
2. Prove that artifact identity breaks across a handoff: a mutable name or an unbound digest must reach a real consumer.
3. Check package manager, repo platform, registry, and signing service defaults at the pinned version. Unclear hosted controls: Needs validation.
4. Do not publish, do not modify real releases, and do not run workflows on shared runners to verify.
