# Chapter 15 — Case Study: A Landscape That Drives a Sysman Feature Test Across Four OSes

> *Volume 6 walked the environment model in fourteen chapters.
> This chapter watches the model in motion.  A Pod with a
> custom Landscape (cooling Features), a custom Topology
> (cluster + spare), a verification override, an SSH-driven
> sysman feature test across four OS variants, and a status
> readout published to a Flask dashboard.  Every chapter's
> concept lands in the example.*

The case study is concrete.  A real-shape automation
campaign exercising the volume's machinery end to end.  The
target: verify that a sysman feature (Volume 4 covers sysman
in depth; this chapter just uses one as worked target —
"deploy a managed network configuration to a fleet") behaves
correctly across four DUT operating systems.  The reader's
takeaway is a *runtime picture* of how Volume 6's classes
compose at scale.

## The Pod

Volume 3 chapter 4's Specification declares the Pod:

```yaml
# spec-cross-os-network-config.yaml
pod:
    name: cross-os-config-pod
    resources:
        - controller-runner-1
        - dut-rhel-1
        - dut-ubuntu-1
        - dut-macos-1
        - dut-windows-1
        - serial-concentrator-1
        - lpc934-1
        - lab-vault-1
```

CI tooling reads the Specification, performs the pool
checkout, emits Landscape and Topology files.  The Landscape
declares:

```yaml
environment:
    label: cross-os-config

apod:
    clients:
        - deviceType: network/client-linux
          name: controller-runner-1
          host: 10.0.0.1
          role: controller
          credentials:
              - lab-runner

infrastructure:
    services:
        - serviceType: network/service
          name: lab-vault-1
          host: 10.0.0.20
          credentials:
              - vault-admin

        - serviceType: serial/concentrator
          name: serial-concentrator-1
          host: 10.0.0.50

        - serviceType: power/lpc934
          name: lpc934-1
          host: 10.0.0.51

    devices:
        - deviceType: network/ssh
          name: dut-rhel-1
          host: 10.0.1.10
          platform: rhel
          role: dut
          group: workers
          credentials: [dut-root]
          features:
              isolation: false
              serial: serial-concentrator-1: 1
              power: lpc934-1: 1
              cooling: zone-a
        # ... dut-ubuntu-1, dut-macos-1, dut-windows-1 with similar shape ...
```

The device declarations include a project-specific Feature
(`cooling: zone-a`) that the project's `MyProjectLandscape`
subclass (chapter 7) recognises.

## The Topology

```yaml
environment:
    label: cross-os-config

clusters:
    - name: workers
      group: workers
      nodes:
          - dut-rhel-1
          - dut-ubuntu-1
          - dut-macos-1
          - dut-windows-1
      spares: []

cooling_zones:
    - name: zone-a
      devices: [dut-rhel-1, dut-ubuntu-1, dut-macos-1, dut-windows-1]

roles:
    controller: controller-runner-1
```

Three sections:

- **`clusters`** — the canonical shape.  The framework's
  `LandscapeOperationalLayer._create_clusters` materialises
  a `LandscapeDeviceCluster` for the four DUTs.
- **`cooling_zones`** — project-specific.  The project's
  `MyProjectLandscape.validate_topology` (chapter 6) verifies
  the section.
- **`roles`** — names the controller for test code to
  query.

## The custom subclasses

The project ships:

```python
# my_project/landscape.py

from ccai.landscaping.landscape import Landscape
from typing import List, Tuple


class MyProjectLandscape(Landscape):

    def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
        errors = []
        warnings = []

        framework_errors, framework_warnings = super().validate_topology(topology_info)
        errors.extend(framework_errors)
        warnings.extend(framework_warnings)

        # Verify each cooling_zone references real devices.
        cooling_zones = topology_info.get("cooling_zones", [])
        for zone in cooling_zones:
            zone_name = zone["name"]
            for dev_name in zone.get("devices", []):
                if dev_name not in self.devices:
                    errors.append(
                        "Cooling zone '{}' references unknown device '{}'.".format(
                            zone_name, dev_name
                        )
                    )

        return errors, warnings
```

```python
# my_project/devices.py

from ccai.landscaping.landscapedevice import LandscapeDevice


class MyProjectDevice(LandscapeDevice):

    def attach_features(self):
        super().attach_features()  # serial, power, etc.

        if "cooling" in self._features:
            cooling_zone = self._features["cooling"]
            self._cooling_zone = cooling_zone
        return

    @property
    def cooling_zone(self) -> Optional[str]:
        rtnval = getattr(self, "_cooling_zone", None)
        return rtnval
```

```python
# my_project/factories.py

from ccai.factories.factoryextension import FactoryExtension
from ccai.landscaping.landscapingfactoryprotocol import LandscapingFactoryProtocol


class MyProjectLandscapingFactory(FactoryExtension, LandscapingFactoryProtocol):

    PRECEDENCE = 30

    @classmethod
    def get_landscape_type(cls):
        return MyProjectLandscape
```

The factory is registered through the
`ccai.extensions.my_project` namespace package.

## The custom test job

```python
# my_project/testjobs.py

from ccai.testplus.distributedtestjob import DistributedTestJob
from ccai.landscaping.wellknown import LandscapeSingleton


class CrossOSConfigTestJob(DistributedTestJob):
    name = "Cross-OS Network Config"
    description = "Verify managed network configuration deployment across the fleet."

    def _get_landscape(self):
        return LandscapeSingleton()  # returns MyProjectLandscape via factory
```

The entry point:

```python
# tests/cross_os_config/run.py

from ccai.testplus.entrypoints import distributed_test_entrypoint
from my_project.testjobs import CrossOSConfigTestJob


if __name__ == "__main__":
    distributed_test_entrypoint(job_type=CrossOSConfigTestJob)
```

A taskplus orchestration (Volume 3 chapter 25's bridge tasks)
invokes this entry point.

## What activation does

When the orchestration triggers the test job:

1. **Configuration phase.**  Files load.  Credentials
   resolve through `codecraftai-credentials` (Volume 2 chapter
   11).  `MyProjectLandscape.validate_topology` runs:
   - The framework's checks pass (clusters section's
     references all resolve).
   - The cooling_zones section's references all resolve.
   - Validation succeeds.
2. **Integration phase.**  Coordinators instantiate devices:
   - `SshCoordinator` creates `MyProjectDevice` instances
     for the four DUTs (with primary access via
     `device.ssh`).
   - The Feature dispatcher walks each DUT's features:
     - `serial: serial-concentrator-1: 1` →
       `SerialProtocolExtension` attached.
     - `power: lpc934-1: 1` → `PowerProtocolExtension`
       attached.
     - `cooling: zone-a` →
       `MyProjectDevice.attach_features` stores the cooling
       zone for query later.
   - Services instantiate (lab-vault-1, serial-concentrator-1,
     lpc934-1).
3. **Operational phase.**
   - `activate_coordinators` brings each Coordinator online.
   - The cluster is materialised: a `LandscapeDeviceCluster`
     instance with the four DUTs as nodes.
   - `establish_connectivity` verifies each device responds.
     All four DUTs respond.
4. **Verification complete.**  The Pod is realised.

## The first test

```python
import concurrent.futures
import testbase

from my_project.devices import MyProjectDevice
from my_project.validators import DegradedDurationValidator, LogScanningValidator


@testbase.attach.parameter(landscape, identifier="landscape")
@testbase.attach.validator(
    DegradedDurationValidator(max_degraded_seconds=15.0),
    suffix="degraded_during_apply",
)
@testbase.attach.validator(
    LogScanningValidator(error_patterns=["network-config-failed"]),
    suffix="error_log_scan",
)
def test_apply_config_to_fleet(landscape):
    """
        Verify managed network config deploys to all four DUTs.
    """
    fleet = landscape.cluster("workers").nodes.values()
    config = build_test_network_config()

    # Single-actor model: the test (controller) drives the apply
    # in fan-out across the fleet.
    with concurrent.futures.ThreadPoolExecutor(max_workers=len(fleet)) as ex:
        futures = {
            ex.submit(apply_config_via_sysman, dut, config): dut
            for dut in fleet
        }
        results = {
            futures[fut]: fut.result()
            for fut in concurrent.futures.as_completed(futures)
        }

    for dut, (status, _) in results.items():
        assert status == 0, "Config apply failed on {}: {}".format(dut.name, status)

    # Verify configured state on each DUT.
    for dut in fleet:
        actual = read_network_config_via_sysman(dut)
        rtnval = config_matches(config, actual)
        assert rtnval, "Config mismatch on {}".format(dut.name)

    return
```

The test:

- Uses `landscape.cluster("workers").nodes` to enumerate the
  four DUTs from the materialised cluster (chapter 5's
  canonical clusters shape).
- Fans out the apply across the four DUTs concurrently —
  the test, as the single actor, distributes its own work
  (Volume 2 chapter 2's commitment).
- Two tandem validators attached: one watching for fleet
  degradation, one tailing each DUT's syslog for error
  patterns (Volume 2 chapter 5).
- Asserts every apply succeeded; asserts every DUT's state
  matches the config.

The test reads naturally because the framework's classes
provide the right vocabulary at the right level.

## What the JSOS stream records

The framework's recorder writes:

- `phase_started` / `phase_finished` for each activation
  phase.
- `validate_topology` event with `success` outcome.
- `establish_connectivity` event with the four DUTs'
  reachability tuples.
- `test_started` for the test.
- `validator_started` for each of the two validators.
- `test_finished` (passed).
- `validator_finished` for each validator (passed).

Per Volume 1 chapter 5's discipline.  A consumer reading
the stream sees the activation flow followed by the test's
outcomes.

## Recording results to GitLab

After the test job completes, an orchestration task (Volume 3
chapter 25) reports results to GitLab:

```python
from ccai.gitlab.rest.clients.gitlabrestclient import GitLabRestClient
from ccai.gitlab.rest.clients.parameters import (
    NotesPostapiv4projectsidissuesissueiidnotesParams as PostNoteParams
)


def post_results_to_gitlab_issue(client, issue_iid, summary):
    params = PostNoteParams(
        path={"id": "my-project", "issue_iid": issue_iid},
        body={"body": format_summary_markdown(summary)},
    )
    client.notes.postapiv4projectsidissuesissueiidnotesparams(params=params)
    return
```

The auto-generated GitLab method posts a comment on the
issue with the run's structured summary.  Volume 6 chapter
13's hand-written foundation handles authentication; the
generated method handles parameter construction and response
parsing.

## Status dashboard via Flask

A separate web service (built on the Flask scaffolding from
chapter 14) reads the JSOS streams from completed runs and
presents a status dashboard:

```python
# webservice/dashboard.py

from ccai.webservice.xflask.securityblueprint import SecurityBlueprint
from ccai.webservice.xflask.basesecurity import RouteSecurityManager
from flask import Flask, render_template, jsonify

app = Flask(__name__)
security = RouteSecurityManager(app)

dashboard = SecurityBlueprint("dashboard", __name__, route_security=security)


@dashboard.route("/runs")
def runs_view():
    runs = list_recent_runs()
    return render_template("runs.html", runs=runs)


@dashboard.route("/api/runs/<run_id>")
def run_summary(run_id: str):
    summary = read_run_summary(run_id)
    return jsonify(summary)


app.register_blueprint(dashboard)
```

The dashboard ties together the run results, the GitLab
notes, the per-Pod history.  Engineers visit the URL,
authenticate via OAuth (Volume 6 chapter 14's eight
providers), see the status of the campaign.

## What the case study has touched

Every chapter's concept appeared:

- **Chapter 1** — the Landscape's apod / infrastructure shape.
- **Chapter 2** — the device declaration with deviceType +
  Features + metadata.
- **Chapter 3** — Coordinators / Agents / ProtocolExtensions /
  Features in the activation flow.
- **Chapter 4** — `FriendlyIdentifier` for device lookup.
- **Chapter 5** — the canonical `clusters` shape and the
  cooling_zones extension.
- **Chapter 6** — `validate_topology` override checking the
  cooling_zones section.
- **Chapter 7** — `MyProjectLandscape` /
  `MyProjectDevice` subclasses registered through the
  factory protocol.
- **Chapter 8** — `BaseExt`-based service extensions (one
  could imagine `LpcExt` extensions on the LPC934 service).
- **Chapter 9** — the realised Pod with the four DUTs and the
  three services.
- **Chapter 10** — the (currently scaffolding) provisioning
  story producing the Landscape file.
- **Chapter 11** — UI automation (could be added by
  testing the GUI of the management tool).
- **Chapter 12** — declarative UI mappings (extension point).
- **Chapter 13** — GitLab integration posting the result
  comment.
- **Chapter 14** — Flask scaffolding hosting the dashboard.

The case study is the proof that Volume 6's machinery
composes.  A reader who reached this point should be able to
take a real environment-model definition in a project,
identify each construct's framework concept, and predict its
behaviour without running it.

## End of Volume 6

The volume started by walking inside a Landscape file
(chapter 1) and ended with a worked campaign that exercises
every piece of the framework's environment model (this
chapter).

The chapters that come next are in other volumes.  Volume 4
walks the sysman feature tier the case study's tests
exercise.  Volume 5 walks the wire (SSH, REST, serial) the
ProtocolExtensions wrap.  Volume 7 walks the AI Agent SDK
that builds on Volume 5's protocols.  Volume 8 walks
production-grade post-mortem debugging using the JSOS streams
the case study produced.

Volume 6's contribution to the series is the *environment
model*.  The model was implicit in Volumes 1, 2, and 3
(referenced as "the Landscape" or "the Topology"); Volume 6
made it explicit.
