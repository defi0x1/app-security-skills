# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses [Semantic Versioning](https://semver.org/).

## [0.4.1]

### Changed

- Usage rewritten as a Goal / Example table; everything after `/appsec` is documented as plain language, not flags.
- README trimmed: removed the comparison table and duplicated principles.
- Removed redundant content across references (duplicate regexes, restated scales, repeated rows) and unsupported claims.
- Change classification simplified: the group is a label only and does not affect review depth, severity or verdict.
- `report-template.md`: classification field renamed to "Result" to avoid clashing with the merge verdict.

### Fixed

- Android/iOS secrets greps never scanned the target files (`--include` combined with explicit paths).
- `jsonwebtoken` fixed version, `arc4random` wrongly listed as weak, Android cleartext and `adb backup` statements, Keychain accessibility advice, Flutter/Dio API names.
- CWE-1426 replaced by CWE-1427 for LLM prompt-input rows; `multi-mr.md` remote parsing for ssh URLs; contradictions in `hunting-method.md`, `review-deep.md` and `web-auth.md`.

### Added

- GraphQL, slowloris/HTTP2 abuse, IaC state files, format-string and native command injection rows; stricter `pull_request_target` / `workflow_run` and IMDS guidance; supply-chain release-age cooldown.

## [0.4.0]

### Changed

- Entire skill set translated to English and normalized: one glossary for verdicts, gates, report sections and domain-file section names.
- Trigger phrases in `SKILL.md` and `workflows/review.md` are now English.
- Response language follows the user's language (default English).
- Examples and regexes made vendor-neutral.
- Reads the repo's own `REVIEW.md` / `CLAUDE.md` as review rules; reports every instance of a confirmed pattern under one finding; convergence rules from round 3; report opens with the finding count.

### Added

- Repository scaffolding: README, LICENSE, THIRD_PARTY_NOTICES, CONTRIBUTING, CHANGELOG.

## [0.3.0]

- Initial internal release: single `/appsec` entry point, Security and Code tracks, deep review engine, nine domain playbooks, mobile, LLM/agent/MCP and supply chain references.
