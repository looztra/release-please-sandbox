# Functional workflows

Visual summary of the three situations the CI in this repository handles:
an ordinary push, a push that becomes a release candidate, and the merge
that turns a candidate into a published release. See
[release-please default behavior](release-please-default-behavior.md) and
[staging deploy gating](staging-deploy-gating.md) for the detailed reasoning
behind each gate shown below, and
[fake build and docker push gating](fake-build-push-gating.md) for the
`code-checks` build/push details.

## 1. Normal push (no releasable changes pending/created)

```mermaid
flowchart TD
    A[Push to main] --> B[detect: is this a release-please merge commit?]
    B -->|false| C[publish-release: skipped]
    C --> D[release-pr: run release-please-action]
    D -->|no releasable commits\nno PR pending| E[prs_created = false]
    E --> F[deploy-stg: skipped]
    F --> G([Staging unchanged])
    D -.parallel.-> H[code-checks: pre-commit\ndetects is-release-please-merge = false]
    H --> H1[fake-build: app + docker build\nimage sha-<commit-sha>]
    H1 --> H2[docker push: NOT skipped\nnot a release-please merge commit]
    D -.parallel.-> I[workflows-checks: actionlint + zizmor]
```

## 2. Candidate release (a `feat`/`fix`/etc. push creates or updates the release PR)

```mermaid
flowchart TD
    A[Push feat/fix/perf/... to main] --> B[detect: not a release merge]
    B --> C[publish-release: skipped]
    C --> D[release-pr: release-please-action]
    D --> E{Releasable commits\nsince last tag?}
    E -->|yes| F[Open/update release PR\nbranch release-please--branches--main\nlabel autorelease: pending]
    F --> G[prs_created = true]
    G --> H[deploy-stg: gated on prs_created]
    H --> I[fake-deploy stg\nimage sha-<commit-sha>]
    I --> J([Staging now matches\nwhat next release will ship])
    F -.awaits.-> K([Maintainer reviews/merges release PR])
    A -.parallel.-> L[code-checks: pre-commit\ndetects is-release-please-merge = false]
    L --> L1[fake-build: app + docker build\nimage sha-<commit-sha>]
    L1 --> L2[docker push: NOT skipped\nnot a release-please merge commit]
```

## 3. Release created (release PR merged → tag/release → prod)

```mermaid
flowchart TD
    A[Merge release PR into main] --> B[Push: merge commit lands on main]
    B --> C[detect: is-release-please-merge = true, pr-number]
    C --> D[publish-release job]
    D --> D1[Read version from .release-please-manifest.json]
    D1 --> D2[Target = merge commit's first parent\nthe already-staged commit]
    D2 --> D3[Create tag vX.Y.Z + GitHub release\nnotes sliced from CHANGELOG.md]
    D3 --> D4[Swap label: pending → autorelease: tagged]
    D4 --> E[release-pr job runs again]
    E --> F[Only the hidden bump commit since tag\n→ no new PR created]
    F --> G[prs_created = false]
    G --> H[deploy-stg: skipped\nalready deployed by the previous push]
    D3 --> I[Tag push v* triggers\nretag-and-deploy-prod.yaml]
    I --> J[retag: fake-retag sha-<commit> → vX.Y.Z]
    J --> K[deploy-prod: environment=prod\nfake-deploy image vX.Y.Z]
    K --> L([Prod runs the same artifact\nalready validated on staging])
    B -.parallel.-> M[code-checks: pre-commit\ndetects is-release-please-merge = true]
    M --> M1[fake-build: app + docker build\nimage sha-<merge-commit-sha>]
    M1 --> M2[docker push: skipped\nrelease-please merge commit\nimage already built+pushed for the\nparent commit by the previous push]
```

Also available: [manual deploy](manual-deploy.md)
(`deploy-manual.yaml`, `workflow_dispatch`) lets someone deploy any existing
tag to `stg` or `prod` on demand, independent of these three flows.
