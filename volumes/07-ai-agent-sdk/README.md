# Volume 7 — Building an AI Agent SDK

*Lifecycle, Tools, Memory, and Observability for Production Agents*

## About this volume

Volume 7 is the *largest* volume by source-material size.
`codecraftai-agent-kit` is the largest codebase in the portfolio
— ~46k LOC, ~217 modules, 292 testbase test files, 3,744
tests at fast-CI cap (`TESTBASE_MAX_EXAMPLES_CAP=10`).  The
volume is the SDK book — *how to build* a production AI
agent that uses tools, manages memory, runs workflows, and
exposes observability.

The volume *uses* the patterns established in Volumes 0, 1,
3, and 5:

- Volume 0's foundation tier — error vocabulary,
  configuration loading, application identity.
- Volume 1's testbase — the 3,744-test suite that exercises
  the SDK is built on testbase.
- Volume 3's orchestration patterns — agent workflows
  borrow taskplus's flow concepts.
- Volume 5's Measured Interoperability — tools follow the
  Aspects pattern for behaviour parameterisation.

## Who it is for

Developers building production AI agents.  This is the book
the team that ships an agent product reads to learn what
discipline an SDK should impose.  Not a tutorial on AI;
not a survey of LLMs.  It is *the operational discipline*
for building an agent that runs reliably in production.

## Source material

- `codecraftai-agent-kit` — the SDK itself.

## Contents

| Part | Chapters | Theme |
|---|---|---|
| I — Foundations | 1, 2, 3 | SDK vs framework; multi-provider LLM; tool design |
| II — State and Flow | 4, 5, 6 | Memory architecture; workflow orchestration; observability |
| III — Discipline | 7, 8 | Mock-first testing (305 files); property tests at scale (72 files) |
| IV — Worked Example | 9 | Agent that uses the rest of the ccai portfolio |

- [Chapter 1 — Why an SDK and Not a Framework](chapters/01-why-an-sdk-not-a-framework.md)
- [Chapter 2 — Multi-Provider LLM Abstraction](chapters/02-multi-provider-llm.md)
- [Chapter 3 — Tool Design](chapters/03-tool-design.md)
- [Chapter 4 — Memory Architecture](chapters/04-memory-architecture.md)
- [Chapter 5 — Workflow Orchestration for Agents](chapters/05-workflow-orchestration.md)
- [Chapter 6 — Observability for Agents](chapters/06-observability.md)
- [Chapter 7 — Testing AI Agents](chapters/07-testing-ai-agents.md)
- [Chapter 8 — Property Tests at Scale](chapters/08-property-tests-at-scale.md)
- [Chapter 9 — Building an Agent That Uses the Rest of the Code Craft AI Portfolio](chapters/09-portfolio-integration.md)

## How it connects to the other volumes

- **Volume 0** — foundation tier (errors, configuration,
  application identity).
- **Volume 1** — testbase (the 3,744-test suite); the
  case study from Vol 1 ch.10 was the agent-kit suite at
  cap=10.
- **Volume 2** — distributed test execution (when an agent
  runs in a Pod).
- **Volume 3** — taskplus orchestration that invokes agent
  workflows.
- **Volume 5** — Measured Interoperability, Aspects pattern
  for tool dispatch.
- **Volume 8** — time-travel debugging an agent run; ch.6's
  observability composes into Vol 8's replay tooling.

---

Part of [The Code Craft AI Books](../../README.md).
