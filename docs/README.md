# Documentation

Study of the release-please workflow and how to align GitHub releases and tags
with the commit that was deployed to staging.

- [Functional workflows](functional-workflows.md): mermaid diagrams of the
  three situations the CI handles — a normal push, a push that becomes a
  release candidate, and a release PR merge that publishes a release.
- [Release-please default behavior](release-please-default-behavior.md):
  how release-please works out of the box, and which commit the GitHub release
  and tag point to (verified on the `v0.1.0` release of this repository).
- [Options considered](release-tag-options.md): the options evaluated to make
  the release tag point to the staging-deployed commit, and why option 3
  (`skip-github-release` plus a custom publish job) was selected.
- [Release workflow](release-workflow-proposal.md): the implemented CI
  design, the revisions coming from its adversarial review, the failure
  modes, and the caveats we accept.
- [Staging deploy gating](staging-deploy-gating.md): options to deploy to
  staging only when release-please would have created (or updated) a release
  PR, the race conditions of each option, and why gating on the
  release-please action outputs is selected (and now implemented).
- [Manual deploy](manual-deploy.md): the `workflow_dispatch`-only workflow
  deploying a given tag to `stg` or `prod`, and the corner cases of dynamic
  GitHub environment selection (auto-created environments, deployment branch
  policies, deployment records).
