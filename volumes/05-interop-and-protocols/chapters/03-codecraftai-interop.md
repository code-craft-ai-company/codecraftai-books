# Chapter 3 — `codecraftai-interop` — Agents, Protocols, CLI

> *Chapters 1 and 2 walked the philosophy and the delegation
> contract.  This chapter walks the package itself — the
> Agent base classes, the shipped protocols (local commands,
> paramiko-based SSH, serial-over-TCP, DNS / mDNS, power),
> and the `am-interop` CLI scaffolding.  By the end, a reader
> can locate any Agent in the package and predict its shape.*

`codecraftai-interop` is a foundation tier package.  It supplies
the Agent base classes that protocol-specific code subclasses,
ships a handful of common protocols directly, and provides
the `am-interop` CLI as a command-line surface for ad-hoc
interop tasks.

This chapter is a *survey*.  Where chapter 4 deep-dives one
protocol (the from-scratch SSH stack), this chapter walks the
*breadth* of `codecraftai-interop` so a reader knows what's in
the package and where to find each thing.

## Package layout

```
codecraftai-interop/source/packages/ccai/interop/
├── aspects.py             # chapter 2 covered this
├── agents/
│   ├── systemagentbase.py     # base for all system agents
│   ├── localcommandagent.py   # local subprocess
│   └── poweragentbase.py      # base for power-control agents
├── protocols/
│   ├── ssh/                   # paramiko-wrapping SSH
│   │   ├── sshbase.py
│   │   ├── sshagent.py
│   │   ├── sshsession.py
│   │   └── sshaspects.py
│   ├── serial/                # serial-over-TCP
│   │   └── tcpserialagent.py
│   ├── dns/                   # DNS / mDNS support
│   └── power/                 # DLI power switches
├── cli/
│   ├── am_interop_command.py  # CLI entry
│   └── cmdtree/               # currently empty
└── ...
```

Three top-level concerns:

- **`agents/`** — Agent base classes and the local-command
  agent.
- **`protocols/`** — protocol-specific code shipped with the
  package.  SSH, serial, DNS, power.
- **`cli/`** — the `am-interop` command-line tool.

A note up front: the SSH code in `codecraftai-interop/protocols/ssh/`
is *paramiko-wrapping* — it uses the `paramiko` library
under the hood and adds the Aspects-driven persistence
behaviours on top.  This is the production SSH stack used by
most projects.  A *separate* package, `codecraftai-protocol-ssh`,
implements SSH from scratch (no paramiko).  Chapter 4 walks
the from-scratch implementation as an in-depth worked example;
most readers using SSH in production are using the
paramiko-wrapping version surveyed here.

## Agents — the base hierarchy

### `SystemAgentBase`

**File:**
`codecraftai-interop/source/packages/ccai/interop/agents/systemagentbase.py:19`

The pure abstract base.  Most production Agents inherit from
the protocol-specific bases (`SshBase`, `SerialAgentBase`,
`PowerAgentBase`) which themselves implement
`ISystemContext`; `SystemAgentBase` is the shared parent.  The
class itself has minimal behaviour — it's a marker that
declares "I'm an Agent" for type-checking and discovery.

### `LocalCommandAgent`

**File:**
`codecraftai-interop/source/packages/ccai/interop/agents/localcommandagent.py:35–247`

```python
class LocalCommandAgent(ISystemContext):
    """
        An agent for running commands on the local machine via
        subprocess.

        Implements all five ActionPattern modes via subprocess.Popen
        plus MonitoredScope plus TimeoutContext.
    """

    def run_cmd(self, command: str,
                exp_status: Union[int, Sequence] = 0,
                aspects: Optional[AspectsCmd] = None) -> Tuple[int, str, str]:
        """
            Run a command locally, return (status, stdout, stderr).

            Honours the chosen ActionPattern via the aspects parameter.
        """
        # ... implementation walks the action_pattern dispatch ...
```

The local-command agent is the simplest concrete agent in
the package.  It runs commands on the same machine via
`subprocess.Popen`.  Despite simplicity, it implements *all
five* `ActionPattern` modes — the same mechanism the SSH and
serial agents use.  This makes it a good reading-order
starting point for understanding the agents pattern: the
Aspects logic is identical across agents; only the underlying
mechanism differs.

A use case for `LocalCommandAgent` in a real test:

```python
from ccai.interop.agents.localcommandagent import LocalCommandAgent
from ccai.interop.aspects import AspectsCmd, ActionPattern

agent = LocalCommandAgent()

# Quick health check.
status, stdout, stderr = agent.run_cmd("which python3")

# Wait for a service that's starting up — DO_UNTIL_SUCCESS.
patient = AspectsCmd(
    action_pattern=ActionPattern.DO_UNTIL_SUCCESS,
    completion_timeout=60,
    completion_interval=2,
)
status, stdout, stderr = agent.run_cmd(
    "systemctl is-active myservice",
    aspects=patient,
)
```

The first call uses defaults; the second declares an
explicit aspects for the persistence behaviour.

### `PowerAgentBase`

**File:**
`codecraftai-interop/source/packages/ccai/interop/agents/poweragentbase.py:24–68`

The base for power-control agents — devices that can turn
outlets on and off, query outlet state, monitor power
consumption.  The shipped concrete implementation targets
DLI Web Power Switches (under
`interop/protocols/power/dlipower/`); a project with a
different power controller subclasses `PowerAgentBase` to
talk to its own hardware.

The base provides:

- `_lookup_power_interface(interface_name)` — routes the
  request to the right concrete `dlipower.PowerSwitch` based
  on configuration.
- Common state-query and state-set methods that subclasses
  fill in.

The class is an example of the framework's
"abstract-base-with-concrete-children" pattern from Volume 0
chapter 4.  An SDET adding a new power-controller integration
follows the pattern.

## Protocols shipped in `codecraftai-interop`

### SSH (paramiko-wrapping)

Three classes co-operate:

**`SshBase`** at `protocols/ssh/sshbase.py:61–627` is the
abstract foundation.  Both `SshAgent` and `SshSession`
inherit from it.  The class implements `_run_cmd` (line 241)
— the template method that walks the chosen `ActionPattern`
applying it to a paramiko-driven SSH command.  Five
`ActionPattern` modes; same algorithm as `LocalCommandAgent`,
different underlying mechanism.

**`SshAgent`** at `protocols/ssh/sshagent.py:37–150` is the
*stateless* SSH agent.  Each call creates and tears down its
own paramiko client.  Useful when:

- Calls are infrequent (no benefit from connection reuse).
- The remote host can't sustain a long-held connection.
- Concurrent calls to the same host benefit from independent
  sessions.

```python
class SshAgent(SshBase):

    def open_session(self, user: str = None, ...) -> "SshSession":
        """
            Vend an SshSession bound to this agent's host and
            credentials.
        """
        # Construct an SshSession; caller uses it as a context manager.
        session = SshSession(self._host, session_credential, ...)
        return session

    def open_tunnel(self, local_port: int, remote_host: str,
                    remote_port: int) -> "SshTunnel":
        """
            Open a port-forwarding tunnel.
        """
        # ...

    def reboot(self, aspects=None):
        """
            Issue a reboot to the remote host and wait for it to come back.
        """
        # ...
```

The agent vends `SshSession` instances and `SshTunnel`
instances.  These are the *stateful* siblings — a session
holds a paramiko client open across multiple calls; a tunnel
maintains an open port-forward.

**`SshSession`** at `protocols/ssh/sshsession.py:44–150` is
the *stateful* SSH session.  Used as a context manager:

```python
with agent.open_session(user="admin") as session:
    session.run_cmd("systemctl restart nginx", aspects=fast_aspects)
    session.run_cmd("systemctl is-active nginx", aspects=DO_UNTIL_SUCCESS)
```

The session holds the paramiko connection open for the
duration of the `with` block.  Multiple `run_cmd` calls reuse
the same connection.

### Serial-over-TCP

**File:**
`codecraftai-interop/source/packages/ccai/interop/protocols/serial/tcpserialagent.py:27–150`

```python
class TcpSerialAgent(SerialAgentBase):
    """
        Serial-console agent connected via TCP (a serial
        concentrator on the network).

        Uses telnetlib.Telnet under the hood; provides the same
        Aspects-driven interface as SshAgent.
    """
```

The serial agent connects to a *serial concentrator* — a
device that exposes serial ports as TCP sockets.  Serial
concentrators are common in lab gear: a single concentrator
holds eight or sixteen serial ports, each reachable over TCP
at a port number.  The agent telnets to the concentrator,
selects the right port, and operates the serial console.

The Volume 6 chapter 1 worked example of a Landscape file
showed device declarations like `serial: serial-concentrator: 1`
— the framework's Feature dispatcher (Volume 6 chapter 2)
resolves `serial-concentrator` to a `TcpSerialAgent` and
attaches a `SerialProtocolExtension` to the device.  A test's
`device.serial.write(...)` call goes through the extension to
the agent.

### DNS / mDNS

**Path:**
`codecraftai-interop/source/packages/ccai/interop/protocols/dns/`

The DNS module contains stateless utilities for DNS protocol
work:

- Record types (A, AAAA, SOA, MX, NAPTR, TXT, PTR).
- mDNS browser for local-network service discovery.
- Validation helpers.
- Flag definitions.

There's no `DnsAgent` class in the same shape as
`SshAgent` — DNS work is mostly stateless lookups, and the
shipped utilities are functions.  Projects that need a DNS
*polling* agent (e.g., "wait until this hostname resolves")
construct their own using `LocalCommandAgent` to invoke
`dig` with `DO_UNTIL_SUCCESS`, or use one of the mDNS
discovery utilities directly.

### Power

**Path:**
`codecraftai-interop/source/packages/ccai/interop/protocols/power/`

The shipped power implementation targets DLI Web Power
Switches:

```
power/
└── dlipower/
    ├── dlipowerclient.py   # the per-switch client
    └── ...
```

Subclasses of `PowerAgentBase` handle the protocol-level
details for each kind of power switch.  The `PowerSwitch`
class exposes outlets, supports on/off/cycle operations, and
provides metering when the hardware supports it.

A project automating against a different power switch (an APC
PDU, a custom relay board) writes its own
`PowerAgentBase` subclass.

## The `am-interop` CLI

**File:**
`codecraftai-interop/source/packages/ccai/interop/cli/am_interop_command.py:48–74`

```python
@click.group("am-interop")
@click.option('-v', '--verbose', count=True)
def am_interop_root_command(verbose):
    """
        am-interop — Command-line interface for the ccai interop layer.
    """
    # ...
```

A note on the CLI's current state: the `am-interop` CLI uses
`click` (not `arcli` from Volume 1 chapter 8) and the command
tree under `cli/cmdtree/` is **currently empty** (only a
`README.md` placeholder).  The discovery mechanism
(`discover_command_groups()` at line 24) exists; the
subcommands haven't been added yet.

The chapter is honest about this: the CLI is *scaffolded* for
ad-hoc interop tasks — running a command on a remote host,
opening a tunnel, querying DNS, controlling power — but the
shipped subcommands are still in flight.  A reader using
`codecraftai-interop` today uses the Python APIs directly; the
CLI will be useful when the subcommands land.

The framework's other CLIs (`testbase`, `taskplus`, etc.) use
arcli; that consistency is broken here.  Future work might
unify the CLI surface, but the chapter doesn't predict that.

## How protocols connect to the Landscape

A reader who's read Volume 6 chapters 1–3 already knows the
connection.  This chapter is a quick recap from the
interop-side perspective:

- A Landscape file declares devices and services with
  `deviceType` / `serviceType` fields.
- Each `deviceType` (e.g., `network/ssh`) maps to a
  *Coordinator* in `codecraftai-landscaping`.
- The Coordinator instantiates one of `codecraftai-interop`'s
  Agents (e.g., `SshAgent`) for each device.
- A Coordinator wraps the Agent in a `ProtocolExtension` and
  attaches it to the `LandscapeDevice`.
- Test code calls `device.ssh.run_cmd(...)` — under the hood,
  the call goes through the extension to the agent, which
  applies the Aspects-driven persistence behaviour from
  chapter 2.

So `codecraftai-interop` provides the *agents*; `codecraftai-landscaping`
provides the *coordinators and protocol extensions*.  The two
packages compose.  Volume 6 walks the landscape side; this
chapter walks the interop side.

## What this chapter set up

By the end of this chapter a reader should:

- Locate the three Agent base classes (`SystemAgentBase`,
  `SshBase`, `SerialAgentBase`, `PowerAgentBase`).
- Know that `LocalCommandAgent` is the simplest concrete
  agent and a good study target.
- Recognise the paramiko-wrapping SSH stack and distinguish
  it from the from-scratch SSH stack chapter 4 walks.
- Know what the shipped protocols are (local commands, SSH,
  serial-over-TCP, DNS / mDNS, power) and where each lives.
- Understand the current state of the `am-interop` CLI
  (scaffolded, subcommands pending).

The next chapter walks the from-scratch SSH stack at
`codecraftai-protocol-ssh` in depth — packets up to agents.  It
is the volume's most code-anchored chapter, and reading it
gives the reader a complete picture of what a substantial
Measured Interoperability protocol looks like all the way
down.
