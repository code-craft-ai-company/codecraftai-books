# Chapter 3 — The Trace-Event Protocol

> **Abstract.** A recording is only as useful as the format that
> carries it from the recording site to the analysis tooling.
> The pod's serialiser turns a `TraceEvent` into a hybrid of a
> fixed-size `struct` envelope and a variable-size `pickle`
> payload, with the filename inlined between them.  The buffered
> writer batches envelopes until the buffer crosses 64 KiB, then
> flushes to a backend.  This chapter walks the format byte-for-
> byte, identifies the design decisions that shape it (constant-
> time scanning, full-fidelity payloads, lossy graceful
> degradation), and surfaces a real, documented mismatch between
> the pod-side serialiser as committed and the server-side
> deserialiser as committed.  The mismatch is part of the
> record; the book does not pretend it isn't there.

## 3.1 Why two parts

A trace event has two kinds of fields.

**Fixed, scalar fields** are the same shape on every event: the
event-type tag, the timestamp, the thread ID, the source line
number.  These are uniform — known size, known position — and
they are the fields a *fast scanner* needs to look at to decide
whether it cares about a particular event.  An off-line tool
that wants to "show every CallEvent in process P between
timestamps T1 and T2" can answer that question without
deserialising the payload, *if* the timestamp and the type tag
are encoded at known offsets in a constant-size header.

**Variable, structured fields** differ event-by-event.  A
`CallEvent` has a function name and an arguments dict; a
`VariableSnapshotEvent` has a locals dict that is whatever
happened to be in scope.  These payloads need full Python
fidelity — the dict may contain arbitrary objects that the
recording wants to preserve as-is.  A scanner that does not
care about a particular event should not pay for deserialising
this payload; a tool that does care should be able to
deserialise it back to the original Python object.

Two requirements: *fast scanning* and *full fidelity*.  The
classical way to satisfy both is the *envelope-and-payload* split
— a fixed-size envelope using `struct` for fast access, and a
variable-size payload using `pickle` for full fidelity.  The
pod's serialiser
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/traceserializer.py:39`)
implements this split.

The Volume 1 ch. 5 JSOS streaming format makes the same trade-
off.  The pod's serialiser is conceptually JSOS one tier up:
the *frame* of a JSOS recording becomes the *envelope* of a
trace recording; the *payload* of a JSOS frame becomes the
*pickled body* of a trace event.

## 3.2 The pod-side envelope

The envelope format string is the second line of business
inside `TraceSerializer`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/traceserializer.py:28`):

```python
# Format: ! = network byte order (big-endian)
#   B  = event_type       (1 byte,  unsigned char)
#   Q  = timestamp_ns     (8 bytes, unsigned long long)
#   Q  = thread_id        (8 bytes, unsigned long long)
#   I  = filename_length  (4 bytes, unsigned int)
#   I  = lineno           (4 bytes, unsigned int)
#   I  = payload_length   (4 bytes, unsigned int)
ENVELOPE_FORMAT = "!BQQIII"
ENVELOPE_SIZE = struct.calcsize(ENVELOPE_FORMAT)  # 29 bytes
```

Twenty-nine bytes, fixed.  Network byte order (big-endian) so
the recording is portable across hosts that may have different
native ordering.  Six fields:

| offset | size | name             | type    | what it carries                    |
|--------|------|------------------|---------|------------------------------------|
| 0      | 1    | event_type       | uint8   | one of `EventType.*` (1-9)         |
| 1      | 8    | timestamp_ns     | uint64  | offset from gateway start, ns      |
| 9      | 8    | thread_id        | uint64  | `threading.get_ident()`            |
| 17     | 4    | filename_length  | uint32  | length of the inline filename str  |
| 21     | 4    | lineno           | uint32  | line number in the source          |
| 25     | 4    | payload_length   | uint32  | length of the pickle body          |

The serialise method
(`traceserializer.py:48-84`) packs the envelope, appends the
filename bytes (UTF-8), and appends the pickled payload:

```python
def serialize(self, event: TraceEvent) -> bytes:
    payload_data = self._extract_payload(event)

    if payload_data is not None:
        payload_bytes = pickle.dumps(payload_data, protocol=5)
    else:
        payload_bytes = b""

    filename = self._extract_filename(event)
    if filename:
        filename_bytes = filename.encode("utf-8")
    else:
        filename_bytes = b""

    lineno = self._extract_lineno(event)

    envelope = self._pack(
        event.event_type,
        event.timestamp_ns,
        event.thread_id,
        len(filename_bytes),
        lineno,
        len(payload_bytes),
    )

    return envelope + filename_bytes + payload_bytes
```

The full byte layout is:

```
+---------+-------------+-------------+
|envelope | filename    | pickle body |
| 29 B    | (variable)  | (variable)  |
+---------+-------------+-------------+
```

Pickle protocol 5 is used.  Protocol 5 is the modern Python
pickle format (PEP 574) that supports out-of-band buffers — for
the pod's purpose, it is the fastest and most compact pickle
available on supported Python versions.  Protocol 5 is *not*
backward-compatible to Python 3.7; the pod is Python 3.8+ in
practice.

## 3.3 What goes in the payload

`_extract_payload`
(`traceserializer.py`) constructs the per-event-type payload
dict.  Each event type has its own payload shape:

| event             | payload                                                |
|-------------------|--------------------------------------------------------|
| `CallEvent`       | `{"function_name": str, "arguments": dict (optional)}` |
| `ReturnEvent`     | `{"function_name": str, "return_value": Any}`          |
| `LineEvent`       | `None` — line events are envelope-only                 |
| `VariableSnapshot`| `{"local_variables": dict}`                            |
| `ExceptionEvent`  | `{"exc_type": str, "exc_value": str, "traceback": str}`|
| `PortalEnter`     | `{"portal_name": str}`                                 |
| `PortalExit`      | `{"portal_name": str}`                                 |
| `GatewayStart`    | `{"wall_clock_time": float}`                           |
| `GatewayStop`     | `{"wall_clock_time": float}`                           |

The most expensive payload is `VariableSnapshotEvent` — its
`local_variables` is a literal `dict(frame.f_locals)` snapshot,
and `frame.f_locals` can include objects of any size.  A trace
that captures variables in a hot loop generates a payload per
line; a trace that captures variables in a function with a
large local dictionary generates a large payload per line.

The cheapest event with a body is `LineEvent` — it has no
payload at all.  Its information is fully encoded in the
envelope (file, line, timestamp, thread).  The serialiser
notices and writes a zero-length payload bytes string
(`payload_bytes = b""`); the deserialiser sees `payload_length
== 0` in the envelope and skips the pickle.load entirely.

## 3.4 The buffered writer

A trace produces events at the rate of code execution.  At
calls-only granularity, an event is emitted on every function
entry and exit; at lines, on every line; at variables, on every
line plus a heavy-payload sibling.  Writing each event
separately to the backend would impose syscall and (for TCP)
network round-trip cost per event — orders of magnitude more
than the work to construct the bytes.

`BufferedTraceWriter`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/bufferedwriter.py:13`)
is the layer that batches.  Its surface is small:

```python
def write(self, data: bytes) -> None:
    self._buffer.extend(data)
    if len(self._buffer) >= self._max_size:
        self.flush()

def flush(self) -> None:
    if len(self._buffer) > 0:
        self._stream.write(bytes(self._buffer))
        self._buffer.clear()
```

The buffer is a `bytearray` — appendable in-place, no copy on
every event.  The flush threshold is 65,536 bytes (64 KiB) by
default, set when the recorder is constructed
(`portal.py:202`).  When `flush()` runs, the buffer is converted
to immutable `bytes` (one copy) and handed to the underlying
stream.

The flush threshold is a tuneable.  Larger thresholds amortise
syscall cost across more events at the cost of higher loss on
abrupt termination — if the process crashes with an unflushed
buffer, every event in that buffer is lost.  Smaller thresholds
trade throughput for durability.  The 64 KiB default is the
typical sweet spot for "the application is shipping events to a
local TCP backend on the same host".

A second flush trigger lives outside the writer: the gateway's
`__exit__`
(`gateway.py:88-92`) calls `flush()` and `close()` on every
recorder before letting the gateway tear down.  This is the
*orderly* shutdown path.  An *abrupt* shutdown — a crash, a
SIGKILL — has no way to flush; the buffer's contents are lost.

The pod does not, by design, install signal handlers or atexit
hooks to attempt a flush on abnormal termination.  Two reasons:
first, the kinds of failure that produce abnormal termination
(segfault, OOM, SIGKILL) are precisely the kinds where
attempting more I/O is unreliable.  Second, the recording's
*usefulness* under abrupt termination is the data already
streamed to the backend up to the last successful flush — that
is what the operator reads to understand the failure boundary.
Trying to extend that envelope by guessing at the buffer's
contents is rarely worth the complexity.

## 3.5 The backend layer

`bufferedwriter` writes its flushed bytes to a *stream backend*.
The backend is the abstraction that turns an opaque byte
sequence into a persisted artefact.  Three backends ship with
the pod
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/backends/`):

- `FileStreamBackend` — `file://path/to/trace.bin`, with
  optional rotation at a configurable byte size.
- `TcpStreamBackend` — `tcp://host:port`, streams to a server
  with reconnection.
- `UnixStreamBackend` — `unix:///path/to/socket`, streams over a
  Unix domain socket for same-host scenarios.

The backend registry
(`backends/registry.py:35-57`) routes a URI to a factory by
scheme:

```python
def create_backend(self, uri: str):
    parsed = urlparse(uri)
    scheme = parsed.scheme

    if scheme not in self._factories:
        errmsg = (
            f"Unknown stream backend scheme: '{scheme}'. "
            f"Known schemes: {list(self._factories.keys())}"
        )
        raise ValueError(errmsg)

    factory = self._factories[scheme]
    rtnval = factory(parsed)
    return rtnval
```

A new backend (`s3://bucket/prefix`, say, or `kafka://topic`) is
added by registering its scheme on the registry without changing
the existing pipeline.  This is the same URI-scheme dispatcher
pattern that Volume 5 ch. 4 documented for `automojo-interop`'s
transport layer.

## 3.6 The TCP backend's failure mode

The TCP backend
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/backends/tcpbackend.py:85-107`)
makes a deliberate choice on saturation:

```python
def write(self, data: bytes) -> None:
    """
        Send data over the TCP connection.

        On failure, attempts reconnection. If reconnection fails, the data
        is dropped and an error is logged.
    """
    try:
        self._socket.sendall(data)
    except OSError:
        if not self._reconnect():
            logger.error(
                "Failed to reconnect to %s:%d, dropping trace data",
                self._host, self._port
            )
            return
        try:
            self._socket.sendall(data)
        except OSError:
            logger.error("Write failed after reconnect, dropping trace data")
    return
```

If the send fails, attempt reconnection (with exponential
backoff — `tcpbackend.py:61-83`).  If reconnection fails, *drop
the data*.

This is a substantial design choice.  The alternative — a
bounded in-memory queue that fills up under saturation and then
applies back-pressure to the recording site — would prevent
drops at the cost of slowing the recorded code.  *Slowing the
recorded code is a worse outcome than losing some recording.*
The recorded code is the production application, the thing
serving real users.  A debugging backend that pushes back on
its application is a debugging backend that is causing the
incident it was meant to diagnose.

The drop-on-saturation choice has consequences that ch. 4 picks
up: the server's flow-control mechanism is the polite way to
*request* slowdown, but the pod is not contractually obliged to
honour it.  If the pod cannot keep up, the pod drops, and the
recording becomes lossy.

The contrast with the file backend is informative.  The file
backend
(`backends/filebackend.py:79-89`) does not drop:

```python
def write(self, data: bytes) -> None:
    self._file.write(data)
    self._bytes_written += len(data)
    if self._max_file_size is not None and self._bytes_written >= self._max_file_size:
        self._rotate()
    return
```

A local file write is, in the typical case, fast and reliable.
A `write()` to a healthy disk takes microseconds.  No queue
depth, no reconnection, no drop logic.  If the disk fills, the
exception propagates — the recorded application sees an
`OSError` and decides what to do (which, per the pod's design,
will probably be "log it and continue if possible").

The Unix-domain backend
(`backends/unixbackend.py:49-57`) is a TCP-shaped backend over
a Unix socket and follows the TCP backend's drop-on-saturation
behaviour.  The same trade-off applies: if the consumer on the
other end of the Unix socket falls behind, the pod drops rather
than blocking.

## 3.7 The reader (pod side)

The pod ships a reader as a partial implementation
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/reader.py`,
117 lines).  Its purpose is two-fold: as a tool to inspect a
trace file produced by the pod itself, and as the algorithmic
reference that the server-side deserialiser is meant to mirror.

The `read_events` generator (lines 43-60) is the canonical
inverse of `serialize`:

```python
def read_events(self):
    for source in self._sources:
        data = self._read_source(source)
        offset = 0
        while offset < len(data):
            event, offset = self._deserializer.deserialize(data, offset)
            yield event
```

The deserialise loop walks the byte stream, parsing one
envelope at a time, advancing the offset, and yielding the
reconstructed `TraceEvent` subclass.  This is the right mental
model for the format: it is a *stream* of self-describing
envelopes, each followed by its filename and payload, packed
back-to-back.  No frame markers, no length prefixes other than
the envelope's own width fields, no magic number.

The absence of a magic number is worth flagging.  An advantage:
no per-event overhead beyond the envelope's six fields.  A
cost: a corrupted or partial stream cannot be resynchronised by
seeking forward to the next valid frame.  If a writer crashes
mid-envelope, the bytes after that point are unparseable; a
robust reader breaks at the corruption rather than recovering
to the next event boundary.

The `scan_headers` method
(`reader.py:80`) is declared in the source but truncated — its
intent (yield envelope-only tuples without deserialising the
payload) is right but the implementation is incomplete.  This is
the partial-maturity signal mentioned in §1.4.  An off-line tool
that wanted to scan a trace cheaply (count CallEvents, filter
by thread ID, find the timestamp window) would use
`scan_headers`; today it must use `read_events` and pay the
pickle cost on every event.

## 3.8 The mismatch

This is the chapter's honesty point.

The pod-side serialiser
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/traceserializer.py:28`)
emits the envelope described in §3.2 — 29 bytes, fixed,
big-endian, six scalar fields, then filename, then pickle.

The server-side deserialiser
(`automojo-timetravel-server/source/packages/automojo/timetravelserver/streamreceiver/deserializer.py:15-65`)
parses a *different* format:

```python
def deserialize_stream(self, stream, process_id, session_id, portal_name):
    while True:
        # Read type length
        type_len_data = stream.read(1)
        if not type_len_data:
            break

        type_len = type_len_data[0]

        # Read type string
        type_str_data = stream.read(type_len)
        ...
        event_type_str = type_str_data.decode('utf-8')
        event_type = EventType(event_type_str)

        # Read timestamp (double) and payload len (unsigned int)
        header_data = stream.read(12)
        ...
        timestamp, payload_length = struct.unpack("!d I", header_data)

        # Read payload
        payload_data = stream.read(payload_length)
        ...
        payload = pickle.loads(payload_data)

        event = self._materialize_event(event_type, timestamp,
                                        process_id, session_id,
                                        portal_name, payload)
        yield event
```

The server's deserialiser expects:

```
+----+--------------+-----------+--------------+--------------+
|len | type_str     | timestamp | payload_len  | payload      |
| 1B | UTF-8 (≤255) | 8B BE     | 4B BE uint32 | (pickle)     |
|    |              | double    |              |              |
+----+--------------+-----------+--------------+--------------+
```

These are *different* formats:

| dimension          | pod emits                  | server expects                 |
|--------------------|----------------------------|--------------------------------|
| event-type field   | `uint8` integer (1-9)      | `uint8` length + UTF-8 string  |
| timestamp field    | `uint64` ns offset         | IEEE-754 double                |
| thread_id field    | `uint64`                   | absent                         |
| filename_length    | `uint32`                   | absent                         |
| inline filename    | UTF-8 bytes                | absent (filename is in payload)|
| line number        | `uint32` in envelope       | absent (lineno is in payload)  |
| payload length     | `uint32` (after envelope)  | `uint32` (after timestamp)     |

A current pod streaming over `tcp://` to a current server's
`streamreceiver/tcp_receiver.py` will not produce parseable
events on the server side.  The server-side `EventType()`
constructor — a string enum — will receive an integer (the
pod's first byte) interpreted as a length, which will cause it
to read the next *N* bytes as a UTF-8 string and fail to match
any known event type.

This is not a hidden bug.  It is the visible state of two
halves of an in-development protocol whose envelope shapes have
diverged.  The book treats it as documentary signal:

- The pod's envelope is *richer*, in line with the JSOS-style
  fast-scan-first design (§3.1) — thread ID, filename, lineno
  inlined in the envelope, type as a single byte for compact
  scanning.
- The server's deserialiser is a *string-tagged* envelope —
  human-readable type strings, IEEE-754 timestamps, less
  scanning capability.  This shape is what the server-side
  `tracedata/events.py` model
  (`tracedata/events.py:15-49`) takes as input — the model
  treats `event_type` as a `str` enum, not an int.

The two designs are reconcilable, and the reconciliation will
land before the protocol stabilises.  Two paths the design can
take:

- **Pod side adopts the server's format.**  The pod's `struct`
  envelope changes shape to match what the deserialiser
  expects.  Cost: smaller events get larger (the type string
  `"variable_snapshot"` is 17 bytes, where the integer `4` is
  one); the fast-scan property degrades because type strings
  vary in length.
- **Server side adopts the pod's format.**  The server's
  deserialiser is rewritten to parse the pod's 29-byte envelope.
  Cost: the server's `EventType` model migrates from string to
  integer; the analysis modules that reach into events with
  `.event_type.value` need a mechanical update.

The book takes no position on which path will win.  What it
does take a position on: *the protocol on the wire today is the
pod's format, because the pod is the producer*.  Any analysis
the book does of the recording's bytes uses the pod's
serialiser as ground truth.  The server-side deserialiser will
be aligned to that ground truth before the protocol is
considered stable.

## 3.9 What the wire actually looks like

A short trace from a small, instrumented function — a call,
two lines, a return — produces this byte stream out of the pod
(annotated):

```
# CallEvent — function "process_invoice" at invoice.py line 42
# event_type=1 (CALL), ts=1500ns, tid=140000000000000
# filename_length=11, lineno=42, payload_length=N
01 00000000000005DC                 # type=1, ts=1500
   0000000000000000 00007F4...      # tid
   0000000B 0000002A 000000NN       # fname_len=11, lineno=42, payload_len
   "invoice.py"                     # 11 bytes UTF-8
   <pickle of {"function_name": "process_invoice", "arguments": {...}}>

# LineEvent — line 43
01 00000000000005DD ...
03 ...                              # type=3 (LINE)
   0000000B 0000002B 00000000       # fname_len=11, lineno=43, payload_len=0
   "invoice.py"
   # no payload — LineEvent is envelope-only

# LineEvent — line 44
03 ...

# ReturnEvent — function "process_invoice"
02 ...                              # type=2 (RETURN)
   ...
   <pickle of {"function_name": "process_invoice", "return_value": "OK"}>
```

Approximate sizes:

| event             | envelope | filename | payload   | total  |
|-------------------|----------|----------|-----------|--------|
| CallEvent         | 29 B     | 11 B     | ~80 B     | ~120 B |
| LineEvent         | 29 B     | 11 B     | 0 B       | 40 B   |
| ReturnEvent       | 29 B     | 11 B     | ~50 B     | ~90 B  |
| VariableSnapshot  | 29 B     | 11 B     | 100-1k+ B | varies |

A function with 10 lines, no variable capture, traced at calls-
plus-lines, produces about 12 * 40 bytes ≈ 500 bytes of trace.
A function with 10 lines and variable capture (a small dict of
say 5 small values) produces about 12 * (40 + ~150) bytes ≈ 2.3
KB.  The cost scales with the *density* and *fidelity*
(§2.9).

## 3.10 Off-line analysis assumes the pod format

Reading a recorded trace off-line — outside the server, with
the pod's reader — uses the pod's deserialiser (the inverse of
the serialiser).  The pod's `reader.py`, `timeline.py`, and
`portalfilter.py` modules all consume `TraceEvent` objects whose
fields match the pod's data model: integer `event_type`,
`timestamp_ns`, `thread_id`, inline filename and lineno.  The
pod-side analysis modules are *internally consistent*.

The server-side analysis modules
(`tracedata/timeline.py`, `tracedata/call_graph.py`,
`tracedata/variable_tracker.py`, `tracedata/search.py`) consume
`TraceEvent` objects whose fields match the *server's* data
model: string `event_type`, float `timestamp`, no thread_id, no
inline filename.  These modules are also internally consistent.
Their unit tests (`source/testroots/testbase/test_*.py`)
construct events that match the server's model and exercise the
algorithms successfully.

What is not yet wired: the deserialiser at the boundary that
turns pod-emitted bytes into server-side `TraceEvent` objects.
The `reader.py` (server side) is a 44-line wrapper around
`HybridFormatDeserializer` that reads the server's expected
format from a file.  Until the deserialiser is reconciled to
the pod's format, the analysis tier is not exercised against
real pod traces — it is exercised against synthetic events
constructed in tests.

## 3.11 Lossy semantics under partition

The protocol's behaviour under network partition is asymmetric.

**Pod → server (TCP).**  If the network breaks while the pod is
streaming, the TCP backend
(`backends/tcpbackend.py:85-107`) attempts reconnection with
exponential backoff (`max_retries=3`, `retry_backoff=0.5s`).  If
all retries fail, the data is dropped and an error is logged;
the next successful write resumes the stream.  Events
*after* the partition resumes are recorded; events *during* the
partition are lost.

**Server → pod (control plane).**  If the network breaks while
a debug-control command is in flight, the
`CommandDispatcher`
(`debugcontrol/command_dispatcher.py:21-74`) raises a
`CommandError` to the calling code.  The pod's portal-activation
state is unchanged — the command was lost in transit, the pod
never received it.  The server's caller is responsible for
deciding whether to retry.

**Pod → server (heartbeat).**  If the network breaks while a
heartbeat is in flight, the heartbeat sender
(`pod/remote/heartbeat.py:75-82`) catches the exception, logs
it at WARNING, and waits for the next interval to try again.
On the server side, missing heartbeats are what drive the state
transitions
(`registration/heartbeat.py:67-100`): `active` → `stale` at
`T + heartbeat_timeout_seconds`, `stale` → `unreachable` at
`T + 2 × heartbeat_timeout_seconds`.

There is no acknowledgement protocol on the trace stream
itself.  The pod sends bytes; the server parses bytes; if the
parse fails, the server logs and drops the event.  If the pod
needs *guaranteed* delivery, the right answer is the file
backend with off-line ingestion — write to disk locally, then
have a separate process ship the file to the server when
network conditions allow.

## 3.12 The flush marker

The session lifecycle on the server
(`streamreceiver/session_lifecycle.py:36-72`) recognises a
literal byte string `b"FLUSH"` in the incoming stream as a
flush marker:

```python
if b"FLUSH" in chunk:
    self._handle_flush()
    # Reset buffer size on flush as data is persisted
    if self.flow_controller:
        buffer_size = 0
        ...
```

When the server sees the marker, it commits the storage backend
(persisting the buffered events to durable storage) and resets
the flow-control buffer size.  This is the protocol's
*pod-initiated* commit point: a place where the pod tells the
server "everything before this is a coherent unit; persist it".

The pod-side trigger for this is gateway exit
(`gateway.py:88-92`), which flushes every recorder and closes
its backend.  The mature TCP backend includes the marker in its
final write before tearing down the socket; an in-flight
recording that completes cleanly arrives at the server with a
trailing `FLUSH` marker, and the server commits.

A torn-down recording without the marker is *not* committed.
The server keeps the bytes in the temp area
(`persistence/trace_storage/filesystem_backend.py:32-42`) until
either a flush marker arrives or a timeout (configured at the
session lifecycle level) decides the connection is dead.  This
is intentional: a recording mid-stream is not a usable artefact;
durable persistence is reserved for committed recordings.

## 3.13 Versioning the protocol

The current envelope has no version field.  This is a pragmatic
choice — the protocol is in flux (§3.8), and adding a version
field today would lock in the wrong shape.  It is also a
liability — when the protocol stabilises, a future version will
require a flag-day upgrade or a dual-format reader.

The book's recommendation, made plain: when the protocol
reconciles to a single envelope shape, the envelope should
gain a leading `uint16` version field, and readers should
dispatch on it.  The version field's cost is two bytes per
event; the benefit is the ability to evolve the envelope in
small steps without flag days.  The cost is small enough that
the book treats this as the obvious right answer for the
post-reconciliation envelope.

## 3.14 Bridging to the next chapter

The protocol on the pod side — envelope, payload, buffered
flush, backend dispatch — is now in place.  The next step is
the *receiving* end: the server-side socket listeners, the
deserialiser the receiver hands bytes to, the per-connection
session lifecycle that brackets a recording, and the flow-
control mechanism that pushes back when the server is saturated.

→ Continue to [`04-server-ingestion.md`](04-server-ingestion.md).

← Back to [`02-pod-instrumentation.md`](02-pod-instrumentation.md).
