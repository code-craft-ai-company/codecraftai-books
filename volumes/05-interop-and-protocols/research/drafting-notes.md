# Drafting notes for Volume 5

Moved here from the volume README when the README became the reader's contents page. Not published on the website.

## Drafting conventions

Same as Volumes 0–4 and 6:

- Markdown throughout.  Code samples in fenced blocks with the
  language tag.
- Every claim is anchored on a concrete file:line reference
  into one of the `codecraftai-*` source repositories.
- Use the formal name **Measured Interoperability** throughout
  — never the shortened "Measured Interop."
- Use **persistence behaviours** as the umbrella term for
  retry / poll / wait / reconnect patterns; the framework's
  `ActionPattern` enum names the canonical behaviours.
- Chapter files start with a one-paragraph chapter abstract,
  then section structure (`##` for sections, `###` for
  subsections).
- Forward / backward cross-references link to the relevant
  chapter file with the `chapters/NN-name.md#section-anchor`
  form.
