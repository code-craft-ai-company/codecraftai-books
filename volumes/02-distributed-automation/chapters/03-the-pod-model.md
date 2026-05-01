# Chapter 3 — The Pod Model — Where Parallelism Actually Lives

> *Tests within a Pod run sequentially because the Pod is a
> shared environment.  Jobs across independent Pods run in
> parallel because Pods are independent environments.  The unit
> of horizontal scaling in distributed automation is the Pod —
> not the test, not the host, not the runner.*

The previous chapter established the single-actor determinism
commitment: within any environment, one actor at a time.  An
honest reader's next question is: *if I can't parallelise tests,
how does my CI run faster?*

The answer is the **Pod model**.

## What a Pod is

A **Pod** is the runtime instance produced by combining a
**Landscape** (the physical inventory and wiring — Volume 6
chapter 1) with a **Topology** (the logical organisation and
roles — Volume 6 chapters 5 and 6).  A Pod is a concrete,
interoperating set of resources that a job runs against.

Volume 1 chapter 9 introduced the vocabulary; Volume 6 walks the
schema in detail.  This chapter establishes what a Pod is *for*:
**a Pod is the unit of horizontal CI parallelism.**

A Pod has the following properties:

1. **Internal interoperability.**  Within a Pod, the
   resources can talk to each other — the Landscape and Topology
   describe their relationships and the framework activates them.
2. **External isolation.**  Pods do not share resources.  A
   device in Pod A is not in Pod B.  A service in Pod A is not
   in Pod B.  An effect caused in Pod A cannot propagate to Pod
   B.
3. **Single-actor execution.**  A Pod has one test job
   running against it at a time (chapter 2's commitment); inside
   that job, tests run sequentially.

The third property is what makes the second property load-bearing.
Pods don't share resources *and* each Pod is single-actor →
**different Pods cannot affect each other** → jobs across Pods
can run in parallel without violating determinism.

## The parallelism table

| Level | Parallel? | Why |
|---|---|---|
| Within a test (multiple actors against the same environment) | **No** | Concurrent actors break determinism (chapter 2) |
| Across tests in one job (one Pod) | **No** | All tests in a job share the Pod's environment |
| Across jobs on different Pods | **Yes** | Pods are independent environments; effects do not cross |

The table is the framework's complete answer to the parallelism
question.  Three rows, two no's, one yes.  The yes is at the
Pod level; the no's are at every finer granularity.

## How Pods get provisioned

A Pod is realised at the orchestration layer.  Volume 3 chapter
4 walks the Specification configuration category — the
declarative manifest that tells CI tooling "I need this set of
resources to form an automation Pod."  CI tooling reads the
Specification, performs the checkout-or-create, and emits the
Landscape and Topology files for the next stage.  Those files
are the Pod's static description; the runtime instantiation
into a *running* Pod happens when a `DistributedTestJob`
activates the Landscape and Topology (chapter 6 of this
volume).

The orchestration-side flow:

```
Specification file (declares "I need a Pod of 5 hosts")
        ↓ CI tooling
   checkout / create resources
        ↓
   emits Landscape file + Topology file
        ↓ orchestration triggers DistributedTestJob
        ↓
   InteropLibraryTestJob.fire_activate_configuration()
   InteropLibraryTestJob.fire_activate_integration()
   InteropLibraryTestJob.fire_activate_operations()
        ↓
   Pod is now realised — devices online, services reachable,
   topology verified
        ↓
   tests run sequentially against this Pod
        ↓
   Pod is torn down (or returned to the pool)
```

A "10-fold parallel CI" in this model means **ten Specifications,
ten Pods, ten DistributedTestJobs running concurrently**.  Each
Specification produces one Pod; each Pod runs one job; the ten
jobs do not affect each other because the ten Pods are
independent.

## A worked example — three Pods running in parallel

Suppose a CI run wants to verify a sysman feature against three
OS variants: RHEL, Ubuntu, macOS.  Each variant needs a 5-host
Pod (one controller + four DUTs).

Three Specifications declare three Pods:

```yaml
# Specification A — RHEL Pod
pod:
    name: rhel-feature-test-pod
    resources:
        - controller-rhel-1
        - dut-rhel-1
        - dut-rhel-2
        - dut-rhel-3
        - dut-rhel-4
```

```yaml
# Specification B — Ubuntu Pod
pod:
    name: ubuntu-feature-test-pod
    resources:
        - controller-ubuntu-1
        - dut-ubuntu-1
        # ... etc.
```

```yaml
# Specification C — macOS Pod
pod:
    name: macos-feature-test-pod
    resources:
        - controller-macos-1
        - dut-macos-1
        # ... etc.
```

CI runs all three Specifications in parallel.  Each emits its
own Landscape and Topology.  Three `DistributedTestJob`
processes run concurrently — one per Pod.  Inside each job,
the sysman feature tests run sequentially.

Three Pods.  Three jobs in parallel.  Within each job, tests
run one at a time.  No tests-in-parallel anywhere — but
three-jobs-in-parallel.

If the sysman feature suite for one OS takes thirty minutes
sequentially, the cross-OS run finishes in *thirty minutes
total* (parallel across Pods), not ninety (serial across Pods).
Three-fold horizontal scaling at the Pod level; zero
test-level parallelism within a Pod.

## What this means for CI scaling

The framework's stance on horizontal scaling is uniform:

> *To go faster, provision more Pods.  Do not parallelise
> tests within a Pod.*

This is a stance that surprises engineers coming from
application-test traditions where pytest-xdist's
`--workers=N` option is the default scaling answer.  The
framework's answer is different because the problem is
different — application tests don't share state across processes,
so parallel pytest workers don't race; distributed automation
tests *do* share state (the Pod's resources), and parallel
workers within a Pod *would* race.

The stance has consequences:

- **Specifications become first-class infrastructure.**
  Volume 3 chapter 4 walks the Specification category as the
  *unit of horizontal scale*.  A team that wants their CI to
  finish twice as fast doesn't double their pytest workers;
  they double their Specifications.
- **Pod isolation must be real.**  A Pod that secretly shares a
  service with another Pod (a database, a license server, a
  shared queue) is not really a Pod.  The framework's
  Specification-to-Landscape pipeline in Volume 3 chapter 4 is
  pessimistic about shared services for this reason.
- **Pod size becomes a tuning parameter.**  A 5-host Pod
  finishes a sysman suite in 30 minutes; a 10-host Pod finishes
  it in 18 minutes (more parallelism within the suite at the
  *test design* level, not at the framework level).  The
  framework's job is to be honest about the trade-off; a
  larger Pod amortises Pod-setup cost over more tests but is
  more expensive to provision.

## What this rules out

The Pod model rules out a few patterns that look attractive
but break the determinism commitment:

### "Parallel tests with shared fixtures"

```python
# Wrong — two tests racing on shared fixture state.
@pytest.mark.parametrize("scenario", scenarios)
@pytest.mark.parallel(workers=4)  # never use this
def test_scenario(scenario):
    ...
```

If `scenarios` exercises shared fixture state in a real Pod,
the four workers race.  The right answer in distributed
automation is to provision one Pod per scenario — four Pods,
four jobs, parallel jobs, sequential tests within each job.

### "Background actors" running between tests

```python
# Wrong — the background actor is a second actor in the Pod.
def setup_module():
    threading.Thread(target=monitor_drift, daemon=True).start()
```

A daemon thread inside the test process can be an *observer*
(if it only watches and records).  But if the daemon ever
*acts* — sends a heartbeat, restarts a service — it becomes a
second actor in the Pod and breaks determinism.  The right
shape for ongoing observation is a tandem validator (chapter
5).

### "Cross-Pod dependencies" in a single job

```python
# Wrong — one test reaches into a different Pod's service.
def test_cross_pod():
    pod_a_service = LandscapeSingleton().get_pod("pod-a").service("X")
    pod_b_service = LandscapeSingleton().get_pod("pod-b").service("X")
    # ... acts on both
```

A test acting on two Pods is acting on what should be a single
larger Pod.  Either combine them into one Pod (and accept the
serial execution) or split the test (and accept the
cross-Pod sequencing concern, which is an *orchestration*-level
problem — Volume 3 chapter 22's flow control covers it).

## What the next chapter does

Chapter 4 walks the `TestJob` inheritance chain in detail.  The
chain is the *customisation cascade* (Volume 0 chapter 1's
meta-thesis applied here): `TestJob` (testbase) →
`InteropLibraryTestJob` (interop) → `DistributedTestJob`
(testplus).  Each layer adds one responsibility.  The chain is
how a project plugs in its custom `Landscape` and `Topology`
subclasses.

Chapter 5 then walks the tandem-validator implementation in
depth — the actual `automojo-testbase/.../tandem/validators.py`
machinery, with the real `attach_to_test` signature and the
JSOS recording flow.

By the end of those two chapters, a reader will have the full
picture of *how* the determinism model from chapter 2 is held
together by the chain of customisation.
