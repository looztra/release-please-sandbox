# Manual deploy of a tag to stg or prod

Status: **implemented** in
[deploy-manual.yaml](../.github/workflows/deploy-manual.yaml).

A `workflow_dispatch`-only workflow (the successor of the old standalone
"Deploy to staging" workflow, whose push-triggered deploy moved into
[release-please.yaml](../.github/workflows/release-please.yaml) — see
[staging deploy gating](staging-deploy-gating.md)). It takes two inputs:

- `environment`: `stg` or `prod` (a **choice** input, see below);
- `tag`: the release tag to deploy (e.g. `v0.3.0`).

The workflow resolves the tag to its commit (dereferencing annotated tags
via `tag^{commit}`), derives the image tag from the target environment
(staging images are keyed by commit SHA, `sha-<sha>`; prod uses the
retagged `vX.Y.Z` image), and runs the shared
[fake-deploy](../.github/actions/fake-deploy/action.yaml) composite action
under `environment: ${{ inputs.environment }}`.

## Corner cases around dynamic environment selection

These were studied before implementing; they are operational knowledge for
anyone touching the workflow or the environment settings.

### Dynamic `environment:` is supported, protection rules apply

`environment.name` accepts expressions (`inputs`, `github`, `needs`, `vars`
contexts). The environment's protection rules (e.g. required reviewers on
`prod`) are enforced against the *resolved* name before the job starts: a
manual prod deploy waits for approval exactly like the tag-triggered
[retag-and-deploy-prod](../.github/workflows/retag-and-deploy-prod.yaml)
one.

### Unknown environment names are silently auto-created

If a job references an environment that does not exist, GitHub creates it
on the fly — **with no protection rules** — and the job proceeds unguarded.
A free-text input with a typo (`pord`) would therefore bypass the `prod`
required reviewers entirely. This is why the `environment` input is a
`type: choice` restricted to `stg` and `prod`: no free-text path exists.
Keep it a choice input.

### Deployment branch policies evaluate the run's ref, not the tag input

A `workflow_dispatch` run executes on the ref it was dispatched from
(usually `main`), even when the `tag` input says `v0.3.0`. If the `prod`
environment ever gets a *deployment branches and tags* policy, it must
allow **both**:

- `main` — for this manual workflow, and
- `v*` tags — for the tag-push-triggered retag workflow,

otherwise the corresponding path fails at job start with "branch not
allowed to deploy". No policy is configured in this sandbox today.

### The deployment record points at the run's ref

For the same reason, the GitHub deployment object created for a manual run
references the SHA of the dispatched ref (`main`'s HEAD), not the tag's
commit. The step summary reports the actual tag, resolved commit, and image.
This is cosmetic here; it would matter for tooling that reads the
deployments API.

The alternative — dispatching the workflow *on the tag ref* so the
deployment record and branch policies see the tag — was rejected: the
workflow file must exist at the dispatched ref, so tags predating the
workflow could never be deployed, and every future workflow fix would only
apply to tags created after it.

### The checkout is the run's ref, on purpose

The job checks out its own ref (not the tag) because the local
`fake-deploy` composite action must exist in the workspace, and older tags
predate it. The tag is resolved separately with `git fetch` +
`git rev-parse`, which also fails fast with a clear error when the tag does
not exist.

### Concurrency

Runs are serialized per target environment
(`group: deploy-${{ inputs.environment }}`, no cancellation). A manual
`stg` deploy can still run concurrently with the automated staging deploy
of [release-please.yaml](../.github/workflows/release-please.yaml) (which
serializes under its own `manage-releases` group) — accepted for this
sandbox, where deployments are simulated.
