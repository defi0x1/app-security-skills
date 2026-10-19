# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses [Semantic Versioning](https://semver.org/).

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
