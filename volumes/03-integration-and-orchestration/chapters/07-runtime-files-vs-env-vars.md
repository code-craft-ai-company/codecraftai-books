# Chapter 7 — Runtime Files vs Environment Variables

> *Configuration files allow arbitrary parameterisation of runtime
> behaviour without modifying the automation plumbing code.  This is
> the central engineering claim of the volume.  Every other chapter
> builds on it.*

This is the thesis chapter.  The chapters in Parts II and III have
introduced the configuration categories one by one and shown what
each looks like as a YAML file.  This chapter steps back and explains
*why* the discipline is expressed as files rather than as
environment variables.

The argument is not that environment variables are bad.  Environment
variables are a primitive of operating systems and process trees;
they are useful, well-understood, and unavoidable in any serious
deployment.  The argument is that environment variables are the
wrong place to put **the variation that distinguishes one run from
another**.  Variation belongs in configuration files.  The reason has
nothing to do with aesthetics and everything to do with *the cost of
changing the plumbing that runs the code*.

## The boundary between plumbing and content

Every automation pipeline has two parts:

- **Plumbing.**  The code that reads configuration, the Dockerfile
  that builds the image, the docker-compose file that wires the
  services, the CI YAML that runs the build, the Kubernetes manifest
  that deploys the result, the developer's `.env` file.  The plumbing
  is the *machinery* that runs the code.  It is read by build
  systems, by container runtimes, by orchestrators.  It is owned by
  multiple teams.  It changes slowly.
- **Content.**  The actual values that distinguish one run from
  another.  Log levels.  Retry counts.  Hostnames.  Database names.
  Timeouts.  Parallelism settings.  Feature flags.  Test scopes.
  Content is the *parameters* the machinery runs with.  It is read
  by the application code at startup.  It is owned by the engineers
  who write the code that consumes it.  It changes fast.

The discipline of this volume is: **the boundary between plumbing
and content should fall in a place where the plumbing is small and
fixed; the content carries all the variation.**

When the boundary is in the wrong place — when content leaks into
plumbing — every new behaviour knob requires a plumbing change.  The
plumbing change is expensive because the plumbing spans repos,
teams, formats, and deployment targets.  When the boundary is in the
right place, every new behaviour knob is a single-file change to
content.

This is not a stylistic preference.  It is a scaling property.  A
team adding its tenth behaviour knob in a small project may not
notice the difference.  A team adding its hundredth knob in a large
project lives the difference daily.

## Why the boundary matters: the plumbing tax

Suppose you want to add a new behaviour knob to an automation
pipeline.  Call it `RETRY_LIMIT`: the number of times the integration
job should retry a flaky test before failing.

Adding `RETRY_LIMIT` as an environment variable is the path of least
resistance for the engineer adding it.  It is *not* the path of least
total cost across the system.  Walk through what actually has to
change:

### Layer 1 — the application code

The code that runs the integration job has to read the variable:

```python
retry_limit = int(os.environ.get("RETRY_LIMIT", "3"))
```

This is the cheap layer.  It is also the layer the engineer
remembers.

### Layer 2 — the Dockerfile

If the integration job runs inside a container, the Dockerfile has
to declare the variable so the build records it as part of the image
contract:

```dockerfile
# Dockerfile
ENV RETRY_LIMIT=3
```

Or — more often — the Dockerfile has to *not* declare it, and
instead the runtime is expected to inject it.  Either way, someone
on the container team has to know the variable exists.

### Layer 3 — the docker-compose file

If the developer-facing equivalent of the pipeline runs through
docker-compose, the compose file has to pass the variable through:

```yaml
# docker-compose.yaml
services:
  integration-runner:
    environment:
      RETRY_LIMIT: "${RETRY_LIMIT:-3}"
```

Now the variable exists in three repositories: the application code,
the container build, and the local-dev compose stack.

### Layer 4 — the CI vendor's YAML

The CI pipeline that runs the integration job has to set the
variable explicitly, or has to inherit it from the vendor's secret
store:

```yaml
# .gitlab-ci.yml
integration:
  variables:
    RETRY_LIMIT: "5"
  script:
    - docker run -e RETRY_LIMIT runner-image:latest integration-job
```

If the team uses GitHub Actions and Jenkins as well, the same
variable has to be plumbed through each of them.  Every CI vendor
has its own YAML for environment variables and its own visibility
rules.

### Layer 5 — the Kubernetes manifest

If the integration job is deployed to a Kubernetes cluster — or if
its container is reused by a long-running service that does the same
work — the deployment manifest has to expose the variable:

```yaml
# integration-job.yaml
spec:
  template:
    spec:
      containers:
        - name: runner
          env:
            - name: RETRY_LIMIT
              value: "5"
```

### Layer 6 — the developer's local environment

The developer who is going to debug the integration job locally
needs the variable on their workstation:

```bash
# ~/.zshrc or .envrc or .env
export RETRY_LIMIT=3
```

And there has to be documentation in the project's README explaining
that this is required.

### What it cost to add one knob

Six layers.  Six different file formats.  Three to five
repositories.  Two to four teams (application, container, CI,
infrastructure).  And the engineer who added the variable has now
acquired an obligation: every future change to `RETRY_LIMIT` — its
default, its name, its removal — requires another walk through all
six layers.

This is the plumbing tax.  It is paid once per environment variable.
A pipeline with fifty environment variables has paid the tax fifty
times, and the tax is *cumulative*: when an engineer wants to
understand which environment variables a job actually reads, the
authoritative answer requires reading code in five repositories.

## The fixed-interface principle

The discipline of this volume is to charge the plumbing tax exactly
once and never pay it again.

The plumbing layers above all reference a *small, fixed* set of
environment variables.  These variables do not change when behaviour
changes.  They name the *configurations* the runtime should load:

- `AM_RUNTIME_NAMES` — which runtime profile to use.
- `AM_LANDSCAPE_NAMES` — which landscape to use.
- `AM_CREDENTIAL_NAMES` — which credential set to use.
- `AM_TOPOLOGY_NAMES` — which topology to use.
- `AM_DATAPROFILE_NAMES` — which data profile set to use.

These five environment variables are the fixed interface between the
plumbing and the content.  They are plumbed through the code, the
Dockerfile, the compose file, the CI YAML, the Kubernetes manifest,
and the developer's environment exactly once.  After that, **the
plumbing is finished**.

When `RETRY_LIMIT` needs to exist, it does not become a sixth
environment variable.  It becomes a field inside the runtime
configuration file:

```yaml
# runtime-staging.yaml
retry_limit: 5
logging:
  levels:
    console: WARNING
timeouts:
  default_seconds: 30
```

To add `RETRY_LIMIT`, the engineer:

1. Adds the field to the runtime YAML for the environments where it
   should be set.
2. Adds the read in the application code.

That's two layers — application and config file — instead of six.
The Dockerfile does not change.  The compose file does not change.
The CI YAML does not change.  The Kubernetes manifest does not
change.  The developer's `.env` does not change.  None of them have
to know `RETRY_LIMIT` exists.

This is what *scale* means in the thesis.  Scale is not the number
of variables.  Scale is the number of layers each variable touches.

## What stays in environment variables: the fixed interface

A small set of environment variables earns its place in the plumbing
forever.  The discipline does not eliminate environment variables;
it *minimises* and *stabilises* them.

### Configuration selection

The five `AM_*_NAMES` variables above are the canonical example.
Each names a configuration set the runtime should load.  Their
*names* never change; their *values* differ between runs.  When a
new runtime profile is added (`runtime-staging-east-canary.yaml`),
nothing in the plumbing changes — only the value of
`AM_RUNTIME_NAMES`.

### Process identity

Variables that identify the running process for observability —
`AM_RUN_ID`, `AM_JOB_ID`, `AM_PIPELINE_ID` — also belong in the
plumbing.  They are read by logging, tracing, and metrics machinery
that has its own fixed interface.

### Vendor-mandated bootstrapping

A few environment variables are required by external tooling and
cannot be replaced: `KUBECONFIG`, `AWS_REGION`, `LANG`,
`PYTHONPATH`.  These are the interface to *other* systems, not the
interface to your application's behaviour.  They are also the
plumbing.

Every other environment variable should be a candidate for migration
into a configuration file.  When you find yourself reaching for
`export` for the *content* of a run — log levels, retry counts,
hostnames, parallelism — you are paying the plumbing tax.

## What moves into configuration files: the content

Everything else.  Concretely, the runtime YAML in
[`codecraftai-testbase`](../../../codecraftai-testbase/source/testroots/report-tests/result-tests/v2/example2/runtime-declared.yaml)
— a real shipped file — has the following top-level keys:

```yaml
breakpoints: [testrun-start]
debugger: debugpy
logging:
  levels:
    console: INFO
    logfile: DEBUG
paths:
  output: /var/log/ccai
results-configuration:
  format: jsos
testroot: source/testroots/ccai
version: 1.0.0
```

A team that wants to switch on a new debugger, change a log level,
or alter the test root path edits the YAML.  Nothing else changes.
A team that wants to ship a *new* runtime profile — say, a
`runtime-staging-canary.yaml` — adds the file and references it by
name from `AM_RUNTIME_NAMES`.  Still nothing else changes.

The file format is arbitrary.  It can have any keys the application
code knows how to read.  The schema can grow without coordination
across repositories or teams.  This is what "arbitrary
parameterisation of runtime behaviour" means in the chapter's
opening claim.

## The boundary case: TESTBASE_MAX_EXAMPLES_CAP

The portfolio ships one well-known environment variable that
controls runtime behaviour:
[`TESTBASE_MAX_EXAMPLES_CAP`](../../../codecraftai-testbase/userguide/10-04-03-testing-frameworks-testbase.rst).
It caps the number of cases each property test will run.  Set
`TESTBASE_MAX_EXAMPLES_CAP=10` and a property declared with
`@given(constraints={"max_examples": 50})` runs only 10 cases.
Leave it unset and the property runs at full strength.

Why does the discipline tolerate this one environment variable?

Because it is *one knob* that controls *one well-defined dimension
of behaviour* — fast-CI vs full-sweep — and it has been plumbed
through the test framework, the CI YAML for several portfolio
projects, and the testbase documentation.  The tax has been paid;
the variable is stable; the value the variable controls is exposed
through a single read in `propertyrunner.py`.

This is the boundary case that proves the rule.  One knob via
environment variable scales fine.  Two knobs scales fine.  Twelve
knobs is where the tax compounds.  The discipline says: when you
catch yourself reaching for the second or third behaviour knob via
environment variable, stop and add a runtime configuration file
instead.  The very next knob would have been the one that crossed
the threshold.

In the long term, even `TESTBASE_MAX_EXAMPLES_CAP` will likely
migrate into a runtime configuration field
(`property_test_caps.max_examples`).  It will keep its environment-
variable form as a backward-compatible override, but the canonical
location will move into the runtime YAML.  The variable will then
be the boundary case in the other direction — a legacy environment
variable that survives because it was reasonable when it was added.

## A migration walk-through

Consider a CI pipeline with the following environment-variable
exports.  This is a composite of patterns common in real
maintained pipelines, not a contrived example:

```yaml
# .gitlab-ci.yml — before
integration:
  variables:
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
    LANDSCAPE: "staging-east"
    DB_HOST: "db-staging-east-1.internal"
    REDIS_HOST: "redis-staging-east-1.internal"
  script:
    - integration-runner $TEST_SUITE
```

Fourteen environment variables.  Each carries a small piece of
content.  Each was added at a different time by a different
engineer.  Each has been plumbed through the layers above.

The migration is mechanical.  Every variable except `LANDSCAPE` is
content.  `LANDSCAPE` is a configuration *selector* — it names a
landscape file — so it stays.  Every other variable becomes a key in
either the landscape file or a new runtime profile:

```yaml
# .gitlab-ci.yml — after
integration:
  variables:
    AM_RUNTIME_NAMES: "fast-ci"
    AM_LANDSCAPE_NAMES: "staging-east"
  script:
    - integration-runner $TEST_SUITE
```

```yaml
# config/runtime/fast-ci.yaml — new
logging:
  levels:
    console: INFO
    logfile: DEBUG
retry:
  limit: 3
  backoff_ms: 500
parallelism: 8
timeouts:
  default_seconds: 30
  db_seconds: 60
  api_seconds: 10
paths:
  output: /var/log/integration
debugger: none
breakpoints: []
```

```yaml
# config/landscape/staging-east.yaml — new (or updated)
environment:
  label: staging-east
infrastructure:
  services:
    - name: db
      hostname: db-staging-east-1.internal
    - name: redis
      hostname: redis-staging-east-1.internal
```

The CI YAML went from fourteen variables to two.  The two that
remain are stable: their *values* will change between runs and
profiles, but their *names* will never change again.

The application code that read `LOG_LEVEL` from the environment now
reads it from the resolved runtime configuration.  The resolution
machinery — `RuntimeVariables`, `RuntimeVariablesDesc`, and
`resolve_runtime_variables` — is owned by `ccai.thisapp` (it moved
there from the now-retired `codecraftai-runtime` package, whose modules
are thin re-export shims during the migration).  The resolved values
are exposed through the `RuntimeVariables` accessor set, which reads
from the application context that the runtime profile was merged into:

```python
# Before
log_level = os.environ.get("LOG_LEVEL", "INFO")

# After
from ccai.thisapp.runtimevariables import RuntimeVariables
log_level = RuntimeVariables.AM_LOG_LEVEL_CONSOLE
```

> *Two ownership levels, one variable surface.*  This rich resolution
> — runtime profile YAML merged into the application context, the
> `RuntimeVariables` accessor set, and the activation that lights up
> the well-known presence folder — belongs to the **integration**
> layer.  `codecraftai-integration-runtime` performs it during its
> [`initialize`](../../../codecraftai-integration-runtime/source/packages/ccai/integrationruntime/initialize.py)
> step (`resolve_runtime_variables` + `activate_profile_job`) and only
> `IntegratedTestJob` consumes it.  The standalone `testbase`
> test-execution core (the `TestJob` / `TestSequencer` / recorder path)
> does *not* import `ccai.thisapp`; it reads only its handful
> of core execution knobs (`AM_TESTROOT`, `AM_OUTPUT_DIRECTORY`,
> `AM_RUN_ID`, `AM_TESTRUN_TIMEOUT`, the `AM_DEBUG_*`/`AM_DIAGNOSTICS_*`
> family) straight from `os.environ` through the tiny dependency-free
> [`ccai.testbase.environment`](../../../codecraftai-testbase/source/packages/ccai/testbase/environment.py)
> module.  The integration layer sets those same environment variables,
> so the fixed-interface variable *names* are identical whether a job
> runs standalone or under the runtime — only the resolution machinery
> behind them differs.

The Dockerfile, the compose file, and the Kubernetes manifest are
all unaffected.  They knew how to pass `AM_RUNTIME_NAMES` and
`AM_LANDSCAPE_NAMES` through; they now pass them through with
different values.  No change to the plumbing.

When the team next wants to add a new behaviour knob — say, a flag
that controls whether the integration job uploads its results to
the central dashboard — the change is one line in the runtime YAML
and one line in the application code.  The plumbing tax does not
get charged.

## The migration pattern: retiring environment variables without breakage

A pipeline cannot migrate fourteen variables in one commit without
risk.  The pattern that works in practice is: introduce the
configuration file, support both forms in parallel for a release,
then remove the environment-variable form.

Use
[`override_variable_value`](../../../codecraftai-this-app/source/packages/ccai/thisapp/thisapplication.py)
on `ThisApplicationSingleton()` to wire each environment variable to
its corresponding runtime configuration field as a fallback.  Both the
singleton and the variable descriptors live in `ccai.thisapp` —
`ThisApplicationSingleton` in
[`ccai.thisapp.wellknown`](../../../codecraftai-this-app/source/packages/ccai/thisapp/wellknown.py)
and the `RuntimeVariablesDesc` descriptor set in
[`ccai.thisapp.runtimevariables`](../../../codecraftai-this-app/source/packages/ccai/thisapp/runtimevariables.py)
(formerly `ccai.runtime.runtimevariables`):

```python
from ccai.thisapp.wellknown import ThisApplicationSingleton
from ccai.thisapp.runtimevariables import (
    RuntimeVariables,
    RuntimeVariablesDesc,
    resolve_runtime_variables,
)


def _bootstrap_runtime_overrides() -> None:
    # Phase 1: resolve the runtime profile into the application context
    resolve_runtime_variables()

    # Phase 2: legacy environment variables override profile values
    if "LOG_LEVEL" in os.environ:
        ThisApplicationSingleton().override_variable_value(
            RuntimeVariablesDesc.AM_LOG_LEVEL_CONSOLE,
            os.environ["LOG_LEVEL"],
        )

    return
```

This is the migration shim.  During the transition release:

- New code reads from the runtime configuration.
- Pipelines that still set `LOG_LEVEL` continue to work — the shim
  promotes the env-var value into the resolved configuration.
- Pipelines that have migrated to the runtime profile work without
  the env var.

Once every pipeline has migrated, the shim is removed.  The
environment variable is gone.  No reach into the plumbing of
multiple repositories was required.

## What environment variables are still good for

The chapter has argued for *minimising* environment variables, not
eliminating them.  In a healthy automation system, the surviving
environment variables fall into three categories:

1. **Configuration selectors.**  The `AM_*_NAMES` family.  These
   name configurations rather than carrying configuration.
2. **Process identity.**  `AM_RUN_ID`, job IDs, pipeline IDs.  Read
   by observability machinery; not behaviour-controlling.
3. **External-system interfaces.**  `KUBECONFIG`, `AWS_REGION`,
   `PYTHONPATH`.  Required by tools the application embeds; the
   discipline does not extend to them.

A pipeline whose environment-variable surface fits in those three
categories is in a healthy state.  A pipeline whose
environment-variable surface includes log levels, retry counts,
hostnames, timeouts, debug flags, output paths, parallelism, or
feature flags has content in the wrong place.  The migration above
is how it gets fixed.

## What the rest of the volume does with this

The thesis of this chapter — *configuration files allow arbitrary
parameterisation of runtime behaviour without modifying the
automation plumbing code* — recurs throughout the rest of the
volume.

- Part III on credentials applies the same boundary: the
  *credential identifier* is plumbing; the *credential material* is
  content.  The plumbing references credentials by name; the
  content is fetched by the resolution layer.
- Part IV on profiles is the discipline of *composing* the content
  side.  A profile is a named bundle of runtime + landscape +
  topology + credentials + data-profile selections.  Switching
  profiles changes the plumbing's *values* (the `AM_*_NAMES`) but
  not the plumbing's *shape*.
- Part V on `taskplus` shows the orchestration layer playing the
  same role: the workflow YAML references jobs and tasks by *name*;
  the job-and-task definitions are content.  Adding a new task type
  does not require modifying the workflow grammar.

The discipline scales because the plumbing is small and the content
is unbounded.

## Summary

- Every automation pipeline has a *plumbing* layer (code, container,
  compose, CI YAML, manifests, dev env) and a *content* layer (the
  values that distinguish one run from another).
- When variation lives in environment variables, every new
  behaviour knob is a plumbing change across six layers, multiple
  repos, and multiple teams.  This is the plumbing tax.
- The discipline keeps the environment-variable interface small and
  fixed: a handful of `AM_*_NAMES` that select configurations, plus
  process identity, plus external-system interfaces.  Everything
  else moves into configuration files.
- A configuration file can be edited freely — new fields, new
  profiles, new schema additions — without touching any plumbing.
  This is what "arbitrary parameterisation of runtime behaviour
  without modifying the automation plumbing code" means.
- `TESTBASE_MAX_EXAMPLES_CAP` is the boundary case: one
  environment-variable knob is fine; many are the failure mode the
  discipline prevents.
- The migration pattern is to introduce the runtime profile,
  support both forms via `override_variable_value` for a transition
  release, then remove the environment variables.
- The thesis recurs throughout the volume: keep the plumbing small
  and fixed; let the content carry the variation.
