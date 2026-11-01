# Volume 2 — Distributed Automation

*The TestJob Inheritance Chain and Multi-Host Test Execution*

## About this volume

Volume 2 is the *implementation* of the contracts Volume 1 set up.
Volume 1 introduced `TestJob`, the five `fire_*` lifecycle hooks
as no-op stubs on single-host, the tandem-testing model in
abstract, and the landscape boundary at the contract level.
Volume 2 walks the *bodies* of those hooks — what
`InteropLibraryTestJob` and `DistributedTestJob` do at each
activation phase, how the `DistributedTestSequencer` walks a
distributed suite, how tandem validators attach in practice, how
the SSH wire propagates results, how a partial run is handled
honestly.

Volume 3 then *invokes* what Volume 2 builds.  Volume 3's
orchestration layer hands off names; Volume 2's job layer
realises them, verifies them, and runs the suite.

## Who it is for

Test engineers running multi-host integration suites.  Platform
engineers building automation that crosses machine boundaries.
SDETs scaling beyond one host.  This is the volume to hand a
senior SDET who finished Volume 1, ran their first 12-host
integration suite, and needs to understand what just happened
when host 7 of 12 went down mid-run.

## Source material

- `codecraftai-testplus` — `DistributedTestJob`,
  `DistributedTestSequencer`, `distributed_test_entrypoint`.
- `codecraftai-interop` — `InteropLibraryTestJob`, the abstract
  `_get_landscape()` template method.
- `codecraftai-landscaping` — `Landscape`, the three activation
  layers (Configuration / Integration / Operational), the
  verification methods.
- `codecraftai-testbase` (tandem half) — `Validator`,
  `ValidatorCoupling`, `attach_to_test`.
- `codecraftai-protocol-ssh` — the remote-execution wire.
- `codecraftai-credentials`, `codecraftai-dataprofiles` — the
  cross-machine bits Volume 3 owns at orchestration scale and
  Volume 2 owns at job scale.

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — The Discipline | 1, 2, 3 | Why distributed automation is hard; single-actor determinism; the Pod model |
| II — The Customisation Cascade | 4, 5, 6, 7 | TestJob inheritance chain; tandem testing; landscape activation; verified CI hand-off |
| III — The Runtime Machinery | 8, 9, 10, 11, 12, 13 | Sequencer; job_seed; the wire; credentials; data profiles; result aggregation |
| IV — Honesty and Diagnosis | 14, 15, 16 | Partial-run semantics; debugging a distributed failure; 5-host case study |

- [Chapter 1 — Why Distributed Automation Is Genuinely Hard](chapters/01-why-distributed-automation-is-hard.md)
- [Chapter 2 — Single Actor, Multiple Observers — The Determinism Model](chapters/02-single-actor-determinism.md)
- [Chapter 3 — The Pod Model — Where Parallelism Actually Lives](chapters/03-the-pod-model.md)
- [Chapter 4 — The `TestJob` Inheritance Chain — The Customisation Cascade](chapters/04-the-testjob-inheritance-chain.md)
- [Chapter 5 — Tandem Testing in Depth](chapters/05-tandem-testing-in-depth.md)
- [Chapter 6 — Landscape Activation](chapters/06-landscape-activation.md)
- [Chapter 7 — The CI → Job Hand-off as a Verified Contract](chapters/07-ci-handoff-as-verified-contract.md)
- [Chapter 8 — The `DistributedTestSequencer`](chapters/08-distributed-test-sequencer.md)
- [Chapter 9 — `job_seed` and Reproducibility](chapters/09-job-seed-and-reproducibility.md)
- [Chapter 10 — The Remote-Execution Wire](chapters/10-the-remote-execution-wire.md)
- [Chapter 11 — Credentials at Scale](chapters/11-credentials-at-scale.md)
- [Chapter 12 — Data Profiles in Remote Workers](chapters/12-data-profiles-in-remote-workers.md)
- [Chapter 13 — Result Aggregation](chapters/13-result-aggregation.md)
- [Chapter 14 — Partial-Run Semantics](chapters/14-partial-run-semantics.md)
- [Chapter 15 — Debugging a Distributed Failure](chapters/15-debugging-a-distributed-failure.md)
- [Chapter 16 — Case Study: A 5-Host Integration Test for a Sysman Feature](chapters/16-case-study.md)

## How it connects to the other volumes

- **Volume 0** — error vocabulary, waiting primitives, application
  identity, configuration loading, the CodeCraftX philosophy
  (especially ch.1's meta-thesis: skeleton + patterns).
- **Volume 1** — the single-host substrate; the contract Volume 2
  extends.  Vol 1 ch.1's seventh gap (tandem-bicycle) introduced
  the model; Vol 1 ch.2's customisation cascade introduced
  `TestJob` as the customisation entry; Vol 1 ch.9 established
  the landscape boundary.
- **Volume 3** — orchestration invokes Volume 2's distributed
  jobs; Vol 3 ch.21 names the verification contract Volume 2 ch.7
  walks in detail; Vol 3 ch.25 walks the bridge tasks.
- **Volume 5** — SSH transport in detail; Volume 2 ch.10 sketches
  the wire and forwards to Volume 5.
- **Volume 6** — landscape and topology files in detail (Volume 2
  treats them as inputs; Volume 6 walks their schemas).
- **Volume 8** — time-travel debugging in production; Volume 2
  ch.15 sketches the post-mortem flow and forwards.

---

Part of [The Code Craft AI Books](../../README.md).
