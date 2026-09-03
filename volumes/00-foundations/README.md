# Volume 0 — Foundations of Automation Engineering with Python

**Working subtitle:** *The CodeCraftX Discipline and the Foundation Tier*

This is the working directory for Volume 0.  The series-level entry
for this volume lives in [`codecraftai-volumes.md`](../../codecraftai-volumes.md);
this directory is where the volume gets drafted.

## Volume role in the series

Volume 0 is **prerequisite reading** for the rest of the series.
Every later volume references the CodeCraftX persona, the error
vocabulary, the Aspects pattern, the explicit-`return` convention,
the configuration model, and the application-identity machinery
introduced here.  Without Volume 0, every later volume would have
to re-explain the foundations.

## Audience

Any Python engineer building a serious automation platform —
SDETs, platform engineers, SREs, integration engineers.  This is
the book a senior engineer hands a new hire on day one.

## Source material

The volume draws from the foundation tier of the `codecraftai-*`
portfolio:

- `codecraftai-foundations` — one distribution holding the contract layer (`ccai.interfaces`),
  the exception vocabulary and enhanced tracebacks (`ccai.errors`), the waiting primitives
  (`ccai.waiting`), the minimal `Aspects` carrier (`ccai.aspects`) and base classes such as
  `Freezable` (`ccai.bases`). Until 2026-09-02 these were the separate `codecraftai-interfaces`,
  `codecraftai-errors` and `codecraftai-waiting` packages; import paths did not change, and the
  chapters below still describe them by their subpackage names.
- `codecraftai-this-app` — application identity + persona variables
- `codecraftai-extension` — factory + plug-in machinery
- `codecraftai-runtime` — runtime variables + well-known paths
- `codecraftai-configuration` — config loading + multi-source
- `codecraftai-credentials` — typed credential vocabulary
- `codecraftai-dataprofiles` — connection profile classes
- `codecraftai-storage-database` — DB scaffolding
- `codecraftai-xmodules` — xformatting, ximport, xcollections, etc.

## Layout

```
00-foundations/
├── README.md            # this file
├── outline.md           # detailed chapter-by-chapter outline
├── chapters/            # per-chapter draft files
├── research/            # findings on open questions; code-anchor index
├── case-studies/        # full worked-example transcripts
├── exercises/           # reader-exercise material per chapter
└── figures/             # diagrams referenced from chapters
```

## Status

| Chapter | Status | Lines |
|---|---|---|
| 1 — The CodeCraftX philosophy | drafted | 438 |
| 2 — Coding standards as a tool, not a style guide | drafted | 546 |
| 3 — The error vocabulary | drafted | 481 |
| 4 — Interfaces and contracts | drafted | 483 |
| 5 — Waiting with rich diagnostics | drafted | 600 |
| 6 — Application identity and presence | drafted | 550 |
| 7 — Extension and factory machinery | drafted | 458 |
| 8 — Runtime variables and well-known paths | drafted | 522 |
| 9 — Configuration loading | drafted | 637 |
| 10 — Credentials, data profiles, storage scaffolding | drafted | 502 |

**Volume 0 fully drafted: 10 chapters, ~5,200 lines, ~48,000 words.**

## Drafting conventions

Same as Volume 3:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference into
  one of the `codecraftai-*` source repositories.
- Chapter files start with a one-paragraph chapter abstract, then
  section structure (`##` for sections, `###` for subsections).
- Forward / backward cross-references link to the relevant chapter
  file with the `chapters/NN-name.md#section-anchor` form.

## Drafting order

1. Chapter 1 (The CodeCraftX philosophy) — voice setter; conceptual
2. Chapter 2 (Coding standards as a tool) — the rationale chapter
3. Chapter 3 (The error vocabulary) — small, self-contained
4. Chapter 5 (Waiting with rich diagnostics) — small package, easy
5. Chapter 4 (Interfaces and contracts) — needs `ccai.interfaces` survey
6. Chapter 6 (Application identity and presence) — partially explored in Vol 3 prep
7. Chapter 8 (Runtime variables and well-known paths) — small
8. Chapter 7 (Extension and factory machinery) — substantial
9. Chapter 9 (Configuration loading) — explored deeply in Vol 3 prep; Vol 0 angle is loader mechanics
10. Chapter 10 (Credentials / data profiles / storage scaffolding) — survey-style closer
