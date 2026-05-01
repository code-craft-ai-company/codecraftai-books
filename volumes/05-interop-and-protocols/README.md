# Volume 5 — Interop and Protocols

**Working subtitle:** *The Aspects Pattern, the Wire, and the Extension Model*

This is the working directory for Volume 5.  The series-level
entry for this volume lives in
[`automojo-volumes.md`](../../automojo-volumes.md); this
directory is where the volume gets drafted.

## Volume role in the series

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

## Audience

Integration engineers writing tests against external systems.
Protocol implementors adding new protocols to the framework.
Anyone whose project's automation reaches across machine
boundaries through any of: SSH, REST, serial, UPnP, Tasker,
Bluetooth, CAN, or a custom protocol.

## Source material

- `automojo-interop` — the foundation: Aspects, ActionPattern,
  LoggingPattern, the agent base, the local command agent,
  shipped protocols (DNS / mDNS, serial, power), `am-interop`
  CLI.
- `automojo-protocol-ssh` — SSH from packet to agent.
- `automojo-networking` — networking primitives.
- `automojo-restclient-base` — REST foundation.
- `automojo-restclient-github-public` — auto-generated worked
  example.
- `automojo-interop-ext-upnp`, `-tasker`, `-bluetooth`, `-can`
  — protocol extension packages.

## Layout

```
05-interop-and-protocols/
├── README.md            # this file
├── chapters/            # per-chapter draft files
├── research/            # findings on open questions; code-anchor index
├── case-studies/        # full worked-example transcripts
├── exercises/           # reader-exercise material per chapter
└── figures/             # diagrams referenced from chapters
```

## Status

| Chapter | Status | Lines |
|---|---|---|
| 1 — Measured Interoperability — the philosophy | drafting | — |
| 2 — The Aspects pattern — the delegation contract | pending | — |
| 3 — `automojo-interop` — agents, protocols, CLI | pending | — |
| 4 — SSH from scratch | pending | — |
| 5 — Networking toolkit primitives | pending | — |
| 6 — REST client foundations | pending | — |
| 7 — The auto-generated GitHub public-API client | pending | — |
| 8 — Extension packages — UPnP, Tasker, Bluetooth, CAN | pending | — |
| 9 — Building your own interop extension | pending | — |

## Drafting conventions

Same as Volumes 0–4 and 6:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference
  into one of the `automojo-*` source repositories.
- Use the formal name **Measured Interoperability** throughout
  — never the shortened "Measured Interop."
- Use **persistence behaviours** as the umbrella term for
  retry / poll / wait / reconnect patterns; the framework's
  `ActionPattern` enum names the canonical behaviours.
- Chapter files start with a one-paragraph chapter abstract,
  then section structure (`##` for sections, `###` for
  subsections).
- Forward / backward cross-references link to the relevant
  chapter file with the `chapters/NN-name.md#section-anchor`
  form.

## Cross-volume reference points

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
