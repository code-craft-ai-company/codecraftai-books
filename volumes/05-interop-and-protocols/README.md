# Volume 5 — Interop and Protocols

*The Aspects Pattern, the Wire, and the Extension Model*

## About this volume

Volume 5 is **Measured Interoperability** — the framework's
discipline for talking to external systems through wire
protocols.  Where Volume 6 covers the *environment model*
(what resources exist, how they're organised), Volume 5 covers
the *protocols* the framework uses to reach into the
environment.

The volume has two sides:

- **The Behaviour side** — Measured Interoperability's
  central thesis.  The Aspects pattern as the delegation
  contract.  The four principles that govern persistence
  behaviours.  The typed exception model that reverts control
  to the application layer.
- **The Mechanism side** — concrete protocol implementations.
  SSH from packet up to agent.  REST foundations.  The
  auto-generated GitHub client.  The four extension packages
  (UPnP, Tasker, Bluetooth, CAN).  The pattern for adding
  your own.

The two sides correspond to the userguide's Mechanism vs.
Behaviour separation — *the* defining commitment of the
package.

## Who it is for

Integration engineers writing tests against external systems.
Protocol implementors adding new protocols to the framework.
Anyone whose project's automation reaches across machine
boundaries through any of: SSH, REST, serial, UPnP, Tasker,
Bluetooth, CAN, or a custom protocol.

## Source material

- `codecraftai-interop` — the foundation: Aspects, ActionPattern,
  LoggingPattern, the agent base, the local command agent,
  shipped protocols (DNS / mDNS, serial, power), `am-interop`
  CLI.
- `codecraftai-protocol-ssh` — SSH from packet to agent.
- `codecraftai-networking` — networking primitives.
- `codecraftai-restclient-base` — REST foundation.
- `codecraftai-restclient-github-public` — auto-generated worked
  example.
- `codecraftai-interop-ext-upnp`, `-tasker`, `-bluetooth`, `-can`
  — protocol extension packages.

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — Philosophy and Foundation | 1, 2, 3 | Measured Interoperability; the Aspects pattern; `codecraftai-interop` survey |
| II — Worked Protocols | 4, 5, 6, 7 | SSH from scratch; networking primitives; REST foundations; the auto-generated GitHub client |
| III — Extensions | 8, 9 | UPnP / Tasker / Bluetooth / CAN; recipe for your own |

- [Chapter 1 — Measured Interoperability — The Philosophy](chapters/01-measured-interoperability.md)
- [Chapter 2 — The Aspects Pattern — The Delegation Contract](chapters/02-the-aspects-pattern.md)
- [Chapter 3 — `codecraftai-interop` — Agents, Protocols, CLI](chapters/03-codecraftai-interop.md)
- [Chapter 4 — SSH from Scratch](chapters/04-ssh-from-scratch.md)
- [Chapter 5 — Networking Toolkit Primitives](chapters/05-networking-toolkit.md)
- [Chapter 6 — REST Client Foundations](chapters/06-rest-client-foundations.md)
- [Chapter 7 — The Auto-Generated GitHub Public-API Client](chapters/07-github-auto-generated-client.md)
- [Chapter 8 — Extension Packages — UPnP, Tasker, Bluetooth, CAN](chapters/08-extension-packages.md)
- [Chapter 9 — Building Your Own Interop Extension](chapters/09-building-your-own.md)
- [Chapter 10 — Remote Objects over a Wire: `codecraftai-python-rpc`](chapters/10-remote-objects-over-a-wire.md)

## How it connects to the other volumes

- **Volume 0** — the foundation tier (errors, waiting,
  application identity, `wait_for_it` primitive at ch.5,
  Aspects pattern introduced philosophically at ch.4).
- **Volume 1 ch.5** — the JSOS streaming format that
  recordings use.
- **Volume 2 ch.5** — tandem validators apply the same
  four-principle discipline at the framework layer.
- **Volume 2 ch.10** — sketches the SSH layer; Volume 5 ch.4
  is the depth-walk.
- **Volume 6 ch.3** — the Coordinator/Agent/ProtocolExtension/
  Feature vocabulary.  Volume 5 zooms into Agents.
- **Volume 6 ch.13** — GitLab integration (REST consumer
  side); Volume 5 ch.6/7 is the foundation Volume 6 ch.13
  builds on.

---

Part of [The Code Craft AI Books](../../README.md).
