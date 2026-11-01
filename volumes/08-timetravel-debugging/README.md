# Volume 8 — Time-Travel Debugging in Production

*Pod Instrumentation, Server Ingestion, and the Replay-Driven Debugging Workflow*

## About this volume

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

## Who it is for

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

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — Motivation and Surface | 1, 2 | Why time-travel debugging; the gateway / portal / registry triad |
| II — Wire and Ingestion | 3, 4 | Hybrid struct+pickle protocol (with a documented format-mismatch honesty point); TCP/Unix receivers, hysteresis flow control, atomic temp→final |
| III — Control and Operations | 5, 6 | JSON-lines + MCP; spec gap on configuration; multi-tenant data, retention, analysis tier, scaffolded REST API tier |
| IV — Practice | 7, 8 | Three case studies (CI flake / agent / sysman) with code-anchored walkthroughs; granularity-cost-storage trade-offs and the when-to-enable policy |

- [Chapter 1 — Why Time-Travel Debugging](chapters/01-why-time-travel.md)
- [Chapter 2 — Pod Instrumentation](chapters/02-pod-instrumentation.md)
- [Chapter 3 — The Trace-Event Protocol](chapters/03-trace-event-protocol.md)
- [Chapter 4 — Server-Side Ingestion](chapters/04-server-ingestion.md)
- [Chapter 5 — Remote Debug Control](chapters/05-remote-debug-control.md)
- [Chapter 6 — Session Management at Scale](chapters/06-session-management-at-scale.md)
- [Chapter 7 — Case Studies](chapters/07-case-studies.md)
- [Chapter 8 — Performance and Storage Trade-offs](chapters/08-performance-storage-tradeoffs.md)

## How it connects to the other volumes

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

---

Part of [The Code Craft AI Books](../../README.md).
