# Chapter 1 — Why `unittest` and `pytest` Fall Short for Automation

> *Application unit tests verify a function in isolation.  Integration
> tests verify two collaborators in isolation.  Automation tests
> verify a system in motion — devices waking, credentials being
> fetched, configurations being merged, jobs being scheduled across
> hosts.  The tools that ship with Python were never designed for
> the third thing.  This chapter is about what's missing, and why
> patching `pytest` to fill the gap eventually fights the framework
> rather than helping it.*

`unittest` and `pytest` are excellent tools.  Across millions of
Python codebases they verify the correctness of functions, classes,
modules, and small graphs of objects.  The `automojo-*` portfolio
uses both — `pytest` runs on the framework's *own* unit tests,
`unittest` is the foundation of `unittest.mock`'s spec-aware
mocking, and the discipline of writing testable functions started
in those communities long before automation engineering picked it
up.

This volume is not a polemic against pytest.  It is an explanation
of why a particular kind of test — the kind that runs against a
*system* rather than a *function* — needs a different framework.
That framework is `automojo-testbase`, and the rest of Volume 1
walks through it.  This chapter establishes *why testbase exists*
by listing the constraints automation testing places on a runner,
then asking which of those constraints `pytest` was designed to
satisfy.

The answer turns out to be: not many.  That's not a criticism of
pytest.  It's a statement about scope.

## The shape of an automation test

An automation test, in this volume's sense, is a test where:

1. The system under test is **not in the test process**.  The
   system under test is a device, a fleet of devices, a service
   running on another machine, or a constellation of services
   coordinated by an orchestration layer.
2. The test must **acquire and release real resources** — SSH
   sessions, credentials, license tokens, devices that other
   tests in the suite may also need, network namespaces.
3. The test must **survive failures it can't reproduce locally**.
   A flaky network on host 7 of 12 is part of the production
   environment; the framework has to tell you what happened on
   host 7 with enough detail that you don't need to reproduce.
4. The test must **stream results** while it runs, not write
   them all at the end.  A four-hour suite that crashes in hour
   three should still tell you what passed in hours one and two.
5. The test must **be machine-readable** — by CI dashboards,
   replay tools, AI agents extracting failure patterns.  Stdout
   and a green-or-red exit code don't carry enough information.
6. The test must **describe its environment** — which devices it
   needs, which credentials, which configuration profile — so that
   a runner on another machine can reproduce or refuse the run.

The first three of these are what most engineers mean when they
say "automation testing is hard."  The last three are what most
engineers don't notice until they try to scale a test suite past
about five hundred tests across more than one host.  Automation
test frameworks live or die on the last three.

## What `pytest` does well

It is worth being precise about pytest's strengths before
explaining why testbase exists, because dismissing pytest is a
mistake too — many of testbase's design choices are pytest's
choices made differently for a different problem.

Pytest does the following extremely well:

- **Test discovery is implicit.**  Functions whose names start
  with `test_` in files whose names match `test_*.py` are tests.
  No registration ceremony.
- **Fixtures compose cleanly.**  A fixture can depend on other
  fixtures, and pytest works out the dependency order.
- **Parametrisation is first-class.**  `@pytest.mark.parametrize`
  generates test variants by inspection of the test function's
  parameters.
- **Plugins are first-class.**  Reporting, parallelism,
  randomisation, coverage, hypothesis, xdist — the plugin
  ecosystem is the framework's strongest feature.
- **Failure reports are diagnostic.**  Pytest's assertion
  rewriting gives you the values of the operands of a failed
  assertion without having to write `assertEqual(a, b, "...")`.

`automojo-testbase` keeps the first three properties.  Tests are
plain functions whose names start with `test_`, found by
inspecting modules under a test root (see Chapter 2).  Fixtures
compose; they are just expressed differently.  Parametrisation
exists in two forms — explicit data-driven (akin to pytest's
`parametrize`) and property-based (akin to Hypothesis).

What testbase *doesn't* keep is the assumption that the framework
runs in the same process as the system under test.

## What automation tests need that pytest doesn't ship

There are seven gaps.  The remainder of Volume 1 is, in effect, an
extended explanation of how testbase fills them.

### 1. A lifecycle wider than the test function

A pytest test has fixtures with `function`, `class`, `module`,
`session` scope.  That's enough when the system under test lives
inside the Python process.  When the system under test is *outside*
the Python process, you need lifecycle hooks for things that don't
map onto fixtures:

- **Attaching to the framework**: registering integration couplings,
  binding the test runner to a results pipeline.
- **Attaching to the environment**: connecting to devices, opening
  SSH sessions, fetching credentials.
- **Activating configuration**: telling the landscape that this
  particular run wants this particular profile.
- **Activating integration**: bringing online the cross-host
  collaboration the suite depends on.
- **Activating operations**: starting the side processes (log
  shippers, metric agents) that other tests will rely on.

In `automojo-testbase`, these are five `fire_*` hooks on `TestJob`:
`fire_attach_to_framework`, `fire_attach_to_environment`,
`fire_activate_configuration`, `fire_activate_integration`,
`fire_activate_operations` (file:
`automojo-testbase/source/packages/automojo/testbase/testjob.py:415–443`).
On a single-host run the last three are no-op stubs — they exist
because Volume 2's `InteropLibraryTestJob` (in `automojo-interop`,
`source/packages/automojo/interop/testing/testjob.py:22`) overrides
them to drive landscape activation.  The hooks are *the* place
where testbase's single-host runner and Volume 2's distributed
runner meet.

A pytest fixture cannot model "I need the entire test session to
have woken up the lab's twelve devices and pre-fetched their
credentials before the first test starts, and put them to sleep
after the last test ends" without contortions.  A `fire_*` hook
on a job-spanning singleton can.

### 2. Tests that declare what they need, not what they import

A pytest test imports its fixtures.  An automation test cannot
always do that, because the resource it needs may not be present
on the host the test author is editing on.  In testbase the
resource declaration is a separate decoration:

```python
@testbase.attach.parameter(some_device, identifier="dev")
def test_device_responds(dev):
    """
        Verify that the lab device responds to a probe.
    """
    response = dev.probe()
    rtnval = response.is_ok()
    assert rtnval
    return
```

The `@testbase.attach.parameter(source, identifier=...)` form
(`automojo-testbase/source/packages/automojo/testbase/declaration/attachments.py`)
uses an originator to produce the parameter, with constraints
on lifespan and assigned scope.  The originator can come from a
*different package* than the test, registered via the namespace
package extension model from Volume 0 chapter 7.  A test author
on a laptop without the device declares the dependency without
having to import a module that would fail at import time.

### 3. Results that stream, not aggregate

Pytest writes its summary at the end.  Plugins exist to write
JUnit XML or other formats, but the framework's mental model is
"run all the tests, then report."  An automation suite that takes
hours to run cannot afford to lose its history when the runner
crashes.

Testbase produces three result artefacts during a run:

- `testsummary.json` — the high-level summary of pass/fail counts.
- `testresults_stream.jsos` — a stream file written *as tests
  finish*, one record per event, prefixed by an ASCII record
  separator character (`automojo-testbase/source/packages/automojo/testbase/results/recorders/jsonresultrecorder.py:90–183`).
- `testresults.json` — the consolidated final document, written
  at the end.

The stream file is the one that matters for hours-long runs.  A
CI dashboard, a local tail process, or an AI agent reading
`failures` (Chapter 6) can begin extracting failures *while the
suite is running*.  A crashed run still leaves a partial stream
that tells you what passed and what hadn't started.  Chapter 5
walks the JSOS format in detail.

### 4. Property-based testing with a CI fast path

Hypothesis is excellent at finding edge cases and shrinking them.
But Hypothesis defaults — generating examples until either the
test passes a budget or finds a failure — are calibrated for
running locally with attention.  A 3,700-test suite that triggers
Hypothesis's default `max_examples=100` on every property test
will run for hours without finding anything new.

Testbase ships an environment-variable cap:
`TESTBASE_MAX_EXAMPLES_CAP`
(`automojo-testbase/source/packages/automojo/testbase/properties/propertyrunner.py:47–79`)
caps the per-test `max_examples` regardless of what the test
declared.  CI runs at `TESTBASE_MAX_EXAMPLES_CAP=10` to keep the
fast path under an hour; full-sweep runs leave the cap unset.
The cap is a runtime knob, not a test-author knob — exactly the
right shape for a build-engineer tuning the suite without editing
hundreds of test files.

Chapter 4 is the full treatment.

### 5. A machine-readable surface

Modern CI dashboards extract test summaries by parsing stdout.
That works until the dashboard wants to do something more
ambitious — diff the failing tests against last week's, group
failures by which device they touched, drive an AI agent that
decides whether a flake is real.  Stdout doesn't survive that.

Testbase ships an `agents` CLI subcommand
(`automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/__init__.py:18–28`)
with three groups: `summary`, `testresult`, `config`.  Each
returns structured data — extracted from the JSOS stream by a
parser shared across the subcommands
(`automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/jsosparser.py`)
— that an agent or dashboard can consume directly.  The
agent-readable surface is *the same* surface a human reads;
Chapter 6 walks the contract.

### 6. A landscape boundary

The most subtle gap.  An automation test runs in an environment
— a "landscape" in the portfolio's vocabulary.  The landscape
includes devices, services, credentials, network reachability.
The test framework needs to be aware that the landscape *exists*,
without dragging the entire landscape implementation into the
unit-test process.

Testbase solves this with a *contract*: `TestJob` exposes the
five `fire_*` hooks as no-op stubs.  `InteropLibraryTestJob`
(Volume 2) overrides them to drive landscape activation.  A
single-host test using `TestJob` directly never needs the
landscape; a distributed test using `DistributedTestJob` (also
Volume 2) does.  The contract sits exactly at the boundary, so
that single-host test authors don't pay landscape's
import-time cost and distributed test authors get landscape
hooks where the lifecycle actually fires.

Chapter 9 carries the single-host half of this contract; Volume 2
carries the distributed half.

### 7. Single actor, multiple observers — tandem testing, not parallel

The least-obvious gap, and the one that most cleanly distinguishes
automation testing from application testing.  In a distributed
environment where the machines under test interact with each other,
a test is only deterministic when *one actor* causes effects in the
environment at a time.  When the controller (the test) interacts
with a machine, the test must wait for the effect to propagate to
all the other machines in the environment before checking that
the effect took hold.  Often there are multiple checks — a poll
loop watching for a state, or an event subscription waiting for
a state change.  If more than one actor is allowed to cause
effects concurrently, the ordering of events stops being
deterministic, and the test stops being deterministic with it.

This is *not* a parallelism problem the framework can solve by
adding locks.  It is a determinism property of the system under
test.  Application testing avoids it by running the system under
test inside the test process, where the test is the only
controller; automation testing has to enforce it explicitly,
because the system under test is *outside* the process and
nothing prevents another actor from reaching it.

Pytest's plugin ecosystem includes `pytest-xdist` for parallel
test execution.  That plugin is exactly the wrong tool for an
automation suite where the tests share an environment.  Two
parallel tests against the same fleet *will* race on shared
state and the suite *will* turn flaky.

Testbase enforces single-actor execution by treating the test
itself as the actor and providing a separate mechanism for
*observers*:

- The **test** is the single actor.
- **Observers** are `Validator` instances (subclassing
  `ValidatorCoupling`) that the framework attaches to the test
  scope via `attach_to_test(testscope, suffix)`.  They watch
  for effects, run *in tandem* with the test, and produce their
  own per-observer outcomes recorded alongside the test's result.

This is **tandem testing**: one actor + N attached observers.
The mental model is a *tandem bicycle*: one driver pedals and
steers; one or more observers ride along watching the world go
by.  The driver is the only one steering — that's the
single-actor commitment — but the observers are full
participants in the ride, with their own perspective and their
own report at the end.

File: `automojo-testbase/source/packages/automojo/testbase/tandem/validators.py`.

Where parallelism *does* live: at the **job level**, across
independent **Pods**.  A Pod is a set of devices configured to
interoperate with each other (Volume 6 covers Pod formation in
detail).  Within one Pod, jobs run sequentially because a Pod
is a shared environment.  Across Pods, jobs run in parallel
because Pods are independent — effects in one Pod cannot
propagate to another.  A CI orchestration that wants more
horizontal scale provisions more Pods, not more parallel tests
within a Pod.  Volume 3 covers the orchestration side; Volume
2 covers the distributed test side; the two volumes meet at
the Pod boundary.

## What this volume builds

Volume 1's ten chapters walk testbase from the outside in:

- Chapter 2 covers the architecture — the pieces that make a run
  go.
- Chapter 3 covers resource factories — how tests acquire and
  release the things they need.
- Chapter 4 covers `@given` and the property-based testing model
  with the CI cap.
- Chapter 5 covers JSOS streaming results.
- Chapter 6 covers the `agents` machine-readable CLI.
- Chapter 7 covers testbase's own mocking primitives.
- Chapter 8 covers `arcli`, the declarative CLI toolkit underneath
  every `automojo-*` CLI.
- Chapter 9 carries the single-host half of the landscape
  contract — the public API Volume 2 will extend.
- Chapter 10 is a case study: the 3,700-test single-host suite
  in `automojo-agent-kit`, with `TESTBASE_MAX_EXAMPLES_CAP=10`,
  finishing in a CI window short enough that engineers run it
  voluntarily.

By the end, a senior SDET should be able to read a testbase test
file and know exactly which framework concept each line is using.
That fluency is the prerequisite for Volume 2 (the distributed
extension) and Volume 3 (orchestration-level invocation) to make
sense.

## A note on continuity

Every chapter in Volume 1 assumes the reader has read Volume 0
— or knows the foundation tier well enough to skip it.  When a
chapter says "the explicit-`return` discipline" or "the
`SemanticError` rule" or "the Aspects pattern," it is referring
to material from Volume 0 chapters 1–4.  Volume 0 anchored those
ideas; Volume 1 uses them.

When a chapter forward-references the distributed extension, it
will say so explicitly: "Volume 2 chapter N covers the
distributed-half of this contract."  The single-host text never
relies on the distributed text — but it carefully marks the
seams, because Volume 2 is the volume that walks across them.

## What we have not written

This volume *is not* a tutorial on Python testing.  It assumes
you can write a passing pytest test, you understand
parametrisation, you have used a fixture.  The volume's job is to
show you how an automation-grade test framework is different —
and why those differences are *necessary*, not preferences.

This volume *is not* a comparison shop.  We do not survey
`Robot Framework`, `behave`, `pytest-bdd`, or any other Python
testing framework.  Those tools have their reasons; testbase has
ours.  The volume's job is to show why testbase is the shape it
is, so that a reader who needs to *extend* it knows where the
seams are.

This volume *does not* cover distributed test execution.  That
is Volume 2.  Where the contract that distributed extends sits
inside testbase, this volume marks the contract.  Where the
*implementation* lives, Volume 2 picks up.

## What we have committed to

By the end of Volume 1 you will:

- Know the five `fire_*` lifecycle hooks of `TestJob` and what
  each does on a single-host run.
- Know the resource-factory model and how originators register
  with the injection registry from Volume 0 chapter 7.
- Know the JSOS streaming format and how the `agents` CLI
  consumes it.
- Know `arcli`'s `Command` / `ArgField` / `OptionField` model
  and the validator-and-parser hardening pattern from PR #13.
- Be able to read a testbase test file and identify, line by
  line, which framework concept each construct represents.

That fluency is what Volume 2 builds on.  And Volume 3.  And
parts of Volumes 6, 7, and 8.  This is the substrate.

The next chapter walks the architecture itself.
