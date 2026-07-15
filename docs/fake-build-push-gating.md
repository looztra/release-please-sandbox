# Fake build and docker push gating

Status: **implemented** in
[code-checks.yaml](../.github/workflows/code-checks.yaml) and the local
composite action
[fake-build](../.github/actions/fake-build/action.yaml).

## What it simulates

A `fake-build` job runs after `pre-commit` in the `code-checks` workflow, on
every push and pull request the workflow already triggers on. It simulates,
in a single job, building the app and its Docker image for the current
commit:

1. **Fake app build**: a no-op step standing in for compiling/packaging the
   (non-existent) app.
2. **Fake docker build**: reports the image that would be built, tagged
   `sha-<commit-sha>` — the same tag convention `deploy-stg` in
   [release-please.yaml](../.github/workflows/release-please.yaml) already
   uses (`ghcr.io/${{ github.repository }}:sha-<sha>`), so the tag reported
   here is the one later fake-deployed to staging.
3. **Fake docker push**: reports pushing that same image, as a separate step
   with its own step-summary section — **gated**, see below.

No real app or Docker image is built or pushed anywhere; every step only
writes to `$GITHUB_STEP_SUMMARY`.

## Why gate the push on the release-please merge commit

`pre-commit` already runs the shared
[detect-release-please-merge](../.github/actions/detect-release-please-merge/action.yaml)
composite action (also used by `release-please.yaml`) and now exposes its
`is-release-please-merge` output at the job level. `fake-build` passes the
opposite of that output as the `push-image` input of the `fake-build` action:

```yaml
push-image: ${{ needs.pre-commit.outputs.is-release-please-merge != 'true' }}
```

The reasoning mirrors [staging deploy gating](staging-deploy-gating.md): on
the push that merges the release PR, the only new commit on `main` is the
hidden `chore(main): release` bump commit — its code is identical to the
commit already built (and pushed) by the previous push, the one that is
about to be tagged as the release ([release-please default
behavior](release-please-default-behavior.md)). Pushing another image for
that bump commit would be redundant: nothing consumes it, and the release
tag is created against the *parent* commit's already-pushed image, not this
one.

| Push on `main` | `is-release-please-merge` | Docker push |
| --------------- | -------------------------- | ----------- |
| ordinary/candidate push (`feat`, `fix`, ...) | `false` | runs |
| release PR merge commit | `true` | skipped |

The docker **build** step and its step-summary report always run regardless
of this gate — only the push (and its summary section) is skipped, so the
job still reports what image *would* have been built for the merge commit.

## Configurability

The gating condition lives in the calling workflow, not the action: the
`fake-build` action only exposes a generic `push-image` boolean input
(default `'true'`) with no knowledge of release-please. This keeps the
composite action reusable — a future caller could wire the same input to a
different condition (e.g. branch name, manual dispatch input) without
changing the action itself.
