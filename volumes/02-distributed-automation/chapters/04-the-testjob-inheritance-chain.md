# Chapter 4 — The `TestJob` Inheritance Chain — The Customisation Cascade

> *Three classes, each adding one responsibility, plus two
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

The chain is small.  Five classes, two of them naming-only.
The non-trivial work happens at three layers, and each layer
adds exactly one responsibility.

## The five classes

```
TestJob (testbase)                    ← single-host base
   │
   ├── DefaultTestJob (testbase)      ← naming fallback
   │
   └── InteropLibraryTestJob (interop)  ← landscape activation
          │
          └── DistributedTestJob (testplus)   ← distributed sequencer
                 │
                 └── DefaultDistributedTestJob (testplus)  ← naming fallback
```

The non-trivial-work layers are `TestJob`,
`InteropLibraryTestJob`, and `DistributedTestJob`.  The other
two are naming-only fallbacks for entry points that need a
default `job_type=`.

## Layer 1 — `TestJob` (the single-host base)

**File:** `automojo-testbase/source/packages/automojo/testbase/testjob.py:50`

`TestJob` is the foundation.  Volume 1 chapter 2 walked it in
detail; this chapter recaps what's relevant for distributed
extension.

The class establishes:

- The constructor with logger, testroot, includes/excludes,
  metafilters, job_seed, and the test module.
- `__enter__` / `__exit__` driving `begin()` / `finalize()`.
- The five `fire_*` lifecycle hooks — *all stubs on single-host*:
  - `fire_attach_to_environment(tseq)` — line 415.
  - `fire_attach_to_framework(tseq)` — line 425.
  - `fire_activate_configuration()` — line 435.
  - `fire_activate_integration()` — line 439.
  - `fire_activate_operations()` — line 443.
- The two template methods that customisation plugs into:
  - `_create_sequencer()` — returns a `TestSequencer` (the
    base) by default; subclasses override to return a
    `DistributedTestSequencer` or other variant.
  - `_create_recorder(...)` — returns a `JsonResultRecorder` by
    default.

The five `fire_*` hooks are **the contract**.  They run at
specific phase boundaries during a test run (Volume 1 chapter 2
walks the order).  On a single-host run they are no-ops; on
distributed, the subclasses below give them bodies.

The hooks are *lifecycle hooks*, not template methods.  They
run *during* the lifecycle; they do not return values that the
framework consumes.  The template methods (`_create_sequencer`,
`_create_recorder`, and the abstract `_get_landscape` introduced
in the next layer) are the *value-returning* customisation
points.

### What `TestJob` does NOT have

A point worth being explicit about because the next layer adds
it: `TestJob` has *no concept of a landscape*.  There is no
`_get_landscape()` method, no landscape parameter, no
import of `automojo-landscaping` from anywhere in `testbase`.
This is intentional — `automojo-testbase` is the foundation
tier (Volume 0); `automojo-landscaping` is upstream of it.  The
dependency arrow runs `landscaping → interop → testbase`, not
the other way.  Volume 0 chapter 4 walks the protocol-based
discipline that makes this possible.

The next layer, `InteropLibraryTestJob`, is the layer that
*introduces* the landscape concept — but does so as an abstract
hook the user (or a downstream package) implements, not as a
direct import.  The result: `testbase` stays landscape-free and
the chain remains importable on a host that doesn't have
landscaping installed.

## Layer 2 — `InteropLibraryTestJob` (the landscape-aware hinge)

**File:** `automojo-interop/source/packages/automojo/interop/testing/testjob.py:22`

`InteropLibraryTestJob` is the *hinge* of the chain.  It adds
exactly one responsibility: **landscape activation on the five
`fire_*` hooks**.  It does so without taking a hard dependency
on `automojo-landscaping` — by introducing an abstract
template method `_get_landscape()` that subclasses must
override.

### The class declaration

```python
class InteropLibraryTestJob(TestJob):
    """
        Interop-tier test-job base class.

        Extends :class:`automojo.testbase.testjob.TestJob` (single-host) with
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
        ``automojo.landscaping.wellknown``).

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
   `automojo-testplus`, which is downstream of
   `automojo-testbase`.  The duplicated code in
   `DistributedTestJob` re-implemented `TestJob` semantics in a
   downstream package — a clear inheritance smell.
3. **The hook implementation.**  The five `fire_*` overrides in
   the original `DistributedTestJob` had hard imports of
   `LandscapeSingleton`.  This made the class un-importable on
   a host without `automojo-landscaping` installed — an
   ergonomic failure at the package boundary.

The c785e9b refactor introduced `InteropLibraryTestJob` as the
intermediate layer that:

- Inherits cleanly from `TestJob` (no duplication).
- Lives in `automojo-interop` (a tier above `testbase` but below
  `landscaping`).
- Defines `_get_landscape()` as abstract, deferring the landscape
  import to the concrete subclass.

Result: `DistributedTestJob` collapsed from ~615 lines to ~90.

## Layer 3 — `DistributedTestJob` (the distributed plug-in)

**File:** `automojo-testplus/source/packages/automojo/testplus/distributedtestjob.py:24`

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
`LandscapeSingleton()` from `automojo.landscaping.wellknown`.
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

## Layers 4 and 5 — the naming-only fallbacks

`DefaultTestJob` (in testbase, line 621) and
`DefaultDistributedTestJob` (in testplus, lines 84–90) are
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
(`generic_test_entrypoint`, `distributed_test_entrypoint`) have
a reasonable default `job_type=` value.  A user with no custom
job-type subclass uses the default; a user with a custom
subclass passes `job_type=YourCustomJob`.

## The customisation cascade

Volume 0 chapter 1's meta-thesis: framework supplies the
skeleton and patterns; the organisation customises and refines.
The chain above is the *concrete realisation* for distributed
testing.

A project building a custom distributed test scenario:

1. **Pick the right starting layer.**
   - Single-host customisation → subclass `TestJob`.
   - Landscape-aware single-pod customisation that doesn't need
     a custom sequencer → subclass `InteropLibraryTestJob` and
     implement `_get_landscape()`.
   - Distributed multi-host customisation → subclass
     `DistributedTestJob` (and override `_create_sequencer()`
     if you need a custom sequencer).

2. **Override the template methods you need.**
   - `_get_landscape()` to return your `Landscape` subclass
     (typically via `LandscapeSingleton()` after registering
     your subclass through the factory).
   - `_create_sequencer()` for a custom sequencer.
   - `_create_recorder()` for a custom recorder.
   - The five `fire_*` hooks if you need extra activation
     behaviour beyond what landscape activation provides
     (rare — usually the layered base classes suffice).

3. **Hand the subclass to the entry point.**
   ```python
   distributed_test_entrypoint(job_type=YourCustomJob)
   ```

The entry point is `automojo-testplus/source/packages/automojo/testplus/entrypoints.py:38`
for distributed jobs and
`automojo-testbase/source/packages/automojo/testbase/entrypoints.py:237`
for single-host.  Real scaffolding examples live in
`automojo-testbase/source/tests/automojo_tests/testbase/integration/test_custom_job_type_integration.py`
(`CustomTestJob`, `FailingCustomTestJob`).

## Not in the chain — `OrchestrationJob`

A point of vocabulary worth re-emphasising (Volume 3 chapter 1
also makes it): `OrchestrationJob` (in
`automojo-landscaping/source/packages/automojo/landscaping/orchestrationjob.py:23`)
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
