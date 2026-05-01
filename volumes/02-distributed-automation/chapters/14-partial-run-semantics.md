# Chapter 14 — Partial-Run Semantics

> *A distributed run that completed every test against every
> host with every validator passing is the easy case.  The
> hard case is the run where one of five hosts crashed
> mid-suite, where verification caught a Topology mismatch on
> one cluster but not another, where the framework activated
> only some of the declared services.  The framework's
> commitment is honesty about scope: report what ran, what
> didn't, and why, without overstating success.*

Single-host test frameworks have a clean answer to "did the
suite pass?" — every test ran; some passed, some failed; count
the failures.  Distributed runs make the question more
complicated.  A test that *didn't run because its host was
unreachable* is not a passing test, but it's also not a failing
test in the same sense as a test whose assertions failed.  A
test whose validator caught a steady-state violation while the
test's own assertions passed is a complicated outcome.  A
verification step that failed for one cluster but not another
means part of the Pod was usable.

This chapter walks the framework's vocabulary for these
outcomes and the discipline for reporting them honestly.

## The four outcome categories per test

The base testbase outcome enum (`ResultCode` from
`automojo-testbase/source/packages/automojo/testbase/results/model/`)
distinguishes four states:

| Outcome | Meaning |
|---|---|
| **Passed** | The test ran and all its assertions succeeded |
| **Failed** | The test ran and an assertion failed |
| **Errored** | The test ran but raised an unexpected exception |
| **Skipped** | The test did not run (e.g., `@skip_if`, deactivation) |

The four categories cover the single-host case.  A distributed
run adds a fifth implicit category that the framework
represents through *test runs that never started*:

| Implicit | **Not attempted** | The test was discovered but the framework never reached it (run aborted, host unreachable for activation, verification failed) |

The framework's reporting acknowledges all five states
explicitly.

## The validator outcome — separate from the test outcome

Chapter 5 walked tandem validators: a test with three
validators is one test with four recorded outcomes (the test
plus three validators).  Each validator has its own outcome
category — passed, failed, errored.  Validators don't get a
"skipped" category because validators only run when their test
runs.

The aggregate outcome of a test-with-validators is:

```
test outcome = "Passed" if test.outcome == Passed AND all validators passed
test outcome = "Failed" if test.outcome == Failed OR any validator failed
test outcome = "Errored" if test.outcome == Errored OR any validator errored
test outcome = "Skipped" if test.outcome == Skipped
```

A test passes its own assertions while a validator catches a
violation — the aggregate is *Failed*.  This is the framework's
commitment: validators are not advisory.  Their failures count.

## Phase outcomes — coarser-grained than test outcomes

Volume 1 chapter 2 walked the sequencer phases (Initial,
Discovery, Collection, Graph, Traversal).  In a distributed
run, the activation phases (Configuration, Integration,
Operational from chapter 6 of this volume) and the verification
step (chapter 7) also have outcomes:

| Phase | Possible outcomes |
|---|---|
| Discovery | success / partial (some test files failed to import) / failed |
| Collection | success / failed (originator resolution failure) |
| Graph | success / failed (cycle detected) |
| Activation: Configuration | success / failed (file load / parse error) |
| Activation: Integration | success / partial (some devices failed init) / failed |
| Activation: Operational | success / partial (some devices unreachable) / failed |
| Verification: Landscape | success / failed (reachability check raised) |
| Verification: Topology | success / partial (some clusters mismatched) / failed |
| Traversal | success / partial (some tests skipped or not attempted) / failed |

Each phase gets its own JSOS phase_started / phase_finished
record (Volume 1 chapter 5).  The aggregate run outcome is
derived from the phase outcomes:

- If any phase failed (no recovery), the run is *aborted*.
- If any phase is partial, the run is *partial*.
- If all phases succeeded, the run is *complete*.

Combined with the test outcomes, the run report has:

- A run-level outcome (aborted / partial / complete).
- A phase-by-phase breakdown.
- A test-by-test breakdown.

## When a partial run is honest

A partial run is honest when the report acknowledges what was
*not* tested.  The framework's reporting is structured to
prevent the reader from accidentally treating a partial run
as a clean run.

A representative partial-run summary:

```json
{
  "run_id": "20260501-153012-abc123",
  "outcome": "partial",
  "phases": {
    "configuration": "success",
    "integration": "success",
    "operational": "partial",
    "verification": {
      "landscape": "partial",
      "topology": "success"
    },
    "traversal": "partial"
  },
  "tests": {
    "passed": 78,
    "failed": 3,
    "errored": 1,
    "skipped": 5,
    "not_attempted": 0
  },
  "operational_partial_reason": "1 of 12 devices unreachable",
  "landscape_partial_reason": "1 of 12 devices unreachable (same)",
  "traversal_partial_reason": "5 tests skipped because their declared host was unreachable",
  "host_log_status": {
    "casey-01": "complete",
    "casey-02": "complete",
    "casey-03": "complete",
    "casey-04": "complete",
    "casey-05": "complete",
    "casey-06": "complete",
    "casey-07": "complete",
    "casey-08": "complete",
    "casey-09": "complete",
    "casey-10": "complete",
    "casey-11": "complete",
    "casey-12": "unreachable_at_activation"
  },
  "missing_devices": ["casey-12"]
}
```

The summary tells the reader:

- 78 tests passed, 3 failed, 1 errored, 5 skipped because of
  the missing device.
- The Pod was 11/12 hosts.
- Verification caught the missing device at the operational
  phase; the run was allowed to continue because
  `allow_missing_devices=True` was in effect.

A consumer reading this can decide:

- "78 / 87 declared tests passed, but only 87 of 92 declared
  tests were attempted, and 1 of 12 hosts was unreachable" —
  is that acceptable for the use case?
- A nightly full-sweep run treats this as a *failure* (we
  expected 12 hosts; we got 11).
- A smoke run treats it as a *success* (the framework worked
  against 11 hosts; the failures are real and shippable
  signal).

The framework's job is not to decide which.  The framework's
job is to report honestly so the orchestrator (Volume 3) or
the human can decide.

## When a partial run is dishonest

A partial run is *dishonest* when the report:

- Reports "passed" without acknowledging that some tests
  weren't attempted.
- Aggregates outcomes such that a `not_attempted` count is
  hidden.
- Omits the phase outcomes, presenting only test outcomes.
- Reports test outcomes without the validator outcomes that
  failed.

The framework prevents these by *structurally* including the
honest fields in every summary.  A consumer that wants only a
boolean "did all the declared tests pass" must compute it from
the explicit fields:

```python
def did_all_declared_tests_pass(summary):
    rtnval = (
        summary["tests"]["passed"]
        == summary["tests"]["passed"]
            + summary["tests"]["failed"]
            + summary["tests"]["errored"]
            + summary["tests"]["skipped"]
            + summary["tests"]["not_attempted"]
    ) and summary["outcome"] == "complete"
    return rtnval
```

The boolean is computable but the consumer has to ask
explicitly.  No accidental "passed."

## The `allow_missing_devices` honesty trade-off

`LandscapeActivationParams.allow_missing_devices=True`
(chapter 6) lets activation proceed with unreachable devices.
The trade-off:

- **Benefit:** smoke runs can complete against partial Pods,
  catching framework bugs without requiring a perfect lab.
- **Cost:** a partial run is harder to interpret than an
  aborted run.  A reader must look at the report to know
  what was tested.

The framework's honesty discipline is what makes the trade-off
acceptable.  Without honest reporting, partial runs would be
misleading; with honest reporting, the trade-off is the
project's to make per use case.

## What partial-run failures look like in the JSOS stream

Volume 1 chapter 5's JSOS stream from the recorder carries
the phase events with their outcomes:

```
{kind: phase_started, name: configuration, ...}
{kind: phase_finished, name: configuration, outcome: success, ...}
{kind: phase_started, name: integration, ...}
{kind: phase_finished, name: integration, outcome: success, ...}
{kind: phase_started, name: operational, ...}
{kind: phase_finished, name: operational, outcome: partial,
 reason: "1 of 12 devices unreachable",
 missing_devices: ["casey-12"], ...}
{kind: phase_started, name: traversal, ...}
{kind: test_started, name: test_replication_in_majority, ...}
{kind: test_finished, name: test_replication_in_majority, outcome: passed, ...}
... (more tests) ...
{kind: test_skipped, name: test_failover_to_casey_12,
 reason: "Required host casey-12 was not in operational landscape", ...}
... (more tests, including the four other skipped ones) ...
{kind: phase_finished, name: traversal, outcome: partial,
 skipped_tests: [
   "test_failover_to_casey_12",
   "test_drift_includes_casey_12",
   "test_quorum_with_casey_12",
   "test_membership_query_casey_12",
   "test_health_check_all_hosts"
 ], ...}
{kind: run_finished, outcome: partial, ...}
```

A consumer parsing the stream sees the partial outcome at
each layer.  An AI agent triaging the run can ask "why was
the run partial?" and trace the answer through the phase
events.  Volume 1 chapter 6's `agents testresult failures`
extracts the failure subset; a similar `agents testresult
notattempted` (a project addition, not shipped) would
extract the not-attempted subset for review.

## The run report — three audiences

Different consumers of a run report want different things:

### CI dashboards

Want a green / red summary.  The right boolean for a CI
dashboard is *strict*: the run is "green" only if `outcome ==
"complete"` AND `failed + errored == 0`.  A partial run is
"yellow" or "red" depending on the project's stance.

### Triaging engineers

Want the failure list and the not-attempted list.  The agents
CLI's `testresult failures` and (project-supplied)
`testresult notattempted` are the surfaces.  The host log
references in the JSOS records lead to the per-host logs in
the results directory.

### Replay tools

Want enough information to *re-run* the suite.  The
`agents config extract` (Volume 1 chapter 6) produces the
exact invocation that triggered the run, including the
job_seed.  Combined with the partial run's
`missing_devices` list, a replay tool can decide whether to
re-attempt against the same Pod or escalate for
re-provisioning.

### AI agents

Want all of the above as structured data.  The framework's
JSOS stream and agents CLI shape are designed for this — same
shape, same fields, every agent reads the same way.

## What the next chapter does

Chapter 15 walks debugging a distributed failure.  Where this
chapter walked the *vocabulary* for partial outcomes, chapter
15 walks the *workflow* for diagnosing them: the audit-diary
pattern from the six-pass landscaping audit (the real
production-grade audit pattern from the portfolio's history),
the cross-host log-correlation discipline, the bisection
pattern for reproducing flakes, and the forward-reference to
Volume 8's time-travel debugging for the production-grade
post-mortem.
