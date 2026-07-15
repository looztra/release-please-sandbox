# Staging deploy gating

Status: **implemented** in
[release-please.yaml](../.github/workflows/release-please.yaml).

This page is reference documentation for the current staging deploy gate. The
option study and race analysis that led to it is archived in
[archives/staging-deploy-gating-study.md](archives/staging-deploy-gating-study.md).

## Rule

Deploy to staging only when Release Please created or updated a release PR for
the push.

In workflow terms, `deploy-stg` consumes the `release-pr` job output:

```yaml
if: >-
  !cancelled() &&
  needs.release-pr.result == 'success' &&
  needs.release-pr.outputs.prs-created == 'true'
```

`prs-created` comes from `googleapis/release-please-action`'s `prs_created`
output. It is `true` when the action created **or updated** a release PR.

## Why this is the right signal

The goal is:

> Staging always matches what the next release will ship.

This requires "release PR exists or was updated" semantics, not "this single
push contains a releasable commit" semantics. If a `chore` commit lands while a
`fix` release PR is already pending, Release Please updates that release PR and
staging must deploy the new head; otherwise the eventual release would include
code that never reached staging.

## Release PR merge commits

On the push that merges the release PR, `publish-release` creates the tag first.
Then `release-pr` sees only the hidden release merge commit since that tag, so
it outputs `prs-created = false` and `deploy-stg` is skipped.

That is intentional: the release tag points at the merge commit's first parent,
whose image was already deployed to staging by the previous push.

## Important GitHub Actions detail

The `!cancelled()` guard is required. On ordinary pushes, `publish-release` is
skipped; without a status-check function, GitHub Actions' implicit `success()`
would propagate that skip through the `needs` chain and skip `deploy-stg`
before evaluating `prs-created`.
