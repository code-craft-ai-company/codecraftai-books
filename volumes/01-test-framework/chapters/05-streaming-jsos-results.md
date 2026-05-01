# Chapter 5 — Streaming JSOS Results

> *A test framework's results pipeline is the part that survives
> when everything else doesn't.  A run that crashes after three
> hours is useless if the framework was buffering its output for
> the final report.  This chapter is about why testbase streams,
> what it streams, and how a consumer reads it.*

`automojo-testbase` writes test results in a streaming format
called **JSOS** — *JSON Separated Object Stream*.  The format
is straightforward: each event is a self-contained JSON object,
prefixed by an ASCII record-separator character (Unicode code
point U+001E).  A consumer reads the file by scanning for the
separator, capturing the JSON between separators, and parsing.

The format exists because the alternative — buffering events
until the run ends and writing one consolidated JSON document —
loses everything when a run crashes.  Streaming means a partial
run is still a useful run.

This chapter walks the format, the recorder that produces it,
the three result files testbase emits, and the consumer patterns
that read the stream.

## What "streaming" means in this context

A streaming results file is one where:

1. Each record is appended to the file *as soon as the event
   happens*.
2. The file is parseable at any moment, including immediately
   after a crash.
3. A consumer can `tail -f` the file and process records in
   near-real-time.

JSOS satisfies all three.  Each record is a complete JSON
object; appending one record does not invalidate any previously
written records; a `tail` consumer can see a record as soon as
the recorder writes it (with Python's default buffering — and
the recorder flushes after each event for runs that need it).

Compared to alternatives:

- **JSON Lines** (newline-delimited JSON) is the closest
  cousin.  JSOS uses the ASCII record-separator character
  (`\x1e`) instead of `\n` because some result fields legitimately
  contain newlines (multi-line stack traces, multi-line log
  excerpts).  A newline-delimited format requires escaping
  every newline in the payload; the record-separator character
  doesn't appear in normal text, so no escaping is needed.
- **JUnit XML** is the format CI dashboards historically read.
  It is not streamable: the document is closed at the end of
  the run.  Testbase produces a JSON-shaped consolidated file
  for batch consumers, but JSOS is the live stream.
- **Protobuf streams** would be more compact but harder to
  inspect by hand.  The portfolio's debuggability principle from
  Volume 0 chapter 1 prefers human-readable text.

## The record separator

The single line that defines the format is in the recorder
import block:

```python
# automojo-testbase/source/packages/automojo/testbase/results/recorders/jsonresultrecorder.py:24

from automojo.xmodules.jsos import CHAR_RECORD_SEPERATOR
```

`CHAR_RECORD_SEPERATOR` is the byte `\x1e`.  Every JSOS file is
a sequence of `(separator, json_object)` pairs.  The first byte
of the file is a separator; from there each separator marks the
start of the next record.

A reader does not need testbase to parse the file.  Any program
that can split on `\x1e` and parse JSON can consume the stream:

```python
# A minimal reader.

def read_jsos(path: Path) -> Iterator[dict]:
    """
        Read a JSOS file and yield each record as a dict.
    """
    with path.open("rb") as handle:
        raw = handle.read()
    parts = raw.split(b"\x1e")
    for part in parts:
        text = part.strip()
        if len(text) == 0:
            continue
        record = json.loads(text)
        yield record
    return
```

The chapter does *not* claim this is the consumer the framework
ships — Chapter 6 walks the agents CLI's parser
(`jsosparser.py`).  But the format is open: any reader that
can find `\x1e` and parse JSON between occurrences is a
conforming consumer.

## The three result files

`JsonResultRecorder`
(`automojo-testbase/source/packages/automojo/testbase/results/recorders/jsonresultrecorder.py:90–183`)
produces three files in the run's results directory:

### `testsummary.json`

Written at finalize.  The high-level summary of the run:

```json
{
  "run_id": "20260430-153012-abc123",
  "started_at": "2026-04-30T15:30:12.123Z",
  "finished_at": "2026-04-30T16:19:43.456Z",
  "duration_seconds": 2971,
  "total": 3712,
  "passed": 3680,
  "failed": 18,
  "errored": 4,
  "skipped": 10,
  "host": {
    "hostname": "ci-runner-04",
    "os": "Linux 6.5.0",
    "python": "3.12.5"
  }
}
```

This is the artefact a CI dashboard polls for status.  It does
not include per-test detail; for that, the consumer reads the
stream or the consolidated file.

### `testresults_stream.jsos`

Written *as tests finish*.  Each record is one of a small set
of event kinds:

- `test_started`
- `test_finished`  (with `outcome` ∈ `{passed, failed, errored, skipped}`)
- `property_iteration`  (one per iteration of a `@given` test)
- `phase_started`  (one per sequencer phase)
- `phase_finished`
- `diagnostic_capture`  (host details, package versions, etc.)
- `coupling_event`  (integration coupling lifecycle)

A representative `test_finished` record:

```json
{
  "kind": "test_finished",
  "test_name": "automojo.agentkit.tests.test_costtracker.test_addition_is_commutative",
  "started_at": "2026-04-30T15:30:14.001Z",
  "finished_at": "2026-04-30T15:30:14.018Z",
  "duration_ms": 17,
  "outcome": "passed",
  "monikers": ["unit"],
  "pivots": null
}
```

A representative `test_finished` for a failure:

```json
{
  "kind": "test_finished",
  "test_name": "automojo.agentkit.tests.test_invoice.test_invoice_total_includes_tax",
  "started_at": "2026-04-30T15:35:21.501Z",
  "finished_at": "2026-04-30T15:35:21.612Z",
  "duration_ms": 111,
  "outcome": "failed",
  "monikers": ["billing"],
  "exception": {
    "type": "AssertionError",
    "message": "expected Decimal('108.75'), got Decimal('100.00')",
    "traceback": [
      "  File \".../test_invoice.py\", line 14, in test_invoice_total_includes_tax",
      "    assert rtnval"
    ]
  }
}
```

The stream is the rich source of truth.  Anything you can ask
about a test run can be answered by parsing the stream.

### `testresults.json`

Written at finalize.  The consolidated final document.  Same
information as the stream, reorganised into a tree by package
→ module → test for batch consumers (post-run analysis tools,
human readers).  Producing both is cheap because the recorder
already has every record in memory by finalize time.

The consolidated file is the one a build engineer commits to a
results-archive bucket; the stream file is what they keep
locally for live monitoring.

## The recorder lifecycle

The recorder is constructed during `TestJob.begin()` (Chapter
2):

```python
# Sketch of the construction flow.

def begin(self) -> None:
    """
        Set up the run's results directory and recorder.
    """
    self._results_dir = self._compute_results_dir()
    self._recorder = JsonResultRecorder(
        results_dir=self._results_dir,
        run_id=self._run_id,
    )
    self._recorder.open()
    return
```

The recorder's `open()` method:

1. Creates the results directory if it does not exist.
2. Opens `testresults_stream.jsos` in append mode.
3. Writes the first record-separator byte.
4. Writes a `phase_started` record for the run.

Every test event the sequencer emits is then forwarded to the
recorder's `record_event(event)` method, which:

1. Serialises the event to JSON.
2. Writes the record-separator byte.
3. Writes the JSON.
4. Calls `flush()` on the underlying file handle.

The flush is what makes the stream tail-able.  Without it
Python's I/O buffering would batch writes and a tail consumer
would see records in lumps.  The flush has a measurable cost on
fast suites — the recorder lets the test author opt out for
unit-suite timings via a constructor argument — but on
automation suites the per-test cost is dwarfed by the test
itself, and the flush is worth its weight.

At `TestJob.finalize()`:

1. `record_event` writes a `phase_finished` record for the run.
2. The recorder builds the consolidated JSON tree from its
   in-memory event log.
3. Writes `testresults.json`.
4. Builds the summary aggregates.
5. Writes `testsummary.json`.
6. Closes the stream file.

If the run crashes between `open()` and `finalize()`, the
stream file is partial but valid: every record between the
opening separator and the last full record is parseable.  A
consumer reading the partial stream can compute its own summary
from what's there.

## Streaming to external services

For runs that need to report to an external dashboard while
running, testbase ships a *streaming encoder* and *result
streamer* alongside the recorder
(`automojo-testbase/source/packages/automojo/testbase/results/streaming/resultstreamer.py:44–78`).

The classes:

- **`StreamingJsonEncoder`** (line 44) — a `json.JSONEncoder`
  subclass that handles `datetime`, `Decimal`, and other types
  the standard encoder doesn't.  Used by the recorder to
  serialise events.
- **`ResultStreamer`** (line 64) — a worker that runs in a
  background thread, reads events from a queue, and POSTs them
  to an external URL.  Used when the test job is configured
  with a `streaming_url`.

The streamer is opt-in: when the job's constructor receives a
`streaming_url`, the recorder also pushes events into the
streamer's queue.  The streamer does best-effort delivery —
failures are logged but don't fail the run — because the local
JSOS stream is the source of truth and the remote stream is a
convenience.

This is the Aspects pattern from Volume 0 chapter 4 once more:
the recorder's behaviour (write locally vs. write locally and
push remotely) is parameterised, not decorated.  A test author
writes the same test; the build engineer picks the recorder's
behaviour.

## Consumer patterns

Three consumer shapes show up across the portfolio:

### Live monitoring

A developer running a long suite watches the stream:

```bash
tail -F results/testresults_stream.jsos | testbase agents testresult list --stream
```

The agents CLI's `testresult list --stream` command (Chapter 6)
parses the JSOS as it arrives and prints a one-line summary per
test.  A failing test is colour-coded.  When the run ends the
stream tail closes cleanly.

### Failure extraction

After a run, an engineer wants the failures only:

```bash
testbase agents testresult failures --results results/
```

The `failures` subcommand reads the stream file, filters to
`outcome != passed`, and emits a structured JSON document an
engineer can grep through or feed to an agent.  The
`automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/testresult/failures.py`
file is the implementation.

### Dashboard ingestion

A CI dashboard polls the summary file every minute, then on
completion ingests the consolidated JSON for per-test detail.
The dashboard does *not* normally consume the JSOS stream — it
is the live tool, not the historical record.

For dashboards that *do* want live updates, the
`ResultStreamer`'s remote-streaming mode pushes events to a URL
the dashboard owns.  The dashboard receives JSON-encoded events
in real time without having to poll a file.

## Replayability

A point worth making explicit: the stream is *the* source of
truth.  The summary and consolidated files are both derivable
from the stream.  If a build engineer keeps only the stream,
they can recompute the other two at any time.

This matters for compliance and audit cases where every test
event must be archived but storage is at a premium.  The stream
is the smallest representation that loses nothing.  The
consolidated file is a convenience.

It also matters for replay: a run's stream can be fed into a
*replayer* that reproduces the run's timing on a different
host, for human or agent viewing.  Volume 8 (*Time-Travel
Debugging in Production*) builds on the same pattern at a
different layer of the stack — the principles are the same.

## What stays out of the stream

Some things deliberately do not become JSOS records:

- **Per-test stdout / stderr.**  These are written to a
  per-test directory under the results directory
  (`results/<run_id>/<test_name>/stdout.log` and `stderr.log`).
  The stream record carries a *reference* to the directory but
  not the contents — log volume can be large, and dragging it
  into the stream would balloon record sizes.
- **Per-test screenshots / artefacts.**  Same — a reference,
  not the bytes.
- **Coverage data.**  Coverage is written as a separate file by
  the coverage plugin, not as a JSOS record.

The principle: the stream carries the *structured* data of the
run; the directory carries the *blob* data.  A consumer that
needs the blobs walks the directory; a consumer that needs the
structure parses the stream.

## The agents-readable contract

A note of forward reference: the *exact same JSOS records* the
recorder produces are what the `agents` CLI subcommands
(Chapter 6) parse.  There is no separate "agent format" — the
contract between the framework and any consumer (human or AI)
is the JSOS stream.  This is why Chapter 6 treats the agents
CLI as a thin parser-and-formatter on top of the stream rather
than a separate subsystem.

## What's next

- Chapter 6 — The `testbase agents` CLI.  The thin layer that
  parses the JSOS stream and presents it to humans and AI
  agents alike.
- Chapter 7 — Mocking.  Tests that don't touch real systems
  still produce JSOS records — the recorder treats mocks
  identically to real resources.
