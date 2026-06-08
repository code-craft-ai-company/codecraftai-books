# Chapter 7 — Case Studies

> **Abstract.** A case study is an honest answer to "what does
> using this *actually look like*?"  This chapter walks three
> debugging scenarios that exercise the time-travel pod / server
> tier against problems readers of the earlier volumes should
> recognise: a flaky distributed test (cross-referencing
> Volume 2's tandem testing model), an agent execution that
> behaved unexpectedly (cross-referencing Volume 7's
> `ExecutionRecorder`), and a sysman feature that misbehaved on
> a remote host (cross-referencing Volume 4's RHEL / Ubuntu /
> macOS / Windows implementations).  Each case study traces a
> path from "the symptom appeared" to "the recording answered
> the question", with the code anchors that drove each
> decision.  All three are built from the source as committed —
> no fictions, with the protocol mismatch from ch. 3 §3.8
> understood as the working assumption.

## 7.1 Case study 1 — The flaky distributed test

### The setup

A team running Volume 2's tandem distributed test framework
(`codecraftai-testplus` driving `codecraftai-pod-tcp`) has a test
that fails intermittently in CI.  The test exercises a
distributed system: a *driver* Pod (capital P, the test-
execution unit from Volume 2) sends a request through a
*frontend* observer Pod, which forwards to a *backend*
observer Pod, which writes to a database; the test asserts on
the database state.

Symptom: about one run in twenty, the assertion fails with
"row not found".  Twenty runs out of twenty pass when the test
is invoked locally; only the CI environment shows the failure.
Standard logs say only that the assertion failed.  Adding more
log lines to either Pod helps localise the assertion-time
state but does not show the *moment* the row failed to write.

### The debugging hypothesis

The team's hypothesis: a race between the database write and
the driver's read.  The test asserts immediately after calling
the frontend; the assertion runs against the database before
the backend has finished its write.

The hypothesis is testable.  If they had a recording of the
backend Pod's write path during a failing run, they could see
whether the write committed before the driver's read started.

### Instrumenting

The test fixture is updated.  In the backend Pod's request
handler, an existing portal `db.transaction.commit` is
known to the developer team — it brackets the database
commit:

```python
with TimeTravelPortal("db.transaction.commit"):
    session.commit()
```

The portal is *registered* but not *active* on the running
backend Pod.  The team adds a snippet to their CI test setup
that activates the portal on each test run via the JSON-lines
remote-control protocol (ch. 5 §5.2):

```python
import socket, json

def activate_db_commit_portal(pod_host: str, pod_port: int,
                              token: str) -> None:
    request = {
        "command": "activate_portal",
        "params": {
            "name": "db.transaction.commit",
            "once": True,
            "capture_id": f"file:///tmp/ci-debug-{run_id}.bin",
        },
        "token": token,
    }

    request_bytes = (json.dumps(request) + "\n").encode("utf-8")

    with socket.create_connection((pod_host, pod_port), timeout=5.0) as sock:
        sock.sendall(request_bytes)
        response_line = b""
        while b"\n" not in response_line:
            response_line += sock.recv(4096)

    response = json.loads(response_line.split(b"\n", 1)[0])
    assert response.get("status") == "ok", response
    return
```

A single round-trip activation: create a TCP connection to
the backend Pod's `9876`, send the `activate_portal` JSON
request, read one line of response, assert the response is
`ok`.  Configured with `once=True` so the portal deactivates
itself after the next call exits — no ongoing overhead, just
the one recording.

The capture path encodes the CI run ID so concurrent test runs
don't overwrite each other's recordings.  The pod's portal
activation
(`codecraftai-timetravel-pod/source/packages/ccai/timetravel/pod/portalregistry.py:25-41`)
stores the capture URI; the next entry into the
`db.transaction.commit` scope routes the recording there
(via `portal.py:187-250`).

### The first failed run with instrumentation

CI runs the test 20 times.  19 pass.  Run 16 fails — and
because of `once=True`, the recording captured only that
failing run's data.

The recording, off-line, walked through the pod-side
`reader.py`:

```python
from ccai.timetravel.pod.reader import TraceFileReader
from ccai.timetravel.pod.timeline import TimelineBuilder

reader = TraceFileReader("/tmp/ci-debug-16.bin")
events = list(reader.read_events())

builder = TimelineBuilder()
builder.add_events(events)
timeline = builder.build()

for ev in timeline:
    print(ev.timestamp_ns, ev.__class__.__name__, getattr(ev, "function_name", ""))
```

The output is a sequence of `CallEvent` and `ReturnEvent`
records, each timestamped in nanoseconds since the gateway's
start.  The team filters to function-level events under the
backend's commit codepath:

```
142_300_000  CallEvent      Session.commit
142_300_500  CallEvent      Session.flush
143_100_000  CallEvent      Connection.execute
143_300_000  CallEvent      Connection.commit
143_400_000  ReturnEvent    Connection.commit
143_410_000  ReturnEvent    Connection.execute
143_420_000  ReturnEvent    Session.flush
143_430_000  ReturnEvent    Session.commit
```

The portal's outermost call ran from 142.3 ms to 143.43 ms
into the gateway-relative timeline.

### Cross-correlating with the driver

The driver Pod runs its assertion check.  The driver Pod's
own gateway timestamps its events relative to its *own* start
— different from the backend's.  But the driver wrote a log
line "asserting database row exists at request_id=R-XXXX" at
wall-clock time `2026-04-26T14:22:05.213Z`.  The backend's
gateway recorded a `GatewayStartEvent` with
`wall_clock_time = 2026-04-26T14:22:05.099Z`
(`pod/traceevents.py:162-174`).

Subtracting: the driver's assertion ran 114 ms after the
backend's gateway started.  But the recording shows the
backend's `Session.commit` only *returned* 143.43 ms after
gateway start.

**The driver's assertion ran 29 ms before the backend's commit
returned.**

The race is confirmed.  The driver was reading before the
backend had finished writing.

### The fix

The hypothesis pointed at a missing synchronisation.  The
driver's request was issued, the frontend was returning *before
the backend's commit* — likely because the frontend's response
was triggered by the queue handoff, not the database commit.

The fix is application-side: the backend's request handler
should not signal completion until the database commit returns.
Implementation in the team's app code, not in the pod.

The pod's role was diagnostic: it gave the team the timeline
that showed the race.  Without it, they had only "row not
found" intermittently and a hypothesis they could not
distinguish from "intermittent network glitch" or "GC pause".
With it, they had the nanosecond-precise timeline and the call
graph that pinned the cause.

### What this case shows about the design

- **Once-shot activation** is the right granularity for a CI
  flake.  The team did not want a continuous recording of every
  CI run; they wanted *the next failing run*, recorded.
  `once=True` makes that the default.
- **Capture-id per run** is what enables CI fan-out.  Twenty
  concurrent test runs each had their own activation with
  their own capture path; no recording was overwritten.
- **The control plane is scriptable.**  The 30-line snippet
  above is the entire integration with CI.  No SDK to vendor,
  no plugin to write — a single TCP call with a JSON request.
- **The pod's reader is internally consistent** (ch. 3 §3.10).
  The team did not need the server to make sense of the
  recording — they read the file directly.  This is the
  "small-scale" debugging path: file-backend, off-line read.
  For larger-scale recording aggregation across many CI runs,
  the server tier would land later.

### Cross-reference to Volume 2

Volume 2 ch. 13 walked the *single-actor + many-observers*
shape of distributed tests in the tandem model.  This case
study is one tier up: the Pod (capitalised) running the
backend service is the *actor*; the time-travel pod
(lower-case) inside it is one of the observers.  The
recording's role is exactly the role Volume 2 assigns to
observers: *to observe the actor's state without changing it*.
The portal does its work *bracketing* the application's actual
behaviour — it does not modify the commit, the database, or
the response; it captures what they did and produces an
artefact the analyst reads later.

The book's recommendation made plain: in Volume 2's distributed
test scenarios, every long-running observer Pod should expose
named portals at every cross-actor boundary — the points where
*this Pod observes the actor's state*.  When a flaky test
fails, those portals are the operator's first switch.

## 7.2 Case study 2 — The agent that skipped a tool

### The setup

A team running an agent built on Volume 7's
`codecraftai-agent-kit` has a customer support bot.  The bot
should call the `lookup_order` tool first, then call
`generate_response` second, with the order data threaded through.

Symptom: about one conversation in fifty, the bot calls
`generate_response` *without first calling* `lookup_order`,
and the response is generic instead of order-specific.  The
team can see the symptom in their logs (the model output is
visibly wrong); they cannot see *why* the model made the
choice.

### The agent's recording posture

The agent's lifecycle is already instrumented with portals from
its declaration in Volume 7 ch. 6:

- `agent.run` brackets the entire agent invocation.
- `agent.tool.dispatch` brackets each tool call.
- `agent.message.compose` brackets the model's final response
  composition.

The agent's recording posture is `calls only` by default
(`trace_calls=True`, others false).  Cheap.  All three
portals are registered; none is currently active.

### The hypothesis

Two competing hypotheses:

1. The model's tool-selection step somehow returned a
   `compose` action before completing the `lookup_order`
   action.  Bug at the model layer — possibly a malformed tool
   schema, possibly a model hallucination.
2. The tool-selection logic on the SDK side has a code path
   that bypasses `lookup_order` under certain conditions.  Bug
   in the agent SDK.

A recording can distinguish.  If `agent.tool.dispatch` was
called for `lookup_order` and *returned an error*, the model
was given a poison-pill response and skipped to compose
(hypothesis 2).  If `agent.tool.dispatch` was *never called*
for `lookup_order`, the model never asked (hypothesis 1).

### Instrumenting

The team activates the three portals on the production agent.
This time they need *variable capture* on the
`agent.tool.dispatch` portal — they need to see the *tool
arguments*, which means line-and-variable granularity on that
portal.

Variable capture is configured at gateway construction;
because of the spec gap from ch. 5 §5.3 the team cannot
hot-flip a running pod's trace configuration.  Their workaround
is to deploy a configuration-update through the standard CI/CD
path:

```python
TimeTravelConfig(
    active_portals=["agent.run", "agent.tool.dispatch", "agent.message.compose"],
    trace_calls=True,
    trace_lines=True,
    trace_variables=True,
    record_inner_portals=True,
    backend_uri="tcp://timetravel.internal:9090",
)
```

This is the operational reality §5.3 describes: *granularity is
set at startup*.  The team accepts the deploy and rolls out the
new configuration to canary, then to production.

### The recording

A few hours after rollout, two more flake instances are
recorded.  The team queries the time-travel server for the
trace sessions that bracket the failing conversation IDs (the
agent logs the conversation ID at the start of each turn; the
team correlates).

The recording for one failure shows, on the call-graph
reconstruction
(`tracedata/call_graph.py:32-62`):

```
agent.run [duration: 1.842s]
├─ agent.tool.dispatch [duration: 0.009s]
│  └─ ToolRegistry.dispatch [duration: 0.009s]
│     ├─ ToolRegistry._lookup [duration: 0.001s]
│     └─ ToolRegistry._invoke [duration: 0.005s]
│        └─ <return: Exception("Connection refused")>
└─ agent.message.compose [duration: 0.420s]
   └─ ModelClient.complete [duration: 0.417s]
```

The `lookup_order` tool *was* invoked.  It returned an
*Exception* (`Connection refused`) at the
`ToolRegistry._invoke` boundary.  The agent SDK's response: it
fell back to `compose` *with the exception in scope*.

The variable snapshot at the `compose` portal entry confirms.
The team queries the variable history
(`tracedata/variable_tracker.py:39-55`) for the `tool_results`
variable at the timestamp of `agent.message.compose` entry:

```python
{
    "lookup_order": {
        "status": "error",
        "error": "Connection refused",
        "fallback_invoked": True,
    }
}
```

The model received a tool-result that said "lookup_order
failed; fallback invoked".  The model — correctly, on its
training — generated a generic response.

### The fix

The bug is in the order-lookup service, not the agent.  The
service has an intermittent connection issue that the team
had not yet correlated with this symptom.  The service team
investigates separately; the agent team adds a retry on
`ToolRegistry._invoke` for transient errors before falling
back to the model.

### What this case shows

- **Variable capture is the high-fidelity option.**  The
  call-only recording would have shown the exception was
  raised and that compose was called next.  The variable-
  capture recording showed *what the model received as its
  prompt*, which was the answer to the actual question
  ("did the model see the failure or did the SDK silently
  proceed?").
- **The deploy-time-only granularity choice is a real
  cost.**  The team had to roll out a configuration change to
  enable variable capture.  The spec gap from ch. 5 §5.3
  is felt operationally: a runtime-flippable granularity would
  have closed the diagnostic loop in minutes instead of hours.
- **The call graph is the right view for the agent path.**
  The team did not care about every line that executed; they
  cared about the *shape* of the call graph — what was
  invoked, in what order, with what return value.  The call-
  graph builder (`tracedata/call_graph.py`) is the analysis
  module that answers this question.

### Cross-reference to Volume 7

Volume 7 ch. 6 introduced `ExecutionRecorder` /
`ExecutionReplayer` as the agent-specific recording mechanism.
This case study shows the *generalisation* in action: instead
of a special-purpose recorder for agent runs, the agent's
lifecycle is bracketed by ordinary portals
(`agent.run`, `agent.tool.dispatch`, `agent.message.compose`),
recorded by the ordinary pod, ingested by the ordinary
server, queried by the ordinary analysis tier.

The benefit of generalisation: an agent's debugging path is
the same as any other production debugging path.  The
operations team has one toolset for "production debugging" and
agent-driven services are not a special case.

The cost of generalisation: the agent-specific affordances
the SDK could provide (richer model-output capture, tool-
schema-aware filtering, dedicated agent-run views) are not
yet in the pod.  The book recommends them as future work that
sits comfortably in the agent SDK rather than in the pod.

## 7.3 Case study 3 — The sysman feature that didn't apply

### The setup

A team running Volume 4's cross-platform sysman tier has a
feature that *applies a firewall rule* on RHEL hosts.  The
feature is implemented in `codecraftai-sysman-rhel` and consumed
through the standard sysman client.  The implementation
shells out to `firewalld`'s `firewall-cmd`.

Symptom: on one specific RHEL 9 host out of a fleet of 200,
the firewall rule does not apply.  The sysman feature reports
success.  An out-of-band check
(`firewall-cmd --list-all`) shows the rule is not in place.

### The hypothesis

The action's stdout said success.  The action's stderr was
empty.  The exit code was zero.  The host's `firewalld` is
running.  But the rule is not present.

A reasonable hypothesis: the action is shelling to the
*runtime* `firewall-cmd`, but the rule is not being persisted
through `firewalld`'s config; the next reload would lose it.
But that does not explain why a `--list-all` immediately after
the action shows no rule — there has been no reload in between.

A second hypothesis: there is a permissions issue.  The action
is running as a service account that *thinks* it can call
`firewall-cmd`, but the call is being silently dropped by
PolicyKit.

### Instrumenting

The sysman feature on the affected host has portals at every
boundary:

- `sysman.feature.firewall.enable`
- `sysman.action.run`
- `sysman.action.exit`

The team activates them with variable capture.  The
`sysman.action.run` portal brackets the actual `subprocess.run`
call; the variable snapshot at line-level captures the command,
arguments, environment, and the resulting `CompletedProcess`.

Activation, this time, goes through the time-travel server's
REST surface:

```
POST /processes/{host_id}/portals/sysman.action.run/activate
  payload: {"capture_id": "file:///var/log/ccai/firewall-debug.bin"}
```

(In the implementation as committed, this endpoint returns mock
data.  The team's workflow uses the `CommandDispatcher`
directly via a server-side script — not the REST API tier —
demonstrating the §6.9 honesty point: the *services* are real;
the REST wrappers are scaffold.)

### The recording

A scheduled re-application of the firewall rule fires.  The
recording captures the action.  The team reads the file:

```python
from ccai.timetravel.pod.reader import TraceFileReader

reader = TraceFileReader("/var/log/ccai/firewall-debug.bin")
events = list(reader.read_events())

# Walk the variable snapshots inside sysman.action.run
in_action = False
for ev in events:
    if isinstance(ev, PortalEnterEvent) and ev.portal_name == "sysman.action.run":
        in_action = True
    elif isinstance(ev, PortalExitEvent) and ev.portal_name == "sysman.action.run":
        in_action = False
    elif in_action and isinstance(ev, VariableSnapshotEvent):
        if "completed_process" in ev.local_variables:
            cp = ev.local_variables["completed_process"]
            print(f"@ {ev.timestamp_ns} cmd={cp.args} rc={cp.returncode}")
            print(f"  stdout={cp.stdout!r}")
            print(f"  stderr={cp.stderr!r}")
```

The output:

```
@ 4_512_000_000 cmd=['firewall-cmd', '--add-rich-rule=...', '--permanent']
  rc=0
  stdout=b'success\n'
  stderr=b''
```

A subsequent variable snapshot, just inside
`sysman.feature.firewall.enable`, shows the team's *own*
verification step:

```
@ 4_530_000_000 verified=True
```

The action ran, returned success, and a verification step
*also* reported success.  But the rule is not on the host.

### The clue in the next snapshot

Looking past the immediate action, the next portal that fires
is `sysman.feature.firewall.persist` — a different portal, in
the *next* feature in the chain.  Its variable snapshot shows:

```
@ 4_580_000_000 cmd=['firewall-cmd', '--reload']
  rc=0
  stdout=b'success\n'
  stderr=b''
```

A reload, immediately after the action.

The team checks the host's
`/etc/firewalld/zones/public.xml`.  The rich-rule the action
added at 4.512 s is *not* in the file at the time of reload.

### The bug

The issue is in the team's automation:

The `--permanent` flag does *not* apply the rule to the
running configuration.  It writes the rule to the
*permanent* configuration files.  A subsequent reload reads
those files and applies them.

The team's verification step ran at 4.530 s.  At that moment:
- The runtime configuration had not changed (no `--add-rich-
  rule` without `--permanent`).
- The permanent configuration files *had* been updated, but
  the reload that applies them had not yet run.
- The verification step checked the *runtime* configuration,
  saw the rule was absent, and — because of a default in the
  team's verifier — assumed *true* if no contradicting evidence
  was found.

A bug compounded by a bug:
- The action used `--permanent` only, not `--permanent
  --add-rich-rule + a separate runtime call`.  This is a
  documented `firewalld` gotcha.
- The verifier's default-true was incorrect.  It should have
  required positive confirmation.

### The fix

The action is updated to call `firewall-cmd --add-rich-rule`
*twice*: once with `--permanent`, once without.  The verifier
is updated to require positive confirmation with an explicit
"rule found" predicate.

### What this case shows

- **Variable capture made the bug visible.**  Without the
  `CompletedProcess` snapshots, the team had only the action's
  reported success and the verifier's reported success — both
  of which were positive — and a host that did not behave as
  the reports claimed.
- **Multi-portal recording provides correlation.**  The action
  portal showed the action's behaviour; the persist portal
  showed the reload's behaviour; the gap between them showed
  the bug was the *runtime configuration not yet updated when
  the verifier checked it*.
- **The recording made the documentation point obvious.**
  Once the team saw the call sequence, the `firewalld`
  documentation about `--permanent` was the immediate next
  read.  The recording did not solve the bug; it pointed
  precisely at the place the documentation needed to be
  consulted.

### Cross-reference to Volume 4

Volume 4's RHEL chapter (ch. 4) walked the
`codecraftai-sysman-rhel` package, including the deliberate
deviation from Convention 4: RHEL-specific exception types are
*not* prefixed with the platform name, because the package's
contract is the OS-specific implementation of a sysman feature
and the platform context is implicit.

This case study exercises that implementation.  The portal
boundaries used here (`sysman.feature.firewall.enable`,
`sysman.action.run`, `sysman.action.exit`,
`sysman.feature.firewall.persist`) are the *generic* sysman
boundaries — the same names appear on the Ubuntu, macOS, and
Windows implementations.  A bug on Ubuntu would surface
through the same portal names with platform-specific
variable contents.  The pod abstracts away the platform-
specific commands; the portal vocabulary is consistent across
the cross-platform tier.

## 7.4 What the three case studies have in common

Pulling the three together, three patterns repeat:

**Static instrumentation of architectural boundaries was
sufficient.**  In all three cases, the portals the team
activated were already in the source code, declared at a
boundary the developer had already identified as
debugging-worthy: the database commit boundary, the agent
tool-dispatch boundary, the sysman action-run boundary.  No
ad-hoc instrumentation was added during the incident.

**The right granularity was case-by-case.**  The flaky test
needed only `calls`; the agent needed `lines + variables`;
the sysman feature needed `lines + variables`.  The
calls-only recording in case 1 was inexpensive and left no
operational footprint.  The variable-capture recordings in
cases 2 and 3 cost an order of magnitude more in volume.
Both choices were appropriate for the question being asked.

**Off-line analysis dominated.**  In every case, the operator
read the recording *after* the symptom had passed.  This is
the time-travel debugging mode the volume's title names: the
symptom occurred, the recording captured the state, the
operator walked the recording to understand what happened.
Replay-during-incident was not the workflow; *post-mortem
state reconstruction* was.

## 7.5 What the three cases revealed about the design

Each case lit a specific design seam.

**Case 1 lit `once=True` activation.**  The CI flake workflow
demanded a single recording, not a continuous stream.  The
flag exists, is honoured, and is the right granularity for
*next failure occurrence* debugging.

**Case 2 lit the spec gap on runtime configuration.**  The
agent team's variable-capture decision required a deploy.
The chapter has been honest: until `update_configuration`
lands on the JSON-lines protocol (ch. 5 §5.3), this is the
operational reality.

**Case 3 lit the multi-portal correlation use case.**  The
firewalld bug only became visible when the team correlated
*two* portals' recordings — the action portal showed the
intent, the persist portal showed the side effect, the gap
between them was the bug.  The pod-side `timeline.py`
(`pod/timeline.py:35-71`) and server-side `timeline.py`
(`tracedata/timeline.py:35-63`) both support multi-source
timeline merging via `heapq.merge`; this is the algorithmic
foundation for cross-portal correlation.

## 7.6 Bridging to the next chapter

The three case studies are *qualitative* — they describe what
the recording revealed and how the team used it.  The remaining
question is *quantitative*: how much overhead does each
granularity actually impose?  How much storage does a recording
consume?  When does the cost cross a threshold that makes the
recording impractical?

The next chapter measures.  It quotes the runtime-overhead
budget for each granularity, the storage budget for each
typical use case, and the policy framework for *when to
enable, when to disable*.

→ Continue to [`08-performance-storage-tradeoffs.md`](08-performance-storage-tradeoffs.md).

← Back to [`06-session-management-at-scale.md`](06-session-management-at-scale.md).
