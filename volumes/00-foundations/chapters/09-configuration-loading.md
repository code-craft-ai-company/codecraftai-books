# Chapter 9 — Configuration Loading

> *Where Volume 3 talked about *what* configuration to put in
> files, this chapter talks about *how the loader actually
> works* — the multi-source, multi-format, optionally-encrypted
> file machinery every higher tier relies on.*

The previous chapter introduced the runtime variable layer.
This chapter introduces
[`codecraftai-configuration`](https://github.com/code-craft-ai-company/codecraftai-configuration/tree/main/source/packages/ccai/configuration),
the foundation-tier package whose job is to *load* configuration
files — from disk, from HTTP, from CouchDB, from MongoDB —
parse them, optionally decrypt them, validate them, and produce
typed `MergeMap` objects the rest of the portfolio consumes.

Volume 3 used this package's machinery extensively without
explaining how the machinery works.  This chapter is the
missing piece.  It complements Volume 3 chapters 3 (the six
configuration categories) and 4–8 (the per-category file
formats) by walking the *loader internals* — the
`ConfigurationLoader`, the `ConfigurationSourceBase` hierarchy,
the cryptography primitives, and the resolution pipeline that
drives all of it.

## The loader's job

A consumer wants the data inside a configuration file by
*name*.  They say "give me the landscape named `staging-east`,"
and the loader handles:

1. **Source resolution** — which place(s) to look (a local
   directory, an HTTP endpoint, a CouchDB database)?
2. **Filename derivation** — what file in the source corresponds
   to `staging-east`?  (Tries `staging-east.yaml`,
   `staging-east.yml`, `staging-east.json` in order.)
3. **Format detection** — based on the extension that resolved.
4. **Reading** — fetching the bytes from wherever they are.
5. **Decryption** — when the bytes form an encrypted document.
6. **Parsing** — turning the bytes into a Python dict.
7. **Composition** — overlaying onto a `MergeMap` if the
   category supports composition.

Seven steps.  The consumer sees one call:

```python
config_uri, config_data = loader.load_configuration_by_name("staging-east")
```

The chapter walks the seven steps from bottom to top.

## `ConfigurationLoader` — the public surface

The
[`ConfigurationLoader`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/configurationloader.py)
class is the consumer-facing entry point.  Constructed with a
list of source URIs:

```python
loader = ConfigurationLoader(
    sources=[
        "dir://~/.ccai/configurations/landscapes",
        "http://config.staging-east.internal/landscapes",
        "couchdb://config-couch.internal/landscapes",
    ],
    credentials={...},
    verify_certificate=True,
)
```

Three parameters:

- **`sources`** — list of source URIs.  Each URI's scheme
  selects the source class.  Sources are tried in order.
- **`credentials`** — credential mapping for sources that need
  authentication.
- **`verify_certificate`** — TLS certificate verification flag
  for HTTPS sources.

The class exposes two main methods:

```python
def load_configuration_by_name(self, config_name: str,
                                key: Optional[str] = None,
                                keyphrase: Optional[str] = None) -> Tuple[str, dict]:
    """Returns (source_uri, parsed_config_dict)."""

def load_configuration_from_file(self, config_file: str,
                                  key: Optional[str] = None,
                                  keyphrase: Optional[str] = None) -> dict:
    """Returns parsed_config_dict for a single explicit file path."""
```

The first method walks every source until one returns a
configuration; the second loads a specific file directly.  Both
take optional `key` or `keyphrase` for encrypted documents.

When neither method finds a configuration, the loader raises
`ConfigurationError` (chapter 3's must-bubble type) listing
every source URI it tried.  The error is *findable*; the
engineer reading the failure knows where the loader looked
without having to instrument anything.

## The source hierarchy

Every source extends
[`ConfigurationSourceBase`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/sources/configurationsourcebase.py),
the abstract base that declares two methods:

```python
class ConfigurationSourceBase:

    scheme: str = None    # subclasses override

    @classmethod
    def parse(cls, uri: str) -> Optional["ConfigurationSourceBase"]:
        ...

    def try_load_configuration(self, config_name: str,
                                credentials: Optional[Dict] = None
                                ) -> Tuple[Optional[ConfigurationFormat], Optional[dict]]:
        ...
```

Two methods, one class attribute:

- **`scheme`** — the URI scheme this source handles (`dir`,
  `http`, `couchdb`, `mongodb`).
- **`parse(uri)`** — given a URI, construct an instance of this
  source class if the URI matches the scheme; otherwise return
  `None`.
- **`try_load_configuration(name, credentials)`** — try to load
  the named configuration; return `(format, dict)` on success
  or `(None, None)` if not found.

The shipped source classes:

### `DirectorySource` — filesystem

URIs of the form `dir://path/to/directory`.  Loads files from a
local filesystem.  Implementation detail worth naming: when the
URI doesn't start with `dir://`, the source treats the entire
URI as a directory path — so `~/.ccai/configurations` works
without the prefix.

The directory is expanded through `os.path.expandvars` and
`os.path.expanduser`, then queried with the standard extension
fallback (`.yaml`, `.yml`, `.json` in order).

### `HttpSource` — HTTP / HTTPS endpoints

URIs of the form `http://host/path` or `https://host/path`.
Loads files via HTTP GET requests.  The same extension fallback:
the source tries `<base_url>/<name>.yaml`, then `.yml`, then
`.json`, and returns the first that responds with HTTP 200.

The `verify_certificate` flag from the loader controls TLS
verification; the `credentials` mapping is consulted for HTTP
basic-auth.

### `CouchDbSource` — CouchDB documents

URIs of the form `couchdb://host:port/database`.  Loads
configurations as CouchDB documents — each document's `_id` is
the configuration name.  The `pycouchdb` (or equivalent) library
is required.

The chapter-9 (Volume 3) cross-reference: a `CouchDbProfile`
data profile points at the same kind of CouchDB.  The two
packages share the same protocol — `codecraftai-dataprofiles`'s
profile is the *connection*; `codecraftai-configuration`'s source
is the *configuration storage* using the same connection
mechanics.

### `MongoDbSource` — MongoDB documents

URIs of the form `mongodb://host:port/database`.  Loads
configurations as MongoDB documents.  Same pattern as CouchDB;
the `pymongo` library is required.

## The format detection

Per the configuration loader's logic, file format is detected
by *extension*:

```python
_FORMAT_BY_EXTENSION = {
    ".yaml": ConfigurationFormat.YAML,
    ".yml": ConfigurationFormat.YAML,
    ".json": ConfigurationFormat.JSON,
}
```

`ConfigurationFormat` is the small enum from
[`configurationformat.py`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/configurationformat.py):

```python
class ConfigurationFormat(str, Enum):
    JSON = "json"
    YAML = "yaml"
```

A string-enum dual-citizenship class.  The same pattern as
chapter 6's `ContextPaths` — both a name and a value, greppable
and rename-safe.

The format dictates the parser:

- YAML → `yaml.safe_load(content)`
- JSON → `json.loads(content)`

The choice of `yaml.safe_load` (not `yaml.load`) is the
chapter-1 *errors-never-silent* discipline applied to YAML
parsing — `yaml.load` accepts arbitrary Python object
constructors, which is a security risk; `safe_load` rejects them.

## Encrypted configurations

When the loader receives a parsed dict and detects an
`encrypted_content` key, it activates the cryptography path.

### The encrypted document format

Encrypted configurations have a known shape:

```yaml
version: "1.0"
format: yaml          # the inner content's format
encrypted_content: "BASE64-CIPHERTEXT-HERE"
```

The
[`codecraftai-configuration/cryptography.py`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/cryptography.py)
module produces and consumes documents in this shape.  Volume 3
chapter 11 ("Credentials in flight") detailed the cryptography
primitives for credentials; the same primitives apply to *any*
configuration file.

### `generate_fernet_key(keyphrase)` — deterministic key derivation

```python
def generate_fernet_key(keyphrase: str) -> str:
    rgen = random.Random(keyphrase)
    key_bytes = rgen.randbytes(32)
    key = base64.urlsafe_b64encode(key_bytes)
    return key
```

Deterministic — the same keyphrase always produces the same
key.  The trade-off (Volume 3 chapter 11 discussed in detail):
shared-keyphrase distribution is easier; weak keyphrases produce
weak keys; PBKDF2 / scrypt / Argon2 derivation is the project's
responsibility when stronger derivation is needed.

The `Fernet` class from `cryptography.fernet` accepts any
32-byte URL-safe-base64 key, so a project that wants stronger
derivation passes that key directly to `encrypt_content` /
`decrypt_content` rather than going through
`generate_fernet_key`.

### `encrypt_content` / `decrypt_content` — symmetric round-trip

```python
def encrypt_content(key: str, plain_content: str) -> str:
    cryptor = Fernet(key)
    plain_bytes = plain_content.encode("utf-8")
    cypherbytes = cryptor.encrypt(plain_bytes)
    encrypted_content = base64.b64encode(cypherbytes).decode("utf-8")
    return encrypted_content


def decrypt_content(key: str, encrypted_content: str) -> str:
    cryptor = Fernet(key)
    cipherbytes = base64.b64decode(encrypted_content.encode("utf-8"))
    plainbytes = cryptor.decrypt(cipherbytes)
    plaincontent = plainbytes.decode("utf-8")
    return plaincontent
```

Symmetric, base64-wrapped, UTF-8 strings on both sides.  The
encryption is `cryptography.fernet.Fernet`'s standard AES-128-CBC
+ HMAC-SHA256 + base64 — a well-understood primitive, not a
roll-your-own.

### `create_encrypted_configuration(key, plain_configuration, format)`

The wrapping helper:

```python
def create_encrypted_configuration(key: str, plain_configuration: str,
                                    format: str = ConfigurationFormat.JSON):
    encrypted_configuration = encrypt_content(key, plain_configuration)

    docinfo = {
        "version": "1.0",
        "format": format,
        "encrypted_content": encrypted_configuration,
    }
    return docinfo
```

Takes a plaintext string, encrypts it, wraps it in the
known-shape document.  The result can be written to disk or to
any of the configuration sources; the loader recognises the
shape and decrypts on read.

## The composition layer

Loaded configurations don't exist in isolation — they compose.
A *single* category has *many* files (per-environment landscapes,
per-environment runtimes, etc.); each file contributes to a
unified resolved configuration.

The composition primitive is
[`MergeMap`](https://github.com/code-craft-ai-company/codecraftai-xmodules/blob/main/source/packages/ccai/xmodules/xcollections/mergemap.py)
from `codecraftai-xmodules` (chapter 0's foundation tier).  A
`MergeMap` is a `MutableMapping` that holds an ordered list of
component maps and composes them with *overlay* semantics — later
maps shadow earlier maps key-by-key, but lower-priority entries
remain inspectable.

In `codecraftai-configuration`,
[`configurationmaps.py`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/configurationmaps.py)
defines the five well-known composition targets:

```python
class CONFIGURATION_MAPS:

    CREDENTIAL_CONFIGURATION_MAP = MergeMap()
    DATAPROFILE_CONFIGURATION_MAP = MergeMap()
    LANDSCAPE_CONFIGURATION_MAP = MergeMap()
    RUNTIME_CONFIGURATION_MAP = MergeMap(ConfigurationDefaults.DEFAULT_CONFIGURATION)
    TOPOLOGY_CONFIGURATION_MAP = MergeMap()
```

Five `MergeMap` instances.  Each is the resolved view of *all*
loaded configurations of that category, composed in load order.

### `resolve_*_configuration` — per-category resolution

For each of the five categories, `configurationmaps.py` ships a
resolution function:

```python
def resolve_landscape_configuration(this_app, keyphrase=None, credentials=None):

    landscape_files = this_app.get_variable_value(ConfigVariablesDesc.AM_CONFIG_LANDSCAPE_FILES)
    landscape_names = this_app.get_variable_value(ConfigVariablesDesc.AM_CONFIG_LANDSCAPE_NAMES)
    landscape_sources = this_app.get_variable_value(ConfigVariablesDesc.AM_CONFIG_LANDSCAPE_SOURCES)

    # ... default fallbacks if names/sources unspecified ...

    loader = ConfigurationLoader(sources=landscape_sources, credentials=credentials)

    for config_name in landscape_names:
        config_uri, config_info = loader.load_configuration_by_name(
            config_name, keyphrase=keyphrase
        )
        CONFIGURATION_MAPS.LANDSCAPE_CONFIGURATION_MAP.maps.insert(0, config_info)
```

Three things visible:

- **Variables drive resolution.**  The function reads
  `AM_CONFIG_LANDSCAPE_NAMES`, `_FILES`, `_SOURCES` through the
  chapter-6 descriptor pipeline.  These are the same env-var
  selectors Volume 3 chapter 7 named — `AM_LANDSCAPE_NAMES` is
  shorthand for `AM_CONFIG_LANDSCAPE_NAMES`.
- **Multiple files compose.**  Each loaded configuration is
  *prepended* (`maps.insert(0, ...)`) to the `MergeMap`.  The
  *first* configuration loaded ends up *last* in the chain;
  later loads have higher priority.  This is intentional — it
  means a chain like `[base, staging, staging-east-overlay]`
  resolves with the overlay on top of staging on top of base.
- **The resolved map is shared.**  Every consumer that needs the
  active landscape reads
  `CONFIGURATION_MAPS.LANDSCAPE_CONFIGURATION_MAP`; the same
  instance is updated by every loaded configuration.

### `resolve_configuration_maps` — the umbrella entry

The umbrella function calls each of the five resolution
functions in turn, conditional on the per-category `USE_*`
flags:

```python
def resolve_configuration_maps(this_app=None, use_credentials=None,
                                use_dataprofile=None, use_landscape=None,
                                use_runtime=None, use_topology=None,
                                keyphrase=None, credentials=None):
    if this_app is None:
        this_app = ThisApplicationSingleton()

    # ... resolve use_* flags from variables if not explicitly passed ...

    if use_credentials:
        resolve_credentials_configuration(this_app, keyphrase=keyphrase, credentials=credentials)
    if use_dataprofile:
        resolve_dataprofiles_configuration(this_app, keyphrase=keyphrase, credentials=credentials)
    if use_landscape:
        resolve_landscape_configuration(this_app, keyphrase=keyphrase, credentials=credentials)
    if use_runtime:
        resolve_runtime_configuration(this_app, keyphrase=keyphrase, credentials=credentials)
    if use_topology:
        resolve_topology_configuration(this_app, keyphrase=keyphrase, credentials=credentials)
```

A consumer (typically the application's startup code) calls this
once after `establish_app_persona` and `activate_profile_*`.  By
the time the call returns, all five `CONFIGURATION_MAPS` are
populated, and the rest of the application reads from them.

## The variable model the loader uses

[`configurationvariables.py`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/configurationvariables.py)
defines `ConfigVariablesDesc`, extending the chapter-8
`RuntimeVariablesDesc`:

```python
class ConfigVariablesDesc(RuntimeVariablesDesc):

    AM_CONFIG_CREDENTIAL_NAMES = AppVarDesc(default=None,
        context_path=ContextPaths.CONFIG_CREDENTIAL_NAMES,
        converter=split_and_normalize_name_list)
    AM_CONFIG_CREDENTIAL_FILES = AppVarDesc(default=None,
        context_path=ContextPaths.CONFIG_CREDENTIAL_FILES,
        converter=split_and_normalize_path_list)
    AM_CONFIG_CREDENTIAL_SOURCES = AppVarDesc(default=None,
        context_path=ContextPaths.CONFIG_CREDENTIAL_SOURCES)

    # ... per-category descriptors for landscape, runtime, topology, dataprofile

    AM_CONFIG_USE_CREDENTIALS = AppVarDesc(default=False, source=VariableSource.NONE)
    AM_CONFIG_REQUIRE_CREDENTIALS = AppVarDesc(default=False, converter=parse_bool)
    # ... USE_* / REQUIRE_* per category
```

Per category, six descriptors:

- `AM_CONFIG_<CAT>_NAMES` — the list of config names to load.
- `AM_CONFIG_<CAT>_FILES` — explicit file paths to load (when
  not loaded by name).
- `AM_CONFIG_<CAT>_SOURCES` — the source URI list.
- `AM_CONFIG_<CAT>_URIS` — the resolved-URI output (read-only;
  populated post-load).
- `AM_CONFIG_USE_<CAT>` — boolean controlling whether to load
  this category at all.
- `AM_CONFIG_REQUIRE_<CAT>` — boolean controlling whether
  failure to load is fatal.

Six descriptors per category × five categories = thirty
descriptors, plus their typed accessors in `ConfigVariables`.
This is the chapter-6 descriptor model at scale.

## The input normalizers

The descriptors above use *converter* functions to turn raw
string values into typed lists:
[`normalize.py`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/normalize.py)
ships three:

- **`split_and_normalize_name_list(value, sep=";")`** — splits
  on `;`, strips each entry.
- **`split_and_normalize_path_list(value, sep=";")`** — splits,
  strips `file://` prefix, expands `~` and `$VARS`, makes
  absolute.
- **`split_and_normalize_source_list(value, sep=";")`** —
  splits, expands non-URI paths.  When an entry starts with
  `http://`, `https://`, `dir://`, `mongodb://`, or `couchdb://`,
  it's left alone; otherwise it's path-expanded.

The default separator is `;` rather than `:` because filesystem
paths on Windows use `:` as a drive separator.

## The settings layer

[`configurationsettings.py`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/configurationsettings.py)
provides the *one-time* initialisation function that establishes
the application persona and the default configuration:

```python
def establish_configuration_settings(*, name: str = None,
                                       logger_name: str = None,
                                       log_basename: str = None,
                                       presence_dir: str = None,
                                       log_directory: str = None,
                                       settings_file: str = None,
                                       extensions: str = None,
                                       service_name: str = None,
                                       default_configuration: dict = None,
                                       **other):
```

The function is *idempotent* — guarded by a
`CONFIG_SETTINGS_ESTABLISHED` flag.  It calls
`establish_app_persona` (chapter 6) with the provided
parameters; if `default_configuration` is provided, replaces
`ConfigurationDefaults.DEFAULT_CONFIGURATION`.

This is the typical application's *first* call:

```python
from ccai.configuration.configurationsettings import establish_configuration_settings

establish_configuration_settings(
    name="my-tool",
    logger_name="my-tool",
    log_basename="my-tool",
)
```

One call.  After this, the persona is established (chapter 6),
the runtime layer is available (chapter 8), and the
configuration loader is ready to be invoked through
`resolve_configuration_maps`.

## What the loader is *not*

Three deliberate non-features.

### Not a configuration *editor*

The loader reads configurations.  It does *not* provide a
write-back path for *modifying* them at runtime.  When a
configuration needs to change (a new entry added, an entry
modified, an entry removed), the change happens at the source
(file edit, database update); the loader's next read picks it
up.

For *runtime overrides*, the chapter-6 `override_variable_value`
mechanism applies.  An override doesn't modify the loaded
configuration; it adds a higher-priority resolution that
shadows it.

### Not a generic schema validator

The loader reads YAML/JSON into Python dicts.  Schema
validation — "this dict has these specific fields with these
specific types" — is the consumer's responsibility.  Per
Volume 3 chapter 5 (Landscape files), the loader doesn't
validate the landscape's apod or infrastructure shape; the
landscape *consumer* (typically `codecraftai-landscaping`) does
that.

The `validate` classmethod pattern from Volume 3 (chapter 8 —
data profiles, chapter 9 — credentials) is the per-category
validation hook.  The loader doesn't enforce it; the manager
that builds typed instances from the loaded dict does.

### Not asynchronous

The loader is synchronous.  Loading a configuration blocks the
calling thread until the source returns.  When a configuration
load is *slow* (large files, slow databases, far-away HTTP
servers), the calling code waits.

For most consumers, configuration loading happens *once* at
startup — synchronous loading is the right choice for a
one-time cost.  Consumers that need asynchronous loading
(streaming configuration updates, hot reload) build the
async layer themselves on top of the synchronous loader.

## Cross-references

The configuration loader is consumed by:

- **Volume 3 (Orchestration)** uses the loader extensively.
  Chapter 9 / 10 / 11 / 12 of Volume 3 explain credentials in
  depth; chapters 4–8 explain the per-category content; chapter
  3 lists the categories.  The current chapter is the *loader*
  side of all that material.
- **Volume 1 (Testing)** — `codecraftai-testbase` calls
  `resolve_configuration_maps` at startup to populate the
  active configurations before tests run.
- **Volume 5 (Interop)** — interop's per-protocol adapters
  read connection details from data profiles and credentials
  loaded by this layer.

The chapter-9 (Volume 3) `${VAULT:...}` resolver discipline is
the layer *above* this one — the loader produces dicts whose
secret values may be `${SCHEME:...}` references; the
chapter-10 (Volume 3) resolver turns those references into
materialised secrets.  The loader doesn't know about
references; the resolver doesn't know about file formats.  The
two layers compose cleanly.

## Summary

- **`codecraftai-configuration`** is the configuration-loading
  package every higher tier uses.  The loader fetches
  configuration files from filesystem, HTTP, CouchDB, or
  MongoDB sources; detects format by extension; optionally
  decrypts; parses into Python dicts; composes into
  per-category `MergeMap`s.
- The **`ConfigurationLoader`** is the public surface.  Its
  two methods (`load_configuration_by_name`,
  `load_configuration_from_file`) take care of source iteration,
  filename derivation, format detection, decryption, and
  parsing.
- Four shipped **source classes** under `sources/`:
  `DirectorySource`, `HttpSource`, `CouchDbSource`,
  `MongoDbSource`.  Each extends `ConfigurationSourceBase` with
  a `scheme` attribute and `parse(uri)` / `try_load_configuration`
  methods.
- **Format detection** is by extension: `.yaml` / `.yml` / `.json`
  in that fallback order.  The format determines the parser
  (`yaml.safe_load` for YAML, `json.loads` for JSON).
- **Encrypted configurations** have a known three-key shape
  (`version`, `format`, `encrypted_content`).  The
  `cryptography.py` module ships `generate_fernet_key`,
  `encrypt_content`, `decrypt_content`, and
  `create_encrypted_configuration` — Fernet symmetric
  encryption with deterministic key derivation from a
  keyphrase.
- The **composition layer** uses `MergeMap` to compose multiple
  loaded configurations per category.  Five
  `CONFIGURATION_MAPS` instances (Credentials, DataProfiles,
  Landscape, Runtime, Topology); five `resolve_*_configuration`
  functions; one umbrella `resolve_configuration_maps`.
- **`ConfigVariablesDesc`** extends the chapter-8
  `RuntimeVariablesDesc` with thirty configuration-related
  descriptors — six per category — driving the resolution
  pipeline.  The chapter-7 `AM_*_NAMES` family in Volume 3 is
  shorthand for `AM_CONFIG_*_NAMES`.
- **Input normalizers** (`split_and_normalize_*`) convert
  string env vars into typed lists; the default separator is
  `;` for cross-platform path safety.
- The loader is *not* a configuration editor (read-only),
  *not* a generic schema validator (the consumer validates),
  *not* asynchronous (synchronous loading is the right choice
  for one-time startup cost).

The next chapter — the last in Volume 0 — is the survey
chapter for the remaining foundation-tier packages
(`codecraftai-credentials`, `codecraftai-dataprofiles`, and
`codecraftai-storage-database`).  These three are smaller in
substance than the packages this chapter covered; the survey
treatment fits.
