# Volume 8 — Code Anchors Index

This file is the index of file:line anchors used by the chapter
drafts.  All claims in the chapters cite paths under
`automojo-timetravel-pod` or `automojo-timetravel-server`.

## Pod — recording engine

| Module | Path | Maturity |
|---|---|---|
| Gateway | `pod/gateway.py` (161 ll.) | mature |
| Portal | `pod/portal.py` (283 ll.) | mature |
| Portal registry | `pod/portalregistry.py` (141 ll.) | mature |
| Portal filter | `pod/portalfilter.py` (42 ll.) | mature |
| Tracing state | `pod/tracingstate.py` (80 ll.) | mature |
| Trace hook manager | `pod/tracehookmanager.py` (138 ll.) | mature |
| Trace recorder ABC | `pod/tracerecorder.py` (44 ll.) | mature |
| Trace events | `pod/traceevents.py` (186 ll.) | mature |
| Trace configuration | `pod/traceconfiguration.py` (100 ll.) | mature |
| Trace clock | `pod/traceclock.py` (74 ll.) | mature |
| Trace serializer | `pod/traceserializer.py` (327 ll.) | mature |
| Buffered writer | `pod/bufferedwriter.py` (61 ll.) | mature |
| Decorators | `pod/decorators.py` (81 ll.) | mature |
| Configuration | `pod/configuration.py` (305 ll.) | mature |
| Reader | `pod/reader.py` (117 ll.) | partial (scan_headers stub) |
| Timeline | `pod/timeline.py` (72 ll.) | mature |
| `__init__` | `pod/__init__.py` | empty (1 ll.) |

### Salient anchors

- **Gateway exit broadcast → flush → close ordering** —
  `gateway.py:73-101`.
- **Portal entry decision tree** (gateway-active? already
  tracing? portal activated? expression evaluates true?) —
  `portal.py:60-141`.
- **Two-tier recorder creation** (real backend chain vs in-memory
  list collector) — `portal.py:187-250`.
- **Portal registry lock** — `portalregistry.py:25-41`.  Every
  public method takes `self._lock` (a `threading.Lock`).
- **Portal info copy-on-read** — `portalregistry.py:71-83`.
- **Thread-local lazy init** — `tracingstate.py:44-55`.  Note that
  `_gateway_active` is process-wide, not thread-local
  (`tracingstate.py:58-79`).
- **Trace hook installation** — `tracehookmanager.py:49-61`
  (`sys.settrace`).
- **Trace function dispatch** (returns `self._trace_function` to
  keep the hook installed across nested calls) —
  `tracehookmanager.py:63-137`.
- **EventType enum values** — `traceevents.py:12-20`.  Nine event
  types, integers 1-9.  Note `EXCEPTION` is type 9.
- **Variable snapshot event shape** —
  `traceevents.py:98-111`.  `local_variables` is a `dict`.
- **Immutable trace configuration** — `traceconfiguration.py:10-63`.
  `__slots__` + read-only properties.
- **Monotonic clock** — `traceclock.py:39-52`.  `RuntimeError` if
  `timestamp()` is called before `start()` — does not silently
  return zero.

### Pod serializer envelope (Layer 1.3 — F1.3)

`pod/traceserializer.py:28-84`:

```
ENVELOPE_FORMAT = "!BQQIII"     # 29 bytes total, big-endian
  B  event_type        (1B,  unsigned char)
  Q  timestamp_ns      (8B,  unsigned long long)
  Q  thread_id         (8B,  unsigned long long)
  I  filename_length   (4B,  unsigned int)
  I  lineno            (4B,  unsigned int)
  I  payload_length    (4B,  unsigned int)

Layout on the wire:
  [envelope 29B] [filename UTF-8 bytes] [pickle payload bytes]
```

Payload pickled at protocol 5.  `LineEvent` has no payload.

### Buffered writer

`pod/bufferedwriter.py:34-43`.  Auto-flush when
`len(buffer) >= max_size`, default 65,536 bytes.  `bytearray`
buffer, `bytes()` copy on flush.

### Decorators

`pod/decorators.py:19-46`.  `@trace_portal("name")` wraps a
function body in `with TimeTravelPortal(name):`.  Branch on
`inspect.iscoroutinefunction()` to handle async.

## Pod — backends

| Module | Path | Maturity |
|---|---|---|
| Backend registry | `pod/backends/registry.py` (123 ll.) | mature |
| File backend | `pod/backends/filebackend.py` (100 ll.) | mature |
| TCP backend | `pod/backends/tcpbackend.py` (127 ll.) | mature |
| Unix backend | `pod/backends/unixbackend.py` (125 ll.) | mature |

- **URI scheme dispatch** —
  `pod/backends/registry.py:35-57`.  Uses `urllib.parse.urlparse`.
- **TCP defaults** — `localhost:9090` if URI omits host/port
  (`pod/backends/registry.py:83-94`).
- **TCP exponential backoff reconnect** —
  `pod/backends/tcpbackend.py:61-83`.  `backoff *= 2`.
- **TCP drop-on-saturation** —
  `pod/backends/tcpbackend.py:85-107`.  No bounded buffer; if
  reconnect fails, data is dropped and an error is logged.
- **File rotation** — `pod/backends/filebackend.py:79-89`.  Default
  100 MB rollover.
- **Unix socket** — `pod/backends/unixbackend.py:49-57`.  Uses
  `settimeout`, not non-blocking.  Permissions inherited from
  filesystem ACL on the socket path.

## Pod — remote control plane

| Module | Path | Maturity |
|---|---|---|
| `__init__` | `pod/remote/__init__.py` | empty |
| Server | `pod/remote/server.py` (219 ll.) | mature, **spec gap** |
| Auth | `pod/remote/auth.py` (36 ll.) | mature |
| Registration | `pod/remote/registration.py` (135 ll.) | mature |
| Heartbeat | `pod/remote/heartbeat.py` (109 ll.) | mature |
| MCP server | `pod/remote/mcpserver.py` (687 ll.) | mature, not in F-spec |

- **JSON-lines dispatch loop** — `pod/remote/server.py:106-157`.
- **Command registry** — `pod/remote/server.py:214-218`.  Three
  handlers only: `list_portals`, `activate_portal`,
  `deactivate_portal`.
  - **Spec gap:** the design spec also lists `get_configuration`
    and `update_configuration` (S2.3 / F0.2) — these are NOT
    implemented in the JSON-lines server.  Honesty point in ch. 5.
- **Constant-time auth** — `pod/remote/auth.py:13-35`.
  `hmac.compare_digest`.  If credential is `None`, all clients
  accepted (no-auth mode).
- **Service registration** —
  `pod/remote/registration.py:65-89`.  POST `/register` with
  `{instance_id, remote_host, remote_port, instance_credential}`.
  Single attempt, no retry.
- **Heartbeat cadence** — `pod/remote/heartbeat.py:75-82`.
  Configurable interval; 30 s default.  POST `/heartbeat` with
  `{instance_id, timestamp}`.  Failures logged but not retried.
- **MCP server** — `pod/remote/mcpserver.py:88-232`.  Eight tools
  (create/renew/close session, list/query captures, list/activate/
  deactivate portal).  Transports: stdio, TCP, Unix pipe.  Session
  directories under `/tmp/automojo_session_<id>` with 60 s TTL
  cleanup.  This bridges pod control to AI agents (Volume 7
  ch. 6 — the agent-driven debug session is what `mcpserver.py`
  enables).

## Server — persistence (Layer S0)

| Module | Path | Maturity |
|---|---|---|
| Engine | `persistence/engine.py` (9 ll.) | mature |
| Credential store | `persistence/credential_store.py` (111 ll.) | mature |
| Server config store | `persistence/server_config_store.py` (75 ll.) | mature |
| Base mixin | `persistence/models/base.py` (12 ll.) | mature |
| Process model | `persistence/models/process.py` (26 ll.) | mature |
| Trace session model | `persistence/models/trace_session.py` (18 ll.) | mature |
| Credential model | `persistence/models/credential.py` (15 ll.) | mature |
| Portal snapshot | `persistence/models/portal_snapshot.py` (16 ll.) | mature |
| Server config model | `persistence/models/server_config.py` (11 ll.) | mature |
| User | `persistence/models/user.py` (13 ll.) | scaffolding (no wiring) |
| Storage backend ABC | `persistence/trace_storage/backend_abc.py` (45 ll.) | mature |
| Filesystem backend | `persistence/trace_storage/filesystem_backend.py` (76 ll.) | mature |

- **Process schema** — `persistence/models/process.py:7-25`.
  Fields: host, pid, application_name, pod_version,
  remote_control_address, lifecycle_state, registered_at,
  last_heartbeat_at, disconnected_at.  `lifecycle_state` indexed.
- **Lifecycle states** — `registered`, `active`, `stale`,
  `unreachable`, `disconnected`.  Transitions in
  `registration/heartbeat.py:67-100`.
- **Trace session schema** —
  `persistence/models/trace_session.py:8-17`.  States: `active`,
  `completed`.  `data_reference` points to backend storage path.
- **Credential schema** — `persistence/models/credential.py:7-14`.
  Stores SHA-256 token hash only.  `revoked_at`, `expires_at` for
  soft revocation.
- **Server config** —
  `persistence/server_config_store.py:7-27`.  Schema-driven typed
  cast: bool/int/str.  Defaults: `listen_port=8080`,
  `data_retention_days=30`, `heartbeat_timeout_seconds=30`.
- **Atomic finalization** —
  `persistence/trace_storage/filesystem_backend.py:32-42`.  Temp
  → final via `Path.rename` (POSIX atomic move).

## Server — registration (Layer S1)

| Module | Path | Maturity |
|---|---|---|
| Service | `registration/service.py` (87 ll.) | mature |
| Heartbeat | `registration/heartbeat.py` (103 ll.) | mature |
| Process registry | `registration/process_registry.py` (93 ll.) | mature |
| Metadata refresh | `registration/metadata_refresh.py` | (not surveyed) |

- **Idempotent register** — `registration/service.py:21-56`.
  `(host, pid)` is the natural key; second call updates.
- **Health checker state machine** —
  `registration/heartbeat.py:67-100`.  Transitions:
  - `active` → `stale` at `T + heartbeat_timeout_seconds`.
  - `stale` → `unreachable` at
    `T + 2 × heartbeat_timeout_seconds`.
- **Pagination + search** —
  `registration/process_registry.py:16-52`.  `ilike` substring on
  application_name, host.  Validates sort column before applying.

## Server — debug control (Layer S2)

| Module | Path | Maturity |
|---|---|---|
| Command dispatcher | `debugcontrol/command_dispatcher.py` (80 ll.) | mature |
| Portal discovery | `debugcontrol/portal_discovery.py` (73 ll.) | mature |
| Portal control | `debugcontrol/portal_control.py` | mature |
| Trace config | `debugcontrol/trace_config.py` | mature |
| Session manager | `debugcontrol/session_manager.py` (89 ll.) | mature |

- **JSON-RPC client** —
  `debugcontrol/command_dispatcher.py:21-74`.  Line-based protocol.
  `{"command": str, "params": dict, "token": str}` request;
  `{"status": "ok"|"error", "data"|"message": ...}` response.
- **Portal discovery upsert** —
  `debugcontrol/portal_discovery.py:23-72`.  Pulls remote portal
  list, upserts `PortalSnapshot` rows, deletes stale.

## Server — stream reception (Layer S3)

| Module | Path | Maturity |
|---|---|---|
| Base receiver | `streamreceiver/base_receiver.py` (136 ll.) | mature |
| TCP receiver | `streamreceiver/tcp_receiver.py` (16 ll.) | mature |
| Unix receiver | `streamreceiver/unix_receiver.py` (27 ll.) | mature |
| Deserializer | `streamreceiver/deserializer.py` (76 ll.) | **format mismatch** |
| Session lifecycle | `streamreceiver/session_lifecycle.py` (100 ll.) | mature |
| Flow control | `streamreceiver/flow_control.py` (69 ll.) | mature |
| Stream router | `streamreceiver/stream_router.py` | mature |

- **Handshake** — `streamreceiver/base_receiver.py:110-135`.
  `[2B token len][token UTF-8]`, server replies `\x00` (auth fail)
  or `\x01` (ok).
- **Flow control hysteresis** —
  `streamreceiver/flow_control.py:42-68`.  States `OK` / `PAUSE` /
  `RESUME` with high/low watermarks; out-of-band `b"PAUSE"` /
  `b"RESUME"` markers from server to pod.
- **Flush marker** —
  `streamreceiver/session_lifecycle.py:36-72`.  Pod sends `b"FLUSH"`,
  server commits + ACKs, resets per-connection buffer.

### **HONESTY POINT: pod serializer ↔ server deserializer mismatch**

This is a real, documentable mismatch between the two sides of
the wire as committed.  The book should state it plainly.

**Pod side** — `pod/traceserializer.py:28`:

```
ENVELOPE_FORMAT = "!BQQIII"   # 29 bytes
  B  event_type (int 1-9)
  Q  timestamp_ns (uint64)
  Q  thread_id (uint64)
  I  filename_length (uint32)
  I  lineno (uint32)
  I  payload_length (uint32)

Wire: [envelope 29B] [filename UTF-8] [pickle payload]
```

**Server side** — `streamreceiver/deserializer.py:15-65`:

```
[1B type_len]
[type_len bytes event_type STRING (e.g. "call")]
[8B BE double timestamp]
[4B BE uint payload_len]
[pickle payload]
```

These are **incompatible**.  Pod emits a fixed binary type code
(int 1-9) and a `uint64` ns timestamp; server expects a
length-prefixed UTF-8 string event type and an IEEE-754 double
timestamp.  Pod emits an inline filename+lineno in the envelope;
server expects neither.

Implications:
- A current pod cannot stream into the current server's
  `streamreceiver` and produce parseable events.
- The server-side `tracedata` analysis modules
  (`reader.py`, `timeline.py`, `call_graph.py`,
  `variable_tracker.py`, `search.py`) are exercised in their
  unit tests against server-side `TraceEvent` models that match
  the deserializer, not against pod-emitted bytes.

The book treats this as a real engineering signal — two halves
of an in-development protocol whose envelopes have diverged.
Volume 8 ch. 3 (the trace-event protocol) calls this out
explicitly so readers understand what the wire looks like *as
committed*, not the idealised version.

## Server — trace data analysis (Layer S4)

| Module | Path | Maturity |
|---|---|---|
| Reader | `tracedata/reader.py` (44 ll.) | mature |
| Events | `tracedata/events.py` (104 ll.) | mature |
| Timeline | `tracedata/timeline.py` (96 ll.) | mature |
| Portal filter | `tracedata/portal_filter.py` (55 ll.) | mature |
| Call graph | `tracedata/call_graph.py` (67 ll.) | mature |
| Variable tracker | `tracedata/variable_tracker.py` (88 ll.) | mature |
| Search | `tracedata/search.py` (100 ll.) | mature |

- **Server-side event hierarchy** —
  `tracedata/events.py:15-49`.  Note: `event_type` is a string
  enum here, not the integer used by the pod-side.
- **Stack-based call graph** —
  `tracedata/call_graph.py:32-62`.  Per-`(process_id, session_id)`
  stacks.  Unmatched returns silently ignored.
- **Variable history binary search** —
  `tracedata/variable_tracker.py:39-55`.  `bisect_right` over a
  precomputed sorted timestamp index — O(log n) for *value-at-
  time*.
- **Search pipeline** — `tracedata/search.py:71-99`.  Four stages:
  portal scope → property criteria → sort → pagination.
- **Timeline merge** —
  `tracedata/timeline.py:66`.  `heapq.merge` over pre-sorted
  per-source timelines (efficient O(n) merge).

## Server — REST API (Layer S5)

| Module | Path | Maturity |
|---|---|---|
| App factory | `site/timetravel_service_main.py` (72 ll.) | scaffolding |
| Health API | `site/.../v1/feature/health.py` (38 ll.) | scaffolding (mock data) |
| Process API | `site/.../v1/feature/processes.py` (131 ll.) | scaffolding (mock data) |
| Trace data API | `site/.../v1/feature/trace_data.py` (109 ll.) | scaffolding (mock data) |
| Debug sessions | `site/.../v1/feature/debug_sessions.py` | scaffolding (mock data) |
| Portal control | `site/.../v1/feature/portal_control.py` | scaffolding |
| Trace config | `site/.../v1/feature/trace_config.py` | scaffolding |
| Server config | `site/.../v1/feature/server_config.py` | scaffolding |
| Credentials | `site/.../v1/feature/credentials.py` | scaffolding |

### **HONESTY POINT: REST API is not wired to the core**

`automojo-webworks` `RestBlueprint` namespaces are registered
correctly, the routes exist, OpenAPI docs render — but every
endpoint returns mock data.  No wiring to `RegistrationService`,
`ProcessRegistry`, `TraceSearchEngine`, `CallGraphBuilder`,
`VariableTracker`, or `DebugSessionManager`.

The book treats this honestly in ch. 4 and ch. 6 — the API
*shape* is locked in (this is the contract Layer 5 will satisfy)
but the *bindings* are not yet attached.

## Tests on the server

`source/testroots/testbase/` holds 34 test files.  Strong unit
coverage on the analysis layer (S4) — `test_reader.py`,
`test_call_graph.py`, `test_timeline.py`,
`test_variable_tracker.py`, `test_portal_filter.py`,
`test_search.py`.  REST endpoint shape tests
(`test_health_api.py`, etc.) confirm the namespaces register but
do not assert against real data.
