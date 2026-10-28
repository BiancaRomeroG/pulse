## Context
<!-- Why this change exists. Link the issue: Closes #NN. Link the spec task id. -->

## Change
<!-- What changed, at the level a reviewer needs. Bullets. Call out anything non-obvious. -->

## How to test
<!-- Exact commands and expected output. Paste the test summary (counts, coverage) you got locally. -->

```
pnpm turbo lint typecheck test
```

## Screenshots / GIF
<!-- Required for any UI change. Before/after when relevant. -->

## Documentation
- [ ] README updated (if user-facing)
- [ ] `docs/architecture.md` updated (if structure or flows changed)
- [ ] `docs/data-model.md` updated (if schema changed)
- [ ] `docs/runbook.md` updated (if setup, env or ops changed)
- [ ] `docs/api/openapi.json` regenerated (if API changed)
- [ ] ADR added/updated: <!-- link or "n/a" -->
- [ ] `docs/decisions-log.md` updated (if ADR added)

## Risk and rollback
<!-- What could break, how we would notice, how to roll back. "Low: additive, behind no flag" is a valid answer. -->

## Checklist
- [ ] Conventional commit messages; PR title reads well in a changelog
- [ ] Tests added or updated; no skipped tests without an issue link
- [ ] No secrets, no `any`, no `console.log`
- [ ] CI green
