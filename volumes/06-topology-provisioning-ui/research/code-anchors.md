# Volume 6 — Code-Anchor Index

## codecraftai-landscaping (chapters 1–9)

### Landscape class

- `source/packages/ccai/landscaping/landscape.py:50–174` — `Landscape` class declaration with the four-layer references and the three thread-safe gates (`_configured_gate`, `_integration_gate`, `_operational_gate`).
- `landscape.py:183–240` — `activate_configuration()`.
- `landscape.py:242–280` — `activate_integration()`.
- `landscape.py:282–312` — `activate_operations()`.
- `landscape.py:414–419` — `get_devices(include_filters, exclude_filters)`.
- `landscape.py:421–431` — `get_services(include_filters, exclude_filters)`.

### Devices and services

- `source/packages/ccai/landscaping/landscapedevice.py:41–136` — `LandscapeDevice(FeatureAttachedObject)` with `_friendly_id`, `_extensions: Dict[str, ProtocolExtension]`, `_features` from `device_config["features"]`.
- `landscapedevice.py:242–249` — `is_configured_for_power`.
- `landscapedevice.py:252–259` — `is_configured_for_serial`.
- `landscapedevice.py:356–376` — power Feature attachment via `attach_extension("power", power_ext)`.
- `source/packages/ccai/landscaping/landscapeservice.py:41–80` — `LandscapeService(FeatureAttachedObject)`.

### FriendlyIdentifier

- `source/packages/ccai/landscaping/friendlyidentifier.py:24–51` — class declaration with `_full_identifier`, `_hint`, `_identity_match` (regex).
- `friendlyidentifier.py:68–107` — `identity` property.
- `friendlyidentifier.py:109–116` — `match(hint)`.
- `friendlyidentifier.py:137–165` — `__eq__` (the one whose audit-pass-4 bug was always-True).

### Coordinator hierarchy

- `source/packages/ccai/landscaping/coordinatorbase.py:32–221` — `CoordinatorBase` with abstract `establish_connectivity` (line 177–182), `activate` (line 148–153).
- `source/packages/ccai/landscaping/clients/clientcoordinatorbase.py:55–100` — `ClientCoordinatorBase`.
- `source/packages/ccai/landscaping/services/servicecoordinatorbase.py:57–80` — `ServiceCoordinatorBase`.
- `source/packages/ccai/landscaping/clusters/nodecoordinatorbase.py:57–80` — `NodeCoordinatorBase` (cluster manager).
- `source/packages/ccai/landscaping/coordinators/ssh/sshcoordinator.py:53–157` — `SshCoordinator` with `create_landscape_device` (line 89–135), `establish_connectivity` (line 137–156).
- `source/packages/ccai/landscaping/coordinators/serial/serialcoordinatorbase.py:26–80` — `SerialCoordinatorBase`.

### Coordinator couplings

- `source/packages/ccai/landscaping/clients/clientcoordinatorcouplingbase.py:53–80` — `ClientCoordinatorCouplingBase`.
- `source/packages/ccai/landscaping/coordinators/ssh/sshcoordinatorcoupling.py:44–80` — `SshCoordinatorCoupling`.

### Extensions

- `source/packages/ccai/landscaping/extensions/protocol/protocolextension.py:28–100` — `ProtocolExtension`.
- `source/packages/ccai/landscaping/extensions/control/controlextension.py:27–99` — `ControlExtension`.
- `source/packages/ccai/landscaping/services/vmware/vsphere/ext/baseext.py:22–45` — `BaseExt(VSphereAgent)`.
- `source/packages/ccai/landscaping/services/vmware/vsphere/ext/topologyext.py:23–35` — `TopologyExt(BaseExt)`.

### Cluster materialisation

- `source/packages/ccai/landscaping/landscapedevicecluster.py:23–79` — `LandscapeDeviceCluster` with `name`, `nodes`, `spares`, `group` properties.
- `source/packages/ccai/landscaping/layers/landscapeoperationallayer.py:581–636` — `_create_clusters` consuming `topology_info["clusters"]` with `name`/`group`/`nodes`/`spares`.

### Activation parameters

- `source/packages/ccai/landscaping/landscapeparameters.py:19–33` — `LandscapeActivationParams(SimpleNamespace)` with eight flags.

### OrchestrationJob

- `source/packages/ccai/landscaping/orchestrationjob.py:23–145` — `OrchestrationJob(ContextUser)` with `requires_landscape`, `requires_topology`, `requires_runtime`, `requires_credentials` class attributes.

## codecraftai-uia (chapters 11–12)

### Class hierarchy

- `source/packages/ccai/uia/platforms/base.py:13–33` — `UIObject(ABC)`.
- `base.py:35–72` — `DisplayElement(UIObject)` with `bounds`, `position`, `size`, `is_visible`.
- `base.py:74–107` — `ScreenElement(DisplayElement)` with `bring_to_front`, `send_to_back`.
- `base.py:109–196` — `UIElement(DisplayElement)` with `find_element`, `find_elements`, `wait_for_element`, `wait_until_gone`.
- `base.py:496–536` — `UIWindow(ScreenElement)` with `name`, `title`.
- `base.py:560–601` — `UIApplication(ScreenElement)` with `_declarative_model`.

### Platform wrappers

- `source/packages/ccai/uia/platforms/macos/wrappers/uiaapplication.py:29–80` — `MacApplication(WrappedObject, UIApplication)`.
- `source/packages/ccai/uia/platforms/linux/wrappers/uiaapplication.py:17–71` — `LinuxApplication(LinuxElement, UIApplication)`.
- Windows: stubbed; `NotImplementedError` in factory.

### Declarative model

- `source/packages/ccai/uia/model/declarative.py:14–99` — `ElementHint(Generic[T])` with `type_class`, `name`, `role`, `access_name`, `path`, `identifier`.
- `declarative.py:489–543` — `DeclarativeApplication` with `attach`, `launch_or_attach`, `APPLICATION_NAME`, `APPLICATION_COMMAND`.
- `source/packages/ccai/uia/platforms/macos/mappings/calculator.py:9–48` — `Calculator(DeclarativeApplication)` worked example.
- `source/packages/ccai/uia/platforms/macos/mappings/excel.py:17–71` — `MicrosoftExcel(DeclarativeApplication)`.
- `source/packages/ccai/uia/platforms/linux/mappings/libreoffice_calc.py:37–75` — `LibreOfficeCalc`.

### Factory

- `source/packages/ccai/uia/factory.py` — `AutomationFactory.get_automation_engine()` with platform.system() dispatch and `NotImplementedError` for Windows.

### Audit history

- Commit `79172d0` — "Fix audit-surfaced bugs across uia platform layers" (17 defects).
- Commit `2479151` — 31-file docstring typo fix ("Represents and instance" → "Represents an instance").
- Commit `e5e4cad` — three more Med-severity bugs.
- Total: 20 bugs caught.
- Verified-correct legacy bundle IDs: Calendar (`com.apple.iCal`), Contacts (`com.apple.AddressBook`), Numbers (`com.apple.iWork.Numbers`), Pages (`com.apple.iWork.Pages`).

## codecraftai-provisioning (chapter 10)

### Vendor-neutral abstraction

- `source/packages/ccai/provisioning/model/cloud/cloudprovisioner.py:2–8` — `CloudProvisioner` (currently scaffold; empty `__init__`).
- `source/packages/ccai/provisioning/venues/aws/awscloudprovisioner.py:9–11` — `AwsCloudProvisioner(CloudProvisioner)` (scaffold).
- `source/packages/ccai/provisioning/venues/azure/azurecloudprovisioner.py:9–11` — Azure scaffold.
- `source/packages/ccai/provisioning/venues/google/googlecloudprovisioner.py:9–11` — Google scaffold.

### ProvisioningEngine

- `source/packages/ccai/provisioning/provisioningengine.py:11–52` — `ProvisioningEngine` parses `apod`/`environment`/`venues` from a YAML specification dict.

### Status note

- The package is currently in scaffolding/planning state: `CloudProvisioner.__init__` is `def __init__(): return`, vendor backends are empty subclasses, no checkout-or-create logic implemented yet.  Volume 6 ch.10 is honest about this.

## codecraftai-gitlab (chapter 13)

### Generator + auto-generated client

- `source/packages/ccai/gitlab/rest/generation/restclientgenerator.py:26` —
  `AUTO_GENERATED_BANNER = "# Auto-generated by RestClientGenerator. DO NOT EDIT.\n"`.
- `source/packages/ccai/gitlab/rest/clients/groups/branches.py:16–41` — representative auto-generated method `deleteapiv4projectsidrepositorybranchesbranch`.

### Hand-written foundation

- `source/packages/ccai/gitlab/rest/restclientbase.py:54–75` — `RestClientBase.__init__` with credential, host, scheme, port, default Aspects.
- `source/packages/ccai/gitlab/rest/restapigroup.py` — `RestApiGroup` base for operation-groups.
- `source/packages/ccai/gitlab/rest/aspectsrest.py` — `AspectsRest` for retry/polling patterns.

## codecraftai-webservice-flask (chapter 14)

### Security blueprint and route security

- `source/packages/ccai/webservice/xflask/securityblueprint.py:17–57` — `SecurityBlueprint(Blueprint)` with `_route_security`.
- `source/packages/ccai/webservice/xflask/basesecurity.py:24–78` — `RouteSecurityManager` with `_endpoint_security_table`, `_public_endpoints`.

### OAuth and auth

- `source/packages/ccai/webservice/xflask/baseauthentication.py:16–30` — `OauthProviderNames` with 8 supported providers (Amazon, Apple, Google, GitHub, Microsoft, Okta, Facebook, Gitlab).
- `source/packages/ccai/webservice/xflask/baseendpoints.py` — signup / login / logout / OAuth-callback handlers.

### ORM

- `source/packages/ccai/webservice/xflask/baseorm.py:35–69` — `Credential(WebServiceBase, SerializableMixIn)`, `CredentialResetToken`.
