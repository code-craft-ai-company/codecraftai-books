# Chapter 5 — Tandem Testing in Depth

> *Chapter 2 carried the rationale for single-actor + multi-
> observer execution.  This chapter walks the framework's
> `Validator` machinery — how a validator attaches to a test
> scope, how its outcome lands in the JSOS stream alongside the
> test's, and how to write one for a real distributed
> verification.*

The tandem-bicycle metaphor from chapter 2: one driver, many
observers riding along.  This chapter walks the *bicycle* —
the actual `automojo-testbase/source/packages/automojo/testbase/tandem/`
machinery that holds it together.

## The class hierarchy

```
BaseCoupling
    └── ValidatorCoupling (testbase/coupling/validatorcoupling.py:44)
            └── Validator (testbase/tandem/validators.py:33)
                    └── (user-defined subclasses)
```

The Volume 1 chapter 2 coupling families (Integration,
Configuration, Validator) are the framework's three coupling
shapes.  ValidatorCoupling is the type that explicitly *does
not* expose a write surface to the system under test — a
validator inheriting from it is structurally constrained to be
an observer.

The hierarchy matters because chapter 2's commitment to
"observers don't act" is partly enforced through types.  A
validator subclass *cannot* accidentally inherit a
`run_command` or `set_state` method from the wrong base class
because no such method exists in the `ValidatorCoupling` /
`BaseCoupling` ancestry.

## The `Validator` class

**File:** `automojo-testbase/source/packages/automojo/testbase/tandem/validators.py:33`

```python
class Validator(ValidatorCoupling):
    """
        Base class for validator implementations used with tandem tests.
    """

    def __init__(self):
        super().__init__()

        self._inst_id = str(uuid.uuid4())

        self._suffix = None
        self._testscope = None
        self._parent_scope_id = None
        self._validator_name = None
        self._recorder = None

        self._logger = getEnhancedLogger()
        return
```

The instance variables (lines 38–49) carry:

- `_inst_id` — UUID for this validator instance.  Each
  attachment is a fresh validator; the inst_id distinguishes
  multiple validators of the same class on different tests.
- `_suffix` — string identifier appended to the validator's
  name to make it unique within a test scope.  A test
  attaching three `LatencyValidator`s passes `suffix="db"`,
  `suffix="cache"`, `suffix="queue"`.
- `_testscope` — the `SequencerTestScope` the validator is
  attached to.  Set by `attach_to_test`.
- `_parent_scope_id` — the test's scope ID, used to anchor the
  validator's `TestResult` in the result tree.
- `_validator_name` — `testscope.test_name + "_" + suffix`.
  The composite name used in JSOS records.
- `_recorder` — handle to the result recorder.  Validators
  call `_recorder.preview()` and `_recorder.record()` to land
  outcomes in the JSOS stream.
- `_logger` — enhanced logger from `automojo-xmodules`.

Per the Volume 0 chapter 2 discipline, every instance variable
is initialised in `__init__`.  No late introduction.

## The lifecycle: `attach_to_test` → `initialize` → run-in-tandem → `finalize`

A validator's life has four phases.  The framework drives them
in order.

### Phase 1 — `attach_to_test(testscope, suffix)`

```python
# Lines 64–76

def attach_to_test(self, testscope: SequencerTestScope, suffix: str):
    """
        The 'attach_to_test' method is called by the sequencer in order to attach the validator to its partner
        test scope.
    """
    self._testscope = testscope
    self._suffix = suffix
    self._parent_scope_id = self._testscope._parent_scope_id
    self._validator_name = testscope.test_name + "_" + self._suffix
    self._recorder = testscope.recorder
    self._result = TestResult(self._inst_id, self._validator_name, self._parent_scope_id, self._testscope._monikers, self._testscope._pivots)
    self._recorder.preview(self._result)
    return
```

`attach_to_test` is the *binding* moment.  The framework calls
it once for each validator attached to a test, before the test
runs.  The validator:

- Stores the test scope, suffix, parent scope ID.
- Computes its composite name.
- Captures the recorder.
- Constructs a `TestResult` for itself with its own UUID, name,
  and the test's monikers + pivots.
- Calls `recorder.preview(result)` — telling the recorder
  "expect a result with this ID" so the JSOS stream's
  result-tree shape is known up front.

After `attach_to_test`, the validator is bound but not yet
initialised.  Its `TestResult` is in the recorder's preview
state but no outcome has been written.

### Phase 2 — `initialize(*args, **kwargs)`

```python
# Lines 78–91

def initialize(self, *args, **kwargs):
    """
        The 'initialize' method should attempt to initialize the validator but it should not allow an exception
        to bubble up...
    """
    self._logger.warning(
        "Default implementation of `Validator.initialize` invoked.  "
        "Subclasses should override.  args=%r kwargs=%r",
        args, kwargs,
    )
    return
```

`initialize` is the validator's setup hook.  The base
implementation logs a no-op warning; subclasses override to
set up state — open a log tail, subscribe to an event stream,
start a polling thread.

The base body's warning is deliberate: a validator with a
no-op `initialize` is almost always a mistake.  The warning
catches the omission early; a project that legitimately wants
a no-op initialize overrides explicitly with a `pass`.

### Phase 3 — Run in tandem with the test

This is where the metaphor lands.  The test runs.  The
validator runs *concurrently* with the test, watching for
effects.  Different validators do different things during this
phase:

- A polling validator runs a `wait_for_it` loop watching for a
  state change.
- A subscribing validator listens on an event channel.
- A snapshot validator takes periodic snapshots.

Crucially, the validator's run-in-tandem behaviour is
*subclass-specific*.  The base `Validator` class doesn't
prescribe a shape — it provides the lifecycle hooks
(`initialize`, `validate`, `finalize`) and lets the subclass
decide what watching means.

The `LoopingValidator` subclass (line 232 of the same file)
demonstrates one pattern — a worker thread that the validator
manages with `wait_for_it` for shutdown:

```python
# Excerpt — LoopingValidator.wait()

wait_for_it(
    self._wait_for_shutdown,
    what_for="LoopingValidator Shutdown",
    timeout=self._shutdown_timeout,
    interval=self._shutdown_interval,
)
```

The `wait_for_it` primitive comes from Volume 0 chapter 5.  It
polls a callable until either the callable returns a truthy
value or the timeout elapses.  `LoopingValidator` uses it to
wait for its background thread to finish on shutdown.

Other patterns: a `ThresholdValidator` that records the maximum
observed value during the test and asserts a bound at the end;
a `LogScanningValidator` that tails a log and flags errors.
Each is a subclass of `Validator` (or one of `Validator`'s own
subclasses) that overrides `initialize`, an instance method
that runs in tandem, and `validate` (called by `finalize`
below).

### Phase 4 — `finalize()`

```python
# Lines 94–131

def finalize(self):
    """
        The 'finalize' method is called by the sequencer after the test has completed in order to allow
        the validator to perform its checks and record the result.
    """
    try:
        self.validate()
        self._result.mark_passed()
    except AssertionError as cause:
        # ... format traceback, mark failed
        self._result.mark_failed(...)
    except Exception as cause:
        # ... format traceback, mark errored
        self._result.mark_errored(...)

    self._recorder.record(self._result)
    return
```

`finalize` is called by the sequencer *after the test has
completed*.  It:

- Calls `validate()` — the subclass-implemented method that
  asserts the validator's check.
- Catches `AssertionError` (test-style failure) and marks the
  validator's result as failed.
- Catches generic `Exception` (validator broke) and marks the
  result as errored.
- Otherwise marks the result as passed.
- Calls `recorder.record(self._result)` — committing the
  validator's outcome to the JSOS stream.

After `finalize`, the validator's `TestResult` is in the JSOS
stream as a sibling of the test's own result.  The result
tree now contains *N+1 results* for a test with *N
validators*: one for the test, one for each validator.

## How the outcome lands in the JSOS stream

Volume 1 chapter 5 walked the JSOS streaming format.  A
validator's outcome lands in the same stream as the test's,
with the same record shape.  The differences:

- The validator's `TestResult` carries the validator's
  `_inst_id` (UUID), not the test's.
- The validator's name is `<test_name>_<suffix>` — composite.
- The result-tree structure links the validator under the
  test's parent scope, so the consumer can navigate from the
  test to its validators.

A consumer reading the JSOS stream sees, for one test with
three validators:

```
{kind: test_started, name: test_replication_lag, ...}
{kind: validator_started, name: test_replication_lag_metric, ...}
{kind: validator_started, name: test_replication_lag_log, ...}
{kind: validator_started, name: test_replication_lag_health, ...}
{kind: test_finished, name: test_replication_lag, outcome: passed, ...}
{kind: validator_finished, name: test_replication_lag_metric, outcome: passed, ...}
{kind: validator_finished, name: test_replication_lag_log, outcome: failed, ...}
{kind: validator_finished, name: test_replication_lag_health, outcome: passed, ...}
```

The test passed.  Two validators passed.  One validator
(`test_replication_lag_log`) failed — meaning the test's
*action* succeeded but one of the *observers* caught something
the test didn't check for.

This is the tandem model's payoff: a test can *pass* its own
assertions while a validator catches a steady-state violation
the test wasn't even looking at.  The aggregate run report
(chapter 13) treats the validator failure as a real failure;
the test's passing assertions don't override it.

## Writing a custom validator

A worked example.  Suppose a distributed test reboots a host
and verifies the cluster recovers.  The test's *action* is the
reboot; the test's *check* is "cluster reports healthy after
reboot."  But during the reboot, the cluster might *briefly
report degraded* — and a strict observer should catch that and
mark it.

```python
import time
from automojo.testbase.tandem.validators import Validator
from automojo.errors.exceptions import SemanticError

class DegradedDurationValidator(Validator):
    """
        Validates that the cluster's degraded period during a
        reboot test does not exceed a threshold.
    """

    def __init__(self, max_degraded_seconds: float = 30.0) -> None:
        super().__init__()
        self._max_degraded_seconds = max_degraded_seconds
        self._degraded_started_at: Optional[float] = None
        self._degraded_total: float = 0.0
        self._observation_thread: Optional[threading.Thread] = None
        self._stop_event = threading.Event()
        return

    def initialize(self, cluster_endpoint: str) -> None:
        """
            Start the observation thread polling cluster health.
        """
        self._cluster_endpoint = cluster_endpoint

        def _observe():
            while not self._stop_event.is_set():
                health = check_cluster_health(self._cluster_endpoint)
                now = time.monotonic()
                if health == "degraded" and self._degraded_started_at is None:
                    self._degraded_started_at = now
                elif health != "degraded" and self._degraded_started_at is not None:
                    self._degraded_total += now - self._degraded_started_at
                    self._degraded_started_at = None
                time.sleep(0.5)
            return

        self._observation_thread = threading.Thread(
            target=_observe, daemon=True
        )
        self._observation_thread.start()
        return

    def validate(self) -> None:
        """
            Stop observation and assert degraded duration is within threshold.
        """
        self._stop_event.set()
        if self._observation_thread is not None:
            self._observation_thread.join(timeout=5.0)

        # Close out any in-flight degraded period.
        if self._degraded_started_at is not None:
            self._degraded_total += time.monotonic() - self._degraded_started_at

        rtnval = self._degraded_total <= self._max_degraded_seconds
        assert rtnval, (
            "Cluster was degraded for {:.2f}s, exceeds threshold of {:.2f}s".format(
                self._degraded_total, self._max_degraded_seconds
            )
        )
        return
```

The validator:

- Initialises a daemon thread polling cluster health every 500ms.
- Tracks the cumulative degraded duration during the test.
- In `validate`, stops the thread, computes the total degraded
  time, asserts it's under the threshold.
- The framework's `finalize` catches the `AssertionError`,
  marks the result failed with a clear message, records it to
  the JSOS stream.

A test using this validator:

```python
@testbase.attach.parameter(cluster_endpoint, identifier="endpoint")
@testbase.attach.validator(DegradedDurationValidator(max_degraded_seconds=30.0),
                            suffix="degraded_duration")
def test_cluster_recovers_from_reboot(endpoint):
    """
        Verify the cluster recovers within the SLA window.
    """
    target_host = pick_a_node(endpoint)
    reboot_host(target_host)
    wait_for_cluster_healthy(endpoint, timeout=120)
    rtnval = cluster_is_healthy(endpoint)
    assert rtnval
    return
```

The test asserts "cluster is healthy after the reboot" — a
single check at the end.  The validator asserts "cluster was
degraded for less than 30 seconds during the reboot" — a
cumulative observation across the test's duration.  The test
might pass its assertion (the cluster *is* healthy at the end)
while the validator fails (the cluster was degraded for too
long during the test).  The aggregate run reports both.

## When to use a validator vs put the check in the test body

A natural question: why use a tandem validator at all?  Why
not just put the check in the test?

Three reasons.

### 1. Continuous observation that the test cannot do while acting

The test is the actor.  While the test is calling
`reboot_host(target_host)`, the test cannot also be polling
cluster health — the call is synchronous, the test code is
blocked.  A validator runs *in parallel with the test* (in a
separate thread / coroutine that the validator manages),
collecting observations the test cannot collect.

### 2. Steady-state checks distinct from the action's checks

A reboot test's action-check is "after the reboot, the cluster
is healthy."  A reboot test's *steady-state* check is "during
the reboot, the cluster's degraded period was bounded."  These
are *different properties* and they belong in *different
result records*.  The validator gives the steady-state check
its own pass/fail outcome that doesn't depend on the test's
outcome.

### 3. Reusability across tests

A `DegradedDurationValidator` written once attaches to *every*
test that does anything to the cluster — reboot, kill, network
partition, drive failure.  The test bodies don't repeat the
validator's logic.  A change to "what counts as degraded" or
"what the threshold is" happens in one class, not N tests.

## What the next chapter does

Chapter 6 walks landscape activation in detail: what happens
when `InteropLibraryTestJob.fire_activate_configuration()` is
called, what `Landscape.activate_configuration()` does, what
the three layers (Configuration / Integration / Operational)
each contribute.  It is the implementation half of Volume 1
chapter 9's landscape boundary.

Chapter 7 follows with the verification contract — the
load-bearing precondition that makes the determinism model
from chapter 2 actually deterministic.
