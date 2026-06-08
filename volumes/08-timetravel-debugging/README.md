# Volume 8 — Time-Travel Debugging in Production

**Working subtitle:** *Pod Instrumentation, Server Ingestion, and the
Replay-Driven Debugging Workflow*

This is the working directory for Volume 8.  The series-level
entry for this volume lives in
[`codecraftai-volumes.md`](../../codecraftai-volumes.md); this
directory is where the volume gets drafted.

## Volume role in the series

Volume 8 covers the marquee artefact of the portfolio:
`codecraftai-timetravel-pod` (the in-process recorder) and
`codecraftai-timetravel-server` (the ingestion + analysis service).
Together they form the production post-mortem tier — the layer
that turns "we couldn't reproduce it" into a replayable trace.

The volume *uses* concepts from earlier volumes:

- Volume 0's foundation tier — error vocabulary, configuration
  loading, the typed-exception convention.
- Volume 1 ch. 5 — the JSOS streaming format and result
  serialisation primitives that the trace protocol echoes.
- Volume 2 ch. 13 — distributed result aggregation; trace
  ingestion sits in the same conceptual slot one tier higher.
- Volume 5 — Measured Interoperability and the Aspects pattern;
  the remote-control client uses the same shape.
- Volume 7 ch. 6 — `ExecutionRecorder` / `ExecutionReplayer` for
  agents.  Volume 8 replaces the per-agent shim with the general
  pod/server tier.

## Audience

Senior engineers and platform / infrastructure teams running
long-lived Python services in production.  The book the on-call
team reads when "logging didn't catch it and we have no repro."
Not a tutorial on debuggers.  Not a survey of tracing tools.
It is *the production-debugging discipline* the portfolio
encodes — instrumented scopes, controlled activation, durable
ingestion, queryable replay.

## Source material

- `codecraftai-timetravel-pod` — the in-process tracer and remote
  control plane.  ~28 modules.  Layers F0 (primitives) through
  F5 (remote control), with F6 (read/replay) partially landed.
- `codecraftai-timetravel-server` — the ingestion + analysis web
  service.  ~45 modules.  Layers S0 (persistence) through S5
  (REST API), built on `codecraftai-webworks`.

## Layout

```
08-timetravel-debugging/
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
| 1 — Why time-travel debugging | drafted | 463 |
| 2 — Pod instrumentation | drafted | 818 |
| 3 — The trace-event protocol | drafted | 688 |
| 4 — Server-side ingestion | drafted | 791 |
| 5 — Remote debug control | drafted | 712 |
| 6 — Session management at scale | drafted | 702 |
| 7 — Case studies | drafted | 674 |
| 8 — Performance and storage trade-offs | drafted | 551 |

**Volume 8 fully drafted: 8 chapters, 5,399 lines, ~52,000 words.**

## Drafting conventions

Same as Volumes 0–7:

- Markdown throughout.
- Every claim is anchored on a concrete file:line reference
  into `codecraftai-timetravel-pod` or `codecraftai-timetravel-server`.
- Formal vocabulary: "Measured Interoperability", "persistence
  behaviours", "Pod" capitalised, "tandem" testing (one driver /
  many observers — bicycle metaphor).
- Chapter files start with a one-paragraph chapter abstract,
  then section structure (`##` for sections, `###` for
  subsections).
- Forward / backward cross-references link to the relevant
  chapter file with the `chapters/NN-name.md#section-anchor`
  form.

## Cross-volume reference points

- **Volume 0** — foundation tier (errors, configuration,
  application identity, `ConfigurationError` / `SemanticError`
  contract).
- **Volume 1 ch. 5** — JSOS streaming format; the trace-event
  protocol's hybrid `struct + pickle` shape borrows the same
  decoupling of envelope from payload.
- **Volume 2 ch. 13** — distributed result aggregation; trace
  ingestion echoes the *single-actor + many-observers* shape
  one tier higher.
- **Volume 4** — cross-platform sysman; a sysman-feature case
  study lives in Volume 8 ch. 7.
- **Volume 5** — Measured Interoperability; the remote-control
  command dispatcher uses Aspects-shaped behaviour parameters.
- **Volume 7 ch. 6** — agent observability; `ExecutionRecorder`
  composes into the pod's portal-scoped recording.
