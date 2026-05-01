# Volume 2 — Code-Anchor Index

Running index of file:line references used across Volume 2 chapters.

## automojo-testbase (the substrate Volume 2 extends)

### TestJob (single-host base)

- `source/packages/automojo/testbase/testjob.py:50` — `TestJob` declaration.
- `source/packages/automojo/testbase/testjob.py:415` — `fire_attach_to_environment` (no-op stub on single-host).
- `source/packages/automojo/testbase/testjob.py:425` — `fire_attach_to_framework` (no-op stub).
- `source/packages/automojo/testbase/testjob.py:435` — `fire_activate_configuration` (no-op stub).
- `source/packages/automojo/testbase/testjob.py:439` — `fire_activate_integration` (no-op stub).
- `source/packages/automojo/testbase/testjob.py:443` — `fire_activate_operations` (no-op stub).

### Tandem testing — Validator base class

- `source/packages/automojo/testbase/tandem/validators.py:33` — `Validator(ValidatorCoupling)` declaration.
- `source/packages/automojo/testbase/tandem/validators.py:38–49` — instance variables (`_inst_id`, `_suffix`, `_testscope`, `_parent_scope_id`, `_validator_name`, `_recorder`, `_logger`).
- `source/packages/automojo/testbase/tandem/validators.py:64` — `attach_to_test(testscope, suffix)`.
- `source/packages/automojo/testbase/tandem/validators.py:74` — `TestResult` constructed from `_inst_id`, `_validator_name`, `_parent_scope_id`, monikers, pivots; `recorder.preview(result)`.
- `source/packages/automojo/testbase/tandem/validators.py:78` — `initialize(*args, **kwargs)`.
- `source/packages/automojo/testbase/tandem/validators.py:94` — `finalize()` — calls `validate()`, catches `AssertionError` / `Exception`, records result.
- `source/packages/automojo/testbase/tandem/validators.py:24` — `from automojo.waiting import wait_for_it, TimeoutContext`.
- `source/packages/automojo/testbase/tandem/validators.py:232` — `LoopingValidator.wait()` using `wait_for_it`.
- `source/packages/automojo/testbase/coupling/validatorcoupling.py:44` — `ValidatorCoupling(BaseCoupling)` base.
- `source/packages/automojo/testbase/results/model/testresult.py:29` — `TestResult` data class.

## automojo-interop (the landscape-aware hinge)

### InteropLibraryTestJob

- `source/packages/automojo/interop/testing/testjob.py:22` — `InteropLibraryTestJob(TestJob)` declaration.
- `source/packages/automojo/interop/testing/testjob.py:60–86` — abstract `_get_landscape()` raises `NotImplementedError` with the descriptive message.
- `source/packages/automojo/interop/testing/testjob.py:88–99` — `fire_attach_to_environment(tseq)` calls `tseq.attach_to_environment(landscape=...)`.
- `source/packages/automojo/interop/testing/testjob.py:101–112` — `fire_attach_to_framework(tseq)` calls `tseq.attach_to_framework(landscape=...)`.
- `source/packages/automojo/interop/testing/testjob.py:114–122` — `fire_activate_configuration` calls `landscape.activate_configuration()`.
- `source/packages/automojo/interop/testing/testjob.py:124–132` — `fire_activate_integration` calls `landscape.activate_integration()`.
- `source/packages/automojo/interop/testing/testjob.py:134–142` — `fire_activate_operations` calls `landscape.activate_operations()`.
- Commit `c785e9b` — introduced `InteropLibraryTestJob`; collapsed `DistributedTestJob` from ~615 → 90 lines.

## automojo-testplus (the distributed runner)

### DistributedTestJob

- `source/packages/automojo/testplus/distributedtestjob.py:24` — `DistributedTestJob(InteropLibraryTestJob)` declaration.
- `source/packages/automojo/testplus/distributedtestjob.py:49–57` — `_get_landscape()` returns `LandscapeSingleton()`.
- `source/packages/automojo/testplus/distributedtestjob.py:59–81` — `_create_sequencer()` returns `DistributedTestSequencer`.
- `source/packages/automojo/testplus/distributedtestjob.py:74` — **CONFIRMED BUG**: references `self.title` but base class has `_title` — `AttributeError` at runtime.
- `source/packages/automojo/testplus/distributedtestjob.py:84–90` — `DefaultDistributedTestJob` (naming-only fallback).

### DistributedTestSequencer

- `source/packages/automojo/testplus/distributedtestsequencer.py:33` — `DistributedTestSequencer(TestSequencer)` declaration.
- `source/packages/automojo/testplus/distributedtestsequencer.py:39–55` — `__init__` storing `_landscape = None`.
- `source/packages/automojo/testplus/distributedtestsequencer.py:57–70` — `attach_to_framework(landscape=...)` propagating landscape to integrations.
- `source/packages/automojo/testplus/distributedtestsequencer.py:72–106` — `attach_to_environment(landscape=...)`.
- **Important property:** the sequencer does NOT parallelise tests; it preserves sequential execution from the base `TestSequencer` per the determinism model.

### Entrypoints + factory

- `source/packages/automojo/testplus/entrypoints.py:38` — `distributed_test_entrypoint(job_type=DefaultDistributedTestJob)`.
- `source/packages/automojo/testplus/testplusdefaulttypefactory.py:24` — `TestPlusDefaultTypeFactory(FactoryExtension, TestBaseFactoryProtocol)` precedence 20.
- `source/packages/automojo/testplus/testplusdefaulttypefactory.py:35–47` — `get_testbase_default_job_type` returns `DistributedTestJob`.

## automojo-landscaping (the environment model)

### Landscape and the three layers

- `source/packages/automojo/landscaping/landscape.py:183–240` — `activate_configuration()`.
- `source/packages/automojo/landscaping/landscape.py:242–280` — `activate_integration()`.
- `source/packages/automojo/landscaping/landscape.py:282–312` — `activate_operations()`.

### Configuration layer

- `source/packages/automojo/landscaping/layers/landscapeconfigurationlayer.py:51` — class declaration.
- `source/packages/automojo/landscaping/layers/landscapeconfigurationlayer.py:201–219` — `attach_to_environment()` walks `installed_integration_couplings`.
- `source/packages/automojo/landscaping/layers/landscapeconfigurationlayer.py:267` — `load_topology()`.
- `source/packages/automojo/landscaping/layers/landscapeconfigurationlayer.py:536` — `validate_topology()` stub returning empty errors/warnings (user-overrides).

### Integration layer

- `source/packages/automojo/landscaping/layers/landscapeintegrationlayer.py:43` — class declaration.
- `source/packages/automojo/landscaping/layers/landscapeintegrationlayer.py:295–341` — `initialize_landscape(activation_params)` instantiates devices/services.

### Operational layer

- `source/packages/automojo/landscaping/layers/landscapeoperationallayer.py:41` — class declaration.
- `source/packages/automojo/landscaping/layers/landscapeoperationallayer.py:391–480` — `establish_connectivity(activation_params)` collects `(host, ipaddr, status, stdout, stderr, exc)` tuples; raises `RuntimeError` if `allow_missing_devices=False`.
- `source/packages/automojo/landscaping/layers/landscapeoperationallayer.py:601–634` — `_create_clusters` consuming `topology_info["clusters"]` with `name`/`group`/`nodes`/`spares`.

### Activation params + verification flags

- `source/packages/automojo/landscaping/landscapeparameters.py:19` — `LandscapeActivationParams` dataclass (`disable_device_activation`, `disable_service_activation`, `allow_missing_devices=False`, `allow_unknown_devices=False`, `allow_missing_services=False`, `allow_unknown_services=False`, `validate_features=True`, `validate_topology=True`).

### LandscapeSingleton

- `source/packages/automojo/landscaping/wellknown.py:28–59` — singleton with double-checked locking; factory-overridable via `ConfiguredSuperFactorySingleton`.

### OrchestrationJob (NOT a TestJob)

- `source/packages/automojo/landscaping/orchestrationjob.py:23` — `OrchestrationJob(ContextUser)` declaration.

### Six-pass landscaping audit (case-study material for ch.15)

- Commit `42f9c86` — pass 1: 6 refactor-leftover bugs (Liskov violations, uninitialized attribute, string-concatenation bugs, mutable default).
- Commit `a92e5d2` — pass 2: 2 attribute-name mismatches (`_primary_ssh_credential` vs `_ssh_credential`; `_orch_results_dir` vs `_test_results_dir`).
- Commit `1679fe7` — pass 3: 2 more refactor-leftover bugs (None-vs-empty-dict, file-handle leak).
- Commit `1545f67` — pass 4: 5 logic / behavioural bugs (exclude-filter duplicate-add, `FriendlyIdentifier.__eq__` always True, identity property null-deref, wrong return annotations, inconsistent exception type).

## automojo-protocol-ssh (the wire — Volume 5 territory; Volume 2 references)

- `source/packages/automojo/protocols/ssh/sshsession.py:45–86` — `SshSession` context manager.
- `source/packages/automojo/protocols/ssh/sshagent.py:38–97` — `SshAgent` distributed coordinator vending `SshSession` instances.
- `source/packages/automojo/protocols/ssh/sshaspects.py:77–80` — `Aspects(Freezable)` for behaviour parameterisation.
- `source/packages/automojo/protocols/ssh/sshconnection.py:14–100` — `SshChannel` for raw channel ops.
- `source/packages/automojo/protocols/ssh/sshauth.py:11` — `SshAuthError`.
- `source/packages/automojo/protocols/ssh/sshsftp.py:26` — `SshSFTPError`.
