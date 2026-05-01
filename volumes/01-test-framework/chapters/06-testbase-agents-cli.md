# Chapter 6 — The `testbase agents` Machine-Readable CLI

> *Most CI dashboards extract test data by parsing stdout.  Most
> AI agents extract test data by asking another agent to scrape
> the dashboard.  Both approaches lose information at every
> hop.  This chapter is about a CLI that is structured at the
> source — so any consumer, human or machine, reads the same
> shape.*

The `testbase agents` subcommand exists because at some point a
test framework's output stops being for humans only.  CI
dashboards want machine-readable data.  AI agents reading test
results want machine-readable data.  Build engineers writing
post-run scripts want machine-readable data.

The cheap answer is "parse the JSOS stream."  Chapter 5 showed
that the stream is the source of truth.  But making every
consumer write its own JSOS parser is a tax — and a tax that
gets paid wrongly.  Some consumers will mishandle the
record-separator byte; others will assume newline-delimited
records; an AI agent will hallucinate the schema.

The `agents` CLI is the *blessed* parser.  Its three subcommand
groups (`summary`, `testresult`, `config`) read the JSOS
stream, parse it correctly, and emit a structured shape — JSON
to stdout — that any consumer can rely on.  Same shape every
time.

This chapter walks the three subcommand groups, the rationale
for each, and the way they compose with the framework's
streaming model.

## The shape of an "agent" subcommand

The vocabulary around the `agents` subcommand was chosen with
AI agents in mind, but the audience is broader.  In the
portfolio's vocabulary, an *agent* is anything that wants
machine-readable output: a CI dashboard, a script, a human
running `--json` for grep-friendliness, an AI agent in a
post-run review loop.

The structural commitments of every `agents` subcommand are:

1. **JSON to stdout.**  No tables, no colour codes, no
   human-formatted text.  A consumer calls the subcommand and
   parses the JSON.
2. **A stable schema.**  The output shape is documented in the
   subcommand's `--help` and does not silently change between
   minor versions.
3. **Failures to stderr.**  Stdout carries only the structured
   answer; stderr carries human-readable context for when the
   subcommand cannot answer.
4. **Exit codes are meaningful.**  0 = success, 1 = subcommand
   error, 2 = usage error, with conventional CLI semantics.

These commitments mean a consumer can pipe `agents` output to
`jq`, to a Python script, to an LLM context window, to anything
that handles JSON — without writing a parser.

## The group declaration

Top of the agents subcommand tree:

```python
# automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/__init__.py:18–28

@arcli.group("agents")
def group_testbase_agents():
    """
        Commands for AI agents to extract and analyze test result data.
    """
    return

group_testbase_agents.add_command(group_testbase_agents_summary)
group_testbase_agents.add_command(group_testbase_agents_testresult)
group_testbase_agents.add_command(group_testbase_agents_config)
```

The pattern is the arcli group pattern from Chapter 8: a
`@arcli.group("name")` decorator that turns a function into a
`GroupingCommand`, then `.add_command(...)` to attach
subcommand groups.  Each of the three sub-groups has its own
file under `cli/cmdtree/agents/`:

- `agents/summary/` — extract summary information
- `agents/testresult/` — list, filter, extract individual test
  results
- `agents/config/` — extract and inspect configuration

The triple maps onto the three result-file pairs from Chapter
5: `summary` reads `testsummary.json`, `testresult` reads the
JSOS stream and the consolidated file, `config` reads the
run's startup-configuration file.

## `agents summary`

The `summary` group answers high-level questions about a run:

- Total counts (passed, failed, errored, skipped).
- Wall-clock duration.
- Host fingerprint.
- Whether the run completed cleanly.

Sample subcommands (representative — the actual surface may
expand):

```bash
testbase agents summary extract --results <results-dir>
testbase agents summary compare --baseline <dir-a> --candidate <dir-b>
```

The `extract` form prints the summary file as JSON, with
nothing else.  The `compare` form is more interesting: it
diffs two summary files and reports the deltas — counts moved,
new failures, regressions.  A CI dashboard runs `compare` on
every PR run against the trunk's last green run; the output is
a structured "what changed" record an agent can consume.

## `agents testresult`

The `testresult` group answers per-test questions:

- List every test in the run with its outcome.
- Filter to the failures.
- Extract one specific test's full record.

Two subcommands matter most:

### `agents testresult list`

```python
# automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/testresult/list.py
```

Emits one JSON object per test, in the order the tests
finished:

```json
{"test_name": "...", "outcome": "passed", "duration_ms": 17}
{"test_name": "...", "outcome": "failed", "duration_ms": 111}
```

This is JSON Lines, not JSOS — because `list` is the
*human-readable* of the agent commands.  A consumer pipes the
output to `jq`, or `grep`, or `awk`.  An AI agent reads the
whole thing and reasons about it.

The subcommand accepts `--include` / `--exclude` flags that
filter by test-name pattern, mirroring the `TestCollector`'s
filter vocabulary (Chapter 2).

### `agents testresult failures`

```python
# automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/testresult/failures.py
```

The shortcut for the most common consumer query: "what failed."
Emits a JSON document with one entry per failed test, including
the exception type, message, and a representative chunk of the
traceback.  A representative output:

```json
{
  "run_id": "20260430-153012-abc123",
  "failures": [
    {
      "test_name": "automojo.agentkit.tests.test_invoice.test_invoice_total_includes_tax",
      "outcome": "failed",
      "exception_type": "AssertionError",
      "exception_message": "expected Decimal('108.75'), got Decimal('100.00')",
      "traceback_excerpt": [
        "  File \".../test_invoice.py\", line 14, in test_invoice_total_includes_tax",
        "    assert rtnval"
      ],
      "duration_ms": 111,
      "monikers": ["billing"]
    }
  ],
  "failed_count": 1,
  "errored_count": 0
}
```

This is the subcommand an AI agent's review loop calls first.
The output is small enough to fit in a context window even for
runs with thousands of tests, because each failure carries only
a *traceback excerpt* — the top frame plus the assertion line —
not the full traceback.  A consumer that needs the full
traceback walks to the per-test directory under the results
directory; the JSOS reference is in the record.

## `agents config`

The `config` group is the smallest of the three but the most
interesting in scope.  Where `summary` and `testresult` answer
"what happened in the run," `config` answers "what *would*
happen, given this configuration."

### `agents config extract`

```python
# automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/config/extract.py:32–50

@arcli.option("--config-file", required=True, help=HELP_CONFIG_FILE)
@arcli.command("extract")
def command_agents_config_extract(config_file: str):
    """
        Extracts key configuration information from a test run
        startup-configuration.json file.
    """
    try:
        resolved = resolve_file_path(config_file)
    except FileNotFoundError as err:
        format_error(str(err))
        sys.exit(1)
    # ...
```

The subcommand reads the run's `startup-configuration.json` —
the file the framework writes at the start of a run capturing
*everything* that influenced how the run was constituted — and
emits a flattened, structured projection.  The projection
includes:

- The test root.
- The includes / excludes that selected the tests.
- The runtime configuration profile in effect.
- The `TESTBASE_MAX_EXAMPLES_CAP` value (Chapter 4).
- The job seed.
- The host fingerprint.

A consumer running this against a known-good run gets enough
information to reproduce the run.  An AI agent reasoning about
"why did this run have only 200 tests when the suite has 3,700"
can read the includes/excludes from the extract and answer in
one query — without having to grok the run's invocation
arguments from a CI log.

## Composability

The three subcommands compose:

```bash
# What changed in failures between the baseline and the candidate?
diff <(testbase agents testresult failures --results baseline/) \
     <(testbase agents testresult failures --results candidate/)
```

```bash
# Every new failure (jq pipeline).
testbase agents testresult failures --results candidate/ \
  | jq '.failures[] | select(.test_name as $n |
       any($baseline_failures; .test_name == $n) | not)' \
       --argjson baseline_failures "$(testbase agents testresult failures --results baseline/ | jq '.failures')"
```

The agents CLI is not the orchestrator.  The agents CLI is the
*adapter* between the JSOS stream (and friends) and a
shell / script / agent that does the orchestrating.  The
composition is the consumer's responsibility.

## What this contract gives an AI agent

A specific scenario the agents CLI was designed for: an AI
agent reviewing a CI run.  The agent's flow:

1. Call `agents summary extract` to learn whether the run
   completed and how many tests failed.
2. If failures exist, call `agents testresult failures` to get
   a structured list.
3. For each failure, decide whether the failure pattern looks
   like a flake (compare to last week's `failures` output),
   a regression (compare to the trunk's `failures` output), or
   a real bug (no precedent).
4. For real bugs, call `agents config extract` to capture the
   exact invocation that triggered the bug.
5. Open an issue with the structured data and the engineer's
   suggested next step.

The agent does not need to parse JSOS.  The agent does not
need to know about the record-separator byte.  The agent's
input is JSON; its output is JSON; its reasoning runs over
structured data the framework guarantees the shape of.

## What this contract gives a human

The same agent-readable surface gives the human a consistent
script library:

```bash
# In every CI run that fails, pipe the failures to my dashboard.
testbase agents testresult failures --results "$RESULTS" \
  | curl -X POST -H 'Content-Type: application/json' --data @- \
    https://dashboard.internal/api/runs

# In every nightly, diff against the trunk.
diff <(testbase agents summary extract --results candidate/) \
     <(testbase agents summary extract --results trunk/)
```

The same data, the same shape.  No human-only formatter, no
agent-only formatter.  The framework commits to one structured
output and lets every consumer build on it.

## Implementation notes

A few patterns worth calling out for an SDET extending the
agents tree:

### The shared JSOS parser

```python
# automojo-testbase/source/packages/automojo/testbase/cli/cmdtree/agents/jsosparser.py
```

Every `agents` subcommand that reads the JSOS stream uses this
parser.  Centralising the parser means the framework's
record-separator handling, JSON deserialisation, and error
reporting are written once.  An extension subcommand that
needs to read the stream should reuse the parser; an
extension that writes its own parser will get the
record-separator byte wrong.

### Failure mode discipline

Each subcommand handles four failure modes explicitly:

- `--results` directory does not exist → exit 1, error to
  stderr.
- Results directory exists but has no JSOS stream →  exit 1,
  error to stderr.
- JSOS stream is partial (run crashed) → exit 0, structured
  output with a `truncated: true` flag.
- Any other unexpected error → uncaught, bubbles per Volume 0
  chapter 3's discipline.

The third mode is the interesting one: a partial run is *still
useful*.  The agents CLI treats partial streams as legitimate
input, flags them clearly, and lets the consumer decide what
to do.  This is the streaming-format's payoff from Chapter 5
realised at the consumer layer.

### Exit codes

```
0 — Success.  Subcommand produced its structured output.
1 — Subcommand error: input invalid, results directory unreadable, etc.
2 — Usage error: unknown flag, missing required argument.
```

Conventional CLI semantics, deliberately so.  A consumer that
uses `set -e` or `if cmd; then` works without surprise.

## Where the agents CLI plugs into Volume 7

A forward reference: Volume 7 (*Building an AI Agent SDK*)
covers the lifecycle of a production AI agent — tools, memory,
observability.  One of the canonical *tools* a testing-focused
agent owns is "run the testbase agents subcommand and parse the
output."

Because the agents CLI's contract is structured JSON, the tool
is a thin wrapper: shell out to `testbase agents ...`, parse
the JSON, hand the result to the agent.  No screen-scraping;
no format-drift risk.  The framework's commitment to structured
output is what makes the tool reliable.

Volume 7 covers the agent SDK side of the bridge.  This
chapter covers the framework side.  The two together are the
contract.

## What's next

- Chapter 7 — Mocking.  Tests that don't touch real systems
  produce JSOS records the agents CLI reads identically to
  real-system tests.
- Chapter 8 — `arcli`.  The decorator system that makes the
  three-line `@arcli.group("agents")` declaration work — and
  the validator-and-parser hardening from PR #13.
