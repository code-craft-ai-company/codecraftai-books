# Chapter 5 — Workflow Orchestration for Agents

> *Some agents handle *one shot* — request, response, done.
> Others run *workflows* — sequences of steps with conditions,
> parallel branches, human approvals, and resumable
> checkpoints.  The SDK ships a DAG-based `WorkflowExecutor`
> for the second case, plus `ApprovalGate` for human-in-the-
> loop and `CircuitBreaker` for resilience under repeated
> failures.*

This chapter is about workflows in the AI-agent context —
specifically, the SDK's mechanism for *agent-driven*
workflows that have decision points, branches, parallel
work, and resumability.  The workflow layer is *additive*:
agents can use it, or they can stay one-shot and skip it
entirely.

## When workflows make sense

Three signals an agent benefits from a workflow:

### Multiple LLM calls in sequence

A simple Q&A agent makes one LLM call per user message.  A
research agent that *plans, searches, reads results,
summarises, writes the final answer* makes five or ten LLM
calls.  Encoding the sequence as a workflow makes the
control flow *visible* — a debugger can see "we're at the
'read results' step" rather than tracing through hundreds
of lines of imperative code.

### Conditional branches

When the agent's behaviour differs based on something
discovered mid-run ("if the search returned no results,
ask the user for clarification; otherwise, summarise"),
encoding the branch declaratively in a workflow makes the
condition explicit.

### Long-running, resumable execution

An agent that runs for hours (a deep-research task, an
overnight code-modification workflow) benefits from
*checkpoints* (chapter 4).  The workflow layer integrates
checkpointing — every step boundary can be a checkpoint;
crashes resume from the last checkpoint.

## The DAG model

**File:** `workflowexecutor.py:41–73`

```python
class WorkflowExecutor:
    """
        Executes a WorkflowDefinition by traversing the DAG,
        evaluating edge conditions, executing steps, managing
        parallel branches, and producing a WorkflowResult.

        Checkpoints are saved via the CheckpointManager at the
        configured interval.  Failed steps are retried according
        to their RetryPolicy when configured.
    """

    def __init__(self, configuration: Optional[WorkflowConfiguration] = None,
                 checkpoint_manager: Optional[CheckpointManager] = None) -> None:
```

A workflow is a *directed acyclic graph* (DAG) of *steps*.
Each step:

- Has a unique `step_id`.
- Has a `step_callable` — the function (or coroutine)
  that does the step's work.
- Optionally has an `RetryPolicy`.
- Has zero or more *outgoing edges* with optional
  conditions.

The executor:

1. Validates the DAG is acyclic and all steps are reachable.
2. Topologically sorts the steps.
3. Walks the sort, executing each step in turn.
4. After each step, evaluates its outgoing edges'
   conditions.
5. Schedules downstream steps that have all incoming
   conditions satisfied.
6. Where multiple downstream steps can run in parallel,
   uses a `ThreadPoolExecutor` bounded by
   `max_parallel_workers`.
7. Saves checkpoints at configured boundaries.
8. Produces a final `WorkflowResult` with per-step
   outcomes.

### `WorkflowDefinition`

**File:** `workflowdefinition.py:1–15`

```python
class WorkflowDefinition:
    """
        Immutable description of a workflow's steps and edges.
        Validates structural integrity at construction:
        - No cycles in the step graph.
        - All steps reachable from the start step.
        - All edges reference defined step IDs.
        - All step callables are present.
    """
```

The definition is *immutable*.  Constructed once, validated
once, executed many times.  This means the workflow's
*shape* can be reasoned about statically — a tool can
inspect a `WorkflowDefinition` and know what steps will
run in what order, regardless of runtime behaviour.

A representative workflow:

```python
from automojo.agentkit.workflowdefinition import WorkflowDefinition
from automojo.agentkit.workflowstep import WorkflowStep
from automojo.agentkit.workflowedge import WorkflowEdge


def step_search(context):
    results = perform_search(context.get("query"))
    context.set("search_results", results)
    return results


def step_no_results(context):
    return ask_user_for_clarification(context)


def step_summarise(context):
    summary = summarise(context.get("search_results"))
    context.set("summary", summary)
    return summary


def step_finalise(context):
    return write_final_answer(context.get("summary"))


workflow = WorkflowDefinition(
    name="research_with_clarification",
    steps=[
        WorkflowStep(step_id="search", step_callable=step_search),
        WorkflowStep(step_id="no_results", step_callable=step_no_results),
        WorkflowStep(step_id="summarise", step_callable=step_summarise),
        WorkflowStep(step_id="finalise", step_callable=step_finalise),
    ],
    edges=[
        # After search, branch on whether results were found.
        WorkflowEdge(
            from_step="search", to_step="no_results",
            condition=lambda ctx: len(ctx.get("search_results")) == 0,
        ),
        WorkflowEdge(
            from_step="search", to_step="summarise",
            condition=lambda ctx: len(ctx.get("search_results")) > 0,
        ),
        # Both paths converge on finalise.
        WorkflowEdge(from_step="no_results", to_step="finalise"),
        WorkflowEdge(from_step="summarise", to_step="finalise"),
    ],
)
```

The workflow:

- Starts with `search`.
- Branches to `no_results` if the search returned empty,
  otherwise to `summarise`.
- Both paths converge on `finalise`.

The conditions are *callables* — they receive the
`WorkflowContext` and return a boolean.  This is more
flexible than declarative-only conditions; the application
can write arbitrary logic.

### Parallel branches

When a step has multiple downstream steps and the conditions
allow more than one to fire, the executor runs them
*concurrently*:

```python
WorkflowEdge(from_step="planning", to_step="research_topic_a"),
WorkflowEdge(from_step="planning", to_step="research_topic_b"),
WorkflowEdge(from_step="planning", to_step="research_topic_c"),

WorkflowEdge(from_step="research_topic_a", to_step="synthesise"),
WorkflowEdge(from_step="research_topic_b", to_step="synthesise"),
WorkflowEdge(from_step="research_topic_c", to_step="synthesise"),
```

After `planning`, the three `research_topic_*` steps run
in parallel (subject to `max_parallel_workers`).  When all
three complete, `synthesise` runs (it has three incoming
edges, all required).

The parallelism is *thread-based* via `ThreadPoolExecutor`.
This works well for I/O-bound steps (LLM calls, HTTP
requests, file operations).  For CPU-bound work, consider
a process pool variant — the SDK ships extension hooks for
this but the default is threads.

## Checkpointing integration

The executor accepts a `CheckpointManager` (chapter 4):

```python
executor = WorkflowExecutor(
    configuration=WorkflowConfiguration(
        checkpoint_interval=1,  # checkpoint after every step
    ),
    checkpoint_manager=CheckpointManager(memory_store=file_store),
)

result = executor.execute(workflow, initial_context)
```

After each step, the executor:

1. Records the step's completion in the checkpoint state.
2. Calls `checkpoint_manager.save_checkpoint(...)`.
3. Continues with the next step.

When the process dies and restarts:

```python
checkpoint = checkpoint_manager.load_checkpoint(execution_id)
result = executor.resume(workflow, checkpoint)
```

The executor:

1. Identifies the steps already completed.
2. Reconstructs the context from `checkpoint.context_state`.
3. Resumes execution from `checkpoint.current_step_id`.

The discipline matches Volume 2 chapter 7's verified-
contract model: the executor *verifies* the checkpoint's
shape against the workflow definition before resuming.  A
checkpoint from a different (incompatible) workflow
version raises `CheckpointMismatchError`.

## Approval gates — human-in-the-loop

**File:** `approvalgate.py:18–38` and `approvalrequest.py:22–71`

Some workflows need human approval at specific steps.  An
agent that's about to send a customer-facing email might
require a human to review.  An agent about to delete data
might require explicit approval.

```python
class ApprovalGate:
    """
        Manages the lifecycle of approval requests within
        workflow execution.
    """

    def __init__(self) -> None:
        self._pending_requests = {}
        self._handlers = []
```

```python
class ApprovalRequest:
    """
        Immutable object representing a pending approval
        within a workflow.

        VALID_STATUSES = {"pending", "approved", "rejected", "expired"}
    """
```

The flow:

1. The workflow's step calls `gate.create_request(...)`.
2. The request is added to pending; handlers are notified
   (e.g., the application sends an email to a reviewer).
3. The step *blocks* on `request.wait_for_resolution(timeout=...)`.
4. A separate process (the human's UI) calls
   `gate.approve(request_id)` or `gate.reject(request_id,
   reason="...")`.
5. The waiting step unblocks and proceeds based on the
   resolution.

If the timeout elapses without resolution, the request
transitions to `"expired"` and the step receives an
`ApprovalTimeoutError`.

A worked usage:

```python
def step_send_email(context):
    gate = context.get("approval_gate")

    request = gate.create_request(
        execution_id=context.execution_id,
        step_id="send_email",
        description="Approve sending email to {} customers".format(
            len(context.get("recipients"))
        ),
        requesting_agent_id=context.get("agent_id"),
        metadata={"recipients": context.get("recipients")},
    )

    # Notify reviewers (out-of-band).
    notify_reviewers(request)

    # Wait for human resolution.
    resolution = request.wait_for_resolution(timeout=3600)

    if resolution == "approved":
        send_emails(context.get("recipients"))
    elif resolution == "rejected":
        raise WorkflowAbortedError(
            f"Send-email step rejected: {request.rejection_reason}"
        )
```

The workflow's *DAG structure* doesn't change — it's still
a graph of steps.  But one of the steps blocks on human
input.  The checkpointing (above) means the workflow can
*pause* at the gate without keeping a process alive — the
checkpoint records "waiting on approval request X"; a
fresh process can resume when the resolution arrives.

## Circuit breakers — resilience

**File:** `circuitbreaker.py:21–170` and
`circuitbreakerstate.py:11–23`

```python
class CircuitBreakerState(enum.Enum):
    """
        CLOSED: Normal operation, requests pass through.
        OPEN: Service is degraded, requests are rejected immediately.
        HALF_OPEN: Recovery testing, limited requests allowed through.
    """
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"
```

The circuit breaker is a classic resilience pattern:

- **CLOSED** — normal.  Requests pass through.  Failures
  increment a counter.
- **OPEN** — failure threshold reached.  Requests are
  *rejected immediately* without attempting the underlying
  call.  Timer counts down toward HALF_OPEN.
- **HALF_OPEN** — recovery testing.  Limited requests are
  allowed through.  Successes count toward closing the
  breaker; failures re-open.

The state transitions:

```python
# circuitbreaker.py — _on_success / _on_failure

def _on_success(self, state):
    if state == CircuitBreakerState.HALF_OPEN:
        self._success_count += 1
        if self._success_count >= self._configuration.success_threshold:
            self._transition_to(CircuitBreakerState.CLOSED, "success threshold reached in half-open")
    elif state == CircuitBreakerState.CLOSED:
        self._success_count += 1


def _on_failure(self, state):
    if state == CircuitBreakerState.HALF_OPEN:
        self._transition_to(CircuitBreakerState.OPEN, "failure in half-open state")
    elif state == CircuitBreakerState.CLOSED:
        self._failure_count += 1
        if self._failure_count >= self._configuration.failure_threshold:
            self._transition_to(CircuitBreakerState.OPEN, "failure threshold reached")
```

The breaker's purpose: *prevent cascading failure*.  When
a downstream service is broken (LLM provider returning
500s, tool's API rate-limiting), the breaker opens and
*stops trying* — saving the agent from spending tokens /
time / quota on calls that will fail.

A typical usage:

```python
breaker = CircuitBreaker(
    name="openai-completions",
    configuration=CircuitBreakerConfiguration(
        failure_threshold=5,    # 5 failures → OPEN
        success_threshold=2,    # 2 successes in HALF_OPEN → CLOSED
        timeout=60.0,           # OPEN → HALF_OPEN after 60s
    ),
)


def call_with_breaker(provider, conversation):
    return breaker.execute(
        callable_to_protect=lambda: provider.complete(conversation),
    )
```

The `breaker.execute(...)` wraps the call.  When the
breaker is OPEN, it raises `CircuitBreakerOpenError` *without*
calling the underlying callable — saving the agent from
the cost of a likely-failing call.

The breaker is *thread-safe* (every transition acquires
`threading.Lock`).  Multiple threads sharing a breaker
see consistent state.

## Combining the pieces

A representative agent workflow combining everything:

```python
class ResearchAgent(Agent):

    def initialize(self) -> None:
        super().initialize()

        # LLM provider with circuit breaker
        provider = OpenAIProvider(model_name="gpt-4o", ...)
        self._llm_provider_registry.register(provider)
        self._llm_breaker = CircuitBreaker(
            name="openai-research",
            configuration=CircuitBreakerConfiguration(...),
        )

        # Memory + checkpointing
        self._memory_store = FileMemoryStore(root_dir="research_memory/")
        self._checkpoint_manager = CheckpointManager(
            memory_store=self._memory_store,
        )

        # Approval gate for sensitive steps
        self._approval_gate = ApprovalGate()

        # Workflow
        self._workflow = self._build_workflow()

    def start(self) -> None:
        super().start()
        executor = WorkflowExecutor(
            configuration=WorkflowConfiguration(checkpoint_interval=1),
            checkpoint_manager=self._checkpoint_manager,
        )
        result = executor.execute(self._workflow, self._initial_context())
        self._handle_result(result)
```

The agent integrates:

- LLM provider (chapter 2).
- Tool registry (chapter 3, registered separately).
- Memory store and checkpointing (chapter 4 + this).
- Workflow executor (this chapter).
- Approval gates and circuit breakers (this chapter).
- Cost tracker (built into the agent base).
- Observability pipeline (chapter 6).

Each subsystem is independent.  Each is configurable.
The agent's `initialize()` and `start()` methods wire them
together for the application's specific shape.

## Agent-driven workflows vs workflow-driven agents

A vocabulary distinction worth pinning down.

### Agent-driven workflows

The agent *owns* the workflow.  It constructs the
`WorkflowDefinition`, drives the `WorkflowExecutor`,
makes decisions about which steps to add or skip based on
runtime state.

This is the SDK's primary pattern.  The agent's
`start()` method runs the executor; the workflow is
the agent's *internal* control flow.

### Workflow-driven agents

The workflow *owns* the agent.  An external orchestrator
(Volume 3's taskplus, for example) defines a workflow with
explicit agent-invocation steps.  The agent is invoked at
each step; the workflow is the *external* control flow.

This pattern is supported through Volume 3 chapter 25's
bridge tasks.  A taskplus workflow has a step that
invokes an agent; the agent runs, returns; the workflow
continues with the next step.

The two patterns can *combine* — a taskplus workflow
invokes an agent that runs its own internal workflow.  The
SDK doesn't distinguish architecturally; what matters is
*who decides* which step runs next.

## What's next

Chapter 6 walks **observability for agents**.  The SDK's
`Tracer` and `MetricsCollector` for emitting structured
observability data, the 13 built-in agent metrics, and
`ExecutionRecorder` / `ExecutionReplayer` for full
post-hoc replay of an agent run.  Chapter 6 is also the
chapter that walks the *cost tracker* in detail — every
LLM call's token usage and cost flow through the
observability layer.
