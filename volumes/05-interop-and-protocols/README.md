# Volume 5 — Interop and Protocols

**Working subtitle:** *The Aspects Pattern, the Wire, and the Extension Model*

This is the working directory for Volume 5.  The series-level
entry for this volume lives in
[`codecraftai-volumes.md`](../../codecraftai-volumes.md); this
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
| 1 — Measured Interoperability — the philosophy | drafted | 403 |
| 2 — The Aspects pattern — the delegation contract | drafted | 521 |
| 3 — `codecraftai-interop` — agents, protocols, CLI | drafted | 385 |
| 4 — SSH from scratch | drafted | 836 |
| 5 — Networking toolkit primitives | drafted | 298 |
| 6 — REST client foundations | drafted | 563 |
| 7 — The auto-generated GitHub public-API client | drafted | 416 |
| 8 — Extension packages — UPnP, Tasker, Bluetooth, CAN | drafted | 553 |
| 9 — Building your own interop extension | drafted | 740 |
| 10 — Remote objects over a wire — `codecraftai-python-rpc` | drafted | 548 |

**Volume 5 fully drafted: 10 chapters, 5,263 lines, ~23,000 words.**

## Drafting conventions

Same as Volumes 0–4 and 6:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference
  into one of the `codecraftai-*` source repositories.
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
