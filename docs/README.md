# Documentation

Current reference documentation for this repository's release automation.
Historical design studies are kept in [archives](archives/).

## Start here

- [Functional workflows](functional-workflows.md): overview and detailed
  Mermaid diagrams of the implemented CI/release/deploy flow.
- [Release workflow](release-workflow.md): reference for the implemented
  `release-please.yaml` workflow and why the release tag points at the
  already-staged commit.

## Reference pages

- [Release-please default behaviour](release-please-default-behavior.md): how
  Release Please tags releases by default, and why this repository overrides
  the publish step.
- [Staging deploy gating](staging-deploy-gating.md): the current rule for
  deploying to staging only when Release Please created or updated a release
  PR.
- [Build and docker push gating](fake-build-push-gating.md): how `code-checks`
  builds the app/image and skips the docker push on release PR merge commits.
- [Manual deploy](manual-deploy.md): the `workflow_dispatch` workflow for
  deploying an existing tag to `stg` or `prod`.

## Archives

- [Release tag options](archives/release-tag-options.md): option study for
  aligning release tags with the staging-deployed commit.
- [Release workflow proposal](archives/release-workflow-proposal.md): original
  proposal, adversarial-review revisions, failure modes, and caveats.
- [Staging deploy gating study](archives/staging-deploy-gating-study.md):
  option study and race analysis for staging deploy gating.
