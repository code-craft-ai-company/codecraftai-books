# Chapter 1 — Why Distributed Automation Is Genuinely Hard

> *Single-host testing has one machine.  Distributed testing has
> twelve.  The difference is not "twelve times the work" — it is a
> different kind of work.  Failure modes that don't exist on one
> host appear on twelve, and the framework's job is to make them
> *named, reproducible, and recoverable*.  This chapter is about
> what's different.*

Volume 1 walked the single-host test framework.  The hooks, the
sequencer, the property-based runner, the JSOS stream, the agents
CLI — every piece of that volume runs in one Python process
against one machine.  Volume 1 chapter 1 named the seven gaps
between application unit testing and automation testing; the
framework's answer to those gaps is what the rest of Volume 1
walked.

Volume 2 is what happens when the system under test is no longer
one machine.  When an automation test reaches across machine
boundaries — a test runner on one host, a database on another, a
device under test on a third, a power controller on a fourth —
the framework's job changes shape.  The lifecycle hooks that
were no-ops on single-host become non-trivial.  The resource
factories that yielded local mocks now yield landscape-aware
couplings.  The JSOS records that streamed cleanly out of one
process now arrive from N processes on N machines, with all the
ordering and failure-mode questions that implies.

This chapter is the volume's thesis.  The remaining fifteen
chapters are the implementation.

## The premise: the system under test is no longer in your process

The single sentence that distinguishes distributed automation
from any other kind of testing:

> *The system under test is not in the test process, may not be
> on the same machine, and may not be reachable when the test
> starts.*

Each clause adds a problem.

**"Not in the test process"** is the boundary Volume 1 already
addressed.  A pytest test calls a function in the same Python
interpreter; an automation test calls into a fleet of services
through wire protocols.  Volume 1's couplings, mocks, and
landscape boundary are about staying sane across that
process-to-process boundary on a single host.

**"May not be on the same machine"** is the boundary Volume 2
addresses.  The wire is no longer a function call into another
Python module; it is SSH, REST, gRPC, serial-over-TCP, or
something stranger.  The test process talks to the system under
test through code that crosses the kernel — and that code can
fail in ways the function call cannot: network partitions,
machine crashes, NAT failures, clock skew.

**"May not be reachable when the test starts"** is the boundary
Volume 3 addresses at the orchestration layer and Volume 2
addresses at the activation layer.  A real environment goes
down between when CI provisions it and when the test runs;
verification (chapter 7 of this volume) is the framework's
defence.

## Six failure modes single-host frameworks miss

Six concrete classes of failure that automation engineers
running multi-host suites discover, in roughly the order they
discover them.  The remainder of this volume gives each one a
named handling pattern.

### 1. One host of N down

Twelve hosts in the fleet.  Eleven respond to ping.  One
doesn't.  What does the framework do?

A pytest fixture would fail at acquisition; the entire suite
would fail with a single import error.  An automation framework
must do better, because *eleven out of twelve hosts running* is
a real and useful state.  The chapter on landscape activation
(chapter 6) walks the `allow_missing_devices` flag in
`LandscapeActivationParams`.  The chapter on partial-run
semantics (chapter 14) walks how a partial run reports.

### 2. Partial credential failure

The credentials store has eight entries; six resolve, two fail.
The two that fail are for hosts the suite needs.  Single-host
testing assumes credentials either exist or don't; distributed
testing has to handle "exist for some hosts, not others."

The chapter on credentials at scale (chapter 11) walks the
resolution model: per-host scoping, multi-source fallback, the
audit trail of what got fetched and what didn't.  The
verification contract (chapter 7) treats unresolved credentials
as a hand-off failure that the orchestrator owns, not something
the test should silently work around.

### 3. Race conditions in setup or teardown

Two tests in the same suite each acquire the device pool.  Both
ask for "host A."  In a distributed run this surfaces as a race;
in single-host, the resource factory's lifespan resolution
(Volume 1 chapter 3) prevented it from being possible.

The chapter on landscape activation (chapter 6) walks the
device-checkout-and-checkin discipline in
`LandscapeOperationalLayer`.  The chapter on partial-run
semantics (chapter 14) walks what happens when a teardown
crashes mid-run and leaves resources held.

### 4. Machine clock drift

Host A reports the test started at 10:32:01; host B reports the
fixture finished at 10:32:00.  Did the fixture finish before the
test started?  On a single host the answer is "obviously yes";
across machines, the answer is "your clocks disagree by 1.7
seconds, and you don't know it."

The chapter on result aggregation (chapter 13) walks the JSOS
stream's clock-disambiguation: every record carries a host
fingerprint, and the aggregator preserves per-host timelines
rather than forcing them into a single global timeline.

### 5. Network partitions during tests

Mid-test, host C loses connectivity to the controller.  The
test was waiting for an event from C.  Now it will wait
forever.  Or until a timeout.

The chapter on the remote-execution wire (chapter 10) walks the
Aspects pattern as it applies to distributed wire calls: every
remote call has an `Aspects` parameter (Volume 5 ch.1) carrying
its timeout, retry policy, and connection-failure handling, so
that "the network broke" becomes a *bounded* failure mode rather
than a hang.  The chapter on debugging distributed failures
(chapter 15) walks how to triage the result.

### 6. Log aggregation gaps

The test failed.  The error message says "host F responded
unexpectedly."  The relevant context is in host F's syslog,
host C's audit trail, and host A's framework log.  Three logs
on three machines.  The single-host chapter 5 of Volume 1
introduced JSOS streaming for a reason; chapter 13 of this
volume walks the cross-host extension.

## The landscape becomes non-trivial

Volume 1 chapter 9 introduced the landscape boundary at the
*contract* level: `TestJob`'s five `fire_*` hooks are no-ops on
single-host; `InteropLibraryTestJob` overrides them to drive
landscape activation; `DistributedTestJob` plugs in the concrete
landscape.  That contract is the bridge.

Volume 2 walks the bodies of the hooks.  When
`fire_attach_to_environment` actually attaches a sequencer to a
landscape (chapter 6 of this volume), it walks
`LandscapeConfigurationLayer.load_landscape`,
`LandscapeIntegrationLayer.initialize_landscape`, and
`LandscapeOperationalLayer.activate_coordinators` —
each of which has substantial machinery that didn't appear at
all in Volume 1.

A chapter-3 (this volume) reader will be told about *Pods*: the
runtime instance produced by combining a Landscape (the physical
inventory) with a Topology (the logical configuration).  A
distributed test runs in *one Pod*; the Pod is the unit of
horizontal scaling at the CI layer.  Within a Pod, tests run
sequentially because the Pod is a shared environment.  Across
Pods, tests run in parallel because Pods are independent.

The Pod model is what makes "twelve-host integration suite"
possible without it being "twelve times the flake of a one-host
suite."

## The determinism problem

The hardest of the six failure modes — and the one that gives
this volume its second chapter — is the question of *whether
distributed tests can be deterministic at all*.

If the controller (the test) and one of the hosts under test
each cause effects in the environment, the order in which their
effects are visible to other hosts is not deterministic.  A
test that says "do X, wait for Y to be observable, do Z" can
produce different outcomes on different runs because Y and the
test's own action race.

The framework's answer — covered in detail in chapter 2 — is
**enforced single-actor execution per environment**.  The test
is the only actor; observation is delegated to validators that
run *in tandem* with the test.  Volume 1 chapter 1 introduced
the metaphor: a tandem bicycle has one driver and many
observers.  Volume 2 chapter 5 walks the
`automojo-testbase/.../tandem/validators.py` machinery.

Without this commitment, distributed automation is flaky
forever.  With it, distributed automation is *deterministic
within a single Pod*; flakes that remain are environmental, not
framework, and the verification contract (chapter 7) rules out
most of them before the first test fires.

## The volume's commitments

By the end of Volume 2 a reader should be able to:

- **Read the TestJob inheritance chain** (chapter 4) and predict
  what each layer adds.
- **Build a custom `DistributedTestJob` subclass** for a
  project's specific environment.
- **Add tandem validators** to a test (chapter 5) for parallel
  observation.
- **Walk the three landscape activation phases** (chapter 6)
  and know which failure surfaces in which phase.
- **Verify a hand-off contract** (chapter 7) before a suite
  runs and refuse to proceed when the verification fails.
- **Understand why the `DistributedTestSequencer` does not
  parallelise tests** (chapter 8) and how it differs from the
  base `TestSequencer` in landscape awareness.
- **Use `job_seed`** (chapter 9) to make a property-based
  distributed run reproducible across hosts and runs.
- **Handle the SSH wire** (chapter 10) with the Aspects pattern
  for behaviour parameterisation.
- **Resolve credentials and data profiles** (chapters 11 and
  12) in cross-machine contexts.
- **Aggregate JSOS streams** (chapter 13) from N workers to one
  coordinator.
- **Scope partial-run outcomes** (chapter 14) honestly — saying
  what passed, what failed, and what couldn't run.
- **Debug a distributed failure** (chapter 15) using the
  framework's audit-diary tools and the six-pass landscaping
  audit pattern as a reference.
- **Read a real worked example** (chapter 16) of a 5-host
  integration test.

## Where this volume sits in the series

A reader's reading order:

- **Volume 0** establishes the foundation (errors, waiting,
  application identity, configurations, philosophy).  Volume 2
  assumes the foundation tier is fluent.
- **Volume 1** establishes the single-host substrate.  Volume 2
  *extends* Volume 1's contracts; the contracts themselves were
  set up at the seam in Volume 1 chapter 9.
- **Volume 3** invokes Volume 2 at the orchestration layer.
  Volume 3 chapter 21 defines the verification contract that
  Volume 2 chapter 7 walks in detail; Volume 3 chapter 25 walks
  the bridge tasks that hand off to a `DistributedTestJob`.
- **Volume 5** covers the wire (SSH, networking, REST) in
  depth.  Volume 2 chapter 10 sketches the wire as it appears
  to distributed tests; Volume 5 walks the wire's own machinery.
- **Volume 6** covers landscapes, topologies, and provisioning
  schemas in depth.  Volume 2 treats them as inputs the framework
  has already loaded; Volume 6 walks what's *in* the files.
- **Volume 8** covers time-travel debugging.  Volume 2 chapter
  15 forwards to it for the production-grade post-mortem flow.

## A note on the meta-thesis

Per Volume 0 chapter 1's meta-thesis, the framework supplies the
*skeleton and patterns*; the organisation customises and refines
to meet the needs of the automation scenarios.  Volume 2 is
where the meta-thesis lands hardest.  The `TestJob` →
`InteropLibraryTestJob` → `DistributedTestJob` chain is the
*entry point* for customisation; the abstract `_get_landscape()`
template method is the *plug-in seam*; the user's `Landscape` and
`Topology` subclasses are the project-specific bodies.

When Volume 2 says "the user implements X" — and it will say so
many times — the framing is always Volume 0 ch.1's: that's
exactly where the framework expects the project to specialise.
The skeleton is opinionated; the bodies are the organisation's
to write.

The next chapter walks the load-bearing design choice that makes
distributed automation possible at all: single-actor determinism
and the tandem-bicycle model of observation.
