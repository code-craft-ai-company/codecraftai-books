# Volume 7 — Code-Anchor Index

## codecraftai-agent-kit overview

- **Package layout** at `codecraftai-agent-kit/source/packages/ccai/agentkit/`.
- **~217 modules**, ~46k LOC.
- **305 test files** under `source/testroots/ccai/tests/singlehost/agentkit/` (~78k LOC of test code).
- **72 property-based test files** (naming convention `test_*_properties.py`).
- **Recent verification**: 1,964 / 1,964 passed (commit `a2e437a`, Apr 27 2026).

## Agent core (chapter 1)

### Agent and AsyncAgent

- `agent.py:94–120` — `Agent(abc.ABC)` declaration with `_configuration`, `_identity`, `_state_machine`, `_parent`, `_sub_agents`, `_extensions`, `_hooks` (six hook lists).
- `agent.py:106–176` — constructor wiring `ToolRegistry`, `LLMProviderRegistry`, `CostTracker`, `MemoryStore`, `ObservabilityPipeline`.  Auto-registers in `AgentRegistry.instance()`.
- `asyncagent.py:24–42` — `AsyncAgent(Agent)` with `ConcurrencyModel.ASYNCIO`.
- `asyncagent.py:79–119` — sync lifecycle delegates to async via `_run_async()`.
- `asyncagent.py:165–186` — `_run_async` detects existing event loop.

### State machine

- `agentstatemachine.py:22–38` — `AgentStateMachine` class with `VALID_TRANSITIONS` dict.
- `lifecyclestate.py:9–22` — `LifecycleState` enum: `CREATED / INITIALIZED / RUNNING / PAUSED / STOPPED / ERROR`.
- Transitions: CREATED → INITIALIZED; INITIALIZED → RUNNING / STOPPED; RUNNING → PAUSED / STOPPED / ERROR; PAUSED → RUNNING / STOPPED; ERROR → STOPPED; STOPPED → INITIALIZED.

### Registry

- `agentregistry.py:19–46` — `AgentRegistry` singleton with thread-safe `instance()` plus `_instance_lock`.
- `agentregistry.py:48–102` — `register`, `deregister`, `lookup_by_id`, `lookup_by_name`, `lookup_by_tags`, `list_all`.

## LLM provider abstraction (chapter 2)

- `llmprovider.py:16–60` — `LLMProvider(abc.ABC)` base with `provider_name`, `model_name`, `config`, `credential_names`, `tags`.
- `llmprovider.py:100–125` — abstract `complete(conversation, **kwargs)` and `complete_stream(conversation, handler, **kwargs)` methods.
- `alternateproviderstrategy.py:12–36` — `AlternateProviderStrategy(DegradationStrategy)` for failover.
- Per-agent `LLMProviderRegistry` keyed on `(provider_name, model_name)` composite.

## Tool architecture (chapter 3)

- `tool.py:23–83` — `Tool(abc.ABC)` with class-level `TOOL_NAME`, `TOOL_DESCRIPTION`, `TOOL_VERSION`, `INPUT_SCHEMA`, `OUTPUT_SCHEMA`.
- `tool.py:execute` — abstract method.
- `toolregistry.py:16–49` — `ToolRegistry` with `register`, `lookup`, `lookup_by_tags` (no duplicates allowed; raises `SemanticError`).
- `toolexecutor.py:21–162` — `ToolExecutor` validates inputs, resolves credentials, invokes execute, validates outputs.
- `toolerrors.py:15–83` — `ToolValidationError(AgentKitError)`.
- `toolerrors.py:99–148` — `ToolExecutionError(AgentKitError)`.
- `toolresult.py:10–96` — `ToolResult` with `STATUS_SUCCESS = "success"`, `STATUS_FAILURE = "failure"`.
- `toolschema.py:14–56` — custom `ToolSchema` with six `SUPPORTED_TYPES` (string, integer, float, boolean, list, dictionary).  NOT Pydantic.
- `toolschema.py:68–111` — `validate(values)` returns list of error strings.

## Memory architecture (chapter 4)

- `memorystore.py:25–127` — `MemoryStore(abc.ABC)`.
- `filememorystore.py:32–195` — `FileMemoryStore(MemoryStore)` JSON-on-disk implementation.
- `memoryscope.py:13–24` — `MemoryScope` enum: `AGENT / SHARED / GLOBAL`.
- `memoryentry.py:17–48` — `MemoryEntry` immutable, tagged.
- `embeddingprovider.py:15–41` — `EmbeddingProvider` abstract base.
- `memoryquery.py:15–45` — `MemoryQuery` with tags, metadata, scope, time range, semantic_text.
- `memoryindex.py:17–52` — `MemoryIndex` with cosine similarity ranking.
- `llmconversation.py:15–84` — `LLMConversation` with `truncate(max_messages)` preserving system messages.
- `contextsummarizer.py:22–108` — `summarize_messages(messages)` using LLM provider.
- `checkpointmanager.py:22–75` — `CheckpointManager.save_checkpoint` / `load_checkpoint`.

## Workflow orchestration (chapter 5)

- `workflowexecutor.py:41–73` — `WorkflowExecutor` traverses DAG with topological sort.
- `workflowdefinition.py:1–15` — `WorkflowDefinition` immutable with cycle / reachability validation.
- `approvalgate.py:18–38` — `ApprovalGate` with pending requests + handler callbacks.
- `approvalrequest.py:22–71` — `ApprovalRequest` with `VALID_STATUSES = {"pending", "approved", "rejected", "expired"}`.
- `circuitbreaker.py:21–170` — `CircuitBreaker` with state machine.
- `circuitbreakerstate.py:11–23` — `CircuitBreakerState` enum: `CLOSED / OPEN / HALF_OPEN`.

## Observability (chapter 6)

- `tracer.py:22–100` — `Tracer` with hierarchical spans + probabilistic sampling.
- `tracespan.py:22–40` — `TraceSpan` with status `("unset", "ok", "error")`.
- `traceevent.py:18–49` — `TraceEvent` for span-attached events.
- `metricscollector.py:21–74` — `MetricsCollector` with counter / gauge / histogram.
- `builtinagentmetrics.py:16–114` — **13 built-in metrics**:
  1. `agent.lifecycle.transitions` (counter)
  2. `agent.tool.execution.duration` (histogram)
  3. `agent.tool.execution.errors` (counter)
  4. `agent.llm.request.duration` (histogram)
  5. `agent.llm.token.usage.prompt` (counter)
  6. `agent.llm.token.usage.completion` (counter)
  7. `agent.llm.request.errors` (counter)
  8. `agent.memory.operations` (counter)
  9. `agent.message.sent` (counter)
  10. `agent.message.received` (counter)
  11. `agent.workflow.step.duration` (histogram)
  12. `agent.workflow.step.errors` (counter)
  13. `agent.active.spans` (gauge)

### Replay

- `executionrecorder.py:17–70` — `ExecutionRecorder` captures `ExecutionRecord` objects with monotonic sequence numbers.
- `executionrecord.py:16–53` — `ExecutionRecord` with `sequence_number`, `event_type`, `correlation_id`, `payload`.
- `executionrecording.py:18–46` — `ExecutionRecording` immutable container.
- `executionreplayer.py:20–57` — `ExecutionReplayer` with forward / backward / seek / filter.

## Cost tracking (chapter 1, ch.6)

- `costtracker.py:14–104` — `CostTracker` with `record(provider_name, token_usage, cost_estimate)`, `total_token_usage`, `total_cost_estimate`, `per_provider_breakdown()`.
- `tokenusage.py:12–36` — `TokenUsage` immutable with validation that `total_tokens == prompt_tokens + completion_tokens`.
- `costestimate.py:13–42` — `CostEstimate` immutable with currency default USD; validates `total_cost ≈ prompt_cost + completion_cost` within 1e-9.

## Testing (chapters 7, 8)

### Test counts

- 305 test files at `source/testroots/ccai/tests/singlehost/agentkit/` (~78,155 LOC).
- 212 top-level test files (mostly `test_*.py`).
- 72 property-based test files (`test_*_properties.py`).
- 5 subdirectory layers: `builtins/` (5), `platform/` (14), `resilience/` (53), `steps/`, `transports/`.

### Mock pattern

- Tests use `ccai.testbase.mocking.Mock` (not unittest.mock).
- `Mock().configure(recursive=True)` enables nested mock attribute access.
- `mock_context().set_attribute(name, value)` for properties.
- `mock_context().mock_method(name, return_value=...)` for return values.
- `mock_context().mock_method(name, side_effect=callable)` for exception simulation.

### Property test structure

- `test_costtracker_properties.py:74–283` — five property functions covering request count, total token usage, total cost, per-provider breakdown, reset.
- `test_tokenusage_properties.py:29–69` — construction-validation properties.
- `test_llmconversation_properties.py:57–84` — JSON round-trip property.
- All property tests use `constraints={"max_examples": 50}` (or 100).

### Recent migration

- Commit `8f31798` (Mar 8 2026) — *"Switch tests over from hypothesis to testbase"* — replaced `hypothesis` with `ccai.testbase.properties`.
- Commit `a2e437a` (Apr 27 2026) — added 89 new modules + 97 new test files.

### No CI workflow file

- Makefile contains only dist targets.
- No `.github/workflows` test runners.
- 49-minute wall-time figure from Vol 1 ch.10 likely refers to local single-host execution.

## Integration with rest of portfolio (chapter 9)

- agent-kit tests do **NOT** import from `sysman`, `landscape`, or `interop`.
- agent-kit is a *library*; downstream packages test their integration with it.
- The integration is one-directional.
