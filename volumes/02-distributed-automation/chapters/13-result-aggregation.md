# Chapter 13 — Result Aggregation

> *Volume 1 chapter 5 walked single-host JSOS streaming.  In a
> distributed run, the controller's stream is the source of
> truth for what the *test* did; the per-host streams (when they
> exist) are the source of truth for what the *hosts* did.
> Aggregation is the discipline of preserving both timelines
> without forcing them into one global timeline.*

The single-host JSOS story from Volume 1 chapter 5 is clean.
One Python process; one recorder; one stream file; per-test
events flushed as they happen.  A consumer tails the stream,
reads records as they appear, builds an incremental view of
the run.

In a distributed run, the picture has a controller and N
hosts.  The controller runs the test job; the hosts execute
the controller's commands and respond.  The question:
*where do the streams come from, and how does a consumer
make sense of them?*

The framework's answer is short: **the controller produces
the JSOS stream.  The hosts produce their own logs,
referenced from the JSOS records but not merged into them.**
This chapter walks why.

## Where the JSOS stream comes from

The `DistributedTestJob` runs on the controller.  It owns:

- The `JsonResultRecorder` (Volume 1 chapter 5) — but it does
  not *install* that recorder itself.  The JSOS-to-disk recorder
  is supplied by the `IntegratedTestJob` layer it inherits from
  (`automojo-integration-runtime`); plain `testbase` `TestJob`
  defaults to a stdout-only recorder and writes nothing to a
  results directory.  See "The recorder in distributed mode"
  below.
- The `testrun_results_stream.jsos` file the recorder writes to
  (rooted under the activated application's presence folder by
  `IntegratedTestJob`'s output context).
- All the test-shaped events: `test_started`, `test_finished`,
  `validator_started`, `validator_finished`, `phase_started`,
  `phase_finished`, `diagnostic_capture`, `coupling_event`.

Every test-shaped event the framework emits during the run is
written to the controller's JSOS stream.  No event is written
by the hosts; the hosts don't have a recorder.

Why?  Because the hosts are not actors (chapter 2).  Hosts
respond to the controller's commands; they don't make
decisions about test outcomes.  The test outcome is decided
*on the controller* by the controller-side code.  The JSOS
stream records the controller's view of the run, which is the
authoritative view of what *tests* did.

## What the hosts produce

Each host produces its own *operational* logs — syslog,
application logs, audit trails.  These are the system under
test's own records, not the framework's.  The hosts don't
know they're being tested; they're just running, logging
ordinary state.

The framework's JSOS records *reference* host logs but don't
contain them:

```json
{
  "kind": "test_finished",
  "test_name": "test_replication_recovery",
  "outcome": "failed",
  "host_logs": {
    "casey-01": "results/20260501-153012/host-logs/casey-01/syslog.log",
    "casey-02": "results/20260501-153012/host-logs/casey-02/syslog.log",
    "casey-03": "results/20260501-153012/host-logs/casey-03/syslog.log"
  },
  "diagnostic_captures": ["pre_testrun", "post_testrun"]
}
```

The record references *paths* — files in the results directory
where the controller has fetched the host's logs.  A consumer
reading the JSOS record can walk to the file for the actual
log content.

This is the same separation Volume 1 chapter 5 set up for
single-host runs: the JSOS stream carries *structured* data;
the directory carries *blob* data.  Distributed runs follow
the same model with per-host blob data added.

## How host logs reach the results directory

The framework fetches host logs via the SSH wire (chapter 10)
during the diagnostic capture phases.  The flow:

1. `fire_diagnostic_capture_pre_testrun` runs (Volume 1
   chapter 2's lifecycle).
2. The framework's diagnostic capture, configured per-Pod,
   walks each device and pulls relevant log files via
   `device.ssh.file_pull`.
3. Files land in `results/<run_id>/host-logs/<device_name>/`.
4. The JSOS record for the diagnostic event references the
   directory.

Same flow at `fire_diagnostic_capture_post_testrun`.  Two
captures per run (pre and post); a project can configure
additional captures (e.g., post-failure) by overriding the
relevant hooks.

The capture is *host-pull*, not host-push.  The controller
pulls; the hosts don't initiate.  Two reasons:

- **Single-actor consistency.**  The controller is the actor;
  pulling logs is one of the actor's actions.  A host
  pushing logs would be a second actor causing effects (the
  push) in the environment.
- **Reliability.**  The controller can re-attempt a pull that
  fails; a host that fails to push has no controller-side
  recovery path.

## Per-host timelines vs one global timeline

A subtle question: should the framework merge the controller's
JSOS records and the hosts' log files into one global
timeline?

The framework's answer: **no, preserve per-host timelines as
separate streams.**

The reason is clock drift (chapter 1's failure mode #4).  Host
A's syslog timestamps and host B's syslog timestamps come from
independent clocks.  Aligning them onto a global timeline
requires assuming the clocks agree — and on real fleets, they
don't.  A per-host millisecond can be a multi-second offset
from the controller's clock, and the offset can drift during
the run.

The framework's discipline:

- The **JSOS stream** uses the *controller's* clock for every
  record.  Test events have controller timestamps.
- **Host logs** use each *host's* clock for their entries.
  Host log entries have host timestamps.
- When aggregating, **don't merge the two**.  Present them
  side-by-side, each with its own timeline, and let the
  consumer reason about offsets if they need to.

For the consumer (a human doing a post-mortem, or an AI agent
investigating a failure), this is more useful than a forced
global timeline.  The forced merge would lie subtly; the
side-by-side presentation is honest about what's known and
what isn't.

## The failure mode where a host's stream stops

Distributed runs have a failure mode that single-host runs
don't: a host can lose connectivity mid-run and the framework
loses access to its logs from that point.

The framework's response:

- The JSOS record for the failed test (or phase) marks the
  host as *unreachable* in its `host_logs` reference.
- The post-run diagnostic capture re-attempts the host log
  fetch; if it fails again, the JSOS record carries a
  `truncated: true` flag for that host's logs.
- The aggregate run report (Volume 1 chapter 6's agents CLI;
  chapter 14 of this volume) preserves the partial state.

A consumer reading a partial run sees:

```json
{
  "summary": {
    "total": 87,
    "passed": 82,
    "failed": 4,
    "errored": 1,
    "host_log_status": {
      "casey-01": "complete",
      "casey-02": "complete",
      "casey-03": "truncated_at_post_capture",
      "casey-04": "complete",
      "casey-05": "complete"
    }
  }
}
```

The summary is honest.  Tests' outcomes are recorded
correctly (the controller's view).  Host logs are
acknowledged as partial where they are partial.

This is the partial-run honesty chapter 14 walks in detail.

## The recorder in distributed mode

A subtle implementation detail worth pinning down: the
`JsonResultRecorder` from Volume 1 chapter 5 is the same
recorder used in distributed mode.  There is no
`DistributedJsonResultRecorder` subclass.  Why?

Because the recorder is *controller-side* and the controller
is doing the same thing it did on single-host: emit JSOS
records as test events fire.  The fact that the test events
involve cross-host calls doesn't change the recorder's job.
The recorder is downstream of the test execution; it doesn't
need to know whether the execution was distributed or local.

But note *which* job installs that recorder.  Plain `testbase`
`TestJob` does **not** write JSOS-to-disk by default: its
`_create_recorder()` returns a `StdoutResultRecorder`
(`automojo.testbase.results.recorders.stdoutresultrecorder`,
testjob.py:564), which prints the run summary to stdout and
reuses the JSOS/JSON serialisation (so the stream and summary
files are still written *when an output directory is in play*)
but skips the rendered HTML summary.  The full controller-side
recorder — JSOS streaming plus rendered HTML — is installed one
layer up, by `IntegratedTestJob` in `automojo-integration-runtime`,
whose `_create_recorder()` returns the `JsonResultRecorder`
(integratedtestjob.py:79).  `IntegratedTestJob` is also what
roots those files under the activated application's presence
folder (via its `RuntimeOutputContext`) and stamps them with
CI/build metadata (via its `RuntimeRunMetadata`).  In the
*design* chain `IntegratedTestJob` sits between `TestJob` and
`InteropLibraryTestJob`, so `DistributedTestJob` picks up the
full JSOS-to-disk + HTML recorder, the presence-folder output
context, and the `RuntimeVariables` metadata **by inheritance** —
not because `DistributedTestJob` (or `testbase`) supplies it.

> *In-flight note — read this before assuming the inheritance.*
> The re-parent has **not yet landed**.  In today's interop
> source `InteropLibraryTestJob` still extends `TestJob`
> directly (`automojo.interop.testing.testjob`, testjob.py:22),
> so the live chain `DistributedTestJob → InteropLibraryTestJob →
> TestJob` does **not** pass through `IntegratedTestJob` yet.  As
> built, a `DistributedTestJob` therefore inherits `TestJob`'s
> `StdoutResultRecorder` + standalone seams — it does *not* yet
> produce the presence-folder JSOS/HTML aggregation this chapter
> describes.  That aggregation is produced by running an
> `IntegratedTestJob` directly, and — once the Phase-5 re-parent
> of `InteropLibraryTestJob` onto `IntegratedTestJob` lands — by
> any `DistributedTestJob`.  The rest of this chapter describes
> the `IntegratedTestJob`-supplied recorder's output: the target
> state for distributed runs.

The recorder's outputs (the files from Volume 1 chapter
5, written by the `IntegratedTestJob`-supplied recorder):

- `testrun_summary.json` — high-level summary.  Distributed
  runs add the `host_log_status` field shown above.
- `testrun_results_stream.jsos` — the streaming record.
  Distributed-test records carry richer host metadata
  (per-call references to host logs, per-validator host
  identity).
- the rendered HTML summary — the human-readable view
  `IntegratedTestJob`'s recorder adds on top of the JSOS
  stream (the stdout-only `testbase` default skips it).

A consumer parsing the stream uses the same JSOS parser
(Volume 1 chapter 6's `agents` CLI) for both single-host and
distributed runs.  The parser doesn't need to be aware of
distribution; it just walks records.

## Streaming to external services in distributed mode

Volume 1 chapter 5 introduced `ResultStreamer` for pushing
events to an external dashboard URL.  In distributed mode, the
streamer runs on the controller — same as the recorder.  The
external dashboard receives the controller's view of the
distributed run.

For dashboards that want per-host data in real time, the
framework's pattern is to add *tandem validators* (chapter 5)
that emit per-host events.  A `LogTailValidator` attached to
a test could subscribe to a host's syslog through SSH and
emit JSOS records for noteworthy entries; the streamer would
push those records to the dashboard alongside the test events.

The framework doesn't ship a built-in `LogTailValidator` (it
would have too many project-specific decisions about what
counts as "noteworthy").  But the validator scaffolding from
chapter 5 is the right shape for projects that want this.

## What the agents CLI sees

The `testbase agents` subcommands from Volume 1 chapter 6
work on distributed JSOS streams unchanged:

- `agents summary extract` — same shape; adds
  `host_log_status` to the output.
- `agents testresult list` — same shape; per-test records
  may have richer metadata about which hosts were involved.
- `agents testresult failures` — same shape; the failure
  records reference host logs via paths.
- `agents config extract` — same shape; the configuration
  reflects the distributed Pod's Landscape and Topology.

The unchanged shape is the framework's commitment to a
*structured surface* for consumers.  An AI agent reading a
single-host run and a distributed run uses the same parsing
code; the agent's reasoning code may handle the per-host
metadata differently (offering to walk to host logs on
demand) but the input shape is identical.

## What the next chapter does

Chapter 14 walks partial-run semantics — what "the suite
passed" means when one of five hosts crashed mid-run, when
verification caught a Topology mismatch on one cluster but
not another, when the framework activated only some of the
declared services because `allow_missing_devices=True`.  The
framework's commitment is *honesty about scope*: a partial
run reports what ran, what didn't, and why, without
overstating success.

Chapter 15 then walks debugging a distributed failure using
all the machinery established so far — the audit-diary
pattern from the six-pass landscaping audit as the reference
shape.
