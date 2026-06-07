# Chapter 8 — The `DistributedTestSequencer`

> *The base `TestSequencer` walks tests in declaration order;
> the `DistributedTestSequencer` does the same.  This chapter
> walks what the distributed sequencer adds — landscape
> awareness on the `attach_to_*` methods — and what it
> deliberately does not add: parallelism.*

The base `TestSequencer` from Volume 1 chapter 2 is the
test-execution loop.  It walks the discovered test tree,
acquires resource factories, runs each test in declaration
order, records results.  It is the heart of testbase.

The `DistributedTestSequencer` is the distributed test job's
sequencer.  A reader's natural assumption — that the
distributed sequencer parallelises tests across hosts or
workers — is the wrong one.  This chapter explains why, walks
what the distributed sequencer *actually* adds, and connects
the design to chapter 2's determinism commitment.

## Class declaration

**File:** `codecraftai-testplus/source/packages/ccai/testplus/distributedtestsequencer.py:33`

```python
class DistributedTestSequencer(TestSequencer):

    def __init__(self,
                 jobtitle: str,
                 root: str,
                 includes: Sequence[str],
                 excludes: Sequence[str],
                 metafilters: List[MetaFilter],
                 job_seed: int = None):
        super().__init__(
            jobtitle=jobtitle,
            root=root,
            includes=includes,
            excludes=excludes,
            metafilters=metafilters,
            job_seed=job_seed,
        )

        self._landscape = None
        return
```

Two things to notice:

1. The class extends `TestSequencer` directly.  It does not
   override the test-execution loop.  The base sequencer's
   `execute_tests()` (in
   `codecraftai-testbase/source/packages/ccai/testbase/sequencing/testsequencer.py:412`)
   walks tests in declaration order, one at a time, exactly as
   the single-host version does.
2. The constructor adds one new instance variable:
   `_landscape = None`.  This is the slot for the landscape
   the distributed sequencer will receive on
   `attach_to_environment` / `attach_to_framework`.  It is
   initialised to None per the Volume 0 chapter 2 discipline
   (every instance variable initialised in `__init__`).

## What the distributed sequencer adds: landscape-aware coupling attachment

```python
# Lines 57–70

def attach_to_framework(self, landscape=None):
    """
        Attach the framework's integrations to the landscape.
    """
    self._landscape = landscape

    integ_coupling_table = self._integration_couplings
    for integ_key, integ_coupling in integ_coupling_table.items():
        integ_coupling.attach_to_framework(landscape)

    return
```

```python
# Lines 72–106

def attach_to_environment(self, landscape=None):
    """
        Attach the framework's integrations to the environment.
    """
    self._landscape = landscape

    # Walk the integration couplings; let each one attach to the environment
    # given the landscape now available to them.
    integ_coupling_table = self._integration_couplings
    for integ_key, integ_coupling in integ_coupling_table.items():
        integ_coupling.attach_to_environment(landscape)

    # ... additional landscape-aware setup ...

    return
```

The two methods accept a `landscape` parameter, store it on
the sequencer, and propagate it to every registered
integration coupling.  This is the *only* substantive
difference between `DistributedTestSequencer` and base
`TestSequencer`.

The semantic difference: in the base sequencer, integration
couplings attach without a landscape (they get None or are
never called for landscape-using methods).  In the distributed
sequencer, integration couplings receive the landscape and
can use it to discover devices, fetch credentials, register
tandem validators against specific hosts, etc.

The flow that delivers the landscape:

```
DistributedTestJob (testplus)
    └── _create_sequencer() returns a DistributedTestSequencer
        ↑
        │ called by TestJob.execute()
        │
InteropLibraryTestJob (interop)
    └── fire_attach_to_environment(tseq):
            landscape = self._get_landscape()
            tseq.attach_to_environment(landscape=landscape)  ← landed here
```

The sequencer never *constructs* a landscape.  It receives
one from the test job's `fire_*` overrides.  The separation is
chapter 4's customisation cascade in action: the test job
chooses which landscape (via `_get_landscape()`); the sequencer
just wires the landscape through.

## What the distributed sequencer does NOT add: parallelism

The framing question: *if the distributed sequencer doesn't
parallelise tests, why does it exist?*

The answer matters because it's the framework's clearest
statement of the determinism model from chapter 2.  The
distributed sequencer:

- Does not parallelise tests within a Pod.
- Does not have a concept of "workers" or "concurrent test
  execution."
- Does not contain a thread pool, an asyncio event loop, or
  any parallelism primitive for test execution.

It exists to be **landscape-aware**, not landscape-parallel.
The landscape gives integration couplings the information they
need to operate against multiple hosts; the *test-execution
loop* still runs one test at a time because the determinism
commitment requires it.

A reader looking for parallelism in the source will find it in
two places:

1. **Inside the wire layer** (Volume 5).  When a test calls
   into multiple hosts, the SSH agent (chapter 10 of this
   volume) may issue concurrent commands across hosts.  This
   is *intra-test* concurrency — the test, as the single
   actor, distributes its own actions.  The Pod's other
   actors are still excluded.
2. **Across Pods, via orchestration** (Volume 3).  Multiple
   Specifications produce multiple Pods; multiple
   `DistributedTestJob` processes run concurrently against
   independent Pods.  This is the *Pod-level parallelism* of
   chapter 3.

Neither of these is in the sequencer.  The sequencer's job is
serial test execution within a Pod.

## What a distributed run looks like through the sequencer

Walking the path from `DistributedTestJob.__enter__` to the
first test:

1. `__enter__` calls `begin()`.  The job constructs the
   recorder.
2. `begin()` calls `_create_sequencer()` — returns a
   `DistributedTestSequencer`.
3. The job fires `fire_attach_to_environment(tseq)`.  The
   `InteropLibraryTestJob` override fetches the landscape via
   `_get_landscape()` and passes it to
   `tseq.attach_to_environment(landscape=landscape)`.
4. The sequencer's `attach_to_environment` stores the
   landscape, walks every integration coupling, and calls
   `integ_coupling.attach_to_environment(landscape)`.  Each
   coupling now has the landscape and can do its work.
5. The job fires `fire_attach_to_framework(tseq)`.  Same shape
   — sequencer propagates the landscape to couplings'
   `attach_to_framework`.
6. The job fires the three `fire_activate_*` hooks.  Each
   delegates to the landscape (chapter 6's three layers).
   After these complete, the Pod is *operationally live*.
7. The job fires `fire_diagnostic_capture_pre_testrun`.  The
   sequencer captures pre-test diagnostic state.
8. The sequencer enters its *Traversal* phase.  For each test
   in declaration order:
   - The recorder receives a *test_started* event.
   - The test runs.  Tandem validators (chapter 5) attached to
     the test scope run in tandem.
   - The test returns; tandem validators finalise.
   - The recorder receives *test_finished* and per-validator
     events.
9. The sequencer's traversal ends.  The job fires
   `fire_diagnostic_capture_post_testrun`.
10. `__exit__` calls `finalize()`.  The recorder writes
    summary and consolidated documents.  The job tears down.

Step 8 is sequential.  N tests in the suite mean N iterations
of the traversal loop, one at a time.  The framework does not
expose a knob to make step 8 concurrent.

## Why this design holds up at scale

Three properties make the serial-within-Pod design hold up
even on large suites:

### Test bodies dominate framework overhead

Volume 1 chapter 10 measured the runner's per-test overhead at
single-digit milliseconds.  A test body that takes 5 seconds
spends 99.98% of its time in the body, not the framework.
Adding parallelism to the framework's loop would optimise the
0.02%; the cost (broken determinism) outweighs the benefit.

### Test concurrency lives inside the test

A test that needs to drive ten hosts in parallel does so
*inside the test body*, by issuing SSH commands concurrently
through the SSH agent (chapter 10).  The test, as the single
actor, distributes its own actions.  The other actors (the
ten hosts) are not framework actors; they are the system
under test responding to the actor's commands.

### Pod fan-out is the right granularity for horizontal scale

When a campaign needs more parallelism, the answer is more
Pods.  Volume 3 chapter 4's Specifications are how Pods are
provisioned; chapter 3 of this volume walked why Pod-level
parallelism is the only safe parallelism level.

So: the sequencer stays serial; the *test body* gets
intra-test concurrency through the wire; *Pod fan-out*
provides horizontal scale.  Three layers, each appropriate.

## Differences from the base sequencer summarised

| Feature | Base `TestSequencer` | `DistributedTestSequencer` |
|---|---|---|
| Test discovery | Yes | Yes (inherited) |
| Resource factory acquisition | Yes | Yes (inherited) |
| Sequential test execution | Yes | Yes (inherited; not overridden) |
| `job_seed` reproducibility | Yes | Yes (inherited) |
| `attach_to_framework(landscape)` | Without landscape param | With landscape param, propagates to couplings |
| `attach_to_environment(landscape)` | Without landscape param | With landscape param, propagates to couplings |
| `_landscape` instance variable | No | Yes (initialised None) |
| Parallel test execution | No | No |

The table is short because the difference is small.  The
distributed sequencer is the base sequencer with a landscape
slot and two methods that take a landscape parameter.
Everything else — including the determinism-preserving serial
execution — is inherited verbatim.

## Implications for custom sequencers

A project that wants a custom sequencer subclasses
`DistributedTestSequencer` (or, for non-distributed but
landscape-aware scenarios, `TestSequencer`).  The custom
sequencer plugs into a `DistributedTestJob` subclass via the
`_create_sequencer()` template method (chapter 4).

Reasons to subclass:

- **Custom integration handling.**  The default `attach_to_*`
  walks every coupling and calls its method; a subclass can
  intercept to add ordering, filtering, parallelism *of the
  attachment* (different from parallelism of test execution),
  or extra logging.
- **Custom diagnostic capture.**  The base capture is
  delegated to the recorder; a subclass can add per-host
  snapshots taken at phase boundaries.
- **Custom traversal logging.**  Adding rich per-test
  pre-flight checks at the sequencer level (rather than
  fixture level) when those checks span couplings.

What *not* to do in a custom sequencer subclass: introduce
test-level parallelism.  The framework's contract with the
test author is that tests run serially within a Pod.  A
sequencer that violates the contract violates chapter 2's
commitment and breaks every downstream assumption.

## What the next chapter does

Chapter 9 walks `job_seed` and reproducibility.  Volume 1
chapter 4 introduced the per-test seed
(SHA-256(job_seed + ":" + test_name)) for property-based
tests; chapter 9 of this volume walks how a single seed
survives the trip across machines in a distributed run, and
why deterministic input generation across N hosts requires
exactly the discipline the sequencer's serial execution
provides.
