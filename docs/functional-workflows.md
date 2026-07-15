# Functional workflows

This page is an **explanation** of the release automation shape: it first gives
a broad view of what happens for each important event, then a detailed view of
which GitHub workflow, job, and composite action produces and consumes each
output.

See [release-please default behavior](release-please-default-behavior.md) and
[staging deploy gating](staging-deploy-gating.md) for the detailed reasoning
behind the release gates, and
[build and docker push gating](fake-build-push-gating.md) for the
`code-checks` build/push details.

## Big feature overview

```mermaid
flowchart TD
    PR([Code pushed to a PR branch])
    MAIN([Code pushed to main])
    PREP([Release prepared by Release Please])
    MERGE([Release PR merged])
    TAG([Release tag v* created])

    PR --> PR_CC[code-checks.yaml\npre-commit + app/docker build\ndocker push runs]

    MAIN --> MAIN_CC[code-checks.yaml\npre-commit + app/docker build\ndocker push runs]
    MAIN --> RP[release-please.yaml\nclassifies commits since last release]

    RP -->|no release PR created/updated| NO_RELEASE[No staging deploy\nstaging unchanged]
    RP -->|release PR created/updated| PREP
    PREP --> STG[release-please.yaml\ndeploy to stg\nimage sha-<commit-sha>]
    PREP --> WAIT([Maintainer validates and merges the release PR])

    WAIT --> MERGE
    MERGE --> MERGE_CC[code-checks.yaml\nbuild still runs\ndocker push is skipped]
    MERGE --> PUBLISH[release-please.yaml\ncreate GitHub release + tag at\nmerge commit's first parent]

    PUBLISH --> TAG
    TAG --> PROD[retag-and-deploy-prod.yaml\nretag sha-<commit> to vX.Y.Z\ndeploy to prod]
```

## Detailed view

The diagrams below keep two levels of grouping:

1. workflow file (`code-checks.yaml`, `release-please.yaml`, ...);
2. job/action blocks inside the workflow.

Diamond-shaped output nodes show the relevant GitHub Actions outputs. Edges
labelled `consumes` show where one job output becomes another job's condition
or action input.

### 1. Code pushed to a PR branch

Only `code-checks.yaml` runs for a pull request branch update. The push becomes
a `pull_request` event (`opened`, `synchronize`, `reopened`, or
`ready_for_review`) targeting `main`; `release-please.yaml` is not triggered.

```mermaid
flowchart TD
    EVT([Pull request branch updated\npull_request event targeting main])

    subgraph CC[code-checks.yaml]
        subgraph PC[pre-commit job]
            DETECT[[Composite action:\ndetect-release-please-merge]]
            DETECT_OUT{{Action outputs:\nis-release-please-merge = false\npr-number = ''}}
            PC_OUT{{Job output:\nis-release-please-merge\nsource: steps.detect.outputs.is-release-please-merge}}
            PC_OTHER[Other steps:\ncheckout, mise setup,\npre-commit checks]

            DETECT --> DETECT_OUT
            DETECT_OUT --> PC_OUT
            DETECT_OUT --> PC_OTHER
        end

        subgraph FB[build job]
            FB_INPUT[Action input:\npush-image =\nneeds.pre-commit.outputs.is-release-please-merge != 'true'\n= true]
            BUILD_ACTION[[Composite action:\napp/docker build]]
            APP_BUILD[App build step]
            DOCKER_BUILD[Docker build step\nreports image sha-<commit-sha>]
            DOCKER_PUSH[Docker push step\nruns]

            FB_INPUT --> BUILD_ACTION
            BUILD_ACTION --> APP_BUILD --> DOCKER_BUILD --> DOCKER_PUSH
        end

        PC_OUT -->|consumes as build input| FB_INPUT
    end

    EVT --> PC
```

### 2. Code pushed to `main`

A push to `main` triggers both `code-checks.yaml` and `release-please.yaml`.
The code-checks side is the same as for a PR branch, but the release side may
either do nothing or prepare a release PR.

```mermaid
flowchart TD
    EVT([Push to main])

    subgraph CC[code-checks.yaml]
        subgraph PC[pre-commit job]
            CC_DETECT[[Composite action:\ndetect-release-please-merge]]
            CC_DETECT_OUT{{Action outputs:\nis-release-please-merge = false\npr-number = ''}}
            CC_PC_OUT{{Job output:\nis-release-please-merge\nsource: steps.detect.outputs.is-release-please-merge}}
            CC_OTHER[Other steps:\ncheckout, mise setup,\npre-commit checks]

            CC_DETECT --> CC_DETECT_OUT
            CC_DETECT_OUT --> CC_PC_OUT
            CC_DETECT_OUT --> CC_OTHER
        end

        subgraph FB[build job]
            CC_FB_INPUT[Action input:\npush-image =\nneeds.pre-commit.outputs.is-release-please-merge != 'true'\n= true]
            CC_BUILD_ACTION[[Composite action:\napp/docker build]]
            CC_DOCKER_BUILD[Docker build step\nimage sha-<commit-sha>]
            CC_DOCKER_PUSH[Docker push step\nruns]

            CC_FB_INPUT --> CC_BUILD_ACTION
            CC_BUILD_ACTION --> CC_DOCKER_BUILD --> CC_DOCKER_PUSH
        end

        CC_PC_OUT -->|consumes as build input| CC_FB_INPUT
    end

    subgraph RP[release-please.yaml]
        subgraph DETECT_JOB[detect job]
            RP_DETECT[[Composite action:\ndetect-release-please-merge]]
            RP_DETECT_OUT{{Action outputs:\nis-release-please-merge = false\npr-number = ''}}
            RP_DETECT_JOB_OUT{{Job outputs:\nis-release-please-merge\npr-number\nsource: steps.detect.outputs.*}}

            RP_DETECT --> RP_DETECT_OUT --> RP_DETECT_JOB_OUT
        end

        PUBLISH_SKIP[publish-release job\nskipped because\nis-release-please-merge = false]

        subgraph RELEASE_PR[release-pr job]
            RP_ACTION[[googleapis/release-please-action]]
            RP_DECISION{Release PR created\nor updated?}
            RP_ACTION_OUT{{Action output:\nprs_created = true or false}}
            RP_JOB_OUT{{Job output:\nprs-created\nsource: steps.release-please.outputs.prs_created}}

            RP_ACTION --> RP_DECISION
            RP_DECISION --> RP_ACTION_OUT --> RP_JOB_OUT
        end

        subgraph DEPLOY_STG[deploy-stg job]
            STG_INPUT[Job condition consumes:\nneeds.release-pr.outputs.prs-created == 'true']
            DEPLOY_ACTION[[Composite action:\ndeploy]]
            STG_RESULT[If true: deploy to stg\nimage sha-<commit-sha>\nIf false: job skipped]

            STG_INPUT --> DEPLOY_ACTION --> STG_RESULT
        end

        RP_DETECT_JOB_OUT -->|consumes is-release-please-merge| PUBLISH_SKIP
        PUBLISH_SKIP --> RELEASE_PR
        RP_JOB_OUT -->|consumes prs-created| STG_INPUT
    end

    EVT --> PC
    EVT -.triggers in parallel.-> DETECT_JOB
```

### 3. Release prepared by Release Please

This is the `prs-created = true` branch of the previous diagram. Release Please
has opened or updated the release PR, and the staging deploy runs for the same
commit.

```mermaid
flowchart TD
    EVT([Push to main with releasable state])

    subgraph RP[release-please.yaml]
        subgraph RELEASE_PR[release-pr job]
            RP_ACTION[[googleapis/release-please-action]]
            RP_PR[Open/update release PR\nbranch release-please--branches--main\nlabel autorelease: pending]
            RP_ACTION_OUT{{Action output:\nprs_created = true}}
            RP_JOB_OUT{{Job output:\nprs-created = true\nsource: steps.release-please.outputs.prs_created}}

            RP_ACTION --> RP_PR --> RP_ACTION_OUT --> RP_JOB_OUT
        end

        subgraph DEPLOY_STG[deploy-stg job]
            STG_INPUT[Job condition consumes:\nneeds.release-pr.outputs.prs-created = true]
            DEPLOY_ACTION[[Composite action:\ndeploy]]
            STG_IMAGE[Action inputs:\nenvironment = stg\nimage-tag = sha-github.sha\ncommit-sha = github.sha]
            STG_DONE([Staging now matches\nwhat the next release will ship])

            STG_INPUT --> DEPLOY_ACTION --> STG_IMAGE --> STG_DONE
        end

        RP_JOB_OUT -->|consumes prs-created| STG_INPUT
    end

    EVT --> RELEASE_PR
```

### 4. Release created

Merging the release PR creates a push to `main`. That single push triggers both
`release-please.yaml` and `code-checks.yaml`. The publish job creates the
release tag at the already-staged parent commit; that tag push then triggers
`retag-and-deploy-prod.yaml`.

```mermaid
flowchart TD
    EVT_MAIN([Release PR merged\npush: merge commit lands on main])

    subgraph RP[release-please.yaml]
        subgraph DETECT_JOB[detect job]
            RP_DETECT[[Composite action:\ndetect-release-please-merge]]
            RP_DETECT_OUT{{Action outputs:\nis-release-please-merge = true\npr-number = N}}
            RP_DETECT_JOB_OUT{{Job outputs:\nis-release-please-merge = true\npr-number = N\nsource: steps.detect.outputs.*}}

            RP_DETECT --> RP_DETECT_OUT --> RP_DETECT_JOB_OUT
        end

        subgraph PUBLISH[publish-release job]
            PUBLISH_INPUT[Job condition/input consumes:\nneeds.detect.outputs.is-release-please-merge = true\nneeds.detect.outputs.pr-number = N]
            PUBLISH_STEPS[Aggregated inline steps:\nread manifest version\nchoose merge commit parent\ncreate GitHub release + tag\nflip release-please labels]
            TAG_OUT([Creates tag vX.Y.Z\nat the already-staged parent commit])

            PUBLISH_INPUT --> PUBLISH_STEPS --> TAG_OUT
        end

        subgraph RELEASE_PR[release-pr job]
            RP_ACTION[[googleapis/release-please-action]]
            RP_NOTE[Only the hidden release merge commit\nis left since the new tag]
            RP_ACTION_OUT{{Action output:\nprs_created = false}}
            RP_JOB_OUT{{Job output:\nprs-created = false\nsource: steps.release-please.outputs.prs_created}}

            RP_ACTION --> RP_NOTE --> RP_ACTION_OUT --> RP_JOB_OUT
        end

        DEPLOY_SKIP[deploy-stg job skipped\nbecause prs-created = false]

        RP_DETECT_JOB_OUT -->|consumes is-release-please-merge and pr-number| PUBLISH_INPUT
        PUBLISH --> RELEASE_PR
        RP_JOB_OUT -->|consumes prs-created| DEPLOY_SKIP
    end

    subgraph CC[code-checks.yaml]
        subgraph PC[pre-commit job]
            CC_DETECT[[Composite action:\ndetect-release-please-merge]]
            CC_DETECT_OUT{{Action outputs:\nis-release-please-merge = true\npr-number = N}}
            CC_PC_OUT{{Job output:\nis-release-please-merge = true\nsource: steps.detect.outputs.is-release-please-merge}}
            CC_OTHER[Other steps:\ncheckout, mise setup,\npre-commit checks]

            CC_DETECT --> CC_DETECT_OUT
            CC_DETECT_OUT --> CC_PC_OUT
            CC_DETECT_OUT --> CC_OTHER
        end

        subgraph FB[build job]
            FB_INPUT[Action input:\npush-image =\nneeds.pre-commit.outputs.is-release-please-merge != 'true'\n= false]
            BUILD_ACTION[[Composite action:\napp/docker build]]
            DOCKER_BUILD[Docker build step\nimage sha-<merge-commit-sha>]
            DOCKER_PUSH[Docker push step\nskipped]

            FB_INPUT --> BUILD_ACTION --> DOCKER_BUILD --> DOCKER_PUSH
        end

        CC_PC_OUT -->|consumes as build input| FB_INPUT
    end

    EVT_TAG([Tag push v*])

    subgraph PROD[retag-and-deploy-prod.yaml]
        RETAG[retag job\ninline steps, no composite action\nretag sha-<commit> → vX.Y.Z]

        subgraph DEPLOY_PROD[deploy-prod job]
            PROD_DEPLOY[[Composite action:\ndeploy]]
            PROD_INPUTS[Action inputs:\nenvironment = prod\nimage-tag = github.ref_name\ncommit-sha = github.sha]
            PROD_DONE([Prod runs the same artifact\nalready validated on staging])

            PROD_DEPLOY --> PROD_INPUTS --> PROD_DONE
        end

        RETAG --> DEPLOY_PROD
    end

    EVT_MAIN --> DETECT_JOB
    EVT_MAIN -.triggers in parallel.-> PC
    TAG_OUT -->|raises| EVT_TAG
    EVT_TAG --> RETAG
```

Also available: [manual deploy](manual-deploy.md)
(`deploy-manual.yaml`, `workflow_dispatch`) lets someone deploy any existing
tag to `stg` or `prod` on demand, independent of these flows.
