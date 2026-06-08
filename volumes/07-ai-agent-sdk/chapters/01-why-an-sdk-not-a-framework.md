# Chapter 1 — Why an SDK and Not a Framework

> *Most AI-agent libraries are *frameworks*: they impose a
> structure that the application has to fit into.  The Agent
> Class.  The Tool Class.  The Memory Class.  Inherit from
> these specific shapes and the library wires you up.
> `codecraftai-agent-kit` is an *SDK*.  It ships base classes,
> but the application owns the shape.  This chapter is about
> the difference, and why a production AI agent benefits from
> the SDK shape.*

`codecraftai-agent-kit` is the volume's subject — a Python SDK
for building AI agents.  The package is large by ccai
standards: ~217 modules, ~46k LOC, 305 test files at last
count.  Volume 7 is the framework's largest book because its
subject is the framework's largest package.

This chapter establishes *why the SDK is shaped the way it
is*.  By the end, a reader should understand the
framework-vs-SDK distinction, what `codecraftai-agent-kit`
commits to, and what it deliberately leaves to the application.

## The framework vs SDK distinction

A *framework* takes control of the application's flow.  You
write your code in the shapes the framework requires, and
the framework calls into your code at the points it decides.
Django is a framework — your code lives in `views.py`,
`models.py`, `urls.py`; Django's request loop calls your
view functions.

An *SDK* gives you base classes and helpers.  Your
application controls the flow; you call into the SDK at
the points *you* decide.  The standard library's
`http.client` is an SDK — your code creates an
`HTTPConnection`, calls `request()`, reads the response.
The library doesn't take over the program.

Most AI-agent libraries published since 2023 are frameworks.
LangChain has chains.  LlamaIndex has indexes.  AutoGen has
agent groups.  Each imposes a *control flow*: you build your
agent inside the framework's structure, and the framework
runs it.

`codecraftai-agent-kit` is an SDK.  It ships:

- `Agent` (an abstract base class) — you subclass it.
- `Tool` (abstract base) — you subclass each tool.
- `LLMProvider` (abstract base) — you instantiate one.
- `MemoryStore` (abstract base) — you choose an
  implementation.
- ...and roughly 200 other classes that solve specific
  problems.

But there is **no framework loop**.  Your application
constructs an Agent, registers tools, calls into the agent's
methods, and decides what to do with the results.  You
write `main()`; the SDK doesn't.

## Why this matters for production agents

A reader from a research-AI background might prefer the
framework model — it's faster to prototype, fewer decisions
to make.  Production agents have different concerns:

### Production agents need to fit existing applications

Production AI agents typically integrate into *existing*
applications: a customer-support workflow, a CI/CD pipeline,
a monitoring system.  The application has its own control
flow, its own logging, its own configuration story.  An
agent framework that wants to *take control* fights the
existing application; an SDK lets the application keep
control.

### Production agents need observability the application owns

When an agent's behaviour is wrong, the application's
operators need to debug it using the application's existing
observability — its tracing, its metrics, its log
aggregation.  A framework that has its *own* observability
adds a second tier the operators have to learn.  An SDK that
emits structured events the application captures into its
existing systems is operationally simpler.

### Production agents need testability

A test that exercises an agent's logic against a mock LLM
should look like every other test in the application's test
suite.  Frameworks often have *test runners* of their own;
SDKs let the application's existing test framework drive.

`codecraftai-agent-kit`'s tests use the framework's standard
testbase machinery (Volume 1 chapter 2).  No special
agent-test runner.  Mocks are constructed using the
portfolio's standard `Mock` class (Volume 1 chapter 7).
The 305 test files exercise the same patterns the rest of
the portfolio uses.

### Production agents need predictable failure modes

A framework that hides failures behind retries, fallbacks,
or magic recovery makes failures harder to diagnose.  An SDK
that surfaces failures through typed exceptions and
structured results lets the application decide what to do.

`codecraftai-agent-kit`'s tools return `ToolResult` objects
with `STATUS_SUCCESS` or `STATUS_FAILURE`; LLM providers
raise typed exceptions; the agent's lifecycle has explicit
states (`CREATED / INITIALIZED / RUNNING / PAUSED / ERROR /
STOPPED`).  Nothing is hidden.

## The agent-kit's three commitments

The SDK makes three commitments to its users.  Each is
load-bearing for the chapters that follow.

### Commitment 1 — The agent is *yours*; the SDK provides building blocks

A reader inheriting from `Agent` writes:

```python
class MyApplicationAgent(Agent):

    def initialize(self) -> None:
        # Your initialization logic.
        super().initialize()
        self._configure_my_tools()

    def start(self) -> None:
        # Your start logic.
        super().start()
        self._run_my_application_loop()

    # ... etc.
```

The agent's behaviour — what it does, when it does it —
is the application's code.  The SDK's `Agent` base provides:

- The lifecycle state machine (`AgentStateMachine` —
  chapter 1's worked example below).
- Hook registration (`on_initialize`, `on_start`,
  `on_pause`, `on_resume`, `on_stop`, `on_error`).
- Identity (`AgentIdentity` with name, version, tags,
  identifier).
- Configuration (`AgentConfiguration` carrying the above).
- The standard subsystems (tool registry, LLM provider
  registry, cost tracker, memory store, observability
  pipeline).

The base doesn't impose a particular *agent loop*.  Some
applications run the agent in a single conversation per
request; others run it as a long-lived service consuming a
queue; others embed it in a workflow engine.  All three are
SDK-supported because the *loop is the application's*.

### Commitment 2 — Tools are typed and validated

Every tool subclasses `Tool` and declares:

```python
class FetchUserProfileTool(Tool):
    TOOL_NAME = "fetch_user_profile"
    TOOL_DESCRIPTION = "Look up a user's profile by user ID."
    TOOL_VERSION = "1.0.0"
    INPUT_SCHEMA = {
        "parameters": [
            {"name": "user_id", "type": "string", "required": True},
            {"name": "include_history", "type": "boolean", "required": False, "default": False},
        ]
    }
    OUTPUT_SCHEMA = {
        "parameters": [
            {"name": "user_id", "type": "string", "required": True},
            {"name": "username", "type": "string", "required": True},
            {"name": "email", "type": "string", "required": True},
            {"name": "history", "type": "list", "required": False},
        ]
    }

    def execute(self, inputs: dict, credentials: dict = None) -> dict:
        # Your tool logic.
        return {
            "user_id": inputs["user_id"],
            "username": "...",
            "email": "...",
        }
```

The schema is *validated* at execution time.  If the LLM
asks the agent to call this tool with `user_id` missing, the
SDK's `ToolExecutor` raises `ToolValidationError` *before*
the tool's `execute` method runs.  If the tool returns
without a `username` field, the same validation catches it.

Chapter 3 walks the schema system in detail.  The point
here: tools have *typed contracts* with the agent, and the
SDK enforces them.

### Commitment 3 — Cost is tracked, not hidden

Every Agent owns a `CostTracker` (chapter 1's worked example
below).  Every LLM completion records a `TokenUsage` and a
`CostEstimate` against the tracker.  At any point, the
application can read:

```python
print(agent.cost_tracker.total_cost_estimate)
# CostEstimate(prompt_cost=0.42, completion_cost=1.18, total_cost=1.60, currency='USD')

print(agent.cost_tracker.per_provider_breakdown())
# {'openai-gpt-4o': {'token_usage': TokenUsage(...), 'cost_estimate': CostEstimate(...)},
#  'anthropic-claude-opus': {'token_usage': TokenUsage(...), 'cost_estimate': CostEstimate(...)}}
```

The SDK's commitment: an agent's cost is always
*observable*.  Production agents that run for hours or days
need this — surprises in the LLM bill at end-of-month are
the most common failure mode for AI products.  The cost
tracker is *built in*, not optional.

## A worked Agent subclass

To make the SDK shape concrete, here's a minimal but
complete Agent subclass.

```python
from ccai.agentkit.agent import Agent
from ccai.agentkit.agentconfiguration import AgentConfiguration
from ccai.agentkit.lifecyclestate import LifecycleState


class HelloAgent(Agent):
    """
        Simplest possible agent — initializes, starts, says
        hello via an LLM, stops.
    """

    def initialize(self) -> None:
        super().initialize()
        # Register one tool, one provider — left as exercise.

    def start(self) -> None:
        super().start()
        # Application's work happens here.
        provider = self._llm_provider_registry.lookup_default()
        conversation = LLMConversation(messages=[
            LLMMessage(role="user", content="Hello!"),
        ])
        response = provider.complete(conversation)
        print(response.content)

    def pause(self) -> None:
        super().pause()

    def resume(self) -> None:
        super().resume()

    def stop(self) -> None:
        super().stop()

    def reset(self) -> None:
        super().reset()


# Application's main:
config = AgentConfiguration(name="hello-agent", version="1.0.0")
agent = HelloAgent(configuration=config)
agent.initialize()
agent.start()
agent.stop()
```

Five things to notice:

1. **Lifecycle methods are explicit.**  The application
   calls `initialize()` / `start()` / `stop()`.  The SDK
   doesn't run the agent for you.
2. **State machine validates transitions.**  Calling `stop()`
   without first calling `start()` raises a `SemanticError` —
   the state machine's `VALID_TRANSITIONS` table doesn't
   allow it.
3. **The Agent's own work is in `start()`.**  This is where
   the application puts its agent loop.  Other patterns
   (running the agent inside a workflow, exposing it as an
   HTTP endpoint, queue-driven) are equally fine; the
   commitment is that *the application chose*.
4. **`super()` calls handle the lifecycle.**  The base class
   transitions the state machine, fires hooks, emits
   observability events.  Skipping `super()` skips that
   discipline.
5. **The provider, tools, memory, and cost tracker are
   *registered* during `initialize()`.**  The SDK provides
   the registries; the application populates them with the
   specific providers / tools / stores it needs.

A reader writing a real agent fills in the blanks (real
tools, real providers, real memory) — but the *shape* stays
the same.

## The state machine

A small diversion into the state machine because it shows
up in every chapter that follows.

**File:** `agentstatemachine.py:22–38`

```python
class AgentStateMachine:
    """
        State machine that enforces valid lifecycle transitions
        for an agent.
    """

    VALID_TRANSITIONS: Dict[LifecycleState, List[LifecycleState]] = {
        LifecycleState.CREATED: [LifecycleState.INITIALIZED],
        LifecycleState.INITIALIZED: [LifecycleState.RUNNING, LifecycleState.STOPPED],
        LifecycleState.RUNNING: [LifecycleState.PAUSED, LifecycleState.STOPPED, LifecycleState.ERROR],
        LifecycleState.PAUSED: [LifecycleState.RUNNING, LifecycleState.STOPPED],
        LifecycleState.ERROR: [LifecycleState.STOPPED],
        LifecycleState.STOPPED: [LifecycleState.INITIALIZED],
    }
```

Six states, ten valid transitions.  The state machine:

- Starts every agent in `CREATED`.
- Allows `CREATED → INITIALIZED` only via the
  `initialize()` lifecycle method.
- Allows `INITIALIZED → RUNNING` via `start()`.
- Allows `RUNNING → PAUSED` (via `pause()`) or `RUNNING →
  STOPPED` (via `stop()`) or `RUNNING → ERROR` (when an
  unhandled exception fires the `on_error` hook chain).
- Allows `PAUSED → RUNNING` (via `resume()`) or `PAUSED →
  STOPPED`.
- From `ERROR`, only `STOPPED` is reachable.
- From `STOPPED`, the agent can be re-initialised
  (`STOPPED → INITIALIZED`).

The transition table is what the application's testing
scaffolding (chapter 7) verifies.  An agent that violates
the state machine — calling `stop()` from `CREATED`,
calling `start()` twice without an intervening `stop()` —
raises `SemanticError`.  The discipline catches application
bugs early.

The state machine is *thread-safe* — every transition
acquires a `threading.Lock`.  An agent that's running on one
thread while another thread observes `agent.state` sees a
consistent value.

## What the SDK *doesn't* do

A short list of things `codecraftai-agent-kit` deliberately
leaves to the application:

### It doesn't decide which LLM to use

The agent-kit ships an abstract `LLMProvider` (chapter 2).
It doesn't ship a default implementation.  The application
chooses OpenAI, Anthropic, Azure, a local model — and
constructs the appropriate provider class (or writes one if
the SDK doesn't ship the provider's class).

This is *intentional*.  Pinning a default provider ties the
SDK's release cadence to the LLM industry's churn.  Leaving
the provider open lets the application switch between
providers (or run multiple in parallel) without the SDK
caring.

### It doesn't decide which tools an agent has

Every tool is registered explicitly:

```python
agent.tool_registry.register(FetchUserProfileTool())
agent.tool_registry.register(SendEmailTool())
```

The SDK doesn't auto-discover tools, scan modules, or
register based on naming convention.  Tools are *the
application's* domain knowledge; the application is
responsible for choosing which it exposes.

### It doesn't decide the agent loop

A *loop* in framework terminology is "what the agent does
when it runs."  Chains things in LangChain.  Conversation
turns in OpenAI Assistants.  The agent-kit doesn't impose
one.  Some applications:

- Run the agent in a request/response loop (one user
  message per agent invocation).
- Run the agent as a long-lived service, consuming a queue.
- Run the agent inside a workflow with checkpoints.
- Run the agent in a single batch invocation.

All four are supported because the SDK doesn't prescribe.
Chapter 5 walks workflow orchestration as one option among
many.

### It doesn't decide observability backends

Chapter 6 walks the observability layer — `Tracer`,
`MetricsCollector`, `ExecutionRecorder`.  The SDK emits
structured events; the application wires them to its
backend (OpenTelemetry, Datadog, Prometheus, custom).  The
SDK doesn't ship integrations to specific backends — those
live in adapter packages outside the agent-kit.

## How this volume is organised

The remaining eight chapters walk the SDK by subsystem:

- **Chapter 2** — multi-provider LLM abstraction.  The
  `LLMProvider` interface, `AlternateProviderStrategy` for
  failover, the per-agent registry.
- **Chapter 3** — tool design.  `Tool` base class,
  `ToolSchema` (custom — not Pydantic), `ToolExecutor`
  validation, `ToolResult` structured outcomes.
- **Chapter 4** — memory architecture.  `MemoryStore`,
  `MemoryScope` (AGENT / SHARED / GLOBAL), `LLMConversation`
  truncation, summarisation, semantic search.
- **Chapter 5** — workflow orchestration.  DAG-based
  `WorkflowExecutor`, `ApprovalGate` for human-in-the-loop,
  `CircuitBreaker` for resilience.
- **Chapter 6** — observability.  `Tracer` /
  `MetricsCollector` / `ExecutionRecorder` plus the 13
  built-in agent metrics.
- **Chapter 7** — testing AI agents.  The 305 test files,
  the mock-first discipline, the `ccai.testbase.mocking.Mock`
  pattern.
- **Chapter 8** — property tests at scale.  72 property-
  based test files, the `@given(constraints={"max_examples":
  50})` cap, what the property tests actually catch.
- **Chapter 9** — building an agent that uses the rest of
  the ccai portfolio.  An agent that runs sysman
  features, observes through Volume 8's time-travel
  debugging, exposes a Flask dashboard.

By the end of the volume, a reader should be able to read
any module in `codecraftai-agent-kit` and predict its shape
from the patterns established here.

## What's next

Chapter 2 walks the multi-provider LLM abstraction.  The
`LLMProvider` base class with its `complete()` and
`complete_stream()` abstract methods.  The
`AlternateProviderStrategy` for failover when a provider is
unavailable.  The per-agent `LLMProviderRegistry` keyed on
`(provider_name, model_name)` so an agent can use multiple
providers in the same session.
