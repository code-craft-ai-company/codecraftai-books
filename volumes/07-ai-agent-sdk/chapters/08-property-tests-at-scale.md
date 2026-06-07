# Chapter 8 — Property Tests at Scale

> *Of the 305 test files in the agent-kit suite, **72** are
> property-based.  They generate hundreds of input
> combinations per run, exercising round-trip serialisation,
> idempotency, accumulation, and ordering invariants.  At
> `max_examples=50`, that's ~250 cost-tracker iterations per
> run; full-sweep runs (cap unset) push tens of thousands of
> individual checks.  This chapter walks what they catch
> and how the SDK keeps them tractable.*

The agent-kit was an early adopter of property-based testing
in the framework.  Volume 1 chapter 4 introduced
`@given` and `TESTBASE_MAX_EXAMPLES_CAP` at the framework
level; the agent-kit *uses* those primitives heavily.

This chapter walks:

- The 72 property-based test files in the suite.
- What property tests catch that example-based tests
  don't.
- The recent migration from `hypothesis` to
  `ccai.testbase.properties` (commit `8f31798`).
- The `@given(constraints={"max_examples": 50})`
  discipline.
- A worked walkthrough of `test_costtracker_properties.py`.

## Why property-based testing for an AI SDK

Three reasons the agent-kit invests in property testing:

### Round-trip invariants

Many SDK objects need to *survive serialisation*.
`LLMConversation` round-trips through JSON.  `ToolResult`
round-trips through structured data.  `ExecutionRecord`
round-trips through the recorder's persistence layer.

A property test for round-trip:

```python
@given(messages=_llm_message_list, constraints={"max_examples": 50})
def test_llmconversation_json_roundtrip(messages: list) -> None:
    original = LLMConversation(messages=messages)
    json_str = original.to_json()
    restored = LLMConversation.from_json(json_str)
    assert original == restored
```

A bug where a special-case message body breaks
serialisation gets caught after a few dozen iterations.
Example-based tests would have to enumerate every special
case manually.

### Accumulation invariants

The `CostTracker` accumulates token usage across many
calls.  A property test verifies that the total *equals*
the sum of the per-call costs:

```python
@given(records=_record_list, constraints={"max_examples": 50})
def test_costtracker_total_token_usage_sums_all_records(records: list) -> None:
    tracker = CostTracker()
    for pname, token_usage, cost_estimate in records:
        tracker.record(pname, token_usage, cost_estimate)

    total = tracker.total_token_usage
    expected_prompt = sum(r[1].prompt_tokens for r in records)
    expected_completion = sum(r[1].completion_tokens for r in records)

    assert total.prompt_tokens == expected_prompt
    assert total.completion_tokens == expected_completion
```

The property is *invariant under composition*: regardless
of how many records (0 through 20 in the test) and what
their values are, the total *must* equal the sum.  A bug
that drops a record or double-counts gets caught.

### Ordering and monotonicity

The `ExecutionRecorder` assigns monotonically-increasing
sequence numbers.  A property test verifies the property
across any sequence of recorded events.  A bug that resets
the counter or skips a number gets caught.

Similarly, `LLMConversation`'s `truncate` method's
ordering invariants — system messages always preserved;
non-system messages truncated oldest-first — are
property-tested across any combination of role / count /
truncation limit.

## The 72 property-based test files

Located at
`source/testroots/ccai/tests/singlehost/agentkit/test_*_properties.py`.

A representative sample organised by what they test:

### Round-trip serialisation

- `test_llmconversation_properties.py` — JSON round-trip.
- `test_llmresponse_properties.py` — JSON round-trip.
- `test_messageserializer_properties.py` —
  serialisation/deserialisation symmetry.
- `test_executionrecord_properties.py` — record
  serialisation.

### Accumulation and aggregation

- `test_costtracker_properties.py` — five property
  functions on totals, breakdowns, reset.
- `test_tokenusage_properties.py` — construction
  validation, addition, total invariant.
- `test_metricssample_properties.py` — sample
  accumulation.

### Ordering and monotonicity

- `test_executionrecording_properties.py` — sequence
  number monotonicity.
- `test_workflowcheckpoint_properties.py` — checkpoint
  sequencing.
- `test_llmconversation_properties.py` (specifically the
  ordering property functions) — append preserves order.

### Validation invariants

- `test_toolschema_properties.py` — schema validation
  catches all type mismatches.
- `test_circuitbreaker_properties.py` — state transitions
  are deterministic given the configuration.

### Resilience patterns

- `test_retrypolicy_properties.py` — retry sequences
  exhibit expected backoff.
- `test_exponentialbackoff_properties.py` — backoff
  monotonically increases.

72 files total covering most of the SDK's data structures
and logic that has invariants.

## The migration from hypothesis to testbase

Commit `8f31798` (Mar 8 2026) moved the property tests from
the `hypothesis` library to
`ccai.testbase.properties` (Volume 1 chapter 4's
`@given` decorator).

The migration involved:

- **101+ test files modified** to use the new `@given(...)`
  syntax.
- **Strategy imports changed** —
  `from hypothesis.strategies import ...` →
  `from ccai.testbase.strategies import ...`.
- **Constraint syntax changed** — hypothesis's
  `@given(integers(...))` decorators replaced with
  `@given(value=_strategy, constraints={"max_examples":
  50})`.

The reasoning:

- **Reduce dependencies.**  The framework's broader story
  is a self-contained foundation tier.  Bringing a large
  external dependency (hypothesis is substantial) into
  the test path conflicts with that.
- **Cap-aware execution.**  testbase's `@given` integrates
  with `TESTBASE_MAX_EXAMPLES_CAP` (Volume 1 chapter 4).
  Hypothesis's runtime is independent; capping requires
  custom hypothesis machinery.
- **Reproducibility.**  testbase derives per-test seeds
  via `SHA-256(job_seed + ":" + test_name)` (Volume 1
  chapter 4 walked the derivation).  Hypothesis's
  per-test seed model differs and doesn't compose with
  the framework's run-level reproducibility.

The migration was *clean* — all tests passed in the
following commit (`a2e437a`'s "1,964 / 1,964 PASSED"
verification).

A reader looking at agent-kit tests sees the
post-migration shape:

```python
import testbase
from testbase.strategies import integers
from ccai.testbase.properties import given

@given(
    prompt_tokens=integers(min_value=0, max_value=100_000),
    completion_tokens=integers(min_value=0, max_value=100_000),
    constraints={"max_examples": 50},
)
def test_tokenusage_valid_construction(prompt_tokens: int, completion_tokens: int) -> None:
    # ...
```

## The `@given(constraints={"max_examples": 50})` discipline

Every property test in the agent-kit specifies
`max_examples`.  Most use `50`; some use `100`.  None
omit the constraint.

The reasoning:

- **Wall time budget.**  At `max_examples=50`, each
  property test runs 50 iterations.  72 property files ×
  ~5 properties per file × 50 iterations = 18,000
  iterations.  At ~50ms per iteration, that's about 15
  minutes — substantial but tractable.
- **CI vs full-sweep.**  Per Volume 1 chapter 4, the
  `TESTBASE_MAX_EXAMPLES_CAP` env var caps `max_examples`
  *down* to a smaller number (10 in CI runs).  At cap=10,
  the property suite shrinks to ~3,600 iterations — fast
  enough for per-PR runs.
- **Coverage at scale.**  Across many runs (each with
  fresh job_seeds), the cumulative coverage grows.  A
  bug that surfaces in 1 in 1000 inputs is caught after
  ~1000 cumulative iterations across ~20 CI runs.

The discipline trades *single-run depth* for
*aggregate coverage across many runs*.  Property-based
testing is most valuable when run *many times*; the cap
makes that feasible.

## A worked walkthrough — `test_costtracker_properties.py`

The file has *five property functions* covering the cost
tracker's invariants.

### Property 1 — Request count equals number of records

```python
@given(records=_record_list, constraints={"max_examples": 50})
def test_costtracker_request_count_equals_number_of_records(records: list) -> None:
    """
        For any sequence of records added to a CostTracker,
        request_count shall equal the number of records.
    """
    tracker = CostTracker()
    for pname, token_usage, cost_estimate in records:
        tracker.record(pname, token_usage, cost_estimate)
    assert tracker.request_count == len(records), (
        "request_count should be {}, got {}".format(
            len(records), tracker.request_count
        )
    )
```

Verifies that the count tracks the number of `record()`
calls.  Sounds trivial; would catch a bug where a specific
provider name causes the record to be silently skipped.

### Property 2 — Total token usage sums all records

```python
@given(records=_record_list, constraints={"max_examples": 50})
def test_costtracker_total_token_usage_sums_all_records(records: list) -> None:
    """
        For any sequence of records, total_token_usage equals
        the per-record sum.
    """
    tracker = CostTracker()
    for pname, token_usage, cost_estimate in records:
        tracker.record(pname, token_usage, cost_estimate)

    total = tracker.total_token_usage
    expected_prompt = sum(r[1].prompt_tokens for r in records)
    expected_completion = sum(r[1].completion_tokens for r in records)
    expected_total = expected_prompt + expected_completion

    assert total.prompt_tokens == expected_prompt
    assert total.completion_tokens == expected_completion
    assert total.total_tokens == expected_total
```

The accumulation invariant.

### Property 3 — Total cost estimate sums with floating-point tolerance

```python
@given(records=_record_list, constraints={"max_examples": 50})
def test_costtracker_total_cost_estimate_sums_all_records(records: list) -> None:
    tracker = CostTracker()
    for pname, token_usage, cost_estimate in records:
        tracker.record(pname, token_usage, cost_estimate)

    total = tracker.total_cost_estimate
    expected_prompt = sum(r[2].prompt_cost for r in records)
    expected_completion = sum(r[2].completion_cost for r in records)
    expected_total = expected_prompt + expected_completion

    # Float comparison with tolerance.
    assert abs(total.prompt_cost - expected_prompt) < 1e-6
    assert abs(total.completion_cost - expected_completion) < 1e-6
    assert abs(total.total_cost - expected_total) < 1e-6
```

Same as Property 2 but for the cost.  The tolerance is
*explicit* — floats accumulate rounding errors over many
additions.

### Property 4 — Per-provider breakdown groups by provider

```python
@given(records=_record_list, constraints={"max_examples": 50})
def test_costtracker_per_provider_breakdown_groups_by_provider(records: list) -> None:
    tracker = CostTracker()
    for pname, token_usage, cost_estimate in records:
        tracker.record(pname, token_usage, cost_estimate)

    breakdown = tracker.per_provider_breakdown()

    # Group records by provider for the expected breakdown.
    expected = {}
    for pname, token_usage, cost_estimate in records:
        if pname not in expected:
            expected[pname] = {
                "prompt_tokens": 0, "completion_tokens": 0,
                "prompt_cost": 0.0, "completion_cost": 0.0,
            }
        expected[pname]["prompt_tokens"] += token_usage.prompt_tokens
        # ... etc ...

    # Verify each provider's breakdown.
    for pname, expected_data in expected.items():
        actual = breakdown[pname]
        assert actual["token_usage"].prompt_tokens == expected_data["prompt_tokens"]
        # ... etc ...
```

The grouping invariant.  Catches bugs where the breakdown
mis-attributes records to the wrong provider.

### Property 5 — Reset clears all records

```python
@given(records=_record_list, constraints={"max_examples": 50})
def test_costtracker_reset_clears_all_records(records: list) -> None:
    tracker = CostTracker()
    for pname, token_usage, cost_estimate in records:
        tracker.record(pname, token_usage, cost_estimate)

    tracker.reset()

    assert tracker.request_count == 0
    assert tracker.total_token_usage.total_tokens == 0
    assert tracker.total_cost_estimate.total_cost == 0.0
```

The idempotency / reset invariant.

Five properties, each running 50 iterations per run = 250
property checks per run *just for the cost tracker*.  Over
the suite's 72 property files, the per-run total is in
the thousands.

## Strategies — generating realistic inputs

The properties use *strategies* to generate input data:

```python
_non_negative_int = st.integers(min_value=0, max_value=100_000)
_non_negative_float = st.floats(min_value=0.0, max_value=10_000.0)
_provider_name = st.text(alphabet="abcdefghijklmnopqrstuvwxyz", min_size=1, max_size=10)


def _record_entry():
    return st.tuples(
        _provider_name,
        st.builds(
            TokenUsage,
            prompt_tokens=_non_negative_int,
            completion_tokens=_non_negative_int,
            total_tokens=st.builds(
                lambda p, c: p + c,
                _non_negative_int,
                _non_negative_int,
            ),
        ),
        # ... CostEstimate similarly ...
    )


_record_list = st.lists(_record_entry(), min_size=0, max_size=20)
```

Strategies compose:

- Primitive strategies (`integers`, `floats`, `text`).
- Composite strategies (`tuples`, `lists`, `builds`).
- Constrained strategies (`min_value`, `max_size`,
  alphabet restrictions).

A test that uses `_record_list` gets generated input that's
*realistic* — non-negative numbers, short provider names,
variable list lengths from 0 to 20.  Hypothesis's
shrinking machinery (preserved in testbase's strategies)
finds *minimal failing examples* when a property is
violated.

## What property tests catch that example-based tests don't

Three classes of bugs property testing surfaces:

### Edge cases the test author didn't think of

The most common.  Property tests generate inputs across
the entire input space the strategy describes.  An empty
list, a list of zeros, a list with a single
maximum-magnitude element, a list with all-identical
elements — each is a different shape the strategy
naturally produces.  Bugs that manifest only on specific
shapes get caught.

### Floating-point and rounding

The `1e-6` tolerance in Property 3 above is there because
*floating-point summation has order-dependent rounding*.
Adding 1000 values left-to-right gives a different result
than adding them right-to-left.  A naive total computed
once can drift from the per-call sum.  Property 3
*verifies* the drift is bounded.

### Composition correctness

Property 4 verifies the per-provider breakdown across
varying numbers of providers and records.  An example-
based test would have to manually construct the test
matrix; the property test generates it.

## Full-sweep runs

The volume's outline mentions "415+ property tests" and
"21,000 individual checks at full sweep."  The numbers
correspond to:

- ~415 property *functions* across the 72 property files.
- At `max_examples=50` (the in-suite default), that's
  ~20,750 individual `@given` iterations per run.
- The "full sweep" = running with the cap unset.  Most
  property tests use `max_examples=50` literally; without
  the framework's cap reducing it.

A representative full-sweep figure mentioned in the volume
outline: *"13,054 passing checks before completion across
415 property tests, 11+ hours, 0 failures."*  The 11-hour
figure is *with* deeper exploration.

The cap exists because 11 hours is too long for CI.  The
50-iteration cap completes in tens of minutes; 10-iteration
caps (used in tightest CI) complete in single-digit
minutes.

## What this chapter set up

By the end:

- A reader knows the SDK has 72 property-based test files.
- The migration from `hypothesis` to `testbase.properties`
  is acknowledged as a clean transition.
- The `max_examples` discipline (50 in the suite, 10 in
  the CI cap) is the framework's wall-time-vs-coverage
  trade-off.
- The five `test_costtracker_properties.py` properties are
  a concrete worked example.
- Three classes of bugs property testing catches that
  example-based testing misses.

The next chapter — the volume's last — is the worked
*case study* of an agent that uses the rest of the
portfolio.

## What's next

Chapter 9 closes the volume with a worked example of an
agent that runs sysman features (Volume 4) against a
landscape (Volume 6) through interop protocols (Volume 5),
observed through this volume's machinery, with full event
recording for time-travel debugging (Volume 8).  The
chapter is the volume's payoff — every subsystem the SDK
exposes lands in the case study.
