# Chapter 8 — Data Profiles

> *A connection string in environment-variable form is a hostname, a
> port, a username, a password, a database name, and three optional
> flags concatenated into a string nobody can parse reliably.  A data
> profile is the same six items as named, typed fields.*

This chapter introduces the fifth and final category in Part II:
**Data Profiles** — the configuration that names where data lives,
how to reach it, and which credential to use against it.  After this
chapter Part II is complete; Part III then digs into Credentials in
the dedicated detail they deserve.

## The connection-string disease

Every long-lived automation pipeline has, at some point, contained
a line like this:

```bash
export DATABASE_URL="postgresql://app_user:Tr0ub4dor3@db-staging-east-1.internal:5432/app?sslmode=require&pool_size=20"
```

This is the connection string as configuration.  It is also the
connection string as bug-magnet.  Six failure modes follow predictably:

1. **Credential leak.**  The password is embedded in the variable.
   It will appear in `env` dumps, in CI logs, in error tracebacks, in
   the developer's shell history.  Masking helps but never completely.
2. **Untyped concatenation.**  Hostname, port, database, query
   parameters all concatenated.  No schema; one missing slash and
   the whole string parses wrong.
3. **Single source of truth, scattered.**  Three tests, two scripts,
   and one CI step each parse the URL with their own regex.  When
   the URL changes shape — Snowflake adds a `warehouse` segment,
   Mongo Atlas changes the `srv` form — every regex has to be
   updated.
4. **Mixed transport semantics.**  TCP databases, Unix-socket
   databases, document stores, and SaaS endpoints all share the
   "URL string" format with subtly different conventions.  Code
   that worked against TCP Postgres breaks against socket-only Postgres.
5. **No environment portability.**  The local-dev DATABASE_URL is
   `postgresql://localhost/dev`; the staging DATABASE_URL is the
   string above; the prod DATABASE_URL is something else.  Three
   different strings, three different failure modes when the wrong
   one leaks into the wrong run.
6. **Credentials and connection details fused.**  When the password
   rotates, you change the URL.  When the host moves, you also
   change the URL.  Two unrelated lifecycles share one variable;
   each rotation requires touching the other.

The discipline of this chapter replaces the connection string with a
**Data Profile** — a typed object naming each component as a
separate field, with the credential reference stored *separately*
from the credential material.

## What a Data Profile is

A Data Profile is a named, typed description of a data source.  It
has three universal fields and a category-specific extension:

```yaml
- identifier: staging-db-readwrite
  category: basic-tcp
  host: db-staging-east-1.internal
  port: 5432
  dbtype: postgres
  dbname: app
  credential: staging-db-rw
```

The three universals come from
[`BaseDataProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/basedataprofile.py):

| Field | Purpose |
|---|---|
| `identifier` | The name the orchestration uses to refer to this profile |
| `category` | The category discriminator that selects the concrete profile class |
| `credential` | A reference to a credential identifier (chapters 9–12) — never the credential value |

Everything else depends on the category.  Five categories ship in
[`codecraftai-dataprofiles`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/tree/main/source/packages/ccai/dataprofiles);
each is implemented as a concrete class inheriting from
`BaseDataProfile` with its own additional fields and its own
`validate` classmethod.

## The five shipped categories

### `basic-tcp` — TCP-connected databases

For databases reached over TCP with host and port, regardless of
engine.  The shipped class is
[`DatabaseBasicTcpProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/databasebasictcpprofile.py).

```yaml
- identifier: staging-db-readwrite
  category: basic-tcp
  host: db-staging-east-1.internal
  port: 5432
  dbtype: postgres
  dbname: app
  credential: staging-db-rw
```

Required fields: `credential`, `dbtype`, `host`, `port`.  Optional
fields: `dbname`.  The `dbtype` field carries the engine type
(`postgres`, `mysql`, `mariadb`, `redis`, etc.) so consuming code can
select the right driver.

### `basic-socket` — Unix-socket databases

For databases reached over a Unix domain socket — same engine
catalogue as `basic-tcp` but with a filesystem path instead of host
and port.  The shipped class is
[`DatabaseBasicSocketProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/databasebasicsocketprofile.py).

```yaml
- identifier: local-postgres
  category: basic-socket
  socket: /var/run/postgresql/.s.PGSQL.5432
  dbtype: postgres
  dbname: app_dev
  credential: local-postgres-dev
```

Required fields: `credential`, `dbtype`, `dbname`, `socket`.

The split between `basic-tcp` and `basic-socket` is deliberate.  A
single "database profile" category that accepted either a host/port
or a socket would force consuming code to branch on which fields
were present.  Two categories make the dispatch a single
`category: ...` lookup.

### `couchdb` — CouchDB

For CouchDB instances.  The shipped class is
[`CouchDbProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/couchdbprofile.py).

```yaml
- identifier: staging-couchdb
  category: couchdb
  host: couchdb.staging.internal
  port: 5984
  scheme: https
  dbname: configurations
  credential: staging-couchdb-rw
```

Required fields: `credential`, `dbname`, `host`, `port`.  Optional:
`scheme` (defaults to `http`).  CouchDB has its own profile category
because its connection-string format is non-standard (the database
name lives in the path, the auth lives in the userinfo) and because
the
[`codecraftai-configuration`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/sources/couchdbsource.py)
loader supports CouchDB as a *configuration source* — the same
profile shape doubles for "database we read from" and "config store
we read configuration from."

### `mongodb-atlas` — MongoDB Atlas

For MongoDB Atlas deployments.  The shipped class is
[`MongoDBAtlasProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/mongodbatlasprofile.py).

```yaml
- identifier: staging-atlas
  category: mongodb-atlas
  connection: "mongodb+srv://<username>:<password>@cluster0.abcde.mongodb.net/?retryWrites=true"
  credential: staging-atlas-rw
```

Required fields: `credential`, `connection`.  The `connection` field
is the Atlas connection *template* — the literal string MongoDB Atlas
gives you in its console — with `<username>` and `<password>`
placeholders.  At runtime the credential is materialised and the
template is expanded by
[`MongoDBAtlasProfile.connection_string(credobj)`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/mongodbatlasprofile.py).

This category exists separately from `basic-tcp` because Atlas's
`mongodb+srv://` scheme expands to a list of hosts via DNS at
runtime — there is no fixed host/port to put in a profile.  The
template-with-placeholders form is the lowest-fidelity capture of
what Atlas actually requires.

### `snowflake` — Snowflake

For Snowflake databases.  The shipped class is
[`SnowflakeProfile`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/snowflakeprofile.py).

```yaml
- identifier: staging-snowflake
  category: snowflake
  account: my-org-staging
  warehouse: COMPUTE_WH_STAGING
  database: APP_STAGING
  schema: PUBLIC
  parameters:
    role: ANALYST
    timezone: UTC
  credential: staging-snowflake-rw
```

Required fields: `account`, `warehouse`, `database`, `schema`,
`credential`.  Optional: `parameters` (a dictionary of additional
connection options that pass through to the Snowflake connector).

Snowflake gets its own category because the connection model is
genuinely different from TCP databases — `account`, `warehouse`,
`database`, `schema` are independent dimensions you select at
connection time, and the same Snowflake account hosts many of each.

## A complete worked example

Building on the staging-east Landscape from chapter 5 (which had a
`staging-db` service and a `staging-api` service), here is the
matching Data Profile file:

```yaml
# dataprofiles-staging-east.yaml
dataprofiles:

    # Primary application database — the one staging-api reads/writes
    - identifier: staging-db-app-rw
      category: basic-tcp
      host: db-staging-east-1.internal
      port: 5432
      dbtype: postgres
      dbname: app
      credential: staging-db-rw

    # Same database, different credential — read-only for analytics
    - identifier: staging-db-app-readonly
      category: basic-tcp
      host: db-staging-east-1.internal
      port: 5432
      dbtype: postgres
      dbname: app
      credential: staging-db-readonly

    # Redis cache the API uses for sessions
    - identifier: staging-redis
      category: basic-tcp
      host: redis-staging-east-1.internal
      port: 6379
      dbtype: redis
      credential: staging-redis-default

    # Snowflake warehouse for analytical queries
    - identifier: staging-warehouse
      category: snowflake
      account: my-org-staging
      warehouse: COMPUTE_WH_STAGING
      database: APP_STAGING
      schema: PUBLIC
      credential: staging-snowflake-rw

    # CouchDB used for storing test artifacts
    - identifier: staging-test-artifacts
      category: couchdb
      host: couchdb.staging.internal
      port: 5984
      scheme: https
      dbname: test-artifacts
      credential: staging-couchdb-rw
```

Five profiles, five categories represented.  Notice three things:

- **The same physical database appears twice** (`staging-db-app-rw`
  and `staging-db-app-readonly`) with different credentials.  The
  Data Profile is the *access intention*, not just the location.
- **Every credential is a reference**, never an inline value.  The
  five credential identifiers
  (`staging-db-rw`, `staging-db-readonly`, `staging-redis-default`,
  `staging-snowflake-rw`, `staging-couchdb-rw`) are looked up by
  the resolution layer at runtime.
- **No connection strings appear anywhere.**  The components are
  named fields; the connection string — when one is needed — is
  built at runtime by the profile class's `connection_string()`
  method.

The orchestration consumes profiles by name:

```python
from ccai.dataprofiles.wellknown import DataProfileManagerSingleton

profile = DataProfileManagerSingleton().lookup_profile(
    "staging-db-app-rw"
)

# profile is a DatabaseBasicTcpProfile instance with
# host/port/dbname/dbtype/credential typed fields

cred = CredentialManagerSingleton().lookup_credential(profile.credential)
conn_str = f"postgresql://{cred.username}:{cred.password}@{profile.host}:{profile.port}/{profile.dbname}"
```

Or, for the categories that ship a `connection_string()` helper:

```python
profile = DataProfileManagerSingleton().lookup_profile("staging-couchdb")
cred = CredentialManagerSingleton().lookup_credential(profile.credential)
conn_str = profile.connection_string(cred)
```

The connection string is built at the point of use, with the
materialised credential, and discarded immediately.  It never
becomes a process-environment value.

## The `category` field as discriminator

Every Data Profile has a `category` field that selects which
concrete class to instantiate when the file is loaded.  The
[`DataProfileManager`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/dataprofilemanager.py)
performs this dispatch:

```python
if category == DatabaseBasicSocketProfile.category:    # "basic-socket"
    profobj = DatabaseBasicSocketProfile(**profile)
elif category == DatabaseBasicTcpProfile.category:     # "basic-tcp"
    profobj = DatabaseBasicTcpProfile(**profile)
elif category == CouchDbProfile.category:              # "couchdb"
    profobj = CouchDbProfile(**profile)
elif category == MongoDBAtlasProfile.category:         # "mongodb-atlas"
    profobj = MongoDBAtlasProfile(**profile)
elif category == SnowflakeProfile.category:            # "snowflake"
    profobj = SnowflakeProfile(**profile)
else:
    logger.warning(f"Unknown category '{category}' ...")
```

This is the standard ccai discriminator pattern: a string field
on the data, a class attribute on the implementations, a dispatch
that matches them at load time.  The pattern shows up in
`codecraftai-credentials` (chapter 9), in
`codecraftai-provisioning` (chapter 4 — `category` on
`CloudDevice`), and elsewhere.  Adding a new Data Profile category —
say, `redis-sentinel` for Redis Sentinel deployments where the
single-host model breaks down — means adding a new class and adding
a new branch in the dispatch.

The `validate` classmethod on each concrete class enforces the
category-specific required fields at load time.  A profile of
category `basic-tcp` missing the `port` field fails to load with a
clear error message naming the missing field; the orchestration
never starts with a malformed profile.

## Composition with credentials and landscape

The Data Profile sits at a specific composition point: it bridges
*where data lives* (the landscape) and *who's allowed to access it*
(credentials).

```
       Landscape
       ────────
       host: db-staging-east-1.internal       ◄─── inventory truth
       (the host is in the infrastructure)


       Data Profile                            ◄─── access intention
       ────────────
       identifier: staging-db-app-rw
       host: db-staging-east-1.internal
       port: 5432
       credential: staging-db-rw   ◄─────────  reference


       Credential                              ◄─── access material
       ──────────
       identifier: staging-db-rw
       type: BasicCredential
       username: app_user
       password: <looked up at runtime>
```

Three deliberate boundaries:

1. **The Data Profile points at hosts named in the Landscape.**  The
   profile's `host` field is the same hostname the Landscape's
   infrastructure section names (`db-staging-east-1.internal`).
   When the host moves, the Landscape's infrastructure entry
   changes — and the Data Profile's `host` field changes alongside.
   The two changes are kept in sync because the same person
   editing the same environment's configuration touches both.

   The Data Profile does *not* resolve the host through the
   Landscape automatically.  This is a deliberate design choice
   from `codecraftai-dataprofiles`: the profile is self-contained
   data; the Landscape is queried separately.  The result is
   that a Data Profile is a *complete* description of how to reach
   a data source, even if the corresponding Landscape isn't loaded.

2. **The Data Profile names a credential identifier.**  It does not
   contain the credential.  The resolution layer (chapter 10)
   produces the credential at the moment of use.  The credential's
   lifecycle is independent of the profile's.

3. **The Landscape's role and the Data Profile's role are
   different.**  The Landscape says "there is a service called
   `staging-db` reachable at `db-staging-east-1.internal:5432`."
   The Data Profile says "to access the `app` database on that
   host, use the `staging-db-rw` credential."  Both can exist; both
   are necessary; they describe orthogonal aspects.

A common confusion: why does the Data Profile have a `host` field if
the Landscape already has it?  The answer is that the Data Profile
is for data sources that may *not* be in the Landscape at all.  A
SaaS data store like Snowflake or MongoDB Atlas isn't in the
infrastructure section of any Landscape — it's a third-party service
the orchestration calls into.  The Data Profile is the *only* place
where its access details live.

For internal data stores that *are* in the Landscape, the
duplication is the cost of keeping the Data Profile self-contained.
A future evolution could allow Data Profile fields to reference
Landscape entries by name (`host: ${landscape:staging-db.host}`), at
the cost of making profiles dependent on a particular Landscape.
The current shipped form keeps the abstraction simple at the price
of one duplicated hostname.

## The `DataProfileManager` and lookup-by-name

At runtime the orchestration interacts with profiles through the
[`DataProfileManagerSingleton`](https://github.com/code-craft-ai-company/codecraftai-dataprofiles/blob/main/source/packages/ccai/dataprofiles/wellknown.py):

```python
from ccai.dataprofiles.wellknown import DataProfileManagerSingleton

manager = DataProfileManagerSingleton()
profile = manager.lookup_profile("staging-db-app-rw")
```

Three behaviours of the manager worth noting:

- **Lookup by identifier returns the typed object.**  The orchestration
  receives a `DatabaseBasicTcpProfile` (or whichever class matches
  the category), not a generic dictionary.  Type-checked attribute
  access (`profile.host`, `profile.port`) replaces dictionary
  fetches with the chance of typos.
- **Missing identifier raises `ConfigurationError`.**  The error
  message includes the list of source URIs the manager loaded from,
  so the engineer can immediately see where to add the profile.  No
  silent `None` returns; loading and lookup both fail loudly.
- **Validation runs at load time.**  Errors and warnings are
  collected from each profile's `validate` classmethod; if any
  errors exist, the load fails before any profile is instantiated.
  The manager never holds a partially-valid profile set.

## What a Data Profile is *not*

Three things to keep out, every one of which a team will be
tempted to put in:

### Not a credential

The most common mistake.  A Data Profile names a credential by
identifier.  It does not embed username, password, token, key, or
any secret material.  When you find yourself adding a `password`
field to a profile, stop — the password belongs in the credential
record, not the profile.

A symptom of getting this wrong is that profile rotation requires
editing the profile.  Done right, password rotation is invisible
to the profile — the credential record changes, the profile is
unchanged, the next connection picks up the new password
automatically.

### Not a Landscape entry

A Data Profile points at data sources; a Landscape lists hosts and
services.  The two have similar shapes but different purposes.
Putting a Data Profile in a Landscape ("infrastructure: services:
... profile fields ...") confuses the inventory with access details.
Putting Landscape entries in a Data Profile file ("hosts: ... ports:
...") loses the apod-vs-infrastructure distinction.

The test of "does this belong in a Data Profile?" is: *is this
about how to reach a data source?*  If yes, profile.  If no, it
belongs elsewhere.

### Not a runtime configuration

A Data Profile carries connection parameters that are intrinsic to
the data source — host, port, account, database name.  It does
*not* carry behaviour parameters like connection-pool size,
statement timeout, or retry count, even though those are commonly
passed to the same connection-builder code.  Behaviour parameters
go in the runtime configuration (chapter 7); the Data Profile is
the *what to connect to*, not the *how to behave when connected*.

The test: *would two different orchestrations against the same
data source need different values of this field?*  If yes, runtime.
If no, profile.

## Migration: from connection strings to Data Profiles

A pipeline that has accumulated connection strings as environment
variables migrates to Data Profiles in three steps:

1. **Enumerate every connection string.**  Walk the codebase, the
   Dockerfiles, the docker-compose files, the CI YAML, and the
   developer's `.env` files.  List every variable that looks like
   a URL pointing at a data source.  Group them by destination —
   `DATABASE_URL`, `STAGING_DB_URL`, `PROD_DB_RO_URL`, etc.

2. **Decompose each string into typed fields.**  For each URL,
   identify the host, the port, the database, the database type,
   the credential.  Each becomes a field in a Data Profile entry.

3. **Replace the URL parsing in code with profile lookups.**  Code
   that did:
   ```python
   url = os.environ["DATABASE_URL"]
   conn = sqlalchemy.create_engine(url)
   ```
   becomes:
   ```python
   profile = DataProfileManagerSingleton().lookup_profile("staging-db-app-rw")
   cred = CredentialManagerSingleton().lookup_credential(profile.credential)
   conn = sqlalchemy.create_engine(
       f"postgresql://{cred.username}:{cred.password}@{profile.host}:{profile.port}/{profile.dbname}"
   )
   ```
   The application code now reads a typed profile and a typed
   credential, builds the connection string at the point of use,
   and never holds the password in a long-lived variable.

After migration, the `DATABASE_URL` and its kin are gone.  The
plumbing tax from chapter 7 has been paid once for `AM_DATAPROFILE_NAMES`;
every new data source after that is a one-file change to the
profile YAML.

## Where Data Profiles don't fit

Honest scoping notes:

- **Truly ephemeral data sources.**  A pipeline that creates a
  one-time database, populates it, queries it, drops it — all in a
  single step — does not need a Data Profile.  The connection
  details are local to the step; promoting them to a profile adds
  ceremony without payoff.
- **Highly dynamic discovery.**  Pipelines that discover data
  sources at runtime through service discovery (Consul, etcd, K8s
  service mesh) typically generate the equivalent of a profile in
  memory.  The shipped `DataProfileManager` doesn't model dynamic
  registration; the pattern is to translate the discovery output
  into a profile dictionary at orchestration start and pass it to
  `load_datasource_profiles`.
- **In-memory or filesystem-only data.**  A pipeline whose "data"
  is a JSON file in the repository doesn't need a Data Profile.
  The file is in the codebase; reading it is direct.

The discipline scales from "two databases" to "dozens of services
across multiple categories"; below that scale the discipline is
optional.

## Summary — and end of Part II

- A **Data Profile** is a typed, named description of a data
  source: where it lives, how to reach it, which credential to use.
- The five shipped categories are `basic-tcp`, `basic-socket`,
  `couchdb`, `mongodb-atlas`, and `snowflake`.  Each is a concrete
  class inheriting from `BaseDataProfile` with category-specific
  fields and category-specific validation.
- Three universal fields — `identifier`, `category`, `credential` —
  are required on every profile.  The `credential` field is a
  *reference*; the credential material lives in the credential
  category and is resolved at runtime.
- The `DataProfileManager` loads profiles, validates them at load
  time, and exposes lookup-by-identifier through the
  `DataProfileManagerSingleton`.
- Profiles compose with the Landscape (the profile's host
  parallels the Landscape's infrastructure host but is
  self-contained) and with credentials (the profile names a
  credential; the resolution layer materialises it).
- A Data Profile is not a credential (no embedded secrets), not a
  Landscape entry (no inventory listing), and not a runtime
  configuration (no behaviour parameters).
- Migration from connection-string env vars is mechanical:
  enumerate, decompose into fields, replace URL parsing with
  profile lookups.

This is the last chapter of Part II.  The volume has now introduced
five of the six categories — Specifications, Landscapes, Topologies,
Runtimes, Data Profiles — with file formats, code anchors, and
worked examples grounded in real portfolio code.

The sixth category — **Credentials** — earns four chapters of its
own, which is the next part of the volume.  Part III digs into
credential vocabularies, the resolution discipline (the most
substantial design exercise in the volume since the portfolio ships
a vocabulary layer but no resolution backends), credentials in
flight (rotation, encryption at rest, audit trails), and credentials
across environments (the same vocabulary, different backends, per
the Part IV profile-and-environment story).
