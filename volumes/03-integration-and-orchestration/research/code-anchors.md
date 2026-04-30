# Volume 3 — Code Anchor Index

A running index of `file:line` references that chapters point at.  Keep
this updated as drafts cite source — it's both a sanity check (the
references resolve) and a discoverability aid for the reader.

Format: each anchor is `[label]: <repo>/<path>:<line>` with a one-line
purpose.  Group by chapter.

---

## Chapter 1 — What integration-level automation is

*(no anchors yet — chapter is conceptual; references will land when
chapter 3 introduces the configuration model)*

## Chapter 2 — Why most CI YAML grows monstrously

*(conceptual chapter; a few illustrative anchors to vendor YAML
patterns may be cited but not from automojo source)*

## Chapter 3 — The contract: six typed configuration categories

- The five existing categories are defined as `MergeMap` instances on
  `CONFIGURATION_MAPS`: `automojo-configuration/source/packages/automojo/configuration/configurationmaps.py`.
- The variable descriptors for each category live in
  `automojo-configuration/source/packages/automojo/configuration/configurationvariables.py`.

## Chapter 4 — Specifications

*(blocked on Q1 — see open-questions.md)*

## Chapter 5 — Landscape files

*(to be researched; see open-questions Q4)*

## Chapter 6 — Topology files

*(to be researched; see open-questions Q4)*

## Chapter 7 — Runtime files vs environment variables

- `TESTBASE_MAX_EXAMPLES_CAP` documented in
  `automojo-testbase/userguide/10-04-03-testing-frameworks-testbase.rst`
  — the smallest possible example of "runtime config switches behaviour
  without code changes."
- `automojo-this-app/source/packages/automojo/thisapp/thisapplication.py`
  `_perform_variable_resolution` — context-path propagation of typed
  configuration into the shared serializable context.

## Chapter 8 — Data profiles

*(to be researched; see open-questions Q4)*

## Chapter 9 — Credential vocabularies

*(blocked on Q2)*

## Chapter 10 — Credential resolution

*(blocked on Q2)*

## Chapter 11 — Credentials in flight

- Fernet encryption: `automojo-configuration/source/packages/automojo/configuration/cryptography.py`.

## Chapter 12 — Credentials across environments

*(blocked on Q2)*

## Chapter 13 — The development-environment ladder

*(blocked on Q3)*

## Chapter 14 — Configuration profiles as a discipline

*(blocked on Q3)*

## Chapter 15 — The local-dev tax

*(blocked on Q3)*

## Chapter 16 — Reproducing CI failures locally

- `TESTBASE_MAX_EXAMPLES_CAP` again as the worked example.

## Chapter 17 — Profile promotion

*(blocked on Q3)*

## Chapter 18 — The taskplus task model

- `FlowNode`: `automojo-taskplus/source/packages/automojo/taskplus/model/flownode.py`.
- `Task`: `automojo-taskplus/source/packages/automojo/taskplus/model/task.py`.
- `TaskGroup`: `automojo-taskplus/source/packages/automojo/taskplus/model/taskgroup.py`.

## Chapter 19 — Built-in task types

- `RunCommand`: `automojo-taskplus/source/packages/automojo/taskplus/tasks/runcommand.py`.
- `ShellScript`: `automojo-taskplus/source/packages/automojo/taskplus/tasks/shellscript.py`.
- `EmbeddedPython`: `automojo-taskplus/source/packages/automojo/taskplus/tasks/embeddedpython.py`.
- `GitCheckout`: `automojo-taskplus/source/packages/automojo/taskplus/tasks/gitcheckout.py`
  (still a stub — chapter calls this out as reader exercise).
- `CreateEnvironment`: `automojo-taskplus/source/packages/automojo/taskplus/tasks/createenvironment.py`
  (chapter argues this is a transitional tool; runtime configs replace
  most uses of it for green-field code).

## Chapter 20 — Workflow YAML grammar

- `Workflow`: `automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py`.
- The `parse` classmethod and `_process_flow_info_items` recursion.
- Validation paths: missing `name`, missing `flow`, conflicting
  `task`+`taskgroup`, missing `items` on a taskgroup, unknown node.

## Chapter 21 — Passing information to jobs

*(needs draft work; the bootstrap-name → resolve-config flow)*

## Chapter 22 — Flow control as data

- `JumpContext`: `automojo-taskplus/source/packages/automojo/taskplus/model/flowengine.py`.
- `FlowEngine.execute` and `process_flowitems`.
- `FlowEngine.evaluate_expression` for the `if_expr` gate.

## Chapter 23 — Extending the task model

- The `import_by_name` / `tasktype: module@ClassName` resolution at
  `automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py:_process_flow_info_items`.

## Chapter 24 — Wiring to a real CI vendor

*(needs sample YAML for GitLab / GitHub / Jenkins shims)*

## Chapter 25 — Bridges to other layers

- Distributed test job invocation: `DistributedTestJob` in
  `automojo-testplus/source/packages/automojo/testplus/`.
- Sysman feature execution: any of the
  `automojo-sysman-*` packages, surfaced through `featurebase`.
- Provisioning: `automojo-provisioning/source/packages/automojo/provisioning/`.

## Chapter 26 — Case study

*(needs the case-study to be selected; see case-studies/ directory)*

## Chapter 27 — Where this approach is not the right fit

*(conceptual; no code anchors)*
