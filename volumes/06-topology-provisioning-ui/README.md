# Volume 6 — Topology, Provisioning, and UI Automation

*Modelling the Environment Under Automation*

## About this volume

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
  `codecraftai-uia` package; declarative application mappings
  for known applications; the same coupling /
  ProtocolExtension model applied to UI elements rather than
  network protocols.

## Who it is for

Test / automation engineers wiring up complex environments.
Platform engineers building automation infrastructure.
Anyone whose project needs the framework to recognise a new
deviceType, a new Feature, a new Topology shape, or a new
GUI application.

## Source material

- `codecraftai-landscaping` — the environment model itself.
- `codecraftai-provisioning` — cloud provisioning backends.
- `codecraftai-uia` — cross-platform UI automation.
- `codecraftai-restclient-gitlab` — REST integration worked example.
- `codecraftai-webservice-flask` — web-service scaffolding (retired; see the note at the top of chapter 14).

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — The Environment Model | 1, 2, 3, 4 | Landscape, device declaration, Coordinator/Agent/ProtocolExtension/Feature, FriendlyIdentifier |
| II — Topology and Customisation | 5, 6, 7, 8, 9 | Canonical clusters shape, verification overload, subclassing, BaseExt, Pod realisation |
| III — Beyond the Environment | 10, 11, 12, 13, 14 | Provisioning scaffolding, UI automation, declarative mappings, GitLab, Flask |
| IV — Worked Example | 15 | 4-OS sysman feature campaign with the full machinery |

- [Chapter 1 — Landscape — Physical Inventory and Wiring](chapters/01-landscape-physical-inventory.md)
- [Chapter 2 — Inside a Device Declaration](chapters/02-inside-a-device-declaration.md)
- [Chapter 3 — Coordinators, Agents, ProtocolExtensions, Features](chapters/03-coordinators-agents-protocol-extensions-features.md)
- [Chapter 4 — Device Discovery and Friendly Identifiers](chapters/04-friendly-identifier.md)
- [Chapter 5 — Topology — Logical Organisation and Roles, with Verification](chapters/05-topology-logical-organisation.md)
- [Chapter 6 — Topology as an Executable Assertion](chapters/06-topology-as-executable-assertion.md)
- [Chapter 7 — Subclassing `Landscape` and `Topology` — The Customisation Mechanism](chapters/07-subclassing-landscape-and-topology.md)
- [Chapter 8 — Service-Specific Framework Extensions](chapters/08-service-specific-baseext.md)
- [Chapter 9 — The Pod Model — Landscape + Topology Realised](chapters/09-the-pod-model.md)
- [Chapter 10 — Cloud Provisioning Model](chapters/10-cloud-provisioning-model.md)
- [Chapter 11 — Cross-Platform UI Automation](chapters/11-cross-platform-ui-automation.md)
- [Chapter 12 — Declarative UI Mappings](chapters/12-declarative-ui-mappings.md)
- [Chapter 13 — GitLab Integration](chapters/13-gitlab-integration.md)
- [Chapter 14 — Web-Service Scaffolding with Flask](chapters/14-flask-scaffolding.md)
- [Chapter 15 — Case Study: A Landscape That Drives a Sysman Feature Test Across Four OSes](chapters/15-case-study.md)

## How it connects to the other volumes

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

---

Part of [The Code Craft AI Books](../../README.md).
