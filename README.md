# The Code Craft AI Books

A series of nine volumes and an appendix on building durable automation software in Python, drawn from the `codecraftai-*` packages.  The
series teaches a discipline (debuggability, explicitness, contracts that cut across packages) that works whether or not you keep using
these packages, and every claim in it is anchored to a file and line in the public source.

The text is written in the open in the [codecraftai-books repository](https://github.com/code-craft-ai-company/codecraftai-books) and
published at <https://code-craft.ai/references/codecraftai>.  The two are the same text.

## The volumes

The recommended reading order is Volume 0 through Volume 8, then the appendix.  Each title opens that volume's contents page.

| Vol | Title | Subtitle | Chapters |
|----:|-------|----------|--------:|
| **0** | [**Foundations of Automation Engineering with Python**](volumes/00-foundations/README.md) | The CodeCraftX Discipline and the Foundation Tier | 10 |
| **1** | [**Building a Test Framework**](volumes/01-test-framework/README.md) | testbase, arcli, and Single-Host Test Infrastructure | 10 |
| **2** | [**Distributed Automation**](volumes/02-distributed-automation/README.md) | The TestJob Inheritance Chain and Multi-Host Test Execution | 16 |
| **3** | [**Integration Automation and CI Orchestration**](volumes/03-integration-and-orchestration/README.md) | Typed Configurations as the Contract Between Orchestrations and Jobs | 28 |
| **4** | [**Cross-Platform System Management**](volumes/04-cross-platform-sysman/README.md) | 52 Cross-OS Contracts and 5 Reference Implementations | 10 |
| **5** | [**Interop and Protocols**](volumes/05-interop-and-protocols/README.md) | The Aspects Pattern, the Wire, and the Extension Model | 10 |
| **6** | [**Topology, Provisioning, and UI Automation**](volumes/06-topology-provisioning-ui/README.md) | Modelling the Environment Under Automation | 15 |
| **7** | [**Building an AI Agent SDK**](volumes/07-ai-agent-sdk/README.md) | Lifecycle, Tools, Memory, and Observability for Production Agents | 9 |
| **8** | [**Time-Travel Debugging in Production**](volumes/08-timetravel-debugging/README.md) | Pod Instrumentation, Server Ingestion, and the Replay-Driven Debugging Workflow | 8 |
| App. | Operating a Code Craft AI Subscription Platform | From Git-SHA Pinning to a Published-Package Ecosystem | planned |

## Who should read which volume

| Profile | Volumes most relevant |
|---|---|
| Junior Python engineer entering automation work | 0, 1, 3 |
| SDET / test-infra engineer | 0, 1, 2, 3 |
| Platform / DevOps engineer building CI pipelines | 0, 3, 4, 6 |
| SRE running cross-OS automation at scale | 0, 3, 4, 6 |
| Integration engineer building protocol implementations | 0, 5 |
| AI / agent developer shipping production agents | 0, 5, 7 |
| Senior engineer / architect needing observability for long-lived services | 0, 8 |
| Internal-tool maintainer running a package distribution | Appendix |

## How the books are written

- The text is Markdown.  Code is in fenced blocks with a language tag.
- Every claim is anchored to a concrete file and line in one of the `codecraftai-*` repositories.  Line numbers move as the code does, so
  treat them as a pointer to where to look.
- The series is narrated in the voice of CodeCraftX, the engineering persona used across the portfolio.
- Each volume lives in `volumes/<NN-name>/`: a `README.md` that is its contents page, and one file per chapter under `chapters/`.

## Status

Volumes 0 to 8 are drafted in full.  The appendix is planned.  These are working drafts, and a draft has errors: where a chapter is wrong,
unclear or out of date with the code, we want to know.

## Errata, questions and suggestions

Open an issue in the [codecraftai-books repository](https://github.com/code-craft-ai-company/codecraftai-books/issues), and say which
chapter and passage you mean.  Issues are the only contribution we accept;
see [CONTRIBUTING.md](https://github.com/code-craft-ai-company/codecraftai-books/blob/main/CONTRIBUTING.md).

## Copyright

Copyright © 2026 Code Craft AI LLC.  All rights reserved.  The software the books describe is licensed separately, in each `codecraftai-*`
repository.
