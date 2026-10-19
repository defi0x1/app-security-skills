# Supply chain and DevSecOps

Rules and check commands. Severity, CWE: [severity-cwe.md](severity-cwe.md). Flow vulnerabilities in CI, build, release, updater, plugins (untrusted code running alongside secrets, workflow expression injection, caches/artifacts mixing trust levels, swaps between build and promotion): [domains/ci-release.md](domains/ci-release.md).

Before checking "dependency source": determine whether the organization has an internal registry/proxy (Artifactory, Nexus, GitLab Package Registry, Verdaccio...) via `.npmrc`, `pip.conf`, `settings.xml`, `GOPROXY`, `FROM <host>/...`, or by asking the user. If it does, every dependency and base image must go through it. If it does not, record a Note only, not a finding.

## Rules during development

| # | Rule | Reason |
|---|---|---|
| 1 | Commit a reviewed lockfile; CI installs with `npm ci` / `pnpm install --frozen-lockfile` / `yarn install --immutable` / `pip install --require-hashes` | A `^` range + no lockfile + `npm install` immediately pulls the latest version, including a hijacked version with injected malicious code. Changing `^` to `~` still receives a malicious patch; `npm audit` does not know about new malware |
| 2 | Dependencies and base images go through a controlled registry (curation, scan, SBOM, with an owner) | An image pulled straight from Docker Hub is outside the inventory: when a major CVE hits (e.g. Log4Shell) nobody is alerted about that component |
| 3 | **New** dependency: verify it exists, the maintainer, history, downloads; do not install on an unverified AI suggestion | Slopsquatting, typosquatting |
| 4 | CVE in a transitive dependency: force the patched version with `overrides` (npm) / `resolutions` (yarn) / `pnpm.overrides` / `dependencyManagement` (Maven) / `constraints` (Gradle); rerun tests, update the SBOM | Do not wait for the parent library. Log4j: force `log4j-core >= 2.17.1`; a WAF blocking `${jndi:` is only temporary |
| 5 | Do not put secrets in Dockerfile `ARG`/`ENV`; use BuildKit `RUN --mount=type=secret` | An `ARG` used in a `RUN` stays in `docker history`; a secret scanner on the repo does not see it |
| 6 | CI/CD tokens: prefer short-lived per-job OIDC (ID token, Workload Identity); if a high-privilege static token is unavoidable, make it short-lived (e.g. ≤ 30 days), narrow scope, protected + masked | A static token that never expires is a major risk when leaked |
| 7 | Secret scanner at both pre-commit and CI. A secret that reached the remote is considered leaked: **rotate** it, do not just delete the commit | Pre-commit can be bypassed with `--no-verify` |
| 8 | Generate two SBOMs: `trivy fs --format cyclonedx` (source) and `trivy image --format cyclonedx` (image) | They do not replace each other: the image SBOM adds the base image's OS packages and binaries installed at build time |
| 9 | Sign the image (cosign), generate SLSA provenance, **and** verify at admission (sigstore policy-controller / Kyverno) | Signing is meaningless if the cluster does not verify when admitting pods; Pod Security Admission does not check signatures |
| 10 | Prioritize patching by risk: KEV + reachable first; then high EPSS; CVSS is only technical severity | A CVSS 6.5 that is in KEV and reachable is patched before a CVSS 9.8 with EPSS 0.4% |

Terms: CVSS = severity; EPSS = probability of exploitation within 30 days; KEV = catalog of CVEs exploited in the wild (CISA); SSVC = action decision tree; VEX `not_affected` + `vulnerable_code_not_in_execute_path` = the component is still present but the vulnerable code is never reached. SAST = code you wrote; SCA = third-party libraries matched against CVEs.

## Checks (diff touching manifest, Dockerfile, CI; whole-repo review)

### Lockfile

| Ecosystem | Manifest | Lockfile |
|---|---|---|
| npm / yarn / pnpm | package.json | package-lock.json / yarn.lock / pnpm-lock.yaml |
| Python | requirements.txt / pyproject.toml / Pipfile | `==` with hashes / poetry.lock, uv.lock / Pipfile.lock |
| Go | go.mod | go.sum |
| Rust, PHP, Ruby | Cargo.toml, composer.json, Gemfile | Cargo.lock, composer.lock, Gemfile.lock |
| Maven | pom.xml | no native lockfile: versions must be pinned, do not report a missing lockfile |
| Gradle | build.gradle | gradle.lockfile |
| Mobile | Podfile, Package.swift, pubspec.yaml | Podfile.lock, Package.resolved, pubspec.lock |

Missing lockfile, floating versions (`^`, `~`, `*`, `latest`) on production dependencies, manifest changed but lockfile unchanged: LOW.

```bash
git ls-files | grep -Ei 'package-lock|npm-shrinkwrap|yarn\.lock|pnpm-lock|poetry\.lock|uv\.lock|Pipfile\.lock|go\.sum|Cargo\.lock|composer\.lock|Gemfile\.lock|gradle\.lockfile|Podfile\.lock|Package\.resolved|pubspec\.lock'
```

### CVE

```bash
trivy fs --scanners vuln --severity HIGH,CRITICAL .
# fallback: npm audit --omit=dev · pip-audit · govulncheck ./...
```

One finding per package: version, CVE/GHSA, severity, fixed version, direct/transitive, reachable if determinable. CWE-1395.

### Dependency source

```bash
# Existing registry configuration
git ls-files | grep -Ei '\.npmrc|\.yarnrc|pip\.conf|settings\.xml|nuget\.config|\.netrc'
# References to public registries in config/lockfiles
grep -rIEn --exclude-dir={.git,node_modules,vendor,dist,build,target} \
  'registry\.npmjs\.org|registry\.yarnpkg\.com|pypi\.org|files\.pythonhosted\.org|repo1?\.maven(\.apache)?\.org|jcenter|proxy\.golang\.org|api\.nuget\.org' .
# Base image: list FROM lines, compare against the internal registry (no host = Docker Hub)
grep -rEn '^[[:space:]]*FROM[[:space:]]' --include='Dockerfile*' --include='*.dockerfile' .
# Secrets via ARG
grep -rEn '^[[:space:]]*ARG[[:space:]].*(TOKEN|SECRET|PASSWORD|KEY)' --include='Dockerfile*' .
```

The `resolved` field in a lockfile is evidence of the registry actually used. A `FROM` pointing at a multi-stage build stage alias is not a violation. When the organization has an internal registry: pulling from public is MEDIUM, HIGH if production.

### CI pipeline

Check `.gitlab-ci.yml` (and every `include:`), `.github/workflows/`, `Jenkinsfile`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`, `.pre-commit-config.yaml`. Three questions:

1. **Automatic scanning** on MR/PR or push, not only on `schedule`.
2. **Enough scanners**: SAST (Semgrep, CodeQL, Gosec), SCA (Trivy, Dependabot, OSV), secrets (Gitleaks), container image (Trivy), IaC/misconfig (Trivy config, Checkov, Hadolint).
3. **No downgrading without reason**: `allow_failure: true` / `continue-on-error: true` on security jobs; `*_DISABLED`; `--exit-code 0`; dropped `--severity`; abuse of `--ignore-unfixed`; `.trivyignore`, `.semgrepignore`, `nosemgrep`, `nosec`, `noqa` without a reason.

```bash
grep -rEn 'schedule|merge_request|pull_request|CI_PIPELINE_SOURCE|^[[:space:]]*(rules|only|workflow|on):' .gitlab-ci.yml .github/workflows 2>/dev/null
grep -rEni 'sast|codeql|dependency|secret.detection|container.scanning|trivy|semgrep|gitleaks|gosec|hadolint|checkov' .gitlab-ci.yml .github/workflows 2>/dev/null
grep -rEn 'allow_failure:[[:space:]]*true|continue-on-error:[[:space:]]*true|_DISABLED|--exit-code[[:space:]]*0|--ignore-unfixed|EXCLUDED_PATHS' .gitlab-ci.yml .github/workflows 2>/dev/null
for f in .trivyignore .semgrepignore; do [ -f "$f" ] && grep -vE '^[[:space:]]*#|^[[:space:]]*$' "$f"; done
grep -rEn 'nosemgrep|nosec|noqa' --exclude-dir={.git,node_modules} . | grep -vEi 'reason|because|issue|#[0-9]'
```

Recommended pre-commit: pre-commit-hooks (detect-private-key, check-added-large-files, check-merge-conflict), gitleaks, trivy (vuln, config), hadolint, semgrep, gosec (Go).
