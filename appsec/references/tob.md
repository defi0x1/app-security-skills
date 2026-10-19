# Trail of Bits

Optional: use only when the `trailofbits` marketplace is installed (`~/.claude/plugins/marketplaces/trailofbits/` exists; installed with `/plugin marketplace add trailofbits/skills`). Never install it yourself. Invoke by task description. Plugin output is a lead, not evidence: it goes through the 5 verification gates, and the read-only and no-network rules of SKILL.md still apply (do not run a plugin step that writes into the repo or probes anything).

| Plugin | Task description | When |
|---|---|---|
| fp-check | "Verify whether this finding is a false positive: {finding, file:line}" | Borderline finding (confidence 7), or before a Changes required verdict rests on it |
| differential-review | "Do a security-focused differential review of `$BASE...HEAD` with blast-radius estimation" | MR over 60 files or touching several services |
| variant-analysis | "Find other occurrences of this vulnerability pattern across the codebase: {pattern}" | After the first finding is confirmed |
| insecure-defaults | "Audit for fail-open insecure defaults: auth flags, default tokens, bind addresses" | Repo audit; diff touching config or bootstrap |
| audit-context-building | "Build deep architectural context for this codebase before a security audit" | Large repo, many services, first audit |
| supply-chain-risk-auditor | "Audit the npm/PyPI/Go dependencies for advisories, maintainer and takeover risk" | New dependency added |
