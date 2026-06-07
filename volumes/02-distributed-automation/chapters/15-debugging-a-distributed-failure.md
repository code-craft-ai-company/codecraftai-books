# Chapter 15 — Debugging a Distributed Failure

> *Distributed automation produces failures that don't reproduce
> on a laptop.  The framework's debugging story is therefore
> not "step through the code"; it's "extract enough structured
> data from the failed run to reason about what happened across
> hosts and produce a hypothesis you can test."  This chapter
> walks the audit-diary pattern, with the six-pass landscaping
> audit as the reference example.*

A distributed test failed in CI.  The summary says one test
errored on host C, two validators caught steady-state
violations, and host F's log fetch was truncated.  The CI
runner has been torn down.  Re-running may or may not
reproduce.  How does an SDET diagnose what happened?

This chapter is the framework's answer.  It is *not* a
universal-debugging chapter — the framework can't replace
domain knowledge of the system under test.  It is a chapter
on *what the framework's structured data lets you do* and the
*audit-diary pattern* for working through it.

The full production-grade post-mortem story — replay-driven
debugging, time-travel reconstruction, pod instrumentation —
lives in Volume 8.  This chapter establishes the workflow at
the framework level; Volume 8 provides the tooling that makes
the workflow easier in production.

## The structured data the framework provides

The structured-data debugging story in this chapter is the
story of an *integration* run — a run driven by
`IntegratedTestJob` (in `automojo-integration-runtime`).  Plain
`testbase` runs standalone and records only to **stdout** (its
default recorder is `StdoutResultRecorder`); it has no presence
folder, no `RuntimeVariables`, and no on-disk results-directory
layout to debug from.  The presence-folder results directory,
the JSOS-to-disk stream, the rendered HTML summary, and the
CI/build metadata are all the responsibility of
`IntegratedTestJob`, which overrides three `TestJob` provider
hooks: `_create_output_context()` returns `RuntimeOutputContext`
(artefacts land under the activated app's well-known presence
folder via `automojo.thisapp.paths` — activation-gated),
`_create_run_metadata()` returns `RuntimeRunMetadata` (build /
pipeline / job from `RuntimeVariables`), and `_create_recorder()`
returns `JsonResultRecorder` (JSOS streaming plus the rendered
HTML summary).  The audit-diary and evidence discipline below
therefore applies to `IntegratedTestJob` runs — that's where the
files exist to reason about.

An integration run that failed leaves a results directory shaped
roughly:

```
results/20260501-153012-abc123/
├── testsummary.json                # high-level
├── testresults_stream.jsos         # streaming records
├── testresults.json                # consolidated
├── startup-configuration.json      # run inputs (Vol 1 ch.6)
├── host-logs/
│   ├── casey-01/
│   │   ├── syslog.log
│   │   ├── auth.log
│   │   └── application.log
│   ├── casey-02/
│   │   └── ...
│   └── casey-12/                   # may be partial — host went down
│       └── pre_only.log
├── diagnostics/
│   ├── pre_testrun/
│   │   ├── package-versions.json
│   │   ├── environment.json
│   │   └── per-host/
│   │       ├── casey-01.json       # host metadata snapshot
│   │       └── ...
│   └── post_testrun/
│       └── ...
└── tc-by-products/
    ├── test_replication_in_majority/
    │   ├── stdout.log
    │   ├── stderr.log
    │   └── per-host-output/
    │       ├── casey-01.log        # commands run on casey-01
    │       └── ...
    └── ...
```

That is the framework's gift to the debugger.  Every
structured fact about the run is recoverable from these
files.  The debugger's job is to reason about what
combination of facts produced the failure.

## The audit-diary pattern

The portfolio's debugging discipline — applied across multiple
audits in real history — is the **audit diary**.  An audit
diary is a structured note an SDET keeps while diagnosing a
failure.  It has six sections:

1. **Symptom.**  What the run reported.  Quote the JSOS
   record.
2. **Hypothesis-1.**  The first plausible explanation.
3. **Evidence for / against H1.**  What the structured data
   says.
4. **Hypothesis-2 (if H1 ruled out).**  Repeat.
5. **Diagnosis.**  The root cause once a hypothesis fits all
   the evidence.
6. **Remediation.**  The fix and how it's verified.

The discipline is not unique to this framework — it's a
generic engineering pattern.  The framework's contribution is
*making each section answerable* through the structured data.
Symptoms come from JSOS records; evidence comes from per-test
logs and host logs; remediation is verified by re-running the
suite with the same `job_seed` (chapter 9).

## A worked audit diary

A real-shape example.  Suppose a run failed with:

```json
{
  "run_id": "20260501-153012-abc123",
  "outcome": "partial",
  "tests": {"passed": 47, "failed": 1, "errored": 1, "skipped": 3},
  "phases": {
    "operational": "partial",
    "verification": {"landscape": "partial", "topology": "success"}
  },
  "missing_devices": ["casey-12"]
}
```

And the failure record:

```json
{
  "kind": "test_finished",
  "test_name": "test_quorum_after_partition",
  "outcome": "errored",
  "exception": {
    "type": "TimeoutError",
    "message": "Timeout waiting for cluster to converge",
    "traceback": [
      "  File \"tests/test_quorum.py\", line 87",
      "    wait_for_quorum(cluster, timeout=60)",
      "  File \"automojo/agentkit/cluster.py\", line 142, in wait_for_quorum",
      "    raise TimeoutError(...)"
    ]
  },
  "host_log_status": {
    "casey-01": "complete", "casey-02": "complete",
    "casey-03": "complete", "casey-04": "complete",
    "casey-05": "complete", "casey-06": "complete",
    "casey-07": "complete", "casey-08": "complete",
    "casey-09": "complete", "casey-10": "complete",
    "casey-11": "complete", "casey-12": "unreachable_at_activation"
  }
}
```

### Symptom

> `test_quorum_after_partition` errored with `TimeoutError` —
> "Timeout waiting for cluster to converge."  The Pod was 11/12
> hosts (casey-12 unreachable from activation).

### Hypothesis-1

> The test depends on casey-12 for quorum.  With casey-12
> down, the cluster cannot reach quorum and the
> `wait_for_quorum` times out.  Root cause: the test should
> have been skipped (like `test_failover_to_casey_12` was) but
> wasn't.

### Evidence for H1

- The test has no `@skip_if(host_unavailable("casey-12"))`
  decoration.  (Read the test source.)
- Tests that *were* skipped have explicit declarations.
  (Read the JSOS skip records: each carries
  `reason: "Required host casey-12 was not in operational
  landscape"` — the framework's deactivation chain caught
  them.)
- The test's host requirements are implicit — it walks the
  full cluster expecting all 12 hosts.

### Diagnosis

> The test's host requirements are implicit; the framework
> couldn't deactivate it because the framework didn't know
> what the test required.  The test ran against an 11-host
> cluster expecting a 12-host quorum.

### Remediation

> Add explicit host requirements to the test.  Either through
> a `@skip_if(host_unavailable(...))` decoration matching the
> ones on the explicitly-host-bound tests, or by reading the
> integrated landscape and skipping when the configured
> majority cannot be reached.  Re-run with the same `job_seed`
> against an 11-host Pod (recreating the failure
> deterministically); the test should now skip rather than
> error.

The audit-diary takes 15-30 minutes for a clear case like
this.  The framework's structured data is what makes it
possible to answer each section without re-running the suite.

## The cross-host log-correlation discipline

When a failure spans multiple hosts (a network partition
test, a replication test, a leader-election test), the audit
diary needs to correlate events across host logs.  The
framework's discipline:

- **Use the JSOS record's timestamps as the controller's
  reference clock.**  Volume 1 chapter 5 said the controller's
  clock is the JSOS clock.
- **Identify each host's offset from the controller** at the
  pre-test diagnostic capture.  The capture includes
  per-host time queries; the offset is `host_time -
  controller_time` at that moment.
- **Apply the offset (with caveats about drift) when reading
  host logs.**  A controller-side JSOS record at
  `T_controller = 15:32:00.500` corresponds to roughly
  `T_host_C = T_controller + offset_C` in casey-03's syslog.

The framework does *not* automatically do this correlation —
chapter 13 explained the design choice to keep timelines
side-by-side rather than forced-merged.  But the diagnostic
capture provides the offset values, and a debugging script
(or AI agent) can compute the correlation when needed.

## The bisection pattern for reproducing flakes

A test that fails 5% of the time is a flake.  The audit diary
for a flake has a different shape than for a deterministic
failure.

Three phases:

### Phase 1 — Reproduce the failure deterministically

Run the test repeatedly with random seeds until you have a
seed that triggers the failure.  Record the seed.  Now you
have a *reproducible* failure (same seed → same input → same
test behaviour, modulo Pod-state non-determinism).

### Phase 2 — Bisect on Pod state

Re-run the failure-inducing seed against a fresh Pod.  If the
failure reproduces, the failure is *Pod-state-independent*
(it's about the test or the system under test, not the
specific Pod state).  If it doesn't, the failure is
*Pod-state-dependent* — re-running won't help; you need
Volume 8's time-travel tooling to reconstruct the original
Pod state.

### Phase 3 — Bisect on inputs

For Pod-state-independent failures, the failure-inducing seed
gives you a specific input.  Use the framework's shrinking
machinery (Volume 1 chapter 4) to find the smallest input
that still triggers.  Add that input as a regression test
case.

The pattern is universal but the tools differ between
Pod-state-dependent and Pod-state-independent flakes.  Most
distributed flakes turn out to be Pod-state-dependent, which
is why Volume 8's tooling is so important for production
debugging.

## Reference: the six-pass landscaping audit

The portfolio's history includes a six-pass audit of
`automojo-landscaping` that fixed 15+ silent runtime bugs.
The audits are real production-grade examples of the
audit-diary pattern at scale.  The commits:

- **Pass 1 (`42f9c86`)** — six refactor-leftover bugs,
  including Liskov-substitution violations (three protocol
  extension subclasses lacked `**kwargs` forwarding), an
  uninitialized `ControlExtension._control_id`, three
  string-concatenation bugs, a mutable default argument in
  `factories.create_landscape()`, and duplicate dataclass
  definitions of `DatacenterSummary` and `FolderSummary`.
- **Pass 2 (`a92e5d2`)** — two attribute-name mismatches: a
  `LandscapeService.__init__` declaring `_primary_ssh_credential`
  but the search loop and property using `_ssh_credential`
  (silent `AttributeError` on credential lookup); an
  `OrchestrationJob.begin()` writing to `_orch_results_dir`
  but reading the nonexistent `_test_results_dir`.
- **Pass 3 (`1679fe7`)** — two more refactor-leftover bugs:
  `LandscapeIntegrationLayer.__init__` initialising pool
  attributes to `None` instead of `{}` (breaking `@property`
  accessors that called `.copy()` unconditionally); a
  `ClientSourcePackager` leaking file handles in zip-file
  digest computation.
- **Pass 4 (`1545f67`)** — five logic / behavioural bugs:
  exclude-filter "duplicate-add" bug at four call sites
  (missing `break`, wrong semantics — items that passed N
  filters were added N times); `FriendlyIdentifier.__eq__`
  always returning True (compared `self._full_identifier` to
  itself rather than `other._full_identifier`);
  `FriendlyIdentifier.identity` property dereferencing regex
  groups without null checks; wrong return-type annotations
  on `get_devices()` and `get_services()` (annotated to return
  single items instead of lists); inconsistent exception type
  in `ClientSourcePackager` (`Exception` instead of
  `RuntimeError`).
- **Passes 5 and 6** — incremental, integrated into specific
  test fixtures.

Three lessons from the six-pass arc:

### 1. Audits surface progressively

Each audit pass caught bugs the previous passes missed.  The
first pass caught structural bugs (Liskov violations,
uninitialised attributes, mutable defaults).  Later passes
caught semantic bugs (filter logic, equality semantics,
type annotations).  The progression reflects the reviewer's
*lens shift* — each pass looks for a different class of
issue.

### 2. Silent failures are common

Every bug fixed across the six passes was *silent at import
time*.  None of them caused an `ImportError`; none of them
broke the package's smoke tests.  They surfaced only when
specific code paths exercised the bug — and the code paths
were sometimes exercised only in distributed scenarios that
were rare in single-host tests.

The lesson: the framework's structured-data discipline is
critical for catching silent failures.  An audit diary working
from JSOS records can spot inconsistencies (a test that should
have been skipped wasn't; a credential that should have
resolved didn't) that no individual test exposed.

### 3. Tests added per fix

Every audit pass added regression tests for the bugs it
caught.  The Volume 1 chapter 10 case study mentioned that
testbase's own test suite runs around 3,700 tests; the
landscaping audit contributed +101 tests across 11 test files
to that count.  The tests are now permanent regressions
ensuring the same classes of bugs can't recur.

## What's not in this chapter — Volume 8

Production-grade post-mortem debugging — replaying a failed
run, walking the Pod's state at a specific moment,
reconstructing what happened with full instrumentation — is
Volume 8 (*Time-Travel Debugging in Production*).  Volume 8's
tools assume:

- The Pod was instrumented during the run.
- The instrumentation produced a replay-able event log.
- A debugger can replay the event log against a reconstructed
  Pod state.

For Pod-state-dependent flakes (the hard ones), Volume 8 is
the right tool.  This chapter walks what an SDET can do with
*just the framework's structured data*; that's enough for
~70% of distributed failures and a starting point for the
remaining 30% that need Volume 8's tooling.

## What the next chapter does

Chapter 16 is the volume's case study: a 5-host integration
test for a sysman feature.  The chapter walks the entire
volume's machinery in one worked example — Pod provisioning,
inheritance chain, landscape activation, verification, tandem
validators, the wire, credentials, partial-run semantics,
audit diary.

By the end of the case study, a reader who started Volume 2
cold should be able to read a real-world distributed test
job's source and predict its behaviour.  That fluency is the
volume's contract.
