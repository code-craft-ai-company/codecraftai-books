# Volume 1 — Building a Test Framework

**Working subtitle:** *testbase, arcli, and Single-Host Test Infrastructure*

This is the working directory for Volume 1.  The series-level entry
for this volume lives in [`codecraftai-volumes.md`](../../codecraftai-volumes.md);
this directory is where the volume gets drafted.

## Volume role in the series

Volume 1 stands up the *single-host* test framework on top of the
foundation tier from Volume 0.  Everything Volume 2 (distributed)
adds is an extension of contracts established here: the
`TestJob` lifecycle, the resource factory model, the streaming
results pipeline, and the landscape boundary.  Volume 3 reaches
into Volume 1 for orchestration-level invocation — taskplus
calling a testbase test run from a workflow.

## Audience

SDETs and test-infrastructure engineers.  This is the volume to
hand a senior SDET who has been writing pytest for a decade and
needs to learn why their tools stop scaling once the system under
test is no longer a single Python process.

## Source material

- `codecraftai-testbase` — the test framework itself
- `codecraftai-arcli` — the declarative CLI toolkit underneath every
  `codecraftai-*` CLI
- `codecraftai-agent-kit` — case study source (a 3,700+ test
  single-host suite)

## Layout

```
01-test-framework/
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
| 1 — Why `unittest` / `pytest` fall short for automation | drafted | 334 |
| 2 — `testbase` architecture | drafted | 386 |
| 3 — Resource factories and dependency injection | drafted | 415 |
| 4 — Property-based testing with `@given` | drafted | 377 |
| 5 — Streaming JSOS results | drafted | 390 |
| 6 — The `testbase agents` machine-readable CLI | drafted | 380 |
| 7 — Mocking and stubbing without the framework getting in the way | drafted | 372 |
| 8 — `arcli` — the declarative CLI toolkit | drafted | 501 |
| 9 — Building tests around landscape concepts (single-host edition) | drafted | 339 |
| 10 — Case study: a 3,700-test suite that runs on one host | drafted | 378 |

**Volume 1 fully drafted: 10 chapters, 3,872 lines, ~36,000 words.**

## Drafting conventions

Same as Volumes 0 and 3:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference into
  one of the `codecraftai-*` source repositories.
- Chapter files start with a one-paragraph chapter abstract, then
  section structure (`##` for sections, `###` for subsections).
- Forward / backward cross-references link to the relevant chapter
  file with the `chapters/NN-name.md#section-anchor` form.

## Cross-volume forward references

- Volume 2 (Distributed Automation) extends `TestJob` →
  `InteropLibraryTestJob` → `DistributedTestJob`.  Chapter 9
  carries the single-host half of the contract; chapter 10's case
  study demonstrates the framework before the chain widens.
- Volume 3 (Integration Automation) reaches into Volume 1 for
  test-run invocation from a taskplus workflow.  The arcli
  decorators in chapter 8 are the same ones taskplus uses.
- Volume 7 (AI Agent SDK) reaches into Volume 1 for the `testbase
  agents` machine-readable subcommand surface in chapter 6.
