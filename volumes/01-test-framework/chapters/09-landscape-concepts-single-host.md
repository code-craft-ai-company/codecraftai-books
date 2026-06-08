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

## Vocabulary: Landscape, Topology, Runtime, Pod

Before going further, four words need precise meanings.  Readers
who have used pytest fixtures or `unittest.mock` will recognise
none of these as testing-framework vocabulary — they belong to
the *automation* vocabulary, and conflating them with each other
is the most common mistake the rest of this chapter exists to
prevent.

**Landscape** answers the question *what resources exist and how
are they physically connected?*  A Landscape file enumerates the
devices, services, and other resources the suite has access to
— their addresses, their protocols, their credentials, the
network they live on.  It is the **physical inventory + wiring**.

**Topology** answers the question *how are those resources
logically organised, and what roles do they play together?*  A
Topology file says things like "Players A, B, C form a Home
Sound System" or "Nodes A, B, C are cluster members and Node D
is a spare."  It is the **functional grouping + role assignment**.

The two are distinct.  The same Landscape can underpin many
Topologies (one fleet of devices, organised differently for
different test campaigns).  The same Topology pattern can apply
to many Landscapes (the cluster-with-spare shape, applied to
different fleets).  In `codecraftai-configuration`'s loader, they
live in *separate* MergeMaps — `LANDSCAPE_CONFIGURATION_MAP` and
`TOPOLOGY_CONFIGURATION_MAP` — exactly because they are
independent inputs.

**Runtime** answers the question *how should the suite execute?*
A Runtime is a behavioural / diagnostic profile — `Normal`,
`Debug`, `Diagnostics`, `UPNP Diagnostics`, `Bluetooth
Diagnostics`.  Runtimes influence how loud the logging is, what
extra state to capture, what protocol traces to enable.  They do
*not* influence what resources exist (that is the Landscape) or
how those resources are organised (that is the Topology).

**Pod** is the runtime instance produced by combining a
Landscape with a Topology.  A Pod is what a test job actually
runs *against* — a concrete set of resources organised concretely.
Pods are the unit of horizontal scaling at the CI layer (more on
that below).

There is **no `codecraftai-topology` package**.  Topology is a
*configuration category*, not a package.  The shape of a topology
file beyond a small canonical convention (a `clusters` section
with `name` / `group` / `nodes` / `spares` keys consumed by
`LandscapeOperationalLayer._create_clusters` in
`codecraftai-landscaping`) is **left to the user to implement**.
The framework loads the file; the user decides what's in it.

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
(`codecraftai-testbase/source/packages/ccai/testbase/testjob.py:399–409`)
— they only log a `debug` line and `return`.  Volume 2's
`InteropLibraryTestJob`
(`codecraftai-interop/source/packages/ccai/interop/testing/testjob.py:22`)
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

> **A new class sits in the chain — but not on these hooks.**
> Between `TestJob` and `InteropLibraryTestJob` there is now an
> intermediate base class, `IntegratedTestJob`
> (`codecraftai-integration-runtime/source/packages/ccai/integrationruntime/integratedtestjob.py`),
> introduced by the integration-runtime package.  It is worth
> naming here so the chain is not surprising in Volume 2, but it
> is important to be precise about *what it adds*:
> `IntegratedTestJob` is about **result recording**, not
> landscape behaviour.  It overrides exactly three *provider*
> hooks — `_create_output_context()`, `_create_run_metadata()`,
> and `_create_recorder()` — to route results into the activated
> app's presence folder, stamp them with CI/build metadata from
> `RuntimeVariables`, and render the HTML summary.  It does **not**
> touch the five `fire_*` landscape hooks; those stay as no-op
> stubs all the way down to `InteropLibraryTestJob`, which is
> still where landscape activation is wired.  (See Chapter 5 for
> the recorder/output-context seams and Volume 2 for the full
> inheritance cascade.)  At the time of writing, the interop
> tier's re-parent onto `IntegratedTestJob` is still in flight:
> `InteropLibraryTestJob` currently extends `TestJob` directly
> (`codecraftai-interop/source/packages/ccai/interop/testing/testjob.py:22`),
> so a distributed run does not yet inherit the integration
> result-recording behaviour through this chain.

## What lives at the boundary

Three concepts straddle the single-host / distributed
boundary.  A single-host author writes tests against them in
shapes that work on either side.

### 1. Couplings with `establish_connectivity`

The `IntegrationCoupling` base class
(`codecraftai-testbase/source/packages/ccai/testbase/coupling/integrationcoupling.py:512`)
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
        from ccai.interop.testing.testjob import InteropLibraryTestJob
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
# codecraftai-interop/source/packages/ccai/interop/testing/testjob.py:88

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
`codecraftai-agent-kit`.

## Determinism: single actor, multiple observers

The biggest single thing distributed automation testing requires
that single-host application testing does not is *enforced
single-actor-per-environment*.  This is not a framework feature
in the implementation sense — there is no lock somewhere
preventing two tests from running.  It is a *design property*
the framework commits to and the test author honours.

The argument:

When a test interacts with a machine in a distributed
environment, the effect of the interaction takes time to
propagate to the other machines.  The test must *wait* — usually
by polling or by subscribing to an event — until the effect is
observable everywhere it needs to be observed.  If a second
actor can cause effects in the same environment during that
wait window, the order in which effects appear is not
controlled by the test.  Two effects that arrive in different
orders on different runs produce different states; the test is
no longer deterministic.

The conclusion: in any given environment, *one actor at a time*.

The framework's resolution to "but I want concurrent observation
of the system under test" is **tandem testing**: the test is the
single actor, and one or more *validators* attach to the test
scope as observers.  The validator base class lives in
`codecraftai-testbase/source/packages/ccai/testbase/tandem/validators.py`
and inherits `ValidatorCoupling`.  The framework calls
`validator.attach_to_test(testscope, suffix)`, the validator
registers its own `TestResult` alongside the test's, and when
the test runs the validator runs in tandem — watching, polling,
producing its own pass/fail outcomes recorded in the JSOS
stream (Chapter 5) the same way the test's are.

A test with three validators attached is *one test* with *four
recorded outcomes* — the test plus three observers.  No test
runs concurrently with another in the same environment.

### Where parallelism lives

Parallelism in distributed automation is at the **job level**,
not the test level.  The unit of parallelism is the **Pod**.

| Level | Parallel? | Why |
|---|---|---|
| Within a test (multiple actors) | **No** | Concurrent actors break determinism |
| Across tests in one job (one Pod) | **No** | All tests share the Pod's environment |
| Across jobs on different Pods | **Yes** | Pods are independent environments |

A CI orchestration that wants more horizontal scale provisions
*more Pods*, not *more parallel tests within a Pod*.  Pytest's
`pytest-xdist` (parallel test execution within a process) is
exactly the wrong model for a distributed automation suite.

## The CI hand-off — verified contracts

The Landscape and Topology files are not just configuration.
They are **contracts** between the CI system and the test job,
and the test job verifies both before it runs the first test.

- The **Landscape hand-off** says: "Here are the resources
  allotted to you."  The test job verifies each resource's
  *contact information is correct* — every resource declared in
  the Landscape can be reached at its declared address with its
  declared protocol using its declared credentials.  Failure
  means the CI system is wrong about what it gave you.
- The **Topology hand-off** says: "Here is the initial
  configuration of those resources."  The test job verifies the
  resources are *configured as the topology indicates* — if the
  topology says "cluster of A, B, C; spare D," verification
  confirms the cluster membership and spare assignment.  Failure
  means the CI system is wrong about how the resources were
  configured.

The framework supports this through the
`LandscapeConfigurationLayer.validate_topology()` override hook
and the `validate_topology` flag on `LandscapeActivationParams`
(default `True`).  The framework ships a stub validator; users
override it with the verification logic appropriate to their
topology schema.

Verification runs **before** the first test fires.  A test
against an unverified environment cannot be deterministic —
verification is a precondition for the single-actor model
above.  Volume 6 walks the verification mechanics in detail;
this chapter establishes the *principle*.

## Inside a Landscape file: services, devices, Features

The Landscape is *not* just a flat list of devices.  It has
two top-level sections that together enable the verification
contract above:

- **`services`** — declares the *infrastructure services* the
  devices use.  A serial-concentrator service offering serial
  connections on an IP and port; a power controller offering
  per-port power control; a credential vault.  Each service has
  a name, an address, and a type.
- **`devices`** — declares each device with its *primary access*
  (the `deviceType` and the fields that configure it: `host`,
  `credentials`, `platform`) and its *Features* (additional
  capabilities provided by the services above).

A real device declaration shape:

```yaml
- deviceType: network/ssh         # primary access type
  host: 192.168.1.30              # primary access address
  platform: linux
  credentials:
      - casey-node                # composable list of named credentials
  features:
      isolation: true                          # boolean capability flag
      serial: serial-concentrator: 1           # service-reference Feature
      power: LPC934: 1                         # service-reference Feature
  skip: false
```

Three layers in one declaration:

1. **Primary access** — `deviceType: network/ssh` plus `host`
   and `credentials`.  The `deviceType` selects the Coordinator
   that attaches the primary ProtocolExtension; SSH access here
   is *not* a Feature, it's the deviceType.
2. **Features** — additional capabilities beyond the primary
   access.  Two shapes:
   - *Boolean capability flags* — `isolation: true`.  Pure
     data; no Agent, no service.  Test code or framework can
     query "does this device have isolation?"
   - *Service-reference Features* — `serial: serial-concentrator:1`,
     `power: LPC934:1`.  Names a service from the `services`
     section + a port.  At runtime the framework resolves the
     service, instantiates the matching Agent against the
     service's address, and exposes a ProtocolExtension on the
     device (`device.serial`, `device.power`).
3. **Declarative metadata** — `platform`, `skip`, `credentials`.

### Coordinators, Agents, ProtocolExtensions, Features

Four words that need precise definitions, because casual usage
of "extension" is ambiguous in the portfolio:

- **Agents** *speak protocols.*  An SSH Agent speaks SSH; a
  UPnP Agent speaks UPnP; a Serial Agent speaks the serial
  wire protocol; a Power Agent speaks the power-control
  protocol.  The Agent is the protocol-speaking object.
- **Coordinators** *manage devices and attach Agents to
  devices.*  A Coordinator owns a class of devices (e.g., the
  SSH Coordinator owns devices reachable over SSH) and
  attaches the right Agent to each Device it manages.
- **ProtocolExtensions** *connect a specific protocol Agent to
  a Device.*  A ProtocolExtension is the binding object that
  wraps the Agent and exposes its protocol surface on the
  Device.  This is the user-facing extension point — a user
  adding a new protocol writes a ProtocolExtension that wraps
  their Agent.
- **Features** *are declared on devices in the Landscape* and
  reference services declared in the same Landscape.  A Feature
  declaration is configuration data on the Device's `features`
  tag that says "I have capability X, here's how to access it."

The full layering for a service-reference Feature:

```
Landscape file
├── services section
│     └── serial-concentrator: type=serial-concentrator, address=10.0.0.50
└── devices section
      └── device-1
            └── features tag: { serial: serial-concentrator: 1 }
                                ↓ at runtime
              device-1 instance
                ├── managed by Serial Coordinator
                └── .serial → ProtocolExtension
                                └── wraps → Serial Agent
                                              └── connects to serial-concentrator port 1
```

This also makes Landscape verification (the CI hand-off check
above) concrete.  The verification walks every Device's
Features, resolves each Feature's named service in the
`services` section, instantiates the Agent against the
service's address, and confirms it can connect.  A broken
Feature → service reference, an unreachable service, a
deviceType whose Coordinator can't reach `host` — these are
the failures verification catches before the first test fires.

### The customisation mechanism: subclass `Landscape` and `Topology`

Per the meta-thesis from Volume 0 chapter 1, the framework
supplies the skeleton and patterns; the organisation
customises.  For the environment model, the concrete
customisation mechanism is **subclassing `Landscape` and
`Topology`**.

The framework supplies:

- The base `Landscape` class — knows how to parse the
  `services` / `devices` / `features` shape and dispatch the
  built-in Features it ships support for.
- The base `Topology` class — knows how to parse the canonical
  `clusters` shape (`name` / `group` / `nodes` / `spares`)
  consumed by `LandscapeOperationalLayer._create_clusters`.
- Verification stub hooks (`validate_topology()` and friends)
  and the activation hooks driven by `TestJob.fire_*`.

The user supplies — by subclassing:

- Recognition and handling of project-specific Feature names
  (anything beyond serial / power / isolation) — by
  subclassing `Landscape`.
- Recognition and handling of project-specific Topology
  sections (anything beyond `clusters`) — by subclassing
  `Topology`.
- The verification logic for what those custom shapes mean —
  by overriding `validate_topology()` (and friends) in the
  subclasses.

The subclasses get plugged into a run by being returned from
the user's `TestJob` subclass's template methods (chapter 2's
"TestJob as customisation entry point" cascade).

This is the *concrete* mechanism behind every "left to the
user" note in the books: open schemas + subclassed handlers,
plumbed in via the user's `TestJob` subclass.  Schema openness
is the *what*; subclassing is the *how*; `TestJob` is the
*where*.

## Composability across categories

A property of `codecraftai-configuration` worth establishing here
because the rest of Volume 1's tests depend on understanding it:
**every configuration category is composable**.

The six categories Volume 3 walks — Specifications, Landscapes,
Topologies, Runtimes, Credentials, Data Profiles — all use the
same `MergeMap` overlay model.  A category-level configuration
the test job sees is the merged result of one or more named
source files at that category.  Each category's resolver
(`resolve_landscape_configuration`, `resolve_topology_configuration`,
etc.) takes a *list* of names and a *list* of files, loads each
in order, and overlays them.  Later names override earlier ones
where keys collide.

Examples:

- **Landscape:** `Checkout Resources + Environment Resources` —
  the resources that came from a pool checkout, merged with the
  fixed environment resources (servers, databases, persistent
  infrastructure).
- **Topology:** combining a cluster-membership topology with a
  gateway-routing topology to form the effective topology for a
  particular test campaign.
- **Runtime:** `UPNP Diagnostics + Bluetooth Diagnostics` —
  diagnostic profiles stack to form one effective runtime.
- **Credentials:** lab-wide credentials merged with
  experiment-specific overrides.

This composability is the property that makes the same suite
runnable on a developer laptop (light Runtime, mock Landscape)
and on a nightly full-sweep run (full Runtime, real Landscape +
Topology) without rewriting the tests.  Volume 3 chapter 14
walks the discipline at orchestration scale; this chapter just
needs the principle named, because Chapter 10's case study is
itself a worked example: the same agent-kit suite runs at
`TESTBASE_MAX_EXAMPLES_CAP=10` (a Runtime knob from Chapter 4)
without changing a single test file.

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

- Chapter 10 — Case study.  The `codecraftai-agent-kit`
  single-host suite at `TESTBASE_MAX_EXAMPLES_CAP=10`,
  finishing in 49 minutes wall time.  A working demonstration
  of every chapter that came before.
