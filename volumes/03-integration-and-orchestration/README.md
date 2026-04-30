# Volume 3 — Integration Automation and CI Orchestration

**Working subtitle:** *Typed Configurations as the Contract Between
Orchestrations and Jobs*

This is the working directory for Volume 3.  The series-level entry for
this volume lives in [`automojo-volumes.md`](../../automojo-volumes.md);
this directory is where the volume gets drafted.

## Layout

```
03-integration-and-orchestration/
├── README.md            # this file
├── outline.md           # detailed chapter-by-chapter outline (one level
│                          deeper than automojo-volumes.md)
├── chapters/            # per-chapter draft files
├── research/            # findings on open questions; code-anchor index
├── case-studies/        # full worked-example pipelines (before / after)
├── exercises/           # reader-exercise material per chapter
└── figures/             # diagrams referenced from chapters
```

## Status

| Part | Chapters | Status |
|---|---|---|
| I — The Discipline | 1–3 | drafting |
| II — Configuration Categories | 4–8 | outlining (research-blocked on chapter 4 — see open questions) |
| III — Credential Management | 9–12 | outlining (research-blocked — see open questions) |
| IV — Workflow Across Environments | 13–17 | outlining (research-blocked — see open questions) |
| V — taskplus and the FlowEngine | 18–23 | outlining (well-known terrain) |
| VI — Real-World Wiring | 24–27 | outlining |

See [`research/open-questions.md`](research/open-questions.md) for the
specific blockers.

## Drafting conventions

- Markdown throughout.  Code samples in fenced blocks with the language
  tag; YAML configurations as `yaml`; shell commands as `bash`; Python
  as `python`.
- Every claim is anchored on a concrete file:line reference into one of
  the `automojo-*` source repositories.  See
  [`research/code-anchors.md`](research/code-anchors.md) for the
  running index.
- Chapter files start with a one-paragraph chapter abstract, then
  section structure (`##` for sections, `###` for subsections).
- Forward / backward cross-references link to the relevant chapter file
  with the `chapters/NN-name.md#section-anchor` form so the chapters
  remain self-contained text.

## Drafting order

1. Foreword + chapter 1 (what integration-level automation is)
2. Chapter 2 (why most CI YAML grows monstrously)
3. Chapter 3 (the six configuration categories) — once research clears
4. Part V (taskplus and FlowEngine, chapters 18–23) — well-known
   terrain, reinforces the discipline before Part II/III/IV land
5. Parts II, III, IV in research-clearance order
6. Part VI last — case-study material composes everything before it
