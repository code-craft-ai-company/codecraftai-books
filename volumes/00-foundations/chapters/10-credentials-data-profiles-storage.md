# Chapter 10 — Credentials, Data Profiles, Storage Scaffolding

> *Three small packages close out the foundation tier.  Each
> provides a typed vocabulary for one domain — credentials,
> data-source profiles, database connections — that higher
> tiers consume without re-implementing.*

This is the last chapter of Volume 0.  It surveys the three
remaining foundation-tier packages whose surface area is small
enough that survey treatment fits, but whose contribution to the
portfolio is not small at all.  Each follows the discipline
the previous chapters established:

- A typed vocabulary class (or class hierarchy) per domain.
- A manager class that loads vocabulary instances from
  configuration data.
- A `Singleton` accessor that other packages call.

The chapter walks each package in turn, then closes Volume 0.

## `codecraftai-credentials` — the typed credential vocabulary

[`codecraftai-credentials`](https://github.com/code-craft-ai-company/codecraftai-credentials/tree/main/source/packages/ccai/credentials)
provides the typed credential vocabulary every other package
consumes.  Volume 3 chapter 9 covered the eleven shipped
credential classes in extensive detail; this chapter is the
*foundation-tier view* that names what's in the package without
the orchestration-discipline framing.

### The shipped classes

Eleven concrete credential classes inherit from
[`BaseCredential`](https://github.com/code-craft-ai-company/codecraftai-credentials/blob/main/source/packages/ccai/credentials/basecredential.py):

| Class | Category | Used for |
|---|---|---|
| `BasicCredential` | basic / rest-basic | Username + password |
| `SshCredential` | ssh | SSH with password / keyfile / agent |
| `ApiTokenCredential` | api-token | Single opaque token |
| `PersonalApiTokenCredential` | personal-api-token | Token + username |
| `SecretTokenCredential` | secret-token | Token + service endpoint |
| `AwsAccessKeyCredential` | aws-access-key | AWS keyid + secret |
| `AzureClientSecretCredential` | azure-client-secret | Azure tenant + client + secret |
| `PublicKeyCredential` | public-key | Generic public/private keypair |
| `GithubTokenCredential` | github-token | GitHub PAT + username |
| `GithubAppCredential` | github-app | GitHub App keyfiles |
| `WifiChoiceCredential` | wifi-choice | List of SSID + password pairs |

Each is a *typed container* — fields named, accessed through
typed properties, no behaviour beyond storage.

### `BaseCredential` — the three universal fields

```python
class BaseCredential:

    def __init__(self, *, identifier: str, categories: List[str],
                 role: Optional[str]="priv"):

        if len(identifier) == 0:
            raise ConfigurationError("...identifier required.")
        if len(categories) == 0:
            raise ConfigurationError("...categories required.")

        self._identifier = identifier
        self._categories = categories
        self._role = role
```

Three universal fields — `identifier`, `categories`, `role` —
with `BaseException`-class validation at construction time
(per chapter 3's discipline).

### `CredentialManager` — the registry

[`CredentialManager`](https://github.com/code-craft-ai-company/codecraftai-credentials/blob/main/source/packages/ccai/credentials/credentialmanager.py)
takes a parsed dict (typically from
`codecraftai-configuration`'s loader) and produces an
identifier-keyed dictionary of typed instances.  Its
dispatch — chapter 19 of Volume 3 walked this in detail — is
the category-as-discriminator pattern:

```python
if category == "basic" or category == "rest-basic":
    credobj = BasicCredential(**credential)
elif category == "ssh":
    credobj = SshCredential(**credential)
elif category == "api-token":
    credobj = ApiTokenCredential(**credential)
# ... eleven branches ...
```

The shipped manager has no resolution layer — secrets are
treated as opaque strings.  Volume 3 chapter 10 develops the
resolver layer the consumer builds on top.

### `CredentialManagerSingleton` — the per-process accessor

[`wellknown.py`](https://github.com/code-craft-ai-company/codecraftai-credentials/blob/main/source/packages/ccai/credentials/wellknown.py)
provides the per-process singleton.  Consumers call
`CredentialManagerSingleton().lookup_credential(name)` and get
the typed credential object.

The singleton's first-call signature requires the parsed
credential info and the source URIs:

```python
def CredentialManagerSingleton(credential_info: Optional[dict] = None,
                               credential_uris: Optional[List[str]] = None
                               ) -> CredentialManager:
    global CREDENTIAL_MANAGER_SINGLETON

    if CREDENTIAL_MANAGER_SINGLETON is None:
        if credential_info is None or credential_uris is None:
            errmsg = "The first call ... must include the credential_info and credential_uris parameters."
            raise SemanticError(errmsg)
        # ... construct the singleton ...

    return CREDENTIAL_MANAGER_SINGLETON
```

The same two-phase pattern from chapter 6 — the *first* call
configures; subsequent calls retrieve.  A subsequent call
without configuration raises `SemanticError` so the failure is
loud.

### Cross-references

The credential package is consumed directly by:

- **`codecraftai-dataprofiles`** (next section) — every data
  profile carries a credential identifier.
- **`codecraftai-storage-database`** (next section) — database
  connections take credentials.
- **`codecraftai-configuration`** — the loader's credentials
  category produces credential objects through the manager.

Volume 3 Part III (chapters 9–12) is the orchestration-layer
treatment.  This chapter is the foundation-tier baseline the
orchestration treatment builds on.

## `codecraftai-dataprofiles` — typed data-source profiles

[`codecraftai-dataprofiles`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/tree/main/source/packages/ccai/dataprofiles)
provides the typed data-source-profile vocabulary.  Volume 3
chapter 8 covered the five shipped profile classes; this
chapter is the foundation-tier baseline.

### The shipped classes

Five concrete profile classes inherit from
[`BaseDataProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/basedataprofile.py):

| Class | Category | Used for |
|---|---|---|
| `DatabaseBasicSocketProfile` | basic-socket | Unix-socket databases |
| `DatabaseBasicTcpProfile` | basic-tcp | TCP-connected databases |
| `CouchDbProfile` | couchdb | CouchDB instances |
| `MongoDBAtlasProfile` | mongodb-atlas | MongoDB Atlas (template form) |
| `SnowflakeProfile` | snowflake | Snowflake databases |

### `BaseDataProfile` — the three universal fields

```python
class BaseDataProfile:

    def __init__(self, identifier: str, *, category: str,
                 credential: Optional[str] = None):

        self._identifier = identifier
        self._category = category
        self._credential = credential
```

Identifier, category, optional credential reference.  Like
`BaseCredential`, the universal fields are minimal; the
category-specific fields live on the concrete subclasses.

### `DataProfileManager` — the registry

[`DataProfileManager`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/dataprofilemanager.py)
mirrors `CredentialManager` — same dispatch pattern, different
domain.  Five dispatch branches; lookup-by-identifier; failure-
loudly on missing entries.

### `DataProfileManagerSingleton`

Same singleton pattern as `CredentialManagerSingleton`.  The
first call configures; subsequent calls retrieve; `SemanticError`
on unconfigured access.

### Cross-references

Data profiles compose with credentials.  A `CouchDbProfile`'s
`connection_string(credobj)` method takes an
`IBasicCredential` (chapter 4's interface) and produces the
connection URL with username/password substituted in.  The
profile carries the *where*; the credential carries the *who*;
the combination produces a connection.

Volume 3 chapter 8 develops the orchestration-layer view.

## `codecraftai-storage-database` — database connection scaffolding

[`codecraftai-storage-database`](https://github.com/code-craft-ai-company/codecraftai-storage-database/tree/main/source/packages/ccai/storage/database)
is the smallest and youngest of the foundation-tier packages.
Its job: take a data profile + a credential and produce a
SQLAlchemy `Engine` ready for queries.

### `DatabaseConnectionFactory` — the factory

[`dbconnection.py`](https://github.com/code-craft-ai-company/codecraftai-storage-database/blob/main/source/packages/ccai/storage/database/dbconnection.py)
defines `DatabaseConnectionFactory`:

```python
class DatabaseConnectionFactory:

    def __init__(self, profile_name: str, *,
                 dbprofile: DatabaseBasicSocketProfile,
                 credential: BaseCredential):
        self._profile_name = profile_name
        self._dbprofile = dbprofile
        self._credential = credential

    def create_engine(self, *, dbname=None, echo=True) -> SqlEngine:
        # subclass implements
        ...

    def format_connection_string(self, dbname=None) -> str:
        # subclass implements
        ...
```

A factory takes a profile + credential and exposes:

- `create_engine(dbname=None, echo=True)` — produces a
  configured SQLAlchemy `Engine`.
- `format_connection_string(dbname=None)` — produces the
  connection-string form of the profile + credential, used by
  the chapter-9 cross-reference (CouchDB profiles' connection
  strings).

The base class has *placeholder* implementations; the
real implementations live in per-engine subclasses that come
in higher tiers (or are shipped as separate packages — RHEL,
Ubuntu, Postgres-specific extensions).

### `dbio.py` — typed CRUD helpers

[`dbio.py`](https://github.com/code-craft-ai-company/codecraftai-storage-database/blob/main/source/packages/ccai/storage/database/dbio.py)
provides the operational helpers built on top of the factory:

- `database_exists(conn_factory, dbname=None)` — check whether
  a database exists.  Wraps `sqlalchemy_utils.database_exists`.
- `database_create(conn_factory, dbname=None, dbmetadata=None)`
  — create a database.  Wraps
  `sqlalchemy_utils.create_database`.
- `database_drop(conn_factory, dbname=None)` — drop a
  database.

The pattern: a small set of typed helpers around SQLAlchemy
primitives.  No reinvention; just the typed envelope that
turns the SQLAlchemy primitives into codecraftai-conventional
calls.

### `mixins.py` — `SerializableMixIn`

[`mixins.py`](https://github.com/code-craft-ai-company/codecraftai-storage-database/blob/main/source/packages/ccai/storage/database/mixins.py)
ships one mixin: `SerializableMixIn`, which adds `as_dict()`
and `to_json()` to SQLAlchemy model classes:

```python
class SerializableMixIn:

    def as_dict(self):
        dval = {}
        model = type(self)
        mapper = inspect(model)
        for col in mapper.attrs:
            col_key = col.key
            dval[col_key] = str(getattr(self, col_key))
        return dval

    def to_json(self, indent=4):
        # ... uses as_dict(), wraps with json.dumps ...
```

A model that mixes in `SerializableMixIn` becomes
`ISerializable`-compatible (chapter 4's interface) — the
chapter-4 protocol describes `as_dict` and `from_dict`; the
mixin provides one half (the write side); SQLAlchemy's row
construction provides the other.

### What the package is *not*

`codecraftai-storage-database` is not a complete storage layer.
It does not provide:

- An ORM beyond SQLAlchemy's defaults.
- Migrations (use Alembic).
- Connection pooling beyond SQLAlchemy's defaults.
- Schema generation.
- Multi-database transactions.

The package is *scaffolding*.  It connects the
data-profile vocabulary to SQLAlchemy in a typed,
discipline-conformant way.  When a project needs more storage
machinery, it builds on top of this package; the package
itself stays small.

### Cross-references

Used by tiers that store structured data:

- **Volume 1 (Testing)** — testbase's result-storage layer
  uses the connection factory for the JSOS results database.
- **Volume 6 (Topology / Provisioning)** — landscape state
  storage uses the connection factory.
- **Volume 8 (Time-Travel)** — server-side ingestion uses the
  connection factory for trace event storage.

The pattern is consistent across consumers: the project ships
its SQLAlchemy models with `SerializableMixIn`; the project
configures a `DataProfile` for the database; the project
constructs a `DatabaseConnectionFactory`; the project does its
work.

## What the three packages share

The three packages this chapter covers — credentials,
dataprofiles, storage-database — share four structural
features:

### A typed vocabulary class hierarchy

Each package has a `BaseX` abstract class plus concrete
subclasses keyed by *category*.  `BaseCredential` /
`BaseDataProfile` carry the universal fields; subclasses
extend with category-specific fields.

### A manager class that dispatches by category

`CredentialManager`, `DataProfileManager` (storage-database
delegates dispatch to SQLAlchemy through the factory).  Each
takes a parsed dict and produces typed instances.

### A `Singleton` accessor

`CredentialManagerSingleton`, `DataProfileManagerSingleton`.
Per-process; first-call configures; subsequent retrieves.

### A `category` discriminator

The dict format always carries a `category` field naming the
concrete subclass to instantiate.  This is the
chapter-3 (Volume 3) category-as-discriminator pattern applied
at the foundation tier.

The pattern is the chapter-6 *typed-descriptor* model applied
to vocabulary objects rather than runtime variables.  The same
discipline; different domain.

## The complete foundation-tier picture

Volume 0 has now walked the full foundation tier:

| Package | Chapter | Role |
|---|---|---|
| (philosophy) | 1 | The ten principles |
| (coding standards) | 2 | The rules |
| `codecraftai-foundations` (`ccai.errors`) | 3 | The exception vocabulary |
| `codecraftai-foundations` (`ccai.interfaces`) | 4 | The contract layer |
| `codecraftai-foundations` (`ccai.waiting`) | 5 | The wait-with-diagnostics primitive |
| `codecraftai-this-app` | 6 | Identity + persona variables |
| `codecraftai-extension` | 7 | Factory and plug-in machinery |
| `codecraftai-runtime` | 8 | Runtime variables + paths + activation |
| `codecraftai-configuration` | 9 | The configuration loader |
| `codecraftai-credentials`, `-dataprofiles`, `-storage-database` | 10 | Typed vocabularies |

Plus the cross-cutting `codecraftai-xmodules` package (chapter 9
mentioned its `MergeMap`; chapter 7 touched its
`ximport.import_by_name`).  `xmodules` is the catch-all utility
package — `xformatting.indent_lines`, `xpython.expand_path`,
`xcollections.MergeMap`, etc.  It's consumed everywhere; it's
not itself a chapter target because its contents are utilities,
not architecture.

A reader who finishes Volume 0 understands:

- The **philosophy** every package is built around.
- The **standards** that turn the philosophy into rules.
- The **error vocabulary** that makes errors-never-silent
  operational.
- The **interfaces** that let packages substitute for each
  other.
- The **waiting primitive** that turns `wait_for_it` from
  bug-prone to bug-free.
- The **identity machinery** that gives every running process
  a structured persona.
- The **factory machinery** that lets packages plug in
  without modifying consumers.
- The **runtime layer** that gives every running process its
  operational identity.
- The **configuration loader** that produces typed
  configurations from multi-source files.
- The **typed vocabulary classes** for credentials, data
  profiles, and database connections.

Every higher volume in the series builds on these foundations.
Volume 1 is testing; it uses the runtime layer for run identity
and the configuration loader for runtime profiles.  Volume 2
is distributed automation; it uses the interfaces for
substitutability and the factory machinery for cross-host
dispatch.  Volume 3 is integration orchestration; it uses
*every* foundation chapter at some point.  Volume 4 is sysman;
it uses the factory machinery for cross-OS dispatch.  Volume 5
is interop; it builds on the Aspects pattern previewed in
chapter 1 and chapter 2.  Volume 6 is landscape; it uses the
typed-vocabulary discipline for device entries and the
configuration loader for landscape files.  Volume 7 is the AI
agent SDK; it uses the testbase substrate from Volume 1.
Volume 8 is time-travel debugging; it uses the storage-database
scaffolding for trace event ingestion.

Every volume uses *some* of Volume 0; Volume 0 is the *common
language* the rest of the series speaks.

## What Volume 0 deliberately doesn't cover

Three areas of the portfolio that aren't in the foundation tier:

### Networking primitives

`codecraftai-networking` ships TCP/IP utility code.  It's a
*support* package rather than a foundation package — it
doesn't define a typed vocabulary or an architecture; it's a
set of helpers.  Volume 5 (Interop) treats it where
appropriate.

### Logging configuration

The detailed logging configuration — how the runtime layer
configures Python's logging system, what handlers and
formatters are wired up, how the log files are named — lives
in `codecraftai-xmodules/xlogging`.  It's referenced through this
volume but not treated as its own chapter; the depth doesn't
warrant it.

### Test framework

`codecraftai-testbase` is foundation-tier *adjacent* — most other
volumes depend on it — but it's substantial enough to deserve
its own volume (Volume 1).  Treating it in Volume 0 would
shortchange both the testing material and the foundation
material.

## Summary — and end of Volume 0

- **`codecraftai-credentials`** ships eleven typed credential
  classes inheriting from `BaseCredential`, plus a
  `CredentialManager` registry and a `CredentialManagerSingleton`
  accessor.  Volume 3 chapter 9 develops the orchestration-
  layer view.
- **`codecraftai-dataprofiles`** ships five typed data-source-
  profile classes inheriting from `BaseDataProfile`, plus a
  `DataProfileManager` registry and a singleton accessor.
  Volume 3 chapter 8 develops the orchestration-layer view.
- **`codecraftai-storage-database`** ships the
  `DatabaseConnectionFactory` that takes a profile + credential
  and produces a SQLAlchemy `Engine`, plus a small set of
  typed CRUD helpers (`database_exists`, `database_create`,
  `database_drop`) and the `SerializableMixIn` for SQLAlchemy
  model classes.
- The three packages share **four structural features**: typed
  vocabulary class hierarchies, manager classes that dispatch
  by category, per-process singleton accessors, and the
  category-as-discriminator pattern.
- The volume's complete picture: ten chapters, ten foundation-
  tier packages walked.  Every higher volume in the series
  builds on these chapters.
- Volume 0 deliberately doesn't cover networking primitives
  (support package, not architecture), logging configuration
  (utility code in `codecraftai-xmodules`), or the test framework
  (Volume 1's territory).

A reader who finishes Volume 0 has the **discipline** of the
portfolio.  Every later volume turns that discipline into a
specific working system: a test framework, a distributed
test infrastructure, an orchestration layer, a cross-OS sysman
tier, an interop layer, an environment-modelling layer, an
agent SDK, a time-travel debugger.  The discipline is the
common language; the volumes are dialects.

The series's reading order is `0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 →
8 → Appendix`.  Volume 0 is the prerequisite for everything
that follows.  Whatever volume the reader picks up next, the
foundation is now in place.

---

This concludes Volume 0 — *Foundations of Automation Engineering
with Python*.
