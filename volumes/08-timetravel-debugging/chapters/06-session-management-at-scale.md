# Chapter 6 — Session Management at Scale

> **Abstract.** A single pod recording into a single trace file
> is a manageable artefact; a fleet of hundreds of pods, each
> producing recordings on demand, becomes a multi-tenant data
> management problem.  This chapter covers what the server does
> *with* recordings once they land — the sessions that group
> them, the credentials that scope them, the retention policies
> that bound them, the analysis tier that queries them, and the
> REST API surface that exposes them.  The chapter is honest
> about what is wired and what is scaffold; the analysis tier
> is built and tested, the API tier exists but is currently
> not bound to the analysis tier.

## 6.1 Sessions, captures, and the unit of debugging

The server's data model has three nested concepts:

- A **registered process** is a pod instance — a host, a PID,
  an application name, and an instance credential.  Lives in
  the `registered_processes` table.
- A **trace session** is a recording window for that process —
  a `started_at`, a `stopped_at`, a status, and a
  `data_reference` to the storage backend.  Lives in the
  `trace_sessions` table.
- A **capture** is a single recording artefact — a `.bin` file
  produced by one portal activation.  In the MCP path
  (`pod/remote/mcpserver.py:373-398`), captures are the
  per-session files in `/tmp/ccai_session_<id>/`.  On the
  server, captures are addressable through the storage
  backend's reference; their relationship to trace sessions is
  one-to-many.

The unit of *debugging* is the trace session.  An operator
asks "show me the recordings from this process between 14:00
and 14:30 yesterday"; the server answers with a list of trace
session records, each with one or more captures.

The unit of *retention* is the recording — the file on the
storage backend.  The retention job
(§6.5) sweeps files older than `data_retention_days` regardless
of which session they belong to.

The unit of *access control* is the credential.  A request to
download a recording authenticates with a credential; the
credential maps to a process; the process owns the session;
the session points to a recording.  Volume 8's credential
model is per-instance; multi-tenant access control would need
an additional layer (per-team or per-organisation credentials)
which the current implementation does not provide.

## 6.2 The credential lifecycle

The credential store
(`codecraftai-timetravel-server/source/packages/ccai/timetravelserver/persistence/credential_store.py:10`)
is the layer that manages credentials end-to-end.  Five
operations:

```python
class CredentialStore:
    def create(self, label: str, expires_in_seconds: Optional[int] = None) -> Tuple[str, Credential]:
        ...  # generate token, store hash

    def validate(self, plain_token: str) -> Optional[Credential]:
        ...  # check hash, revocation, expiry; update last_used_at

    def revoke(self, credential_id: uuid.UUID) -> None:
        ...  # set revoked_at = utcnow()

    def rotate(self, credential_id: uuid.UUID) -> Tuple[str, Credential]:
        ...  # create new credential with same label, revoke old

    def list_credentials(self) -> List[Credential]:
        ...
```

Three notable behaviours.

**Generation is one-way.**  `create` returns a tuple of
`(plain_token, Credential)`.  The plain token is the *only*
time the secret leaves the store.  The store keeps only the
SHA-256 hash; if the operator loses the plain token, there is
no way to recover it.  The remedy is to rotate
(`rotate`), which creates a new credential under the same
label and revokes the old one.

**Validation is constant-time at the storage layer.**  The
SHA-256 hash is what the database indexes on; the lookup is a
single-row `SELECT` by hash.  The pod's
`hmac.compare_digest` (ch. 5 §5.4) protects the in-memory
comparison; the server's hash-and-lookup makes the storage
read constant-time on the client-supplied token's content
(though not on the lookup miss vs hit timing — an attacker can
distinguish "valid hash, not in DB" from "valid hash, in DB"
by response time).  The chapter notes this as a residual
side-channel; the practical exposure is small because the
attacker would also have to know how to construct a valid hash,
and the server-side error response is identical in either case.

**Revocation is soft.**  A revoked credential's row stays in
the table with `revoked_at` set; the validate path rejects it
because of the field, not because of its absence.  This is the
right design for audit — the operator can answer "what
credentials existed and when were they revoked?" without a
deletion log.

The credential's link to a process is established at
registration: the pod presents the *service credential* in the
Bearer header of `/register`; the server validates, creates a
`RegisteredProcess` row, and stores the (now-known-to-be-valid)
`service_credential.id` against it.  Subsequent inbound
requests from the same pod (heartbeats, unregister) re-present
the same service credential.

The credential's link to a trace session is indirect.  A trace
session belongs to a process; the process belongs to a
credential; therefore the session belongs (transitively) to the
credential.  No row stores the credential ID on the session —
the link is through the foreign key chain.

## 6.3 The server configuration store

The server's runtime knobs live in a small, schema-driven
key-value table.
`ServerConfigStore`
(`persistence/server_config_store.py:17`) wraps the raw rows
with typed access:

```python
CONFIG_SCHEMA = {
    "listen_address": {"type": str, "default": "0.0.0.0"},
    "listen_port": {"type": int, "default": 8080},
    "tls_enabled": {"type": bool, "default": False},
    "data_retention_days": {"type": int, "default": 30},
    "max_concurrent_connections": {"type": int, "default": 100},
    "default_trace_granularity": {"type": str, "default": "function"},
    "heartbeat_timeout_seconds": {"type": int, "default": 30},
}
```

Seven knobs, each with a declared type and default.  The
`get(key)` method reads the database row (if present) and casts
to the declared type; if the row is absent, the default is
used.  The `set(key, value)` method validates the cast,
serialises the value to a string, and writes the row.

Two design notes.

First, *all values are stored as strings in the database* and
cast on read.  This trades type fidelity for schema simplicity
— a single `value` column, no per-type column proliferation,
no schema migration when a new key type is needed.  The cast
function
(`server_config_store.py:7-27`) handles bool / int / str
explicitly; adding a new type adds a branch.

Second, the schema is a *Python dict* in the source.  Adding
a new configuration key requires a code change, not a database
migration.  This is appropriate for the server's operational
posture — the configuration knobs are not user-defined; they
are the server's own runtime parameters.

The keys with the most operational impact:

- `data_retention_days` — how long recordings are kept.
  Default 30.  The retention job runs against this.
- `max_concurrent_connections` — the cap on simultaneous
  inbound trace streams.  The base receiver consults this on
  `accept`; over-cap connections are rejected.
- `heartbeat_timeout_seconds` — the freshness window for the
  lifecycle state machine (ch. 4 §4.7).
- `listen_address`, `listen_port`, `tls_enabled` — bind
  parameters.

A typical deployment overrides `data_retention_days` upward
(60 or 90 days) for environments with regulatory retention
requirements, and overrides
`max_concurrent_connections` upward for high-fanout fleets.

## 6.4 The trace session manager

`DebugSessionManager`
(`codecraftai-timetravel-server/source/packages/ccai/timetravelserver/debugcontrol/session_manager.py:12`)
is the server-side surface for managing trace sessions.  Its
operations:

- `create_session(process_id)` — open a new session, status
  `active`, `started_at` = utcnow().
- `close_session(session_id)` — finalise the session, status
  `completed`, `stopped_at` = utcnow().
- `list_sessions(...)` — paginated listing with filters.
- `get_session(session_id)` — fetch one session.
- `update_data_reference(session_id, ref)` — set the
  `data_reference` on the session row when the recording
  completes.

The session manager *does not* run the receive loop.  The
receive loop (ch. 4 §4.3) lives in `StreamSessionLifecycle` and
operates per-connection.  The session manager is the
*database-side* representation; the receive lifecycle is the
*network-side* representation.  They are reconciled by the
ingestion layer: when a stream finalises (flush marker, clean
close), the receive lifecycle calls into the session manager
to set `data_reference` and close the session row.

This split — *one entity handles the wire, another handles the
record* — is the same separation Volume 4 used for sysman
features (the action runner is one thing, the result emitter
is another).  It keeps the per-connection thread free to
actually drain bytes; the database transaction lives in a
short call at session boundaries, not on every chunk.

## 6.5 Retention

The retention job is a periodic sweep that deletes recordings
older than `data_retention_days`.  Two paths interact:

**Storage retention.**  The filesystem backend
(`persistence/trace_storage/filesystem_backend.py`) provides
`list_expired(age_threshold)` and `delete(reference)`.  The
expired listing scans the `final/` directory's files by
`stat().st_mtime` and yields references older than the
threshold.  The retention job iterates and deletes.

**Database retention.**  Rows in `trace_sessions` and
`registered_processes` have their own retention.  A session
whose recording has been deleted from storage but whose row
still exists is *orphaned* — accessing the row's
`data_reference` will fail to read.  The retention job deletes
session rows whose `data_reference` no longer exists, or whose
`stopped_at` is older than the retention window.

The two paths run independently — by design.  An orphaned
session row is recoverable (the operator sees it in the
session list, gets a "data not available" indicator, and can
delete the row manually or wait for the retention sweep).  An
orphaned storage file is recoverable (the retention sweep
deletes it on the next run).  The system tolerates either
inconsistency for at most the retention sweep interval.

A subtle interaction is the `ondelete='CASCADE'` on
`PortalSnapshot.process_id` and `TraceSession.process_id`
(`persistence/models/process.py:7-25`,
`persistence/models/trace_session.py:8-17`).  Deleting a
process cascades to its portal snapshots and trace session
rows.  Trace files on storage are *not* cascaded — they
require the storage retention sweep to clean up.

## 6.6 Replay-bandwidth budget

The operator's question "show me the recording" is, on the
server side, a question about *bandwidth*: how fast can a
recording's bytes be moved from storage into the analysis
tier, and how many concurrent operators can the server support?

Three knobs:

**Storage I/O.**  A recording is a file on disk; reading it is
sequential I/O.  At a typical SATA SSD throughput of 500 MB/s,
a 100 MB recording reads in 200 ms.  At 5 GB, it reads in 10 s.
The bottleneck is rarely I/O for typical recording sizes; it
becomes the bottleneck for the variable-snapshot recordings
ch. 8 measures (which can be 1+ GB for a few seconds of trace).

**Deserialisation cost.**  Pickle loading a recording's
events imposes the cost ch. 3 §3.3 alluded to: the pickle
protocol is fast for simple types but slow for complex ones.
A recording with `local_variables` dicts of arbitrary
application objects pays for each pickle.  For recordings with
heavy variable capture, deserialisation can dominate.

**Analysis cost.**  The analysis tier
(`tracedata/`) does its work on the deserialised events: build
the timeline, build the call graph, build the variable
history, run the search query.  These are O(n) or O(n log n)
in the number of events.  For a recording with a million
events, this is seconds to tens of seconds of single-threaded
CPU.

The server, today, does not parallelise the analysis across
recordings.  Each query reads its recording, deserialises it,
runs the analysis, and returns results.  Concurrent queries
are concurrent at the OS level (many threads) but each query
sees no help from the others; the scheme is "embarrassingly
parallel by request, sequential per request".

A future optimisation — pre-built indices on each recording —
would amortise the analysis cost across queries.  The
recording is finalised once; on finalisation, an index could
be built and stored alongside the recording.  Subsequent
queries would consult the index rather than re-deserialising
the events.  This is not in the source as committed; the book
notes it as the obvious next investment for high-volume
deployments.

## 6.7 The analysis tier

The analysis tier
(`codecraftai-timetravel-server/source/packages/ccai/timetravelserver/tracedata/`)
is the suite of modules that turn a deserialised event stream
into the structures an operator's queries return.  Six modules,
all internally consistent, all unit-tested.

**Timeline reconstruction.**
`tracedata/timeline.py:35-63` accumulates events into a
sorted-by-timestamp list with metadata:

```python
def build(self) -> Timeline:
    self.events.sort(key=lambda e: e.timestamp)

    start_time = 0.0
    end_time = 0.0
    duration = 0.0
    event_counts: Dict[str, int] = {}

    if self.events:
        start_time = self.events[0].timestamp
        end_time = self.events[-1].timestamp
        duration = end_time - start_time

        for event in self.events:
            event_type_str = event.event_type.value
            event_counts[event_type_str] = event_counts.get(event_type_str, 0) + 1

    return Timeline(
        events=self.events,
        start_time=start_time,
        end_time=end_time,
        duration=duration,
        event_counts=event_counts
    )
```

The `merge` method
(`timeline.py:66`) is `heapq.merge` over per-source pre-sorted
timelines — a linear-time merge.  This is what an operator
gets when they request "events from process A and process B
between T1 and T2 on a unified timeline".

**Portal scope filtering.**
`tracedata/portal_filter.py:35-54`:

```python
@staticmethod
def filter_by_portals(events: Iterable[TraceEvent], target_portals: Set[str]) -> Iterator[TraceEvent]:
    if not target_portals:
        return

    active_portals: Set[str] = set()

    for event in events:
        if event.event_type == EventType.PORTAL_ENTER:
            active_portals.add(event.portal_name)

        if active_portals.intersection(target_portals):
            yield event

        if event.event_type == EventType.PORTAL_EXIT:
            if event.portal_name in active_portals:
                active_portals.remove(event.portal_name)
```

A small state machine that tracks which portals are currently
open and yields events whose timestamp falls within a target
portal scope.  This is what makes the *named portal*
abstraction useful for analysis: an operator can ask "show me
the calls that occurred while `invoice.charge.attempt` was
active" and get back only those events.

**Call-graph construction.**
`tracedata/call_graph.py:32-62` is a stack-based builder.
Per `(process_id, session_id)`, it maintains a call stack;
on a `CallEvent` it pushes a node onto the stack and links it
to its parent; on a `ReturnEvent` it pops and attaches the
return.  The result is a forest of `CallGraphNode` trees, each
with a `duration` derived from the call's enter timestamp and
the matched return's timestamp.

The note on edge cases: unmatched return events (a return
without a preceding call in the recording) are silently
ignored.  This is the right behaviour for a *recording* — the
recording started in the middle of an active call stack on its
first event, so the early returns may exit calls that were
already on the stack before the recording window opened.

**Variable history.**
`tracedata/variable_tracker.py:39-55` is the
binary-search-by-timestamp lookup for a variable's value at a
given moment:

```python
def value_at(self, variable_name: str, timestamp: float) -> Optional[VariableSnapshotEvent]:
    if variable_name not in self._timestamps:
        return None

    timestamps = self._timestamps[variable_name]

    idx = bisect.bisect_right(timestamps, timestamp)

    if idx == 0:
        return None

    return self._histories[variable_name][idx - 1]
```

A precomputed sorted timestamp list per variable; `bisect_right`
finds the index of the *most recent snapshot at or before* the
queried timestamp.  O(log n) per lookup.  This is the *time-
travel* mechanic exposed: "what was the value of `amount` at
T = 142.5?" answered in microseconds against a recording of
millions of events.

**Search.**
`tracedata/search.py:71-99` is a four-stage pipeline:

```python
def search(self, query: SearchQuery) -> SearchResult:
    # Step 1: Portal scope filtering
    filtered_events = iter(self.events)
    if query.portals:
        filtered_events = PortalScopeFilter.filter_by_portals(filtered_events, query.portals)

    # Step 2: Apply properties criteria
    matched_events = []
    for event in filtered_events:
        if self._matches_query(event, query):
            matched_events.append(event)

    # Step 3: Sorting
    matched_events.sort(key=lambda e: getattr(e, query.order_by, e.timestamp), reverse=not query.ascending)

    # Step 4: Pagination
    total_count = len(matched_events)
    start_idx = query.offset
    end_idx = query.offset + query.limit

    paginated_events = matched_events[start_idx:end_idx]

    return SearchResult(
        events=paginated_events,
        total_count=total_count,
        offset=query.offset,
        limit=query.limit
    )
```

Portal scope, then property criteria (file, function, line,
text substring), then sort, then page.  An operator query
"show me CallEvents in `invoice.py` containing the variable
`amount` greater than 5000, sorted by timestamp, page 2 of
50" composes through these four stages.

## 6.8 The REST API as designed

Layer S5 specs the HTTP surface that exposes everything
above to operators and tools.  The `codecraftai-webworks`
blueprint pattern is in place
(`codecraftai-timetravel-server/source/site/`); the routes
register; OpenAPI documentation renders.  The endpoint set is:

- `GET /processes` — list registered processes with pagination,
  filtering, sorting.
- `GET /processes/{id}` — process detail.
- `POST /processes/{id}/refresh` — force metadata refresh
  (re-query the pod via the dispatcher).
- `POST /processes/{id}/disconnect` — mark disconnected.
- `DELETE /processes/{id}` — delete an archived (disconnected)
  process record.
- `POST /processes/register`, `/unregister`, `/heartbeat` — the
  endpoints the pod talks to (`pod/remote/registration.py` and
  `pod/remote/heartbeat.py`).
- `GET /processes/{id}/portals` — list portals for a process
  (calls `PortalDiscoveryService.discover`).
- `POST /processes/{id}/portals/{name}/activate` — activate a
  portal (calls `PortalControlService.activate`).
- `POST /processes/{id}/portals/{name}/deactivate` —
  deactivate.
- `GET /processes/{id}/trace-config` — read trace
  configuration (blocked on the spec gap from ch. 5 §5.3).
- `PUT /processes/{id}/trace-config` — write trace
  configuration (blocked on the same spec gap).
- `POST /sessions` — create a debug session.
- `GET /sessions` — list sessions with pagination/filtering.
- `GET /sessions/{id}` — session detail.
- `DELETE /sessions/{id}` — close a session.
- `GET /trace-data/events` — search trace events (calls
  `TraceSearchEngine`).
- `GET /trace-data/timeline` — reconstructed timeline.
- `GET /trace-data/call-graph` — call graph forest.
- `GET /trace-data/variables` — variable list.
- `GET /trace-data/variables/{name}/history` — variable
  history.
- `POST /credentials` — create credential.
- `GET /credentials` — list credentials.
- `POST /credentials/{id}/rotate` — rotate.
- `POST /credentials/{id}/revoke` — revoke.
- `DELETE /credentials/{id}` — delete.
- `GET /server/config` — read server configuration.
- `PUT /server/config` — write server configuration.
- `GET /server/health` — server health check.

The routes are registered.  The OpenAPI spec exists at
`/api/docs`.  The endpoint signatures match the design.

## 6.9 What the REST API does not yet do

This is the chapter's second honesty point (the first is the
configuration spec gap from ch. 5 §5.3).

The endpoint *bodies*, as committed, return mock data.  The
process-list endpoint does not call `ProcessRegistry.list_processes`;
it returns a hand-built dict.  The trace-data endpoints do not
call `TraceSearchEngine.search` or `CallGraphBuilder.add_events`;
they return synthetic event lists.

The wiring — taking the request, instantiating the service,
calling the right method, formatting the response — is the
work that closes the gap.  It is not architecturally hard.  The
service layer is in place, well-tested, and accepts the right
inputs.  The gap is the per-endpoint implementation that
translates between HTTP and the service interface.

The book's framing is structural rather than procedural: *the
contract is locked, the algorithms are written, the storage is
real, the binding is the last mile*.  An operator using the
server today has the analysis tier through programmatic Python
calls (the unit tests are the canonical examples); a future
release will land the bindings, and the operator's experience
shifts to the REST surface.

## 6.10 Multi-tenant by credential

The current architecture has *some* multi-tenant capability
through the credential model.  Multiple credentials can exist;
each credential maps to a process; processes are isolated by
their credential.

Two things the architecture does *not* yet provide:

**Per-team or per-organisation grouping.**  A credential
identifies a process; there is no notion of "this credential
belongs to team A, that one to team B, both report to the same
server but cannot see each other's traces."  Adding this is a
schema and authorisation change — a `tenant_id` on the
credential, an authorisation predicate on every list endpoint
filtering by tenant.  Not architecturally complex but not
shipped.

**Role-based access control.**  The `User` table
(`persistence/models/user.py:6-12`) has a `role` field
(`viewer`, implicit `admin`), but no endpoint in the current
REST tier consults it.  The roles are placeholder.  When the
REST tier wires through, the role check is the authorisation
gate — viewers can read, admins can write configuration and
manage credentials.

The book's recommendation: the credential-based isolation is
sufficient for the small-fleet case (one team, many pods).  The
team-based isolation is the next step for the multi-team case;
it is a small-to-medium amount of work and a substantial
operational improvement.

## 6.11 The operational dashboard, in shape

A typical operations dashboard reading the REST API renders
four panes:

**Process list.**  Paginated, with state filters
(`active` / `stale` / `unreachable` / `disconnected`), a
search box on application name, and a sort on
`registered_at`.  The shape comes from
`ProcessRegistry.list_processes`
(`registration/process_registry.py:16-52`).

**Process detail.**  A single process: host, PID, application,
pod version, lifecycle state, last heartbeat, the portals
discovered (with their active/inactive flags), the recent
trace sessions.  Aggregated from `RegisteredProcess`,
`PortalSnapshot`, `TraceSession`.

**Session viewer.**  A single trace session: timeline (events
plotted on a timeline), call graph (collapsible tree),
variable inspector (drop-down by name, then a history table).
Drives `TraceSearchEngine`, `CallGraphBuilder`,
`VariableTracker`.

**Server health.**  Active connections, ingestion rate,
storage utilisation, registered process count by state,
oldest session waiting on flush.  Server-internal metrics
exposed at `GET /server/health`.

These four panes describe the operator's interaction with the
system.  The first three are query-driven (the operator picks
a target and the server retrieves its data); the fourth is
metric-driven (the operator watches the server's own state).

## 6.12 The control plane in flow

A complete debugging session, end-to-end:

1. **Pod starts.**  `configure_timetravel` activates whatever
   portals the deployment-time configuration declared, opens
   the JSON-lines server, registers with the time-travel
   server, starts the heartbeat sender.  `RegisteredProcess`
   row inserted with state `registered`.
2. **First heartbeat lands.**  `last_heartbeat_at` updated,
   state flipped to `active`.
3. **Operator opens dashboard.**  `GET /processes` lists the
   pod.
4. **Operator drills in.**  `GET /processes/{id}` shows
   detail; `GET /processes/{id}/portals` triggers a
   `PortalDiscoveryService.discover` call out to the pod's
   JSON-lines `list_portals` command, reconciling the local
   `PortalSnapshot` rows.
5. **Operator activates a portal.**  `POST
   /processes/{id}/portals/{name}/activate` triggers a
   `PortalControlService.activate` call, which dispatches
   `activate_portal` to the pod.  The pod's portal registry
   flips active.
6. **Application traffic crosses the portal.**  The pod
   records, the TCP backend streams to the server's
   `TcpStreamReceiver`, the `StreamSessionLifecycle` drains
   bytes to a temp file in `<base>/temp/`.
7. **Recording finalises.**  Flush marker arrives or
   `once=True` triggers cleanup.  Temp file is renamed to
   `<base>/final/`.  `TraceSession.update_data_reference`
   fires; the session row's `data_reference` and
   `stopped_at` are set; status flipped to `completed`.
8. **Operator inspects the recording.**  `GET
   /sessions/{id}/timeline` runs the analysis tier,
   reconstructing a timeline from the file's events.  `GET
   /sessions/{id}/call-graph` returns the call graph.  `GET
   /sessions/{id}/variables/{name}/history` returns the
   variable's history.
9. **Operator deactivates the portal.**  `POST
   /processes/{id}/portals/{name}/deactivate` mirrors step 5.
   Future calls into the portal scope take the cheap path.
10. **Retention sweep.**  After
    `data_retention_days` (default 30), the recording is
    eligible for deletion.  The retention job sweeps it.

In the implementation as committed, steps 6, 7, and the
service-layer guts of steps 4, 5, 8, and 9 are *real*.  Steps
3 and the HTTP wrappers around 4, 5, 8, 9 return mock data.
Closing the gap is the wire-it-up work the next investment
cycle does.

## 6.13 What scale looks like in numbers

A small deployment: 10 pods, each producing recordings on
demand, average 50 MB per recording, average 5 recordings per
day per pod, retention 30 days.

- Storage budget: 10 × 5 × 50 MB × 30 = 75 GB.
- Concurrent connections: at most 10 (one per pod), often 0-2.
- Database growth: ~50 trace_session rows per day × 30 day
  retention = ~1500 rows; trivial.

A medium deployment: 100 pods, 100 MB average, 10 recordings
per pod per day, 30 days.

- Storage budget: 100 × 10 × 100 MB × 30 = 3 TB.
- Concurrent connections: ~10 typical, ~100 peak.
- Database growth: ~1000 rows/day × 30 = 30k rows.

A large deployment: 1000 pods, 200 MB average, variable
recording cadence (most days zero, incident days hundreds of
recordings).

- Storage budget: dominated by incident days.  A single
  day's worth of incident recordings can be 100 GB or more
  per pod; cluster-wide a single bad incident day can be 10+
  TB.
- Concurrent connections: incident-driven, can spike to
  hundreds.
- Database growth: skewed but bounded.

The architecture as implemented is fit for the small and
medium case.  The large case will exercise things the current
implementation does not yet do — the per-recording index from
§6.6, partitioned storage backends (S3 / object store rather
than filesystem), multi-tenant isolation from §6.10, and a
priority queue on the receive path so incident-related streams
can be admitted ahead of routine ones.  These are addressable;
they are not in the current commit.

## 6.14 Bridging to the next chapter

The architectural and operational story is now in place.  The
remaining two chapters bring the system to ground in concrete
debugging scenarios.  Chapter 7 walks three case studies — a
flaky distributed test (cross-referencing Volume 2's tandem
testing model), an agent execution (cross-referencing Volume
7's `ExecutionRecorder`), and a sysman feature on a remote
host (cross-referencing Volume 4).  Chapter 8 quantifies the
performance and storage trade-offs the recording-fidelity-vs-
cost knobs control.

→ Continue to [`07-case-studies.md`](07-case-studies.md).

← Back to [`05-remote-debug-control.md`](05-remote-debug-control.md).
