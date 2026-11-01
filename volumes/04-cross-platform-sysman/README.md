# Volume 4 — Cross-Platform System Management

*52 Cross-OS Contracts and 5 Reference Implementations*

## About this volume

Volume 4 is the **"do it this way"** book of the series.  The
sysman tier is the most disciplined product line in the
portfolio: 52 cross-OS contracts in
`codecraftai-sysman-common/specs/contracts/` plus five reference
implementations (RHEL, Ubuntu, macOS, Windows, UEFI), each
tested green to thousands of testbase tests.

Where Volume 5 covered *how to talk to remote systems*
(protocols, the wire), Volume 4 covers *what to ask remote
systems to do* — install a package, start a service,
configure the firewall, manage boot order.  The tier sits
*above* the wire and *above* the landscape, in the
"actuator" position: it's the layer test code reaches for
when it needs to *change the system under test*.

## Who it is for

Platform / SRE engineers who need cross-OS automation that
actually works.  System-management-tier maintainers who want
to extend the contract or add a new OS implementation.
Anyone whose project's automation must reach across RHEL,
Ubuntu, macOS, Windows, or UEFI consistently.

## Source material

- `codecraftai-sysman-common` — the 52 contracts, the
  `featurebase` substrate, and the two feature bases (the
  abstract VPN base and the concrete python-environment base).
- `codecraftai-sysman-linux-redhat` — RHEL implementation.
- `codecraftai-sysman-linux-ubuntu` — Ubuntu / Debian
  implementation.
- `codecraftai-sysman-macos` — macOS implementation.
- `codecraftai-sysman-windows` — Windows implementation.
- `codecraftai-sysman-uefi` — UEFI firmware implementation.

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — The Contract Layer | 1, 2, 3 | Why contracts; the 52 contracts; `featurebase` substrate |
| II — Five Reference Implementations | 4, 5, 6, 7, 8 | RHEL, Ubuntu, macOS, Windows, UEFI |
| III — The Discipline | 9, 10 | 11 Implementation Conventions; cross-OS test parity |

- [Chapter 1 — Why a Contract Layer](chapters/01-why-a-contract-layer.md)
- [Chapter 2 — The 52 Cross-OS Contracts](chapters/02-the-50-contracts.md)
- [Chapter 3 — `featurebase` and the Per-OS Implementation Pattern](chapters/03-featurebase.md)
- [Chapter 4 — RHEL Implementation](chapters/04-rhel-implementation.md)
- [Chapter 5 — Ubuntu / Debian Implementation](chapters/05-ubuntu-implementation.md)
- [Chapter 6 — macOS Implementation](chapters/06-macos-implementation.md)
- [Chapter 7 — Windows Implementation](chapters/07-windows-implementation.md)
- [Chapter 8 — UEFI Firmware Implementation](chapters/08-uefi-implementation.md)
- [Chapter 9 — Spec-Driven Development for System Administration](chapters/09-spec-driven-development.md)
- [Chapter 10 — Building Cross-OS Test Parity](chapters/10-cross-os-test-parity.md)

## How it connects to the other volumes

- **Volume 0** — error vocabulary, configuration loading,
  application identity (the foundation tier).
- **Volume 1 / 2** — testbase / tandem testing as the
  framework that runs sysman feature tests.
- **Volume 3** — orchestration that invokes sysman feature
  execution; ch.25's bridge tasks.
- **Volume 5** — protocols (SSH, REST) that sysman features
  use to reach remote systems.
- **Volume 6** — landscape (which devices to reach) and
  topology (the configured state).

---

Part of [The Code Craft AI Books](../../README.md).
