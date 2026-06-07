# Chapter 5 — Networking Toolkit Primitives

> *Below the protocol layer is the networking layer — the
> primitives every protocol implementation needs but no single
> protocol owns.  Address resolution, interface enumeration,
> multicast / unicast / broadcast socket creation.
> `codecraftai-networking` ships these as a foundation utility.*

`codecraftai-networking` is the *only* package in the volume
that's not itself a protocol implementation.  It's a
foundation utility — a small library of stateless helpers
and well-shaped socket factories that the protocol packages
above it depend on.

The chapter is short.  The package is small (one flat
directory of modules, no class hierarchies, no agents).  But
its contents are load-bearing: the protocol stacks, the
extension packages, and even the landscape's reachability
checks all depend on these primitives.

## What's in the package

```
codecraftai-networking/source/packages/ccai/networking/
├── interfaces.py             # interface enumeration
├── resolution.py             # DNS / IPv4 / IPv6 / ARP
├── multicast.py              # multicast socket helpers
├── unicast.py                # unicast socket helpers
├── broadcast.py              # broadcast socket helpers
├── arp.py                    # ARP protocol helpers
├── trafficcapturecontext.py  # traffic capture context manager
├── constants.py              # IP/regex constants
└── __init__.py
```

A flat module layout.  No subdirectories.  No agents.  Each
module is a small set of helper functions plus a few enums.

## Interface enumeration

**File:** `interfaces.py:24–82`

The first concern: what network interfaces does this host
have?

```python
class InterfaceClass(IntEnum):
    """
        Classification of a network interface.
    """
    LOOPBACK = 0
    BRIDGE = 1
    WIRED = 2
    WIRELESS = 3
    TUNNEL = 4
```

Five classes covering most lab scenarios.  Helpers:

- `get_ipv4_address(ifname)` — return IPv4 for an interface
  by name, using `netifaces`.
- `get_all_ipv4_interfaces()` / `get_all_ipv6_interfaces()`
  — enumerate all interfaces with addresses of each family.
- `encode_address(address)` — pack IPv4 or IPv6 to
  on-wire bytes via `socket.inet_pton`.

A test that needs to bind a socket to a specific interface
walks `get_all_ipv4_interfaces()`, picks the right one based
on `InterfaceClass`, and uses `get_ipv4_address` to get the
bind address.

The `InterfaceClass` enum is a small but useful abstraction:
classifying interfaces by *class* (loopback / wired /
wireless / tunnel / bridge) makes test logic readable.  Code
asking "is this device on a tunnel interface?" is clearer
than code parsing interface name patterns.

## DNS / IPv4 / IPv6 / ARP

**File:** `resolution.py:23–192`

The largest module in the package.  Three concerns:

### Address validation

```python
def is_ipv4_address(candidate: str) -> bool:
    """
        Return True if the candidate string is a syntactically
        valid IPv4 address.

        Uses regex matching plus octet-bounds checking; does NOT
        perform DNS resolution.
    """
    # ... regex match plus per-octet bounds 0-255 ...

def is_ipv6_address(candidate: str) -> bool:
    """
        Same for IPv6, including '::' wildcard expansion.
    """
    # ...
```

These helpers are *syntactic* — they don't resolve names,
don't query DNS, just check whether a string parses as an
address.  Useful at configuration-load time when you want to
know whether a `host:` field in a Landscape file is an
address or a hostname.

### IPv6 expansion

```python
def expand_ipv6_addr(addr: str) -> str:
    """
        Expand the IPv6 '::' shorthand to a full address.

        e.g. '::1' -> '0000:0000:0000:0000:0000:0000:0000:0001'
    """
    # Handles the '::' wildcard expansion per RFC 4291.
```

A small but tedious helper.  Comparing IPv6 addresses for
equality without expansion is unreliable (the same address
has multiple shorthand forms); expansion normalises.

### ARP table

```python
def get_arp_table(normalize_hwaddr: bool = True) -> List[ArpEntry]:
    """
        Parse /proc/net/arp on Linux into ArpEntry tuples.
    """
    # ...

def refresh_arp_table() -> None:
    """
        Ping the local subnet to populate the ARP cache.
    """
    # Uses ARPING_RANGE_CMD; ranges from constants.py
```

ARP table access is Linux-specific (parses `/proc/net/arp`).
Useful for tests that need to know which devices are on the
local network — for example, a test that wakes a device via
Wake-on-LAN and then waits for it to appear in the ARP table.

A note on portability: the package targets Linux primarily;
macOS users get partial functionality from `netifaces` but
ARP table access and some interface classification work less
well.  The chapter doesn't pretend otherwise.

## Multicast / unicast / broadcast sockets

Three modules with parallel APIs.

### `multicast.py:25–199`

```python
def create_multicast_listener_socket(
    family: int,
    multicast_addr: str,
    port: int,
    ttl: int = 1,
    loopback: bool = False,
    interfaces: Optional[List[str]] = None,
) -> socket.socket:
    """
        Create a UDP socket joined to the given multicast address.

        Handles per-platform IGMP/MLD membership requests, TTL,
        loopback flag, and per-interface joining.
    """
    # ...
```

Multicast listening is fiddly across platforms.  Linux
requires `IP_ADD_MEMBERSHIP`; some OSes need `IPV6_JOIN_GROUP`
for IPv6; loopback flags differ.  The factory handles the
per-platform variations and returns a socket ready to
`recvfrom`.

A worked use: mDNS-based service discovery (Volume 6 chapter
1's services section is sometimes populated by mDNS at
startup) opens a multicast listener on `224.0.0.251` port
5353, listens for service announcements, populates the
landscape.

### `unicast.py` and `broadcast.py`

Same shape, different transport.  `unicast` for
point-to-point UDP; `broadcast` for subnet broadcast.

Most automation tests prefer TCP (reliable, ordered) but
discovery and some control protocols use UDP.  The networking
package's UDP helpers are what they reach for.

## Traffic capture

**File:** `trafficcapturecontext.py`

```python
class TrafficCaptureContext:
    """
        Context manager that starts a packet capture on entry
        and stops it on exit.

        Wraps tcpdump (Linux) or tshark, dumps to a file the
        caller specifies.
    """

    def __enter__(self):
        # Spawn capture process.
        # Wait for it to be ready.

    def __exit__(self, *args):
        # Signal stop; wait for clean shutdown.
        # Capture file is now usable.
```

The use case: a test that asserts on wire-level behaviour
opens a capture context, does its work, exits the context,
then parses the capture file.  Useful for tests that verify
"this protocol exchange happened correctly" rather than just
"the operation succeeded."

The capture is *external* (subprocess `tcpdump`); the package
just wraps the lifecycle as a context manager.

## How `codecraftai-networking` integrates with the rest of the stack

A point worth being explicit about: `codecraftai-networking` is a
**foundation utility, not an interop extension**.

The package's README states:

> *Provides the reusable primitives that higher-level protocol,
> discovery, and landscaping packages build on top of, without
> pulling in any heavy third-party networking stacks.*

What this means concretely:

- `codecraftai-networking` has zero hard dependency on
  `codecraftai-interop`.
- Higher-level packages (the protocol stacks, the extensions)
  depend on `codecraftai-networking`, not the reverse.
- The package doesn't ship Aspects-driven agents.  Its
  helpers are *synchronous and stateless* — single-call
  functions that return immediately.

This is correct for primitives.  The Aspects discipline from
chapters 1–2 applies to *protocol calls* — the agents that
talk to remote systems.  Networking primitives are below that
layer; they don't need persistence behaviours because they
*are* the things the persistence behaviours retry.

## When to use these primitives directly

A reader who's building a new protocol package or extension
uses these primitives at the lowest layer.  Specifically:

- **Service discovery** — open a multicast listener with
  `multicast.create_multicast_listener_socket`; parse
  announcements.
- **Address validation** — at configuration-load time, use
  `resolution.is_ipv4_address` / `is_ipv6_address` to
  validate `host:` fields.
- **Interface awareness** — when binding to a specific
  interface, use `interfaces.get_all_ipv4_interfaces` and
  `InterfaceClass` to pick the right one.
- **Wire-level test assertions** — use
  `TrafficCaptureContext` to capture and verify protocol
  exchanges.

A reader writing typical interop test code (calling
`device.ssh.run_cmd`, checking responses, etc.) probably
never reaches into `codecraftai-networking` directly — the
protocol packages above it do that work.

## What this chapter set up

A short chapter for a small, important package.  By the end
a reader should:

- Know the package is a *foundation utility*, not a protocol
  implementation.
- Locate the four kinds of primitives: interface
  enumeration, address resolution, socket factories, traffic
  capture.
- Understand why the package has no Aspects-driven agents
  (its helpers are below the layer where Aspects apply).

The next chapter walks `codecraftai-restclient-base` — the REST
foundation.  Where SSH (chapter 4) was a stream-oriented
protocol, REST is request/response.  The chapter walks the
`RestClientBase` class, the `AspectsRest` subclass, the
`RestApiGroup` model, and the `ItemsResult` / `Paging` lazy
container.  Chapter 7 then walks the auto-generated GitHub
client built on top.
