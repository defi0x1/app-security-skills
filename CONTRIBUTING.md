# Contributing

Thanks for helping improve `appsec`. The skill is a set of markdown files; contributions are edits to those files.

## Principles

- **Precision over volume.** A new hunting row must say when it is a finding and when it is not. Prefer rules that remove false positives over rules that add checks.
- **Evidence-gated.** Every check must be verifiable from source (`file:line`). Facts outside the repo belong in the *Needs validation* verdict, never guessed.
- **Read-only.** Nothing in the skill may modify the reviewed project or probe deployed systems.
- **Concise.** The skill is loaded into a model context. Reference files are read on demand, so keep each focused and avoid duplication across files.

## Vocabulary

See the layout in the [README](README.md#repository-layout). Keep the shared vocabulary consistent: `Needs validation`, `Approve` / `Approve with comments` / `Changes required before merge`, `Gate 1-5` (security), `Gate 1-4` (code), `Discipline` / `Common moves` / `Verification before reporting` (domain files).

## Making a change

1. Open an issue describing the false negative, false positive or gap, ideally with a minimal code sample.
2. Edit the relevant reference file. If you add a domain playbook, follow the structure of the existing ones and register it in the domain selection table in `references/hunting-method.md`.
3. Check relative links resolve and no real names, hosts or secrets appear in examples. Use placeholders such as `example.com`.
4. Add a line to `CHANGELOG.md`.
5. Open a pull request.

By contributing you agree your work is released under the MIT license.
