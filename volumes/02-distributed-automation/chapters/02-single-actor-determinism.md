# Chapter 2 — Single Actor, Multiple Observers — The Determinism Model

> *Distributed tests are deterministic when one actor causes
> effects in the environment at a time and observation is
> delegated to validators that ride along with the actor.  This
> chapter is the design rationale; the next chapter is what it
> means for parallelism; chapter 5 walks the framework's
> implementation.*

The most important load-bearing design choice in distributed
automation testing is *enforced single-actor-per-environment*.
This is not a feature of the implementation in the sense of "a
class enforces it" or "a lock prevents violations."  It is a
*design property* the framework commits to — and the framework's
shape (the `TestJob` lifecycle, the sequencer's serial
execution, the tandem-validator mechanism) is what makes the
property hold.

This chapter is the rationale.  Chapter 5 is the implementation.
Both are short; the rationale is short because the argument is
sharp, and the implementation is short because the framework's
correctness rests on a small set of code.

## The argument

When a controller (the test) interacts with a machine in a
distributed environment, the effect of the interaction takes
time to propagate to the other machines.  The test must wait —
usually by polling or by subscribing to an event — until the
effect is observable everywhere it needs to be observable.  If
a *second actor* can cause effects in the same environment
during that wait window, the order in which effects appear is
not controlled by the test.  Two effects that arrive in
different orders on different runs produce different states; the
test is no longer deterministic.

The conclusion: **in any given environment, one actor at a
time.**

Stated differently: distributed automation testing has a
property that single-host application testing does not have to
worry about — the *environment under test is a shared resource
across actors*, and the framework's only defence against
non-determinism is to ensure exactly one actor is acting in
the environment at any given moment.

The single-host case avoids this entirely because the
"environment" is the test process; nothing else can act on it.
Application testing avoids it because the system under test is
inside the test process, where the test is the only controller.
Automation testing has to enforce it explicitly.

## What "actor" means

An *actor* is anything that *causes* effects in the
environment.  Examples:

- The test code itself (calls into the system under test).
- A scheduler or cron job running on one of the hosts.
- A second test running concurrently against the same hosts.
- A human running commands by hand.
- An external monitoring system that reaches in and modifies
  state to test recovery.

The framework's commitment is that *the test* is the actor.
Everything else either:

- Is suspended for the duration of the test (cron, schedulers).
- Operates in a different environment (a second test running
  in a different Pod — chapter 3).
- Is forbidden by operational discipline (no humans in the lab,
  no concurrent tests).

In practice, the third item is the hardest.  Operational
discipline is not a framework property; it is a *team* property.
But the framework's design must be compatible with the team
discipline — and it is, because the framework provides exactly
one mechanism for getting work done in the environment (the
test) and exactly one mechanism for *watching* work happen
(tandem validators).

## What "observer" means

An *observer* is anything that *watches* effects in the
environment without causing them.  Examples:

- A validator polling for a state change.
- A log tail streaming syslog from a host.
- A metric collector reading SNMP / Prometheus.
- A snapshot of a host's running processes taken every five
  seconds.

Observers do not violate the single-actor rule because they do
not cause effects.  Many observers can run concurrently with
each other and with the actor, because none of them can change
the environment's state.  The framework's tandem-validator
mechanism is built specifically around this distinction.

A point of pedantry worth landing: an observer that *could*
change state (e.g., a "monitor" that helpfully restarts a
failing service) is *not an observer*.  It's a second actor.
The framework's Validator base class
(`automojo-testbase/source/packages/automojo/testbase/tandem/validators.py:33`)
makes this distinction structural — validators inherit from
`ValidatorCoupling`, not from any class that has the framework's
write surface.  A validator that needed to act would need to be
a different class; the discipline is in the type system as much
as the design.

## The tandem-bicycle metaphor

A tandem bicycle has one driver and one or more observers
(passengers).  The driver pedals and steers; the observers ride
along watching the world go by.  The bike's direction and speed
are determined entirely by the driver; the observers contribute
their attention but not their muscle.

The metaphor maps onto the framework cleanly:

- The **test** is the driver.  It pedals (causes effects) and
  steers (decides what to do next).
- The **validators** attached to the test scope are the
  observers.  They ride along, watch for the effects the driver
  is causing, and produce their own report when the ride is
  over.
- A test with three validators is *one ride* with *four
  recorded outcomes* — the test plus three observers, each with
  its own perspective and its own report.

The crucial property the metaphor captures: the observers don't
steer.  An observer that started causing effects would be
trying to ride two bikes at once — the framework refuses to
allow it.

## What this rules out

A short list of patterns the determinism model rules out, with
the right answer for each:

### Concurrent tests in the same Pod

```python
# Wrong — two tests racing on shared state.
import asyncio

async def main():
    await asyncio.gather(
        run_test("test_a"),
        run_test("test_b"),
    )
```

If `test_a` and `test_b` run against the same Pod, they can
produce non-deterministic outcomes by racing.  The framework's
`DistributedTestSequencer`
(`automojo-testplus/source/packages/automojo/testplus/distributedtestsequencer.py:33`)
preserves serial test execution for exactly this reason.  Tests
within a Pod are sequential, period.

The right answer: if you want concurrent test execution, run
them in *different Pods*.  Chapter 3 walks Pod-level parallelism
in detail.

### A "monitor" that fixes things

```python
# Wrong — the monitor is a second actor.
def health_monitor():
    while True:
        if service_is_down():
            restart_service()  # second actor!
        time.sleep(5)
```

If a monitor reaches in and restarts a service while a test is
verifying recovery behaviour, the test's outcome depends on
whether the monitor or the test acted first.  Non-deterministic.

The right answer: a monitor that watches and reports without
acting is an *observer*; a monitor that acts is a second actor
and must run in a different Pod or be suspended for the
duration of the test.

### A scheduler that fires during the test

```cron
# Wrong — backup runs during the test window.
0 * * * * /usr/local/bin/backup
```

If a scheduled backup runs against a host the test is using,
the backup's effect on the host is concurrent with the test's
effect.  Non-deterministic.

The right answer: schedulers are a *deployment* concern, not a
framework concern.  The team running the lab disables
schedulers during test runs.  The framework's role is to make
the *test's actions* the only framework-driven actor; the team
ensures no other framework is acting on the same Pod
concurrently.

## What this enables

The single-actor commitment is what makes everything else in
the framework possible:

- **Reproducibility.**  The same test, the same job_seed, the
  same Pod state → the same outcome.  Chapter 9 walks the
  reproducibility machinery.
- **Tandem testing.**  Many validators can attach to one test
  *because* they don't compete with each other for actor
  privileges.  Chapter 5 walks the implementation.
- **Pod-level parallelism.**  Different Pods can run different
  tests concurrently *because* each Pod has exactly one actor
  (its own test).  Chapter 3 walks the Pod model.
- **Honest partial-run semantics.**  When a test fails, the
  framework can say *what* failed, *which validator caught it*,
  and *what hosts were affected* — because the actor and the
  observers are explicitly distinguished.  Chapter 14 walks
  partial-run reporting.

Without single-actor determinism, none of these work cleanly.
Reproducibility becomes "reproducible most of the time."
Tandem testing becomes "parallel testing with a different
name."  Pod parallelism becomes "concurrent tests with a
different name."  Partial-run semantics become "we don't
really know what happened."

The single-actor commitment buys all four properties at once,
at the cost of refusing pytest-xdist-style test parallelism
within a Pod.  The trade-off, the framework makes loudly: more
horizontal scale comes from more Pods, not more tests-within-
a-Pod.  Chapter 3 walks the trade-off.

## The framework's commitments to the model

Three structural commitments hold the model together:

### 1. Sequential test execution within a Pod

`DistributedTestSequencer`
(`automojo-testplus/source/packages/automojo/testplus/distributedtestsequencer.py:33`)
extends `TestSequencer` from `automojo-testbase` but does *not*
override the test-execution loop.  The base sequencer's
`execute_tests()` method (`testsequencer.py:412`) walks tests in
declaration order, one at a time.  The distributed sequencer
adds landscape-aware coupling attachment (chapter 8 of this
volume walks the differences); it does not add parallelism.
This is by design — the framework refuses to expose the API
that would let an SDET parallelise tests within a Pod, because
exposing that API would invite the determinism violation the
model rules out.

### 2. The actor / observer type distinction

The `Validator` class
(`automojo-testbase/source/packages/automojo/testbase/tandem/validators.py:33`)
inherits from `ValidatorCoupling` (in
`automojo-testbase/.../coupling/validatorcoupling.py:44`), which
inherits from `BaseCoupling`.  Validators have access to the
test scope and the recorder — they can *observe and report* —
but they do not have access to the test's *write surface*
(the methods that act on the system under test).  The type
system encodes the distinction.

A reader writing a custom validator is structurally prevented
from reaching the actor's write surface from inside an
observer.  The discipline is enforced not by code review but by
the class hierarchy.

### 3. The single attach point per test

A test is one function decorated with `@testbase.testcase` (or
the relevant decorator).  There is no "concurrent test" decorator,
no "parallel test group" construct, no "actor pool" mechanism.
A test is a function; a function runs once per invocation; the
framework's run loop processes one test at a time.

The simplicity is the safety.  When the framework only offers
one shape, the shape is hard to use wrongly.

## What the next chapter does

Chapter 3 walks the corollary: if tests within a Pod must run
sequentially, where does parallelism live?  The answer is the
*Pod model*.  Pods are independent environments; jobs in
different Pods run concurrently because they cannot affect each
other; horizontal CI scaling means *more Pods* (provisioned
through more Specifications), not more parallel tests within
one Pod.

The Pod model is the corollary that makes "more horizontal
scale" possible without violating the single-actor commitment
this chapter just established.
