# Volume 3 — Integration Automation and CI Orchestration

*Typed Configurations as the Contract Between Orchestrations and Jobs*

## About this volume

The orchestration layer is structurally distinct from the job layer, and the thesis that unifies the portfolio's integration story is
*typed configurations as the contract between orchestrations and jobs*.  Volumes 1, 2, 4, 5 and 6 all refer to Part IV ("Workflow
Across Environments"), because profiles are how you eliminate `os.environ.get('CI')` branches across a codebase.  This is the headline
volume of the series and its largest non-AI volume.

## Who it is for

Platform engineers, DevOps and QA infrastructure engineers running multi-step pipelines that need to provision, run, observe and tear
down across multiple machines and configurations.  Also any Python engineer who wants a development workflow that scales from a local
laptop to a nightly full sweep without rewriting code at any boundary.

## Source material

- `codecraftai-taskplus` — the task model and the FlowEngine
- `codecraftai-configuration`, `codecraftai-credentials` — the typed configuration categories and credential handling
- Parts of `codecraftai-this-app`, `codecraftai-landscaping` and `codecraftai-provisioning`

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — The Discipline | Foreword + 1, 2, 3 | The orchestration-vs-jobs layered model; the six-category contract |
| II — Configuration Categories | 4, 5, 6, 7, 8 | Specifications, Landscapes, Topologies, Runtimes, Data Profiles |
| III — Credential Management | 9, 10, 11, 12 | Vocabulary, resolution discipline, in-flight lifecycle, environment spanning |
| IV — Workflow Across Environments | 13, 14, 15, 16, 17 | The four-environment ladder, profile composition, local-dev tax cancellation, CI-failure replay, profile promotion |
| V — taskplus + FlowEngine | 18, 19, 20, 21, 22, 23 | Task model, built-in tasks, YAML grammar, info-passing contract, flow control, extension recipe |
| VI — Real-World Wiring | 24, 25, 26, 27 | CI-vendor shim, bridges to other volumes, case-study capstone, honest scoping |

- [Foreword](chapters/00-foreword.md)
- [Chapter 1 — What Integration-Level Automation Is](chapters/01-what-integration-automation-is.md)
- [Chapter 2 — Why Most CI YAML Grows Monstrously](chapters/02-why-ci-yaml-grows.md)
- [Chapter 3 — The Contract: Six Typed Configuration Categories](chapters/03-the-six-categories.md)
- [Chapter 4 — Specifications](chapters/04-specifications.md)
- [Chapter 5 — Landscape Files](chapters/05-landscape-files.md)
- [Chapter 6 — Topology Files](chapters/06-topology-files.md)
- [Chapter 7 — Runtime Files vs Environment Variables](chapters/07-runtime-files-vs-env-vars.md)
- [Chapter 8 — Data Profiles](chapters/08-data-profiles.md)
- [Chapter 9 — Credential Vocabularies](chapters/09-credential-vocabularies.md)
- [Chapter 10 — Credential Resolution](chapters/10-credential-resolution.md)
- [Chapter 11 — Credentials in Flight](chapters/11-credentials-in-flight.md)
- [Chapter 12 — Credentials Across Environments](chapters/12-credentials-across-environments.md)
- [Chapter 13 — The Development-Environment Ladder](chapters/13-the-development-environment-ladder.md)
- [Chapter 14 — Configuration Profiles as a Discipline](chapters/14-configuration-profiles-as-a-discipline.md)
- [Chapter 15 — The Local-Dev Tax and How Typed Configs Cancel It](chapters/15-the-local-dev-tax.md)
- [Chapter 16 — Reproducing CI Failures Locally](chapters/16-reproducing-ci-failures-locally.md)
- [Chapter 17 — Profile Promotion](chapters/17-profile-promotion.md)
- [Chapter 18 — The taskplus Task Model](chapters/18-the-taskplus-task-model.md)
- [Chapter 19 — Built-in Task Types](chapters/19-built-in-task-types.md)
- [Chapter 20 — Workflow YAML Grammar](chapters/20-workflow-yaml-grammar.md)
- [Chapter 21 — Passing Information to Jobs](chapters/21-passing-information-to-jobs.md)
- [Chapter 22 — Flow Control as Data](chapters/22-flow-control-as-data.md)
- [Chapter 23 — Extending the Task Model](chapters/23-extending-the-task-model.md)
- [Chapter 24 — Wiring to a Real CI Vendor](chapters/24-wiring-to-a-real-ci-vendor.md)
- [Chapter 25 — Bridges to Other Layers](chapters/25-bridges-to-other-layers.md)
- [Chapter 26 — Case Study: Porting a Real Pipeline](chapters/26-case-study.md)
- [Chapter 27 — Where This Approach Is *Not* the Right Fit](chapters/27-where-not-to-fit.md)

---

Part of [The Code Craft AI Books](../../README.md).
