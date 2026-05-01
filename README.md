# automojo-books

Planning and drafting workspace for the book series drawn from the
`automojo-*` software portfolio.

This repo is a sibling of the code repositories at
`/Users/myron/repos/automojo/`.  It holds:

- [`automojo-volumes.md`](automojo-volumes.md) — the foundation document
  detailing every volume, the chapter-level table of contents, audience,
  cross-references, and source-material notes.  Start here for the
  full series picture.

As volumes move from planning into drafting, each forks into its own
working directory under `volumes/<NN-volume-name>/`.  The master
`automojo-volumes.md` stays the single source of truth for cross-volume
structure.

## Volume index

The series is nine volumes plus an appendix.  The recommended technical
reading order is `0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → Appendix`.

| Vol | Title | Subtitle | Status |
|----:|-------|----------|--------|
| 0 | [Foundations of Automation Engineering with Python](automojo-volumes.md#volume-0--foundations-of-automation-engineering-with-python) | The AutomationX Discipline and the Foundation Tier | planned |
| 1 | [Building a Test Framework](automojo-volumes.md#volume-1--building-a-test-framework) | testbase, arcli, and Single-Host Test Infrastructure | planned |
| 2 | [Distributed Automation](automojo-volumes.md#volume-2--distributed-automation) | The TestJob Inheritance Chain and Multi-Host Test Execution | planned |
| **3** | [**Integration Automation and CI Orchestration**](volumes/03-integration-and-orchestration/) | Typed Configurations as the Contract Between Orchestrations and Jobs | **drafted** |
| 4 | [Cross-Platform System Management](automojo-volumes.md#volume-4--cross-platform-system-management) | 50 Cross-OS Contracts and 5 Reference Implementations | planned |
| 5 | [Interop and Protocols](automojo-volumes.md#volume-5--interop-and-protocols) | The Aspects Pattern, the Wire, and the Extension Model | planned |
| 6 | [Topology, Provisioning, and UI Automation](automojo-volumes.md#volume-6--topology-provisioning-and-ui-automation) | Modelling the Environment Under Automation | planned |
| 7 | [Building an AI Agent SDK](automojo-volumes.md#volume-7--building-an-ai-agent-sdk) | Lifecycle, Tools, Memory, and Observability for Production Agents | planned |
| 8 | [Time-Travel Debugging in Production](automojo-volumes.md#volume-8--time-travel-debugging-in-production) | Pod Instrumentation, Server Ingestion, and the Replay-Driven Debugging Workflow | planned |
| App. | [Operating an Automojo Subscription Platform](automojo-volumes.md#appendix-volume--operating-an-automojo-subscription-platform) | From Git-SHA Pinning to a Published-Package Ecosystem | planned |

### Volume 3 — drafted

Volume 3 is fully drafted in [`volumes/03-integration-and-orchestration/`](volumes/03-integration-and-orchestration/).

| Part | Chapters | Theme |
|---|---|---|
| I — The Discipline | Foreword + 1, 2, 3 | The orchestration-vs-jobs layered model; the six-category contract |
| II — Configuration Categories | 4, 5, 6, 7, 8 | Specifications, Landscapes, Topologies, Runtimes, Data Profiles |
| III — Credential Management | 9, 10, 11, 12 | Vocabulary, resolution discipline, in-flight lifecycle, environment spanning |
| IV — Workflow Across Environments | 13, 14, 15, 16, 17 | The four-environment ladder, profile composition, local-dev tax cancellation, CI-failure replay, profile promotion |
| V — taskplus + FlowEngine | 18, 19, 20, 21, 22, 23 | Task model, built-in tasks, YAML grammar, info-passing contract, flow control, extension recipe |
| VI — Real-World Wiring | 24, 25, 26, 27 | CI-vendor shim, bridges to other volumes, case-study capstone, honest scoping |

Drafting reading order: start at the volume directory's [`README.md`](volumes/03-integration-and-orchestration/README.md) for status, then [`outline.md`](volumes/03-integration-and-orchestration/outline.md) for the granular chapter outline, then the chapter files in numeric order.

## Conventions

- The series narrator is the `AutomationX` Principal-SWE persona used
  across the portfolio's `AGENTS.md` files.
- Every claim is anchored on a concrete file:line reference into one
  of the `automojo-*` source repositories.
- Audit transcripts captured under `volumes/<vol>/case-studies/` become
  appendix material for the relevant volume.
- Markdown throughout for chapter content; YAML / Python / shell in
  fenced blocks with the appropriate language tag.
