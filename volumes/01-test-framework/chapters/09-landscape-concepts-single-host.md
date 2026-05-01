# Chapter 9 — Building Tests Around Landscape Concepts (Single-Host Edition)

> *A test framework that knows about the landscape becomes a
> distributed test framework.  A test framework that does not
> know about the landscape stays a unit test framework.  The
> hard part is the third option: a framework that is aware
> the landscape exists, ships a contract for the landscape to
> implement, but stays usable on a single host without a
> landscape at all.  This chapter is about how testbase walks
> that line.*

The previous eight chapters of Volume 1 walked testbase as a
single-host test framework.  Tests are functions, fixtures are
originators, results stream to a JSOS file, mocks stand in
where real systems would be too expensive.  Everything in those
chapters works on an SDET's laptop with no special
infrastructure.

This chapter is the bridge.  It carries the *contract* that a
distributed test framework must implement to extend testbase —
the one Volume 2 walks in detail.  But it carries it from
inside the single-host frame: what does a single-host test
look like that *would*, if the landscape were available,
exercise multi-host behaviour?  And how does testbase let a
single-host author write that test without the landscape
machinery being a hard dependency?

## Vocabulary: what is a "landscape"

In the portfolio's vocabulary, a *landscape* is the structured
description of the environment a suite runs against:

- The **devices** under test.
- The **services** the suite expects to find running.
- The **credentials** required to reach those devices and
  services.
- The **topology** — what can talk to what.
- The **configuration profile** — runtime parameters that
  influence how the suite is constituted.

Landscapes live in `automojo-landscaping` (Volume 2).
Topology lives in `automojo-topology` (Volume 6).  Credentials
live in `automojo-credentials` (Volumes 0 and 3).
Configurations live in `automojo-configuration` (Volume 0
chapter 9 and Volume 3).

A *single-host* landscape is the special case where the
"landscape" is one machine — possibly the SDET's laptop, possibly
a dedicated CI runner — and the structured description
collapses to "this host."  The interesting question is:
*how does a test express that it needs landscape concepts
without requiring the landscape package?*

## The contract: testbase's `fire_*` hooks

The contract sits on `TestJob`'s five `fire_*` lifecycle
hooks (Chapter 2):

- `fire_attach_to_framework(tseq)` — register integration
  couplings with the test framework.
- `fire_attach_to_environment(tseq)` — establish connectivity
  to the environment.
- `fire_activate_configuration()` — activate the run's
  configuration profile.
- `fire_activate_integration()` — bring up cross-host
  integration.
- `fire_activate_operations()` — start side processes that
  other tests will rely on.

On a single-host run using `TestJob` directly, the last three
are *no-op stubs*
(`automojo-testbase/source/packages/automojo/testbase/testjob.py:435–443`).
Volume 2's `InteropLibraryTestJob`
(`automojo-interop/source/packages/automojo/interop/testing/testjob.py:22`)
overrides each of them to drive landscape activation:

- `fire_activate_configuration()` calls
  `landscape.activate_configuration()` (line 114).
- `fire_activate_integration()` calls
  `landscape.activate_integration()` (line 124).
- `fire_activate_operations()` calls
  `landscape.activate_operations()` (line 134).

The hooks are the *contract*.  A single-host test never
notices they exist; a distributed test relies on them; both
use the same `TestJob` lifecycle.

## What lives at the boundary

Three concepts straddle the single-host / distributed
boundary.  A single-host author writes tests against them in
shapes that work on either side.

### 1. Couplings with `establish_connectivity`

The `IntegrationCoupling` base class
(`automojo-testbase/source/packages/automojo/testbase/coupling/integrationcoupling.py:512`)
exposes a method:

```python
def establish_connectivity(self,
                           allow_missing_devices: bool = False) -> None:
    """
        Bring the coupling online.

        :param allow_missing_devices: When True, the coupling reports
            but does not raise when one or more declared devices are
            unreachable.  Used in single-host smoke runs where the
            full device set is not present.
    """
    ...
```

The `allow_missing_devices` parameter is the boundary
mechanism.  On a distributed run with all devices present, the
default `False` is correct: a missing device is a hard failure.
On a single-host run where the test author wants to verify the
coupling's *graceful-degradation* path, `allow_missing_devices=True`
lets the coupling come online with a partial device set.

This is the Aspects pattern from Volume 0 chapter 4 once more:
the behaviour parameter is a parameter, not a decorator.  The
single-host test author and the distributed test author write
*the same coupling code* — they pass different values for the
parameter.

### 2. Mock-yielding originators

A test that needs landscape-shaped resources but runs on a
single host can attach a *mock* originator (Chapter 7):

```python
@testbase.factories.resource_factory(scope=ResourceLifespan.Test)
def device_pool_mock():
    """
        Yields a Test-scoped mock of the device pool for single-host
        tests.
    """
    pool = Mock(spec=DevicePool, mock_type=MockType.Instance)
    pool.list_devices = SideEffect(lambda: [])
    pool.acquire = SideEffect(lambda name: Mock(spec=Device,
                                                 mock_type=MockType.Instance))
    yield pool
    return
```

A test attaches `device_pool_mock` exactly as it would attach
the real device pool.  On a single-host run, the mock is what
the test sees; on a distributed run, swap the originator for
the real one (`device_pool_real`) and the test runs unchanged.

The decision of *which* originator to attach is best made at
the package level — a `single_host_factories.py` module that
exports mock originators for unit tests, a `distributed_factories.py`
module that exports real originators for the integration suite.

### 3. Landscape-conditional test markers

A test that *only* makes sense with a real landscape can be
marked:

```python
@testbase.skip_if(lambda: not is_distributed_landscape())
def test_replication_lag_is_under_threshold():
    """
        Verify replication lag across the cluster.
    """
    ...
```

The `is_distributed_landscape` callable runs at discovery time
(per Chapter 2's discovery / collection split).  On a single
host the callable returns False; the test is skipped before
the landscape package is even imported.  On a distributed
landscape it returns True and the test runs.

The trick: the callable cannot import the landscape package at
module top-level.  A safe shape is:

```python
def is_distributed_landscape() -> bool:
    """
        Return True if the running framework has a distributed
        landscape attached.  Imports landscape lazily to avoid
        single-host import failures.
    """
    try:
        from automojo.interop.testing.testjob import InteropLibraryTestJob
    except ImportError:
        rtnval = False
        return rtnval

    rtnval = isinstance(testbase.current_job, InteropLibraryTestJob)
    return rtnval
```

The `try / except ImportError` here is *not* swallowing an
exception in the Volume 0 chapter 3 sense — it is a sentinel
import-time check.  The function returns False if the
distributed framework is not installed; the function returns
False if the running job is not a distributed job; in both
cases the test is skipped.  No landscape package is required
to evaluate the predicate.

## The single-host test that would also run distributed

A test author who wants their test to run on either side
writes it to depend on the *coupling* not the *landscape*:

```python
@testbase.attach.parameter(some_billing_service, identifier="svc")
def test_invoice_round_trips(svc):
    """
        Verify an invoice round-trips through the billing service.

        Runs identically on single-host (with svc as a coupling-mock)
        and distributed (with svc as a coupling to the real billing
        service).
    """
    invoice = svc.create_invoice(amount=Decimal("100.00"))
    looked_up = svc.get_invoice(invoice.id)
    rtnval = (looked_up.amount == invoice.amount)
    assert rtnval
    return
```

The originator `some_billing_service` decides what `svc`
actually is.  The test does not know.  The test depends on the
coupling's *interface* — Volume 0 chapter 4's contracts.

This is the shape that maximises reuse.  The same test runs as
a unit test (mock coupling), as a single-host integration test
(real billing service on the same host), and as a distributed
test (real billing service on another host) — by choosing the
originator appropriately.

## Landscape-aware fire_* on the distributed side

Just so the contract is concrete: here is what
`InteropLibraryTestJob`'s overrides look like (forward
reference to Volume 2):

```python
# automojo-interop/source/packages/automojo/interop/testing/testjob.py:88

def fire_attach_to_environment(self, tseq) -> None:
    """
        Attach the sequencer to the environment, providing the
        landscape as the connection vehicle.
    """
    landscape = self._get_landscape()
    tseq.attach_to_environment(landscape=landscape)
    return
```

The landscape gets *passed in* as a kwarg to the sequencer's
`attach_to_environment`.  The sequencer accepts the landscape
via `**kwargs`; on a single-host run no landscape is passed
and the sequencer's environment attachment runs without one.
On a distributed run the landscape is the vehicle through
which couplings discover devices, fetch credentials, and
schedule cross-host activity.

The single-host author does not write the override.  The
single-host author writes tests; testbase's `fire_*` stubs
fire as no-ops; the run completes.  The distributed author
inherits `InteropLibraryTestJob` and gets the overrides for
free.

## A pragmatic split: `singlehost/` and `distributed/`

In the portfolio, large test suites split their test roots:

```
<repo>/source/testroots/<vendor>/tests/
                           ├── singlehost/      # mock coupling, fast
                           ├── distributed/     # real coupling, slow
                           └── ...
```

A run picks one root or the other.  The `singlehost/` root
has its own factory module that yields mocks; the
`distributed/` root has its own factory module that yields
real coupling instances.  The tests *themselves* may share
between the two roots — but it is more common, and clearer,
to keep them separate so a reader knows immediately which
side they are on.

The case study in Chapter 10 is the `singlehost/` root of
`automojo-agent-kit`.

## The landscape as a stable contract

The reason this design works is that *landscape is a contract,
not an implementation*.  The contract is:

- A `TestJob` subclass overrides the five `fire_*` hooks.
- The overrides accept the landscape via a template method
  (`_get_landscape()`) the subclass implements.
- The sequencer's environment / framework attachment accepts
  the landscape via `**kwargs`.

`InteropLibraryTestJob` (Volume 2) is *one* implementation of
the contract.  A different implementation — a research-grade
landscape that lives in a future package — would override the
same hooks differently and the same tests would run.  The
`fire_*` hooks are the *interface*; the landscape is the
*implementation strategy*.

This is Volume 0 chapter 4 again: protocols at the top, code
beneath.  The single-host author depends on the protocol; the
distributed author implements the protocol; neither writes
code that depends on the *other side*.

## What this chapter set up

By the end of this chapter the reader should be able to:

- Identify which `fire_*` hooks fire as no-ops on single-host
  and which become non-trivial on distributed.
- Decide whether a given test should mock the coupling, use a
  real local coupling, or skip on single-host.
- Write a test whose factory choice (mock vs. real) decides
  whether the test runs in single-host or distributed mode.
- Read a `_get_landscape()` template method (in Volume 2) and
  understand how it bridges to the single-host runner.

That fluency is the single-host half of the contract.  Volume
2 carries the distributed half.  The two volumes share the
*same* `TestJob` lifecycle, the *same* coupling base class,
the *same* JSOS streaming format, and the *same* `agents` CLI
output shape.  Only the hooks' bodies differ.

## What's next

- Chapter 10 — Case study.  The `automojo-agent-kit`
  single-host suite at `TESTBASE_MAX_EXAMPLES_CAP=10`,
  finishing in 49 minutes wall time.  A working demonstration
  of every chapter that came before.
