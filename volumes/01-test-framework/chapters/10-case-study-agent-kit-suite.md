# Chapter 10 — Case Study: A 3,700-Test Suite That Runs on One Host

> *Chapters 1 through 9 walked the framework one concept at a
> time.  This chapter watches the framework run.  The subject
> is the `codecraftai-agent-kit` single-host suite — 292 test
> files, more than 3,700 test invocations, finishing under an
> hour with `TESTBASE_MAX_EXAMPLES_CAP=10`.  Every concept from
> the previous chapters lands in the same run.*

A book chapter on test framework architecture is only as
honest as the test suite it can point at.  This chapter
points at one: `codecraftai-agent-kit`, the test root at
`source/testroots/ccai/tests/singlehost/`, used as the
reference single-host suite for the portfolio.

The suite predates this book.  It was written by the same
SDETs who wrote testbase, against the framework testbase
itself.  It is the place in the portfolio where the
framework's design assumptions get exercised at scale —
where "thousands of tests" stops being an aspirational claim
and becomes a measurement.

## The shape of the suite

```
codecraftai-agent-kit/source/testroots/ccai/tests/singlehost/
├── agentkit/
│   ├── test_costtracker_properties.py
│   ├── test_costtracker_observability.py
│   ├── test_invocation_log.py
│   ├── test_memory_management.py
│   └── ... (dozens more)
├── tooling/
│   ├── test_tool_registry.py
│   ├── test_tool_dispatch.py
│   └── ...
├── observability/
│   └── ...
└── ...
```

By the numbers (counted at the time of writing this volume):

- **Test files:** 292 (across the `singlehost/` root).
- **`def test_` declarations:** 308.
- **Test invocations:** more than 3,700 (a single property-
  based test with `max_examples=10` becomes 10
  invocations; with `max_examples=50` it becomes 50; the
  total across the suite is the sum).
- **Wall time at `TESTBASE_MAX_EXAMPLES_CAP=10`:** ~49
  minutes on a single CI runner.
- **Wall time at full sweep (cap unset):** several hours.

The 3,700-invocation count is the one that matters for the
case study: the suite is *worth* running because it covers
that many distinct executions; the cap makes it *runnable*
in a CI window.

## Why this suite is the right case study

`codecraftai-agent-kit` is the agent SDK from Volume 7 — the
Python toolkit for building production AI agents on top of
the foundation tier.  The single-host test root exercises:

- **Cost tracking** (token counting, budgeting).
- **Tool dispatch** (mapping tool calls to handlers).
- **Memory management** (long-conversation truncation).
- **Observability hooks** (events, traces).
- **Configuration loading** (Volume 0 chapter 9 in
  practice).
- **Error vocabulary** (Volume 0 chapter 3 — the agent
  raises typed errors the host application can catch).

Every chapter of Volume 1 lands somewhere in this suite.
Property-based testing (Chapter 4) is heavily used in the
cost-tracker tests.  The JSOS stream (Chapter 5) is the
suite's output.  The agents CLI (Chapter 6) is the way the
suite's results are read.  Mocks (Chapter 7) stand in for
the model-call boundary.  arcli (Chapter 8) drives the
test runner.  The landscape boundary (Chapter 9) is
explicitly *not* invoked — the suite runs on one host with
mock couplings.

## Run anatomy: the wall-clock breakdown

A representative full run, instrumented with phase markers
in the JSOS stream, breaks down approximately as:

| Phase | Duration | Share |
|---|---|---|
| Discovery | ~12 s | <1% |
| Collection | ~3 s | <1% |
| Graph | ~1 s | <1% |
| Traversal — example-based tests | ~9 min | ~18% |
| Traversal — property-based tests (cap=10) | ~38 min | ~78% |
| Diagnostic capture (pre / post) | ~2 min | ~4% |
| Finalize | ~5 s | <1% |
| **Total** | **~49 min** | **100%** |

The numbers are not surprising — the property-based tests
dominate.  At cap=10, each property test runs 10 iterations;
at the full Hypothesis-default of 100 the same tests would
run 10× longer, pushing the suite north of 7 hours.

This is the budget Chapter 4 promised.  The cap is the knob.

## Property-based tests in the wild

A representative cost-tracker property test:

```python
# codecraftai-agent-kit/source/testroots/ccai/tests/singlehost/
# agentkit/test_costtracker_properties.py

import testbase
from testbase.strategies import integers
from ccai.agentkit.costtracker import CostTracker

@testbase.given(
    input_tokens=integers(min_value=0, max_value=10**6),
    output_tokens=integers(min_value=0, max_value=10**6),
    constraints={"max_examples": 50},
)
def test_total_cost_is_input_plus_output_for_any_token_counts(
    input_tokens: int,
    output_tokens: int
) -> None:
    """
        Verify that total cost equals input cost plus output cost
        across the full range of token counts.
    """
    tracker = CostTracker(model="claude-opus-4-7")
    tracker.record_input(input_tokens)
    tracker.record_output(output_tokens)

    expected = (
        tracker.input_unit_cost * input_tokens
        + tracker.output_unit_cost * output_tokens
    )
    actual = tracker.total_cost()
    rtnval = (actual == expected)
    assert rtnval
    return
```

Three observations:

1. The strategy is `integers(0, 10**6)` for both tokens —
   covers realistic and adversarial inputs.
2. `max_examples=50` is what the test author wrote.  At
   `TESTBASE_MAX_EXAMPLES_CAP=10` the runner runs 10.  At
   `TESTBASE_MAX_EXAMPLES_CAP=200` the runner runs 50 (the
   cap is an upper bound, not a multiplier).
3. The test follows Volume 0 chapter 2's discipline:
   typed parameters, explicit `return`, `rtnval` named
   intermediate, no compound `return` statements.

A run with cap=10 generates 10 (input_tokens, output_tokens)
pairs from the SHA-256-derived per-test seed (Chapter 4).
The same job seed reproduces the same 10 pairs.  A failure
is reproducible end-to-end without re-running the suite.

## Mocking the model-call boundary

The agent SDK's central concern is talking to language
models.  None of the single-host tests actually call a
model.  Instead, the test root's factory module yields a
mock:

```python
# codecraftai-agent-kit/source/packages/ccai/agentkit/factories.py

@testbase.factories.resource_factory(scope=ResourceLifespan.Test)
def mock_anthropic_client():
    """
        Yields a Test-scoped mock of the Anthropic client for
        single-host tests.
    """
    client = Mock(spec=AnthropicClient, mock_type=MockType.Instance)
    client.messages.create = SideEffect(
        lambda **kwargs: build_canned_response(**kwargs)
    )
    yield client
    return
```

The `build_canned_response` function returns a deterministic
response shape for any prompt.  The test does not depend on
the language model's actual output — it depends on the
*shape* of the response, which the SDK then parses.

Tests of *the parser* attach `mock_anthropic_client`; tests
of *the cost tracker* attach a different mock that yields
specific token counts; tests of *the tool dispatcher* mock
the tool callable.  Each mock is local to its test domain;
none of them pretends to be the full model.

## What the suite proves about the framework

A test suite of this size, finishing in under an hour, is a
specific claim about the framework: that the framework's
overhead per test is *tiny*.  Working back from the wall
times:

- Property-based: 38 min / ~3,000 invocations ≈ 760 ms per
  invocation — most of which is the test body itself, not
  the runner.
- Example-based: 9 min / ~700 tests ≈ 770 ms per test —
  similar.

The runner overhead per invocation is on the order of
single-digit milliseconds.  This is the cost of one JSOS
record write, one resource-factory generator step, one
sequencer dispatch.  The test framework's *plumbing* is
cheap; the test bodies themselves are what take the time.

This is by design.  Chapter 5's commitment to flushing
after every JSOS record was a per-record cost the framework
deliberately accepts; the case study shows that even at
3,700 invocations the cost is in the noise.

## Reading the run with the agents CLI

After the suite finishes, the standard inspection pattern:

```bash
# Did anything fail?
testbase agents summary extract --results results/20260430-153012/
# Output: {"passed": 3680, "failed": 18, "errored": 4, "skipped": 10, ...}

# What failed?
testbase agents testresult failures --results results/20260430-153012/
# Output: {"failures": [...18 entries...]}

# How was the run configured?
testbase agents config extract --config-file results/20260430-153012/startup-configuration.json
# Output: {"testroot": "...", "max_examples_cap": 10, "job_seed": "...", ...}
```

The three commands are the three audiences from Chapter 6.
A CI dashboard reads the summary; an engineer (or AI agent)
reads the failures; a build engineer auditing a flaky run
reads the config extract to learn the seed and the cap that
were in effect.

A specific use of the config-extract output: the build
engineer can re-run the suite *with the same job seed* —
making property-based runs reproducible across machines.
The seed is in the JSON; pasting it into the next run's
invocation reproduces every per-test seed via the
SHA-256 derivation (Chapter 4).

## What the cap does to coverage

The cap reduces the number of generated examples per
property test.  Does it reduce coverage?

In the strict sense, yes — fewer inputs are tested.  In the
practical sense, less than expected, for two reasons:

1. **The strategies are designed to find edge cases
   quickly.**  `integers(0, 10**6)` doesn't generate uniform
   random integers; it biases toward boundary values (0,
   1, the maximum) and around the midpoint.  Most failures
   surface in the first few examples regardless of cap.
2. **The job seed varies across runs.**  A property test
   with cap=10 sees different inputs in tomorrow's CI run
   than in today's, because the job seed is fresh.  Over
   many runs the suite covers a much wider input space than
   any single run.

The trade-off is "depth in any single run" vs. "breadth
across many runs."  The portfolio chooses breadth: cap=10
in CI on every PR; full-sweep cap-unset in nightly.  An
adversarial input that surfaces in nightly's run gets
captured (with its job seed) and replayed in regression
tests; the next day's CI runs include it as a deterministic
example.

This pattern — "find adversarial inputs at full sweep, lock
them as regression tests" — is the loop that makes
property-based testing economical.  The cap makes the loop
fast.

## What the case study does *not* show

A few things this case study deliberately doesn't
demonstrate, because they belong to other volumes:

- **Real model calls.**  Volume 7 (the agent SDK volume)
  walks tests that actually call a language model.  Those
  tests run as a *separate* (smaller, slower) integration
  suite.  Volume 1's case study is the unit-and-property
  suite that runs on every PR.
- **Distributed execution.**  Volume 2's distributed-test
  case study runs on a multi-host landscape.  The single-
  host suite here is the substrate; the distributed suite
  extends it.
- **CI integration.**  Volume 3 walks the orchestration
  layer that invokes a testbase suite from a taskplus
  workflow.  The case study here treats testbase as
  invoked directly; Volume 3 wraps it.

This volume's job is to make the framework legible.  The
case study is a demonstration of legibility at scale.

## The shape of a successful run

What the engineer sees on a green run, abbreviated:

```text
$ testbase run --testroot source/testroots/ccai/tests/singlehost/ \
              --max-examples-cap 10

[2026-04-30 15:30:12] Phase: Discovery (started)
[2026-04-30 15:30:24] Phase: Discovery (finished, 292 modules, 308 tests)
[2026-04-30 15:30:25] Phase: Collection (started)
[2026-04-30 15:30:28] Phase: Collection (finished, 47 originators)
[2026-04-30 15:30:28] Phase: Graph (started)
[2026-04-30 15:30:29] Phase: Graph (finished, 354 nodes, 802 edges)
[2026-04-30 15:30:29] fire_attach_to_environment
[2026-04-30 15:30:31] fire_attach_to_framework
[2026-04-30 15:30:32] Phase: Traversal (started)
[2026-04-30 15:30:33] PASS test_addition_is_commutative (17 ms)
[2026-04-30 15:30:33] PASS test_subtraction_is_anti_commutative (12 ms)
[2026-04-30 15:30:34] PASS test_total_cost_is_input_plus_output... (10 examples, 84 ms)
... (3,700+ more lines) ...
[2026-04-30 16:19:38] Phase: Traversal (finished)
[2026-04-30 16:19:39] fire_diagnostic_capture_post_testrun
[2026-04-30 16:19:42] Run complete.

Summary: 3680 passed, 18 failed, 4 errored, 10 skipped (49m 27s)
Stream:  results/20260430-153012/testresults_stream.jsos
```

Every line traces back to a chapter.  Discovery: Chapter 2.
Collection / Graph: Chapter 3.  `fire_attach_*`: Chapter 9.
PASS / FAIL counts: Chapter 5.  The wall-time figure: this
chapter.

## What this volume gives the reader

By the end of the case study a reader who started Volume 1
cold should be able to:

- Read a testbase test file and identify each construct's
  framework concept.
- Choose a sensible `TESTBASE_MAX_EXAMPLES_CAP` for their
  CI and full-sweep runs.
- Use the `agents` CLI to extract failures, summaries, and
  configuration from a results directory.
- Tell when a test should mock its dependency vs. depend on
  a real coupling.
- Tell which tests in a suite cross the single-host /
  distributed boundary.
- Read an `arcli`-decorated CLI's source and predict its
  behaviour from the decorators alone.

That fluency is the substrate Volume 2 builds on.  Volume 2
extends `TestJob` to `InteropLibraryTestJob` to
`DistributedTestJob`.  The lifecycle does not change; the
hooks do.  Volume 2 picks up exactly where this case study
ends — with the same suite running across multiple hosts,
the same JSOS streaming format, the same agents CLI
output, but with the `fire_*` hooks doing landscape
activation instead of nothing.

## End of Volume 1

The volume started by asking what's missing from `unittest`
and `pytest` for automation testing (Chapter 1) and ended
by watching a 3,700-test suite finish in under an hour
because the framework filled the gaps (this chapter).

The chapter that comes next is in another volume.  Volume 2
opens with "Why distributed automation is genuinely hard"
and then walks the inheritance chain that extends what this
volume built.
