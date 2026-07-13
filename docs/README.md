# Documentation

Study of the release-please workflow and how to align GitHub releases and tags
with the commit that was deployed to staging.

- [Release-please default behavior](release-please-default-behavior.md):
  how release-please works out of the box, and which commit the GitHub release
  and tag point to (verified on the `v0.1.0` release of this repository).
- [Options considered](release-tag-options.md): the options evaluated to make
  the release tag point to the staging-deployed commit, and why option 3
  (`skip-github-release` plus a custom publish job) was selected.
- [Proposed release workflow](release-workflow-proposal.md): the proposed CI
  design, implementation sketch, and the caveats we accept.
