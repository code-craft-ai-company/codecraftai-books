# Chapter 26 — Case Study: Porting a Real Pipeline

> *Twenty-five chapters of discipline applied to one file.  The
> file goes from 200 lines and 47 environment variables to 30
> lines and zero application-content variables.  This is the
> chapter that shows the discipline pays off when applied to
> something with the proportions of a real problem.*

The case study below is composite — drawn from patterns that
recur across real CI pipelines without being any one team's
specific code.  The shape is realistic: a pipeline for a Python
web service with a Postgres backing store, deployed across two
regions of staging plus production-readonly debugging.  The
pipeline has accumulated cruft over three years; the engineers
who originally wrote each section have rotated off the team; the
README's "you might also need to..." section runs to fifteen
lines.

The chapter walks the migration in commit-sized steps.  Each step
produces a reviewable diff; each step is independently revertable;
each step makes the pipeline shorter and more maintainable.  By
the end of the chapter, the pipeline's surface area is a third
of what it was, every value has a typed home, and a vendor
migration would take thirty days rather than a quarter.

## Day 0 — The before state

The current `.gitlab-ci.yml`:

```yaml
# .gitlab-ci.yml — before (200 lines, abridged)
stages:
  - build
  - test
  - deploy
  - cleanup

variables:
  PYTHON_VERSION: "3.11"
  POETRY_VERSION: "1.7.1"
  AWS_DEFAULT_REGION: "us-east-1"
  AWS_ACCOUNT_ID: "123456789012"
  STAGING_DB_HOST: "db-staging-east-1.internal"
  STAGING_DB_PORT: "5432"
  STAGING_DB_NAME: "app"
  STAGING_DB_USER: "app_user"
  STAGING_API_HOST: "api-staging-east.internal"
  STAGING_API_PORT: "8443"
  REDIS_HOST: "redis-staging-east-1.internal"
  REDIS_PORT: "6379"
  PROD_DB_HOST: "db-prod-east-1.internal"
  PROD_DB_PORT: "5432"
  PROD_API_HOST: "api-prod-east.internal"
  PROD_API_PORT: "443"
  WEST_DB_HOST: "db-staging-west-1.internal"
  WEST_REDIS_HOST: "redis-staging-west-1.internal"
  LOG_LEVEL: "INFO"
  LOG_FILE_LEVEL: "DEBUG"
  RETRY_LIMIT: "3"
  RETRY_BACKOFF_MS: "500"
  PARALLELISM: "8"
  TIMEOUT_DEFAULT: "30"
  TIMEOUT_DB: "60"
  TIMEOUT_API: "10"
  OUTPUT_DIR: "/var/log/integration"
  DEBUGGER: "none"
  BREAKPOINT_AT: "none"
  ARTIFACT_BUCKET: "company-ci-artifacts"
  CACHE_BUCKET: "company-ci-cache"
  IMAGE_REGISTRY: "registry.example.com"
  RUNNER_IMAGE: "registry.example.com/runner:latest"
  TEST_DB_SEED_FILE: "/tmp/seed.sql"
  TEST_TIMEOUT_MULTIPLIER: "1"
  ENABLE_NEW_CACHING: "false"
  ENABLE_METRICS_V2: "false"
  ENABLE_TRACING: "true"
  TRACING_SAMPLE_RATE: "0.1"
  CLEANUP_AFTER_RUN: "true"
  NOTIFY_ON_FAILURE: "true"
  SLACK_WEBHOOK_BOT: "ci-bot-staging-east"
  PAGERDUTY_SERVICE: "staging-east-ci"
  GITHUB_TOKEN: ""              # populated from CI variables (masked)
  AWS_ACCESS_KEY_ID: ""         # populated (masked)
  AWS_SECRET_ACCESS_KEY: ""     # populated (masked)
  STAGING_DB_PASSWORD: ""       # populated (masked)
  PROD_DB_RO_PASSWORD: ""       # populated (masked)
  REDIS_PASSWORD: ""            # populated (masked)

default:
  image: $RUNNER_IMAGE
  before_script:
    - poetry install --no-interaction
    - if [ "$CI_PIPELINE_SOURCE" == "schedule" ]; then export TEST_TIMEOUT_MULTIPLIER=3; fi
    - if [ "$CI_COMMIT_REF_NAME" == "main" ]; then export DEPLOY_ENABLED=true; else export DEPLOY_ENABLED=false; fi

build-image:
  stage: build
  script:
    - docker build -t $IMAGE_REGISTRY/app:$CI_COMMIT_SHORT_SHA .
    - echo "$DOCKER_PASSWORD" | docker login $IMAGE_REGISTRY -u $DOCKER_USER --password-stdin
    - docker push $IMAGE_REGISTRY/app:$CI_COMMIT_SHORT_SHA
    - if [ "$CI_COMMIT_REF_NAME" == "main" ]; then docker tag $IMAGE_REGISTRY/app:$CI_COMMIT_SHORT_SHA $IMAGE_REGISTRY/app:latest; docker push $IMAGE_REGISTRY/app:latest; fi
  artifacts:
    when: on_success
    paths:
      - .build/
    expire_in: 1 day

provision-staging:
  stage: build
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - aws ec2 describe-instances --filters "Name=tag:Project,Values=staging-east" --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' --output text | grep running || aws ec2 run-instances --image-id ami-0abcd1234 --count 3 --instance-type t3.medium --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=staging-east},{Key=Lifecycle,Value=ephemeral}]'
    - aws ec2 wait instance-running --filters "Name=tag:Project,Values=staging-east"
    - sleep 30   # let userdata complete
    - ./scripts/configure-firewall.sh staging-east
    - ./scripts/seed-database.sh $STAGING_DB_HOST $STAGING_DB_USER $STAGING_DB_PASSWORD < $TEST_DB_SEED_FILE

smoke-tests:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - export TEST_LOG_LEVEL=$LOG_LEVEL
    - export TEST_PARALLELISM=$PARALLELISM
    - export TEST_TIMEOUT=$(($TIMEOUT_DEFAULT * $TEST_TIMEOUT_MULTIPLIER))
    - if [ "$CI_PIPELINE_SOURCE" == "schedule" ]; then export TEST_PROFILE=full; else export TEST_PROFILE=fast; fi
    - export DATABASE_URL="postgresql://$STAGING_DB_USER:$STAGING_DB_PASSWORD@$STAGING_DB_HOST:$STAGING_DB_PORT/$STAGING_DB_NAME?sslmode=require"
    - export REDIS_URL="redis://default:$REDIS_PASSWORD@$REDIS_HOST:$REDIS_PORT"
    - export API_BASE_URL="https://$STAGING_API_HOST:$STAGING_API_PORT"
    - poetry run pytest tests/smoke -v --output-dir=$OUTPUT_DIR/smoke
    - if [ "$NOTIFY_ON_FAILURE" == "true" ] && [ $? -ne 0 ]; then curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"Smoke tests failed in $CI_JOB_URL\"}" $SLACK_WEBHOOK_URL; fi
  artifacts:
    when: always
    paths:
      - $OUTPUT_DIR/smoke
    expire_in: 7 days

integration-tests:
  stage: test
  needs: [smoke-tests]
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    # ... 35 more lines, structurally identical to smoke-tests but with
    # different test paths, slightly different DATABASE_URL, retries
    # configured by hand, conditional logic for nightly vs PR runs ...
  artifacts:
    when: always
    paths:
      - $OUTPUT_DIR/integration
    expire_in: 7 days

deploy-staging:
  stage: deploy
  needs: [integration-tests]
  rules:
    - if: $CI_COMMIT_REF_NAME == "main" && $CI_PIPELINE_SOURCE == "push"
  script:
    - kubectl set image deployment/app app=$IMAGE_REGISTRY/app:$CI_COMMIT_SHORT_SHA -n staging-east
    - kubectl rollout status deployment/app -n staging-east --timeout=5m
    # ... 20 lines of post-deploy smoke verification with hardcoded URLs ...

cleanup:
  stage: cleanup
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: always
  script:
    - if [ "$CLEANUP_AFTER_RUN" == "true" ]; then
        aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Project,Values=staging-east" --query 'Reservations[*].Instances[*].InstanceId' --output text);
      fi
```

Two hundred lines.  Forty-seven environment variables (six masked
secrets plus forty-one application-content vars).  Five stages.
Inline AWS commands.  Hardcoded hostnames in five places.  Three
shell-conditional branches keyed off `CI_PIPELINE_SOURCE`.  Two
URL-construction lines that concatenate hostnames, ports,
credentials, database names, and TLS flags.  One
notification-on-failure block embedded in the test script.

Walk through what's wrong using the chapter-2 failure-mode
catalogue:

| Failure mode | Where it appears |
|---|---|
| 1. Env-var accretion | The whole `variables:` block — 47 entries |
| 2. Conditional thicket | The `before_script:` lines that set `TEST_TIMEOUT_MULTIPLIER` and `DEPLOY_ENABLED` based on multiple env-var checks |
| 3. Inlined recipe | `provision-staging`'s `aws ec2 run-instances` invocation |
| 4. Credential leak through env | `STAGING_DB_PASSWORD`, `REDIS_PASSWORD`, etc. exposed as environment variables |
| 5. Host name in YAML | `STAGING_DB_HOST`, `WEST_DB_HOST`, `PROD_DB_HOST`, all hardcoded |
| 6. Topology buried in step ordering | `needs:` declarations encode the test-then-deploy graph; cleanup runs `when: always` regardless of which earlier steps fired |

All six failure modes present in one file.  This is the proof
that the chapter-2 failure modes are not abstract — they
actually compound, and they actually accumulate in real
pipelines.

## Day 1 — Inventory and triage

The first migration step is *no code change*.  The team
inventories the file and groups every value by category.  An
afternoon of work; produces a spreadsheet (or for a smaller
project, a markdown table):

| Value | Category | Notes |
|---|---|---|
| `STAGING_DB_HOST` / `PORT` / `NAME` | Landscape (infrastructure service) | Will become a `web/database` infrastructure entry |
| `STAGING_API_HOST` / `PORT` | Landscape (infrastructure service) | Will become a `web/api` infrastructure entry |
| `REDIS_HOST` / `PORT` | Landscape (infrastructure service) | Will become a `cache/redis` infrastructure entry |
| `WEST_DB_HOST` / `WEST_REDIS_HOST` | Landscape (different env) | Will be in `landscape-staging-west.yaml` |
| `PROD_DB_HOST` / `PROD_API_HOST` | Landscape (different env, prod-readonly only) | Will be in `landscape-prod-readonly.yaml` |
| `LOG_LEVEL`, `LOG_FILE_LEVEL` | Runtime | Goes into `runtime-staging-east.yaml` |
| `RETRY_LIMIT`, `RETRY_BACKOFF_MS` | Runtime | Same |
| `PARALLELISM` | Runtime | Same |
| `TIMEOUT_DEFAULT` / `_DB` / `_API` | Runtime | Same |
| `OUTPUT_DIR` | Runtime | Same |
| `DEBUGGER`, `BREAKPOINT_AT` | Runtime | Same |
| `TEST_TIMEOUT_MULTIPLIER` | Runtime (different per profile) | Encodes the nightly-vs-PR difference; goes in profile-keyed runtimes |
| `ENABLE_NEW_CACHING`, `ENABLE_METRICS_V2`, `ENABLE_TRACING`, `TRACING_SAMPLE_RATE` | Runtime (feature flags) | Same |
| `STAGING_DB_PASSWORD` | Credential | Becomes `staging-db-rw` credential reference |
| `PROD_DB_RO_PASSWORD` | Credential | Becomes `prod-db-readonly` |
| `REDIS_PASSWORD` | Credential | Becomes `staging-redis-default` |
| `GITHUB_TOKEN` | Credential | Becomes `ci-bot-github` |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Credential | Becomes `ci-runner-aws` |
| `DATABASE_URL` (constructed) | Data Profile | The string isn't stored; it's built from the data-profile + credential at runtime |
| `REDIS_URL` (constructed) | Data Profile | Same |
| `API_BASE_URL` (constructed) | Data Profile | Same (a `web/api` profile) |
| `PYTHON_VERSION`, `POETRY_VERSION` | Toolchain manifest | Stays in `.tool-versions` |
| `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID` | Specification (provisioning venue) | Goes in `spec-staging-east.yaml` |
| `aws ec2 run-instances ...` recipe | Specification (provisioning device) | Same |
| `ARTIFACT_BUCKET`, `CACHE_BUCKET`, `IMAGE_REGISTRY` | Runtime (infrastructure addresses for CI) | Could go in runtime; arguable |
| `CLEANUP_AFTER_RUN`, `NOTIFY_ON_FAILURE` | Workflow parameters | Per-run flags; go in workflow YAML's `parameters:` |
| `SLACK_WEBHOOK_BOT`, `PAGERDUTY_SERVICE` | Landscape (infrastructure service) | These name external services |
| `RUNNER_IMAGE` | Vendor YAML | Stays in `.gitlab-ci.yml`'s `image:` |

Forty-seven values become entries in six categories (plus a
short fixed-vendor-shim list).  No code changes yet — the
inventory is the *plan*.  The implementation that follows is
mechanical.

## Day 5 — Author the typed configurations

The team authors five configuration files plus a profile set
plus a Specification file, in that order.  Each file lands as
its own commit; the team reviews each separately.

### Commit 1: `config/landscape/landscape-staging-east.yaml`

```yaml
environment:
    label: staging-east

apod:
    clients:
        - deviceType: network/client-linux
          name: ci-runner-1
          host: <set by spec-staging-east>
          role: runner
          credentials: [ci-runner-ssh]
          features: {parallelism_max: 8}
          skip: false

infrastructure:
    services:
        - serviceType: database/postgres
          name: staging-db
          host: db-staging-east-1.internal
          port: 5432
          database: app
          credentials: [staging-db-rw]

        - serviceType: cache/redis
          name: staging-redis
          host: redis-staging-east-1.internal
          port: 6379
          credentials: [staging-redis-default]

        - serviceType: web/api
          name: staging-api
          endpoint: https://api-staging-east.internal:8443
          credentials: [staging-api-token]

        - serviceType: web/webhook
          name: slack-notifications
          endpoint: ${VAULT:/secret/staging-east/slack-webhook}
          credentials: [slack-webhook]

        - serviceType: pagerduty/service
          name: ci-pager
          service_id: "staging-east-ci"
          credentials: [pagerduty-token]
```

Diff: +28 lines added.  No removals from `.gitlab-ci.yml` yet.
The commit is *additive* — the team adds the file to the
repository without breaking anything.

### Commit 2: similar landscape files for staging-west and prod-readonly

```
config/landscape/
├── landscape-staging-east.yaml         (commit 1)
├── landscape-staging-west.yaml         (commit 2)
├── landscape-prod-readonly.yaml        (commit 2)
└── landscape-local-dev.yaml            (commit 2)
```

Three more landscape files for the other environments.  Each
follows the same shape; only the hostnames and credential
identifiers differ.

### Commit 3: data profiles file

```yaml
# config/dataprofiles/dataprofiles.yaml
dataprofiles:
    - identifier: staging-db-rw
      category: basic-tcp
      host: db-staging-east-1.internal
      port: 5432
      dbtype: postgres
      dbname: app
      credential: staging-db-rw

    - identifier: staging-redis
      category: basic-tcp
      host: redis-staging-east-1.internal
      port: 6379
      dbtype: redis
      credential: staging-redis-default

    # ... staging-west variants, prod-readonly variants
```

The data profiles are dispatch entries for the per-environment
landscape variants.  Each profile carries its `host` field
(self-contained per chapter 8); the credential is referenced by
identifier.

### Commit 4: credentials file

```yaml
# config/credentials/credentials.yaml
credentials:
    - identifier: ci-runner-ssh
      categories: [ssh, environment:integration-ci]
      username: runner
      keyfile: ${FILE:/run/secrets/ssh/ci-runner-rsa}

    - identifier: staging-db-rw
      categories: [basic, environment:staging-east]
      username: app_user
      password: ${VAULT:/secret/staging-east/db-rw#password}

    - identifier: staging-redis-default
      categories: [basic, environment:staging-east]
      username: default
      password: ${VAULT:/secret/staging-east/redis#password}

    - identifier: ci-bot-github
      categories: [github-token, environment:integration-ci]
      username: ci-bot
      token: ${VAULT:/secret/ci/github-pat#token}

    - identifier: ci-runner-aws
      categories: [aws-access-key, environment:integration-ci, ttl:900]
      keyid: ${VAULT:/secret/ci/aws/runner#keyid}
      secret: ${VAULT:/secret/ci/aws/runner#secret}

    - identifier: prod-db-readonly
      categories: [basic, environment:prod, role:readonly, audit:required, ttl:300]
      username: ${VAULT:/secret/prod/db-readonly#username}
      password: ${VAULT:/secret/prod/db-readonly#password}

    # ... and so on
```

Every secret value is a `${VAULT:...}` reference (per chapter
10).  Every credential carries an `environment:<name>` scope tag
(per chapter 12).  The file is committable to the repository
without leaking any secret material.

### Commit 5: runtime profiles

```yaml
# config/runtime/runtime-base.yaml
breakpoints: []
debugger: none
logging:
    levels:
        console: INFO
        logfile: DEBUG
parallelism:
    max_workers: 8
retry:
    limit: 3
    backoff_ms: 500
timeouts:
    default_seconds: 30
    db_seconds: 60
    api_seconds: 10
paths:
    output: ./.output
feature_flags:
    enable_new_caching: false
    enable_metrics_v2: false
tracing:
    enabled: true
    sample_rate: 0.1
version: 1.0.0
```

```yaml
# config/runtime/runtime-integration-ci.yaml
# (inherits from base via profile composition)
property_test_caps:
    max_examples: 10           # fast-CI cap
```

```yaml
# config/runtime/runtime-nightly-full-sweep.yaml
property_test_caps:
    max_examples: null         # full sweep
timeouts:
    default_seconds: 90        # 3x multiplier from the original
                               # TEST_TIMEOUT_MULTIPLIER=3 logic
```

```yaml
# config/runtime/runtime-local-dev.yaml
logging:
    levels:
        console: DEBUG
timeouts:
    default_seconds: 300       # generous for breakpoints
property_test_caps:
    max_examples: 5
```

Four runtimes; each layers on top of `base`.  The chapter-7
runtime-vs-env-var thesis applied: every behaviour knob lives
in a runtime YAML field; the workflow does not export any of
them.

### Commit 6: provisioning specification

```yaml
# config/specs/spec-staging-east.yaml
environment:
    venues:
        - name: aws-us-east-1
          provider: aws
          management_url: https://ec2.us-east-1.amazonaws.com
          subscription: "123456789012"
          region: us-east-1
          credential: ci-runner-aws
    groups:
        - name: staging-east
          description: "Staging east runner pool"
          tags:
              - {key: Project, value: staging-east}
              - {key: Lifecycle, value: ephemeral}

profiles:
    - name: runner-medium
      description: "Standard medium-sized integration runner"
      credential: ci-runner-aws
      parameters:
          instance_type: t3.medium
          image: ami-0abcd1234
      tags: [{key: role, value: runner}]

apod:
    devices:
        - identifier: ci-runner-1
          category: ec2-instance
          group: staging-east
          profile: runner-medium
          role: runner
        - identifier: ci-runner-2
          category: ec2-instance
          group: staging-east
          profile: runner-medium
          role: runner
        - identifier: ci-runner-3
          category: ec2-instance
          group: staging-east
          profile: runner-medium
          role: runner
```

The chapter-4 worked example specialised to this project.  The
40-line `aws ec2 run-instances` shell from the original is now
20 lines of declarative YAML; the chapter-4 idempotent-on-re-run
property means the cleanup-and-recreate dance the original did
manually is unnecessary.

### Commit 7: profile files

```yaml
# config/profiles/base.yaml
profile:
    name: base
    select:
        landscape: staging-east
        runtime: base
        credentials: base
        dataprofiles: dataprofiles
```

```yaml
# config/profiles/integration-ci.yaml
profile:
    extends: base
    name: integration-ci
    select:
        runtime: integration-ci
```

```yaml
# config/profiles/nightly-full-sweep.yaml
profile:
    extends: base
    name: nightly-full-sweep
    select:
        runtime: nightly-full-sweep
```

```yaml
# config/profiles/local-dev.yaml
profile:
    extends: base
    name: local-dev
    select:
        landscape: local-dev
        runtime: local-dev
```

Four profiles; each chooses a small set of overrides on top of
`base`.  The discipline from chapter 17's profile promotion in
action: each profile is a delta from the base, additive.

## Day 12 — Author the workflow

The taskplus workflow file replaces the *flow* of the original
GitLab YAML — the build, provision, test, deploy, cleanup
sequence.  But not the *configuration* (already in the typed
files committed days 5–7) and not the *vendor wiring* (will be
handled in commit 9).

### Commit 8: `config/workflows/integration.yaml`

```yaml
# config/workflows/integration.yaml
name: integration

parameters:
    cleanup_after_run: true
    notify_on_failure: true

flow:
    # Build phase
    - taskgroup: Build
      items:
          - task: Build app image
            tasktype: my.taskplus.tasks.docker_build@DockerBuild
            image_name: app
            tag: ${parameters[commit_sha]}

          - task: Push app image
            tasktype: my.taskplus.tasks.docker_push@DockerPush
            image: ${parameters[commit_sha]}

    # Provision phase (only on MR / scheduled runs)
    - task: Provision
      tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
      specification: config/specs/spec-staging-east.yaml
      action: apply
      if: "parameters['source'] in ('merge_request_event', 'schedule')"

    # Test phase
    - taskgroup: Test
      items:
          - task: Smoke
            tasktype: automojo.taskplus.tasks.distributedtest@DistributedTestJobTask
            test_root: source/testroots/automojo
            includes: [automojo.tests.smoke]

          - task: Integration
            tasktype: automojo.taskplus.tasks.distributedtest@DistributedTestJobTask
            test_root: source/testroots/automojo
            includes: [automojo.tests.integration]
            onfailure: NotifyAndCleanup

    # Deploy phase (only on push-to-main)
    - taskgroup: Deploy
      if: "parameters['source'] == 'push' and parameters['ref'] == 'main'"
      items:
          - task: Deploy to staging
            tasktype: my.taskplus.tasks.kubectl_set_image@KubectlSetImage
            deployment: app
            namespace: staging-east
            image: ${parameters[commit_sha]}

    # Cleanup phase (always runs)
    - taskgroup: NotifyAndCleanup
      items:
          - task: Notify failure
            tasktype: my.taskplus.tasks.slack_notify@SlackNotify
            webhook_service: slack-notifications
            template: "Pipeline failed in {pipeline_url}"
            if: "parameters['notify_on_failure'] == True"

          - task: Teardown
            tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
            specification: config/specs/spec-staging-east.yaml
            action: teardown
            if: "parameters['cleanup_after_run'] == True"
```

About fifty lines.  Walk what changed:

- **No env-var soup.**  No `LOG_LEVEL`, no `RETRY_LIMIT`, no
  `STAGING_DB_HOST`.  All of those are configuration content
  loaded by the workflow's startup.
- **No inline AWS commands.**  Provisioning is a chapter-25
  bridge task consuming the chapter-4 Specification.
- **No URL construction.**  The smoke and integration tests
  invoke the chapter-25 `DistributedTestJobTask`, which finds
  `staging-db`, `staging-redis`, and `staging-api` through the
  active landscape.
- **No conditional shell.**  The `if:` fields on tasks and
  taskgroups are typed Python expressions reading from
  `parameters`.

### Revisiting the chapter-22 cross-group jump subtlety

The workflow above has the `Integration` task at ordinal `"3.2"`
(inside the `Test` taskgroup) declaring `onfailure:
NotifyAndCleanup`.  This is the chapter-22 cross-group jump
case.

Chapter 22 documented a subtlety: when seek mode is active and
the outer loop encounters a TaskGroup whose own label doesn't
match, the entire group is skipped.  In the workflow above:

- `Integration` fails; `seeking_node` becomes `"NotifyAndCleanup"`.
- The loop reaches the `Deploy` taskgroup.  Its label is
  `"Deploy"`; mismatch; the group is skipped (correct).
- The loop reaches the `NotifyAndCleanup` taskgroup.  Its label
  is `"NotifyAndCleanup"`; *match*; the seek state clears; the
  group runs.

The key here is naming the *taskgroup* with the jump target's
label, not naming an interior task.  When the workflow author
follows this convention — *the failure-handler taskgroup's
own label is the seek target* — cross-group jumps work as
intuited.

The alternative — declaring `onfailure: Teardown` (a label
inside the `NotifyAndCleanup` taskgroup) — would *not* work,
because the engine would skip past the `NotifyAndCleanup` group
without descending.  The workflow above avoids the trap by
naming the group itself as the target.

The workflow's `NotifyAndCleanup` group runs *both* the
notification task and the teardown task on failure, which
matches the original GitLab YAML's intent (cleanup runs
`when: always`).

## Day 14 — The vendor shim

### Commit 9: shrink `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — after (30 lines)
default:
  image: registry.example.com/runner:latest
  before_script:
    - poetry install --no-interaction

stages:
  - run

integration-mr:
  stage: run
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    AM_PROFILE: integration-ci
  script:
    - automojo testbase test run --workflow config/workflows/integration.yaml ||
      (automojo profile capture --output failure-snapshot.json && exit 1)
  artifacts:
    when: on_failure
    paths:
      - failure-snapshot.json
      - .output/
    expire_in: 1 month

nightly-full-sweep:
  stage: run
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  variables:
    AM_PROFILE: nightly-full-sweep
  script:
    - automojo testbase test run --workflow config/workflows/nightly.yaml
  artifacts:
    when: always
    paths: [.output/]
    expire_in: 1 month
```

Thirty lines.  The chapter-24 pattern applied.  The diff in
the GitLab YAML is enormous: 200 lines down to 30, with the
deleted lines distributed across removed `variables:` (47 →
just `AM_PROFILE`), removed shell commands (the `aws ec2`, the
URL constructions, the conditional setups), and removed staging
declarations (the workflow handles its own internal sequencing).

## Day 30 — Migration complete

The repository's structure post-migration:

```
project-root/
├── .gitlab-ci.yml                      30 lines (was 200)
├── .tool-versions                      4 lines  (kept)
├── config/
│   ├── credentials/credentials.yaml    50 lines (NEW)
│   ├── dataprofiles/dataprofiles.yaml  40 lines (NEW)
│   ├── landscape/
│   │   ├── landscape-staging-east.yaml 30 lines (NEW)
│   │   ├── landscape-staging-west.yaml 30 lines (NEW)
│   │   ├── landscape-prod-readonly.yaml 25 lines (NEW)
│   │   └── landscape-local-dev.yaml    25 lines (NEW)
│   ├── runtime/
│   │   ├── runtime-base.yaml           20 lines (NEW)
│   │   ├── runtime-integration-ci.yaml 3 lines  (NEW)
│   │   ├── runtime-nightly-full-sweep.yaml 5 lines (NEW)
│   │   └── runtime-local-dev.yaml      8 lines  (NEW)
│   ├── specs/
│   │   └── spec-staging-east.yaml      45 lines (NEW)
│   ├── profiles/
│   │   ├── base.yaml                   8 lines  (NEW)
│   │   ├── integration-ci.yaml         5 lines  (NEW)
│   │   ├── nightly-full-sweep.yaml     5 lines  (NEW)
│   │   └── local-dev.yaml              7 lines  (NEW)
│   └── workflows/
│       ├── integration.yaml            50 lines (NEW)
│       └── nightly.yaml                30 lines (NEW)
└── README.md                           (now 5 lines, was 80)
```

Total lines of CI-related configuration:
- Before: 200 (one file, one repository).
- After: ~390 lines spread across 17 files.

The total grew.  That's a feature.  Before, two-hundred lines
in one file; the file had nineteen orthogonal concerns smeared
together.  After, three-hundred-ninety lines in seventeen files;
each file owns one concern.  The total is bigger; the per-file
diff size is much smaller.  Code review at the per-change level
gets sharper, not noisier.

## The operational payoff in numbers

Six concrete metrics worth measuring:

| Metric | Before | After |
|---|---|---|
| `.gitlab-ci.yml` line count | 200 | 30 |
| Application-content env vars in vendor YAML | 41 | 0 |
| Files touched to add a new test phase | 1 (the YAML) | 1 (the workflow) |
| Files touched to switch staging-east → staging-west | 1 (the YAML, 8 places) | 1 (set `AM_LANDSCAPE_NAMES`) |
| Time to reproduce a CI failure locally | ~30 min (read logs, set up env) | <10 min (download capture, replay) |
| README's "you might also need to..." length | 80 lines | 5 lines |

The metrics are not the only payoff.  Two qualitative changes
matter as much:

- **Configuration drift across environments became visible.**
  Before the migration, "what's different between staging-east
  and staging-west?" was a question requiring archaeology.
  After, it's a `automojo profile diff` invocation that prints a
  side-by-side diff in two seconds.
- **The team's mental model of the pipeline got smaller.**
  Before, debugging a pipeline failure required holding two
  hundred lines and forty-seven env vars in your head.  After,
  debugging starts with the workflow's flow tree (fifty lines),
  drills into the relevant configuration file (twenty to fifty
  lines), and stops there.

## What the migration didn't fix

Honesty about what didn't change.

### The runner image is still a black box

The `registry.example.com/runner:latest` image contains the
toolchain (Python, Poetry, the automojo packages, AWS CLI,
kubectl, docker).  The image's *contents* are still managed
through a separate Dockerfile and image-build pipeline; the
discipline in this volume doesn't apply to that layer.  When the
image breaks (a Poetry upgrade introduces a regression, an OS
package's ABI changes), debugging still requires going into the
Dockerfile and the image-build pipeline.

A team applying the discipline more broadly might bring the
image-build pipeline under the same typed-configuration
treatment — Specifications for the image, profiles for image
variants — but that's a follow-on project rather than part of
the migration above.

### Some accumulated weirdness survived

The original pipeline had a fifteen-line workaround in
`provision-staging` for a known issue where `aws ec2 wait
instance-running` doesn't actually wait for userdata to
complete.  The workaround (`sleep 30`) survives the migration —
it's now an explicit Specification field
(`post_create_delay_seconds: 30`), still ugly, but at least named.

The discipline doesn't *eliminate* workarounds; it makes them
*nameable*.  A workaround with a name is one a future engineer
can find, understand, and eventually replace.  A workaround
buried in a 200-line shell script tends to survive longer
because no one notices it.

### Cross-pipeline coordination

The team also had a pipeline for the deployment of the same app
to *production* (separate from the staging-east one above).  The
migration didn't unify the two pipelines — each kept its own
`.gitlab-ci.yml`, even though both now reference the same
typed configurations.  This is a deliberate scoping choice; the
two pipelines have different triggers, different on-call
escalation paths, different operational concerns.  Sharing
configuration without sharing the *pipeline structure* is the
right boundary.

## Summary

- A 200-line GitLab YAML with 47 environment variables migrated
  to a 30-line vendor shim plus 17 typed configuration files.
- The migration ran in nine commits over thirty days.  Each
  commit was independently reviewable; each commit was
  independently revertable; the migration made progress every
  day without breaking the pipeline.
- The chapter-2 failure modes (env-var accretion, conditional
  thicket, inlined recipe, credential leak through env, host
  name in YAML, topology buried in step ordering) all appeared
  in the original; all were closed in the migration.
- The chapter-22 cross-group jump subtlety was navigated by
  naming the *taskgroup* with the jump target's label, rather
  than naming an interior task.  The convention is small but
  important.
- The total lines of configuration *grew* (from 200 to ~390),
  spread across 17 files instead of 1.  The per-file diff size
  shrunk dramatically; code review got sharper.
- Operational payoffs: zero application-content env vars,
  single-file changes for previously cross-cutting edits, sub-
  ten-minute CI failure reproduction, a five-line README
  section in place of an 80-line one.
- Honest residual: the runner image is still a black box; some
  workarounds survive (now nameable rather than buried);
  cross-pipeline structural unification is out of scope.

The volume's last chapter — chapter 27 — is the inverse of
this one's claim: where this chapter showed what the discipline
buys when applied, chapter 27 names where the discipline does
*not* fit and what to use instead.
