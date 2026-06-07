# Chapter 4 — Server-Side Ingestion

> **Abstract.** A trace is data on a wire; data on a wire is
> only useful if something on the other end will catch it,
> persist it, and let an analyst query it.  The server's
> ingestion path runs from the socket listener through a
> handshake, into a per-connection session that drains the
> stream, deserialises it, and hands events to a storage
> backend; back-pressure rides the same connection in the
> reverse direction.  This chapter walks the receivers (TCP and
> Unix), the deserialiser (with the format mismatch from ch. 3
> in mind), the session lifecycle, the flow-control mechanism,
> and the persistence backend that turns received bytes into
> durable artefacts.  Anchored on
> `codecraftai-timetravel-server`.

## 4.1 The shape of the receiving end

The server's job at ingestion is to:

1. Bind a socket — TCP or Unix — and accept incoming
   connections from pod processes.
2. Authenticate each connection against a credential store.
3. Open a *session* for the duration of the connection: drain
   bytes, deserialise events, hand them to storage.
4. Apply *flow control* when the storage layer cannot keep up,
   asking the pod to slow down.
5. Commit the recording on a flush marker; clean up the
   per-connection state on close.
6. Survive abrupt disconnects without leaving partially-written
   trace files in the durable area.

The implementation splits across four directories under
`codecraftai-timetravel-server/source/packages/ccai/timetravelserver/`:

- `streamreceiver/` — the listener, session, and flow-control
  layer.
- `persistence/trace_storage/` — the durable-storage backend
  that received events flow into.
- `persistence/credential_store.py` — the credential store
  used for handshake authentication.
- `registration/` — the lifecycle layer that ties an inbound
  stream to a registered process record.

The first three are the subject of this chapter; the fourth is
the subject of ch. 6.

## 4.2 The base receiver

`BaseStreamReceiver`
(`codecraftai-timetravel-server/source/packages/ccai/timetravelserver/streamreceiver/base_receiver.py:9`)
is the abstract socket-server.  It handles the loop that accepts
connections, performs the handshake, and hands each accepted
connection off to a session for receive-side draining.

The handshake
(`base_receiver.py:110-135`) is intentionally small:

```python
def _handle_connection(self, client_sock: socket.socket, addr: tuple):
    # Format for handshake:
    # [2 bytes length][token bytes]
    length_bytes = self._read_exactly(client_sock, 2)
    if not length_bytes:
        return

    token_len = struct.unpack("!H", length_bytes)[0]
    token_bytes = self._read_exactly(client_sock, token_len)
    if not token_bytes:
        return

    token = token_bytes.decode('utf-8')

    if not self.auth_callback(token):
        logger.warning(f"Authentication failed for {addr}")
        # Send auth failed response
        client_sock.sendall(b'\x00')
        return

    # Send auth success response
    client_sock.sendall(b'\x01')

    # Hand off to session
    session = self.session_factory(client_sock, token)
    session.run()
```

Two bytes for the token length (big-endian `uint16`, max 65,535
bytes — far more than any sensible token), then the token bytes
themselves, UTF-8 encoded.  The server replies with a single
byte: `\x00` for auth failure, `\x01` for auth success.

The token is validated through an `auth_callback` injected at
construction.  In the running server this callback delegates to
`CredentialStore.validate`
(`persistence/credential_store.py:31-47`), which:

1. Hashes the provided token with SHA-256.
2. Looks up the hash in the `credentials` table.
3. Rejects if the credential is revoked
   (`revoked_at IS NOT NULL`) or expired
   (`expires_at < utcnow()`).
4. Updates `last_used_at` to the current timestamp.
5. Returns the matching `Credential` ORM record.

The credential is what scopes the connection: the server can,
later, look up the credential to find the registered process
that owns it.  Volume 8 ch. 6 walks the credential lifecycle
end-to-end.

The `BaseStreamReceiver` is subclassed by:

- `TcpStreamReceiver`
  (`streamreceiver/tcp_receiver.py:5`) — binds to a `(host,
  port)` with `SO_REUSEADDR` set so the server can restart
  without lingering bind failures.
- `UnixSocketStreamReceiver`
  (`streamreceiver/unix_receiver.py:6`) — binds to a Unix
  domain socket path with `0o600` permissions (owner-only),
  cleaning up any stale socket file at startup.

The TCP receiver is the production path for distributed
deployments — pod and server on separate hosts.  The Unix
receiver is the production path for same-host deployments —
pod and server in the same network namespace, where Unix
sockets eliminate the TCP loopback overhead.

## 4.3 The session

Once the handshake passes, the connection is handed off to a
`StreamSessionLifecycle`
(`streamreceiver/session_lifecycle.py:7`).  The session owns the
connection for the rest of its life: it drains the socket in
4 KiB chunks, hands each chunk to the storage backend, watches
for flush markers, and applies flow-control responses.

The receive loop
(`session_lifecycle.py:36-72`) is the heart of the path:

```python
def _receive_loop(self):
    """Reads chunks of data and appends to storage."""
    buffer_size = 0

    while self._running:
        try:
            # Set a small timeout to allow graceful shutdown checks
            self.sock.settimeout(1.0)
            chunk = self.sock.recv(4096)
            if not chunk:
                break  # Connection closed gracefully

            self.storage_backend.append(chunk)

            # Flow control integration
            if self.flow_controller:
                buffer_size += len(chunk)
                status = self.flow_controller.update_buffer_size(self._connection_id, buffer_size)

                if status.value == "PAUSE":
                    self.sock.sendall(b"PAUSE")
                elif status.value == "RESUME":
                    self.sock.sendall(b"RESUME")

            # Check for flush marker
            if b"FLUSH" in chunk:
                self._handle_flush()
                # Reset buffer size on flush as data is persisted
                if self.flow_controller:
                    buffer_size = 0
                    status = self.flow_controller.update_buffer_size(self._connection_id, 0)
                    if status.value == "RESUME":
                        self.sock.sendall(b"RESUME")

        except socket.timeout:
            continue
```

The loop receives 4 KiB at a time.  It does *not* deserialise on
the receive path — bytes go directly to `storage_backend.append`.
Deserialisation is deferred to the analysis tier.  This is a
load-shedding decision: the receive path's bottleneck is *making
the bytes durable*, not *understanding* them.  Pushing
deserialisation off the receive path keeps the receiver's CPU
budget low and lets the server accept more concurrent streams.

The 1-second `settimeout` is the graceful-shutdown check.  Every
second the loop wakes, sees `socket.timeout`, and re-checks
`self._running`.  When `stop()` is called from the outside (the
process is shutting down, the connection should close), the
loop exits within at most one second.  Without the timeout, the
loop would block on `recv` indefinitely and only respond to
shutdown after the next byte arrived.

The `b"FLUSH" in chunk` test is a substring check — the marker
can appear anywhere in the received chunk.  It is a deliberate
design choice: the marker need not be aligned to the chunk
boundary the network happens to produce.  The cost is that an
event payload that happens to include the literal bytes
`b"FLUSH"` would also trigger a commit.  In a pickle-encoded
payload this is unlikely (pickle's framing makes literal
ASCII strings rare in the binary), but it is not impossible.  A
production-hardened version of the protocol would use a less
ambiguous marker (a length-prefixed sentinel, an out-of-band
control message, or — best — a magic number with a known
position in a framed protocol).  The book treats this as a known
limitation of the protocol-as-committed.

The flow-control branches send literal bytes `b"PAUSE"` and
`b"RESUME"` *back* through the same socket.  This is an
out-of-band signalling channel multiplexed onto the in-band data
stream — the pod's `tcpbackend.write()` does not currently know
how to read these bytes back; the channel is one-directional in
the pod's implementation as committed.  This is a second
known-asymmetric edge: the server *signals* flow control, but
the pod *does not yet honour* the signal.  Drop-on-saturation
(§3.6) is what kicks in if the server cannot drain fast enough.

## 4.4 The storage backend

The session writes received chunks to a `TraceStorageBackend`
(`persistence/trace_storage/backend_abc.py:5`).  The ABC is
small — `write`, `append`, `finalize`, `read`,
`list_references`, `delete`, `exists`, `list_expired` — and
maps to a *reference-based* interface.  Each recording is
identified by an opaque reference string returned by `write`;
subsequent operations on the same recording (append more bytes,
finalise, read it back later) take the reference as input.

The shipped backend is filesystem-based
(`persistence/trace_storage/filesystem_backend.py:9`).  Its
storage layout is two directories under a configured base:

- `<base>/temp/` — in-flight recordings.  Each new recording is
  opened as a `.tmp` file under this directory; bytes are
  appended as the session drains them.
- `<base>/final/` — finalised recordings.  When a recording
  is finalised, the temp file is *atomically renamed* to the
  final directory:

```python
def finalize(self, reference: str) -> str:
    ref_path = Path(reference)
    if not ref_path.exists():
        raise FileNotFoundError(f"Reference not found: {reference}")

    final_filename = ref_path.name.replace(".tmp", "")
    final_path = self.final_dir / final_filename

    # Atomic rename
    ref_path.rename(final_path)
    return str(final_path)
```

The `Path.rename` uses POSIX `rename(2)`, which is atomic on the
same filesystem.  Either the file is in `temp/` (recording is
in flight) or it is in `final/` (recording is committed); there
is no intermediate state where the recording is "half-
finalised".  This is the property the database relies on when
it stores `data_reference` on the `TraceSession` record:
`data_reference` always points at a finalised file, never an
in-flight one.

Two operations reach into `temp/` directly: `append`, which
opens the temp file in append mode and writes bytes, and the
`finalize` above.  The atomic rename means a consumer reading
from `final/` never sees a partial recording.

The backend also exposes `list_expired`
(`filesystem_backend.py`, late in the file) which uses
`stat().st_mtime` to identify recordings older than a
configurable retention threshold.  This is what the retention
job (ch. 6) uses to delete recordings older than
`data_retention_days` (default 30).

## 4.5 Flow control with hysteresis

The flow controller
(`streamreceiver/flow_control.py:13`) tracks two bookkeeping
items: per-connection buffer sizes (how many bytes are in flight
for each session) and a global memory used (the sum across all
connections).  It maintains a state per connection — `OK`,
`PAUSE`, `RESUME` — and transitions on watermark crossings.

The `update_buffer_size` method
(`flow_control.py:42-68`) implements the hysteresis:

```python
def update_buffer_size(self, connection_id: str, new_size: int) -> FlowControlStatus:
    with self._lock:
        if connection_id not in self.connection_buffers:
            return FlowControlStatus.OK

        old_size = self.connection_buffers[connection_id]
        self.connection_buffers[connection_id] = new_size

        # Update global memory
        self.global_memory_used += (new_size - old_size)

        current_state = self.connection_states[connection_id]

        if current_state == FlowControlStatus.OK or current_state == FlowControlStatus.RESUME:
            if new_size >= self.high_watermark:
                self.connection_states[connection_id] = FlowControlStatus.PAUSE
                return FlowControlStatus.PAUSE
        elif current_state == FlowControlStatus.PAUSE:
            if new_size <= self.low_watermark:
                self.connection_states[connection_id] = FlowControlStatus.RESUME
                return FlowControlStatus.RESUME

        return FlowControlStatus.OK
```

A connection's state machine:

- Starts in `OK`.
- Crosses `high_watermark` → transitions to `PAUSE`, returns
  `PAUSE`.  The session emits `b"PAUSE"` to the pod.
- Drops below `low_watermark` → transitions to `RESUME`,
  returns `RESUME`.  The session emits `b"RESUME"`.
- Stays in any state otherwise → returns `OK` (no signal).

The two watermarks are the *hysteresis*.  Without them — a
single threshold for both pause and resume — a connection
hovering near the threshold would oscillate, generating
PAUSE/RESUME every time a chunk arrived.  Two watermarks with
a gap give the connection a *window* in which the state does
not change.  A typical configuration sets the high watermark
several times higher than the low watermark.

The lock around the body is one reader-writer-style lock —
`threading.Lock` — protecting `connection_buffers`,
`connection_states`, and `global_memory_used` from concurrent
update by multiple session threads.  Each session thread that
receives a chunk calls `update_buffer_size` once, holding the
lock for microseconds.

There is also a global guard.  `should_accept_new`
(`flow_control.py:37`) checks the global memory used against a
configured limit; the receiver consults this *before* accepting
a new connection.  If the server is already over budget, new
connections are refused at the accept stage rather than
admitted to compete for a saturated buffer.

## 4.6 The deserialiser as committed

The deserialiser
(`streamreceiver/deserializer.py:9`) is what parses bytes back
into events when the analysis tier needs them.  Its format
expectation is the one ch. 3 documented as *not yet matching
the pod's emitted bytes* (§3.8).

The book treats this honestly.  The deserialiser as committed:

```python
def deserialize_stream(self, stream, process_id, session_id, portal_name):
    while True:
        # Read type length
        type_len_data = stream.read(1)
        ...
        # Read type string
        type_str_data = stream.read(type_len)
        ...
        # Read timestamp (double) and payload len (unsigned int)
        header_data = stream.read(12)
        ...
        timestamp, payload_length = struct.unpack("!d I", header_data)

        # Read payload
        payload_data = stream.read(payload_length)
        ...
        payload = pickle.loads(payload_data)

        try:
            event = self._materialize_event(event_type, timestamp, process_id, session_id, portal_name, payload)
            yield event
        except Exception as e:
            logger.error(f"Failed to materialize event: {e}. Payload was: {payload}")
            continue
```

Recovery semantics: a failed pickle parse logs and continues to
the next iteration of the loop — but *the next iteration will
be reading bytes that follow whatever pickle was attempting to
consume*.  Without a magic number or framing marker, recovery
is approximate: the next iteration may succeed (if the byte
boundary happened to land on the start of a new envelope) or
may fail (if the byte boundary did not).  In practice, a
deserialiser that fails on one event tends to fail on all
events afterwards in the stream.

The chapter does not pretend the deserialiser is robust.  It
is the right shape — envelope, then pickle, with substring-
addressable type tags — but its concrete bytes do not currently
match the pod's emission, and its error recovery is not
resyncable.  Both are addressable; both are acknowledged.

## 4.7 The session lifecycle states

The session lifecycle, as a state machine, has five visible
states for the pod-side process record (§S1.4 in the spec, with
state transitions in `registration/heartbeat.py:67-100`):

- `registered` — the pod's `/register` request was accepted but
  no heartbeat has yet arrived.
- `active` — at least one heartbeat has been received within
  the freshness window.
- `stale` — the most recent heartbeat is older than
  `heartbeat_timeout_seconds` (default 30 s).
- `unreachable` — the most recent heartbeat is older than
  `2 × heartbeat_timeout_seconds` (default 60 s).
- `disconnected` — the pod sent a clean `/unregister` request,
  or the server marked the process disconnected manually.

The transitions are driven by the `HealthChecker`
(`registration/heartbeat.py`) running on a daemon thread
roughly every `heartbeat_timeout_seconds / 2` seconds.  The
checker scans the registered processes table, computes the
freshness threshold, and updates `lifecycle_state` in place:

```python
stale_threshold = now - datetime.timedelta(seconds=timeout_seconds)
unreachable_threshold = now - datetime.timedelta(seconds=2 * timeout_seconds)

# Active -> Stale
active_processes = session.query(RegisteredProcess).filter_by(lifecycle_state="active").all()
for process in active_processes:
    if process.last_heartbeat_at and process.last_heartbeat_at < stale_threshold:
        process.lifecycle_state = "stale"

# Stale -> Unreachable
stale_processes = session.query(RegisteredProcess).filter_by(lifecycle_state="stale").all()
for process in stale_processes:
    if process.last_heartbeat_at and process.last_heartbeat_at < unreachable_threshold:
        process.lifecycle_state = "unreachable"
```

Three observations.

First, transitions only ever go forward (active → stale →
unreachable).  The reverse transition — `stale` back to
`active` because a heartbeat finally arrived — is handled at
the heartbeat-reception path
(`registration/heartbeat.py:18`), not the health checker.  When
a heartbeat lands, the receiver updates `last_heartbeat_at` and
sets `lifecycle_state="active"`.  The two paths (forward in the
checker, backward in the receiver) cannot race because both
take the SQLAlchemy session's row-level update.

Second, the `disconnected` transition is *only* triggered by a
clean unregister request.  A pod that crashes silently goes to
`unreachable` and stays there until an operator manually marks
it disconnected (or until the retention job sweeps unreachable
records older than the retention threshold).

Third, the state transitions are *advisory* — they do not
affect ingestion.  A process in `unreachable` whose previous
trace stream is still being received continues to have its bytes
written.  Lifecycle state is *operator-facing telemetry*; it is
the answer to "is this process alive?", not the gate on "should
the server accept its bytes?".

## 4.8 Trace session records

A *trace session* (the `TraceSession` ORM model,
`persistence/models/trace_session.py:8-17`) is the database
record that ties a recording to a process.  Its fields:

```python
class TraceSession(BaseMixin, Base):
    __tablename__ = 'trace_sessions'

    process_id: Mapped[uuid.UUID] = mapped_column(ForeignKey('registered_processes.id', ondelete='CASCADE'), nullable=False, index=True)
    started_at: Mapped[datetime.datetime] = mapped_column(DateTime, default=datetime.datetime.utcnow, nullable=False)
    stopped_at: Mapped[Optional[datetime.datetime]] = mapped_column(DateTime, nullable=True)
    status: Mapped[str] = mapped_column(String(32), default="active", nullable=False, index=True)
    data_reference: Mapped[Optional[str]] = mapped_column(String(1024), nullable=True)
```

The `data_reference` is the storage backend's reference string
— for the filesystem backend, the path to the finalised file
under `<base>/final/`.  It is `NULL` while the recording is in
flight and is populated by `update_data_reference`
(`debugcontrol/session_manager.py:78`) when the session manager
is told the recording has finalised.

The `ondelete='CASCADE'` on the `process_id` foreign key means
that deleting a process deletes its trace sessions.  This is
the right semantics for retention: when a process is purged
(unreachable for too long, or explicitly deleted by an
operator), its associated trace sessions go too.  The
`data_reference` they pointed at is then orphaned — the trace
file lives on disk until the storage-retention job sweeps it
based on `mtime`.

The two retention paths — database (process / session
deletion) and storage (file deletion based on age) — are
intentionally decoupled.  A trace file may be older than its
session record (the file was finalised, the session record
not yet updated) or younger (the session was deleted but the
file is still on disk).  Both cases are tractable: orphaned
files are swept by storage retention; orphaned session records
are tolerable until the next process delete.

## 4.9 The persistence engine

`persistence/engine.py:4-9` is a tiny module — nine lines —
that constructs the SQLAlchemy engine and a scoped session
factory:

```python
def create_db_engine(): ...  # line 4
def create_session_factory(): ...  # line 7
```

The engine is shared across the server process; sessions are
*scoped* — a new session per thread, with `autocommit=False`
and `autoflush=False`.  Explicit commit control is the right
default for this server: an ingestion thread that has just
written a chunk's worth of received bytes to storage commits
*together* the storage write *and* the session-record update,
so the database state and the storage state stay coherent.

The schema is defined by the ORM models under
`persistence/models/` — `RegisteredProcess`, `TraceSession`,
`Credential`, `PortalSnapshot`, `ServerConfiguration`, `User`.
Each inherits from a `BaseMixin`
(`persistence/models/base.py:8-11`):

```python
class BaseMixin:
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    created_at: Mapped[datetime.datetime] = mapped_column(DateTime, default=datetime.datetime.utcnow, nullable=False)
    updated_at: Mapped[datetime.datetime] = mapped_column(DateTime, default=datetime.datetime.utcnow, onupdate=datetime.datetime.utcnow, nullable=False)
```

Three uniform fields on every model: a UUID primary key, a
`created_at`, and an `updated_at` that auto-updates on every
write.  The UUIDs are generated client-side
(Python `uuid.uuid4`) rather than server-side, so an inserter
that needs to know the ID before commit (to write a related
record in the same transaction) can compute it.

A single Alembic migration is currently shipped
(`persistence/migrations/versions/71afa3096c71_initial_migration.py`).
Future schema changes will land as additional migrations in
the standard Alembic style.

## 4.10 The metadata refresh path

A registered pod can be queried at any time for its current
state — its portal list, its trace configuration, its remote-
control reachability.  This is the metadata-refresh path
(`registration/metadata_refresh.py`).  It uses the
`CommandDispatcher`
(`debugcontrol/command_dispatcher.py:21-74`) — the JSON-RPC
client that connects *out* from the server *to* the pod's
remote-control server (ch. 5):

```python
def dispatch(self, remote_address: str, command: str, params=None) -> Dict[str, Any]:
    ...
    request = {
        "command": command,
        "params": params,
        "token": self.credential_token,
    }

    request_json = json.dumps(request) + "\n"

    try:
        with socket.create_connection((host, port), timeout=self.timeout_seconds) as sock:
            with sock.makefile("rw", encoding="utf-8") as f:
                f.write(request_json)
                f.flush()

                response_json = f.readline()
                ...
                response = json.loads(response_json)
                ...
                return response.get("data", {})
```

The dispatcher is the *server-as-client* end of the same
JSON-lines protocol the pod's `RemoteControlServer` speaks
(ch. 5).  A line-based JSON request — `{"command": ..., "params":
..., "token": ...}` — followed by a line-based JSON response
— `{"status": "ok"|"error", "data"|"message": ...}`.

The dispatcher is the *only* mechanism by which the server
reaches into a pod.  All Layer-2 operations
(portal discovery, portal activation/deactivation, trace-config
read/write) compose of one or more dispatcher calls.  Volume 8
ch. 5 walks the dispatcher's command set in detail.

## 4.11 Portal discovery is an upsert

A canonical Layer-2 operation is *portal discovery* — pulling
the pod's current portal list and reconciling it with the
local snapshot.  `PortalDiscoveryService.discover`
(`debugcontrol/portal_discovery.py:23-72`):

```python
def discover(self, process_id: uuid.UUID) -> List[PortalInfo]:
    process = self._process_registry.get_process(process_id)
    if not process:
        raise ValueError(f"Process with ID {process_id} not found.")

    if not process.remote_control_address:
        raise ValueError(f"Process {process_id} has no remote control address.")

    response_data = self._command_dispatcher.dispatch(
        process.remote_control_address,
        "list_portals"
    )

    portals_data = response_data.get("portals", [])
    portal_infos = []
    for p in portals_data:
        portal_infos.append(PortalInfo(name=p.get("name"), is_active=p.get("is_active", False)))

    # Update local snapshots
    db_session = self._session_factory()
    try:
        existing_snapshots = db_session.query(PortalSnapshot).filter_by(process_id=process_id).all()
        existing_map = {s.portal_name: s for s in existing_snapshots}

        for p_info in portal_infos:
            if p_info.name in existing_map:
                snapshot = existing_map[p_info.name]
                snapshot.is_active = p_info.is_active
                del existing_map[p_info.name]
            else:
                snapshot = PortalSnapshot(
                    process_id=process_id,
                    portal_name=p_info.name,
                    is_active=p_info.is_active
                )
                db_session.add(snapshot)

        # Remaining in existing_map were not returned by pod, so delete them
        for s in existing_map.values():
            db_session.delete(s)

        db_session.commit()
    finally:
        db_session.close()

    return portal_infos
```

The reconciliation is an upsert with deletion: portals the pod
returned that the database already knew → update; portals the
pod returned that the database did not know → insert; portals
the database knew that the pod did not return → delete.

The deletion case matters.  A pod restart with a different code
path may register fewer portals than the database remembered;
the discovery sweep must remove the no-longer-present portals
or the server's portal list will accumulate dead names.

## 4.12 The receive path under load

A useful exercise to ground the architecture is to follow a
single received byte from the network through the full path.

A pod opens a TCP connection to the server.  The receiver
accepts.

The handshake reads two bytes — the length prefix of the auth
token.  Suppose the token is 36 bytes (a UUID hex).  The
receiver reads 36 more bytes, decodes them as UTF-8, hashes the
result with SHA-256, and looks up the hash in the credentials
table.  The credential is valid; the receiver responds with
`\x01`.

The receiver constructs a `StreamSessionLifecycle` for the
connection and starts the session loop on a worker thread.  At
the same moment, a `process_id` is associated with the session
— derived from the credential's owner.

The pod begins streaming envelope-prefixed events.  The first
recv pulls 4 KiB; the session calls
`storage_backend.append(chunk)`.  In the filesystem backend,
this opens the temp file (or appends to an already-open one)
and writes the bytes; the file's `mtime` updates.

The session updates the flow controller: `update_buffer_size`
takes the lock, increments `connection_buffers[connection_id]`
by 4096, increments `global_memory_used` by 4096, and returns
`OK` (we're well under the high watermark).

The session does not deserialise.  The next iteration of the
loop reads another 4 KiB.

After many chunks, the buffer for this connection has grown
larger than the high watermark.  The session emits `b"PAUSE"`
back to the pod.  (The pod, today, does not honour this
signal; if the pod is also pushing too fast, drop-on-saturation
on the pod side eventually trips at the *socket* level when
the kernel's sendbuf fills, and the data is dropped before the
flow-control message would have helped.)

Eventually the storage backend has caught up — `mtime` updates
have flushed page cache to disk — and a flush marker
`b"FLUSH"` arrives in a chunk.  The session calls
`_handle_flush`, which invokes `storage_backend.flush()` and
optionally finalises the recording (renames temp → final).
The session then resets `buffer_size` to zero and the flow
controller transitions back to `RESUME`.

The pod closes the connection on gateway exit.  The session's
receive loop sees `chunk = b""`, breaks, runs cleanup
(`storage_backend.commit()`), and the worker thread exits.  The
trace session record is updated with `stopped_at = utcnow()`,
`status = "completed"`, and the `data_reference` of the
finalised file.

## 4.13 What ingestion does not do

Three deliberate non-features of the ingestion tier are worth
calling out.

**No deserialisation on the receive path.**  As §4.3 notes,
bytes go to storage as bytes.  The cost of deserialisation
lives entirely in the analysis tier (ch. 6).  This makes the
ingestion path predictably-fast — `recv`, `append`, `update
buffer size`, repeat — and lets the server accept many
concurrent streams without each one paying the pickle overhead
in the receive thread.

**No event-level acknowledgement.**  The pod does not get a per-
event "received" signal from the server.  The protocol's
durability story is at the *flush boundary*: the pod knows that
events emitted before its last successful `flush` are at least
in flight; events emitted after are in the local buffer.  The
server's commit (the temp → final rename) happens at the flush
marker, not per event.  Per-event ack would be possible but
would multiply the wire round trips and is not worth the cost.

**No magic-number framing.**  The protocol relies on the
envelope's self-describing length fields to advance from one
event to the next.  An out-of-sync reader has no way to seek
forward to the next event start.  This is a pragmatic
limitation; ch. 3 §3.13 recommends versioning the envelope
when the protocol stabilises and adding a magic number at the
same time.

## 4.14 The honesty point on wiring

Two sub-systems exist in the source today that are functionally
*not yet wired into the receive path*:

The first is the format mismatch from ch. 3 §3.8: the
deserialiser's expected format is not the format the pod is
emitting.  The receive path stores bytes; the analysis path
deserialises bytes; the analysis path is not currently
exercised against pod-emitted bytes.

The second is the REST API tier.  The
`codecraftai-webworks` blueprint pattern is in place for every
specced endpoint, OpenAPI docs render from the route
declarations, and the namespaces register at startup —
but every endpoint body returns hand-coded mock data.  The
service-layer modules (`RegistrationService`,
`ProcessRegistry`, `DebugSessionManager`,
`PortalDiscoveryService`, `TraceSearchEngine`,
`CallGraphBuilder`, `VariableTracker`) all exist and have unit
tests; what is absent is the wiring from each Layer-5 endpoint
into the corresponding service.

The book treats both honestly.  The architecture *intends*
to bind: the spec is locked, the algorithms are written, the
schemas are migrated, the protocol is the protocol the wire
will eventually carry.  The chapters that follow describe the
intent-as-locked, with notes where the wiring is still to land.

## 4.15 Bridging to the next chapter

Ingestion turns a stream of pod-emitted bytes into a durable
recording on disk and a `TraceSession` row in the database.
The next chapter steps sideways from the *trace stream* to the
*control stream* — the JSON-lines protocol the server uses to
*talk to* a registered pod over its remote-control port: list
portals, activate them, deactivate them, and (in the spec, not
yet in the pod's implementation) read or update the trace
configuration.

→ Continue to [`05-remote-debug-control.md`](05-remote-debug-control.md).

← Back to [`03-trace-event-protocol.md`](03-trace-event-protocol.md).
