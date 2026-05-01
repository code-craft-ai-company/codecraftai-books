# Chapter 11 — Credentials at Scale

> *Volume 3 walks credentials at the orchestration layer:
> resolution, rotation, scoping, environment spanning.  This
> chapter walks the same vocabulary at the job layer — when one
> test job needs credentials for twelve hosts and fifteen
> services, what does the framework do, and how does the
> activation chain make it look simple?*

Volume 3 Part III (chapters 9 through 12) is the canonical
treatment of credentials in the `automojo-*` portfolio.  It
walks the credential vocabularies (basic auth, SSH keypairs,
API tokens, OAuth, certificates, derived credentials), the
resolution model (vault, file, environment-as-fallback,
prompt, derived), rotation in flight, and credential portability
across local-dev / CI / staging / prod.

This chapter is the *job-layer* view of the same machinery.
A distributed test job's credentials story has three concerns
that the orchestration story doesn't fully address:

- How credentials get from the orchestration to the job at
  hand-off (chapter 7's verified contract sets it up; this
  chapter walks the consumption side).
- How a single job uses credentials for *many hosts* without
  the wiring becoming N×M complex.
- How credentials flow from the controller (where they're
  resolved) to the hosts under test (where they're used) —
  *or do they not flow at all?*

The answers are clean.  Credentials live on the controller;
the controller authenticates *to* the hosts; the hosts never
see the credential material.  The framework's machinery makes
this idiomatic.

## Credentials in the Landscape — declared, not resolved

A device declaration in the Landscape (Volume 1 chapter 9 walked
the shape) references credentials *by name*:

```yaml
- deviceType: network/ssh
  host: 192.168.1.30
  credentials:
      - casey-node      # ← named reference, not material
  features:
      isolation: true
```

The `casey-node` string is a *credential reference*.  The
actual material — the SSH keypair, the password, the API token
— lives in the Credentials configuration category (Volume 3
chapter 9).  At activation time, the framework's
`CredentialManager` resolves the reference to the material.

This separation is critical.  The Landscape file can be
checked into version control because it has no secret material;
only references.  The Credentials configuration is what
actually has secrets, and it's encrypted (Volume 3 chapter 11
walks Fernet encryption) or fetched from a vault.

## The CredentialManager — one resolver per Pod

`CredentialManager` (in `automojo-landscaping`) is the job's
credential lookup service.  Initialised by
`LandscapeConfigurationLayer.initialize_credentials()` during
`fire_activate_configuration()` (chapter 6 walked the flow),
it holds the resolved credentials for the duration of the job.

The lookup is by name:

```python
manager = LandscapeSingleton().credentials_manager
ssh_cred = manager.lookup_by_name("casey-node")
```

The returned `ssh_cred` carries the actual auth material:
private key, optional passphrase, username, etc.  The
framework's `SshAgent` for any device declaring `casey-node`
gets the credential from the manager and uses it to
authenticate.

Two properties of the manager:

1. **Single-pass resolution.**  The manager resolves all
   credentials at activation time.  A name that doesn't
   resolve fails activation (chapter 7's Landscape
   verification catches it).  Once activation succeeds, the
   manager's lookups are pure dict reads — no further
   resolution-side failures during the test.
2. **Per-Pod scope.**  Each Pod has its own manager.  The
   credentials for Pod A and Pod B are independent.  A
   credential rotation that affects Pod A doesn't touch Pod
   B's manager mid-run.

## How a credential reaches a host

A test issuing an SSH call against a device:

```python
device = LandscapeSingleton().device("casey-01")
status, stdout, stderr = device.ssh.run_cmd("uptime")
```

The flow underneath:

1. `device.ssh` is the device's SSH ProtocolExtension,
   attached at activation time (chapter 6).
2. The ProtocolExtension wraps an `SshAgent` (chapter 10).
3. The agent holds a reference to the device's resolved
   `SshCredential` (looked up by name from the
   `CredentialManager` at activation time).
4. The agent's `run_cmd` opens an `SshSession`; the session
   uses the credential to authenticate against the device's
   `host` (192.168.1.30 in our example).
5. The host validates the auth.  If valid, the session is
   established and the command runs.

The credential **never travels to the host as data**.  The
host receives an *authenticated SSH connection*; the auth
material stays on the controller.  This is the standard SSH
model — the client proves possession of the credential through
the auth handshake without sending the material.

The implication for the test author: *credentials are an input
to the controller's wire calls, not data the test ever sees*.
A test never has reason to pass a credential through its body;
the framework wires credentials to devices at activation time
and the test's calls inherit the auth.

## Composable credential lists in a device declaration

A device's `credentials:` is a list, not a single name:

```yaml
- deviceType: network/ssh
  host: 192.168.1.30
  credentials:
      - casey-node
      - emergency-recovery
      - ci-runner
```

The list is *composable* per the universal-composability
property (Volume 3 chapter 3, Vol 0 ch.9).  At activation, the
framework loads each named credential and the SshAgent has
*all of them* available.  Different operations can use
different credentials:

```python
# Default — uses primary_credential (first in the list).
device.ssh.run_cmd("uptime")

# Specific user — opens a session with a different credential.
admin_session = device.ssh.open_session(user="ci-runner")
admin_session.run_cmd("systemctl reload service")
```

The first name in the list is the *primary* credential.  The
others are alternates the test can switch to.  The list is
useful when:

- Different tasks need different privilege levels (regular
  user for most operations; root for some).
- A primary credential might rotate during a long run; the
  alternate is the fallback.
- Different test phases have different identity requirements
  (CI runner during setup; user-level during tests).

## Credentials per service vs per device

The Landscape's `services` section (Volume 1 chapter 9) carries
its own credentials:

```yaml
infrastructure:
    services:
        - serviceType: network/service
          name: someservice
          credentials:
              - adminuser
```

Service credentials work the same way as device credentials —
named, resolved at activation, never flowing to remote hosts.
The framework exposes service credentials through the service's
ProtocolExtension (e.g., `service.api.client`) just as device
credentials are exposed through `device.ssh`.

A test using both a device and a service:

```python
device = LandscapeSingleton().device("casey-01")
service = LandscapeSingleton().service("someservice")

# Device call uses device's primary credential (casey-node).
device.ssh.run_cmd("verify-state")

# Service call uses service's primary credential (adminuser).
service.api.post("/admin/refresh", data={...})
```

Two different credentials, two different remote endpoints,
zero credential material in the test code.

## Cross-host concurrency and credential safety

Chapter 10 walked intra-test concurrency — fanning a command
out to N hosts via a `ThreadPoolExecutor`.  The credential
implications:

- Each device's `SshAgent` holds *its own* credential.
  Concurrent calls from a fan-out to N devices use N
  independent credentials, with no cross-contamination.
- Credentials are *not* shared between threads in the
  thread pool.  Each thread's call goes through the device's
  agent, which uses the device's credential.  No mutex
  needed; no race.
- The `Aspects` object passed to each call is `Freezable`
  (chapter 10) — same instance shared across the fan-out
  cannot be mutated.

Credential safety in concurrent fan-out is therefore
*architectural*: the framework's design (per-device agents,
frozen aspects, no shared credential cache) makes the
parallelism safe by construction.  The test author writes
straightforward concurrent code; the framework holds the
invariants.

## Credential rotation during a long run

A four-hour integration suite outlives many credential
rotation windows.  Volume 3 chapter 11 walks rotation as a
configuration discipline; the job-side question is: *what
happens when a credential rotates mid-run?*

Three patterns the framework supports:

### TTL caching with rotation

The `CredentialManager` (per its Volume 3 ch.11 contract) can
cache resolved credentials with a TTL.  When the TTL expires
mid-run, the next lookup re-resolves through the underlying
backend (vault, file, etc.).  If the backend rotated, the new
material is fetched.

For SSH agents, the rotation is *transparent to the test*:
the next session opened against the device picks up the
freshly-resolved credential.  Sessions already open continue
with the old credential until they close.

### Composable list with rotation fallback

When a primary credential rotates and the SSH session fails
auth on next use, the agent can fall back to the *alternate*
credentials in the device's list.  The test sees the fall-back
transparently; the JSOS record can flag that a fallback
occurred for post-run review.

### Explicit rotation events

For tests that *exercise* rotation (testing the rotation
itself), the test is the actor that triggers the rotation.
The test's pattern:

```python
def test_rotation_does_not_disrupt_active_sessions():
    """
        Verify that rotating credentials does not disrupt active
        sessions.
    """
    device = LandscapeSingleton().device("casey-01")
    long_running_session = device.ssh.open_session()

    # Trigger rotation through the credential vault's API.
    rotate_credential("casey-node")

    # Verify the long-running session still works.
    status, _, _ = long_running_session.run_cmd("uptime")
    rtnval = (status == 0)
    assert rtnval

    # Open a new session — should use the rotated credential.
    new_session = device.ssh.open_session()
    status, _, _ = new_session.run_cmd("uptime")
    rtnval = (status == 0)
    assert rtnval
    return
```

The test, as the single actor, drives the rotation; the
framework's credential machinery handles the consequences
(active sessions hold old creds; new sessions get fresh ones).

## What the controller owns vs what the host owns

The clear ownership separation that the framework imposes:

| Concern | Controller (where the test runs) | Host (system under test) |
|---|---|---|
| Credential material | Has it (resolved by `CredentialManager`) | Never sees it |
| Auth handshake | Initiates | Validates |
| Session establishment | Owns the session object | Owns the SSH daemon |
| Command execution | Sends commands | Executes commands, returns status / stdout / stderr |
| Credential rotation | Re-resolves through the backend | Has new auth-validating material delivered out-of-band |

The line is sharp.  The host never has data the controller
gave it from the credential.  The controller never has data
the host gave it about its own auth state.  The two sides
trust the SSH handshake to validate identity without exchanging
secrets in either direction.

This is the foundation of the determinism model from chapter
2.  If credentials flowed bidirectionally, the test could
become non-deterministic depending on what the host sent
back about its auth state.  The unidirectional ownership keeps
the test's behaviour entirely a function of the controller's
state.

## What this chapter set up

The credentials story at the job layer is short because the
framework's design does most of the work:

- Credentials are declared by name in the Landscape.
- The `CredentialManager` resolves them at activation.
- ProtocolExtensions hold credential references; agents use
  them.
- Credential material never flows to remote hosts.
- Concurrent fan-out is safe by construction.
- Rotation is handled at the manager level, transparent to
  most tests; tests that exercise rotation drive it as the
  actor.

A reader who understands chapter 6 (activation), chapter 10
(the wire), and Volume 3's credential vocabulary has the full
picture.  This chapter assembled the parts.

## What the next chapter does

Chapter 12 walks data profiles in remote workers — the
analogous story for connection profiles to data stores.  Where
credentials are about *authentication*, data profiles are
about *connection topology* (which database, which queue,
which cache).  The framework's discipline is the same: declare
by name in the Landscape, resolve at activation, expose
through ProtocolExtensions.  The chapter is short because the
discipline is uniform.
