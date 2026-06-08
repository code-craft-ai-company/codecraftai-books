# Chapter 6 — Observability for Agents

> *Production agents fail in unfamiliar ways — the LLM
> hallucinates a tool call, the workflow takes an
> unexpected branch, the memory store returns stale data.
> Diagnosing these requires *structured observability* the
> SDK ships out of the box.  Tracer, MetricsCollector, 13
> built-in metrics, ExecutionRecorder for full replay.
> This chapter walks each.*

The SDK ships an observability stack with three concerns:

- **Tracing** — hierarchical spans capturing causality
  across the agent's execution.
- **Metrics** — counters, gauges, and histograms recording
  rates and durations.
- **Recording** — full replay-ready event capture for
  post-hoc analysis.

Each concern is independent.  Each is integrated with the
agent's lifecycle.  All three emit *structured* data the
application's observability backend (OpenTelemetry,
Datadog, Prometheus, etc.) consumes.

## Tracing — `Tracer` and `TraceSpan`

**File:** `tracer.py:22–100`

```python
class Tracer:
    """
        A component that creates and manages TraceSpan objects
        for an Agent.  Implements probabilistic sampling at the
        root span level, where non-sampled spans propagate
        context but are not exported.
    """

    def __init__(self, agent_id: str, trace_context: TraceContext = None,
                 sampling_rate: float = 1.0) -> None:
```

The tracer is *per-agent*.  Each agent owns one.  It
manages a stack of `TraceSpan` objects representing the
agent's call hierarchy.

**File:** `tracespan.py:22–40`

```python
class TraceSpan:
    """
        A mutable object representing a single unit of work
        within a distributed trace.

        status must be one of: ("unset", "ok", "error")
    """
```

A span has:

- `trace_id` and `span_id` (16-byte and 8-byte hex strings).
- `parent_span_id` (when nested).
- `name` — human-readable label.
- `start_time` and `end_time`.
- `status` — `"unset"`, `"ok"`, `"error"`.
- `attributes` — dict of structured metadata.
- `events` — list of `TraceEvent` objects attached to
  the span.

Spans are *mutable* during their lifetime — the application
adds attributes and events as work progresses.  When the
span closes, its data is exported to whatever exporter the
application registered.

### Hierarchical spans

A typical agent run produces a *tree* of spans:

```
agent.run                                    [root span]
├── agent.lifecycle.transition (CREATED → INITIALIZED)
├── agent.lifecycle.transition (INITIALIZED → RUNNING)
└── agent.workflow.execute
    ├── workflow.step.execute (search)
    │   └── llm.complete (gpt-4o)
    │       └── tool.execute (web_search)
    ├── workflow.step.execute (summarise)
    │   └── llm.complete (gpt-4o)
    └── workflow.step.execute (finalise)
        └── llm.complete (gpt-4o)
```

Each child span declares its parent.  The exporter
reconstructs the hierarchy.  A debugger viewing the trace
sees the full causality.

### Probabilistic sampling

The `sampling_rate` parameter controls how many *root*
spans get exported.  At `1.0`, every span is exported.  At
`0.1`, ten percent of root spans (and all their children).
At `0.0`, no spans (but parent context still propagates so
non-sampled traces can still be cross-referenced if a
descendant span is sampled by another component).

Sampling matters at scale.  An agent serving millions of
requests can't trace every one; sampling reduces the cost
of observability while still providing statistical
visibility.

## Metrics — `MetricsCollector` and the 13 built-ins

**File:** `metricscollector.py:21–74`

```python
class MetricsCollector:
    """
        Collects metric values and routes them to registered
        exporters.  Validates metric names against the
        MetricsRegistry to prevent typos / missing definitions.
    """
```

The collector is *per-agent*.  It receives metric
recordings (counter increments, gauge updates, histogram
observations) and routes them to exporters.

Three metric kinds:

- **Counter** — monotonically increasing.  "How many tool
  errors have occurred?"  `inc(name, value=1, labels=...)`.
- **Gauge** — current value.  "How many spans are open?"
  `set(name, value, labels=...)`.
- **Histogram** — distribution of values.  "How long do
  tool calls take?"  `observe(name, value, labels=...)`.

Every recording is validated against the
`MetricsRegistry` — calling `inc("agent.tools.errors")`
when no metric with that name is registered raises
`MetricNotRegisteredError`.  Typos surface as test
failures, not silently-dropped data.

### The 13 built-in agent metrics

**File:** `builtinagentmetrics.py:16–114`

```python
BUILTIN_METRICS = [
    MetricDefinition(name="agent.lifecycle.transitions", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.tool.execution.duration", metric_type=MetricType.HISTOGRAM),
    MetricDefinition(name="agent.tool.execution.errors", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.llm.request.duration", metric_type=MetricType.HISTOGRAM),
    MetricDefinition(name="agent.llm.token.usage.prompt", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.llm.token.usage.completion", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.llm.request.errors", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.memory.operations", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.message.sent", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.message.received", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.workflow.step.duration", metric_type=MetricType.HISTOGRAM),
    MetricDefinition(name="agent.workflow.step.errors", metric_type=MetricType.COUNTER),
    MetricDefinition(name="agent.active.spans", metric_type=MetricType.GAUGE),
]
```

Thirteen metrics covering the agent's main concerns:

| # | Metric | Type | Purpose |
|---|---|---|---|
| 1 | `agent.lifecycle.transitions` | counter | Count state-machine transitions. |
| 2 | `agent.tool.execution.duration` | histogram | Tool-call latency distribution. |
| 3 | `agent.tool.execution.errors` | counter | Tool failures. |
| 4 | `agent.llm.request.duration` | histogram | LLM-call latency distribution. |
| 5 | `agent.llm.token.usage.prompt` | counter | Cumulative prompt-token usage. |
| 6 | `agent.llm.token.usage.completion` | counter | Cumulative completion-token usage. |
| 7 | `agent.llm.request.errors` | counter | LLM failures. |
| 8 | `agent.memory.operations` | counter | Memory read/write/search count. |
| 9 | `agent.message.sent` | counter | Inter-agent messages sent. |
| 10 | `agent.message.received` | counter | Inter-agent messages received. |
| 11 | `agent.workflow.step.duration` | histogram | Workflow-step latency. |
| 12 | `agent.workflow.step.errors` | counter | Workflow-step failures. |
| 13 | `agent.active.spans` | gauge | Live span count. |

These thirteen are emitted *automatically* by the SDK's
internal hooks.  Application code that wants *additional*
metrics defines them in the registry and emits them.

### Labels for dimensionality

Most metrics support *labels* — key/value pairs that
partition the metric:

```python
collector.inc(
    "agent.tool.execution.errors",
    labels={"tool_name": "fetch_weather", "error_type": "ValidationError"},
)
```

The exporter aggregates by label combinations.  A
dashboard can show "errors per tool name" or "errors per
error type" or both.

The agent's *agent_id* is included as a default label on
every metric, so multi-agent processes can attribute
metrics to specific agents.

## Recording — `ExecutionRecorder` and `ExecutionReplayer`

The most powerful piece of the observability stack —
*full replay* of an agent run.

### `ExecutionRecorder`

**File:** `executionrecorder.py:17–70`

```python
class ExecutionRecorder:
    """
        A component that captures ExecutionRecord objects
        during agent execution, storing them in an ordered
        sequence for later replay and inspection.

        Sequence numbers are monotonically increasing starting
        from 1.
    """

    def __init__(self, agent_id: str) -> None:
        self._records = []
        self._sequence_counter = 0
        self._is_recording = False
```

The recorder is the agent's *event stream sink*.  Every
significant event during the agent's run is captured as
an `ExecutionRecord`:

```python
class ExecutionRecord:
    """
        :param sequence_number: Monotonically increasing starting from 1.
        :param event_type: e.g. 'lifecycle.transition', 'llm.request.completed',
                           'tool.execution.failed'.
        :param correlation_id: Span ID, request ID, or other correlation key.
        :param payload: Event-specific structured data.
        :param timestamp: When the event happened.
    """
```

Recording is *opt-in*.  The application calls
`recorder.start()` before the agent runs and
`recorder.stop()` after.  Between, every event flows in.

### `ExecutionReplayer`

**File:** `executionreplayer.py:20–57`

```python
class ExecutionReplayer:
    """
        A component that reads an ExecutionRecording and allows
        step-by-step inspection of the recorded events.

        Supports forward and backward traversal, reset, seek
        by sequence number, and filtering by event type.
    """
```

The replayer reads a recording and lets the application
walk through it:

```python
replayer = ExecutionReplayer(recording=loaded_recording)

# Step forward through events.
while replayer.has_next:
    record = replayer.next()
    print(f"[{record.sequence_number}] {record.event_type}: {record.payload}")

# Or seek to a specific point.
replayer.seek_to(sequence_number=42)

# Or filter to specific event types.
errors = list(replayer.filter(event_types=["tool.execution.failed", "llm.request.errors"]))
```

This is the foundation for **time-travel debugging**
(Volume 8's subject).  An agent run that produced an
unexpected outcome can be *replayed* to walk through what
happened, when each event fired, what data flowed through.

### What gets recorded

The SDK emits records for:

- Lifecycle transitions (CREATED → INITIALIZED, etc.).
- LLM requests (started / completed / failed).
- Tool executions (started / completed / failed).
- Memory operations.
- Workflow step boundaries.
- Approval gate events.
- Circuit breaker state transitions.
- Custom events the application emits.

The *type vocabulary* is open — the application can record
any event type it cares about.  The SDK's events use
namespaced names (`lifecycle.transition`,
`llm.request.completed`) so application events use
different namespaces (`my-app.business-event`).

## Cost tracking — the agent's bill

A reader who's been counting subsystems will notice cost
tracking belongs in this chapter.  The `CostTracker` is
*observability for cost*.

**File:** `costtracker.py:14–104`

```python
class CostTracker:
    """
        Accumulates TokenUsage and CostEstimate records across
        multiple LLM requests, providing totals and per-provider
        breakdowns.
    """

    def record(self, provider_name: str, token_usage: TokenUsage,
               cost_estimate: CostEstimate) -> None:
        # ...

    @property
    def total_token_usage(self) -> TokenUsage:
        # ...

    @property
    def total_cost_estimate(self) -> CostEstimate:
        # ...

    def per_provider_breakdown(self) -> dict:
        # ...
```

Every LLM call's tokens and cost flow through the
tracker.  The provider implementation does the recording
(chapter 2 walked the integration).  At any time, the
application can query:

```python
tracker = agent.cost_tracker

print(tracker.total_token_usage)
# TokenUsage(prompt_tokens=15234, completion_tokens=4127, total_tokens=19361)

print(tracker.total_cost_estimate)
# CostEstimate(prompt_cost=0.46, completion_cost=0.83, total_cost=1.29, currency='USD')

for provider, data in tracker.per_provider_breakdown().items():
    print(f"{provider}: {data['cost_estimate'].total_cost}")
# openai-gpt-4o: 0.81
# anthropic-claude-opus-4-7: 0.48
```

### `TokenUsage` validation

**File:** `tokenusage.py:12–36`

```python
class TokenUsage:
    def __init__(self, prompt_tokens: int, completion_tokens: int, total_tokens: int) -> None:
        expected_total = prompt_tokens + completion_tokens
        if total_tokens != expected_total:
            raise SemanticError(
                "TokenUsage total_tokens must equal sum of prompt + completion"
            )
        # ...
```

`TokenUsage` is *immutable* and *validated* at construction.
The total must equal the sum.  This catches provider
implementations that report inconsistent token counts.

### `CostEstimate` validation

**File:** `costestimate.py:13–42`

```python
class CostEstimate:
    def __init__(self, prompt_cost: float, completion_cost: float,
                 total_cost: float, currency: str = "USD") -> None:
        expected_total = prompt_cost + completion_cost
        if abs(total_cost - expected_total) > 1e-9:
            raise SemanticError(
                "CostEstimate total_cost must equal sum of prompt + completion"
            )
        # ...
```

Same pattern, with floating-point tolerance (1e-9).
Default currency is USD; provider implementations that
charge in other currencies pass `currency="..."`.

The validation discipline catches calculation bugs in the
provider's cost-computation logic.  An OpenAIProvider that
miscomputes prompt_cost would fail to construct a
`CostEstimate`; the test suite would catch the bug
before it reached production.

## Composing the observability stack

A typical agent's observability setup:

```python
class MyAgent(Agent):

    def initialize(self) -> None:
        super().initialize()

        # Tracer with OTLP exporter
        self._tracer.register_exporter(OTLPTraceExporter(endpoint="..."))

        # Metrics with Prometheus exporter
        self._metrics_collector.register_exporter(
            PrometheusMetricsExporter(port=9090)
        )

        # Recorder writing to a file
        self._execution_recorder = ExecutionRecorder(agent_id=self.identity.identifier)
        self._execution_recorder.register_sink(
            FileExecutionSink(path="recordings/")
        )
        self._execution_recorder.start()

    def stop(self) -> None:
        super().stop()
        self._execution_recorder.stop()
```

Three subsystems, three exporters / sinks, all
configured at agent initialisation.  The SDK's events flow
through automatically; the agent's specific events go
through the same pipeline.

## What the observability stack *doesn't* do

A few patterns the SDK leaves to the application:

### No automatic exporters

The SDK ships abstract `TraceExporter` and
`MetricsExporter` base classes but no concrete exporters.
The application provides exporters from a separate
package (e.g., `codecraftai-agentkit-otlp` for
OpenTelemetry, `codecraftai-agentkit-prometheus` for
Prometheus).

This is intentional separation.  Pinning specific
backend SDKs in the agent-kit would make every consumer
pay the dependency cost.  Adapter packages let the
application choose.

### No PII redaction

Recordings, traces, and metrics may contain user content.
The SDK doesn't redact — it records faithfully.  An
application that needs to redact (per GDPR, HIPAA, or
similar) wraps the recorder / tracer / metrics with
filters that strip sensitive fields.

The framing is honest: redaction is *application-specific*
and depends on the data's classification.  A general-purpose
SDK can't decide which fields are sensitive.

### No retention policies

The recorder stores events; retention (how long, how much)
is the application's concern.  An application that
retains recordings for 30 days writes a sweeper that
deletes recordings older than that; the SDK doesn't.

## What's next

Chapter 7 walks **testing AI agents**.  The 305-test-file
discipline.  The mock-first pattern using
`ccai.testbase.mocking.Mock`.  What gets tested at
each layer.  The 49-minute wall time at
`TESTBASE_MAX_EXAMPLES_CAP=10`.

Chapter 8 is the property-tests-at-scale chapter — 72
property test files, the `@given(constraints=...)`
discipline, what property tests actually catch in an
LLM-driven SDK.

Chapter 9 is the case study — a worked example of an
agent that uses sysman features (Volume 4) against a
landscape (Volume 6) through interop protocols (Volume
5), observed through this chapter's machinery.
