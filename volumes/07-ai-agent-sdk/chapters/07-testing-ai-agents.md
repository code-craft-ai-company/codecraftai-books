# Chapter 7 — Testing AI Agents

> *Testing AI agents is unusual.  The output is non-
> deterministic at the LLM layer; the system under test
> includes a service that costs money to call; the right
> answer often depends on subjective evaluation.  The SDK's
> answer is **mock-first testing**: 305 test files, every
> LLM and tool mocked, no real API calls.  This chapter
> walks the discipline and what it catches.*

A reader who's tested traditional Python code is used to:

- Tests that produce deterministic output.
- Test runs that complete in seconds (no network calls).
- Test failures that point at a specific line.

AI-agent testing has different default constraints:

- LLM responses vary even with the same prompt
  (temperature, model versioning).
- Real LLM calls cost money and rate-limit.
- LLM "correctness" is often subjective.

The `codecraftai-agent-kit`'s answer: **don't test the LLM**.
Test *the SDK's behaviour around the LLM*.  The 305 test
files in the suite test the *agent's plumbing* — state
machine transitions, tool dispatch, memory operations,
workflow execution, observability emission.  None of them
hit a real LLM API.

This chapter walks the discipline.

## The numbers

Verified from the source:

- **305 test files** at
  `codecraftai-agent-kit/source/testroots/ccai/tests/singlehost/agentkit/`.
- **~78,155 LOC** of test code.
- **212 top-level test files** (mostly `test_*.py`).
- **72 property-based test files** (`test_*_properties.py` —
  chapter 8 walks these).
- **5 subdirectory layers**:
  - `builtins/` (5 files) — built-in tools.
  - `platform/` (14 files) — cloud-platform adapters.
  - `resilience/` (53 files) — circuit breakers, retry,
    degradation, dead-letter.
  - `steps/` — workflow step execution.
  - `transports/` — message transport.

The most recent verified test count: **1,964 / 1,964
PASSED** (commit `a2e437a`, Apr 27 2026).

## Mock-first — `ccai.testbase.mocking.Mock`

The SDK's tests use the framework's standard mocking
discipline (Volume 1 chapter 7).  Every external dependency
is mocked:

- **LLM providers** — `Mock(spec=LLMProvider)`.
- **Tools** — `Mock(spec=Tool)` or test-specific
  subclasses.
- **Memory stores** — `Mock(spec=MemoryStore)` or in-memory
  fakes.
- **Coordinators / external services** — mocked.

A representative pattern from
`test_contextsummarizer.py:16–46`:

```python
def _create_mock_llm_provider(summary_text="This is a summary."):
    mock_message = LLMMessage(role="assistant", content=summary_text)
    mock_response = Mock()
    mock_response.configure(recursive=True)
    mock_response.mock_context().set_attribute("message", mock_message)

    mock_provider = Mock()
    mock_provider.configure(recursive=True)
    mock_provider.mock_context().set_attribute("provider_name", "test-provider")
    mock_provider.mock_context().mock_method("complete", return_value=mock_response)
    return mock_provider


def _create_failing_llm_provider():
    def _raise_error(**kwargs):
        raise RuntimeError("LLM failure")
    mock_provider = Mock()
    mock_provider.configure(recursive=True)
    mock_provider.mock_context().set_attribute("provider_name", "failing-provider")
    mock_provider.mock_context().mock_method("complete", side_effect=_raise_error)
    return mock_provider
```

The pattern:

1. Construct a `Mock`.
2. `.configure(recursive=True)` to enable nested mock
   attribute access.
3. `.mock_context().set_attribute(name, value)` for
   properties.
4. `.mock_context().mock_method(name, return_value=...)`
   for return values.
5. `.mock_context().mock_method(name, side_effect=callable)`
   for exception simulation.

The mock pattern is consistent across all 305 test files.
A reader who learns it once reads any test in the suite.

## What gets unit-tested

Looking at the test file organisation, the test surface
covers:

### Lifecycle and state machine

`test_agentstatemachine.py` exercises:

- Every valid transition.
- Every invalid transition (asserts `SemanticError` raised).
- Concurrent transitions (thread-safety).
- Hook invocation order.

A typical test:

```python
def test_agent_state_machine_rejects_running_to_initialized():
    """
        Verify that RUNNING → INITIALIZED is not a valid transition.
    """
    sm = AgentStateMachine()
    sm.transition(LifecycleState.INITIALIZED)  # CREATED → INITIALIZED
    sm.transition(LifecycleState.RUNNING)       # INITIALIZED → RUNNING

    # RUNNING → INITIALIZED is not in VALID_TRANSITIONS.
    try:
        sm.transition(LifecycleState.INITIALIZED)
        rtnval = False
    except SemanticError:
        rtnval = True

    assert rtnval
    return
```

### Tool dispatch

`test_toolexecutor.py` exercises:

- Valid input / valid output → success.
- Invalid input → `ToolValidationError`.
- Tool's `execute()` raises → `ToolExecutionError`.
- Credentials resolved correctly.
- Default values applied.
- Schema validation errors aggregated correctly.

### LLM provider abstraction

`test_llmprovider_properties.py` exercises (a *property*
test — chapter 8 covers these):

- Constructing with valid provider / model name → succeeds.
- Constructing with empty / None name → raises `SemanticError`.
- Tags and config defaults applied.

### Memory operations

`test_filememorystore.py`, `test_memoryquery.py`,
`test_memoryindex.py` exercise:

- Storing entries and retrieving by ID.
- Searching by tags / metadata / scope.
- Semantic search (with mocked embedding provider).
- Cosine similarity correctness.
- Expiration policy enforcement.

### Workflow execution

`test_workflowexecutor.py` exercises:

- Linear workflows.
- Branching workflows (condition evaluation).
- Parallel branches.
- Checkpoint save / load / resume.
- Step retries on failure.
- Workflows with approval gates.

### Observability emission

`test_tracer.py`, `test_metricscollector.py`,
`test_executionrecorder.py` exercise:

- Spans nest correctly.
- Sampling decisions are stable (same trace_id → same
  decision).
- Metrics validate against the registry.
- Records have monotonic sequence numbers.

### Resilience subsystems

`test_circuitbreaker.py` exercises:

- CLOSED → OPEN → HALF_OPEN → CLOSED transitions.
- Failure thresholds.
- Recovery timeout enforcement.
- Concurrent calls preserve state correctness.

The 53 files under `resilience/` cover circuit breakers,
retry policies, degradation strategies, dead-letter
queues, and similar patterns.  Resilience is one-fifth of
the test surface — production agents *fail*, and the SDK
treats failure handling as load-bearing.

## What's *not* tested

A short list of things the test suite deliberately doesn't
exercise:

### Real LLM API calls

No tests in the SDK hit OpenAI, Anthropic, or any other
real provider.  All providers are mocked.

The reasoning:

- **Cost.**  A test suite that cost $50 per run discourages
  running it.
- **Rate limiting.**  CI runs that hit shared rate limits
  are non-deterministic.
- **Vendor variability.**  Provider responses change over
  time as models update; tests against real providers
  would have to track the changes.
- **Test scope.**  The SDK's job is to *correctly route
  the LLM's responses through its plumbing*, not to verify
  the LLM's responses themselves.

Application code that *uses* the SDK can write integration
tests against real providers if it wants to verify
end-to-end behaviour.  Those tests live in the
application's repo, not the SDK's.

### LLM output quality

The SDK doesn't verify that the LLM's responses are
correct, helpful, or well-formatted.  That's the
application's responsibility.

A test that wanted to verify "the agent gives sensible
answers about the weather" would have to:

1. Run the agent against a real LLM.
2. Score the response somehow (LLM-as-judge, human
   evaluation, regex matching).
3. Decide whether the score meets a bar.

None of those are in the SDK's scope.  The SDK's tests
verify "the agent dispatches tool calls correctly when
the LLM returns this specific tool-call response."  What
*makes* the LLM return that specific response is the
application's prompt-engineering concern.

### Cross-package integration

The agent-kit's tests *don't* import from `sysman`,
`landscape`, or `interop`.  Those packages have their own
test suites that integrate with agent-kit.

The integration is *one-directional*:

- `codecraftai-sysman-*` packages may use agent-kit
  (downstream).
- `codecraftai-landscaping` may use agent-kit.
- `codecraftai-interop` may use agent-kit.
- agent-kit doesn't depend on or test against any of them.

Agent-kit is a *library*; downstream packages test their
*use* of it.

## The 49-minute wall time

Volume 1 chapter 10's case study mentioned "49 minutes wall
time" for the agent-kit suite at
`TESTBASE_MAX_EXAMPLES_CAP=10`.  This figure refers to:

- Local execution (no CI workflow file in the repo).
- All 305 test files.
- Property tests at the cap (chapter 8).

There's no `.github/workflows` test runner.  The Makefile
contains only distribution targets.  The 49-minute figure
is from running the suite manually — likely the original
SDET's measurement on representative hardware.

A reader running the suite themselves should expect
similar time on similar hardware.  Faster machines (Apple
Silicon, modern Xeons) come in under 30 minutes.  Slower
machines (older laptops, constrained CI runners) take
longer.

The wall time is *not unusual for an SDK at this scale*.
3,744 individual test invocations × ~0.8 seconds average
≈ 50 minutes.  Most of that time is property tests
running their `max_examples=50` iterations.

## Test fixtures vs factory functions

Pytest users would expect `conftest.py` files with shared
fixtures.  The agent-kit's test suite uses a different
pattern: **factory functions per test module**.

```python
# Not a pytest fixture.
def _create_mock_llm_provider(summary_text="default"):
    # ... construct mock ...
    return mock_provider


# Used directly in test functions.
def test_summarizer_with_custom_text():
    provider = _create_mock_llm_provider(summary_text="custom")
    # ... test logic ...
```

Why factories instead of fixtures?

- **Visibility.**  A test that calls
  `_create_mock_llm_provider(summary_text="custom")` is
  *explicit* about what mock it's using.  A pytest
  fixture is invisible until you trace its definition.
- **Reuse boundary.**  Factories live in the test module;
  they're reused across tests in that module but don't
  leak into others.  Pytest fixtures span modules unless
  scoped carefully.
- **Independence from pytest.**  The agent-kit's tests run
  through testbase (Volume 1).  Pytest fixtures would
  couple the tests to pytest specifically; factories work
  with any test runner.

The framework's broader test-fixture story is in Volume 1
chapter 3 — `@testbase.attach.parameter` for test-scoped
resources.  The agent-kit uses factories *in addition* to
the framework's standard fixture pattern, depending on
whether the resource is per-test (factory) or
per-package (fixture).

## A representative test walkthrough

A worked test exercising several SDK pieces at once:

```python
import testbase

from ccai.testbase.mocking import Mock
from ccai.agentkit.contextsummarizer import ContextSummarizer
from ccai.agentkit.llmconversation import LLMConversation
from ccai.agentkit.llmmessage import LLMMessage


def _create_mock_llm_provider(summary_text="This is a summary."):
    mock_message = LLMMessage(role="assistant", content=summary_text)
    mock_response = Mock()
    mock_response.configure(recursive=True)
    mock_response.mock_context().set_attribute("message", mock_message)

    mock_provider = Mock()
    mock_provider.configure(recursive=True)
    mock_provider.mock_context().set_attribute("provider_name", "test-provider")
    mock_provider.mock_context().mock_method("complete", return_value=mock_response)
    return mock_provider


def test_summarize_messages_calls_llm_with_combined_content():
    """
        Verify that ContextSummarizer combines messages and calls the LLM.
    """
    provider = _create_mock_llm_provider(summary_text="The user asked for help.")
    summarizer = ContextSummarizer(provider=provider)

    messages = [
        LLMMessage(role="user", content="I need help with..."),
        LLMMessage(role="assistant", content="What kind of help?"),
        LLMMessage(role="user", content="Specifically..."),
    ]

    result = summarizer.summarize_messages(messages)

    # Verify the result is a system-role message.
    assert result.role == "system"
    assert result.content == "The user asked for help."

    # Verify the provider was called with a conversation that includes
    # the formatted messages.
    call = provider.mock_context().last_call("complete")
    sent_conversation = call.kwargs.get("conversation") or call.args[0]
    sent_content = sent_conversation.messages[-1].content
    assert "I need help with..." in sent_content
    assert "What kind of help?" in sent_content


def test_summarize_messages_raises_on_empty_input():
    """
        Verify that ContextSummarizer raises SemanticError on empty input.
    """
    provider = _create_mock_llm_provider()
    summarizer = ContextSummarizer(provider=provider)

    try:
        summarizer.summarize_messages([])
        rtnval = False
    except SemanticError:
        rtnval = True

    assert rtnval
```

Two tests covering one feature:

- The first verifies the *happy path* — summarize calls
  the LLM, returns the result.
- The second verifies the *guard* — empty input raises
  `SemanticError`.

The mock provider is a small fixture (factory function);
the rest of the test is straightforward setup → call → assert.

## What this chapter set up

A reader who's read this chapter should understand:

- The test suite's scale (305 files, 78k LOC).
- The mock-first discipline (no real LLM calls).
- What gets tested at each layer (lifecycle, tools, LLM
  abstraction, memory, workflows, observability,
  resilience).
- What deliberately doesn't get tested (real LLM calls,
  output quality, cross-package integration).
- The factory-function pattern instead of pytest fixtures.

The next chapter walks **property tests at scale** — the
72 property-based test files in the suite, what they
catch that example-based tests don't, and the
`@given(constraints={"max_examples": 50})` discipline that
keeps the suite tractable.

## What's next

Chapter 8 walks the property-test layer.  Examples from
`test_costtracker_properties.py`,
`test_tokenusage_properties.py`,
`test_llmconversation_properties.py`.  What property
tests buy when LLM behaviour drifts.  The recent migration
from `hypothesis` to `ccai.testbase.properties`
(commit `8f31798`).  How the suite balances coverage with
wall-time budget.
