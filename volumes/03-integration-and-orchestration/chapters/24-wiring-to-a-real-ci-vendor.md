# Chapter 24 — Wiring to a Real CI Vendor

> *The runner is dumb.  The workflow is portable.  When the CI
> vendor's YAML is small enough to fit on a screen, you have done
> the wiring correctly.*

Welcome to Part VI.  The volume's discipline (Parts I–IV) and
implementation (Part V) are now complete.  This part is the
*operational* layer — how the typed configurations and the
taskplus engine actually meet a CI vendor's pipeline runner; how
the discipline composes with the rest of the ccai portfolio
across volume boundaries; the case-study capstone that walks one
real port end-to-end; and the honest discussion of where the
discipline is *not* the right fit.

Four chapters in Part VI.  This one is the wiring.  The thesis is
narrow and uniform: **the CI vendor's YAML has exactly two jobs —
trigger the workflow and provide a runner image — and everything
else is the taskplus workflow's responsibility.**  When the
discipline holds, the vendor YAML shrinks to ten lines; when it
doesn't, you can see the discipline failing in the size of the
file.

The chapter walks three vendors.  GitLab CI, GitHub Actions, and
Jenkins.  Each gets a complete shim — the full vendor YAML from
trigger through runner — followed by the same handful of
operational notes (artifact upload, secret pass-through, parallel
matrix builds, retries) that recur across all three.

## The taskplus invocation

Every CI vendor's shim eventually contains the same one-line
invocation:

```bash
AM_PROFILE=integration-ci ccai testbase test run \
    --workflow config/workflows/integration.yaml
```

(Or, if the workflow is invoked directly by the taskplus runner
rather than wrapped through testbase:)

```bash
AM_PROFILE=integration-ci ccai taskplus workflow run \
    config/workflows/integration.yaml
```

That's it.  One environment-variable export (`AM_PROFILE`) plus
one command.  The vendor YAML's job is to set up the environment
in which this command runs and to capture its output afterward.

The volume's chapter-7 thesis applies most concretely here: the
plumbing layer is *small and stable*.  The configuration content
— landscapes, topologies, runtimes, credentials, data profiles —
all live in the typed YAML files the workflow loads.  The vendor
YAML never needs to know about any of them.

## GitLab CI shim

The full `.gitlab-ci.yml` for a project that runs an integration
suite on every merge request, plus a nightly full-sweep on
schedule:

```yaml
# .gitlab-ci.yml
default:
  image: registry.example.com/codecraftai-runner:latest
  before_script:
    - poetry install --no-interaction

stages:
  - test

integration-mr:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    AM_PROFILE: "integration-ci"
  script:
    - ccai testbase test run --workflow config/workflows/integration.yaml ||
      (ccai profile capture --output failure-snapshot.json && exit 1)
  artifacts:
    when: on_failure
    paths:
      - failure-snapshot.json
      - .output/
    expire_in: 1 month

nightly-full-sweep:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  variables:
    AM_PROFILE: "nightly-full-sweep"
  script:
    - ccai testbase test run --workflow config/workflows/nightly.yaml
  artifacts:
    when: always
    paths:
      - .output/
    expire_in: 1 month
```

Twenty-eight lines.  Walk what's there.

### The `default:` block

```yaml
default:
  image: registry.example.com/codecraftai-runner:latest
  before_script:
    - poetry install --no-interaction
```

Two responsibilities.  The `image:` provides the runner — a
container with the toolchain and the `codecraftai-*` packages
installed.  The `before_script:` runs the project's dependency
installation step.  These are the *one-time setup* steps for the
job; everything after is the workflow.

The chapter-15 local-dev migration suggested that toolchain
installation is a legitimate machine-setup imperative.  The same
applies in CI: the runner needs the toolchain.  The CI vendor's
`image:` field is the right place to provide it.

### The `integration-mr` job

```yaml
integration-mr:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    AM_PROFILE: "integration-ci"
  script:
    - ccai testbase test run --workflow config/workflows/integration.yaml ||
      (ccai profile capture --output failure-snapshot.json && exit 1)
  artifacts:
    when: on_failure
    paths:
      - failure-snapshot.json
      - .output/
    expire_in: 1 month
```

Six fields.  Two are the per-job concerns: `stage:` (where in the
pipeline this job runs) and `rules:` (when this job runs).  The
other four are the workflow's *narrow* contract with the vendor:

- `variables: AM_PROFILE: "integration-ci"` — the chapter-14
  profile selector.  One env var; the rest of the configuration
  selection cascades from the profile.
- `script:` — one line of `ccai testbase test run` plus the
  chapter-16 capture-on-failure pattern.
- `artifacts:` — the chapter-16 capture artifact plus the test
  output directory.

Notice what's *not* in the vendor YAML:

- **No env-var soup.**  The chapter-2 fail mode of fifteen
  `LOG_LEVEL=...`, `RETRY_LIMIT=...`, `STAGING_HOST=...`
  exports does not exist here.  The runtime profile carries the
  log level; the runtime profile carries the retry limit; the
  landscape carries the staging host.
- **No imperative setup.**  No `kubectl apply`, no `aws ec2
  create-instance`, no `docker pull`.  When provisioning is
  needed, it's a step *inside* the taskplus workflow (chapter
  4's Specification consumed by `ccai provisioning apply`).
- **No conditional branching by environment.**  No `if [ "$CI"
  = "true" ]; then ...; fi`.  The profile selector handles every
  environment-specific behaviour at the configuration layer.

### The `nightly-full-sweep` job

```yaml
nightly-full-sweep:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  variables:
    AM_PROFILE: "nightly-full-sweep"
  script:
    - ccai testbase test run --workflow config/workflows/nightly.yaml
  artifacts:
    when: always
    paths:
      - .output/
    expire_in: 1 month
```

The same shape as `integration-mr` — different profile, different
workflow file, different artifact policy.  Adding a new job to
the pipeline is a copy of the structure with the per-job fields
filled in.  No coordination across the pipeline; each job stands
alone.

The two jobs together demonstrate the chapter-13 ladder: the
same code runs in two CI scenarios, and what differs is the
profile selection plus the workflow YAML.

### Vendor secrets

The shim above doesn't show secret pass-through because the
chapter-12 discipline keeps secrets *out* of the vendor YAML
entirely.  The runner image has the credentials it needs to
fetch from Vault (or AWS Secrets Manager, or whichever backend);
the workflow's resolver fetches credentials at startup.  The
GitLab project's protected-variable settings hold the
*Vault-bootstrap* credentials (the AWS IAM role assumption, the
Vault token), not the application's runtime secrets.

A typical GitLab project's variable list is short:

| Variable | Purpose |
|---|---|
| `VAULT_ADDR` | Address of the Vault instance |
| `VAULT_ROLE_ID` | App-role ID for the runner to authenticate with Vault |
| `VAULT_SECRET_ID` | App-role secret ID, masked |

Three variables.  The runner image's startup uses them to
authenticate with Vault; once authenticated, the resolver fetches
every other credential from there.  None of the application's
runtime secrets ever appear in the GitLab project's variable
settings.

## GitHub Actions shim

The same project's `.github/workflows/integration.yml`:

```yaml
# .github/workflows/integration.yml
name: integration

on:
  pull_request:
  schedule:
    - cron: "0 4 * * *"   # nightly at 04:00 UTC

jobs:
  integration-pr:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    container: ghcr.io/example/codecraftai-runner:latest
    env:
      AM_PROFILE: integration-ci
      VAULT_ADDR: ${{ vars.VAULT_ADDR }}
      VAULT_ROLE_ID: ${{ vars.VAULT_ROLE_ID }}
      VAULT_SECRET_ID: ${{ secrets.VAULT_SECRET_ID }}
    steps:
      - uses: actions/checkout@v4
      - run: poetry install --no-interaction
      - run: |
          ccai testbase test run --workflow config/workflows/integration.yaml \
            || (ccai profile capture --output failure-snapshot.json && exit 1)
      - name: Upload failure artifact
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: failure-snapshot
          path: |
            failure-snapshot.json
            .output/

  nightly-full-sweep:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    container: ghcr.io/example/codecraftai-runner:latest
    env:
      AM_PROFILE: nightly-full-sweep
      VAULT_ADDR: ${{ vars.VAULT_ADDR }}
      VAULT_ROLE_ID: ${{ vars.VAULT_ROLE_ID }}
      VAULT_SECRET_ID: ${{ secrets.VAULT_SECRET_ID }}
    steps:
      - uses: actions/checkout@v4
      - run: poetry install --no-interaction
      - run: ccai testbase test run --workflow config/workflows/nightly.yaml
      - name: Upload artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: nightly-output
          path: .output/
```

Forty-three lines.  More verbose than the GitLab shim because of
GitHub Actions' explicit `actions/checkout`, `actions/upload-artifact`,
and per-step structure.  But the *structure* is identical:

- The vendor declares two jobs (`integration-pr`, `nightly-full-sweep`).
- Each job picks a runner (`runs-on: ubuntu-latest`, the runner
  image).
- Each job sets exactly four env vars: `AM_PROFILE` plus the
  three Vault-bootstrap secrets.
- Each job runs `ccai testbase test run`.
- Each job uploads the appropriate artifact.

The same workflow YAML — `config/workflows/integration.yaml` —
runs in both vendors.  The same profiles, same configurations,
same credentials apply.  The vendor YAML is *only* the wiring;
the *behaviour* is in the portable workflow.

### What changes between GitLab CI and GitHub Actions

Three categories of change when switching:

- **Trigger conditions.**  GitLab uses `rules: - if:
  $CI_PIPELINE_SOURCE == ...`; GitHub uses
  `if: github.event_name == '...'` plus the top-level `on:` block.
  Both express the same intent (run on PR; run on schedule) in
  vendor-specific shape.
- **Variable interpolation.**  GitLab uses `$VAR` directly in
  YAML; GitHub uses `${{ vars.VAR }}` and `${{ secrets.VAR }}`.
- **Artifact upload.**  GitLab has a top-level `artifacts:`
  field; GitHub uses an explicit `actions/upload-artifact@v4`
  step.

The translation is mechanical.  When a team migrates from one
vendor to the other, the shim files change; the workflow YAML,
the configuration files, the credential file — none of them
change.  This is the chapter-7 fixed-interface principle paying
off operationally.

## Jenkins shim

A Jenkins `Jenkinsfile` for the same project:

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'registry.example.com/codecraftai-runner:latest'
            args '-v ${WORKSPACE}:/workspace -w /workspace'
        }
    }

    environment {
        AM_PROFILE = 'integration-ci'
        VAULT_ADDR = credentials('vault-addr')
        VAULT_ROLE_ID = credentials('vault-role-id')
        VAULT_SECRET_ID = credentials('vault-secret-id')
    }

    stages {
        stage('Setup') {
            steps {
                sh 'poetry install --no-interaction'
            }
        }
        stage('Test') {
            steps {
                sh '''
                    ccai testbase test run --workflow config/workflows/integration.yaml \
                        || (ccai profile capture --output failure-snapshot.json && exit 1)
                '''
            }
        }
    }

    post {
        failure {
            archiveArtifacts artifacts: 'failure-snapshot.json,.output/**', allowEmptyArchive: true
        }
        always {
            archiveArtifacts artifacts: '.output/**', allowEmptyArchive: true
        }
    }
}
```

Thirty-three lines.  Jenkins's syntax differs more from GitLab and
GitHub than the two differ from each other (Groovy DSL versus
declarative YAML), but the structure is the same:

- Pick a runner (`agent { docker { image '...' } }`).
- Set the env vars (`environment { AM_PROFILE = 'integration-ci'
  ... }`).
- Run the toolchain install (`stage('Setup') { steps { sh ... } }`).
- Run the workflow (`stage('Test') { steps { sh '...' } }`).
- Archive artifacts (`post { failure { archiveArtifacts ... } }`).

A team running Jenkins for one project and GitHub Actions for
another can keep the *workflow YAML* and *configuration files*
identical between them; only the per-vendor shim differs.

## What stays in vendor YAML

The discipline asks the vendor YAML to do *exactly two things*:

### 1. Trigger

When does this pipeline run?  Per-vendor mechanism:

- GitLab: `rules:` with `$CI_PIPELINE_SOURCE` checks.
- GitHub Actions: top-level `on:` with `pull_request`,
  `schedule`, `push`, etc.
- Jenkins: triggers on the job's General page (or the `triggers
  { ... }` block in Jenkinsfile).

The trigger logic is genuinely vendor-specific — different
vendors expose different events with different conditional
syntax.  The discipline accepts this and confines it to the shim.

### 2. Runner image plus toolchain bootstrap

Which container, with which Python version and which ccai
packages, runs the job?  Per-vendor mechanism:

- GitLab: `default: image:` plus `before_script:`.
- GitHub Actions: `container:` plus a `run:` step.
- Jenkins: `agent { docker { image '...' } }` plus a setup stage.

The runner image typically pre-installs the toolchain
(`asdf`-managed Python, `poetry`, the `codecraftai-*` packages).
The shim's setup step is then a one-liner (`poetry install`) that
fetches the project's specific dependencies.

## What does *not* stay in vendor YAML

Five categories of content the discipline keeps *out* of the
vendor YAML:

- **Configuration content.**  Log levels, timeouts, retry counts,
  parallelism, hostnames, ports, database names — all live in
  the typed configuration files.  The vendor YAML mentions none
  of them.
- **Imperative provisioning.**  `aws ec2 create-instance`,
  `kubectl apply`, `terraform apply` — all become tasks inside
  the taskplus workflow when needed, consumed by chapter-4
  Specifications.
- **Step ordering of test phases.**  The order of "build → test
  → deploy" is in the taskplus workflow's `flow:` field, not in
  the vendor's `stages:` field.  The vendor YAML has *one* job
  per pipeline trigger; that job runs the workflow; the workflow
  handles its own internal ordering.
- **Cross-environment branching.**  No `if [ "$CI_COMMIT_BRANCH"
  == "main" ]; then ...; fi` in the vendor YAML.  The branch
  determines which profile to load (per chapter 13's ladder);
  the profile selector chooses the right configurations.
- **Secrets values.**  Only the *Vault-bootstrap* secrets live
  in the vendor's secret store.  The application's runtime
  secrets are fetched by the resolver from Vault.

## Multi-vendor support: same workflow, different shims

A team supporting both GitLab CI and GitHub Actions has two shim
files in the repository:

```
project-root/
├── .gitlab-ci.yml
├── .github/
│   └── workflows/
│       └── integration.yml
├── Jenkinsfile               # if also using Jenkins
├── config/
│   ├── workflows/
│   │   ├── integration.yaml          ← portable
│   │   └── nightly.yaml              ← portable
│   ├── profiles/
│   │   ├── integration-ci.yaml       ← portable
│   │   └── nightly-full-sweep.yaml   ← portable
│   └── ...                            ← all portable
└── ...
```

The vendor-specific files are short.  The portable files (under
`config/`) are long and information-dense.  The ratio is the
discipline's most visible operational property.

When the team migrates entirely off GitLab to GitHub Actions, the
`.gitlab-ci.yml` is deleted; everything else is unchanged.

## Operational concerns common to all vendors

Five concerns recur across vendors.  Each has a per-vendor
mechanism, but the *discipline* is the same.

### Artifact handling

The taskplus workflow writes outputs to `.output/` (the
chapter-7 runtime configuration's `paths.output` field).  The
vendor's job uploads the directory as an artifact when the job
ends.  Per-vendor:

- GitLab: `artifacts: paths: [.output/]`.
- GitHub Actions: `actions/upload-artifact@v4` with `path:
  .output/`.
- Jenkins: `archiveArtifacts artifacts: '.output/**'`.

When the test fails, the chapter-16 capture artifact
(`failure-snapshot.json`) is included alongside `.output/`.  The
developer downloads the failure snapshot and replays it
locally per chapter 16.

### Parallel matrix builds

When the workflow needs to run against multiple landscapes (e.g.,
staging-east *and* staging-west), the vendor's matrix-build
feature handles the fan-out.  The matrix variable becomes the
profile name:

```yaml
# GitLab CI
integration-multi:
  parallel:
    matrix:
      - PROFILE: [staging-east-ci, staging-west-ci]
  variables:
    AM_PROFILE: $PROFILE
  script:
    - ccai testbase test run --workflow config/workflows/integration.yaml
```

The same workflow runs twice in parallel, once with
`AM_PROFILE=staging-east-ci` and once with
`AM_PROFILE=staging-west-ci`.  No code change in the workflow;
two profile files declare the per-region differences (per chapter
14 and chapter 17).

GitHub Actions has `strategy.matrix:`; Jenkins has `parallel { ... }`
or `matrix { ... }` blocks.  The shape is consistent.

### Retries

When a CI vendor's *job* needs to be retryable (because of
infrastructure flakiness, not test failures), each vendor exposes
a retry mechanism:

- GitLab: `retry: max: 2 when: [runner_system_failure]`.
- GitHub Actions: there isn't one built in; teams use a
  third-party action like `nick-fields/retry`.
- Jenkins: `retry(2) { ... }` block.

The taskplus workflow's *internal* retry concerns are handled by
the chapter-23 `Retry` wrapper task — the Aspects pattern again,
applied to per-task flakiness rather than per-job infrastructure
flakiness.

### Caching

CI vendors offer caching for the dependency-install step.  The
runner image's pre-installed toolchain is the *primary* cache
(it doesn't change between runs); the project's poetry venv is
the *secondary* cache.

- GitLab: `cache: paths: [.venv/]`.
- GitHub Actions: `actions/cache@v4` with the venv path.
- Jenkins: `stash`/`unstash` or external cache plugins.

Caching is a vendor-specific optimisation.  The workflow's
behaviour is unchanged with or without caching; caching only
affects job latency.

### Notifications

Slack/email notifications on pipeline failure are a vendor-side
concern.  Each vendor has its own integration; the workflow
itself does not need to know.

When the workflow *itself* needs to send notifications during
execution (a "deploy started" message, a "smoke tests passed"
message), that's a *task* in the workflow — typically a
`RunCommand` calling a notification CLI or an `EmbeddedPython`
calling a webhook.  Chapter 23's extension recipe applies.

## Vendor migration: a worked example

A team currently on Jenkins decides to migrate to GitHub Actions.
The discipline makes the migration concrete and bounded.

### Day 0 — current state

Repository contents:

```
project-root/
├── Jenkinsfile               (33 lines, the shim)
├── config/
│   └── ... (all the typed configurations)
└── ...
```

Jenkins runs the integration suite on every PR via a webhook.

### Day 1 — add the GitHub Actions shim

A developer adds `.github/workflows/integration.yml` with the
GitHub Actions shape from earlier in the chapter.  The Jenkins
file remains unchanged.

```
project-root/
├── Jenkinsfile               (33 lines, unchanged)
├── .github/
│   └── workflows/
│       └── integration.yml   (43 lines, NEW)
├── config/
│   └── ... (unchanged)
└── ...
```

The first PR after this commit runs *both* CI systems against the
change.  The team uses this period to validate that GitHub
Actions produces the same results Jenkins does.

### Day 14 — disable Jenkins

The team disables the Jenkins webhook.  The `Jenkinsfile` remains
in the repository for now, in case rollback is needed.

### Day 30 — delete the Jenkins shim

After thirty days of clean GitHub Actions runs, the team deletes
`Jenkinsfile`.  The repository is now GitHub-only.

```
project-root/
├── .github/
│   └── workflows/
│       └── integration.yml   (43 lines, the only shim)
├── config/
│   └── ... (still unchanged)
└── ...
```

The `config/` directory was never touched during the migration.
Every typed configuration file, every workflow YAML, every
profile, every credential reference — none changed.  The
migration was confined to the vendor-specific shim.

This is the operational payoff the volume has been promising.
Vendor migration as a thirty-day project rather than a multi-
quarter rewrite is what the discipline delivers.

## Summary

- The CI vendor's YAML has **exactly two responsibilities**:
  trigger the workflow and provide a runner image.  Everything
  else is the taskplus workflow's responsibility.
- The taskplus invocation in every vendor's shim is one line:
  `ccai testbase test run --workflow <file>`, with
  `AM_PROFILE` set in the environment.
- **Three full vendor shims** (GitLab CI, GitHub Actions,
  Jenkins) are shown end-to-end; each is 28–43 lines.  The
  *structure* is identical across vendors; only the syntax
  differs.
- **What stays in vendor YAML**: trigger logic, runner image,
  toolchain bootstrap, vendor-specific Vault-bootstrap secrets.
  **What does not**: configuration content, imperative
  provisioning, step ordering of test phases, cross-environment
  branching, application secret values.
- **Operational concerns** — artifact handling, parallel
  matrix builds, retries, caching, notifications — each have
  per-vendor mechanisms with consistent shapes.  The discipline
  is the same; the syntax is per-vendor.
- **Multi-vendor support** is straightforward: one shim per
  vendor; the portable workflow + configurations under
  `config/` are unchanged across all vendors.
- **Vendor migration** is a 30-day project rather than a
  multi-quarter rewrite when the discipline holds.  The shims
  change; everything in `config/` does not.

The next chapter develops the bridges to other layers of the
ccai portfolio — orchestration → distributed test job (Volume
2), orchestration → sysman feature (Volume 4), orchestration →
provisioning (Volume 6).  Where this chapter showed the
*outward* wiring (CI vendor), the next chapter shows the
*downward* wiring (the layers below the orchestration that the
workflow's tasks invoke).
