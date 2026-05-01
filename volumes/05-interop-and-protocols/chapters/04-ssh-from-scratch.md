# Chapter 4 — SSH from Scratch

> *Most projects use the paramiko-wrapping SSH agent in
> `automojo-interop`.  But the portfolio also ships a *from-
> scratch* SSH implementation in `automojo-protocol-ssh` —
> packets up to agents, no paramiko.  This chapter walks that
> stack as the volume's worked example of Measured
> Interoperability applied to a substantial protocol all the
> way down.*

A reader has two reasons to care about
`automojo-protocol-ssh`:

1. **Pedagogical**: the package shows what a full Measured
   Interoperability protocol looks like end to end, from
   binary packet framing up to the agent surface.  The
   chapter walks five layers; a reader who follows them gets
   a complete picture of the discipline applied to a
   complex protocol.
2. **Operational**: in environments where paramiko isn't
   available or has a known issue, the from-scratch stack is
   the alternative.  Its public API mirrors the paramiko-
   wrapping agent's, so swapping is a one-line change.

The chapter is the volume's longest.  By the end, a reader
should be able to answer "what happens when I call
`SshAgent.run_cmd(...)` on the from-scratch stack" at every
layer.

## The five layers

```
Layer 5 — Agents and Sessions     (sshbase / sshagent / sshsession)
                ↓
Layer 4 — Client wrapper          (sshclient.py)
                ↓
Layer 3 — Authentication          (sshauth.py / sshkeys.py)
                ↓
Layer 2 — Connection (channels)   (sshconnection.py)
                ↓
Layer 2 — Transport (encryption)  (sshtransport.py)
                ↓
Layer 1 — Packets                 (sshpacket.py)
```

The package's file count by layer (from
`automojo-protocol-ssh/source/packages/automojo/protocols/ssh/`):

- `sshpacket.py` — 132 lines.
- `sshtransport.py` — 685 lines.
- `sshconnection.py` — 223 lines.
- `sshauth.py` — 89 lines.
- `sshkeys.py` — 197 lines.
- `sshclient.py` — 133 lines.
- `sshbase.py` — 781 lines.
- `sshagent.py` — 376 lines.
- `sshsession.py` — 429 lines.
- `sshsftp.py` — 294 lines.
- `sshtunnel.py` — 371 lines.
- `sshhelpers.py` — 684 lines.
- `sshaspects.py` — 228 lines.

About 4,800 lines of from-scratch SSH code.  Substantial, but
small enough that a reader can hold the whole thing in their
head with this chapter as a guide.

## Layer 1 — Packets

**File:** `sshpacket.py` (132 lines)

The lowest layer.  An SSH packet is a binary blob with a
specific encoding rules.  `SshPacket` is the class that does
the encoding.

```python
class SshPacket:
    """
        Represents an SSH Packet and provides methods for
        packing/unpacking data types.
    """

    def __init__(self, message_id: int = None, payload: bytes = b''):
        self.message_id = message_id
        self.payload = bytearray(payload)
        self._position = 0
```

The class is two things:

- A **mutable buffer** while the packet is being constructed
  (`add_string`, `add_mpint`).
- A **read cursor** while the packet is being parsed
  (`get_byte`, `get_uint32`, `get_mpint`).

### Packing primitives

SSH's binary formats are specific.  Strings carry a 32-bit
length prefix:

```python
# Lines 80–84
def add_string(self, value: bytes) -> None:
    """
        Encode a string as uint32 length followed by the data.
    """
    length = len(value)
    self.payload.extend(struct.pack(">I", length))
    self.payload.extend(value)
    return
```

Multi-precision integers (used for cryptographic operations)
have a different shape:

```python
# Lines 86–100
def add_mpint(self, value: int) -> None:
    """
        Encode a multi-precision integer per RFC 4251 §5.
    """
    if value == 0:
        bytes_value = b''
    else:
        # Compute byte length; positive numbers prepend 0x00 if MSB is set
        byte_length = (value.bit_length() + 7) // 8
        bytes_value = value.to_bytes(byte_length, "big")
        if bytes_value[0] & 0x80:
            bytes_value = b'\x00' + bytes_value
    self.payload.extend(struct.pack(">I", len(bytes_value)))
    self.payload.extend(bytes_value)
    return
```

The `bytes_value[0] & 0x80` check is RFC 4251's rule:
positive integers whose most-significant bit would be
interpreted as a sign bit get a `0x00` prefix.

### Unpacking

The cursor-based read methods (`get_byte`, `get_uint32`,
`get_string`, `get_mpint`) advance `_position` as they read.
Errors are simple — running off the end of the payload
raises an explicit error.

### Packet finalisation

`to_bytes()` (lines 105–116) prepends the message ID:

```python
def to_bytes(self) -> bytes:
    """
        Return the on-wire representation: message_id + payload.
    """
    if self.message_id is None:
        raise ValueError("message_id must be set before to_bytes()")
    return bytes([self.message_id]) + bytes(self.payload)
```

Layer 1 is small and *pure* — no I/O, no state beyond the
buffer, no cryptography.  Just packing and unpacking.

## Layer 2 — Transport (encryption + key exchange)

**File:** `sshtransport.py` (685 lines)

The transport layer is the heart of SSH.  It handles:

- The TCP socket I/O.
- Version exchange ("SSH-2.0-...").
- Key exchange (Diffie-Hellman).
- Cipher activation.
- Message authentication codes (MACs).
- Packet sequencing.

`SshTransport` runs on its own thread:

```python
class SshTransport(threading.Thread):
    """
        SSH Transport Layer implementation.

        Handles encryption, decryption, compression, and key exchange.
        Run as a thread to handle incoming data.
    """

    def __init__(self, sock: socket.socket):
        self.sock = sock
        self.active = False
        self.encryptor = None
        self.decryptor = None
        self.seqno_out = 0
        self.seqno_in = 0
```

### The handshake

`connect()` at line 78 drives the SSH handshake:

1. Send the version banner (`SSH-2.0-automojo`).
2. Receive the server's banner.
3. Send a `KEXINIT` proposing algorithm sets.
4. Receive the server's `KEXINIT`.
5. Negotiate algorithms (key exchange, cipher, MAC,
   compression).
6. Perform Diffie-Hellman key exchange (`send_kex_init` at
   227, `_handle_kexdh_reply` at 450).
7. Verify the server's host key.
8. Activate encryption.

After step 8, the transport is in *encrypted mode*.  Every
packet is encrypted, MAC'd, and sequenced.

### Key derivation

The `_activate_encryption()` method at line 579 derives the
session keys from the Diffie-Hellman shared secret per RFC
4253:

```python
def _activate_encryption(self):
    """
        Derive session keys via the SSH KDF and activate
        encryption.

        Per RFC 4253:
            HASH(K || H || letter || session_id)

        where K is the shared secret, H is the exchange hash,
        and letter selects which key (A through F).
    """
    # K = f^x mod p (Diffie-Hellman shared secret, already computed)
    # session_id = H from the first key exchange

    c2s_iv = self._compute_key(b'A', iv_len)
    c2s_key = self._compute_key(b'C', key_len)
    c2s_mac_key = self._compute_key(b'E', mac_key_len)

    s2c_iv = self._compute_key(b'B', iv_len)
    s2c_key = self._compute_key(b'D', key_len)
    s2c_mac_key = self._compute_key(b'F', mac_key_len)

    cipher_enc = Cipher(algorithms.AES(c2s_key), modes.CTR(c2s_iv))
    self.encryptor = cipher_enc.encryptor()
    cipher_dec = Cipher(algorithms.AES(s2c_key), modes.CTR(s2c_iv))
    self.decryptor = cipher_dec.decryptor()
```

Six keys derived from a single shared secret using SHA1
(modern SSH uses SHA-256; the implementation supports both).
Three for client-to-server (encryption IV, encryption key,
MAC key), three for server-to-client.

### The packet I/O loop

`run()` at line 96 is the transport thread's main loop:

```python
def run(self):
    """
        Read packets from the socket; dispatch them to the
        connection layer.
    """
    while self.active:
        try:
            packet = self._read_packet()
            self.dispatch(packet)
        except Exception as ex:
            # Transport-level failures bubble up to the
            # session, which raises ConnectionError.
            self._fail(ex)
            return
```

Reading a packet (`_read_packet` at line 262):

1. Read 4 bytes (the encrypted length field — first block).
2. Decrypt the block; extract the packet length.
3. Read the rest of the packet (still encrypted).
4. Decrypt the remainder.
5. Verify the MAC against the decrypted payload.
6. Strip padding.
7. Increment `seqno_in`.

Sending a packet (`send_packet` at line 140) is the mirror.

### Failure modes

The transport layer raises `ConnectionError` on:

- Socket-level failure (broken pipe, timeout, refused).
- Key exchange failure (no common algorithm, bad signature).
- MAC verification failure.
- Packet framing error.

These bubble up through the layers above; eventually the
agent's `run_cmd` propagates them to the application layer
per principle 4.

## Layer 2 — Connection (channel multiplexing)

**File:** `sshconnection.py` (223 lines)

A single transport carries multiple *channels* — independent
streams multiplexed over the same encrypted connection.  A
typical SSH session uses one channel for the shell, another
for the SFTP subsystem, more for tunnels.

### `SshChannel`

```python
class SshChannel(threading.Thread):
    """
        Represents a logical channel within the SSH connection.
        Implements a file-like interface.
    """

    def __init__(self, connection, local_id: int):
        self.local_id = local_id
        self.remote_id = None
        self.max_packet_size = 32768
        self.window_size = 1024 * 1024
        self.in_buffer = Queue()
```

Each channel has:

- A `local_id` and `remote_id` — both sides of the
  connection assign IDs to track packets.
- A `window_size` — flow-control window; the receiver
  advertises how much data it can accept.
- An `in_buffer` — incoming data queued for the consumer.

Channel methods:

- `send(data)` (line 35) — chunks data, wraps in
  `SSH_MSG_CHANNEL_DATA`, gives to the transport.
- `recv(size, timeout)` (line 50) — pulls from `in_buffer`
  with optional timeout.
- `request_pty()` (line 80) — allocate a PTY for interactive
  shells.
- `request_exec()` (line 74) — execute a single command.
- `request_shell()` (line 71) — request an interactive shell.

### `SshConnection`

`SshConnection` at line 161 multiplexes the channels:

```python
class SshConnection:
    """
        Multiplexes channels over a single transport.
    """

    def dispatch_message(self, packet: SshPacket):
        """
            Route incoming packets to the correct channel based on
            channel ID.
        """
        # SSH_MSG_CHANNEL_DATA: route to channel.in_buffer
        # SSH_MSG_CHANNEL_OPEN_CONFIRMATION: complete channel open
        # SSH_MSG_CHANNEL_CLOSE: close channel
        # ... etc.

    def open_channel(self, kind="session") -> SshChannel:
        """
            Open a new channel of the given kind.
        """
        channel = SshChannel(self, local_id=self._next_id())
        # Send SSH_MSG_CHANNEL_OPEN; wait for confirmation.
        # Return the channel.
```

The connection layer's job is *traffic routing*.  It doesn't
care what the channels carry; it just delivers bytes from
the right packet to the right channel.

## Layer 3 — Authentication

**File:** `sshauth.py` (89 lines)

Authentication runs over the established transport.  The
client sends an `SSH_MSG_USERAUTH_REQUEST`; the server
responds with success, failure, or a list of further methods
to try.

```python
class SshAuthError(Exception):
    pass


class SshAuthAgent:
    """
        Handles Client-side User Authentication.
    """

    def authenticate_password(self, username: str, password: str):
        self._send_auth_request("password", password=password)
        self._wait_for_response()

    def authenticate_publickey(self, username: str, key: SshKey):
        # Send the key; sign a challenge; wait for confirmation.
        ...

    def _wait_for_response(self):
        # Block until SSH_MSG_USERAUTH_SUCCESS or USERAUTH_FAILURE.
        # On failure raise SshAuthError.
        ...
```

`SshAuthError` (line 11) is the typed exception that reverts
control to the application layer when authentication fails.
The application can catch it and decide whether to refresh
credentials, escalate, or fail the test.

The supported methods are:

- `"none"` — query the server for which methods it accepts.
- `"password"` — username + password.

(Additional methods like `"publickey"` are implemented in
`sshkeys.py` integration.)

## Layer 3 — Keys

**File:** `sshkeys.py` (197 lines)

Key handling for public-key authentication and host-key
verification.  Two key types supported:

```python
class SshRSAKey(HostKey):
    """
        RSA key — load from PEM, sign with SHA1 PKCS#1 v1.5.
    """

    @classmethod
    def from_private_key_file(cls, filename, password=None):
        # Load PEM RSA key
        ...

    def sign(self, data: bytes) -> bytes:
        return self.private_key.sign(data, padding.PKCS1v15(), hashes.SHA1())

    def verify(self, data: bytes, signature: bytes) -> bool:
        # Verify signature; returns False on InvalidSignature
        ...


class SshEd25519Key(HostKey):
    """
        Ed25519 key — load from PEM, sign with EdDSA.
    """

    def verify(self, data: bytes, signature: bytes) -> bool:
        self.public_key.verify(signature, data)  # raises on failure
```

Both implement the abstract `HostKey` interface with `sign`,
`verify`, `get_name`.  The `cryptography` library does the
underlying math; the wrapper's job is to translate to and
from the SSH wire format.

## Layer 5 — Agents and Sessions

The top of the stack.  `SshAgent` and `SshSession` are the
public API.

### `SshBase` — the shared foundation

**File:** `sshbase.py` (781 lines)

Both `SshAgent` and `SshSession` inherit from `SshBase`.
The base provides the *template method* `_run_cmd` (line
501–632) that walks the chosen `ActionPattern`:

```python
def _run_cmd(self, ssh_runner, command: str,
             exp_status: Union[int, Sequence] = 0,
             user: str = None,
             pty_params: dict = None,
             aspects: Optional[AspectsCmd] = None) -> Tuple[int, str, str]:
    # Resolve aspects fallback
    if aspects is None:
        aspects = self._aspects

    # Build the timeout / interval envelope
    completion_toctx = TimeoutContext(aspects.completion_timeout,
                                      aspects.completion_interval)
    inactivity_timeout = aspects.inactivity_timeout
    inactivity_interval = aspects.inactivity_interval
    monitor_delay = aspects.monitor_delay
    logging_pattern = aspects.logging_pattern

    # Wrap in MonitoredScope (chapter 2)
    if aspects.action_pattern == ActionPattern.SINGLE_CALL:
        with MonitoredScope("RUNCMD-SINGULAR", monmsg, completion_toctx,
                            notify_delay=monitor_delay):
            status, stdout, stderr = _ssh_execute_command(
                ssh_runner, command, ...)
        self._log_command_result(command, status, stdout, stderr,
                                 exp_status, logging_pattern)
        return status, stdout, stderr

    elif aspects.action_pattern == ActionPattern.DO_UNTIL_SUCCESS:
        # Retry loop honoring completion_toctx.
        while not completion_toctx.final_attempt:
            with MonitoredScope(...):
                status, stdout, stderr = _ssh_execute_command(...)
            self._log_command_result(...)
            if self._matches_expected(status, exp_status):
                return status, stdout, stderr
            completion_toctx.next_attempt(aspects.completion_interval)

        # Loop exhausted — raise per principle 4.
        raise completion_toctx.create_timeout(what_for="command success")

    elif aspects.action_pattern == ActionPattern.DO_WHILE_SUCCESS:
        # Mirror loop ...
```

The same shape across `SINGLE_CALL`, `DO_UNTIL_SUCCESS`, and
`DO_WHILE_SUCCESS`.  The `_ssh_execute_command` helper does
the actual SSH operation; the rest is the persistence
behaviour from chapter 1's principle list.

### `SshAgent` — stateless

**File:** `sshagent.py` (376 lines)

```python
class SshAgent(SshBase):
    """
        Stateless agent.  Each operation creates and destroys
        its own SSH client.
    """

    def open_session(self, user: str = None, primitive: bool = False,
                     pty_params: Optional[dict] = None,
                     interactive=False,
                     basis_session: Optional[SshSession] = None,
                     aspects: Optional[AspectsCmd] = None) -> SshSession:
        """
            Vend a SshSession bound to this agent's host and credentials.
        """
        session_credential = self._primary_credential
        if user is not None:
            session_credential = self._users[user]
        session = SshSession(self._host, session_credential, ...)
        return session

    def open_tunnel(self, local_port: int = 0,
                    remote_host: str = 'localhost',
                    remote_port: int = 5432) -> SshTunnel:
        """
            Open a port-forwarding tunnel.
        """
        ssh_client = self._create_client()
        try:
            tunnel = SshTunnel._create_internal(ssh_client, local_port,
                                                remote_host, remote_port)
            return tunnel
        except Exception as e:
            ssh_client.close()
            raise RuntimeError(f"Failed to create SSH tunnel: {e}") from e
```

`SshAgent.run_cmd` (line 186) creates a temporary client,
runs the command via `_run_cmd`, cleans up:

```python
def run_cmd(self, command, exp_status=0, user=None, pty_params=None,
            aspects=None, ssh_client=None):
    cleanup_client = False
    try:
        if ssh_client is None:
            ssh_client = self._create_client(user)
            cleanup_client = True
        status, stdout, stderr = self._run_cmd(ssh_client, command, ...)
    finally:
        if cleanup_client:
            ssh_client.close()
    return status, stdout, stderr
```

The optional `ssh_client` parameter lets callers reuse a
client (typically because they're inside an `SshSession` and
want to skip the create/destroy cycle).

### `SshSession` — stateful, context-managed

**File:** `sshsession.py` (429 lines)

```python
class SshSession(SshBase):
    """
        Stateful session.  Holds the SSH client open across
        multiple calls.  Used as a context manager.
    """

    def __enter__(self):
        # Lines 88–176
        ssh_client = self._create_client()
        if self._interactive:
            # Open a channel, request PTY, request shell, wait for prompt
            channel = ssh_client.connection.open_channel("session")
            channel.request_pty(...)
            channel.request_shell()
            self._wait_for_prompt(channel)
            self._ssh_runner = channel
        else:
            self._ssh_runner = ssh_client
        return self

    def __exit__(self, ex_type, ex_val, ex_tb):
        # Lines 178–186
        self.close()
        return False

    def run_cmd(self, command, exp_status=0, pty_params=None, aspects=None):
        # Lines 356–370
        status, stdout, stderr = self._run_cmd(self._ssh_runner, command,
                                               exp_status=exp_status,
                                               pty_params=pty_params,
                                               aspects=aspects)
        return status, stdout, stderr
```

The session's `__enter__` is more complex than `SshAgent`'s
single-call setup.  An interactive session opens a channel,
requests a PTY, requests a shell, waits for the shell's
prompt — so subsequent `run_cmd` calls execute *inside the
shell*, not as separate exec channels.

### Basis sessions — connection reuse

`SshSession.open_session` at line 293 supports a *basis
session* — a parent session whose client is reused:

```python
def open_session(self, user: str = None, ...) -> SshSession:
    """
        Open a derived session.  When this session has a basis,
        the new session reuses the basis's client.
    """
    if self._basis_session is not None:
        # Reuse the basis session's client.
        bs = self._basis_session
        session = SshSession(bs._host, session_credential, ...,
                             basis_session=bs, ...)
    else:
        session = SshSession(self._host, session_credential, ...)
    return session
```

A test that needs to run many commands as different users on
the same host benefits: open one basis session; derive
per-user sessions from it; the underlying SSH connection is
shared.  Avoids the overhead of multiple TCP connections
plus key exchanges.

## Specialised features

### SFTP

**File:** `sshsftp.py` (294 lines)

SFTP runs as a *subsystem* over an SSH channel:

```python
class SshSFTPClient:
    def __init__(self, channel):
        self.channel = channel
        self.request_id = 0
        self.channel.request_subsystem("sftp")
        self._init_sftp()
```

The protocol is request/response: send a numbered request,
receive the response with the same number.  Operations:

- `open(filename, pflags)` (line 56) — open a file, return
  handle.
- `read(handle, offset, length)` (line 90).
- `write(handle, offset, data)` (line 111).
- `listdir(path)` (line 126).
- `stat(path)` (line 173).

`SshSFTPError` (line 26) is the typed exception for
protocol-level errors.  `EOFError` (line 293) is raised
when the SFTP `SSH_FX_EOF` status code is received during
a read.

### Tunnels

**File:** `sshtunnel.py` (371 lines)

Port forwarding over SSH.  A `SshTunnel` listens on a local
port; for each incoming connection, opens a new SSH channel
to the remote target, then bidirectionally pumps bytes:

```python
class SshTunnel:
    def __init__(self, ssh_client: SshClient, local_port: int,
                 remote_host: str, remote_port: int,
                 _internal_create: bool = False):
        self._ssh_client = ssh_client
        self._local_port = local_port
        self._remote_host = remote_host
        self._remote_port = remote_port
        self._start_tunnel()

    def _forward_loop(self):
        # Lines 183
        # Accept incoming connections in a daemon thread.
        # For each: open SSH channel "direct-tcpip", spawn pump thread.

    def _pump_channel(self, sock, channel):
        # Lines 266
        # Use select() to relay bytes between socket and channel.
```

The use case: tests against a service that's only reachable
through a jump host.  Open a tunnel; point the test client at
`localhost:<local_port>`; the tunnel forwards traffic to the
service.

## Aspects — SSH-specific defaults

**File:** `sshaspects.py` (228 lines)

The from-scratch SSH stack ships its own `AspectsCmd` subclass
with SSH-tuned defaults:

```python
DEFAULT_COMPLETION_TIMEOUT = 600    # 10 minutes total
DEFAULT_CONNECTION_TIMEOUT = 16     # 16 seconds to establish SSH
DEFAULT_INACTIVITY_TIMEOUT = 600    # 10 minutes without progress
DEFAULT_INACTIVITY_INTERVAL = 0.5   # poll for output every 500ms
DEFAULT_MONITOR_DELAY = 30          # 30s buffer for MonitoredScope
```

These defaults are *the same* as the interop-level defaults
(chapter 2) — SSH doesn't need different baseline values
than other protocols.  The class exists to add SSH-specific
fields (e.g., paramiko-allowed exceptions for the
paramiko-wrapping version) where needed.

## A request flow — start to finish

A test calls `agent.run_cmd("uptime")`.  The full flow on the
from-scratch stack:

1. **Test code** calls `agent.run_cmd("uptime")`.
2. **`SshAgent.run_cmd`** — line 186.  No `ssh_client`
   passed; creates one.
3. **`SshAgent._create_client`** — opens TCP socket to host:port.
4. **`SshTransport.connect`** — performs SSH handshake;
   exchanges versions; runs Diffie-Hellman; activates
   encryption.
5. **`SshAuthAgent.authenticate_password`** — runs
   user-auth using the credential.
6. **`SshClient`** — wrapper around the now-authenticated
   transport.
7. **`SshAgent._run_cmd`** — line 186 redirects to
   `SshBase._run_cmd` (line 501).
8. **`SshBase._run_cmd`** — resolves aspects defaults,
   wraps the call in `MonitoredScope`, dispatches on
   `action_pattern` (default `SINGLE_CALL`).
9. **`_ssh_execute_command`** — opens a channel, sends
   `SSH_MSG_CHANNEL_REQUEST` with `exec` and the command;
   reads channel data until close.
10. **Status / stdout / stderr** — extracted from the channel.
11. **`SshBase._log_command_result`** — emits log per
    `LoggingPattern`.
12. **`SshAgent.run_cmd`** — closes the temporary client.
13. **Test code** receives `(status, stdout, stderr)`.

Each step has a chapter reference somewhere in this volume.
Steps 4–6 are layers 2–3 above; step 8's
`_run_cmd` walks the four principles from chapter 1.

## What the from-scratch stack gives you

Three things:

### Pedagogical clarity

Reading the from-scratch stack is the way to *understand*
SSH.  The paramiko-wrapping version hides the protocol
details behind paramiko's API.  The from-scratch version
shows every layer, every packet, every key exchange.

### Operational fallback

When paramiko has a known bug (and over the years it has),
the from-scratch stack is the alternative.  Its public API
mirrors the paramiko-wrapping agent's (`open_session`,
`run_cmd`, `open_tunnel`, etc.), so a project can swap
implementations with minimal disruption.

### A worked example for new protocols

A reader building a new protocol package follows this
chapter as a *template*.  Layer the protocol from packets up
to agents.  Apply the same Aspects-driven persistence
discipline.  Ship `LayerXBase` / `LayerXAgent` /
`LayerXSession` classes mirroring the SSH shape.

## Limitations

A few points of honesty about the from-scratch stack:

- **Algorithm support.**  The stack implements *enough* of
  SSH to talk to typical Linux / macOS servers.  Older
  algorithms are not supported.  Modern strict servers
  (FIPS-mode, restricted cipher lists) may need tuning.
- **Performance.**  Paramiko has had years of performance
  tuning.  The from-scratch implementation is correct but
  not aggressively optimised; for very high-throughput SFTP,
  paramiko is faster.
- **Rare features.**  Agent-forwarding, X11-forwarding,
  GSS-API authentication — not implemented.  Tests needing
  these features should use a paramiko-based agent or
  contribute the missing features.

A reader weighing which stack to use should treat the
from-scratch as the *educational* and *fallback* option, the
paramiko-wrapping as the *production* default.

## What the next chapter does

Chapter 5 walks `automojo-networking` — the foundation
utilities for DNS resolution, ARP table queries, multicast /
unicast / broadcast socket handling.  Where this chapter
walked one substantial protocol in depth, chapter 5 walks the
*primitives* every protocol package builds on.
