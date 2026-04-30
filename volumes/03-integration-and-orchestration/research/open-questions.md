# Volume 3 — Open Research Questions

The questions below block specific chapters until they're answered.
Each question lists: the chapter it blocks, what we need to know, where
to look for the answer, and the answer if found.

---

## Q1 — Specifications: configuration category or separate concept?

**Status:** answered.

**Answer:** **(b)** Specifications is a separate concept that lives in
`automojo-provisioning`.  It is **not** a sixth managed type within
`automojo-configuration`.

Evidence:
- `automojo-configuration/source/packages/automojo/configuration/configurationvariables.py:37-42`
  — exactly five `CONFIGURATION_MAPS` instances exist (Credentials,
  DataProfiles, Landscapes, Runtimes, Topologies); no
  `SPECIFICATION_*` map.
- `automojo-configuration/source/packages/automojo/configuration/configurationmaps.py:41-337`
  — five resolution functions; no `resolve_specifications_*`.
- `automojo-provisioning/source/packages/automojo/provisioning/provisioningengine.py:55-73`
  — `create_provisioning_engine(specification: str)` loads a YAML
  specification with top-level `apod` and `environment` keys.

Per-repo `specs/` directories (containing `design.md`,
`requirements.md`, `tasks.md`) are design *documentation*, not runtime
configuration.  The same word is used for two different concepts in
the portfolio.

**Implication for Chapter 4:** anchors on
`automojo-provisioning`'s `ProvisioningEngine.create_provisioning_engine`
and the YAML `apod`/`environment` schema.  The chapter introduces
Specifications as parallel-but-distinct from the five managed
configuration types — a sixth category at the discipline level even
though the portfolio implements it through a different package.

---

## Q2 — Credential resolution backends

**Status:** answered.

**Answer:** `automojo-credentials` is a **pure vocabulary layer**.
Zero resolution backends are shipped.

Vocabulary (11 credential classes):

| Class | Purpose |
|---|---|
| `BasicCredential` | username + password |
| `SshCredential` | SSH with optional password / keyfile / agent |
| `ApiTokenCredential` | single opaque API token |
| `PersonalApiTokenCredential` | API token + username |
| `SecretTokenCredential` | secret token + endpoint pair |
| `AwsAccessKeyCredential` | AWS keyid + secret |
| `AzureClientSecretCredential` | Azure tenant + client + secret |
| `PublicKeyCredential` | generic public/private keypair |
| `GithubTokenCredential` | GitHub PAT + optional username |
| `GithubAppCredential` | GitHub app private key + optional password |
| `WifiChoiceCredential` | list of SSID + password pairs |

All inherit from `BaseCredential` (`identifier`, `categories`, `role`
properties only).  `CredentialManager` is a registry that loads
credentials from a dictionary the consumer parsed; it performs no I/O,
encryption, or backend resolution.

Not shipped: file backend, env-var backend, vault, AWS Secrets
Manager, GCP Secret Manager, Azure Key Vault, keychain/keyring,
1Password / LastPass / Bitwarden, prompt, derived.  No first-class
host scoping, no first-class encryption, no first-class lifetime /
rotation API.

**Implication for Part III (chapters 9–12):** the part documents the
shipped vocabulary, then proposes the backend-resolution discipline
the reader needs to add.  It does *not* tour shipped backends because
there are none.  This is a stronger chapter than the original outline
suggested — the reader gets a clear "here's the contract you're
implementing" anchor.

---

## Q3 — Profile introspection commands and accessors

**Status:** answered.

**Answer:** the portfolio has solid building blocks but **lacks the
user-facing profile machinery** the chapters need.  Chapters 14–17
position the profile discipline as a proposed addition rather than as
documentation of shipped tooling.

Affirmative findings (building blocks that exist):

- **`override_variable_value`** at `automojo-this-app/source/packages/automojo/thisapp/thisapplication.py:222-256`
  — sets `_application_variables[vkey] = new_value`, updates
  `os.environ` if the descriptor source is `ENVIRONMENT`, and
  hot-swaps a small set of `ThisApplication` instance fields
  (`service_name`, `log_basename`, `log_directory`, `logger_name`).
- **`ConfigOptionOverrides`** at `automojo-configuration/source/packages/automojo/configuration/optionoverrides.py:20-254`
  — static methods that wrap `override_variable_value` for the five
  managed configuration types.
- **`MergeMap`** at `automojo-xmodules/source/packages/automojo/xmodules/xcollections/mergemap.py:42-77`
  — `MutableMapping` that holds an ordered list of maps and composes
  them with overlay semantics.  Used as the storage for each
  `CONFIGURATION_MAPS` entry.
- **`get_variable_value`** at `automojo-this-app/source/packages/automojo/thisapp/thisapplication.py:161-219`
  — retrieves a current resolved variable; resolution order is
  override → environment → settings → default.
- **`TESTBASE_MAX_EXAMPLES_CAP`** at `automojo-testbase` (env var
  `ENV_MAX_EXAMPLES_CAP` in `propertyrunner.py`) — the canonical
  ad-hoc runtime knob; this is the smallest example of the
  "config-as-runtime-behavior-switch" pattern.

Negative findings (what's not there):

1. **No CLI command** for "show me the resolved configuration" —
   `testbase` and the other CLIs do not expose `profile show`,
   `config current`, or equivalent.
2. **No config-diff tool** — no `profile diff` between two profiles.
3. **No CI-replay mechanism** — no way to capture full resolved
   variable state from a CI run and replay it locally.
4. **No declarative profile composition syntax** — `MergeMap` exists
   in code, but config files have no `extends:` / `includes:` syntax.
5. **No active-profile name variable** — `ContextPaths` enum
   (`automojo-this-app/source/packages/automojo/thisapp/contextpaths.py:52-82`)
   has `JOB_ID`, `JOB_NAME`, `PIPELINE_ID`, `RUNID`, but no
   `ACTIVE_PROFILE` slot.
6. **No "give me all active variables as a dict" accessor** —
   consumers must iterate descriptor sets and call `get_variable_value`
   per descriptor.

**Implication for Part IV (chapters 13–17):** the part presents the
profile discipline as a *recipe the reader implements*, anchored on
the building blocks above.  Each chapter proposes the missing piece
and shows how to build it on top of the shipped primitives:

- Chapter 14 proposes a `config current` CLI command that iterates
  `AppVarSet.items()` and prints resolved values.
- Chapter 15 proposes a `ContextPaths.ACTIVE_PROFILE` slot and the
  convention of setting it at process start.
- Chapter 16 proposes a config-serialization format and a
  `--replay-config=<file>` flag.
- Chapter 17 proposes a `MergeMap`-driven file-level composition
  syntax (`extends:` / `overlay:`).

This makes the volume more universally applicable.  The discipline
translates to any toolkit; the portfolio gives the reader concrete
building blocks; the missing pieces are exercises the reader can
ship.

---

## Q4 — File formats for each configuration category

**Status:** partial.

**Answer:** all five managed categories accept either YAML or JSON;
the loader tries `.yaml`, `.yml`, `.json` in that order at
`automojo-configuration/source/packages/automojo/configuration/sources/directorysource.py`
and `configurationloader.py`.

Real example files found in the portfolio:

- **Landscape:** `automojo-automation-kit/userguide/examples/landscape.yaml`
  — top-level keys `environment`, `apod`, `infrastructure`.
- **Runtime:** `automojo-testbase/source/testroots/report-tests/result-tests/v2/example2/runtime-declared.yaml`
  — top-level keys `breakpoints`, `debugger`, `logging`, `paths`,
  `paths-templates`, `results-configuration`, `testroot`, `version`.

Real example files **not** found in the portfolio (need to be created
for the book):

- Credentials
- Topologies
- Data Profiles
- Specifications (other than implied by `provisioningengine.py`)

**Implication:** chapters 4 (Specifications), 6 (Topologies), 8 (Data
Profiles), and 9 (Credentials) introduce canonical example files as
part of the chapter.  These should be checked into the book's
`examples/` directory and (where appropriate) contributed back to the
portfolio's `userguide/examples/` directories so the book and the
portfolio stay in sync.

---

## Status legend

- **open** — not yet investigated
- **investigating** — agent dispatched, waiting on results
- **partial** — some answer in hand, gaps remain
- **answered** — sufficient material for the dependent chapters
