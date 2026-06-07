# Volume 1 — Code-Anchor Index

Running index of file:line references used across Volume 1 chapters.

## codecraftai-testbase

### Test shape and discovery

- `source/packages/ccai/testbase/testref.py:25–65` — `TestRef` class wraps a `FunctionType` and stores monikers, pivots, and the async flag.
- `source/packages/ccai/testbase/queries.py:29–94` — `collect_test_references()` uses `inspect.getmembers(mod, inspect.isfunction)` (line 68) and wraps matches in `TestRef`.
- `source/packages/ccai/testbase/testcollector.py:60–231` — `TestCollector` orchestrates discovery and builds a `TestGroup` tree.

### Verification

- `source/packages/ccai/testbase/verification.py:520–578` — `verify_dict_has_keys()` and `verify_dict_has_paths()` return `List[str]` of error messages.
- `source/packages/ccai/testbase/verification.py:34–99` — `assert_dict_response_has_keys()` and `assert_dict_response_has_paths()` call the `verify_*` family, then raise `AssertionError`.

### TestJob lifecycle (the runner)

- `source/packages/ccai/testbase/testjob.py:50–131` — `TestJob` class declaration and constructor.
- `source/packages/ccai/testbase/testjob.py:133–139` — `__enter__` / `__exit__` calling `begin()` / `finalize()`.
- `source/packages/ccai/testbase/testjob.py:415` — `fire_attach_to_environment`.
- `source/packages/ccai/testbase/testjob.py:425` — `fire_attach_to_framework`.
- `source/packages/ccai/testbase/testjob.py:435` — `fire_activate_configuration` (no-op stub on single-host).
- `source/packages/ccai/testbase/testjob.py:439` — `fire_activate_integration` (no-op stub on single-host).
- `source/packages/ccai/testbase/testjob.py:443` — `fire_activate_operations` (no-op stub on single-host).
- `source/packages/ccai/testbase/testjob.py:447` — `fire_diagnostic_capture_post_testrun`.
- `source/packages/ccai/testbase/testjob.py:499` — `fire_diagnostic_capture_pre_testrun`.

### Sequencer phases

- `source/packages/ccai/testbase/sequencing/testsequencer.py:106–150` — `TEST_SEQUENCER_PHASES` enum: Initial → Discovery → Collection → Graph → Traversal.

### JSOS streaming results

- `source/packages/ccai/testbase/results/recorders/jsonresultrecorder.py:90–183` — `JsonResultRecorder` writes summary, stream, and consolidated files.
- `source/packages/ccai/testbase/results/recorders/jsonresultrecorder.py:126–151` — JSOS = "JSON Separated Object Stream", each object prefixed by `CHAR_RECORD_SEPERATOR`.
- `source/packages/ccai/testbase/results/streaming/resultstreamer.py:44–78` — `StreamingJsonEncoder` and `ResultStreamer` for background streaming.

### Resource factories and injection

- `source/packages/ccai/testbase/declaration/registrations.py:45–93` — `@integration_factory()` decorator.
- `source/packages/ccai/testbase/declaration/originations.py:31–92` — `parameter()` originator.
- `source/packages/ccai/testbase/declaration/attachments.py` — `@testbase.attach.parameter()` for test-scoped parameters.
- `source/packages/ccai/testbase/resourcelifespan.py` — `ResourceLifespan` enum.
- `source/packages/ccai/testbase/injection/injectionregistry.py` — global injection registry.

### Property-based testing

- `source/packages/ccai/testbase/properties/decorators.py:23–97` — `@given(...)` decorator.
- `source/packages/ccai/testbase/properties/decorators.py:42–44` — `max_examples` and `max_shrink_attempts` constraint keys.
- `source/packages/ccai/testbase/properties/propertyrunner.py:47–79` — `TESTBASE_MAX_EXAMPLES_CAP` env var read and applied.
- `source/packages/ccai/testbase/properties/propertyrunner.py:82–97` — `derive_test_seed()` SHA-256(job_seed + ":" + test_name).
- `source/packages/ccai/testbase/properties/propertyrunner.py:100–120` — `PropertyRunner` orchestration class.

### Mocking primitives

- `source/packages/ccai/testbase/mocking.py:25–43` — `SpecViolationError`.
- `source/packages/ccai/testbase/mocking.py:46–56` — `MockType` enum (Generic, Class, Instance, Property, Function, Method).
- `source/packages/ccai/testbase/mocking.py:59–120` — `SideEffect` class.
- `source/packages/ccai/testbase/mocking.py:304` — `MockContext`.
- `source/packages/ccai/testbase/mocking.py:1603` — `Mock` class.

### CLI: agents subcommand

- `source/packages/ccai/testbase/cli/cmdtree/agents/__init__.py:18–28` — `@arcli.group("agents")`, with subcommands `summary`, `testresult`, `config`.
- `source/packages/ccai/testbase/cli/cmdtree/agents/jsosparser.py` — JSOS parsing utilities.
- `source/packages/ccai/testbase/cli/cmdtree/agents/testresult/list.py` — list-tests subcommand.
- `source/packages/ccai/testbase/cli/cmdtree/agents/testresult/failures.py` — filter-by-failure subcommand.
- `source/packages/ccai/testbase/cli/cmdtree/agents/config/extract.py:32–50` — config extract subcommand.
- `source/packages/ccai/testbase/cli/testbase_command.py:32–64` — root `testbase` entry.

### Couplings

- `source/packages/ccai/testbase/coupling/integrationcoupling.py:512` — `establish_connectivity(allow_missing_devices=False)`.

## codecraftai-arcli

### Command model

- `source/packages/ccai/arcli/model/command.py:22–61` — `Command` class.
- `source/packages/ccai/arcli/model/groupingcommand.py:17–52` — `GroupingCommand` subclass.

### Fields

- `source/packages/ccai/arcli/model/fields/argfield.py:16–62` — `ArgField`.
- `source/packages/ccai/arcli/model/fields/optionfield.py:40–122` — `OptionField` with `OptionMode` (Simple, Multiple, Counter).

### Parameter types

- `source/packages/ccai/arcli/model/parameters/parameterbase.py:13–34` — `ParameterBase`.
- `source/packages/ccai/arcli/model/parameters/stringparameter.py:13–37` — `StringParameter`.
- `source/packages/ccai/arcli/model/parameters/integerparameter.py:12–49` — `IntegerParameter`.
- `source/packages/ccai/arcli/model/parameters/listofstringparameter.py:13–47` — `ListOfStringParameter`.
- `source/packages/ccai/arcli/model/parameters/validators.py:14–227` — `Choice`, `Range`, `Path`, `Regex`, `Custom` validator factories.

### Decorators / registration

- `@arcli.command("name")`, `@arcli.group("name")`, `@arcli.entry("name")` — top-level decorator entry points.
- `@arcli.option(...)`, `@arcli.argument(...)` — field decorators.

### Help generation

- `source/packages/ccai/arcli/helpgenerator.py:15–72` — auto-generated help from command metadata + function `__doc__`.

### Validator hardening (PR #13)

- `source/tests/test_validators.py` — 162-line regression suite for list-mode validators.
- `source/packages/ccai/arcli/model/parameters/validators.py:22–46, 70–97, 129–144, 158–173, 190–215` — list-aware `get_value()` branches.

## codecraftai-interop (forward reference)

- `source/packages/ccai/interop/testing/testjob.py:22` — `InteropLibraryTestJob(TestJob)`.
- `source/packages/ccai/interop/testing/testjob.py:60` — abstract `_get_landscape()`.
- `source/packages/ccai/interop/testing/testjob.py:88, 101, 114, 124, 134` — landscape-aware overrides of the five `fire_*` hooks.

## codecraftai-agent-kit (case study)

- `source/testroots/ccai/tests/singlehost/` — the 3,700-test single-host suite (292 test files; ~3,700+ test functions).
- `Makefile` — dist targets only; test invocation goes through testbase's CLI.
- `source/testroots/ccai/tests/singlehost/agentkit/test_costtracker_properties.py` — example of `@given(..., constraints={"max_examples": 50})`.
