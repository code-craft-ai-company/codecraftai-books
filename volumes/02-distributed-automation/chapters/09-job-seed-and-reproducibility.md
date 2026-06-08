# Chapter 9 — `job_seed` and Reproducibility

> *A distributed run that produces different inputs on
> different hosts is not reproducible.  A distributed run that
> produces the same inputs across hosts and across runs is.
> The single seed that travels with the job is what makes the
> second case possible.*

Volume 1 chapter 4 walked property-based testing on a single
host: the per-test seed is derived from the job seed and the
test name via SHA-256, the runner caps `max_examples` per
runtime configuration, the seed survives across runs because
re-running with the same job seed reproduces every per-test
seed.

This chapter walks the same machinery in a distributed
context.  When the test runs across N hosts — with different
hosts generating different inputs, observing different
states, returning different results — the question "what does
*reproducibility* even mean?" becomes load-bearing.

The framework's answer: **reproducibility is per-Pod, not
per-host.**  The job seed travels with the job; every host
sees inputs derived from the same seed; across runs of the
same suite with the same seed against the same Pod, the
inputs are identical.

## The job seed across machines — what travels and what doesn't

The `job_seed` is a single integer, set once at the start of
the test job, and carried by the `DistributedTestJob` through
its construction:

```python
# DistributedTestJob from chapter 4

def __init__(self, ..., job_seed=None, ...):
    super().__init__(..., job_seed=job_seed)
    # ...
    return
```

The seed is stored on the job instance and propagated to the
sequencer:

```python
# From DistributedTestJob._create_sequencer()

inst = DistributedTestSequencer(
    self.title,
    self._testroot,
    includes=self.includes,
    excludes=self.excludes,
    metafilters=self.metafilters,
    job_seed=self.job_seed,    # ← here
)
```

The sequencer carries the seed to every property-based test
runner (Volume 1 chapter 4) it spawns.  Each runner derives
its per-test seed via
`SHA-256(job_seed + b":" + test_name)` and feeds it into the
strategy generators.

What travels:

- The `job_seed` integer itself.
- Implicitly, the *test name* — derived from the test's
  module path and function name.
- The test's monikers and pivots (which influence the
  result's identity but not the input generation).

What does *not* travel:

- The *host identity*.  A test running on Host A and the same
  test running on Host B (different runs of the same suite
  against different Pods) would derive *the same per-test
  seed* — because the seed derivation does not include host
  information.
- Random state from the framework's own RNG (no such state
  exists; the framework derives all randomness from the job
  seed).
- Wall-clock time.  Tests that depend on `time.time()` for
  randomness are non-reproducible by construction; the
  framework cannot help.

## How the per-test seed reaches the test on a single host

Volume 1 chapter 4 walked this:

```python
# codecraftai-testbase/source/packages/ccai/testbase/properties/propertyrunner.py:82

def derive_test_seed(job_seed: bytes, test_name: str) -> int:
    payload = job_seed + b":" + test_name.encode("utf-8")
    digest = hashlib.sha256(payload).digest()
    rtnval = int.from_bytes(digest[:8], byteorder="big")
    return rtnval
```

The function takes the job seed (as bytes) and the test name
and produces a 64-bit per-test seed.  For a given
`(job_seed, test_name)` pair, the output is deterministic.

## How the per-test seed reaches the test in a distributed context

In a distributed run, the test code runs on the *controller*
— the host running the `DistributedTestJob`.  When the test
issues calls into the Pod (via SSH, REST, or another wire), it
sends data the controller generated locally.  The hosts under
test do not generate test inputs themselves; they respond to
the controller's actions.

This means:

- **The per-test seed is computed once** on the controller
  using `derive_test_seed(job_seed, test_name)`.
- **The strategy generators run on the controller**, producing
  inputs deterministically from the seed.
- **The inputs are sent to the hosts** as command arguments,
  REST payloads, fixture data, etc.

The hosts under test never see the seed.  They see the
*generated inputs*.  The hosts have no role in randomness;
they are deterministic responders.

This is consistent with chapter 2's single-actor model.  The
controller is the single actor.  Random number generation is
an *actor* operation (the actor decides what to do next).
The hosts are the system under test, not actors; they don't
generate random numbers for the test.

## Cross-run reproducibility

Re-running the suite with the same job_seed against the same
Pod reproduces every input the controller generates, because:

- The job_seed is the same.
- The test names are the same (same suite, same discovery
  order).
- The per-test seeds are deterministic functions of those.
- The strategy generators are deterministic functions of the
  per-test seed.

If the *Pod* is also in the same state, the responses from the
hosts will be the same, and the test outcomes will be the
same.  The ", same Pod state" caveat is what verification
(chapter 7) provides — verification confirms the Pod state
matches the Topology's claim, so re-running against a freshly-
verified Pod is re-running against the same starting state.

## What happens when a property-based test fails distributedly

A property-based test in a distributed run might fail because:

- A specific input combination triggers a real bug in the
  system under test.
- A specific input combination triggers a flake (e.g., a
  timing race in the Pod that doesn't occur with other
  inputs).
- A network partition mid-test produced an unexpected error
  that's not the property the test was checking.

The framework's reporting (Volume 1 chapter 4 walked the
single-host shape) carries:

- The input that failed.
- The seed used.
- The number of shrinks attempted.

In a distributed run, this is enriched with:

- The host(s) involved in the failure (extracted from the
  call paths or the JSOS records — chapter 13 walks the
  aggregation).
- The Pod's state at failure time (if the framework's
  diagnostic capture caught it).

A re-run with the same job_seed reproduces the same input.
Whether the failure reproduces depends on the failure mode —
real bugs reproduce, flakes are flakes regardless of seed.
The tool the framework gives the SDET is the *separation*: a
property-based failure with a recorded seed is reproducible
with one command; if the failure doesn't reproduce, it's a
flake worth investigating, not a phantom.

## Reproducibility across re-provisioned Pods

A subtlety: re-running against a *re-provisioned* Pod (CI
provisions a fresh Pod for each run) is *not* the same as
re-running against the same Pod.

- **Same Pod, same seed.**  Inputs the same, responses the
  same, outcomes the same.  Full reproducibility.
- **New Pod, same seed.**  Inputs the same.  Responses
  potentially different (different physical hardware, different
  IP addresses, different MAC addresses).  Outcomes mostly
  the same for tests that don't depend on hardware-specific
  values; potentially different for tests that do.

This is *not a framework limitation*.  It's a property of the
distributed system being tested.  A test that depends on
hardware specifics has hidden non-reproducibility; a test that
doesn't is reproducible across Pods provided the Pods
*verify* (chapter 7) against the same Topology.

The test author's discipline: when a test fails in a CI run,
re-run with the same seed against a *Pod with the same
Topology*.  If it reproduces, it's a real bug.  If it
doesn't, the test depends on Pod-specific state (a hardware
fingerprint, an IP address baked into a test) and the test
itself needs to be made Pod-agnostic.

## When to set the seed manually

The default behaviour: each run uses a fresh job_seed.  This
is correct for normal CI — every PR gets fresh inputs, the
test space is broadly covered across runs, regressions surface
on subsequent runs even when individual runs cap
`max_examples` low.

When to override:

### Replaying a recorded failure

A property-based test failed in CI run #4823 with seed
`0xDEADBEEF12345678` at test `test_replication_invariant` with
input `(nodes=3, partition_at=2)`.  An SDET wants to reproduce
the failure locally:

```bash
testbase run --testroot tests/ \
             --include test_replication_invariant \
             --job-seed 0xDEADBEEF12345678
```

The same seed against a Pod with the same Topology reproduces
the same input.  If the failure reproduces, the SDET can
debug.  If it doesn't, the failure was Pod-state-dependent
and the SDET needs the Pod's state too (Volume 8's
time-travel debugging is the production-grade tool for this).

### Comparing two implementations

A team compares "old code path" and "new code path" by
running the suite against each with the same seed.  Inputs
are identical; differences in outcomes isolate to the code
change.  This is the classic A/B-test pattern at the property
level.

### Investigating a flake

A test passes 95% of the time.  An SDET runs it with seed
`X`, captures the inputs it generated, runs it again with seed
`X`.  The same inputs reproduce.  If the *outcome* still
varies, the test is non-deterministic for reasons other than
input generation — a real flake worth investigating, not a
seed problem.

## What seed metadata the JSOS records carry

Per Volume 1 chapter 5, JSOS records can carry rich metadata.
For property-based tests, every iteration's record includes:

- The iteration number (1 through `max_examples`).
- The strategy values for each parameter.
- The per-test seed (so the iteration can be replayed with the
  same input).

A consumer reading the JSOS stream can see exactly which
iteration of which test failed and what input triggered it.
The agents CLI (Volume 1 chapter 6) `agents testresult
failures` subcommand surfaces this for AI-agent or human
review.

In a distributed run, the JSOS records also carry host
identity for each call the test made — chapter 13 walks the
aggregation.  A failure on iteration 7 of
`test_replication_invariant` with input `(nodes=3, partition=2)`
that surfaced as "Host C did not respond" is a complete
forensic record: same seed, same input, identifiable host,
specific failure.

## Volume 0's foundation in play

Two pieces of Volume 0's foundation tier hold up the
reproducibility story:

- **Configuration loading** (Volume 0 chapter 9) is what
  determines the per-Pod seed-relevant inputs.  The
  Specifications, Landscapes, Topologies, Runtimes are all
  loaded deterministically; same files → same loaded
  structures → same seed-relevant inputs.
- **Runtime configuration** (Volume 0 chapter 9; Volume 3
  chapter 7) controls the runtime parameters that influence
  behaviour.  A test that depends on a timeout uses the
  timeout from the runtime; same runtime → same timeout →
  reproducible test.

Without these, even a fixed seed wouldn't reproduce — because
the *configuration* the test ran against would vary.  The
configuration discipline is part of the reproducibility
discipline.

## What the next chapter does

Chapter 10 walks the remote-execution wire — the SSH-based
mechanism that carries the controller's actions to the hosts
under test.  The Aspects pattern from Volume 5 chapter 1
shows up here as the behaviour-parameterisation knob for
timeout, retry, connection failure handling on every wire
call.  The chapter also surveys the `SshAgent` and
`SshSession` machinery from `codecraftai-protocol-ssh`.
