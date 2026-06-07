# Chapter 9 — Building an Agent That Uses the Rest of the Code Craft AI Portfolio

> *Eight chapters established the SDK.  This chapter watches
> the SDK in motion — an agent that runs sysman features
> against a landscape through interop protocols, with full
> event recording for time-travel debugging.  Every subsystem
> the SDK exposes lands in this case study.*

The volume's payoff.  A worked example of an agent that
*uses* the framework's other tiers — sysman from Volume 4,
landscape from Volume 6, interop protocols from Volume 5,
testbase from Volume 1.  By the end, a reader should be
able to take the case study's shape and apply it to their
own automation domain.

The case study is realistic.  An agent that *triages
production incidents* — when an alert fires, the agent
gathers context from production hosts, queries logs, runs
diagnostics, summarises findings, and either fixes the
issue or escalates to a human via approval gate.

## The scenario

A small operations team receives ~10 alerts per day across
their fleet of ~200 production hosts.  Each alert needs:

1. Context gathered (logs, metrics, recent deployments).
2. Diagnostics run (system state checks).
3. Severity assessed.
4. Either a fix applied or escalated to humans.

The team's SLA is 15 minutes for triage.  Manually, this
takes 10–15 minutes per alert.  An AI-driven triage agent
could handle low-severity alerts autonomously and produce
rich context for high-severity ones.

The agent is *not* trying to replace human judgement.  It's
trying to make the human's job easier — present what
matters, suppress what doesn't, propose actions.

## The agent's shape

```python
from ccai.agentkit.agent import Agent
from ccai.agentkit.agentconfiguration import AgentConfiguration
# ... other imports ...


class IncidentTriageAgent(Agent):
    """
        Triages production incidents.  Gathers context, runs
        diagnostics, summarises, escalates when severity exceeds
        an autonomous-action threshold.
    """

    def initialize(self) -> None:
        super().initialize()

        # LLM provider with circuit breaker
        self._configure_llm_providers()

        # Tools — sysman feature wrappers, log queries, etc.
        self._configure_tools()

        # Memory store for retaining incident-history context
        self._configure_memory()

        # Workflow definition
        self._workflow = self._build_workflow()

        # Approval gate for autonomous-action threshold
        self._approval_gate = ApprovalGate()

        # Observability — recorder for time-travel debugging
        self._execution_recorder = ExecutionRecorder(
            agent_id=self.identity.identifier,
        )
        self._execution_recorder.register_sink(
            FileExecutionSink(path="recordings/")
        )

        return

    def start(self) -> None:
        super().start()
        # Wait for an incident; when one arrives, run the workflow.
        self._wait_for_incident()
```

Eight subsystems wired in `initialize()`:

- LLM providers (chapter 2).
- Tool registry (chapter 3).
- Memory store (chapter 4).
- Workflow definition (chapter 5).
- Approval gate (chapter 5).
- Circuit breaker (chapter 5).
- Execution recorder (chapter 6).
- Cost tracker (built into the Agent base, chapter 1).

## The tool surface — sysman feature wrappers

The agent needs to *do things* on production hosts:

- Read system state (free disk, running services, recent
  log entries).
- Run sysman feature queries (Volume 4's read verbs).
- Apply sysman feature changes (Volume 4's write verbs)
  when authorised.

Each capability is a *tool* (chapter 3).  A representative
set:

```python
class GetServiceStatusTool(Tool):
    """
        Query the status of a service on a production host.
    """
    TOOL_NAME = "get_service_status"
    TOOL_DESCRIPTION = "Get the status of a service on a host."
    TOOL_VERSION = "1.0.0"
    INPUT_SCHEMA = {
        "parameters": [
            {"name": "host_name", "type": "string", "required": True},
            {"name": "service_name", "type": "string", "required": True},
        ]
    }
    OUTPUT_SCHEMA = {
        "parameters": [
            {"name": "host_name", "type": "string", "required": True},
            {"name": "service_name", "type": "string", "required": True},
            {"name": "active_state", "type": "string", "required": True},
            {"name": "load_state", "type": "string", "required": True},
        ]
    }

    def __init__(self, landscape: "Landscape"):
        super().__init__()
        self._landscape = landscape
        return

    def execute(self, inputs: dict, credentials: dict = None) -> dict:
        # Look up the host's device from the landscape (Vol 6 ch.3).
        device = self._landscape.device(inputs["host_name"])

        # The landscape's per-OS sysman ServiceManager (Vol 4 ch.5,
        # ch.6, ch.7) — the right ServiceManager for the host's OS.
        service_mgr = device.sysman.service_manager

        # Sysman read verb (Vol 4 ch.3 — reads never elevate).
        info = service_mgr.query(inputs["service_name"])

        return {
            "host_name": inputs["host_name"],
            "service_name": inputs["service_name"],
            "active_state": info.active_state,
            "load_state": info.load_state,
        }
```

The tool is *thin*.  It converts agent-level inputs to
sysman-level calls and back.  The actual work — querying
the service via SSH, parsing systemctl output —
happens in the sysman tier (Volume 4) over the interop
wire (Volume 5).

A typical agent ships *dozens* of such tool wrappers,
covering the sysman surface the agent needs.  The
wrappers are *application-specific*; the SDK provides the
patterns; the application chooses which capabilities to
expose.

### Tools that don't elevate vs tools that do

A subtle point worth pinning down.  Volume 4's contract
discipline says *reads never elevate; writes elevate*.
The agent's tool design respects this:

- Read tools (`GetServiceStatusTool`) construct a
  `ServiceManager` with `config.elevated=False` (the
  default).
- Write tools (`RestartServiceTool`) construct a
  `ServiceManager` with `config.elevated=True` *only when
  the agent has been granted authority*.

Authority comes from the agent's
configuration — typically a per-run flag like
`autonomous_actions=False` (default) that the operator
overrides for specific high-confidence runs.

## The workflow

The agent's workflow handles an incident in five steps:

```python
def _build_workflow(self) -> WorkflowDefinition:
    return WorkflowDefinition(
        name="incident_triage",
        steps=[
            WorkflowStep(step_id="gather_context",
                         step_callable=self._step_gather_context),
            WorkflowStep(step_id="run_diagnostics",
                         step_callable=self._step_run_diagnostics),
            WorkflowStep(step_id="assess_severity",
                         step_callable=self._step_assess_severity),
            WorkflowStep(step_id="propose_action",
                         step_callable=self._step_propose_action),
            WorkflowStep(step_id="execute_or_escalate",
                         step_callable=self._step_execute_or_escalate),
        ],
        edges=[
            WorkflowEdge(from_step="gather_context", to_step="run_diagnostics"),
            WorkflowEdge(from_step="run_diagnostics", to_step="assess_severity"),
            WorkflowEdge(from_step="assess_severity", to_step="propose_action"),
            WorkflowEdge(from_step="propose_action", to_step="execute_or_escalate"),
        ],
    )
```

Linear DAG.  Five steps, one path through.  Each step:

1. **`gather_context`** — query Volume 6's landscape for
   the host's group, role, recent events.  Pull recent log
   entries via Volume 5's SSH wire.
2. **`run_diagnostics`** — run targeted diagnostic
   commands via Volume 4's sysman feature managers.
3. **`assess_severity`** — call the LLM with the gathered
   context; LLM returns a severity score 1–5.
4. **`propose_action`** — call the LLM with the severity
   and context; LLM proposes specific actions.
5. **`execute_or_escalate`** — branch on severity and
   the agent's authorisation:
   - Severity ≤ 2 with autonomous_actions=True →
     execute via sysman write verbs.
   - Severity ≥ 3 OR autonomous_actions=False →
     escalate via approval gate.

The workflow is *deterministic* in shape — same five steps
every run.  The variation lives in the LLM responses
(severity score, proposed action) and the conditional in
step 5.

### A worked step — `gather_context`

```python
def _step_gather_context(self, context: WorkflowContext) -> dict:
    """
        Step 1 — Gather context for the incident.
    """
    incident = context.get("incident")
    host_name = incident["host_name"]

    # Pull recent log entries.
    log_tool = self._tool_registry.lookup("read_recent_logs")
    log_result = self._tool_executor.execute(
        log_tool,
        inputs={"host_name": host_name, "lines": 200},
    )

    # Pull host metadata from the landscape.
    device = self._landscape.device(host_name)

    # Pull recent deployments from the team's deployment service.
    deployment_tool = self._tool_registry.lookup("recent_deployments")
    deploy_result = self._tool_executor.execute(
        deployment_tool,
        inputs={"host_name": host_name, "since": incident["alert_timestamp"]},
    )

    context.set("logs", log_result.output["lines"])
    context.set("device_info", {
        "host": device.host,
        "os": device.os,
        "role": device.role,
    })
    context.set("recent_deployments", deploy_result.output["deployments"])

    return {"context_gathered": True}
```

Three tool calls, three pieces of context, all stored in
the workflow's `context` for downstream steps.  The
*method* on the agent class wires the tools through the
executor; the *workflow* sequences the methods.

### A worked step — `execute_or_escalate`

```python
def _step_execute_or_escalate(self, context: WorkflowContext) -> dict:
    """
        Step 5 — Either execute the proposed action or escalate.
    """
    severity = context.get("severity")
    proposed_action = context.get("proposed_action")
    autonomous_threshold = self._configuration.autonomous_threshold

    if severity <= autonomous_threshold:
        # Execute autonomously.
        return self._execute_action(proposed_action, context)
    else:
        # Escalate via approval gate.
        return self._escalate_for_approval(proposed_action, context)


def _execute_action(self, action: dict, context: WorkflowContext) -> dict:
    """
        Run the proposed action through sysman feature managers.
    """
    device = self._landscape.device(action["host_name"])
    if action["type"] == "restart_service":
        # Sysman write verb (Vol 4 ch.5/ch.6/ch.7 ServiceManager).
        result = device.sysman.service_manager.restart(action["service_name"])
        return {"action_executed": True, "sysman_result": result}
    # ... other action types ...


def _escalate_for_approval(self, action: dict, context: WorkflowContext) -> dict:
    """
        Create an approval request and wait for human resolution.
    """
    request = self._approval_gate.create_request(
        execution_id=context.execution_id,
        step_id="execute_or_escalate",
        description="Approve action: {} on {}".format(
            action["type"], action["host_name"]
        ),
        requesting_agent_id=self.identity.identifier,
        metadata={"proposed_action": action, "context": context.as_dict()},
    )

    # Notify on-call (out-of-band — Slack, email, etc.).
    self._notify_oncall(request)

    # Block on resolution.
    resolution = request.wait_for_resolution(timeout=900)  # 15 min

    if resolution == "approved":
        return self._execute_action(action, context)
    elif resolution == "rejected":
        return {"action_skipped": True, "reason": request.rejection_reason}
    elif resolution == "expired":
        return {"action_skipped": True, "reason": "approval_timeout"}
```

The branch:

- **Severity ≤ threshold** → autonomous action via sysman
  write verbs.
- **Severity > threshold** → escalate through the
  approval gate to an on-call human.

## The cost tracker in action

Throughout the run, every LLM call records its cost.  At
the end:

```python
def _on_workflow_complete(self, result: WorkflowResult) -> None:
    cost = self._cost_tracker.total_cost_estimate
    self.logger.info(
        "Triage complete. Cost: ${:.4f} ({} prompt + {} completion tokens)".format(
            cost.total_cost,
            self._cost_tracker.total_token_usage.prompt_tokens,
            self._cost_tracker.total_token_usage.completion_tokens,
        )
    )
```

A typical incident triage might use:

- ~3,000 prompt tokens (system prompt + gathered context).
- ~500 completion tokens (severity assessment).
- ~3,500 prompt tokens (system prompt + action proposal).
- ~1,000 completion tokens (proposed action).

Total: ~8,000 tokens per incident, ~$0.05 at typical
GPT-4-class pricing.  At 10 incidents per day, $0.50 per
day, $15 per month per agent.

The cost is *visible*.  The team's monthly review pulls
totals from the cost tracker (perhaps aggregated to a
dashboard via the metrics layer from chapter 6).

## Observability — recording for replay

Every event in the run is captured by the
`ExecutionRecorder`:

- Lifecycle transitions.
- Workflow step boundaries.
- Tool calls (started / completed / failed).
- LLM calls (started / completed / failed).
- Approval gate events.
- The agent's own custom events (e.g., "alerts received,"
  "incident closed").

After the run, the recording is *persistent*.  A debugger
loads the recording via `ExecutionReplayer` and walks
through it step by step:

```python
recording = ExecutionRecording.load("recordings/triage-20260501-1432.json")
replayer = ExecutionReplayer(recording=recording)

while replayer.has_next:
    record = replayer.next()
    print(f"[{record.sequence_number}] {record.event_type}")
    if record.event_type == "tool.execution.completed":
        print(f"  Tool: {record.payload['tool_name']}")
        print(f"  Duration: {record.payload['duration_ms']}ms")
        print(f"  Output: {record.payload['output']}")
    elif record.event_type == "llm.request.completed":
        print(f"  Provider: {record.payload['provider_name']}")
        print(f"  Tokens: {record.payload['token_usage']}")
```

The replay is *the bridge to Volume 8* (time-travel
debugging).  Volume 8 walks the production-grade
post-mortem tooling that consumes recordings to
reconstruct what happened during a past run.

## Integration with the rest of the framework

Stepping back, this case study touches every drafted
volume:

- **Volume 0** — error vocabulary, configuration,
  application identity (the agent has all of these).
- **Volume 1** — testbase tests for every tool wrapper,
  workflow step, and integration point (chapter 7's mock-
  first discipline applies).
- **Volume 2** — when the agent runs as a distributed
  workload, the tandem-validator pattern from Vol 2 ch.5
  applies for monitoring its behaviour.
- **Volume 3** — taskplus orchestration that *invokes*
  the agent (Vol 3 ch.25's bridge tasks) for scheduled
  triage runs.
- **Volume 4** — every sysman tool wrapper goes through
  Volume 4's contract layer.
- **Volume 5** — every wire call (SSH, REST) goes through
  Volume 5's Measured Interoperability discipline.
- **Volume 6** — the landscape provides the device
  metadata; the topology provides the cluster membership
  the agent reasons about.
- **Volume 7 (this volume)** — the SDK ties them all
  together.

The case study is the volume's *proof* that the SDK's
patterns compose with the rest of the portfolio.  An
agent that's built on the SDK gets:

- Predictable tool dispatch (chapter 3's
  validate-resolve-execute-validate).
- Predictable LLM routing with failover (chapter 2).
- Predictable memory and conversation handling (chapter 4).
- Predictable workflow execution (chapter 5).
- Predictable observability (chapter 6).
- Predictable testing through testbase mock patterns
  (chapter 7).
- Predictable property-test coverage (chapter 8).

Eight predictabilities.  An agent built outside the SDK
has to invent each.  Building on the SDK means inheriting
each.

## What this volume accomplished

Looking back across the nine chapters:

- **Chapter 1** — SDK vs framework, four production-agent
  concerns, the agent state machine.
- **Chapter 2** — multi-provider LLM abstraction with
  failover.
- **Chapter 3** — tool design with custom schema
  validation.
- **Chapter 4** — memory architecture with three scopes
  and semantic search.
- **Chapter 5** — DAG-based workflow with approval gates
  and circuit breakers.
- **Chapter 6** — three observability concerns: tracing,
  metrics, replay.
- **Chapter 7** — 305-test mock-first testing discipline.
- **Chapter 8** — 72-property-test invariant coverage.
- **Chapter 9** — the case study tying it all together.

A reader who's read the volume can:

- Build a production agent on the SDK.
- Choose which subsystems they need; the SDK is composable.
- Test the agent with the same testbase machinery the rest
  of the framework uses.
- Observe the agent's behaviour through the standard stack.
- Integrate the agent with the rest of the
  `codecraftai-*` portfolio.

## End of Volume 7

The series's largest volume by source-material size and
test-suite scale.  The agent-kit is ~46k LOC, ~217
modules, 305 test files.  The volume's nine chapters
covered the SDK's structure end-to-end, with a worked
case study as the payoff.

The chapters that come next are in the volume that follows.
**Volume 8** walks **time-travel debugging in production**
— the production-grade post-mortem tooling that consumes
the `ExecutionRecording` artifacts this volume's chapter 6
introduced.  Where Volume 7 said "every run produces a
recording," Volume 8 says "here's what to do with the
recordings when something goes wrong."

Volume 7's contribution to the series is the *AI-agent
discipline*.  The SDK's structure — base classes, custom
schemas, structured exceptions, mock-first tests, property
tests at scale — is the framework's answer to "how do you
build production AI agents that don't surprise you?"
Every other volume's contribution to the agent's
correctness is mediated through this volume's patterns.
