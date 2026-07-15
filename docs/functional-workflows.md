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
        subgraph J_detect[detect job]
            A_detect[[Action: detect-release-please-merge]]
            O_detect{{Job output:\nis-release-please-merge = false\npr-number = ''}}
            A_detect --> O_detect
        end

        J_publish[publish-release job: skipped]

        subgraph J_rp[release-pr job]
            A_rp[[googleapis/release-please-action]]
            O_rp{{Job output:\nprs-created = false}}
            A_rp -->|no releasable commits\nno PR pending| O_rp
        end

        J_deploy[deploy-stg job: skipped]
        STG([Staging unchanged])

        O_detect -->|consumes: is-release-please-merge| J_publish
        J_publish --> J_rp
        O_rp -->|consumes: prs-created| J_deploy
        J_deploy --> STG
    end

    subgraph CC[code-checks.yaml]
        subgraph J_precommit[pre-commit job]
            A_pc[[Action: detect-release-please-merge]]
            O_pc{{Job output:\nis-release-please-merge = false}}
            A_pc --> O_pc
            OTHER_PC[other steps:\nmise setup, run pre-commit checks]
            O_pc --> OTHER_PC
        end

        subgraph J_fb[fake-build job]
            A_fb[[Action: fake-build]]
            FB_BUILD[docker build step\nimage sha-<commit-sha>]
            FB_PUSH[docker push step: NOT skipped\ninput push-image = true]
            A_fb --> FB_BUILD --> FB_PUSH
        end

        O_pc -->|consumes: is-release-please-merge\nas push-image input| J_fb
    end

    EVT --> J_detect
    EVT -.triggers in parallel.-> J_precommit
```

## 2. Candidate release (a `feat`/`fix`/etc. push creates or updates the release PR)

```mermaid
flowchart TD
    EVT([Push feat/fix/perf/... to main])

    subgraph RP[release-please.yaml]
        subgraph J_detect[detect job]
            A_detect[[Action: detect-release-please-merge]]
            O_detect{{Job output:\nis-release-please-merge = false}}
            A_detect --> O_detect
        end

        J_publish[publish-release job: skipped]

        subgraph J_rp[release-pr job]
            A_rp[[googleapis/release-please-action]]
            RP_DECISION{Releasable commits\nsince last tag?}
            RP_PR[Open/update release PR\nbranch release-please--branches--main\nlabel autorelease: pending]
            O_rp{{Job output:\nprs-created = true}}
            A_rp --> RP_DECISION
            RP_DECISION -->|yes| RP_PR
            RP_PR --> O_rp
        end

        subgraph J_deploy[deploy-stg job]
            A_deploy[[Action: fake-deploy]]
            DEPLOY_NOTE[stg, image sha-<commit-sha>]
            A_deploy --> DEPLOY_NOTE
        end

        STG([Staging now matches\nwhat next release will ship])
        AWAIT([Maintainer reviews/merges release PR])

        O_detect -->|consumes: is-release-please-merge| J_publish
        J_publish --> J_rp
        O_rp -->|consumes: prs-created| J_deploy
        J_deploy --> STG
        RP_PR -.awaits.-> AWAIT
    end

    subgraph CC[code-checks.yaml]
        subgraph J_precommit[pre-commit job]
            A_pc[[Action: detect-release-please-merge]]
            O_pc{{Job output:\nis-release-please-merge = false}}
            A_pc --> O_pc
            OTHER_PC[other steps:\nmise setup, run pre-commit checks]
            O_pc --> OTHER_PC
        end

        subgraph J_fb[fake-build job]
            A_fb[[Action: fake-build]]
            FB_BUILD[docker build step\nimage sha-<commit-sha>]
            FB_PUSH[docker push step: NOT skipped\ninput push-image = true]
            A_fb --> FB_BUILD --> FB_PUSH
        end

        O_pc -->|consumes: is-release-please-merge\nas push-image input| J_fb
    end

    EVT --> J_detect
    EVT -.triggers in parallel.-> J_precommit
```

## 3. Release created (release PR merged → tag/release → prod)

```mermaid
flowchart TD
    EVT1([Merge release PR into main\n→ push: merge commit lands on main])

    subgraph RP[release-please.yaml]
        subgraph J_detect[detect job]
            A_detect[[Action: detect-release-please-merge]]
            O_detect{{Job output:\nis-release-please-merge = true\npr-number = N}}
            A_detect --> O_detect
        end

        subgraph J_publish[publish-release job]
            PUB_1[Read version from .release-please-manifest.json]
            PUB_2[Target = merge commit's first parent\nthe already-staged commit]
            PUB_3[Create tag vX.Y.Z + GitHub release\nnotes sliced from CHANGELOG.md]
            PUB_4[Swap label: pending → autorelease: tagged]
            PUB_1 --> PUB_2 --> PUB_3 --> PUB_4
        end

        subgraph J_rp[release-pr job\nruns again]
            A_rp[[googleapis/release-please-action]]
            RP_NOTE[Only the hidden bump commit since tag\n→ no new PR created]
            O_rp{{Job output:\nprs-created = false}}
            A_rp --> RP_NOTE --> O_rp
        end

        J_deploy[deploy-stg job: skipped\nalready deployed by the previous push]

        O_detect -->|consumes: is-release-please-merge,\npr-number| J_publish
        J_publish --> J_rp
        O_rp -->|consumes: prs-created| J_deploy
    end

    subgraph CC[code-checks.yaml]
        subgraph J_precommit2[pre-commit job]
            A_pc2[[Action: detect-release-please-merge]]
            O_pc2{{Job output:\nis-release-please-merge = true}}
            A_pc2 --> O_pc2
            OTHER_PC2[other steps:\nmise setup, run pre-commit checks]
            O_pc2 --> OTHER_PC2
        end

        subgraph J_fb2[fake-build job]
            A_fb2[[Action: fake-build]]
            FB_BUILD2[docker build step\nimage sha-<merge-commit-sha>]
            FB_PUSH2[docker push step: skipped\ninput push-image = false\nimage already built+pushed for the\nparent commit by the previous push]
            A_fb2 --> FB_BUILD2 --> FB_PUSH2
        end

        O_pc2 -->|consumes: is-release-please-merge\nas push-image input| J_fb2
    end

    EVT2([Tag push v*])

    subgraph RD[retag-and-deploy-prod.yaml]
        J_retag[retag job:\ninline steps, no composite action]

        subgraph J_deployprod[deploy-prod job]
            A_deployprod[[Action: fake-deploy]]
            DEPLOYPROD_NOTE[prod, image vX.Y.Z]
            A_deployprod --> DEPLOYPROD_NOTE
        end

        PROD([Prod runs the same artifact\nalready validated on staging])

        J_retag --> J_deployprod --> PROD
    end

    EVT1 --> J_detect
    EVT1 -.triggers in parallel.-> J_precommit2
    PUB_3 -->|creates the tag,\nwhich raises| EVT2
    EVT2 --> J_retag
```

Also available: [manual deploy](manual-deploy.md)
(`deploy-manual.yaml`, `workflow_dispatch`) lets someone deploy any existing
tag to `stg` or `prod` on demand, independent of these three flows.
