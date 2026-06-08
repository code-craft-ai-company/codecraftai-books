# Volume 4 — Cross-Platform System Management

**Working subtitle:** *50 Cross-OS Contracts and 5 Reference Implementations*

This is the working directory for Volume 4.  The series-level
entry for this volume lives in
[`codecraftai-volumes.md`](../../codecraftai-volumes.md); this
directory is where the volume gets drafted.

## Volume role in the series

Volume 4 is the **"do it this way"** book of the series.  The
sysman tier is the most disciplined product line in the
portfolio: 50 cross-OS contracts in
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

## Audience

Platform / SRE engineers who need cross-OS automation that
actually works.  System-management-tier maintainers who want
to extend the contract or add a new OS implementation.
Anyone whose project's automation must reach across RHEL,
Ubuntu, macOS, Windows, or UEFI consistently.

## Source material

- `codecraftai-sysman-common` — the 50 contracts and the
  `featurebase` substrate.
- `codecraftai-sysman-linux-redhat` — RHEL implementation.
- `codecraftai-sysman-linux-ubuntu` — Ubuntu / Debian
  implementation.
- `codecraftai-sysman-macos` — macOS implementation.
- `codecraftai-sysman-windows` — Windows implementation.
- `codecraftai-sysman-uefi` — UEFI firmware implementation.

## Layout

```
04-cross-platform-sysman/
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
| 1 — Why a contract layer | drafted | 381 |
| 2 — The 50 cross-OS contracts | drafted | 587 |
| 3 — `featurebase` and the per-OS implementation pattern | drafted | 575 |
| 4 — RHEL implementation | drafted | 468 |
| 5 — Ubuntu / Debian implementation | drafted | 347 |
| 6 — macOS implementation | drafted | 406 |
| 7 — Windows implementation | drafted | 420 |
| 8 — UEFI firmware implementation | drafted | 361 |
| 9 — Spec-driven development for system administration | drafted | 494 |
| 10 — Building cross-OS test parity | drafted | 476 |

**Volume 4 fully drafted: 10 chapters, 4,515 lines, ~42,000 words.**

## Drafting conventions

Same as Volumes 0–3, 5, 6:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference
  into one of the `codecraftai-sysman-*` source repositories.
- Chapter files start with a one-paragraph chapter abstract,
  then section structure (`##` for sections, `###` for
  subsections).
- Forward / backward cross-references link to the relevant
  chapter file with the `chapters/NN-name.md#section-anchor`
  form.

## Cross-volume reference points

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
