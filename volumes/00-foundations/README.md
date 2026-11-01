# Volume 0 — Foundations of Automation Engineering with Python

*The CodeCraftX Discipline and the Foundation Tier*

## About this volume

Volume 0 is **prerequisite reading** for the rest of the series.
Every later volume references the CodeCraftX persona, the error
vocabulary, the Aspects pattern, the explicit-`return` convention,
the configuration model, and the application-identity machinery
introduced here.  Without Volume 0, every later volume would have
to re-explain the foundations.

## Who it is for

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

## Contents

- [Chapter 1 — The CodeCraftX Philosophy](chapters/01-the-codecraftai-x-philosophy.md)
- [Chapter 2 — Coding Standards as a Tool, Not a Style Guide](chapters/02-coding-standards-as-a-tool.md)
- [Chapter 3 — The Error Vocabulary](chapters/03-the-error-vocabulary.md)
- [Chapter 4 — Interfaces and Contracts](chapters/04-interfaces-and-contracts.md)
- [Chapter 5 — Waiting with Rich Diagnostics](chapters/05-waiting-with-rich-diagnostics.md)
- [Chapter 6 — Application Identity and Presence](chapters/06-application-identity-and-presence.md)
- [Chapter 7 — Extension and Factory Machinery](chapters/07-extension-and-factory-machinery.md)
- [Chapter 8 — Runtime Variables and Well-Known Paths](chapters/08-runtime-variables-and-well-known-paths.md)
- [Chapter 9 — Configuration Loading](chapters/09-configuration-loading.md)
- [Chapter 10 — Credentials, Data Profiles, Storage Scaffolding](chapters/10-credentials-data-profiles-storage.md)

---

Part of [The Code Craft AI Books](../../README.md).
