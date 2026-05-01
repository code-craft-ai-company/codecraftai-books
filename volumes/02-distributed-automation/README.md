# Volume 2 — Distributed Automation

**Working subtitle:** *The TestJob Inheritance Chain and Multi-Host Test Execution*

This is the working directory for Volume 2.  The series-level entry
for this volume lives in [`automojo-volumes.md`](../../automojo-volumes.md);
this directory is where the volume gets drafted.

## Volume role in the series

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

## Audience

Test engineers running multi-host integration suites.  Platform
engineers building automation that crosses machine boundaries.
SDETs scaling beyond one host.  This is the volume to hand a
senior SDET who finished Volume 1, ran their first 12-host
integration suite, and needs to understand what just happened
when host 7 of 12 went down mid-run.

## Source material

- `automojo-testplus` — `DistributedTestJob`,
  `DistributedTestSequencer`, `distributed_test_entrypoint`.
- `automojo-interop` — `InteropLibraryTestJob`, the abstract
  `_get_landscape()` template method.
- `automojo-landscaping` — `Landscape`, the three activation
  layers (Configuration / Integration / Operational), the
  verification methods.
- `automojo-testbase` (tandem half) — `Validator`,
  `ValidatorCoupling`, `attach_to_test`.
- `automojo-protocol-ssh` — the remote-execution wire.
- `automojo-credentials`, `automojo-dataprofiles` — the
  cross-machine bits Volume 3 owns at orchestration scale and
  Volume 2 owns at job scale.

## Layout

```
02-distributed-automation/
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
| 1 — Why distributed automation is genuinely hard | drafting | — |
| 2 — Single actor, multiple observers — the determinism model | pending | — |
| 3 — The Pod model — where parallelism actually lives | pending | — |
| 4 — The TestJob inheritance chain — the customisation cascade | pending | — |
| 5 — Tandem testing in depth | pending | — |
| 6 — Landscape activation | pending | — |
| 7 — The CI → job hand-off as a verified contract | pending | — |
| 8 — The `DistributedTestSequencer` | pending | — |
| 9 — `job_seed` and reproducibility | pending | — |
| 10 — The remote-execution wire | pending | — |
| 11 — Credentials at scale | pending | — |
| 12 — Data profiles in remote workers | pending | — |
| 13 — Result aggregation | pending | — |
| 14 — Partial-run semantics | pending | — |
| 15 — Debugging a distributed failure | pending | — |
| 16 — Case study: a 5-host integration test for a sysman feature | pending | — |

## Drafting conventions

Same as Volumes 0, 1, and 3:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference into
  one of the `automojo-*` source repositories.
- Chapter files start with a one-paragraph chapter abstract,
  then section structure (`##` for sections, `###` for
  subsections).
- Forward / backward cross-references link to the relevant
  chapter file with the `chapters/NN-name.md#section-anchor`
  form.

## Cross-volume reference points

- **Volume 0** — error vocabulary, waiting primitives, application
  identity, configuration loading, the AutomationX philosophy
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
