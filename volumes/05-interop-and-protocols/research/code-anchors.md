# Volume 5 — Code-Anchor Index

## automojo-interop (chapters 1–3)

### Aspects (the delegation contract)

- `automojo-interop/source/packages/automojo/interop/aspects.py:24–32` — `ActionPattern(IntEnum)` with five members:
  - `SINGLE_CALL = 0`
  - `SINGLE_CONNECTED_CALL = 1`
  - `DO_UNTIL_SUCCESS = 2`
  - `DO_WHILE_SUCCESS = 3`
  - `DO_UNTIL_CONNECTION_FAILURE = 4`
- `aspects.py:34–40` — `LoggingPattern` with `ALL_RESULTS = 0`, `FAILURE_ONLY = 1`, `SUCCESS_ONLY = 2`.
- `aspects.py:43–62` — defaults: `DEFAULT_COMPLETION_TIMEOUT = 600`, `DEFAULT_COMPLETION_INTERVAL = 10`, `DEFAULT_CONNECTION_TIMEOUT = 16`, `DEFAULT_CONNECTION_INTERVAL = 2`, `DEFAULT_ALLOWED_ERROR_CODES: Tuple[int] = []`, `DEFAULT_ALLOWED_CONNECTION_EXCEPTIONS: Tuple[BaseException] = []`, `DEFAULT_MUST_CONNECT = False`.
- `aspects.py:63–130` — `Aspects(Freezable)` base class; fields: `action_pattern`, `completion_timeout`, `completion_interval`, `connection_timeout`, `connection_interval`, `inactivity_timeout`, `inactivity_interval`, `logging_pattern`, `allowed_error_codes`, `allowed_connection_exceptions`, `must_connect`, `logger`.
- `aspects.py:133–197` — `AspectsCmd(Aspects)` — adds `expected_status: int|Tuple[int]`, `user: Optional[str]`, `pty_params: Optional[dict]`.
- `aspects.py:202` — `DEFAULT_ASPECTS = Aspects()`.
- `aspects.py:203` — `DEFAULT_CMD_ASPECTS = AspectsCmd()`.

### Freezable (immutability discipline)

- `automojo-xmodules/source/packages/automojo/xmodules/freezable.py:13–29` — `Freezable` base.  `__init__` sets `__frozen__ = False`; `freeze()` sets `__frozen__ = True`; `__setattr__` (line 23) raises `AttributeError` after freeze.

### MonitoredScope (deadlock detection)

- `automojo-xmodules/source/packages/automojo/xmodules/xlogging/monitoredscope.py:34–200` — context manager that registers thread-scoped monitoring with global `ScopeMonitor` daemon.  Used by `LocalCommandAgent.run_cmd:104` and `SshBase.run_cmd:545`.

### Agents (in `automojo-interop`)

- `automojo-interop/source/packages/automojo/interop/agents/systemagentbase.py:19` — `SystemAgentBase` (pure base; derived by `LocalCommandAgent`).
- `automojo-interop/source/packages/automojo/interop/agents/localcommandagent.py:35–247` — `LocalCommandAgent(ISystemContext)`; `run_cmd(command, exp_status, aspects)` at line 70.  Implements all five `ActionPattern` modes via `subprocess.Popen` + `MonitoredScope` + `TimeoutContext`.
- `automojo-interop/source/packages/automojo/interop/agents/poweragentbase.py:24–68` — `PowerAgentBase`.

### Protocols shipped *in* automojo-interop (paramiko-wrapping versions)

- `automojo-interop/source/packages/automojo/interop/protocols/ssh/sshbase.py:61–627` — `SshBase(ISystemContext)`.  `run_cmd` at line 241 implements all five `ActionPattern` modes; `_create_client` at line 270; `_log_command_result` per `LoggingPattern`.
- `automojo-interop/source/packages/automojo/interop/protocols/ssh/sshagent.py:37–150` — `SshAgent(SshBase)`; `open_session` at line 63; `open_tunnel` at line 98; `reboot` at line 131.
- `automojo-interop/source/packages/automojo/interop/protocols/ssh/sshsession.py:44–150` — `SshSession(SshBase)`; context-managed; uses `paramiko.SSHClient`.
- `automojo-interop/source/packages/automojo/interop/protocols/ssh/sshaspects.py:19–27` — paramiko-allowed exceptions (`AuthenticationException`, `NoValidConnectionsError`, `BadAuthenticationType`).
- `automojo-interop/source/packages/automojo/interop/protocols/serial/tcpserialagent.py:27–150` — `TcpSerialAgent(SerialAgentBase)`; uses `telnetlib.Telnet`.
- `automojo-interop/source/packages/automojo/interop/protocols/dns/` — DNS / mDNS support; record types, mDNS browser, validation, flags.
- `automojo-interop/source/packages/automojo/interop/protocols/power/dlipower/` — DLI power switch support.

### `am-interop` CLI

- `automojo-interop/source/packages/automojo/interop/cli/am_interop_command.py:48–74` — `@click.group("am-interop")` entry.  Uses click (NOT arcli).
- `automojo-interop/source/packages/automojo/interop/cli/cmdtree/` — currently empty (only README.md); `discover_command_groups()` mechanism exists but no subcommands shipped.

## automojo-protocol-ssh (chapter 4 — "SSH from scratch")

This is a **separate from-scratch SSH implementation** distinct from the paramiko-wrapping `automojo-interop/protocols/ssh/`.  Five-layer stack:

### Layer 1 — Packets

- `automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshpacket.py` (132 lines) — `SshPacket` with `add_string`/`add_mpint` (lines 80–100), `to_bytes()` (lines 105–116).

### Layer 2 — Transport (key exchange + encryption)

- `sshtransport.py` (685 lines) — `SshTransport(threading.Thread)`.  `connect()` at line 78 (handshake + KEX); `run()` at 96 (main loop); `send_packet()` at 140; `_read_packet()` at 262; `send_kex_init()` at 227; `_handle_kexdh_reply()` at 450; `_activate_encryption()` at 579–631 (RFC 4253 KDF).

### Layer 2 — Connection (channel multiplexing)

- `sshconnection.py` (223 lines) — `SshChannel(threading.Thread)` at line 14; `send`/`recv`/`request_pty`/`request_exec`/`request_shell`.  `SshConnection` at 161 multiplexes channels.

### Layer 3 — Authentication

- `sshauth.py` (89 lines) — `SshAuthError` at line 11; `SshAuthAgent` at 27.  Methods: `"none"`, `"password"`.
- `sshkeys.py` (197 lines) — `SshRSAKey`, `SshEd25519Key`; PEM loading; `sign`/`verify`.

### Layer 4 — Client wrapper

- `sshclient.py` (133 lines) — high-level SSH client wrapper.

### Layer 5 — Agents and sessions

- `sshbase.py` (781 lines) — abstract foundation; `_run_cmd` template method at 501–632.
- `sshagent.py` (376 lines) — stateless `SshAgent`; `open_session` at 64; `open_tunnel` at 99; `run_cmd` at 186.
- `sshsession.py` (429 lines) — stateful `SshSession`; context manager.
  - `__enter__` at 88–176; `__exit__` at 178–186; `run_cmd` at 356–370; `open_session` (basis-session reuse) at 293–326.

### Specialised features

- `sshsftp.py` (294 lines) — `SshSFTPClient`; SFTP v3 over channel; `SshSFTPError` at line 26.
- `sshtunnel.py` (371 lines) — `SshTunnel` for port forwarding; `_start_tunnel` at 157; `_forward_loop` at 183; `_pump_channel` at 266.
- `sshhelpers.py` (684 lines) — primitive directory listing, file I/O fallbacks.
- `sshaspects.py` (228 lines) — `AspectsCmd(Freezable)` SSH-specific subclass with defaults: `DEFAULT_COMPLETION_TIMEOUT = 600`, `DEFAULT_CONNECTION_TIMEOUT = 16`, `DEFAULT_INACTIVITY_TIMEOUT = 600`, `DEFAULT_INACTIVITY_INTERVAL = 0.5`, `DEFAULT_MONITOR_DELAY = 30`.

## automojo-networking (chapter 5)

Foundation utility (not interop extension); zero dependency on `automojo-interop`.

- `interfaces.py:24–82` — `InterfaceClass` enum (`LOOPBACK`, `BRIDGE`, `WIRED`, `WIRELESS`, `TUNNEL`); `get_ipv4_address`, `get_all_ipv4_interfaces`, `encode_address`.
- `resolution.py:23–192` — `expand_ipv6_addr` (23–65); `get_arp_table` (68–102); `is_ipv4_address`/`is_ipv6_address` (105–156); `refresh_arp_table` (159–192).
- `multicast.py:25–199` — `create_multicast_listener_socket` and per-interface variant.
- `unicast.py`, `broadcast.py` — analogous.
- `arp.py` — ARP helpers.
- `trafficcapturecontext.py` — context-manager-based traffic capture.

## automojo-restclient-base (chapter 6)

- `restclientbase.py:41–88` — `RestClientBase` with `AUTH_LEAF`, `VERIFY_CERTIFICATE = True`, `AUTHENTICATION_FAILURES = (HttpForbiddenError)`.  `__init__` accepts `host`, `credential`, `port`, `scheme`, `headers`, `verify_certificate`, `aspects: AspectsRest = DEFAULT_REST_ASPECTS`.
- `aspectsrest.py:24–116` — `ActionPattern` (same five members) and `AspectsRest` with REST-specific defaults; immutable via `__setattr__` override (frozen pattern, not `Freezable` superclass).
- `restapigroup.py:1–43` — `RestApiGroup` with weak ref to client; `_parse_item` deserialiser.
- `itemsresult.py:210–275` — `ItemsResult(Generic[CT])` with `is_paged`, `items`, `paging`, `next_page_result`, `prev_page_result` (lazy fetch).
- `restclientgenerator.py:77–100` — `RestClientGenerator` with `REST_CLIENT_BASE_CLASS`, `GROUP_BASE_TYPE`, `SCHEMA_BASE_TYPE`, `ASPECTS_TYPE`, `PAGEDLIST_TYPE`; `DEFAULT_PAGINATION_DETECTION_HINTS` walks `page` / `per_page` query parameters.

## automojo-restclient-github-public (chapter 7)

- 571 Python modules total: ~525 schema files, 42 group files, 76 OpenAPI spec JSON files.
- `clients/public/groups/repos.py:84–162` — representative method `accept_invitation_for_authenticated_user` and `add_app_access_restrictions`.  Pattern: extract path params via templating, build query params, call `client.perform_call`, parse via `resp.json()` + `_parse_item`.
- `githubpublicrestclient.py:53–70` — `GithubPublicRestClient(GithubRestClientBase)` instantiates 42 API groups as attributes.
- `githubrestclientbase.py:34–49` — adds GitHub-specific Accept header (`vnd.github.v3+json`).

## automojo-interop-ext-{upnp, tasker, bluetooth, can} (chapter 8)

### UPnP

- `automojo-interop-ext-upnp/source/packages/automojo/landscaping/extensions/protocol/upnpprotocolextension.py:24–38` — `UpnpProtocolExtension(UpnpRootDevice, ProtocolExtension)` — dual inheritance.
- `automojo-interop-ext-upnp/source/packages/automojo/interop/protocols/upnp/` — protocol stack: model/devices, mapping, content, metadata.

### Tasker

- `automojo-interop-ext-tasker/source/packages/automojo/interop/protocols/tasker/taskercontroller.py:50–75` — `TaskerController` with `TaskerAspects`.

### Bluetooth (Bumble)

- `automojo-interop-ext-bluetooth/source/packages/automojo/interop/bluetooth/bluetoothcoordinator.py:10–72` — `BluetoothCoordinator` with background asyncio loop (no Aspects integration).

### CAN

- `automojo-interop-ext-can/source/packages/automojo/interop/ext/can/cancoordinator.py:9–49` — `CanCoordinator` wrapping `python-can`.  No Aspects integration.

### Registration pattern

- All four use Python implicit namespace packages via `pyproject.toml` `packages = [{include="automojo", from="source/packages"}]`.
- UPnP and Tasker integrate via `ProtocolExtension` inheritance + landscape coordinator's namespace discovery at startup.
- Bluetooth and CAN are standalone coordinators (no landscape integration).
