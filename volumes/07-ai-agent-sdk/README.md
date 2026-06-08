# Volume 7 — Building an AI Agent SDK

**Working subtitle:** *Lifecycle, Tools, Memory, and Observability for Production Agents*

This is the working directory for Volume 7.  The series-level
entry for this volume lives in
[`codecraftai-volumes.md`](../../codecraftai-volumes.md); this
directory is where the volume gets drafted.

## Volume role in the series

Volume 7 is the *largest* volume by source-material size.
`codecraftai-agent-kit` is the largest codebase in the portfolio
— ~46k LOC, ~217 modules, 292 testbase test files, 3,744
tests at fast-CI cap (`TESTBASE_MAX_EXAMPLES_CAP=10`).  The
volume is the SDK book — *how to build* a production AI
agent that uses tools, manages memory, runs workflows, and
exposes observability.

The volume *uses* the patterns established in Volumes 0, 1,
3, and 5:

- Volume 0's foundation tier — error vocabulary,
  configuration loading, application identity.
- Volume 1's testbase — the 3,744-test suite that exercises
  the SDK is built on testbase.
- Volume 3's orchestration patterns — agent workflows
  borrow taskplus's flow concepts.
- Volume 5's Measured Interoperability — tools follow the
  Aspects pattern for behaviour parameterisation.

## Audience

Developers building production AI agents.  This is the book
the team that ships an agent product reads to learn what
discipline an SDK should impose.  Not a tutorial on AI;
not a survey of LLMs.  It is *the operational discipline*
for building an agent that runs reliably in production.

## Source material

- `codecraftai-agent-kit` — the SDK itself.

## Layout

```
07-ai-agent-sdk/
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
| 1 — Why an SDK and not a framework | drafted | 454 |
| 2 — Multi-provider LLM abstraction | drafted | 480 |
| 3 — Tool design | drafted | 602 |
| 4 — Memory architecture | drafted | 457 |
| 5 — Workflow orchestration for agents | drafted | 512 |
| 6 — Observability for agents | drafted | 494 |
| 7 — Testing AI agents | drafted | 453 |
| 8 — Property tests at scale | drafted | 485 |
| 9 — Building an agent that uses the rest of the ccai portfolio | drafted | 518 |

**Volume 7 fully drafted: 9 chapters, 4,455 lines, ~41,000 words.**

## Drafting conventions

Same as Volumes 0–6:

- Markdown throughout.
- Every claim is anchored on a concrete file:line reference
  into `codecraftai-agent-kit`.
- Chapter files start with a one-paragraph chapter abstract,
  then section structure (`##` for sections, `###` for
  subsections).
- Forward / backward cross-references link to the relevant
  chapter file with the `chapters/NN-name.md#section-anchor`
  form.

## Cross-volume reference points

- **Volume 0** — foundation tier (errors, configuration,
  application identity).
- **Volume 1** — testbase (the 3,744-test suite); the
  case study from Vol 1 ch.10 was the agent-kit suite at
  cap=10.
- **Volume 2** — distributed test execution (when an agent
  runs in a Pod).
- **Volume 3** — taskplus orchestration that invokes agent
  workflows.
- **Volume 5** — Measured Interoperability, Aspects pattern
  for tool dispatch.
- **Volume 8** — time-travel debugging an agent run; ch.6's
  observability composes into Vol 8's replay tooling.
