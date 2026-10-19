# Trail of Bits

Use only when `~/.claude/plugins/marketplaces/trailofbits/` exists. Invoke by task description. Results go through the 5 verification gates like any other source.

| Plugin | Task description | When |
|---|---|---|
| fp-check | "Verify whether this finding is a false positive: {finding, file:line}" | Finding with confidence 7, or before concluding a merge block |
| differential-review | "Do a security-focused differential review of `$BASE...HEAD` with blast-radius estimation" | MR with more than 60 files or touching multiple services |
| variant-analysis | "Find other occurrences of this vulnerability pattern across the codebase: {pattern}" | After the first finding is confirmed |
| insecure-defaults | "Audit for fail-open insecure defaults: auth flags, default tokens, bind addresses" | Repo audit; diff touching config or bootstrap |
| audit-context-building | "Build deep architectural context for this codebase before a security audit" | Large repo, many services, first audit |
| supply-chain-risk-auditor | "Audit the npm/PyPI/Go dependencies for advisories, maintainer and takeover risk" | New dependency added |
