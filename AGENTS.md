# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## What this repository is

A sandbox to experiment with [release-please](https://github.com/googleapis/release-please)
release automation. There is **no application source code** — the repository *is* its
CI/CD, tooling, and release configuration. Changes here are almost always to workflows,
pre-commit hooks, mise tasks, or the release-please setup.

## Contributing rules (mandatory)

- **Pre-commit must pass before pushing any commit.** Run `mise run precommit:run` and
  ensure it is green locally before pushing. The same checks run in CI
  ([code-checks.yaml](.github/workflows/code-checks.yaml)) and will block the PR otherwise.
- **Semantic (Conventional Commits) messages are mandatory, and a scope is required.**
  Format: `type(scope): description` (e.g. `feat(ci): add zizmor audit`). Commits without a
  scope are rejected — PR titles are enforced with `requireScope: true` in
  [lint-pr-title.yaml](.github/workflows/lint-pr-title.yaml), and release-please parses these
  types to build the changelog.
- Recognized types (see [release-please-config.json](release-please-config.json)):
  `feat`/`feature`, `fix`, `perf`, `revert`, `docs`, `style`, `chore`, `refactor`, `test`,
  `build`, `ci`. `chore`, `test`, and `build` are hidden from the changelog.

## Tooling

All tools and tasks are managed by [mise](https://mise.jdx.dev) via [.mise.toml](.mise.toml).
Run `mise install` once to provision pinned tool versions (actionlint, pre-commit, shellcheck,
shfmt, taplo, uv, zizmor).

## Common commands

```bash
mise run precommit:install   # install the git pre-commit hook (do this once after clone)
mise run precommit:run       # run ALL pre-commit hooks on ALL files (the gate before pushing)
mise run check:actionlint    # lint GitHub Actions workflows
mise run check:zizmor        # security-audit GitHub Actions workflows
```

Task definitions live in [toolbox/mise/tasks/toml/](toolbox/mise/tasks/toml/) and are wired in
through `task_config.includes` in [.mise.toml](.mise.toml). Add new tasks as TOML files there.

## Release flow

Releases are fully automated by [release-please.yaml](.github/workflows/release-please.yaml),
which runs on every push to `main`:

- Release type is `simple` (no language package manager). release-please opens/maintains a
  "release PR" that accumulates changelog entries from Conventional Commits.
- Merging the release PR tags the version and updates [CHANGELOG.md](CHANGELOG.md).
- The version is propagated to [version.txt](version.txt) (declared as an `extra-files`
  target; the `# x-release-please-version` marker on its line is what gets bumped) and tracked
  in [.release-please-manifest.json](.release-please-manifest.json).
- `initial-version` is `0.1.0`; `always-update: true` keeps extra-files in sync every run.

## CI overview

Three workflow groups run on PRs to / pushes to `main`:

- [code-checks.yaml](.github/workflows/code-checks.yaml) — runs `mise run precommit:run`.
- [workflows-checks.yaml](.github/workflows/workflows-checks.yaml) — actionlint + zizmor over
  `.github/` (workflow linting and security audit).
- [lint-pr-title.yaml](.github/workflows/lint-pr-title.yaml) — enforces a Conventional-Commit
  PR title with a required scope.

When editing workflows, keep third-party actions pinned to a commit SHA (with a version
comment) — zizmor enforces this. Note that `mise run check:zizmor` audits the whole `.github/`
tree, matching CI.
