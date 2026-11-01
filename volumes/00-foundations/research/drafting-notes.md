# Drafting notes for Volume 0

Moved here from the volume README when the README became the reader's contents page. Not published on the website.

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
