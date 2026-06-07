# Chapter 21 — Passing Information to Jobs

> *The orchestration knows which configurations are in scope.  The
> job knows how to use them.  The seam between the two is the
> chapter that ties Part V back to the discipline of Parts II–IV.*

The previous chapter walked the workflow YAML grammar.  This
chapter walks the *contract* between the orchestration that the
YAML describes and the *jobs* the orchestration invokes.  The
contract is small — four communication channels, each with a
specific purpose — and it is the chapter where chapter 7's
plumbing-vs-content thesis pays off most concretely.

The orchestration's job is *not* to package its content into the
job's environment.  The orchestration's job is to package the
*names* of configurations the job should load, hand those names
across the seam, and trust the job to resolve them through the
shipped configuration machinery.  The seam is small; the trust is
mechanical.

## The four channels

Four mechanisms move information from an orchestration to a job:

| Channel | Direction | Lifetime | Use |
|---|---|---|---|
| **1. Environment exports** | orchestration → job process | process lifetime | Configuration *selectors* (`AM_*_NAMES`); process identity (run ID, job ID) |
| **2. Workflow parameters** | orchestration → tasks within the orchestration | workflow lifetime | Flow-level values that vary per run (branch, build ID, feature flags) |
| **3. Flowstate** | task ↔ task within the orchestration | workflow lifetime | Inter-task ad-hoc state |
| **4. Task return codes + side effects** | task → orchestration | per task | Success/failure, logs, artifact paths |

The four channels are layered.  Channel 1 (environment exports)
crosses the *process boundary* — the orchestration's process to a
subprocess job's process.  Channels 2 and 3 (parameters and
flowstate) live *inside* the orchestration's process — they pass
between tasks within the same workflow.  Channel 4 is the tasks'
outbound channel — values they return to the engine, side
effects they produce on disk or over the network.

The discipline is to use the right channel for each kind of
information.  The volume's thesis is most concretely this
sentence: **environment exports carry only configuration
selectors and process identity; *content* travels through
parameters, flowstate, and resolved configurations.**

## Channel 1 — Environment exports

The first and narrowest channel.  The
[`Workflow`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py)
class's `environment:` field becomes a set of `os.environ`
exports applied before the flow runs:

```yaml
environment:
    AM_PROFILE: integration-ci
    AM_LANDSCAPE_NAMES: staging-east
    AM_RUNTIME_NAMES: integration-ci
    AM_CREDENTIAL_NAMES: integration-ci
    AM_DATAPROFILE_NAMES: integration-ci
    AM_TOPOLOGY_NAMES: staging-east-leader-follower
```

Six entries.  Each names a *configuration*.  Each has a name that
will not change (the `AM_*_NAMES` family is the fixed-interface
plumbing from chapter 7).  Each value identifies which named
configuration is in scope for this orchestration run.

When the orchestration spawns a subprocess job (a `RunCommand`
shell, a child Python process invoked through `EmbeddedPython`,
a containerised job started by a custom task), the subprocess
inherits the parent's `os.environ` — so the subprocess sees the
same six entries.  At its own startup the subprocess uses
[`automojo-configuration`](../../../automojo-configuration/source/packages/automojo/configuration/configurationmaps.py)
to resolve each `AM_*_NAMES` value into the corresponding loaded
configuration.

The result: the orchestration and the job share *configuration
identity*.  When the orchestration is running with
`landscape: staging-east`, every job the orchestration spawns
runs against the same staging-east landscape — without the
orchestration having to pass the landscape's content through any
other channel.

### What does *not* go in `environment:`

Three categories of value that *do not* belong in the workflow's
`environment:` block:

- **Configuration content.**  A workflow that exports
  `LOG_LEVEL=DEBUG` is putting content in plumbing.  The right
  answer is to put `LOG_LEVEL` in a runtime configuration
  (chapter 7); the workflow's `environment:` exports
  `AM_RUNTIME_NAMES=debug-runtime` to select that runtime, not
  the value itself.
- **Secrets.**  Credentials never travel as `environment:`
  values.  The discipline from chapter 12: the workflow exports
  the *credential identifier* (or, more precisely, exports the
  `AM_CREDENTIAL_NAMES` selector that names the credential set);
  the credential resolver materialises the secret value at the
  job's startup using the local backend.
- **Per-task variation.**  When task A needs to know about task
  B's outcome, the right channel is flowstate (channel 3), not an
  environment-variable export.  Setting an env var to communicate
  between tasks is the chapter-2 conditional-thicket failure
  mode wearing yet another costume.

The discipline of channel 1 is fixed and small: configuration
selectors plus process identity.  Anything else lives in another
channel.

## Channel 2 — Workflow parameters

The workflow's `parameters:` block is a typed dict that flows to
*every* task's `perform_task` as the second argument:

```yaml
parameters:
    branch: main
    build_id: nightly-2026-04-30
    feature_flags:
        enable_new_caching: true
        enable_metrics_v2: false
    cleanup_after_run: true
```

Tasks consume parameters by reading the dict in `perform_task`:

```python
def perform_task(self, environment, parameters, flowstate):
    branch = parameters.get("branch", "main")
    build_id = parameters["build_id"]
    flags = parameters.get("feature_flags", {})

    # ... do work ...

    return 0
```

Three properties matter:

### Flow-scope, not process-scope

Parameters are visible to every task in the workflow, but they
do *not* automatically reach subprocess jobs.  When a task spawns
a subprocess (via `RunCommand`, for example), the subprocess
inherits `os.environ` but does *not* inherit `parameters` —
because parameters live in the orchestration's Python memory,
not in the process environment.

When a subprocess job needs a parameter value, the parent task
explicitly passes it.  Most commonly this happens by *name* —
the parameter holds a configuration identifier that the
subprocess's startup machinery resolves.  Sometimes it happens
by *value* — the parent task interpolates the parameter into the
subprocess's command line:

```yaml
- task: Tag deployment
  tasktype: automojo.taskplus.tasks.runcommand@RunCommand
  command: "kubectl set image deployment/api api=registry/api:${BUILD_ID}"
```

The shipped `RunCommand` does *not* substitute `${BUILD_ID}`
itself — that interpolation happens in the shell.  For parameter
substitution the workflow author can either:

- Write a custom task that does the substitution explicitly
  (chapter 23 develops the extension recipe).
- Use `EmbeddedPython` to construct the command string from
  `parameters` and write it back into `flowstate` for a
  subsequent `RunCommand` to pick up.
- Use shell-level variable expansion through a `ShellScript`
  whose first lines establish the variables.

### Read-only by convention

Tasks should *read* parameters, not modify them.  The shipped
`Task.perform_task` does not enforce this — Python doesn't have
a "frozen dict" passed by value — but the discipline is uniform:
parameters are workflow-level values set at orchestration start
and unchanged for the duration.

When a task needs to *produce* a value for downstream tasks,
the right channel is flowstate (channel 3).  Mutating
parameters defeats the workflow-author's mental model that the
parameters are stable.

### Type-as-data

Unlike environment-variable exports, parameters can carry rich
data — nested dicts, lists, booleans, integers.  YAML's type
fidelity is preserved through the parser:

```yaml
parameters:
    feature_flags:
        enable_new_caching: true        # bool, not the string "true"
    retry_attempts: 3                   # int, not the string "3"
    target_hosts:                       # list, not a comma-separated string
        - api-1.staging.internal
        - api-2.staging.internal
        - api-3.staging.internal
```

Tasks consume the typed values directly, no parsing required.
This is one of the most concrete payoffs of putting variation in
content rather than plumbing — environment-variable values are
always strings, parameter values are whatever YAML says they are.

## Channel 3 — Flowstate

The third argument to `perform_task` is `flowstate`, a per-run
mutable dictionary that tasks read *and write*:

```python
def perform_task(self, environment, parameters, flowstate):
    # Read state produced by an earlier task
    artifact_path = flowstate.get("build_artifact_path")

    # ... do work ...

    # Write state for a later task to consume
    flowstate["smoke_test_passed"] = True

    return 0
```

Flowstate is the *inter-task communication channel*.  Use cases:

- A build task produces an artifact at a path; a deploy task
  later in the workflow needs the path.  The build task writes
  it to `flowstate["build_artifact_path"]`; the deploy task
  reads it.
- A smoke-test task writes summary metrics; a notification task
  later reads them and posts them to Slack.
- A cleanup-decision task computes whether teardown should
  happen; the teardown task gates on it.

Flowstate is *ad-hoc*.  No schema, no type checking, no
contractual ownership of keys.  The discipline:

- **Use namespaced keys.**  Prefer `build.artifact_path` over
  `artifact_path`; prefer `smoke.passed` over `passed`.  The
  namespace makes ownership obvious and reduces collision when
  the workflow grows.
- **Document key contracts close to the producing task.**  When
  a custom task writes a flowstate key, the task's class
  docstring should list the keys it produces.  When a task reads
  a flowstate key, the docstring should name which task is
  expected to produce it.
- **Keep the dict small and ad-hoc.**  Flowstate is for
  workflow-internal coordination, not for replacing data stores.
  When the value is large (gigabytes of build output, thousands
  of test results), the right pattern is to write to disk and
  put the *path* in flowstate.

Three things flowstate is *not*:

- **Not persistent.**  When the workflow ends, flowstate is
  gone.  Subsequent runs start with an empty flowstate.
- **Not visible across orchestrations.**  Two parallel workflows
  running on the same machine each have their own flowstate.
- **Not visible to subprocess jobs by default.**  When a task
  spawns a subprocess, the subprocess does not inherit
  flowstate.  Cross-process state requires explicit passing
  (typically by writing to a file the subprocess reads).

## Channel 4 — Task return codes and side effects

Tasks communicate *outward* — to the engine, to subsequent
tasks, to observability infrastructure — through a small set of
well-known channels.

### Return codes

`perform_task` returns an integer.  Zero is success; non-zero is
failure.  The engine reads the return code and uses it to drive
flow control (chapter 22):

- A non-zero return triggers the task's `onfailure` jump, if
  declared.
- A zero return triggers the task's `onsuccess` jump, if
  declared.
- Otherwise execution proceeds to the next node in sequence.

The integer can carry semantic information beyond
zero/non-zero — a `RunCommand` task returns the wrapped shell's
exit code, so a failure with code 42 distinguishes from a
failure with code 1.  The engine's flow control treats both as
"failure" but the engineer reading the log sees the specific
code.

### Logger output

Every task is constructed with a logger reference (chapter 18).
Tasks emit log lines through the logger; the engine routes the
output to the workflow's configured destinations (console,
logfile, structured-log sinks).

The discipline from chapter 11 applies: tasks never log
credentials, never log resolved secrets, never log values that
could leak through the audit boundary.  The
`__repr__` redaction and the logging filter are the defences.

### Filesystem side effects

Tasks routinely produce files — build artifacts, test reports,
captured profiles, logs.  The convention in
[`automojo-taskplus`](../../../automojo-taskplus/) is that every
task that produces files writes them under the configured
output directory (the runtime configuration's `paths.output`
field).  The output directory is per-orchestration-run; tasks
write into it freely; subsequent tasks (or post-orchestration
tooling) read from it.

Filesystem side effects are *not* part of the four-channel
contract directly — they're a side channel.  But they're the
typical answer to "how does task A pass a 5-megabyte artifact
to task B?": task A writes to a file, task A puts the path in
flowstate, task B reads the path from flowstate and opens the
file.  The pattern works without any framework support beyond
the shipped channels.

### Custom side channels

For richer cross-task communication — pub/sub between tasks,
shared streaming buffers, cross-process state — custom tasks
can establish their own side channels.  Common patterns:

- A task that spawns a long-running subprocess writes a PID
  file, and a later task reads the PID file to attach.
- A task that produces telemetry writes structured events to a
  named pipe, and a metrics-collection task reads from the
  pipe.
- A task that reserves a resource (a port, a slot in a queue)
  writes the reservation to a lock file, and the cleanup task
  releases it.

These patterns are *outside* the four-channel framework
deliberately.  The framework gives you the channels you need
for normal orchestration; specialised needs are the workflow
author's responsibility to design.

## The bootstrap-name → resolve-config flow

The chapter's central engineering claim: an orchestration
spawning a subprocess job hands across only *names*; the job
*resolves* the names into structured configurations using the
shipped machinery.

Walk the flow end-to-end.

### Step 1 — Orchestration startup

The orchestration's process starts.  It reads the workflow YAML.
The workflow's `environment:` block exports the `AM_*_NAMES`
family into `os.environ`.  At this point the orchestration
itself has *not* loaded any configurations — it has only
established which configurations are in scope.

```python
# At orchestration startup, conceptually:
for name, value in workflow.environment.items():
    os.environ[name] = value

# os.environ now contains:
#   AM_PROFILE=integration-ci
#   AM_LANDSCAPE_NAMES=staging-east
#   AM_RUNTIME_NAMES=integration-ci
#   AM_CREDENTIAL_NAMES=integration-ci
#   AM_DATAPROFILE_NAMES=integration-ci
#   AM_TOPOLOGY_NAMES=staging-east-leader-follower
```

### Step 2 — Orchestration loads configurations

The orchestration's runtime calls `establish_configuration_settings()`
and `resolve_configuration_variables()` from
`automojo-configuration`.  These functions:

- Read the `AM_*_NAMES` env vars set in step 1.
- Locate the named configuration files on disk.
- Load and parse them.
- Insert the resulting `MergeMap`s into the application context
  at the well-known `ContextPaths`.

After step 2, the orchestration's process holds typed
configurations.  The orchestration's tasks consume them through
the runtime API (chapter 8's `DataProfileManagerSingleton`,
chapter 9's `CredentialManagerSingleton`, etc.).

### Step 2.5 — The verification step (the load-bearing precondition)

There is a step between "configurations loaded" and "the first
task runs" that the framework treats as a *precondition*, not
a feature: **verification of the resolved Landscape and
Topology against the actual environment.**

- **Landscape verification.**  The framework walks every
  resource declared in the resolved Landscape and confirms it
  is reachable: every host is contactable at its declared
  address with its declared protocol, every named credential
  resolves, every named service in the `services` section is
  alive at the address claimed.  Failure here means the CI
  system handed the orchestration a Landscape that doesn't
  match reality — wrong before the first test runs.
- **Topology verification.**  The framework walks the resolved
  Topology and confirms the resources are *configured as the
  Topology declares*.  If the Topology says "Nodes A, B, C are
  cluster members and Node D is a spare," verification confirms
  the cluster membership and the spare assignment match.  If
  the Topology says a sound system has three players in
  master/slave order, verification confirms the wiring.  The
  canonical `clusters` shape (chapter 6) is verified by the
  framework directly; user-defined Topology sections are
  verified by the user's `Topology` subclass overriding
  `validate_topology()`.

If either verification fails, the orchestration refuses to
proceed.  This is **the** load-bearing contract that makes
distributed automation deterministic.  Volume 2 chapter 2's
single-actor determinism model — one driver, observers
attached in tandem — depends on the *environment* matching
the *expectation* before the actor starts acting.  An
orchestration that ran tests against an unverified Landscape
would be running against an environment whose state is not
what the tests assume; a test that says "modify cluster member
A" against an environment where A is actually a spare would
produce non-deterministic outcomes.  Verification removes that
class of failure before it can happen.

The verification contract also explains why the
`InteropLibraryTestJob` from Volume 2 chapter 4 fires
`fire_attach_to_environment` *before* the first test — that
hook is where the test job runs the verification on the
landscape it was handed.  This volume's orchestration layer
hands the job the *names*; the job realises the names into
typed configurations *and verifies* them; only then does the
test sequencer start traversing tests.

Step 2 produced typed configurations; step 2.5 made them
*verified* typed configurations.  Step 3 (below) is the first
moment the orchestration begins doing work in the realised
environment.

### Step 3 — Task spawns a subprocess

A `RunCommand` task spawns a subprocess.  The subprocess inherits
the parent's `os.environ` automatically — so the subprocess sees
the same `AM_*_NAMES` values the orchestration set in step 1.

The subprocess does *not* inherit:

- The orchestration's loaded configurations (those live in
  Python memory, not the environment).
- The orchestration's parameters dict (Python memory).
- The orchestration's flowstate dict (Python memory).
- The orchestration's resolved credentials (Python memory; and
  by design, since secret material should not cross the process
  boundary as values).

The subprocess *does* inherit:

- The orchestration's `os.environ` (configuration selectors,
  process identity, plus whatever else the parent had set).
- The orchestration's working directory.
- The orchestration's stdin/stdout/stderr (or whatever the
  parent connected).

### Step 4 — Subprocess loads configurations

The subprocess starts an `automojo-configuration`-aware
application.  At its own startup the subprocess reads the
`AM_*_NAMES` env vars (the same ones the parent had), loads the
named configuration files, and produces its own typed
configurations.

The result: the subprocess has the *same configurations* the
parent had — but loaded *independently*, in the subprocess's
own memory.  No serialisation, no IPC, no shared memory.  The
configurations are reproduced through the same machinery on both
sides.

### Step 5 — Subprocess runs

The subprocess's code reads typed configurations through the
runtime API (the same calls the orchestration made in step 2).
When it needs the staging-api endpoint, it asks the data-profile
manager.  When it needs a credential, it asks the credential
manager.  When it needs a runtime knob, it asks the runtime.

The code is identical to what runs in the orchestration's own
process, in local-dev, in CI, in staging.  The environment
provides the *names*; the configuration machinery provides the
*content*.  This is the chapter-13 four-environment ladder
applied at the orchestration-to-job seam.

### Which test job the orchestration spawns

When the subprocess the orchestration spawns is a *test job*,
there are now two shapes it can take — and the `AM_*_NAMES`
selector contract is identical for both.

- **Plain `testbase` `TestJob`.**  The near-standalone job from
  Volume 1.  It runs *without activation*: no presence folder, no
  `RuntimeVariables`, no dependency on `automojo-this-app`.  Its
  default recorder is `StdoutResultRecorder` — it prints the run
  summary to stdout and (when an output directory is supplied)
  reuses the JSOS/JSON serialisation so `testrun_results_stream.jsos`
  and `testrun_summary.json` are still written for the `testbase
  agents` CLI to consume, but it *skips* the HTML render.  This is
  the right shape for a unit-style job invoked by a `RunCommand`
  that simply needs a pass/fail and a stdout transcript.  It is
  launched through testbase's standalone
  [`generic_test_entrypoint`](../../../automojo-testbase/source/packages/automojo/testbase/entrypoints.py).
- **`IntegratedTestJob`.**  The integration-tier job from
  [`automojo-integration-runtime`](../../../automojo-integration-runtime/source/packages/automojo/integrationruntime/integratedtestjob.py).
  It *extends* `TestJob` and overrides exactly three provider
  hooks: `_create_output_context()` returns a
  `RuntimeOutputContext` (results land under the activated app's
  well-known presence folder — activation-gated; it raises
  `SemanticError` unless the process was activated as a job),
  `_create_run_metadata()` returns a `RuntimeRunMetadata`
  (build / pipeline / job sourced from `RuntimeVariables` instead
  of the standalone `(unknown)` placeholders), and
  `_create_recorder()` returns a `JsonResultRecorder` (JSOS
  streaming *plus* the rendered HTML summary).  It is launched
  through
  [`integrated_test_entrypoint`](../../../automojo-integration-runtime/source/packages/automojo/integrationruntime/entrypoints.py),
  which first performs this-app activation (establish app persona,
  `resolve_runtime_variables`, `activate_profile_job(JobType.TestRun)`)
  and *then* delegates to the same standalone
  `generic_test_entrypoint`.

The two jobs differ only in *result recording and run
metadata* — the presence-folder results-directory layout,
JSOS-to-disk-plus-HTML, and CI/build metadata are the
responsibility of `IntegratedTestJob`, not of plain `testbase`.
The configuration-selector seam this chapter is about is
unchanged: both jobs read the same inherited `AM_*_NAMES` values
and resolve the same configurations independently.

`IntegratedTestJob` does not have to be named explicitly.  When
`automojo-integration-runtime` is installed, its
`IntegrationRuntimeDefaultTypeFactory`
([`integrationruntimedefaulttypefactory.py`](../../../automojo-integration-runtime/source/packages/automojo/integrationruntime/integrationruntimedefaulttypefactory.py))
registers through `automojo-extension`'s factory protocol so the
super-factory search path resolves `DefaultIntegratedTestJob` as
the integration-tier default job type — sitting above testbase's
`DefaultTestJob` and below testplus's distributed default.  An
orchestration that activates the integration runtime therefore
gets the integration recording behaviour by default, without the
workflow author hand-wiring the job class.

## What the job sees vs what the orchestration knew

A subprocess job does not have access to the orchestration's
parameters or flowstate.  The discipline acknowledges this by
recommending three patterns:

### Make values configurations, not parameters

When a value is genuinely workflow-time-stable but needs to
reach a subprocess, the right home is in a runtime configuration
field:

```yaml
# config/runtime/runtime-integration-ci.yaml
build:
    target_branch: main
    feature_flags:
        enable_new_caching: true
```

The subprocess reads `runtime["build"]["target_branch"]` through
the runtime API.  The same field is the same field in every
process — orchestration and subprocess alike.

### Pass values through command-line arguments

When a value is genuinely per-task and the subprocess accepts
arguments, the parent task interpolates the value into the
command line:

```python
# A custom task that knows about parameters
def perform_task(self, environment, parameters, flowstate):
    build_id = parameters["build_id"]
    cmd = f"./scripts/tag-deployment.sh {build_id}"
    return run_subprocess(cmd)
```

The subprocess reads `argv[1]`; the parent has packaged the
value at the call site.  Explicit, traceable, and consistent
with how non-orchestration command-line tools work.

### Write to a file and pass the path

When a value is large or structured, the parent writes it to a
file and passes the path:

```python
def perform_task(self, environment, parameters, flowstate):
    build_metadata = {
        "build_id": parameters["build_id"],
        "feature_flags": parameters["feature_flags"],
        "smoke_test_results": flowstate.get("smoke_test_results", {}),
    }
    metadata_path = os.path.join(output_dir, "build-metadata.json")
    with open(metadata_path, "w") as fh:
        json.dump(build_metadata, fh)

    return run_subprocess(f"./scripts/deploy.sh --metadata {metadata_path}")
```

The subprocess reads the file.  The pattern handles arbitrarily
large values without polluting `os.environ`.

The three patterns combined cover every common
orchestration-to-subprocess information-flow case.  When all
three are insufficient, a custom protocol is the answer — but
the threshold for needing one is high.

## Distributed orchestrations

Volume 2 develops the distributed-orchestration story in detail.
This chapter notes that the four channels generalise cleanly:

- **Channel 1** generalises to the
  [`InteropLibraryTestJob`](../../../automojo-interop/source/packages/automojo/interop/testing/testjob.py)
  hooks: when the coordinator activates a remote worker, the
  hook propagates the `AM_*_NAMES` env vars to the worker's
  process.  Same names, same configurations on both sides.
- **Channel 2** (parameters) generalises through
  serialisable-context propagation.  The coordinator's parameters
  dict is JSON-serialised and shipped to the worker; the worker
  reconstructs the dict at startup.
- **Channel 3** (flowstate) is *not* generalised.  Cross-machine
  flowstate would be a distributed-state problem; the discipline
  in distributed orchestration is to use external coordination
  primitives (a shared key-value store, a message broker)
  rather than treating flowstate as cross-machine.
- **Channel 4** (return codes + side effects) generalises through
  the JSOS streaming results pipeline (Volume 1's territory):
  worker-side return codes and logged events stream back to the
  coordinator over the same channels the single-host orchestration
  uses.

The discipline in this chapter holds end-to-end across the
distributed case.  Volume 2 develops the specifics; Part V's
foundation is what Volume 2 builds on.

## What this is *not*

Three boundaries to keep clear.

### Not RPC

The four-channel contract is *unidirectional* per channel and
*ad-hoc* in semantics.  When tasks need true RPC — a
request/response pattern with a service contract — the right
answer is to use a real RPC library, not to bend flowstate or
the engine into a poor imitation.

### Not a message queue

The contract has no notion of *deferred* delivery, no fanout,
no durable queue, no consumer groups.  When the orchestration
needs message-queue semantics — a producer task and several
parallel consumer tasks, with at-least-once semantics — that's
a message queue, and the workflow author should bring one in.

### Not shared state

Flowstate is per-orchestration.  When two parallel orchestrations
need to coordinate, the right answer is an external coordination
primitive (Redis, etcd, a shared filesystem with lock files).
The flowstate dict is *not* a substitute.

The chapter's discipline applies *within* one orchestration's
process and to the subprocesses it spawns.  Cross-orchestration
coordination is out of scope; the discipline does not pretend
otherwise.

## Summary

- The orchestration-to-job contract has **four channels**:
  environment exports (process-boundary-crossing, narrow,
  configuration-selectors-only), workflow parameters
  (orchestration-internal, flow-scope, typed), flowstate
  (orchestration-internal, ad-hoc inter-task state), and task
  return codes plus side effects (the outbound channel).
- The discipline of channel 1: **only configuration selectors
  and process identity travel through environment exports**.
  Configuration *content* travels through resolved configurations
  the subprocess loads independently using `automojo-configuration`
  and the inherited `AM_*_NAMES` selectors.
- The discipline of channel 2: parameters are flow-scope,
  read-only by convention, and carry typed YAML data (not
  string-encoded content).  Parameters do not automatically
  reach subprocesses; explicit passing is required.
- The discipline of channel 3: flowstate is ad-hoc inter-task
  communication.  Use namespaced keys; document contracts close
  to the producing task; keep the dict small.
- The discipline of channel 4: return codes carry binary
  outcome (zero / non-zero) plus optional semantic information
  (specific error codes); logger output goes through the shared
  logger with chapter-11 redaction; filesystem side effects use
  the configured output directory.
- The bootstrap-name → resolve-config flow is the central
  engineering claim: orchestration sets `AM_*_NAMES`, subprocess
  inherits them, subprocess loads the same configurations
  independently.  Same code, same configurations, no
  serialisation across the seam.
- Three patterns cover orchestration-to-subprocess information
  flow when channels 1 and 2 don't reach: make the value a
  runtime configuration; pass the value as a command-line
  argument; write the value to a file and pass the path.
- The four-channel contract generalises to distributed
  orchestrations (Volume 2's territory) for channels 1, 2, and
  4; channel 3 (flowstate) is intentionally local and not
  generalised.
- The contract is not RPC, not a message queue, not shared
  state.  Each is a separate engineering problem with its own
  primitive.

The next chapter turns to flow control: how `if_expr`,
`onfailure`, and `onsuccess` become the engine's actual
sequencing decisions.  The `JumpContext` and the
`evaluate_expression` machinery from
[`flowengine.py`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/flowengine.py)
get walked end-to-end with the same code-anchored fidelity that
chapters 18–20 used.
