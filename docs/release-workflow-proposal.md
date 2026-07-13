# Release workflow: publish at the staging-deployed commit

Status: **implemented** in
[release-please.yaml](../.github/workflows/release-please.yaml) (PR #10),
after an adversarial review of the initial proposal (see
[Design revisions](#design-revisions-from-the-adversarial-review)).

This implements option 3 from [options considered](release-tag-options.md):
release-please only manages the release PR, and a dedicated job creates the
tag and the GitHub release at the commit that was deployed to staging.

## Overview

A single workflow triggered by `push` on `main`, with three serialized jobs:

1. **`detect`**: runs the local composite action
   [detect-release-please-merge](../.github/actions/detect-release-please-merge/action.yaml)
   (shared with the code-checks and deploy-stg workflows) and exposes
   `is-release-please-merge` and `pr-number` outputs.
2. **`publish-release`** (only when the pushed commit is a release PR merge
   commit): a single `actions/github-script` step that creates the tag and
   the GitHub release in one `createRelease` API call, already pointing at
   the staging-deployed commit, then updates the release-please labels.
3. **`release-pr`**: the release-please action with
   `skip-github-release: true`, so it only creates and updates the release
   PR. It is gated to never run when detection or publishing failed.

Key building blocks:

- **Target commit**: the first parent of the release PR merge commit
  (`merge_commit_sha^`). This is the last commit pushed to `main` before the
  release PR merge, i.e. the commit the staging deployment ran for. Parent 0
  is the previous `main` HEAD for squash, merge-commit, and rebase merge
  methods alike.
- **Version**: parsed from `.release-please-manifest.json` at the merge
  commit — the authoritative, structured source (release-please wrote it in
  the release PR).
- **Release notes**: the section release-please generated for this version in
  `CHANGELOG.md` at the merge commit. The version is matched anchored at the
  start of a `##`/`###` heading: a plain substring search would false-match
  the compare URLs of other version headings.
- **Steps in JavaScript**: custom logic uses `actions/github-script` rather
  than shell.

## Design revisions from the adversarial review

The initial proposal triggered the publish job on the `pull_request: closed`
event. This was dropped: merging the release PR fires both a `pull_request`
event and a `push` event, so the publish job and the release-please PR job
raced each other. If the PR job won, release-please evaluated "commits since
last release" while the new tag did not exist yet, and could transiently
open a wrong release PR. The implemented design keeps everything on the
`push` event and serializes publish before the PR phase with `needs`, plus a
single `concurrency` group (without `cancel-in-progress`) across runs — the
same ordering stock release-please uses within one run. A side benefit is
that no event from a fork can ever reach the publish job.

Other revisions against the first draft:

- Version read from the manifest instead of `version.txt` (generated
  extra-file, format owned by the `x-release-please-version` mechanism).
- Release notes extracted from `CHANGELOG.md` instead of slicing the PR body
  between `---` lines (fragile against `---` in changelog content, PR body
  edits, format changes, or a null body).
- The publish step is idempotent: an already-existing tag is tolerated
  (creation is skipped, label bookkeeping still runs), so re-running the job
  is always safe.
- The publish job reports version, tag, target commit, and release URL to
  the step summary, like the other workflows of this repository.

## Design notes

- **Use the PAT (`RELEASE_PLEASE_TOKEN`), not `GITHUB_TOKEN`**, in the
  github-script step. This is load-bearing: tags created with the default
  `GITHUB_TOKEN` do **not** trigger other workflows, so the retag workflow
  would never fire.
- **Keep the `vX.Y.Z` tag naming** identical to what release-please would
  have produced. Release-please finds the "latest release" by scanning tags
  and releases, so the next release PR computes correctly as long as the
  naming matches.
- **Label flip** (`autorelease: pending` to `autorelease: tagged`) keeps the
  repository state indistinguishable from a stock release-please run and
  prevents confusion if `skip-github-release` is ever removed later.

## Failure modes and recovery

- **Detection fails** (e.g. transient GitHub API error): `publish-release`
  and `release-pr` are both skipped — fail closed. Re-run the workflow.
- **Publishing fails after the merge**: `main` carries the bumped manifest
  and changelog but no tag exists yet (merged-but-untagged window).
  `release-pr` is skipped so release-please cannot observe that inconsistent
  state. Re-run the workflow: the publish step is idempotent (existing tag
  tolerated, label operations tolerate already-applied states).
- **No changelog section found for the version**: the release is still
  created (a release blocked on notes formatting would be worse); the body
  falls back to a link to the release PR and a warning is emitted.

## Caveats (accepted)

- **Bump files out of sync with the tag**: the tagged tree carries the
  *previous* `version.txt` and `CHANGELOG.md` (the bump commit is the tag's
  child, not its ancestor). Accepted by design: the version is carried by the
  Docker tag applied by the retag workflow, not by the file content.
- **Extra commit in the next release range**: on the next run, release-please
  walks commits since the tag's SHA, so the `chore(main): release X.Y.Z` merge
  commit falls inside the range. It is a hidden `chore` and never bumps the
  version, so this is harmless.
- **The release merge commit also deploys to staging**: **resolved**. This
  was originally accepted as noise (one extra staging image whose only delta
  is the bump files). It is no longer the case: the staging deployment
  workflow now gates on release-please detection. A `check-release-please`
  job runs the local composite action
  [detect-release-please-merge](../.github/actions/detect-release-please-merge/action.yaml)
  (shared with the code-checks workflow), and the `deploy-stg` job is skipped
  when the pushed commit is the merge commit of a merged release-please PR.
  No staging image is built or deployed for the bump commit, and nothing
  needs one: the release tag points to the merge commit's *parent*, whose
  image was built and deployed by the previous push.

  Remaining nuance, accepted: **the gate fails closed**. If the detection job
  itself fails (e.g. a transient GitHub API error), `deploy-stg` is skipped
  because its `needs` dependency failed. The failure mode is therefore "a
  legitimate feature push does not reach staging", never "a release commit
  does"; re-running the workflow recovers.
- **Inherent race (present in stock release-please too)**: if a feature commit
  lands on `main` between the release PR's last update and its merge, that
  commit is part of the tagged and deployed code but absent from the
  changelog. Branch protection or merging promptly after approval avoids it.
