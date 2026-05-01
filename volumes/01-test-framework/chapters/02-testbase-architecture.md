# Chapter 2 — `testbase` Architecture

> *A test framework's architecture is the answer to a single
> question: what happens, in what order, when you run the suite?
> The interesting part of the answer is never the test functions
> themselves — those are just the leaves.  The interesting part
> is the path the runner takes from "here is a test root" to
> "here are the results."  This chapter walks that path.*

`automojo-testbase` is not a small framework.  Counting only
its own source tree it is around 40,000 lines of Python; the
test suite that exercises it is 146 test files
(`automojo-testbase/source/packages/automojo/testbase/`,
`automojo-testbase/source/tests/`).  But the architecture is
small in the sense that matters: the number of *concepts* a
reader has to hold in mind to follow a run is fewer than ten.

This chapter introduces those concepts and walks a single
non-distributed run from CLI invocation to result file.  Later
chapters revisit the same path looking at a single concept in
depth — the resource factory model in Chapter 3, the property
runner in Chapter 4, the JSOS recorder in Chapter 5.

## The five concepts

A reader who finishes this chapter should be able to define
each of the following in their own words.  Everything else in
the framework is built from them.

1. **TestRef** — a lazy, metadata-rich reference to a single
   test function.
2. **TestCollector** — the discovery component that builds a
   tree of `TestRef` objects from a test root.
3. **TestSequencer** — the component that walks the discovered
   tree through five phases, applying graph operations.
4. **TestJob** — the job-spanning runner that owns the
   lifecycle: enter → fire hooks → run → finalize → exit.
5. **JsonResultRecorder** — the recorder that writes the three
   result files (summary / stream / consolidated).

Around these five there are couplings (integration,
configuration, validator), the property runner, the agents CLI,
and the mocking module — but those are extensions of the core,
not the core.

## A test as code

Before the architecture, a single example, so that the rest of
the chapter can refer to a concrete shape.  A testbase test is a
plain Python function:

```python
# tests/example/test_arithmetic.py
import testbase

def test_addition_is_commutative():
    """
        Verify that integer addition is commutative.
    """
    a = 2
    b = 3
    rtnval = (a + b) == (b + a)
    assert rtnval
    return
```

No subclass.  No `@pytest.mark.unit`.  No fixtures imported by
default.  The function name starts with `test_`, the file lives
under a test root, and the framework finds it.  The
explicit-`return` discipline from Volume 0 chapter 1 is in play:
the function ends with `return` even though the function returns
nothing, because future-you may add a return value, and the
breakpoint anchor is worth the line.

That is the leaf.  Now the path that gets us to the leaf.

## Discovery: TestRef and TestCollector

The discovery process answers a single question: given a test
root path, what test functions exist under it?

`TestRef`
(`automojo-testbase/source/packages/automojo/testbase/testref.py:25–65`)
is the data class.  Each `TestRef` wraps a `FunctionType` and
stores:

- The function reference itself (`testfunc: FunctionType`).
- A list of *monikers* — string identifiers used to filter
  tests by name pattern at discovery time.
- A list of *pivots* — keyed parametrisation values that vary
  across instances of the same test function.
- An async flag — whether the function is `async def`.

`TestCollector`
(`automojo-testbase/source/packages/automojo/testbase/testcollector.py:60–231`)
is the orchestrator.  Given a test root, includes/excludes, and
metafilters, the collector:

1. Walks the file system under the test root.
2. Imports each test module.
3. Calls `collect_test_references()`
   (`automojo-testbase/source/packages/automojo/testbase/queries.py:29–94`)
   on each imported module.  That function uses
   `inspect.getmembers(mod, inspect.isfunction)` to find
   functions whose names match the test prefix and wraps each
   in a `TestRef`.
4. Applies the includes/excludes/metafilters to prune the
   references that don't match.
5. Builds a `TestGroup` tree — a hierarchical structure mirroring
   the package layout, with `TestRef` objects at the leaves.

The output of discovery is a `TestGroup` tree.  Nothing has run
yet; nothing has been imported except the test modules themselves.

## Sequencing: TestSequencer's five phases

Once discovery has produced a tree, the sequencer drives it
through five phases.  The phases are an enum:

```python
# automojo-testbase/source/packages/automojo/testbase/sequencing/testsequencer.py:106–150

class TEST_SEQUENCER_PHASES(IntEnum):
    Initial = 0
    Discovery = 1
    Collection = 2
    Graph = 3
    Traversal = 4
```

Each phase has a job:

- **Initial** — sequencer constructed; no work done.
- **Discovery** — the `TestCollector` walks the test root.  At
  the end of this phase the tree of `TestRef` objects exists
  but no resources have been resolved.
- **Collection** — the sequencer collects *originators* (the
  resource factories from Chapter 3) and *integrations* needed
  to satisfy the tests' declared dependencies.
- **Graph** — the sequencer builds the resource dependency
  graph and computes the order in which resources must be
  acquired and released.  This is where cyclic dependencies
  surface.
- **Traversal** — the sequencer walks the graph, executing
  tests in dependency-respecting order.

The sequencer's phases are *not* the same as the `TestJob`'s
`fire_*` hooks.  The phases describe how the sequencer thinks
about the suite; the hooks describe what events the job
broadcasts to the integrations.  A given `fire_*` hook can fire
at a particular phase boundary (the docstrings call out which
ones — `fire_diagnostic_capture_pre_testrun`'s docstring at
`testjob.py:524–529` documents its position).

## TestJob: the lifecycle owner — and the customisation entry point

`TestJob`
(`automojo-testbase/source/packages/automojo/testbase/testjob.py:50–131`)
is the singleton-like object that owns the run.  It has a
constructor (line 73) that stores the logger, the test root, the
includes/excludes, and run-level metadata.  It is used as a
context manager:

```python
with TestJob(logger=logger, testroot="tests/", ...) as job:
    job.run()
```

`__enter__`
(`automojo-testbase/source/packages/automojo/testbase/testjob.py:133`)
calls `begin()`.  `__exit__`
(`automojo-testbase/source/packages/automojo/testbase/testjob.py:139`)
calls `finalize()`.  Between them, the job:

1. **`begin()`** (line 162): sets up the results directory,
   constructs the recorder, instantiates the sequencer.
2. **`fire_attach_to_environment(tseq)`** (line 415): asks the
   sequencer to attach integrations to the environment.  On a
   single-host run the integrations may have nothing to attach
   to; on a distributed run this hook drives device wake-up.
3. **`fire_attach_to_framework(tseq)`** (line 425): asks the
   sequencer to attach integrations to the framework — register
   originators, wire up couplings.
4. **`fire_activate_configuration()`** (line 435): no-op on
   single-host.  In Volume 2's `InteropLibraryTestJob` this
   activates the landscape's configuration.
5. **`fire_activate_integration()`** (line 439): no-op on
   single-host.
6. **`fire_activate_operations()`** (line 443): no-op on
   single-host.
7. **`fire_diagnostic_capture_pre_testrun(tseq, level)`** (line
   499): captures pre-test diagnostic state — host details, env
   vars, package versions.
8. **The run itself**: the sequencer's traversal phase walks
   the tree, executing tests and streaming results.
9. **`fire_diagnostic_capture_post_testrun(tseq, level)`** (line
   447): captures post-test diagnostic state for failure
   triage.
10. **`finalize()`**: closes the recorder (which writes
    `testresults.json`), tears down integrations, releases
    resources.

The order is the contract.  Volume 2's `InteropLibraryTestJob`
inherits the order; it overrides only the no-op hooks.  A reader
who understands this lifecycle understands the distributed
runner's lifecycle too — the order doesn't change, just the
hooks' bodies.

### TestJob is the customisation entry point

`TestJob` is not just the lifecycle owner — it is *the* place
where the user customises the framework.  Per the meta-thesis
from Volume 0 chapter 1 ("framework supplies the skeleton and
patterns; the organisation customises"), `TestJob` is the
specific, concrete plug-in point.

The customisation works by **subclassing**.  The user picks a
starting layer and overrides the template methods they need:

- **`TestJob`** (single-host base, this chapter).  Override
  `_create_sequencer()`, `_create_recorder()`, or any of the
  five `fire_*` hooks for single-host customisation.
- **`InteropLibraryTestJob`** (Volume 2; in `automojo-interop`).
  Override the abstract `_get_landscape()` to return your
  project's `Landscape` subclass; the framework picks up the
  custom Features and topology shapes through that landscape.
- **`DistributedTestJob`** (Volume 2; in `automojo-testplus`).
  Override `_get_landscape()` and optionally `_create_sequencer()`
  for distributed multi-host scenarios.

The framework's entry points accept the user's subclass:

```python
# Single-host
generic_test_entrypoint(job_type=YourCustomJob)

# Distributed
distributed_test_entrypoint(job_type=YourCustomDistributedJob)
```

Real scaffolding examples live in
`automojo-testbase/source/tests/automojo_tests/testbase/integration/test_custom_job_type_integration.py`
(`CustomTestJob`, `FailingCustomTestJob`).  An SDET writing a
custom job type follows that pattern: subclass at the right
layer, override the template methods, plug into the entry
point.

The construction cascade that follows from a custom `TestJob`:

```
User-defined TestJob subclass             ← CUSTOMISATION ENTRY POINT
    │
    ├─ creates → TestSequencer (custom or default)
    │              │
    │              ├─ creates → Landscape (user's subclass — Vol 6)
    │              ├─ creates → Topology  (user's subclass — Vol 6)
    │              └─ orchestrates the lifecycle
    │
    └─ owns the lifecycle (the five fire_* hooks above)
```

Everything Vols 2 and 6 cover about Landscape and Topology
*subclassing* hangs off this cascade.  The user's `TestJob`
subclass is the integration point; the rest is what the
template methods return.

## The recorder pipeline

Results are produced by `JsonResultRecorder`
(`automojo-testbase/source/packages/automojo/testbase/results/recorders/jsonresultrecorder.py:90–183`).
The recorder is constructed in `begin()` and wired to the
sequencer; every test event the sequencer fires (test started,
test finished, error, skip) is forwarded to the recorder.

The recorder produces three artefacts:

- **`testsummary.json`** — written at finalize.  High-level pass/
  fail counts, duration, environment fingerprint.  This is what
  a green/red dashboard reads.
- **`testresults_stream.jsos`** — written *as tests finish*.
  Each test event becomes a JSON record; records are separated
  by an ASCII record-separator character.  A consumer can tail
  this file and parse incrementally.  Chapter 5 covers the
  format in depth.
- **`testresults.json`** — written at finalize.  The
  consolidated final document containing the same information
  as the stream file but reorganised for batch consumption.

The three files exist because three audiences exist: dashboards
(summary), live monitors (stream), and post-mortem analysis
(consolidated).  Producing all three is cheaper than asking the
analysis tools to recompute one from another.

## Couplings: where integrations live

A *coupling* is a piece of code that lives between testbase and
some other system.  Three families exist:

- **IntegrationCoupling** — the most common.  An integration
  coupling represents "testbase needs to know about *that*
  external thing."  Examples: a coupling to a credentials
  store, a coupling to a device pool, a coupling to a
  dataprofile registry.  The base class is at
  `automojo-testbase/source/packages/automojo/testbase/coupling/integrationcoupling.py`,
  and the public method `establish_connectivity(allow_missing_devices=False)`
  (line 512) is the place where the coupling decides whether
  it can run with whatever it has.
- **ConfigurationCoupling** — represents "testbase needs to
  consume a configuration source" — runtime files, landscape
  files, etc.
- **ValidatorCoupling** — represents "testbase needs to validate
  something against a schema, a policy, or an evolving system
  state."  Validators are the framework's *observer* mechanism:
  one or more validators attach to a test scope via
  `attach_to_test(testscope, suffix)` and run *in tandem* with
  the test, producing their own outcomes recorded alongside the
  test's result.  The base class lives in
  `automojo-testbase/source/packages/automojo/testbase/tandem/validators.py`.
  This is the substrate for **tandem testing** — the framework's
  answer to "how do you run a test deterministically against a
  distributed system without forbidding observation."  Volume 2
  covers the tandem model in depth; the single-host text in
  Chapter 9 of this volume carries the bridge.

Couplings are registered as integrations during the `Collection`
phase, attached to the framework during `fire_attach_to_framework`,
and connected during `fire_attach_to_environment`.  Chapter 3
covers the coupling lifecycle in detail when discussing how
resource factories are scoped.

Volume 2 extends the coupling model with landscape-aware
couplings.  Volume 3 extends it further with credential-store
couplings that fetch from external secret managers.  The model
is the same in all three volumes — only the kinds of things
being coupled change.

## Decorators: what's in the public API

A reader skimming a testbase test file will see a small set of
decorators.  They divide cleanly:

- **Discovery / classification** — none.  Tests are plain
  functions; no `@testbase.test` decorator exists.  The name
  prefix is the contract.
- **Resource attachment** — `@testbase.attach.parameter(...)`
  attaches a parameter (a fixture, in pytest vocabulary) to a
  specific test
  (`automojo-testbase/source/packages/automojo/testbase/declaration/attachments.py`).
- **Resource origination** — `@integration_factory()`,
  `@configuration_factory()`, and `@resource_factory()` register
  the *source* of a resource
  (`automojo-testbase/source/packages/automojo/testbase/declaration/registrations.py:45–93`).
  These are usually written in framework or extension modules,
  not test files.
- **Property-based testing** — `@given(...)` is the
  Hypothesis-style decorator declaring strategies for a test's
  parameters
  (`automojo-testbase/source/packages/automojo/testbase/properties/decorators.py:23–97`).
  Chapter 4 covers it.
- **Skip / conditional** — `@skip_if(...)`, `@only_if(...)`,
  and the metafilter family.  These are evaluated at discovery
  time and may prune the `TestRef` from the tree before the
  Graph phase.

The set is intentionally small.  Per the Volume 0 chapter 2
discipline on decorators, testbase decorators are either
metadata-attaching (`@given`, `@skip_if`) or registration
(`@integration_factory`).  None of them modify behaviour
silently.  None of them inject a retry or timeout — those are
parameters on the test or on the fixture, per the Aspects pattern
from Volume 0 chapter 4.

## Async support

Tests can be `async def`.  The `TestRef` carries an async flag,
and the property runner / sequencer call into asyncio for those
tests.  The handling is uniform — async tests live in the same
tree, get the same lifecycle, produce the same JSOS records.
The framework does not present "async tests" as a separate
category to the test author.

## Where the test root lives

Throughout testbase the *test root* is the directory that
contains test packages.  The convention across the portfolio is:

```
<repo>/source/testroots/automojo/tests/
                                 ├── singlehost/
                                 ├── distributed/
                                 └── ...
```

`automojo-agent-kit`'s test root is the basis for Chapter 10's
case study.  `automojo-testbase`'s own test root is at
`automojo-testbase/source/tests/` and demonstrates testbase
testing testbase — a useful self-application that catches
framework regressions.

## A run, end to end

Bringing the pieces together: what happens when an SDET runs

```bash
testbase run --testroot tests/ --include "test_arithmetic*"
```

(rough form — the actual CLI uses arcli; Chapter 8 walks the
syntax):

1. The arcli entry point parses the command and constructs a
   `TestJob`.
2. The job's `__enter__` runs `begin()`, building the recorder
   and the sequencer.
3. The sequencer enters the *Discovery* phase: walks
   `tests/`, imports modules, calls
   `collect_test_references()`, applies filters, builds the
   `TestGroup` tree.
4. The sequencer enters *Collection*: collects originators and
   integrations.
5. The job fires `fire_attach_to_framework` and
   `fire_attach_to_environment`.  Couplings register and
   establish connectivity.
6. The sequencer enters *Graph*: builds the resource
   dependency graph, plans acquisition order.
7. The job fires the three `fire_activate_*` hooks (no-ops
   here).
8. The job fires `fire_diagnostic_capture_pre_testrun`.
9. The sequencer enters *Traversal*: for each test in
   dependency-respecting order, the recorder receives a
   "started" event, the test runs, the recorder receives a
   "finished" event, and a record is written to the JSOS
   stream.
10. The job fires `fire_diagnostic_capture_post_testrun`.
11. `__exit__` calls `finalize()`: the recorder writes the
    summary and consolidated documents; couplings tear down.

Each numbered step has a chapter reference somewhere in this
volume.  The next chapter goes deep on step 4 and 6 — the
resource dependency graph, the way originators register, and the
way `@testbase.attach.parameter` lines up with it all.

## What's next

- Chapter 3 — Resource factories and dependency injection.
  How originators register, how `@testbase.attach.parameter`
  and `@integration_factory` differ, how the dependency graph
  is built.
- Chapter 4 — Property-based testing with `@given`.  The
  `TESTBASE_MAX_EXAMPLES_CAP` ceiling, the
  SHA-256(job_seed + ":" + test_name) per-test seed, the
  distinction between a strategy's `max_examples` and the
  cap.
- Chapter 5 — Streaming JSOS results.  The format, the three
  output files, the consumer patterns.
