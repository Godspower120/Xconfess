# Contributing

Thanks for helping improve this project. This guide covers the local workflow, the module map that helps you find the right owners and tests, and the validation commands CI runs.

## Getting started

1. Fork the repository and create a topic branch from `main`.
2. Install dependencies with `npm ci`.
3. Make your change, add focused tests, and run the validation commands below before opening a pull request.

## Module map

The contributor-ready module map lives at `docs/MODULE_MAP.md`. It is generated from maintained source metadata — primarily `.github/CODEOWNERS` and the repository layout — and lists, for each domain module:

- the module path and a short description,
- the owning team(s) from `CODEOWNERS`,
- the key test files that exercise the module,
- the validation commands to run before requesting review.

### Regenerating the map

After you add, move, or remove a module, or change ownership in `.github/CODEOWNERS`, regenerate the map:

```bash
npm run docs:module-map
```

Commit the regenerated `docs/MODULE_MAP.md` alongside your change. CI runs the same generator in check mode (`npm run docs:module-map:check`) and fails the build if the committed map is stale, so an out-of-date artifact will block your pull request.

### Failure behavior

The generator exits non-zero with a clear message when:

- `.github/CODEOWNERS` is missing or cannot be read,
- a `CODEOWNERS` entry cannot be parsed (for example, a pattern with no owners),
- a module has no matching owner in `CODEOWNERS`,
- a referenced key test path does not exist on disk.

Fix the underlying metadata rather than editing the generated file by hand; manual edits will be overwritten on the next regeneration.

## Validation

Run the documentation checks locally before pushing:

```bash
npm run ci:doc-audit && npm run ci:doc-links
```

These are the same checks CI enforces. If they fail, address the reported issue and re-run until they pass.

## Pull requests

- Keep changes scoped to the issue you are solving; avoid unrelated refactors.
- Include tests for the primary flow and for the privacy, compatibility, and regression risks specific to your change.
- Document any schema, configuration, rollout, or user-facing contract changes and confirm they are safe for existing data and clients.
- Make sure the module map is regenerated and committed when ownership or module layout changes.
