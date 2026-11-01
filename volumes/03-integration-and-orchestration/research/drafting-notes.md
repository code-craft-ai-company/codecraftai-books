# Drafting notes for Volume 3

Moved here from the volume README when the README became the reader's contents page. Not published on the website.

## Drafting conventions

- Markdown throughout.  Code samples in fenced blocks with the language
  tag; YAML configurations as `yaml`; shell commands as `bash`; Python
  as `python`.
- Every claim is anchored on a concrete file:line reference into one of
  the `codecraftai-*` source repositories.  See
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
