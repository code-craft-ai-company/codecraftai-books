# Volume 6 — Topology, Provisioning, and UI Automation

**Working subtitle:** *Modelling the Environment Under Automation*

This is the working directory for Volume 6.  The series-level
entry for this volume lives in
[`automojo-volumes.md`](../../automojo-volumes.md); this
directory is where the volume gets drafted.

## Volume role in the series

Volume 6 is the *environment model* the rest of the platform
operates on.  Volumes 1, 2, and 3 referenced "the Landscape" and
"the Topology" as inputs the framework had already loaded;
Volume 6 walks what's *in* those inputs.  Specifically:

- Landscape: physical inventory + wiring as a configuration
  category.  The framework supplies the loader and the device-
  control machinery (Coordinators, Agents, ProtocolExtensions);
  the project supplies the device-specific schemas via
  subclassing.
- Topology: logical organisation + roles + verification.  The
  framework supplies the canonical `clusters` shape consumed by
  `LandscapeOperationalLayer._create_clusters`; the project
  supplies the rest by subclassing `Topology`.
- Provisioning: the machinery that *creates* a Landscape and
  Topology from a Specification.  Volume 3 chapter 4 walked the
  orchestration side; Volume 6 walks the implementation.
- UI automation: the special case where the system under test
  is a graphical application.  Cross-platform via the
  `automojo-uia` package; declarative application mappings
  for known applications; the same coupling /
  ProtocolExtension model applied to UI elements rather than
  network protocols.

## Audience

Test / automation engineers wiring up complex environments.
Platform engineers building automation infrastructure.
Anyone whose project needs the framework to recognise a new
deviceType, a new Feature, a new Topology shape, or a new
GUI application.

## Source material

- `automojo-landscaping` — the environment model itself.
- `automojo-provisioning` — cloud provisioning backends.
- `automojo-uia` — cross-platform UI automation.
- `automojo-gitlab` — REST integration worked example.
- `automojo-webservice-flask` — web-service scaffolding.

## Layout

```
06-topology-provisioning-ui/
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
| 1 — Landscape — physical inventory and wiring | drafted | 400 |
| 2 — Inside a device declaration | drafted | 321 |
| 3 — Coordinators, Agents, ProtocolExtensions, Features | drafted | 386 |
| 4 — Device discovery and friendly identifiers | drafted | 324 |
| 5 — Topology — logical organisation and roles, with verification | drafted | 296 |
| 6 — Topology as an executable assertion | drafted | 364 |
| 7 — Subclassing `Landscape` and `Topology` — the customisation mechanism | drafted | 385 |
| 8 — Service-specific framework extensions | drafted | 229 |
| 9 — The Pod model — landscape + topology realised | drafted | 238 |
| 10 — Cloud provisioning model | drafted | 293 |
| 11 — Cross-platform UI automation | drafted | 353 |
| 12 — Declarative UI mappings | drafted | 380 |
| 13 — GitLab integration | drafted | 344 |
| 14 — Web-service scaffolding with Flask | drafted | 310 |
| 15 — Case study | drafted | 475 |

**Volume 6 fully drafted: 15 chapters, 5,098 lines, ~47,000 words.**

## Drafting conventions

Same as Volumes 0, 1, 2, 3:

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

- **Volume 0** — foundation tier (configuration loading, error
  vocabulary, factory model from ch.7).
- **Volume 1** — single-host substrate; ch.9's landscape
  boundary contract.
- **Volume 2** — distributed test execution; the three
  activation layers walked at the contract level.  Volume 6
  walks them at the schema / implementation level.
- **Volume 3** — orchestration; ch.4 (Specifications), ch.5
  (Landscape files), ch.6 (Topology files) — Volume 3 walked
  the orchestration-side; Volume 6 walks the
  implementation-side.
- **Volume 4** — sysman; UI automation in Volume 6 chapter 11
  is one *substrate* for sysman tests; Volume 4 walks the
  sysman feature contracts.
- **Volume 5** — interop and protocols; the wire that
  ProtocolExtensions wrap.  Volume 6 references the wire as
  given.
