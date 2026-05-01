# Chapter 5 — Remote Debug Control

> **Abstract.** A pod that ships traces but cannot be controlled
> from outside is a recording device with the *record* button
> wired in advance.  The remote-control plane is what lets an
> operator (or, increasingly, an AI agent) reach into a running
> pod, list the portals it knows about, activate or deactivate
> them, and adjust trace configuration *while the application
> is running, without a restart, without a deploy, without a
> code change*.  This chapter walks the JSON-lines protocol on
> the pod side, the credential-and-handshake model that gates
> it, the registration / heartbeat flow that the server uses to
> track liveness, the MCP server that exposes the same control
> surface to AI agents, and a real spec-vs-implementation gap
> the book treats honestly.

## 5.1 Two control surfaces, not one

A pod listening for control instructions exposes *two*
servers, each on its own port, each speaking a different
protocol:

- The **JSON-lines remote-control server**
  (`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/remote/server.py:23`),
  default port `9876`.  Speaks a small command set
  (list_portals, activate_portal, deactivate_portal) over a
  newline-delimited JSON request/response protocol.  Designed
  to be called by the time-travel server's
  `CommandDispatcher` (ch. 4 §4.10) or by an operator's CLI
  client.
- The **MCP server**
  (`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/remote/mcpserver.py:35`),
  speaks the *Model Context Protocol* — the standard Claude
  agents and other tooling use to discover and call tools.
  Exposes a richer surface (eight tools, including session
  lifecycle and capture queries) over MCP's transport options
  (stdio, TCP, Unix pipe).

Both servers are off by default.  Both are activated by setting
the corresponding flag on `TimeTravelConfig`
(`pod/configuration.py:259-304`); a typical production pod runs
with the JSON-lines server enabled and the MCP server enabled
only in the agent-driven debugging path.

The two surfaces are deliberately distinct.  The JSON-lines
server is a *minimal RPC channel for an operations tool* —
small command set, simple framing, easy to call from a shell
script or a curl-equivalent.  The MCP server is a *richer
toolset for an autonomous agent* — typed inputs, structured
outputs, transport-layer flexibility.  The book walks them in
order.

## 5.2 The JSON-lines protocol

The pod's `RemoteControlServer`
(`pod/remote/server.py:23`) is a background-thread TCP server.
On `start()` it binds to `(host, port)` and spins a worker
thread that runs an accept loop; each accepted connection is
handled synchronously and closed at the end of one
request/response.

The dispatch loop
(`server.py:106-157`) reads a single line, parses it as JSON,
runs a handler, writes one line of JSON back, and closes:

```python
def _handle_client(self, client_socket: socket.socket) -> None:
    try:
        data = b""
        while b"\n" not in data:
            chunk = client_socket.recv(4096)
            if not chunk:
                return
            data += chunk

        line = data.split(b"\n", 1)[0]
        request = json.loads(line.decode("utf-8"))
        response = self._dispatch(request)
        response_bytes = json.dumps(response).encode("utf-8") + b"\n"
        client_socket.sendall(response_bytes)
    except Exception as exc:
        logger.error("Error handling remote control request: %s", exc)
    finally:
        client_socket.close()
    return
```

A request is `{"command": str, "params": dict, "token": str}`.
A response is `{"status": "ok"|"error", "data"|"message": ...}`.

The dispatch step
(`server.py:131-157`) authenticates the request and routes the
command:

```python
def _dispatch(self, request: dict) -> dict:
    # Authentication check
    token = request.get("token")
    if not verify_credential(token, self._credential):
        rtnval = {"status": "error", "message": "Authentication failed"}
        return rtnval

    command = request.get("command", "")
    params = request.get("params", {})

    handler = _COMMAND_HANDLERS.get(command)
    if handler is None:
        rtnval = {
            "status": "error",
            "message": f"Unknown command: {command}",
        }
        return rtnval

    rtnval = handler(self, params)
    return rtnval
```

The command-handlers registry
(`server.py:214-218`) is short:

```python
_COMMAND_HANDLERS = {
    "list_portals": _handle_list_portals,
    "activate_portal": _handle_activate_portal,
    "deactivate_portal": _handle_deactivate_portal,
}
```

## 5.3 The spec gap on configuration

This is the chapter's spec-vs-implementation honesty point.

The server-side specification (`automojo-timetravel-server`'s
`AGENTS.md`, Layer S2.3) declares:

> S2.3 — Trace Configuration Query and Update: Retrieves and
> modifies the trace configuration (granularity, inner-portal
> annotations) on a registered process via its remote-control
> interface (pod F5.1 / pod F0.2).

The pod-side specification (`automojo-timetravel-pod`'s
`AGENTS.md`, Layer F5.1) describes the JSON-lines server in
terms that imply the matching commands:

> F5.1 — Remote Control Server: Background-thread TCP server
> with JSON-lines protocol for listing/activating/deactivating
> portals and querying/updating trace configuration at runtime.

But the implementation
(`pod/remote/server.py:214-218`) registers only three handlers
— `list_portals`, `activate_portal`, `deactivate_portal`.
There is no `get_configuration` handler and no
`update_configuration` handler.

The server-side `CommandDispatcher` will happily *try* to issue
those commands; the pod's dispatch will return
`{"status": "error", "message": "Unknown command: get_configuration"}`.
A `TraceConfigService` on the server constructed to call those
commands does not get a configuration back.

The book takes a position on this gap:

- The *trace configuration model* on the pod
  (`pod/traceconfiguration.py:10-63`) is well-formed,
  immutable, and serialisable.  The data side is ready.
- The *handlers* on the pod are absent — three lines per
  handler would close the gap, plus the dictionary entry.
- The *consumer* on the server is partially-built — the
  `TraceConfigService` exists in the source as a Layer-2
  module, but its end-to-end exercise is blocked on the
  missing handlers.

A future commit will close the gap.  Until then, the
operational reality is: the pod's trace configuration is
*set at startup* via `TimeTravelConfig` (or `configure_timetravel`)
and is *not modifiable at runtime over the JSON-lines protocol*.
A pod that needs to flip from `trace_calls=True` only to
`trace_calls=True, trace_lines=True` requires the operator to
restart the pod with the new configuration.

This is a meaningful operational limitation.  Volume 8 ch. 8
discusses the consequence: the production-debug workflow today
involves *pre-deciding the granularity menu*, not
*hot-adjusting the granularity in response to a bug*.

## 5.4 Authentication

The pod's authentication is intentionally minimal but
correct.  `verify_credential`
(`pod/remote/auth.py:13-35`) is a constant-time token compare:

```python
def verify_credential(provided: Optional[str], expected: Optional[str]) -> bool:
    if expected is None:
        # No credential configured — allow all
        rtnval = True
        return rtnval

    if provided is None:
        rtnval = False
        return rtnval

    rtnval = hmac.compare_digest(provided, expected)
    return rtnval
```

Three observations.

First, `hmac.compare_digest` is the right primitive.  A naive
`==` would leak information about the token via timing — an
attacker iterating over candidate tokens could observe how
deep into the comparison the equality test went and narrow
the search.  `compare_digest` runs in constant time relative to
the lengths.

Second, the credential is *optional*.  If the server is started
with `credential=None`, every request authenticates.  This is
the no-auth mode used in development and in same-host
scenarios where the pod's port is bound to `127.0.0.1` and
exposed only via Unix socket.  In production, the credential
should always be set.

Third, the credential is *in-memory only*.  It is passed at
construction
(`pod/remote/server.py:42`) and stored as
`self._credential`; it is never written to disk by the pod.
Persistence — and rotation, revocation, expiry — lives in the
*server*'s credential store
(`automojo-timetravel-server/source/packages/automojo/timetravelserver/persistence/credential_store.py`),
which the pod's `ServiceRegistration`
(`pod/remote/registration.py`) consumes by negotiating an
*instance credential* at registration time.

The two credentials are distinct:

- The **service credential** authenticates the pod to the
  server's HTTP endpoints (`/register`, `/unregister`,
  `/heartbeat`).  It is a long-lived token the operator
  configures on the pod side and the server side at deployment.
- The **instance credential** authenticates the server's
  command-dispatch traffic *back to the pod*.  It is generated
  per-instance at registration and is what fills `self._credential`
  on the `RemoteControlServer`.

Splitting the two prevents a credential leaked at one layer
from compromising the other.  An attacker who steals a service
credential cannot dispatch commands to a pod's
`RemoteControlServer` — they would need that pod's instance
credential, which is generated freshly per registration and
known only to the pod and the server.

## 5.5 The registration handshake

When a pod starts and `configure_timetravel` finds
`remote_enabled=True` and a `service_url`, the pod registers
itself with the server before accepting any control traffic.

`ServiceRegistration.register`
(`pod/remote/registration.py:65-89`) is a single POST to the
server's `/register` endpoint:

```python
def register(self) -> bool:
    payload = {
        "instance_id": self._instance_id,
        "remote_host": self._remote_host,
        "remote_port": self._remote_port,
        "instance_credential": self._instance_credential,
    }
    try:
        self._send_request("/register", payload)
        self._registered = True
        logger.info(
            "Registered with debugging service: %s (instance: %s)",
            self._service_url, self._instance_id
        )
        rtnval = True
        return rtnval
    except Exception as exc:
        logger.error("Failed to register with debugging service: %s", exc)
        rtnval = False
        return rtnval
```

The payload tells the server:

- Who I am (`instance_id`, a UUID generated at construction).
- Where to reach me (`remote_host`, `remote_port`).
- The credential I'll accept on inbound commands
  (`instance_credential`).

The HTTP request itself is a Bearer-token-authenticated POST
(`registration.py:112-134`):

```python
def _send_request(self, path: str, payload: dict) -> bytes:
    url = f"{self._service_url}{path}"
    data = json.dumps(payload).encode("utf-8")
    request = urllib.request.Request(
        url,
        data=data,
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {self._service_credential}",
        },
        method="POST",
    )
    with urllib.request.urlopen(request, timeout=10) as response:
        rtnval = response.read()
    return rtnval
```

A 10-second timeout, no retries.  If the registration fails,
the pod logs the error and returns `False`; the application is
free to decide whether to retry, fall back to a no-server
configuration, or fail to start.

The single-attempt-no-retry policy is an explicit choice.  An
auto-retry loop in registration would tie pod startup to server
availability — a server outage would block every new pod from
starting.  The pod's operating principle is *never block the
application on the debug-tier's availability*; if the server
is unreachable, the pod runs without server registration and
keeps recording locally to its file backend.  An operator
notices via the pod's log (the WARNING line) and can re-run
registration manually, or wait for the next deploy.

## 5.6 The heartbeat

After registration, the pod sends a heartbeat to the server at
a configurable interval.  `HeartbeatSender`
(`pod/remote/heartbeat.py:21`) runs on a daemon thread:

```python
def _heartbeat_loop(self) -> None:
    while not self._stop_event.is_set():
        self._send_heartbeat()
        self._stop_event.wait(timeout=self._interval)
    return
```

The default interval is 30 s
(`heartbeat.py:18`).  The payload
(`heartbeat.py:84-108`):

```python
def _send_heartbeat(self) -> None:
    payload = {
        "instance_id": self._instance_id,
        "timestamp": time.time(),
    }
    try:
        url = f"{self._service_url}/heartbeat"
        ...
        with urllib.request.urlopen(request, timeout=10) as response:
            response.read()
    except Exception as exc:
        logger.warning("Heartbeat failed: %s", exc)
    return
```

A POST with the instance ID and a wall-clock timestamp.
Authenticated with the same Bearer token registration used.
Failure logs a WARNING and continues — the loop will try again
in `interval` seconds.

On the server side, the heartbeat handler
(`server, registration/heartbeat.py:18`) updates
`last_heartbeat_at` on the `RegisteredProcess` row and flips
`lifecycle_state="active"` if it was `registered` or `stale`.
Volume 8 ch. 4 §4.7 walks the state-machine transitions; the
heartbeat is the *only* signal that drives the
`registered → active` and `stale → active` transitions.

## 5.7 The MCP server

The MCP server
(`pod/remote/mcpserver.py:35`) exposes a richer control
surface to *AI agents* — Claude or another model that speaks
the Model Context Protocol.

Where the JSON-lines server has three handlers, the MCP server
has *eight tools*
(`pod/remote/mcpserver.py:88-232`):

- `create_session` — open a new time-travel debugging session,
  returning a session ID.
- `renew_session` — extend an existing session's TTL.
- `close_session` — finalise a session and clean up its temp
  area.
- `list_captures` — list the captures (recordings) attached to a
  session.
- `query_capture` — query the variable history or call graph
  inside a specific capture.
- `list_portals` — list the portal registry's contents.
- `activate_portal` — activate a portal, optionally with a
  capture-id, expression, or once-flag.
- `deactivate_portal` — deactivate a portal.

The portal-control tools (`list_portals`, `activate_portal`,
`deactivate_portal`) duplicate the JSON-lines server's commands
but are exposed via MCP's structured-tool interface.  An agent
can call them with typed arguments and receive typed responses.

The session-management tools (`create_session`, `renew_session`,
`close_session`, `list_captures`, `query_capture`) are
specific to the MCP path — they are about an *agent-driven
debug session*, where the agent activates a portal, observes
the capture it produced, and asks questions about the
recording.

The `activate_portal` MCP tool
(`mcpserver.py:373-398`) shows the integration with the
portal registry:

```python
elif name == "activate_portal":
    portal_name = arguments.get("name")
    if not portal_name:
        raise ValueError("Missing 'name' argument")
    expression = arguments.get("expression")
    once = arguments.get("once", False)
    capture_id = arguments.get("capture_id")
    session_id = arguments.get("session_id")

    if session_id:
        if not capture_id:
            capture_id = str(uuid.uuid4())

        if not capture_id.endswith(".bin"):
            capture_id += ".bin"

        session_dir = os.path.join(tempfile.gettempdir(), f"automojo_session_{session_id}")
        if not os.path.exists(session_dir):
            os.makedirs(session_dir, exist_ok=True)

        capture_path = os.path.join(session_dir, capture_id)
        capture_id = f"file://{capture_path}"

    registry = get_portal_registry()
    registry.activate_portal(portal_name, expression=expression, once=once, capture_id=capture_id)
    return [{"type": "text", "text": f"Activated portal: {portal_name} with capture_id: {capture_id}"}]
```

When an agent activates a portal in the context of a session,
the MCP server allocates a per-session directory under
`/tmp/automojo_session_<id>/`, generates a capture file path
inside it, and passes the file URI as the portal's `capture_id`.
The portal records into that file; the agent later queries the
file via `query_capture`.

This is the agent-driven debugging workflow:

1. Agent calls `create_session()` → gets a session ID.
2. Agent calls `activate_portal("invoice.charge.attempt",
   session_id=...)` → portal records to a per-session capture
   file.
3. Application runs.  Portal records.
4. Agent calls `query_capture(session_id=..., capture_id=...,
   variable="amount")` → gets the variable history.
5. Agent reasons about the recording, decides whether to
   activate another portal or call `close_session`.

The MCP transport is configurable
(`mcpserver.py:411-574`): stdio (the default for an agent
co-located with the pod), TCP (`start_background_tcp`), or Unix
pipe (`start_background_pipe`).  Stdio is the standard MCP
transport for tools spawned as subprocesses; TCP and Unix pipe
are alternatives for the case where the agent runs in a
separate process and needs network or namespace access.

A 60-second TTL cleanup (`mcpserver.py:61-86`) sweeps idle
session directories.  Sessions that the agent abandons (without
calling `close_session`) leave their temp data on disk for at
most 60 seconds before the cleanup thread removes them.

## 5.8 The dispatcher pattern, in two directions

Volume 5 documented the *Aspects pattern*: behaviour-parameter
objects passed by callers to interop layers.  The
remote-control plane uses the pattern in both directions.

**Server → pod (the dispatcher).**  The server's
`CommandDispatcher`
(`server, debugcontrol/command_dispatcher.py:21-74`) takes
behaviour parameters at construction:

```python
class CommandDispatcher:
    def __init__(self, credential_token: str, timeout_seconds: float = 5.0):
        self.credential_token = credential_token
        self.timeout_seconds = timeout_seconds
```

Two parameters: the token and the timeout.  Different callers
of the dispatcher can configure different timeouts — a portal
discovery query is patient (longer timeout, the pod may be
slow to enumerate); a heartbeat-related ping is impatient
(shorter timeout, the dispatcher should fail fast and let the
state machine flip to `unreachable`).

**Pod → server (registration / heartbeat).**  The pod's
`ServiceRegistration` and `HeartbeatSender` similarly take
configuration at construction.  The retry / backoff parameters
are not exposed (because the design choice was *no retry*) but
the timeout is fixed at 10 seconds in the pod's source — a
slightly different posture from the dispatcher.  Both ends are
honoring the *control reverts via typed exception* contract:
the request either succeeds within the budget or raises a
specific exception that the caller decides what to do with.

## 5.9 Reachability is not delivery

A subtle property of the control plane: a pod's lifecycle
state on the server says how recently the *server* heard from
the *pod*; it does not say whether a *command from the server*
will reach the pod.

The reverse asymmetry is real.  The pod calls the server's
`/heartbeat` over outbound HTTP — usually permitted by
firewalls and load balancers.  The server calls the pod's
JSON-lines port over inbound TCP — frequently *not* permitted
by default firewall configurations.

The gap appears in production when the pod is registered (its
heartbeat made it through), looks `active` to the server, but
the dispatcher's outbound TCP connection times out because no
inbound rule allows traffic to the pod's `9876` port.  The
state-machine view of the world is wrong from the dispatcher's
perspective: the pod is reachable for *outgoing* but not
*incoming*.

The book's recommendation, made explicit: in production, run
the pod's JSON-lines server *behind* an authenticated reverse
proxy that the server can route to (a sidecar pattern is
common — the proxy is the only inbound listener, and it
forwards to the pod's local port over loopback), or
co-locate the pod and the server on the same host and use the
Unix domain socket transport.  The MCP server can additionally
take a stdio transport — the agent and the pod share a process,
and there is no network at all.

The current `RegisteredProcess` schema
(`server, persistence/models/process.py`) does not record an
*inbound-reachable* flag separate from the heartbeat-driven
lifecycle state.  An operator looking at the process registry
sees `lifecycle_state="active"` and can be misled into thinking
the pod is fully reachable, when in fact only the outbound
heartbeat path is.  Adding an "inbound-confirmed" flag —
toggled when the most recent dispatcher call to the pod
succeeded — is a small, useful addition the book flags as a
future improvement.

## 5.10 The control plane in incident mode

A canonical incident workflow ties the chapters together.

**Time T:** a pod is running normally.  Portals are registered
with the registry; nothing is activated.  Trace overhead is
negligible (a few thread-local lookups per portal entry).
Heartbeats arrive at the server every 30 s.  Lifecycle state
is `active`.

**Time T + Δ:** an alert fires.  The operator opens the
operations console.  The console (in the architecture as
specced; in the implementation as scaffolded) calls the
server's REST API: `GET /processes` to find the pod;
`GET /processes/{id}` to load detail; `GET /processes/{id}/portals`
to list portals.  The dispatcher behind these endpoints calls
the pod's `list_portals` command and reconciles with the
local snapshot (ch. 4 §4.11).

**Time T + 2Δ:** the operator sees the
`invoice.charge.attempt` portal.  Activates it via
`POST /processes/{id}/portals/{portal_name}/activate` (REST
shape; the dispatcher calls
`activate_portal` on the pod).  The pod's portal registry
flips the portal active.  Concurrent and subsequent calls
through the portal record to the configured backend.

**Time T + 3Δ:** the bug fires.  A call enters the portal,
the portal evaluates its expression (if one was set), the
portal installs `sys.settrace`, the recording captures the
call's events.  Bytes flow over the pod's TCP backend to
the server's stream receiver (ch. 4); the server's session
appends them to a temp file; the flush marker arrives on
gateway exit (the request body completes, the portal exits,
the gateway is still up so no flush yet — until the operator
manually triggers a flush, OR a one-shot activation
auto-deactivates and triggers cleanup).

**Time T + 4Δ:** the operator is satisfied.  Deactivates the
portal via the same path in reverse.  Future calls into the
portal scope take the cheap path again.  The captured
recording sits in the server's storage; the operator queries
it through the analysis API.

The control plane is what makes this workflow *interactive*.
Every step from "open the operations console" to "deactivate
the portal" is a request through the JSON-lines protocol or
the REST API that wraps it.  The operator does not deploy code,
restart processes, or modify configuration files.  They flip
named portals on and off.

## 5.11 An MCP-driven variant

The same workflow, done by an agent over MCP:

```
agent: > Why is invoice.charge.attempt erroring at elevated rate?
agent: → list_portals(...) → see invoice.charge.attempt is registered
agent: → create_session(...) → session_id = abc123
agent: → activate_portal(name="invoice.charge.attempt",
                         once=True,
                         expression="invoice.amount > 5000",
                         session_id="abc123")
agent: (waits for application traffic)
agent: → list_captures(session_id="abc123") → one capture file
agent: → query_capture(session_id="abc123",
                       capture_id="<id>",
                       query_type="variable_history",
                       variable="amount")
agent: ← [{ts: 142.5, value: 8723.50}, {ts: 142.7, value: 8723.50}]
agent: → query_capture(session_id="abc123",
                       capture_id="<id>",
                       query_type="call_graph")
agent: ← <call graph as structured data>
agent: > the call graph shows the failing path: invoice.charge.attempt
agent: > → stripe.charge → http.post → timeout. Recommend...
agent: → close_session(session_id="abc123")
```

The agent is doing what the operator does, with MCP as the
calling convention.  The same pod, the same recording, the
same query API; a different *interface*.  This is the value of
the two-surfaces design — the pod's underlying capabilities are
the same; the surface the caller uses is whichever one fits the
caller's competence.

A direct cross-reference back to Volume 7: the
`ExecutionRecorder` / `ExecutionReplayer` from Volume 7 ch. 6
is the special-cased version of this — an agent recording its
*own* execution as a single named capture for later
inspection.  In the time-travel pod's vocabulary, the
agent-execution recorder is *one tool that calls the same
underlying mechanism the MCP path generalises*.  The
architectural pattern is consistent across the portfolio.

## 5.12 The retention boundary on the control plane

The control plane has *no* state of its own beyond what the
server's database holds.  A pod's portal registry is in-memory
only; restarting the pod loses the activation state.  A
session created via MCP is tied to a temp directory that is
swept after 60 s of inactivity.  A trace session record on the
server outlives the pod that created it, until the retention
job sweeps it.

This is the right division: ephemeral state is in-memory on
the entity that owns it; durable state is in the server's
database; trace artefacts are on the server's storage backend;
all of it is governed by *retention policy*, which is the
subject of the next chapter.

## 5.13 What the control plane will not do

Three things the control plane is, by design, unwilling to do:

**Permit the server to overwrite the pod's local code.**  The
JSON-lines protocol's commands are bounded — list, activate,
deactivate, configure (in the spec).  There is no `eval`
command.  There is no `inject_code` command.  The pod accepts
control over *which existing portals are active*, not *which
code runs*.  This is a deliberate security boundary: a leaked
instance credential (or a compromised server) should not be
able to execute arbitrary code in the pod's process.

**Permit a portal to be activated in a process where the
portal is not declared.**  A `RemoteControlServer` accepts
`activate_portal("foo")` and writes the activation to the
registry; if no `with TimeTravelPortal("foo"):` block exists in
the application code, the activation has no effect.  The
operator gets no error feedback.  This is a corollary of the
static-instrumentation choice (ch. 2 §2.13): the operator can
only activate what the developer pre-instrumented.

**Permit the trace stream to be redirected at runtime.**  The
backend URI is set at gateway construction
(`pod/configuration.py`) and is not currently mutable via the
control plane.  An operator who wants the trace to flow to a
different endpoint must change the configuration and restart
the pod.  This is consistent with the spec gap on
`update_configuration` (§5.3): the runtime mutability surface
is currently smaller than the spec describes.

## 5.14 Bridging to the next chapter

The control plane is the *interactive* tier — the layer that
lets the operator and the agent reach into the production
process.  The recordings the control plane produces have to
*go somewhere* and have to *be governed*: many concurrent
sessions, finite storage budget, finite replay-bandwidth
budget, a credential lifecycle, retention policy.

The next chapter walks the *operational* tier — the
multi-tenant story, the retention job, the credential
lifecycle, the bandwidth budget that the server's flow control
is one half of.

→ Continue to [`06-session-management-at-scale.md`](06-session-management-at-scale.md).

← Back to [`04-server-ingestion.md`](04-server-ingestion.md).
