# Volume 1 — Building a Test Framework

*testbase, arcli, and Single-Host Test Infrastructure*

## About this volume

Volume 1 stands up the *single-host* test framework on top of the
foundation tier from Volume 0.  Everything Volume 2 (distributed)
adds is an extension of contracts established here: the
`TestJob` lifecycle, the resource factory model, the streaming
results pipeline, and the landscape boundary.  Volume 3 reaches
into Volume 1 for orchestration-level invocation — taskplus
calling a testbase test run from a workflow.

## Who it is for

SDETs and test-infrastructure engineers.  This is the volume to
hand a senior SDET who has been writing pytest for a decade and
needs to learn why their tools stop scaling once the system under
test is no longer a single Python process.

## Source material

- `codecraftai-testbase` — the test framework itself
- `codecraftai-arcli` — the declarative CLI toolkit underneath every
  `codecraftai-*` CLI
- `codecraftai-agent-kit` — case study source (a 3,700+ test
  single-host suite)

## Contents

- [Chapter 1 — Why `unittest` and `pytest` Fall Short for Automation](chapters/01-why-unittest-pytest-fall-short.md)
- [Chapter 2 — `testbase` Architecture](chapters/02-testbase-architecture.md)
- [Chapter 3 — Resource Factories and Dependency Injection](chapters/03-resource-factories-and-di.md)
- [Chapter 4 — Property-Based Testing with `@given`](chapters/04-property-based-given.md)
- [Chapter 5 — Streaming JSOS Results](chapters/05-streaming-jsos-results.md)
- [Chapter 6 — The `testbase agents` Machine-Readable CLI](chapters/06-testbase-agents-cli.md)
- [Chapter 7 — Mocking and Stubbing Without the Framework Getting in the Way](chapters/07-mocking-and-stubbing.md)
- [Chapter 8 — `arcli`: The Declarative CLI Toolkit](chapters/08-arcli-the-declarative-cli-toolkit.md)
- [Chapter 9 — Building Tests Around Landscape Concepts (Single-Host Edition)](chapters/09-landscape-concepts-single-host.md)
- [Chapter 10 — Case Study: A 3,700-Test Suite That Runs on One Host](chapters/10-case-study-agent-kit-suite.md)

## How it connects to the other volumes

- Volume 2 (Distributed Automation) extends `TestJob` →
  `InteropLibraryTestJob` → `DistributedTestJob`.  Chapter 9
  carries the single-host half of the contract; chapter 10's case
  study demonstrates the framework before the chain widens.
- Volume 3 (Integration Automation) reaches into Volume 1 for
  test-run invocation from a taskplus workflow.  The arcli
  decorators in chapter 8 are the same ones taskplus uses.
- Volume 7 (AI Agent SDK) reaches into Volume 1 for the `testbase
  agents` machine-readable subcommand surface in chapter 6.

---

Part of [The Code Craft AI Books](../../README.md).
