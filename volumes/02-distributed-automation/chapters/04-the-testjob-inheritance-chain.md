# Chapter 4 — The `TestJob` Inheritance Chain — The Customisation Cascade

> *Four classes, each adding one responsibility, plus three
> naming-only fallbacks.  The chain is the framework's
> customisation entry point — the meta-thesis from Volume 0
> chapter 1 (skeleton + patterns) made concrete.  This chapter
> walks the chain from base to distributed and shows how a
> project plugs in its custom Landscape and Topology.*

The `TestJob` inheritance chain is the single most important
piece of code in distributed automation.  It is *the* place
where:

- The single-host substrate from Volume 1 meets the distributed
  extension Volume 2 walks.
- The framework's lifecycle hooks become non-trivial.
- The user's customisations (custom `Landscape`, custom
  `Topology`, custom `Validator`s) plug in.

The chain is small.  Seven classes, three of them naming-only.
The non-trivial work happens at four layers, and each layer
adds exactly one responsibility.

## The seven classes

*The diagram below is the **intended** chain. One edge —
`IntegratedTestJob → InteropLibraryTestJob` — has **not yet landed**
(see "In-flight migration" immediately after): in today's source
`InteropLibraryTestJob` extends `TestJob` directly, so as built it
and `IntegratedTestJob` are **siblings**, both extending `TestJob`.*

```
TestJob (testbase)                       ← single-host base + stdout recorder + standalone seams
   │
   ├── DefaultTestJob (testbase)         ← naming fallback
   │
   └── IntegratedTestJob (integration-runtime)   ← integration result recording
          │
          ├── DefaultIntegratedTestJob (integration-runtime)  ← naming fallback
          │
          └── InteropLibraryTestJob (interop)    ← landscape activation
                 │
                 └── DistributedTestJob (testplus)   ← distributed sequencer
                        │
                        └── DefaultDistributedTestJob (testplus)  ← naming fallback
```

The non-trivial-work layers are `TestJob`, `IntegratedTestJob`,
`InteropLibraryTestJob`, and `DistributedTestJob`.  The other
three are naming-only fallbacks for entry points that need a
default `job_type=`.

> **In-flight migration.** `IntegratedTestJob` is newly built
> and `TestJob` already exposes the three provider hooks it
> overrides.  But the *re-parenting* of interop's
> `InteropLibraryTestJob` onto `IntegratedTestJob` has not yet
> landed: in the interop source today,
> `InteropLibraryTestJob` still extends `TestJob` directly
> (`codecraftai-interop/source/packages/ccai/interop/testing/testjob.py:22`
> reads `class InteropLibraryTestJob(TestJob)`).  The diagram
> above is the *intended* chain; the section on
> `InteropLibraryTestJob` notes where the source currently
> diverges.  This is the same kind of partially-landed refactor
> the c785e9b narrative below describes — reported honestly
> rather than overclaimed.

## Layer 1 — `TestJob` (the near-standalone single-host base)

**File:** `codecraftai-testbase/source/packages/ccai/testbase/testjob.py:45`

`TestJob` is the foundation.  Volume 1 chapter 2 walked it in
detail; this chapter recaps what's relevant for distributed
extension.

`testbase` is now *near-standalone* — it no longer depends on
`codecraftai-configuration`, `codecraftai-runtime`, or
`codecraftai-this-app`.  A plain `TestJob` runs **without
activation**: no presence folder, no `RuntimeVariables`, no
this-app.  It reads the handful of core env knobs it needs
(AM_DEBUG_*, AM_DIAGNOSTICS_*, AM_TESTROOT, AM_OUTPUT_DIRECTORY,
AM_RUN_ID, AM_TESTRUN_TIMEOUT) directly from `os.environ` via the
tiny, dependency-free `ccai.testbase.environment` module, and
records its run summary to **stdout**.  Everything heavier —
the presence-folder results layout, JSOS-to-disk, the HTML
summary, CI/build metadata — is the responsibility of
`IntegratedTestJob` in the next layer.

The class establishes:

- The constructor with logger, testroot, includes/excludes,
  metafilters, job_seed, and the test module.
- `__enter__` / `__exit__` driving `begin()` / `finalize()`.
- The five `fire_*` lifecycle hooks — *all stubs on single-host*:
  - `fire_attach_to_environment(tseq)` — line 379.
  - `fire_attach_to_framework(tseq)` — line 389.
  - `fire_activate_configuration()` — line 399.
  - `fire_activate_integration()` — line 403.
  - `fire_activate_operations()` — line 407.
- The provider/template methods that customisation plugs into:
  - `_create_sequencer()` (line 575) — returns a `TestSequencer`
    (the base) by default; subclasses override to return a
    `DistributedTestSequencer` or other variant.
  - `_create_recorder(...)` (line 564) — returns a
    `StdoutResultRecorder` by default.  It prints the run
    summary to stdout and reuses the JSOS/JSON serialisation (so
    `testrun_results_stream.jsos` / `testrun_summary.json` are
    still written when an output dir is used, and still consumed
    by the `testbase agents` CLI), but it **skips the HTML
    rendering**.  Subclasses that want disk-backed JSOS plus a
    rendered HTML summary override this to return a
    `JsonResultRecorder`.
  - `_create_output_context()` (line 583) — returns a
    `SimpleOutputContext` (an `IOutputContext`) by default,
    rooting artefacts under AM_OUTPUT_DIRECTORY or `<cwd>/.output`.
  - `_create_run_metadata()` (line 595) — returns a
    `StandaloneRunMetadata` (an `IRunMetadata`) by default:
    auto-generated UUID `run_id`, `"(unknown)"` build / pipeline
    / job, title from the job's name.

The five `fire_*` hooks are **the contract**.  They run at
specific phase boundaries during a test run (Volume 1 chapter 2
walks the order).  On a single-host run they are no-ops; on
distributed, the subclasses below give them bodies.

The hooks are *lifecycle hooks*, not template methods.  They
run *during* the lifecycle; they do not return values that the
framework consumes.  The provider methods (`_create_sequencer`,
`_create_recorder`, `_create_output_context`,
`_create_run_metadata`, and the abstract `_get_landscape`
introduced two layers down) are the *value-returning*
customisation points.  `_create_run_metadata()` /
`_create_output_context()` are invoked once in the constructor
`__init__` (lines 95–96) so the run's identity and artefact roots
are fixed before `begin()` or any test executes.

### What `TestJob` does NOT have

A point worth being explicit about because the next layer adds
it: `TestJob` has *no concept of a landscape*.  There is no
`_get_landscape()` method, no landscape parameter, no
import of `codecraftai-landscaping` from anywhere in `testbase`.
This is intentional — `codecraftai-testbase` is the foundation
tier (Volume 0); `codecraftai-landscaping` is upstream of it.  The
dependency arrow runs `landscaping → interop → testbase`, not
the other way.  Volume 0 chapter 4 walks the protocol-based
discipline that makes this possible.

The landscape concept enters two layers down, in
`InteropLibraryTestJob` — but does so as an abstract hook the
user (or a downstream package) implements, not as a direct
import.  The result: `testbase` stays landscape-free and the
chain remains importable on a host that doesn't have landscaping
installed.

## Layer 2 — `IntegratedTestJob` (integration result recording)

**File:** `codecraftai-integration-runtime/source/packages/ccai/integrationruntime/integratedtestjob.py:51`

`IntegratedTestJob` is the layer that *re-supplies the
heavyweight reporting* the near-standalone `TestJob`
deliberately leaves out.  It lives in the **new**
`codecraftai-integration-runtime` package (deps: `testbase` +
`this-app` + `extension`) and adds exactly one responsibility:
**integration result recording**.  It does so by overriding the
*three* provider hooks `TestJob` exposes — and nothing else.

> **Single responsibility.** `IntegratedTestJob` does **not**
> override the five `fire_*` landscape hooks.  Those were *not*
> moved here; they stay in `InteropLibraryTestJob` (Layer 3).
> `IntegratedTestJob`'s only job is to swap the standalone
> stdout/`.output` result plumbing for activation-backed
> presence-folder plumbing.

### The three provider overrides

```python
def _create_output_context(self) -> IOutputContext:
    inst = RuntimeOutputContext()
    return inst

def _create_run_metadata(self) -> IRunMetadata:
    run_title = "Test Run"
    if self.name is not None and len(self.name) > 0:
        run_title = self.name
    inst = RuntimeRunMetadata(title=run_title)
    return inst

def _create_recorder(self, runid, start, render_info, build_info,
                     pipeline_info, job_info) -> ResultRecorder:
    inst = JsonResultRecorder(runid=runid, start=start, render_info=render_info,
                              apod=self._apod, build_info=build_info,
                              pipeline_info=pipeline_info, job_info=job_info)
    return inst
```

Each override replaces a standalone default with its
integration counterpart:

- `_create_output_context()` → `RuntimeOutputContext`
  (`codecraftai-integration-runtime/source/packages/ccai/integrationruntime/runtimeoutputcontext.py`)
  — delegates to `ccai.thisapp.paths` so results, diagnostics,
  and per-testcase by-products land under the activated app's
  **well-known presence folder**.  It is **activation-gated**:
  it raises `SemanticError` unless the process was activated as a
  job (this-app activation sets `AM_JOB_TYPE` / `AM_PRESENCE_DIR` /
  `AM_RESULT_PATH_FOR_TESTS`).  This replaces `TestJob`'s
  `SimpleOutputContext`, which roots under `<cwd>/.output`.
- `_create_run_metadata()` → `RuntimeRunMetadata`
  (`codecraftai-integration-runtime/source/packages/ccai/integrationruntime/runtimerunmetadata.py`)
  — sources build / pipeline / job metadata from the resolved
  `RuntimeVariables` (the real CI/runtime context) instead of the
  `"(unknown)"` placeholders `StandaloneRunMetadata` supplies.
- `_create_recorder()` → `JsonResultRecorder` — JSOS streaming to
  disk **plus** the rendered HTML summary, in place of
  `TestJob`'s `StdoutResultRecorder`.

### Activation is a precondition, not a hook

`IntegratedTestJob` *assumes* the process was already activated
as a test-run job before `begin()` resolves the presence-folder
paths.  The activation itself — `establish_app_persona` +
`resolve_runtime_variables` + `activate_profile_job(JobType.TestRun)`
— lives in `ccai.integrationruntime.initialize`, and
integration runs reach the job through
`ccai.integrationruntime.entrypoints.integrated_test_entrypoint`,
which activates and then delegates to testbase's standalone
`generic_test_entrypoint`.  This keeps activation out of the
class hierarchy: `IntegratedTestJob` consumes the activated
context (via `RuntimeOutputContext` / `RuntimeRunMetadata`); it
does not perform activation.

`DefaultIntegratedTestJob` (line 91) is the naming-only fallback
for this layer — same role as `DefaultTestJob` and
`DefaultDistributedTestJob`.

## Layer 3 — `InteropLibraryTestJob` (the landscape-aware hinge)

**File:** `codecraftai-interop/source/packages/ccai/interop/testing/testjob.py:22`

`InteropLibraryTestJob` is the *hinge* of the chain.  It adds
exactly one responsibility: **landscape activation on the five
`fire_*` hooks**.  It does so without taking a hard dependency
on `codecraftai-landscaping` — by introducing an abstract
template method `_get_landscape()` that subclasses must
override.

> **In-flight: the parent is moving.** In the intended chain,
> `InteropLibraryTestJob` extends `IntegratedTestJob` (Layer 2)
> so a distributed run inherits both integration result
> recording *and* landscape activation.  That re-parenting has
> **not yet landed**.  In the interop source today the class
> still extends `TestJob` directly — see the declaration below.
> Until the migration completes, an interop/testplus run does
> **not** automatically pick up `IntegratedTestJob`'s
> presence-folder recording from the class hierarchy; it gets
> the standalone stdout plumbing from `TestJob` unless the
> entry point arranges otherwise.  This is reported as-built,
> not as-intended.

### The class declaration

```python
# codecraftai-interop/source/packages/ccai/interop/testing/testjob.py:22
# As built today — still extends TestJob.  Intended parent: IntegratedTestJob.

class InteropLibraryTestJob(TestJob):
    """
        Interop-tier test-job base class.

        Extends :class:`ccai.testbase.testjob.TestJob` (single-host) with
        the **landscape-activation lifecycle** required by any test job that
        runs against a real environment described through the landscape model
        — devices, services, coordinators, control points, integration
        couplings.
    """
```

### The abstract `_get_landscape()` template method

```python
# Lines 60–86

def _get_landscape(self) -> Any:
    """
        Returns the landscape object used by the activation /
        attachment hooks.

        Concrete subclasses must override this method to return the
        landscape instance their environment is built from
        (typically ``LandscapeSingleton()`` from
        ``ccai.landscaping.wellknown``).

        :raises NotImplementedError: When called on a class that does
                                     not override the hook.
    """
    err_msg_lines = [
        "InteropLibraryTestJob subclasses must override _get_landscape().",
        "    SUBCLASS: {}".format(type(self).__name__),
    ]
    err_msg = "\n".join(err_msg_lines)
    raise NotImplementedError(err_msg)
```

The body raises `NotImplementedError` with a descriptive
message naming the subclass that failed to override.  This is
the Volume 0 chapter 3 discipline: errors should never pass
silently.  An `InteropLibraryTestJob` subclass that forgot
`_get_landscape()` fails loudly on first attempt to activate.

### The five `fire_*` overrides

```python
# Line 88 — fire_attach_to_environment

def fire_attach_to_environment(self, tseq: TestSequencer) -> None:
    """
        Attach the sequencer to the environment using the landscape.
    """
    landscape = self._get_landscape()
    tseq.attach_to_environment(landscape=landscape)
    return
```

```python
# Line 101 — fire_attach_to_framework

def fire_attach_to_framework(self, tseq: TestSequencer) -> None:
    """
        Attach the sequencer to the framework using the landscape.
    """
    landscape = self._get_landscape()
    tseq.attach_to_framework(landscape=landscape)
    return
```

```python
# Line 114 — fire_activate_configuration

def fire_activate_configuration(self) -> None:
    """
        Activate the landscape's configuration layer.
    """
    landscape = self._get_landscape()
    landscape.activate_configuration()
    return
```

```python
# Line 124 — fire_activate_integration

def fire_activate_integration(self) -> None:
    """
        Activate the landscape's integration layer.
    """
    landscape = self._get_landscape()
    landscape.activate_integration()
    return
```

```python
# Line 134 — fire_activate_operations

def fire_activate_operations(self) -> None:
    """
        Activate the landscape's operational layer.
    """
    landscape = self._get_landscape()
    landscape.activate_operations()
    return
```

The pattern is identical for every hook: fetch the landscape via
`_get_landscape()`, delegate to the right method on the
landscape.  The landscape is fetched fresh each time because
`LandscapeSingleton()` (the typical implementation) is a
singleton — the cost is one dict lookup, not a re-instantiation.

### The c785e9b commit — what this layer was extracted from

The introduction of `InteropLibraryTestJob` happened in commit
`c785e9b` ("Add InteropLibraryTestJob — interop-tier base class
for landscape-aware test jobs").  Before that commit,
`DistributedTestJob` was a standalone class that *duplicated
~600 lines of `TestJob` verbatim* with the five `fire_*`
overrides bolted on.  The duplication had three problems:

1. **The duplication itself.**  Two copies of the lifecycle
   logic meant any change to `TestJob` had to be made twice.
2. **The dependency direction.**  `DistributedTestJob` was in
   `codecraftai-testplus`, which is downstream of
   `codecraftai-testbase`.  The duplicated code in
   `DistributedTestJob` re-implemented `TestJob` semantics in a
   downstream package — a clear inheritance smell.
3. **The hook implementation.**  The five `fire_*` overrides in
   the original `DistributedTestJob` had hard imports of
   `LandscapeSingleton`.  This made the class un-importable on
   a host without `codecraftai-landscaping` installed — an
   ergonomic failure at the package boundary.

The c785e9b refactor introduced `InteropLibraryTestJob` as the
intermediate layer that:

- Inherits cleanly from `TestJob` (no duplication).
- Lives in `codecraftai-interop` (a tier above `testbase` but below
  `landscaping`).
- Defines `_get_landscape()` as abstract, deferring the landscape
  import to the concrete subclass.

Result: `DistributedTestJob` collapsed from ~615 lines to ~90.

The `IntegratedTestJob` layer (Layer 2) is the *next* step of
the same factoring discipline: rather than baking
presence-folder recording, `RuntimeVariables` metadata, and HTML
rendering into `TestJob` (which would re-couple the foundation
tier to `this-app` and `RuntimeVariables`), that behaviour was
lifted into a separate `codecraftai-integration-runtime` package
and slotted in *between* `TestJob` and `InteropLibraryTestJob`.
As noted above, the final wiring — re-parenting
`InteropLibraryTestJob` onto `IntegratedTestJob` — is the
in-flight step; until it lands, interop's class still extends
`TestJob`, the same way `DistributedTestJob` once extended
`TestJob` before c785e9b.

## Layer 4 — `DistributedTestJob` (the distributed plug-in)

**File:** `codecraftai-testplus/source/packages/ccai/testplus/distributedtestjob.py:24`

`DistributedTestJob` is the layer that *concretes* the abstract
`_get_landscape()` and adds *the distributed sequencer*.  It is
~90 lines, almost all of which is the two override methods.

### `_get_landscape()` — concrete

```python
# Lines 49–57

def _get_landscape(self) -> Any:
    """
        Returns the singleton :class:`Landscape` instance for the
        distributed test environment.

        :returns: The active :class:`Landscape`.
    """
    landscape = LandscapeSingleton()
    return landscape
```

The concrete `_get_landscape` returns
`LandscapeSingleton()` from `ccai.landscaping.wellknown`.
The singleton is factory-overridable (a custom `Landscape`
subclass registered through the factory protocol replaces the
default — Volume 0 chapter 7 walks the factory model), so a
project's custom landscape plugs in *here* without modifying
`DistributedTestJob` itself.

The customisation pattern: the project subclasses `Landscape`,
registers the subclass through the factory protocol,
`LandscapeSingleton()` returns the subclass, the rest of the
chain is unchanged.

### `_create_sequencer()` — distributed

```python
# Lines 59–81

def _create_sequencer(self) -> DistributedTestSequencer:
    """
        Hook used by :meth:`TestJob.execute` to construct the test
        sequencer for this job.  Returns a
        :class:`DistributedTestSequencer` so the sequencer's
        ``attach_to_environment`` / ``attach_to_framework`` methods
        accept the landscape supplied by
        :class:`InteropLibraryTestJob`.
    """
    inst = DistributedTestSequencer(
        self.title,
        self._testroot,
        includes=self.includes,
        excludes=self.excludes,
        metafilters=self.metafilters,
        job_seed=self.job_seed,
    )
    return inst
```

`_create_sequencer` returns a `DistributedTestSequencer`
(chapter 8 of this volume walks it in detail).  The sequencer
is the test-execution loop; the distributed variant differs
from the base in that it accepts a landscape parameter on
`attach_to_environment()` / `attach_to_framework()` — which is
where `InteropLibraryTestJob`'s overrides land the landscape.

### A note on the `self.title` bug

Line 74 references `self.title`, but the `TestJob` base class
defines the attribute as `self._title`.  There is no `title`
property defined anywhere in the chain.  At runtime, this
raises `AttributeError`.

This is a real bug in the source.  The book reports it
faithfully because pretending the chain is bug-free would
mislead the reader.  The fix is straightforward — change
`self.title` to `self._title` (or expose a `@property title`
on `TestJob`).  An SDET reading this chapter and running into
the same issue should know they're not imagining it.

The bug also illustrates a benefit of the layered chain: the
fix is a one-line change in one file.  Before the c785e9b
refactor, the same bug — if it existed — could have been in
the duplicated copy and gone uncaught.

## Layers 5, 6, and 7 — the naming-only fallbacks

`DefaultTestJob` (in testbase, line 613),
`DefaultIntegratedTestJob` (in integration-runtime, line 91),
and `DefaultDistributedTestJob` (in testplus, lines 84–90) are
trivial:

```python
class DefaultDistributedTestJob(DistributedTestJob):
    """
        The DefaultDistributedTestJob is utilized as a job container when
        a job was not specified.
    """
    name = "Test Job"
    description = "Unspecified test job."
```

They exist so that the entry points
(`generic_test_entrypoint`, `integrated_test_entrypoint`,
`distributed_test_entrypoint`) have a reasonable default
`job_type=` value.  A user with no custom job-type subclass uses
the default; a user with a custom subclass passes
`job_type=YourCustomJob`.

## The customisation cascade

Volume 0 chapter 1's meta-thesis: framework supplies the
skeleton and patterns; the organisation customises and refines.
The chain above is the *concrete realisation* for distributed
testing.

A project building a custom distributed test scenario:

1. **Pick the right starting layer.**
   - Single-host, no-activation customisation (stdout recorder,
     `.output` artefacts) → subclass `TestJob`.
   - Single-host customisation that needs activation-backed
     presence-folder recording, `RuntimeVariables` metadata, and
     an HTML summary → subclass `IntegratedTestJob` (and run via
     `integrated_test_entrypoint` so activation happens first).
   - Landscape-aware single-pod customisation that doesn't need
     a custom sequencer → subclass `InteropLibraryTestJob` and
     implement `_get_landscape()`.
   - Distributed multi-host customisation → subclass
     `DistributedTestJob` (and override `_create_sequencer()`
     if you need a custom sequencer).

2. **Override the provider methods you need.**
   - `_get_landscape()` to return your `Landscape` subclass
     (typically via `LandscapeSingleton()` after registering
     your subclass through the factory).
   - `_create_sequencer()` for a custom sequencer.
   - `_create_recorder()` for a custom recorder (stdout vs. JSOS
     + HTML is itself decided here — `TestJob` defaults to
     `StdoutResultRecorder`, `IntegratedTestJob` to
     `JsonResultRecorder`).
   - `_create_output_context()` / `_create_run_metadata()` for a
     custom artefact root or run-identity source.
   - The five `fire_*` hooks if you need extra activation
     behaviour beyond what landscape activation provides
     (rare — usually the layered base classes suffice).

3. **Hand the subclass to the entry point.**
   ```python
   distributed_test_entrypoint(job_type=YourCustomJob)
   ```

The entry point is `codecraftai-testplus/source/packages/ccai/testplus/entrypoints.py:38`
for distributed jobs,
`codecraftai-integration-runtime/source/packages/ccai/integrationruntime/entrypoints.py`
(`integrated_test_entrypoint`) for activated integration jobs,
and `codecraftai-testbase/source/packages/ccai/testbase/entrypoints.py:135`
(`generic_test_entrypoint`) for standalone single-host.  Real
scaffolding examples live in
`codecraftai-testbase/source/tests/ccai_tests/testbase/integration/test_custom_job_type_integration.py`
(`CustomTestJob`, `FailingCustomTestJob`).

## Not in the chain — `OrchestrationJob`

A point of vocabulary worth re-emphasising (Volume 3 chapter 1
also makes it): `OrchestrationJob` (in
`codecraftai-landscaping/source/packages/ccai/landscaping/orchestrationjob.py:23`)
is **not a `TestJob` subclass**.  It inherits from
`ContextUser`.

`OrchestrationJob` is for automation tasks that need a landscape
but are not tests — environment provisioning, inventory
discovery, compliance audits.  It carries job metadata
(`name`, `title`, `description`, `owner`,
`requires_landscape`, `requires_topology`, `requires_runtime`,
`requires_credentials` as class attributes) and wraps the
lifecycle in a `begin()` → `execute()` → `end()` contract
that is *not* the `fire_*` lifecycle from `TestJob`.

The two hierarchies meet at the bridge tasks of Volume 3
chapter 25 — taskplus's `DistributedTestJobTask` invokes a
`DistributedTestJob`; `OrchestrationJob` is invoked by
taskplus directly without passing through the test machinery.

When this volume says "the test job," it means
`DistributedTestJob` or its subclasses.  When Volume 3 says
"the orchestration," it means a taskplus workflow that may or
may not invoke a test job.  The two meet at the bridge; they
don't share a hierarchy.

## What the next chapter does

Chapter 5 walks the tandem-validator implementation in detail:
the `Validator` base class, `attach_to_test`, the JSOS
recording flow, the `wait_for_it` integration from Volume 0
chapter 5.  It is the implementation half of chapter 2's
single-actor + multi-observer rationale.
