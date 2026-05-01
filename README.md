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
| **0** | [**Foundations of Automation Engineering with Python**](volumes/00-foundations/) | The AutomationX Discipline and the Foundation Tier | **drafted** |
| **1** | [**Building a Test Framework**](volumes/01-test-framework/) | testbase, arcli, and Single-Host Test Infrastructure | **drafted** |
| **2** | [**Distributed Automation**](volumes/02-distributed-automation/) | The TestJob Inheritance Chain and Multi-Host Test Execution | **drafted** |
| **3** | [**Integration Automation and CI Orchestration**](volumes/03-integration-and-orchestration/) | Typed Configurations as the Contract Between Orchestrations and Jobs | **drafted** |
| **4** | [**Cross-Platform System Management**](volumes/04-cross-platform-sysman/) | 50 Cross-OS Contracts and 5 Reference Implementations | **drafted** |
| **5** | [**Interop and Protocols**](volumes/05-interop-and-protocols/) | The Aspects Pattern, the Wire, and the Extension Model | **drafted** |
| **6** | [**Topology, Provisioning, and UI Automation**](volumes/06-topology-provisioning-ui/) | Modelling the Environment Under Automation | **drafted** |
| 7 | [Building an AI Agent SDK](automojo-volumes.md#volume-7--building-an-ai-agent-sdk) | Lifecycle, Tools, Memory, and Observability for Production Agents | planned |
| 8 | [Time-Travel Debugging in Production](automojo-volumes.md#volume-8--time-travel-debugging-in-production) | Pod Instrumentation, Server Ingestion, and the Replay-Driven Debugging Workflow | planned |
| App. | [Operating an Automojo Subscription Platform](automojo-volumes.md#appendix-volume--operating-an-automojo-subscription-platform) | From Git-SHA Pinning to a Published-Package Ecosystem | planned |

### Volume 4 — drafted

Volume 4 is fully drafted in [`volumes/04-cross-platform-sysman/`](volumes/04-cross-platform-sysman/).

| Part | Chapters | Theme |
|---|---|---|
| I — The Contract Layer | 1, 2, 3 | Why contracts; the 50 contracts; `featurebase` substrate |
| II — Five Reference Implementations | 4, 5, 6, 7, 8 | RHEL, Ubuntu, macOS, Windows, UEFI |
| III — The Discipline | 9, 10 | 11 Implementation Conventions; cross-OS test parity |

### Volume 5 — drafted

Volume 5 is fully drafted in [`volumes/05-interop-and-protocols/`](volumes/05-interop-and-protocols/).

| Part | Chapters | Theme |
|---|---|---|
| I — Philosophy and Foundation | 1, 2, 3 | Measured Interoperability; the Aspects pattern; `automojo-interop` survey |
| II — Worked Protocols | 4, 5, 6, 7 | SSH from scratch; networking primitives; REST foundations; the auto-generated GitHub client |
| III — Extensions | 8, 9 | UPnP / Tasker / Bluetooth / CAN; recipe for your own |

### Volume 6 — drafted

Volume 6 is fully drafted in [`volumes/06-topology-provisioning-ui/`](volumes/06-topology-provisioning-ui/).

| Part | Chapters | Theme |
|---|---|---|
| I — The Environment Model | 1, 2, 3, 4 | Landscape, device declaration, Coordinator/Agent/ProtocolExtension/Feature, FriendlyIdentifier |
| II — Topology and Customisation | 5, 6, 7, 8, 9 | Canonical clusters shape, verification overload, subclassing, BaseExt, Pod realisation |
| III — Beyond the Environment | 10, 11, 12, 13, 14 | Provisioning scaffolding, UI automation, declarative mappings, GitLab, Flask |
| IV — Worked Example | 15 | 4-OS sysman feature campaign with the full machinery |

### Volume 2 — drafted

Volume 2 is fully drafted in [`volumes/02-distributed-automation/`](volumes/02-distributed-automation/).

| Part | Chapters | Theme |
|---|---|---|
| I — The Discipline | 1, 2, 3 | Why distributed automation is hard; single-actor determinism; the Pod model |
| II — The Customisation Cascade | 4, 5, 6, 7 | TestJob inheritance chain; tandem testing; landscape activation; verified CI hand-off |
| III — The Runtime Machinery | 8, 9, 10, 11, 12, 13 | Sequencer; job_seed; the wire; credentials; data profiles; result aggregation |
| IV — Honesty and Diagnosis | 14, 15, 16 | Partial-run semantics; debugging a distributed failure; 5-host case study |

### Volume 1 — drafted

Volume 1 is fully drafted in [`volumes/01-test-framework/`](volumes/01-test-framework/).

| Chapter | Theme |
|---|---|
| 1 | Why `unittest` / `pytest` fall short for automation |
| 2 | `testbase` architecture: TestRef, TestCollector, TestSequencer, TestJob, JsonResultRecorder |
| 3 | Resource factories and dependency injection (originators, attachments, the Graph phase) |
| 4 | Property-based testing with `@given`, `TESTBASE_MAX_EXAMPLES_CAP`, deterministic per-test seeds |
| 5 | Streaming JSOS results (JSON Separated Object Stream) |
| 6 | The `testbase agents` machine-readable CLI (`summary`, `testresult`, `config`) |
| 7 | Mocking and stubbing — testbase's bespoke `Mock` / `SpecViolationError` / `SideEffect` |
| 8 | `arcli` — the declarative CLI toolkit underneath every `automojo-*` CLI |
| 9 | Single-host landscape contract — the boundary Volume 2 extends |
| 10 | Case study: the `automojo-agent-kit` 3,700-test single-host suite |

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
