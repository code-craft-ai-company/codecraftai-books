# Chapter 16 — Case Study: A 5-Host Integration Test for a Sysman Feature

> *Fifteen chapters established the framework, the discipline,
> and the machinery.  This chapter watches them work together.
> The subject is a sysman feature integration test running
> against a 5-host Pod — controller, four DUTs of different
> OSes — verifying the feature behaves correctly across the
> fleet.  Every concept the volume introduced lands in this
> single run.*

The case study is concrete.  A real-shape distributed test
exercising a sysman feature (Volume 4 covers sysman in depth;
this chapter just uses one as the worked target).  The
feature: *deploying a managed network configuration to a
fleet*.  The hosts: a controller running the test, four DUTs
running RHEL, Ubuntu, macOS, and Windows respectively.  The
goal: verify the feature applies the configuration to all four
DUTs, each DUT enters the expected state, and the controller
can verify the state from outside.

Reading order: each section walks one part of the volume's
machinery as it appears in this run.  By the end, every
chapter (1 through 15) has its concept exercised at least
once.

## The Pod (chapters 1, 3, 7)

A Specification declares the Pod:

```yaml
# spec-cross-os-network-config.yaml
pod:
    name: cross-os-network-config-pod
    resources:
        - controller-runner-1
        - dut-rhel-1
        - dut-ubuntu-1
        - dut-macos-1
        - dut-windows-1
```

CI tooling (Volume 3 chapter 4) reads the Specification, checks
out the five resources from the lab pool, emits the Landscape
and Topology files for the next stage.  The CI run continues
with the orchestration triggering a `DistributedTestJob`
against the emitted files.

The Pod is one unit.  Five hosts.  One actor (the
controller).  Four DUTs as responders.  Tests within this Pod
will run sequentially (chapter 2 / 3 / 8 commitments).  If
the campaign needs cross-OS coverage in less wall-clock time,
*more Specifications* (a separate Pod per OS) is the answer
(chapter 3); not parallel tests within this one.

## The Landscape (chapters 9, 11)

The emitted Landscape:

```yaml
environment:
    label: integration-cross-os-config

apod:
    clients:
        -   deviceType: network/client-linux
            name: controller-runner-1
            host: 10.0.0.1
            role: taskerserver
            credentials:
                - lab-runner

infrastructure:
    services:
        -   serviceType: network/service
            name: lab-vault
            credentials:
                - lab-vault-admin

    devices:
        -   deviceType: network/ssh
            name: dut-rhel-1
            host: 10.0.1.10
            platform: rhel
            credentials:
                - dut-root
            features:
                serial: serial-concentrator: 1
                power: lpc934: 1
                isolation: false
            skip: false

        -   deviceType: network/ssh
            name: dut-ubuntu-1
            host: 10.0.1.11
            platform: ubuntu
            credentials:
                - dut-root
            features:
                serial: serial-concentrator: 2
                power: lpc934: 2
                isolation: false
            skip: false

        # ... dut-macos-1, dut-windows-1 similarly ...
```

The `apod` is one host (the controller).  The `infrastructure`
is one service (a credentials vault) plus four devices (the
DUTs).  Each DUT has Features for serial console and power
control routed through a shared serial concentrator and power
controller.

The credential references resolve at activation time
(chapter 11): `lab-runner`, `lab-vault-admin`, `dut-root`.
None of the auth material flows to the DUTs; the controller
authenticates via SSH using the resolved credentials.

## The Topology (chapter 7, with verification)

The emitted Topology:

```yaml
environment:
    label: integration-cross-os-config

clusters:
    -   name: managed-fleet
        group: configuration-targets
        nodes: [dut-rhel-1, dut-ubuntu-1, dut-macos-1, dut-windows-1]
        spares: []

roles:
    controller: controller-runner-1
    fleet_members: [dut-rhel-1, dut-ubuntu-1, dut-macos-1, dut-windows-1]
```

The Topology says four nodes form a `managed-fleet` cluster, no
spares.  The project has subclassed `Topology` (Volume 6
chapter 7's customisation mechanism — applied here) to
recognise the `roles` section and validate that the
controller's role is uniquely assigned and the fleet members
are reachable as a cluster.

The verification override:

```python
class CrossOSConfigTopology(Topology):

    def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
        errors = []
        warnings = []

        # Inherit framework cluster verification
        framework_errors, framework_warnings = super().validate_topology(topology_info)
        errors.extend(framework_errors)
        warnings.extend(framework_warnings)

        # Project-specific role verification
        roles = topology_info.get("roles", {})
        controller_name = roles.get("controller")
        if controller_name is None:
            errors.append("Topology declares no controller role.")
        elif controller_name not in self.devices:
            errors.append(
                "Topology declares controller '{}' but device not in landscape.".format(
                    controller_name
                )
            )

        fleet_members = roles.get("fleet_members", [])
        for member in fleet_members:
            if member not in self.devices:
                errors.append(
                    "Topology declares fleet member '{}' but device not in landscape.".format(
                        member
                    )
                )

        return errors, warnings
```

When the framework calls `validate_topology` during the
`InteropLibraryTestJob.fire_activate_configuration` hook
(chapter 6), this override runs.  Errors abort activation;
warnings log but don't abort.

## The custom `DistributedTestJob` subclass (chapter 4)

The project's job class:

```python
class CrossOSConfigTestJob(DistributedTestJob):

    name = "Cross-OS Network Config"
    description = "Verify managed network configuration deployment across the fleet."

    def _get_landscape(self) -> Any:
        # The custom CrossOSConfigLandscape is registered through
        # the factory protocol, so LandscapeSingleton() returns it.
        landscape = LandscapeSingleton()
        return landscape

    # Inherit _create_sequencer from DistributedTestJob — no override needed
```

The `_get_landscape()` returns the project's `CrossOSConfigLandscape`
subclass (registered via the factory protocol in
`codecraftai-landscaping/wellknown.py:LandscapeSingleton`).  The
landscape's `Topology` subclass is our `CrossOSConfigTopology`
above.  The framework's chain plugs both in.

The entry point:

```python
# tests/cross_os_config/run.py
from ccai.testplus.entrypoints import distributed_test_entrypoint
from myproject.testjobs import CrossOSConfigTestJob

if __name__ == "__main__":
    distributed_test_entrypoint(job_type=CrossOSConfigTestJob)
```

A CI orchestration's bridge task (Volume 3 chapter 25) invokes
this entry point, passing the includes / excludes / runtime /
landscape / topology names.  The test job picks up from there.

## Activation — the three layers in action (chapter 6)

The job starts.  `__enter__` runs `begin()`, which constructs
the recorder and the sequencer.  Then the lifecycle hooks
fire:

1. **`fire_attach_to_environment(tseq)`** — the
   `InteropLibraryTestJob` override fetches the landscape and
   passes it to the sequencer's `attach_to_environment(landscape=...)`.
   The sequencer propagates to integration couplings.
2. **`fire_attach_to_framework(tseq)`** — same shape.
3. **`fire_activate_configuration()`** — calls
   `landscape.activate_configuration()`:
   - `LandscapeConfigurationLayer.load_landscape` reads the
     emitted Landscape file.
   - `load_topology` reads the emitted Topology file.
   - `initialize_credentials` resolves `lab-runner`,
     `lab-vault-admin`, `dut-root` through the credentials
     backend.
   - `validate_topology` runs the project's override.  Roles
     check out; cluster shape checks out.  Configuration phase
     completes.
4. **`fire_activate_integration()`** — calls
   `landscape.activate_integration()`:
   - `LandscapeIntegrationLayer.initialize_landscape` walks the
     Landscape's devices and instantiates `LandscapeDevice`
     objects for each.
   - For each DUT, the `serial` and `power` Features are
     resolved against the `serial-concentrator` and `lpc934`
     services.  ProtocolExtensions are wired:
     `dut.serial`, `dut.power`.
   - The lab-vault service is instantiated.
5. **`fire_activate_operations()`** — calls
   `landscape.activate_operations()`:
   - `LandscapeOperationalLayer.activate_coordinators` brings
     each device online (probe + metadata + extension
     attachment).
   - `establish_connectivity` verifies every device responds.
     SSH handshakes against `dut-rhel-1` (10.0.1.10),
     `dut-ubuntu-1` (10.0.1.11), `dut-macos-1`, `dut-windows-1`.
     All four respond; connectivity is established.

The Pod is now operationally live.  Verification (chapter 7)
has confirmed: every Landscape resource is reachable; the
Topology's cluster and role shape match the Pod's actual
configuration.

## The first test — the actor and its observers (chapters 2, 5)

The first test:

```python
@testbase.attach.parameter(landscape, identifier="landscape")
@testbase.attach.validator(
    DegradedDurationValidator(max_degraded_seconds=15.0),
    suffix="degraded_during_apply",
)
@testbase.attach.validator(
    LogScanningValidator(error_patterns=["network-config-failed"]),
    suffix="error_log_scan",
)
def test_apply_config_to_fleet(landscape: Landscape) -> None:
    """
        Verify that applying a managed network config to all four DUTs
        succeeds, the fleet stays connected, and no error logs appear.
    """
    fleet = [landscape.device(name) for name in [
        "dut-rhel-1", "dut-ubuntu-1", "dut-macos-1", "dut-windows-1"
    ]]
    config = build_test_network_config()

    # Single-actor model: the test (controller) drives the apply
    # in a fan-out across the fleet.
    with concurrent.futures.ThreadPoolExecutor(max_workers=len(fleet)) as ex:
        futures = {
            ex.submit(apply_config_via_sysman, dut, config): dut
            for dut in fleet
        }
        results = {
            futures[fut]: fut.result()
            for fut in concurrent.futures.as_completed(futures)
        }

    # Verify success on every node.
    for dut, (status, _) in results.items():
        assert status == 0, "Config apply failed on {}: {}".format(dut.name, status)

    # Verify the fleet's reported state matches the config.
    for dut in fleet:
        actual = read_network_config_via_sysman(dut)
        rtnval = config_matches(config, actual)
        assert rtnval, "Config mismatch on {}".format(dut.name)

    return
```

The test:

- Has the test as the *single actor* — fan-out is the test's
  parallelism, each DUT is a responder (chapter 2's commitment
  preserved).
- Has two tandem validators attached: a `DegradedDurationValidator`
  watching for fleet-wide degradation during the apply, and a
  `LogScanningValidator` watching each DUT's logs for error
  patterns (chapter 5).
- Uses the `device.sysman` ProtocolExtension (a hypothetical
  per-OS sysman extension; Volume 4 walks the real sysman
  feature tier in detail).
- Asserts twice: every apply succeeded, every DUT's actual
  state matches the config.

While the test runs:
- The `DegradedDurationValidator`'s background thread polls
  fleet-wide health every 500ms.
- The `LogScanningValidator` tails each DUT's syslog over
  SSH and watches for the error patterns.
- The test's fan-out issues the four SSH commands
  concurrently; each completes; the test gathers results.

When the test finishes:
- The two validators' `finalize()` runs.  The
  `DegradedDurationValidator` stops its thread, computes the
  total degraded duration, asserts it's under 15.0 seconds.
  The `LogScanningValidator` stops tailing, asserts no error
  patterns matched.
- The recorder writes the test's `test_finished` record plus
  the two `validator_finished` records.

A passing run produces three "passed" records: the test, plus
each of the two validators.  A failing run might have any
combination — for example, the test's apply succeeded (test
passed) but a validator caught a degraded-duration violation
(validator failed).  The aggregate outcome is failed (chapter
14: any validator failure → test outcome failed).

## The wire and credentials (chapters 10, 11)

Each fan-out call uses the device's `SshAgent` (chapter 10).
The agent holds the resolved `dut-root` credential.  The SSH
session opens using the credential; the command runs.

The credential never travels to the DUT — only the
authenticated session.  The `Aspects` parameter on the
`run_cmd` call is a frozen instance with
`completion_timeout=120`, `connection_timeout=16`.

If a DUT's SSH session drops mid-call (network glitch), the
Aspects' `action_pattern` (e.g., `DO_UNTIL_SUCCESS`) drives
retry behaviour.  Real wire failures surface as
`ConnectionError` or `TimeoutError`; the test's body decides
how to handle each.

## The result aggregation (chapter 13)

As tests finish, the controller's `JsonResultRecorder` writes
JSOS records.  The `LogScanningValidator` produces *its own*
JSOS records as it observes log lines — the records carry
host identity (which DUT's log a line came from) without
forcing a global timeline.

After the suite completes, the controller pulls each DUT's
syslog into `host-logs/<dut_name>/` via the wire.  The JSOS
records reference the pulled log files by path.

The framework's recorder produces:

- `testsummary.json` — `passed: 8, failed: 0, errored: 0,
  skipped: 0, outcome: complete`.
- `testresults_stream.jsos` — every test, every validator,
  every phase recorded.
- `testresults.json` — consolidated.

## A failure case — partial-run semantics (chapter 14)

Suppose `dut-windows-1` becomes unreachable mid-suite (machine
crashed).  What happens?

- The next test's SSH call against `dut-windows-1` raises
  `ConnectionError`.
- The test's body catches the error and either fails or
  errors (depending on the test's logic).
- The recorder writes the test's `test_finished` record with
  `outcome: errored` and the exception.
- Subsequent tests that depend on `dut-windows-1` either
  also error or are skipped (depending on whether the test
  framework can tell ahead of time).
- The `fire_diagnostic_capture_post_testrun` hook runs.
  Pulling `dut-windows-1`'s syslog times out; the JSOS
  record marks `dut-windows-1` as
  `unreachable_at_post_capture`.

The summary reflects the partial state:

```json
{
  "outcome": "partial",
  "tests": {"passed": 5, "failed": 0, "errored": 2, "skipped": 1},
  "host_log_status": {
    "dut-rhel-1": "complete",
    "dut-ubuntu-1": "complete",
    "dut-macos-1": "complete",
    "dut-windows-1": "unreachable_at_post_capture"
  },
  "missing_devices_observed_during_run": ["dut-windows-1"]
}
```

A consumer reading this knows what was tested and what wasn't.
The audit diary (chapter 15) for this run starts with the
symptom, hypothesises the network failure or the OS crash,
walks the available DUT logs to narrow it down.

## What the case study has touched

Every chapter's concept has appeared:

- **Chapters 1, 3** — the Pod, the determinism table, the
  Pod-level parallelism story.
- **Chapter 2** — the single-actor model in the fan-out.
- **Chapter 4** — the custom `DistributedTestJob` subclass with
  `_get_landscape()` returning the project's landscape.
- **Chapter 5** — the two tandem validators in the test.
- **Chapter 6** — the three-layer activation walked through
  the run.
- **Chapter 7** — the verification of Landscape and Topology
  before the first test.
- **Chapter 8** — the `DistributedTestSequencer` walking tests
  serially within the Pod.
- **Chapter 9** — `job_seed` reproducibility (implicit in the
  CI invocation).
- **Chapter 10** — the SSH wire with Aspects.
- **Chapter 11** — credential resolution at activation; safe
  cross-host concurrency.
- **Chapter 12** — data profiles (vault service is the
  example).
- **Chapter 13** — JSOS aggregation with per-host log
  references.
- **Chapter 14** — partial-run semantics on the failure case.
- **Chapter 15** — audit diary for the failure case.

The case study is the proof that the volume's machinery
composes.  A reader who reached this point should be able to
take a real distributed test job in a project, identify each
construct's framework concept, and predict its behaviour
without running it.

## End of Volume 2

The volume started by asking what's different about
distributed automation (chapter 1) and ended by watching a
real cross-OS integration suite use every piece of the
framework's machinery (this chapter).

The chapters that come next are in other volumes.  Volume 3
extends this volume's hand-off contract through orchestration.
Volume 5 walks the wire's machinery in depth.  Volume 6 walks
the Landscape and Topology schemas in depth.  Volume 8
provides the production-grade post-mortem tooling for the
Pod-state-dependent flakes chapter 15 named.

Volume 2's contribution to the series is the *implementation
of the contract*.  The contract was set up at the seam in
Volume 1 chapter 9 (the five `fire_*` hooks as no-op stubs);
this volume gave them bodies.  Every later volume that
references "the distributed test job" is referencing what
was built here.
