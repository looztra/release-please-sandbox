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
    EVT([Push to main])

    subgraph RP[release-please.yaml]
        B[detect job\naction: detect-release-please-merge]
        B -->|is-release-please-merge = false| C[publish-release: skipped]
        C --> D[release-pr job\ngoogleapis/release-please-action]
        D -->|no releasable commits\nno PR pending| E[prs_created = false]
        E --> F[deploy-stg: skipped]
        F --> G([Staging unchanged])
    end

    subgraph CC[code-checks.yaml]
        H[pre-commit job\naction: detect-release-please-merge\nis-release-please-merge = false]
        H --> H1[fake-build job\naction: fake-build\nimage sha-<commit-sha>]
        H1 --> H2[docker push step: NOT skipped\npush-image input = true]
    end

    EVT --> B
    EVT -.triggers in parallel.-> H
```

## 2. Candidate release (a `feat`/`fix`/etc. push creates or updates the release PR)

```mermaid
flowchart TD
    EVT([Push feat/fix/perf/... to main])

    subgraph RP[release-please.yaml]
        B[detect job\nis-release-please-merge = false]
        B --> C[publish-release: skipped]
        C --> D[release-pr job\ngoogleapis/release-please-action]
        D --> E{Releasable commits\nsince last tag?}
        E -->|yes| F[Open/update release PR\nbranch release-please--branches--main\nlabel autorelease: pending]
        F --> G[prs_created = true]
        G --> H[deploy-stg job\ngated on prs_created]
        H --> I[action: fake-deploy\nstg, image sha-<commit-sha>]
        I --> J([Staging now matches\nwhat next release will ship])
        F -.awaits.-> K([Maintainer reviews/merges release PR])
    end

    subgraph CC[code-checks.yaml]
        L[pre-commit job\naction: detect-release-please-merge\nis-release-please-merge = false]
        L --> L1[fake-build job\naction: fake-build\nimage sha-<commit-sha>]
        L1 --> L2[docker push step: NOT skipped\npush-image input = true]
    end

    EVT --> B
    EVT -.triggers in parallel.-> L
```

## 3. Release created (release PR merged → tag/release → prod)

```mermaid
flowchart TD
    EVT1([Merge release PR into main\n→ push: merge commit lands on main])

    subgraph RP[release-please.yaml]
        C[detect job\naction: detect-release-please-merge\nis-release-please-merge = true, pr-number]
        C --> D[publish-release job]
        D --> D1[Read version from .release-please-manifest.json]
        D1 --> D2[Target = merge commit's first parent\nthe already-staged commit]
        D2 --> D3[Create tag vX.Y.Z + GitHub release\nnotes sliced from CHANGELOG.md]
        D3 --> D4[Swap label: pending → autorelease: tagged]
        D4 --> E[release-pr job runs again\ngoogleapis/release-please-action]
        E --> F[Only the hidden bump commit since tag\n→ no new PR created]
        F --> G[prs_created = false]
        G --> H[deploy-stg job: skipped\nalready deployed by the previous push]
    end

    subgraph CC[code-checks.yaml]
        M[pre-commit job\naction: detect-release-please-merge\nis-release-please-merge = true]
        M --> M1[fake-build job\naction: fake-build\nimage sha-<merge-commit-sha>]
        M1 --> M2[docker push step: skipped\npush-image input = false\nimage already built+pushed for the\nparent commit by the previous push]
    end

    EVT2([Tag push v*])

    subgraph RD[retag-and-deploy-prod.yaml]
        I[retag job\ninline steps, no composite action]
        I --> J[deploy-prod job\naction: fake-deploy\nprod, image vX.Y.Z]
        J --> L([Prod runs the same artifact\nalready validated on staging])
    end

    EVT1 --> C
    EVT1 -.triggers in parallel.-> M
    D3 -->|creates the tag,\nwhich raises| EVT2
    EVT2 --> I
```

Also available: [manual deploy](manual-deploy.md)
(`deploy-manual.yaml`, `workflow_dispatch`) lets someone deploy any existing
tag to `stg` or `prod` on demand, independent of these three flows.
