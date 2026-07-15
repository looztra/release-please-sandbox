# Release workflow

Status: **implemented** in
[release-please.yaml](../.github/workflows/release-please.yaml).

This page is reference documentation for the current release workflow. The
design study that led to it is archived in
[archives/release-workflow-proposal.md](archives/release-workflow-proposal.md).

## Purpose

The workflow keeps the Release Please release-PR experience while making the
release tag point at the same commit that was built and deployed to staging.
That lets the production retag workflow reuse the exact same image:

1. Release Please opens or updates the release PR.
2. The staging deploy runs for the candidate commit.
3. When the release PR is merged, the publish job creates the GitHub release
   and tag at the merge commit's first parent: the already-staged commit.
4. The tag push triggers production retag/deploy.

See [functional workflows](functional-workflows.md) for diagrams of the full
flow.

## Jobs

| Job | Runs when | Main responsibility |
| --- | --- | --- |
| `detect` | every push to `main` | Runs `detect-release-please-merge` and exposes `is-release-please-merge` plus `pr-number`. |
| `publish-release` | only when `detect` says the push is a release PR merge commit | Creates the GitHub release and tag at the merge commit's first parent. |
| `release-pr` | ordinary pushes and successful publish runs | Runs `googleapis/release-please-action` with `skip-github-release: true`, so it only manages the release PR. |
| `deploy-stg` | only when `release-pr` outputs `prs-created = true` | Deploys the candidate image to `stg`. |

## Load-bearing details

- `skip-github-release: true` is required: Release Please must not create the
  tag itself, because its default tag target is the release PR merge commit.
- The publish job uses `secrets.RELEASE_PLEASE_TOKEN`, not `GITHUB_TOKEN`, so
  the created tag can trigger `retag-and-deploy-prod.yaml`.
- The tag name remains `vX.Y.Z`, matching Release Please defaults so future
  version calculation continues to work.
- The publish job reads the version from `.release-please-manifest.json` and
  release notes from `CHANGELOG.md` at the merge commit.
- Release tag updates/deletions should be protected outside the workflow
  because production deploys trust tags to be immutable.

## Accepted caveat

The tagged tree carries the previous `version.txt` and `CHANGELOG.md` content,
because the tag points at the parent of the release PR merge commit. In this
repository, the Docker tag is the source of truth for the running version, so
this is intentional.
