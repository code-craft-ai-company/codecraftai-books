# Chapter 10 — The Remote-Execution Wire

> *Distributed automation lives or dies on the wire.  The
> controller's actions reach the hosts through SSH, REST, gRPC
> or a stranger protocol; whatever the wire, the framework's
> requirements are the same: every call has a bounded timeout,
> a parameterised retry policy, an explicit failure mode, and
> a result type that fits in the JSOS stream.*

The previous chapters walked the framework's *control plane* —
the lifecycle, the activation layers, the verification
contract, the sequencer.  This chapter walks the *data plane*
— the actual wire that carries commands from the controller
to the hosts and results back.

This volume's coverage of the wire is deliberately
introductory.  Volume 5 (*Interop and Protocols*) walks the
wire's machinery in depth: the SSH transport, the REST client
foundations, the protocol extensions, the per-protocol agents.
This chapter establishes *how the wire fits into a distributed
test job* — what API a test calls, what the framework
guarantees about the call, what failures look like, how
results land in the JSOS stream.

The forward references to Volume 5 are deliberate; the back
references to Volumes 0 and 1 are the foundation.

## The Aspects pattern — behaviour parameters on every call

Volume 0 chapter 4 introduced the Aspects pattern at the
philosophy level: behaviour knobs (timeout, retry, sync flag,
logging policy) live as *parameters*, not as decorators or
implicit defaults.  Volume 5 chapter 1 walks the pattern in
depth.

For distributed wire calls, every call accepts an `Aspects`
parameter (or the protocol-specific subclass).  In SSH:

```python
# automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshaspects.py:77

class Aspects(Freezable):
    """
        Aspects are utilized with the interop APIs and agents such as the :class:`SSHAgent`
        to modify the behavior of APIs with respect to retry parameter such as timeout, interval,
        looping patterns
    """
```

The class is declared `Freezable` — once an `Aspects` instance
is bound to a call, it cannot be mutated.  This prevents the
classic bug where a shared `Aspects` instance gets modified
mid-call by another thread.

The default values from the same file:

```python
DEFAULT_COMPLETION_TIMEOUT = 600
DEFAULT_COMPLETION_INTERVAL = 10
DEFAULT_CONNECTION_TIMEOUT = 16
DEFAULT_INACTIVITY_TIMEOUT = 600
DEFAULT_INACTIVITY_INTERVAL = .5
DEFAULT_ALLOWED_ERROR_CODES: Tuple[int] = []
```

Six constants.  Each one is a default a project will want to
override for some calls and accept for others.  An `Aspects`
instance can change any of them while leaving the rest at
default:

```python
# A custom Aspects for a long-running call.

slow_aspects = AspectsCmd(
    completion_timeout=3600,    # one hour
    inactivity_timeout=300,     # five minutes
    # connection_timeout, completion_interval, etc. → default
)

ssh_session.run_cmd("apt-get dist-upgrade", aspects=slow_aspects)
```

The Aspects parameter on `run_cmd` is what makes the call
*tunable per invocation* without proliferating method
signatures.  Every distributed wire call works the same way.

## The SSH stack — what a test calls

The SSH machinery in `automojo-protocol-ssh` exposes three
levels:

### `SshSession` — context manager around an SSH client

**File:** `automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshsession.py:45`

```python
class SshSession(SshBase):
    def __init__(self,
                 host: str,
                 primary_credential: SshCredential,
                 users: Optional[Dict[str, SshCredential]] = None,
                 port: int=22,
                 jump: Union[str, SshJumpParams, None] = None,
                 pty_params: Optional[dict] = None,
                 session_user=None,
                 interactive=False,
                 basis_session: Optional["SshSession"]=None,
                 called_id: Optional[str]=None,
                 look_for_keys: bool = False,
                 primitive: Optional[bool] = None,
                 aspects: AspectsCmd=DEFAULT_CMD_ASPECTS):
```

A session represents one open SSH connection.  Used as a
context manager:

```python
with SshSession(host="10.0.0.1", primary_credential=cred) as session:
    status, stdout, stderr = session.run_cmd("uptime")
    session.file_push(local_path, remote_path)
```

`run_cmd` returns `(int, str, str)` — status code, stdout,
stderr.  The integer is *the exit status*; the strings are
captured output.  Failed commands return a non-zero status —
they do *not* raise, except for transport-level failures
(timeouts, disconnects).

The choice to return status rather than raise is deliberate.
A test that expects a non-zero status (e.g., "this command
should fail") can check `status != 0` directly.  Raising would
force every caller to wrap in try/except even for expected
failures.

### `SshAgent` — vending sessions for a host

**File:** `automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshagent.py:38`

```python
class SshAgent(SshBase):
    """
        The :class:`SshAgent` provides an extension of the :class:`SshBase` class interface and api(s).
        The :class:`SshAgent` manages connection and interop settings that are used to perform operations
        and interact with a remote SSH service...
    """

    def open_session(self, user: str = None, primitive: bool = False, pty_params: Optional[dict] = None,
                     interactive=False, basis_session: Optional[SshSession] = None,
                     aspects: Optional[AspectsCmd] = None) -> SshSession:
        """
            Provides a mechanism to create a :class:`SshSession` object with derived settings...
        """
```

An `SshAgent` is bound to a host and credentials.  It vends
`SshSession` instances tailored to specific tasks or users.
The agent's `open_session()` lets a caller override user,
primitive mode, PTY params, and aspects per session.

In the framework's distributed model, every device with a
`network/ssh` deviceType has an attached `SshAgent` (chapter
6 walked the activation flow that attaches it).  A test
working with the device calls into the device's
ProtocolExtension (`device.ssh`) which delegates to the
`SshAgent`.

### `SshChannel` — raw channel ops

**File:** `automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshconnection.py:14`

For tests that need direct channel-level control (request a
PTY, push raw bytes, drive an interactive shell), the
`SshChannel` class is the API.  Most tests don't need this —
`run_cmd` and `file_push` / `file_pull` cover the common cases.

Tests reaching for `SshChannel` are typically driving an
interactive remote shell, debugging a connection, or
implementing a custom protocol over SSH.  The lower the
level, the more responsibility on the test author for
correctness; the framework's higher-level APIs take care of
the common-case correctness.

## The four error classes a wire call can raise

Distributed wire calls have four classes of failure, each
handled differently:

### 1. Authentication failures — `SshAuthError`

```python
# automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshauth.py:11

class SshAuthError(Exception):
    pass
```

Raised when SSH authentication fails.  Verification (chapter
7) catches this at activation time; if it surfaces during a
test, the credentials rotated mid-run or the host's auth
config changed.

### 2. SFTP failures — `SshSFTPError`

```python
# automojo-protocol-ssh/source/packages/automojo/protocols/ssh/sshsftp.py:26

class SshSFTPError(Exception):
    pass
```

Raised on SFTP protocol violations.  Less common than
`SshAuthError`; usually surfaces when a file operation hits a
permissions problem or a remote-path issue the framework
can't recover from.

### 3. Timeouts — `TimeoutError`

Raised when a `completion_timeout`, `connection_timeout`, or
`inactivity_timeout` from the call's Aspects elapses.  The
test's response: typically retry (with new Aspects), fail the
test (the wire is unreliable enough to fail), or escalate to
the orchestrator (this Pod is unhealthy).

### 4. Connection failures — `ConnectionError`

Raised when the underlying SSH transport drops.  Different
from a timeout (the connection is *gone*, not just slow).
The framework's response is similar: retry, fail, or escalate
based on the `Aspects` policy.

The four error classes give the test author *typed dispatch*
on failure: a `try / except SshAuthError / except TimeoutError`
block can take different actions based on the failure class.
Volume 0 chapter 3's exception-handling discipline applies —
catch only what the code can recover from; let everything else
bubble.

## How wire results land in the JSOS stream

Volume 1 chapter 5 walked the JSOS streaming format.  Wire
calls don't produce *separate* JSOS records — they're
ordinary Python operations from the framework's perspective.
A test that calls `device.ssh.run_cmd(...)` produces:

- A *test_started* record when the test begins.
- (No record for the wire call itself; the wire is internal
  to the test body.)
- A *test_finished* record when the test ends, with the
  outcome derived from the test's assertions.

The wire calls' *results* (status codes, stdout, stderr) are
visible in the test's logs and in any diagnostic captures,
but they don't get their own JSOS records by default.  This
is a deliberate choice — wire calls are part of the test's
plumbing, not part of the test's shape.

A project that wants per-wire-call records can attach a
*tandem validator* (chapter 5) that intercepts wire calls
through the device's ProtocolExtension and records each one.
The validator's record-per-call shape lands in the JSOS
stream alongside the test's regular records.

## Cross-host concurrency inside a test

Chapter 8 noted that the distributed sequencer doesn't
parallelise tests, but the *test body* may legitimately use
concurrency to drive multiple hosts at once.  Here is what
that looks like in practice:

```python
import concurrent.futures

def test_apply_config_to_cluster():
    """
        Verify that applying a configuration to all cluster nodes
        succeeds on all nodes.
    """
    nodes = LandscapeSingleton().devices_by_role("cluster_node")
    config = build_test_config()

    with concurrent.futures.ThreadPoolExecutor(max_workers=len(nodes)) as executor:
        futures = {
            executor.submit(node.ssh.run_cmd,
                            f"apply-config {config.path}",
                            aspects=DEFAULT_CMD_ASPECTS): node
            for node in nodes
        }
        results = {
            futures[fut]: fut.result()
            for fut in concurrent.futures.as_completed(futures)
        }

    rtnval = all(status == 0 for status, _, _ in results.values())
    assert rtnval, "config application failed on at least one node"
    return
```

The test:

- Fans out an SSH command to N nodes concurrently.
- Waits for all to complete.
- Asserts success across the fan-out.

This is *intra-test* concurrency, driven by the test (the
single actor) distributing its own work.  Chapter 2's
single-actor model is preserved: the test, as the actor, is
issuing commands; the nodes are responders, not actors.  The
fan-out finishes before the next test starts.

A note on the Aspects: each fan-out call uses the *same*
`Aspects` instance.  This is safe because Aspects are
`Freezable` — none of the concurrent calls can mutate the
shared instance.  A project wanting per-call Aspects creates
a fresh instance per call.

## A worked example — orchestration through a single agent

A test that needs to orchestrate work across the Pod often
goes through a single agent rather than fanning out to many.
The classic pattern is the *task agent*:

```python
def test_rolling_restart_via_taskerserver():
    """
        Verify rolling restart through the taskerserver agent.
    """
    tasker_node = LandscapeSingleton().device("taskerserver-1")
    target_nodes = LandscapeSingleton().devices_by_role("worker")

    for target in target_nodes:
        # Issue the restart command through the tasker; the tasker
        # restarts the target and waits for health.
        status, stdout, stderr = tasker_node.tasker.restart_and_wait(
            target=target.name,
            timeout=120,
        )
        assert status == 0, f"restart of {target.name} failed: {stderr}"

    rtnval = all_nodes_healthy(target_nodes)
    assert rtnval
    return
```

The test issues commands through one tasker rather than
fanning out.  The tasker handles the per-node sequencing
(restart node A, wait for health, then node B, etc.).  This
is a common pattern when the *system under test* has a
control surface (the tasker) that the test should exercise.

In this pattern, the `SshAgent` for the tasker is what gets
used; the worker nodes' SSH access is incidental (used by
verification at activation time, not used by the test).

## Where Volume 5 picks up

This chapter establishes the wire's role in distributed
testing.  Volume 5 walks:

- The full SSH protocol stack (auth, transport, channels,
  SFTP, tunnels).
- The REST client foundations
  (`automojo-restclient-base`).
- The serial / power / mDNS / UPnP protocols and their
  agents.
- The Aspects pattern in depth, with the patterns for
  composing aspects across protocols.
- The extension model for adding new protocols.

A reader of Volume 2 should know *how the wire surfaces in a
test job* — what API to call, what to expect from a call,
what failure modes to handle.  Volume 5 is for the reader who
needs to *implement* a new protocol or debug the wire itself.

## What the next chapter does

Chapter 11 walks credentials at scale: how the Volume 3
credentials story (resolution, rotation, scoping) plays out
when a single test job needs credentials for twelve hosts and
the resolver lives in one process while the consumers are
spread across many.
