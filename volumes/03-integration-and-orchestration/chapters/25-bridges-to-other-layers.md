# Chapter 25 — Bridges to Other Layers

> *Where chapter 24 showed the outward wiring (CI vendor), this
> chapter shows the downward wiring — the taskplus tasks that
> invoke the layers below the orchestration.*

The previous chapter wired the orchestration to the CI vendor.
This chapter wires the orchestration *downward* — to the layers
of the automojo portfolio that the workflow's tasks invoke.
Three bridges in particular:

1. **Orchestration → distributed test job** — the
   `DistributedTestJob` from Volume 2's `automojo-testplus`.
2. **Orchestration → sysman feature** — the cross-OS feature
   execution machinery from Volume 4's `automojo-sysman-*`
   packages.
3. **Orchestration → provisioning** — the Specification
   application from Volume 6's `automojo-provisioning` (which
   chapter 4 introduced).

Each bridge is a *task* — a concrete `Task` subclass that wraps
the relevant downstream API.  The pattern is uniform: the task
reads its required configuration from the active typed
configurations (per chapter 21's information-passing contract),
calls into the downstream library, and returns a result code.

This chapter is shorter than most because the pattern is small.
The chapter's job is to make the *pattern* visible across three
worked examples and to draw out the discipline that ties them
together.

## The composition pattern

The three bridges share a structure.  Each is a `Task` subclass
that:

1. **Reads its required configuration from the runtime API** —
   not from `os.environ`, not from the workflow's `parameters`,
   not from constructor-passed values that the workflow had to
   know about.  The active landscape, the active credentials,
   the active runtime are all already loaded by the time the
   task's `perform_task` runs.

2. **Calls into the downstream library through its public API.**
   The downstream library — testplus, sysman-X, provisioning —
   exposes a small surface that the bridge task uses.  No
   reaching past the API into internals; no mutating internal
   state directly.

3. **Returns a single integer result.**  The downstream library
   may have rich result types (test result objects, sysman
   feature outcomes, provisioning state diffs); the bridge task
   collapses the result to the chapter-18 success/failure
   contract and writes anything richer to flowstate or to disk.

The discipline keeps the bridges *thin*.  A bridge task is
typically forty to eighty lines of Python — it's mostly glue,
not logic.

## Bridge 1: orchestration → distributed test job

A taskplus workflow that runs a distributed test job invokes a
`DistributedTestJob` from Volume 2's
[`automojo-testplus`](../../../automojo-testplus/source/packages/automojo/testplus/).
The chain of classes — `TestJob` (in testbase) →
`InteropLibraryTestJob` (in interop) → `DistributedTestJob` (in
testplus) — is Volume 2's territory; this chapter just shows the
bridge from a workflow into the chain.

### YAML shape

```yaml
- task: Run integration suite
  tasktype: automojo.taskplus.tasks.distributedtest@DistributedTestJobTask
  test_root: source/testroots/automojo
  includes:
      - automojo.tests.integration
  excludes: []
  job_seed: ${parameters[job_seed]:-${RANDOM}}
```

Four task-specific fields:

- `test_root` — the testbase test-root directory.
- `includes` — list of test paths the job should include.
- `excludes` — list of test paths the job should exclude.
- `job_seed` — the chapter-16 reproducibility seed, optionally
  pulled from workflow parameters.

The active landscape, topology, runtime, credentials, and data
profiles are *already loaded* — the bridge doesn't need to read
them from the YAML.

### Task implementation

The bridge is roughly sixty lines:

```python
"""
.. module:: distributedtest
    :synopsis: Bridge task that runs a DistributedTestJob from within
               a taskplus workflow.
"""

from typing import Any, Dict

from automojo.errors.exceptions import ConfigurationError
from automojo.taskplus.model.task import Task


class DistributedTestJobTask(Task):
    """
        Runs a DistributedTestJob (Volume 2) from within a taskplus
        workflow.  The active landscape, topology, credentials, and
        runtime configurations are picked up automatically through
        the runtime API.
    """

    def __init__(self, ordinal: str, label: str, task_info: dict, logger):
        super().__init__(ordinal, label, task_info, logger)

        if "test_root" not in task_info:
            errmsg = "DistributedTestJobTask requires a 'test_root' field."
            raise ConfigurationError(errmsg)

        self._test_root = task_info["test_root"]
        self._includes = task_info.get("includes", [])
        self._excludes = task_info.get("excludes", [])
        self._job_seed = task_info.get("job_seed", None)

        return

    def perform_task(self, environment, parameters, flowstate) -> int:

        # Late import to avoid coupling taskplus to testplus at module load.
        from automojo.testplus.distributedtestjob import DistributedTestJob

        # The active landscape comes from the resolved configuration —
        # the chapter-21 bootstrap-name flow already loaded it.
        job = DistributedTestJob(
            test_root=self._test_root,
            includes=self._includes,
            excludes=self._excludes,
            job_seed=self._job_seed,
        )

        self._logger.info(
            "Distributed test job: includes=%s excludes=%s seed=%s" %
            (self._includes, self._excludes, self._job_seed)
        )

        result = job.run()

        flowstate["distributedtest.last_results_path"] = result.results_path
        flowstate["distributedtest.passed_count"] = result.passed_count
        flowstate["distributedtest.failed_count"] = result.failed_count

        if result.failed_count > 0:
            self._logger.error(
                "Distributed test job: %d tests failed (results: %s)" %
                (result.failed_count, result.results_path)
            )
            return 1

        return 0
```

Three things worth naming.

#### Late import

The `from automojo.testplus.distributedtestjob import
DistributedTestJob` is inside `perform_task`, not at module top.
This is deliberate: a workflow that uses taskplus but *not*
testplus shouldn't pay the cost of importing testplus.  The late
import means the dependency is *optional* — `automojo-taskplus`
can be installed without `automojo-testplus` and the bridge
class is only constructible when the dependency is present.

The technique generalises.  Bridges that wrap optional dependencies
(testplus, sysman-X, gitlab, etc.) use late imports so the
bridge is loadable at module-import time even when the
dependency isn't installed.

#### Configuration is implicit

The bridge does *not* take `landscape:`, `topology:`,
`credentials:`, `runtime:` fields.  Those configurations are
*already loaded* — the chapter-21 bootstrap-name flow set
`AM_LANDSCAPE_NAMES`, etc., in the orchestration's environment;
the orchestration's startup loaded the named files.  The
distributed-test-job machinery reads them through the runtime
API (`LandscapeSingleton()`, `CredentialManagerSingleton()`,
etc.).

This is the chapter's structural point: the bridge wraps an API
call.  The API has its own contract for finding configurations
(through the runtime API).  The bridge does not duplicate the
contract; it relies on it.

#### Flowstate writes

The bridge writes three flowstate keys with the
`distributedtest.` namespace prefix (chapter 21's convention):

- `distributedtest.last_results_path` — where the JSOS results
  stream lives on disk.
- `distributedtest.passed_count` — count of passed tests.
- `distributedtest.failed_count` — count of failed tests.

A subsequent task in the workflow — say, a notification task
that posts pass/fail counts to Slack — reads these flowstate
keys and uses them.  The bridge does *not* attempt to do the
notification itself; that's a separate task with its own
concerns.

### Cross-references to Volume 2

The full distributed-test-job machinery — the `TestJob` →
`InteropLibraryTestJob` → `DistributedTestJob` inheritance
chain, the landscape-activation hooks, the
`DistributedTestSequencer`, the result aggregation — is the
subject of Volume 2.  This chapter's bridge task is the
*entrance* to that machinery from a taskplus workflow.

A reader who wants more detail than the bridge provides — how
does the distributed sequencer work?  what are the
`fire_*` lifecycle hooks?  how does landscape activation
propagate to remote workers? — should read Volume 2 chapter 4
(landscape activation) and chapter 9 (result aggregation).

## Bridge 2: orchestration → sysman feature

A workflow that runs a sysman feature against a fleet — for
example, "set the firewall ruleset on every host in the active
landscape" — invokes one of the
[`automojo-sysman-*`](../../../automojo-sysman-common/) packages
through a small bridge task.

### YAML shape

```yaml
- task: Configure firewall on staging
  tasktype: automojo.taskplus.tasks.sysmanfeature@SysmanFeatureTask
  feature: networking.firewall
  action: configure
  parameters:
      ruleset_name: ci-restricted
      allow_inbound:
          - tcp/22
          - tcp/443
      default_deny: true
```

Three task-specific fields:

- `feature` — the cross-OS contract identifier (chapter 4 of
  Volume 4 catalogues these).
- `action` — the verb (`configure`, `enable`, `query`, `apply`)
  the feature exposes.
- `parameters` — feature-specific arguments.

### Task implementation

```python
class SysmanFeatureTask(Task):
    """
        Runs a sysman feature (Volume 4) against the apod members
        of the active landscape.  The feature is dispatched to the
        per-OS implementation automatically.
    """

    def __init__(self, ordinal, label, task_info, logger):
        super().__init__(ordinal, label, task_info, logger)

        for field in ("feature", "action"):
            if field not in task_info:
                errmsg = "SysmanFeatureTask requires '%s' field." % field
                raise ConfigurationError(errmsg)

        self._feature = task_info["feature"]
        self._action = task_info["action"]
        self._parameters = task_info.get("parameters", {})

        return

    def perform_task(self, environment, parameters, flowstate) -> int:

        from automojo.sysman.common.featurebase import (
            run_feature_action_on_landscape,
        )

        self._logger.info(
            "Sysman feature: %s.%s on landscape members" %
            (self._feature, self._action)
        )

        results = run_feature_action_on_landscape(
            feature=self._feature,
            action=self._action,
            parameters=self._parameters,
        )

        flowstate["sysmanfeature.last_results"] = results

        failed_hosts = [
            host for host, result in results.items()
            if result.exit_code != 0
        ]

        if len(failed_hosts) > 0:
            self._logger.error(
                "Sysman feature %s.%s failed on %d hosts: %s" %
                (self._feature, self._action, len(failed_hosts), failed_hosts)
            )
            return 1

        return 0
```

Three observations:

#### The feature runs against the landscape

The bridge does not take a list of hosts.  The
`run_feature_action_on_landscape` function (a stand-in for
Volume 4's actual API) iterates the *active landscape*'s apod
members and dispatches the feature to each.  The same task
running against `staging-east` runs against three runners;
running against `local-dev` runs against one local machine; the
task's YAML doesn't change.

This is the same configuration-implicit pattern as Bridge 1.
The discipline is uniform: the bridge wraps an API; the API
already knows how to find the landscape; the bridge passes
through.

#### The cross-OS contract is the boundary

A workflow that says `feature: networking.firewall` on a
landscape with one Linux runner and two macOS runners gets
*three different per-OS implementations* dispatched
automatically.  The cross-OS contract from Volume 4 chapter 2
(the 50 contracts catalogued there) is the typed surface; the
per-OS implementations are the dispatch targets.

The workflow author writing `feature: networking.firewall` does
not need to know that the Linux implementation calls `ufw` and
the macOS implementation calls `pfctl` and the Windows
implementation calls PowerShell `NetSecurity`.  The contract
is the abstraction; the dispatch happens at the sysman library
level.

#### Per-host results

Sysman features run against multiple hosts; the result is a
dict keyed by host name.  The bridge writes the full dict to
flowstate (`sysmanfeature.last_results`) and computes a single
pass/fail decision for the engine — succeed if all hosts
succeeded; fail if any host failed.

A more sophisticated workflow might want partial success
semantics — "fail only if more than 50% of hosts failed" — and
would use a custom bridge task or a chapter-23 wrapper.  The
shipped bridge is uniform; teams that need richer semantics
extend.

### Cross-references to Volume 4

The full sysman story — the 50 cross-OS contracts, the
`featurebase` machinery, the per-OS implementations for RHEL,
Ubuntu, macOS, Windows, UEFI — is Volume 4.  The bridge above
is the *entrance* from a workflow.  Readers who want more than
the bridge provides should read Volume 4 chapters 2
(contracts), 3 (`featurebase`), and the per-OS chapters
relevant to their fleet.

## Bridge 3: orchestration → provisioning

A workflow that needs to provision (or tear down) cloud
resources before (or after) the rest of its work invokes
`ProvisioningEngine` from
[`automojo-provisioning`](../../../automojo-provisioning/source/packages/automojo/provisioning/),
introduced in chapter 4.

### YAML shape

```yaml
- task: Provision staging-east
  tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
  specification: config/specs/spec-staging-east.yaml
  action: apply           # apply | teardown | query
```

Two task-specific fields:

- `specification` — path to the chapter-4 Specification YAML.
- `action` — `apply` (default), `teardown`, or `query`.

### Task implementation

```python
class ProvisioningTask(Task):
    """
        Applies, tears down, or queries an automojo-provisioning
        specification (chapter 4) from within a taskplus workflow.
    """

    VALID_ACTIONS = frozenset({"apply", "teardown", "query"})

    def __init__(self, ordinal, label, task_info, logger):
        super().__init__(ordinal, label, task_info, logger)

        if "specification" not in task_info:
            errmsg = "ProvisioningTask requires a 'specification' field."
            raise ConfigurationError(errmsg)

        self._specification = task_info["specification"]
        self._action = task_info.get("action", "apply")

        if self._action not in self.VALID_ACTIONS:
            errmsg = (
                "ProvisioningTask action must be one of %s; got %r" %
                (sorted(self.VALID_ACTIONS), self._action)
            )
            raise ConfigurationError(errmsg)

        return

    def perform_task(self, environment, parameters, flowstate) -> int:

        from automojo.provisioning.provisioningengine import (
            ProvisioningEngine,
        )

        self._logger.info(
            "Provisioning %s: %s" % (self._action, self._specification)
        )

        engine = ProvisioningEngine.create_provisioning_engine(
            specification=self._specification
        )

        if self._action == "apply":
            result = engine.apply()
        elif self._action == "teardown":
            result = engine.teardown()
        elif self._action == "query":
            result = engine.query()

        flowstate["provisioning.last_action"] = self._action
        flowstate["provisioning.last_specification"] = self._specification
        flowstate["provisioning.last_result"] = result

        if result.errors:
            for err in result.errors:
                self._logger.error("Provisioning error: %s" % err)
            return 1

        return 0
```

Three observations:

#### Action enumeration

The task validates `action` against a fixed set of allowed
values at construction time.  An unknown action fails *to load*,
not *to execute* — the workflow author finds out about the typo
before the workflow starts.  The pattern is the same as the
chapter-19 task validation pattern; both use the same fail-fast
discipline.

#### The Specification is path-based

The `specification:` field is a path, not a name.  This is a
deliberate departure from the chapter-3 lookup-by-name pattern
the other categories use.  Specifications live outside the
configuration loader (chapter 4 explained why); the workflow
references them by path.

When a team wants the lookup-by-name behaviour for
specifications, the workflow author can build a thin adapter —
construct the path from a chapter-7 runtime configuration's
`paths.specifications_dir` plus the spec name.  The discipline
is the workflow author's; the shipped bridge is intentionally
narrow.

#### Apply, teardown, query as separate calls

The three actions are *separate methods* on the engine.  This is
deliberate: each has different semantics, different failure
modes, different audit requirements.  The bridge task chooses
the right one and lets the engine do the work.

A workflow that provisions before the test phase and tears down
after looks like this:

```yaml
flow:
    - task: Provision
      tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
      specification: config/specs/spec-staging-east.yaml
      action: apply

    - taskgroup: Test phase
      items:
          # ... test tasks ...

    - task: Teardown
      tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
      specification: config/specs/spec-staging-east.yaml
      action: teardown
      if: "parameters['cleanup_after_run'] == True"
```

The same Specification is applied at the start and torn down at
the end.  The Specification's idempotent-by-design property
(chapter 4) makes both safe.

### Cross-references to Volume 6

The full provisioning story — vendor-neutral resource models,
AWS / Azure / GCP backends, the four-layer abstraction (Venue,
Group, Profile, Device) — is Volume 6's territory.  The bridge
above is the *entrance* from a workflow.  Volume 6 chapter 4
(the cloud provisioning model) develops the implementation in
depth.

## A workflow combining all three bridges

To make the composition concrete, here is a workflow that uses
all three bridges in one pipeline:

```yaml
name: nightly-integration-with-provisioning
parameters:
    cleanup_after_run: true

flow:
    # Bridge 3: provision the test environment
    - task: Provision staging-east
      tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
      specification: config/specs/spec-staging-east.yaml
      action: apply

    # Bridge 2: configure firewalls on the provisioned hosts
    - task: Configure firewall
      tasktype: automojo.taskplus.tasks.sysmanfeature@SysmanFeatureTask
      feature: networking.firewall
      action: configure
      parameters:
          ruleset_name: ci-restricted
          allow_inbound:
              - tcp/22
              - tcp/443
          default_deny: true

    # Bridge 1: run the distributed integration suite
    - task: Run integration suite
      tasktype: automojo.taskplus.tasks.distributedtest@DistributedTestJobTask
      test_root: source/testroots/automojo
      includes:
          - automojo.tests.integration
      job_seed: ${parameters[job_seed]:-${RANDOM}}

    # Bridge 3 again: tear down the test environment
    - task: Teardown
      tasktype: automojo.taskplus.tasks.provisioning@ProvisioningTask
      specification: config/specs/spec-staging-east.yaml
      action: teardown
      if: "parameters['cleanup_after_run'] == True"
```

Four tasks; four downstream-library invocations.  The workflow
itself is twenty-five lines.  The downstream libraries —
provisioning, sysman, testplus, plus the testbase substrate —
together represent thousands of lines of substantial
implementation, all hidden behind the small typed surface this
chapter has shown.

This is the chapter's central claim: **the orchestration
expresses *what* should happen as a sequence of typed
invocations into other layers**.  The *how* of each invocation
is the responsibility of the layer.  When the workflow is
right-sized (two dozen lines), the downstream layers do the
substantial work; when the workflow is wrong-sized (hundreds of
lines), the workflow has absorbed responsibilities that
belong elsewhere.

## Anti-patterns

Three patterns that look like bridges but aren't.

### Reaching past the API

A bridge task that imports `automojo.testplus.internal._sequencer_state`
and mutates it directly is reaching past the public API.  When
the downstream library refactors, the bridge breaks; when the
breaking change reaches the workflow, the workflow author is
the one debugging it.

The discipline is: the bridge calls *only* the downstream
library's public API.  When the public API doesn't expose what
the workflow needs, the right answer is to *propose the API*
(submit a PR to the downstream library), not to reach past it.

### Doing the layer's work in the bridge

A bridge task that *re-implements* part of the downstream
library's logic — loading credentials manually instead of
letting the testplus job load them, dispatching per-OS code
manually instead of using the sysman library's dispatch — has
absorbed responsibilities that belong elsewhere.

The bridge is *thin* by design.  When the bridge gets fat —
more than a hundred lines for a single downstream invocation —
something has gone wrong.  Either the downstream API is
inadequate (propose a PR to it) or the bridge is doing work it
shouldn't (extract the work back into the downstream library).

### Bridges that compose multiple layers

A bridge task that "runs distributed tests against a freshly
provisioned environment" — combining Volume 2 and Volume 6 in
one task — is doing the workflow's job inside a task.  The
right shape is two tasks (one for provisioning, one for
distributed testing) sequenced in the workflow.

The discipline is: each bridge wraps *one* downstream library.
Composition happens at the workflow layer, not inside any
single task.  When the workflow needs the composition shape
"provision, then test, then tear down," the workflow expresses
it as three tasks; the engine sequences them.

## Summary

- Three bridges connect the taskplus orchestration layer to the
  rest of the automojo portfolio: **DistributedTestJob** (Volume
  2), **sysman feature** (Volume 4), **provisioning** (Volume
  6).  Each is a thin `Task` subclass — typically 50–80 lines
  — that wraps a single downstream API call.
- The composition pattern is uniform: read configuration through
  the runtime API (not from `os.environ` or workflow YAML
  values), call into the downstream library through its public
  surface, return a single integer result, write rich state to
  flowstate.
- Bridges use **late imports** so the dependency on the
  downstream library is optional — `automojo-taskplus` can
  install without `automojo-testplus` and the testplus bridge
  class is only constructible when the dependency is present.
- The downstream APIs read configuration through the runtime
  API automatically.  The bridge does *not* take landscape,
  topology, credentials, or runtime fields in its YAML — those
  are already loaded by chapter-21's bootstrap-name flow.
- A worked example combines all three bridges in one workflow:
  provision the environment, configure firewalls, run the
  distributed test suite, tear down.  Twenty-five lines of YAML;
  thousands of lines of underlying implementation.
- Three anti-patterns: reaching past the downstream library's
  API into internals, doing the layer's work in the bridge,
  composing multiple layers in a single bridge.  The discipline
  is uniform — bridges are thin, composition is at the
  workflow layer.

The next chapter is the case-study capstone of the volume — a
worked port of a real CI pipeline (a 200-line GitLab YAML with
47 environment variables) into the discipline.  Where this
chapter showed three abstract bridges, the next chapter shows
the discipline as an end-to-end migration of a system that
actually runs in production.
