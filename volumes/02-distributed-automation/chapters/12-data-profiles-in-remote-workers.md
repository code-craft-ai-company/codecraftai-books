# Chapter 12 — Data Profiles in Remote Workers

> *Volume 3 chapter 8 walks data profiles at the orchestration
> layer.  This chapter walks the analogue at the job layer:
> when a distributed test job needs to talk to a database, a
> message queue, or a cache, what does the framework do?  The
> answer is short — it follows the same pattern as credentials
> from chapter 11, applied to a different category.*

A *data profile* in the `codecraftai-*` portfolio is a
*connection profile* for a data store.  Where a credential
answers "how do I authenticate," a data profile answers "where
do I connect."  The two go together — most data-store calls
need both — but they're different categories with different
configuration files.

This chapter is short.  The discipline that holds for
credentials (chapter 11) holds for data profiles, with the
same shape and the same boundaries.  The brevity is the
payoff of universal patterns: when the same shape applies,
the documentation is the same shape too.

## Declared by name, resolved at activation, never a hardcoded URI

A data profile is declared in the Landscape's `services`
section (or, equivalently, referenced from a device or as a
top-level Landscape entity, depending on the project's
convention):

```yaml
infrastructure:
    services:
        - serviceType: database/postgres
          name: orders-db
          connection_profile: orders-db-profile
```

The `connection_profile: orders-db-profile` is a *named
reference* to a data profile.  The actual connection details
(host, port, database name, schema, connection-pool size) live
in the Data Profiles configuration category (Volume 3 chapter
8) — composable via `MergeMap` like every other category.

A representative data profile file:

```yaml
# data-profiles/orders-db-profile.yaml
name: orders-db-profile
type: database/postgres
host: db.staging-east.internal
port: 5432
database: orders
schema: public
pool:
    min_connections: 2
    max_connections: 10
    timeout_seconds: 30
```

The file has *no credentials*.  Authentication is a separate
concern (chapter 11); the data profile says only "where to
connect."  Credentials are referenced from the same service
declaration:

```yaml
- serviceType: database/postgres
  name: orders-db
  connection_profile: orders-db-profile
  credentials:
      - orders-db-app    # ← the role the test connects as
```

Two named references in the service declaration.  The
framework resolves both at activation; the test gets a
connection-ready service object without ever seeing host
strings or auth material in test code.

## The DataProfileManager — the parallel to CredentialManager

`DataProfileManager` (analogous to `CredentialManager` from
chapter 11; in `codecraftai-dataprofiles`) is the job's
data-profile lookup service.  Initialised during landscape
activation.  Holds the resolved profiles for the duration of
the job.

```python
manager = LandscapeSingleton().dataprofiles_manager
profile = manager.lookup_by_name("orders-db-profile")

# profile carries the connection details
db_host = profile.host
db_port = profile.port
```

In practice the test rarely calls `lookup_by_name` directly.
The framework wires the profile through the service's
ProtocolExtension:

```python
service = LandscapeSingleton().service("orders-db")

# The service has a 'db' ProtocolExtension wired by the
# database-Coordinator at activation time.
result = service.db.execute("SELECT count(*) FROM orders")
```

Same shape as `device.ssh.run_cmd(...)` from chapter 11 — the
ProtocolExtension is the test's surface; the underlying
machinery (connection pool, cursor, transaction) is hidden.

## Composable data profile lists

Data profiles are composable like every other category.  A
service can reference multiple profiles:

```yaml
- serviceType: database/postgres
  name: orders-db
  connection_profile:
      - orders-db-base
      - orders-db-staging-overrides
```

The framework merges them via `MergeMap` (later names override
earlier names where keys collide) and the service uses the
merged result.  Common patterns:

- **Base + environment overrides.**  A `base-postgres-profile`
  with sensible pool defaults; a `staging-postgres-profile`
  with staging-specific host and timeout adjustments.
- **Pool size variants.**  A `small-pool` for unit-test
  databases; a `large-pool` for integration runs.
- **Schema variants.**  A `read-only` profile pointing at a
  read replica; a `read-write` profile pointing at the primary.

The composability is what makes the four-environment ladder
from Volume 3 chapter 13 work.  Local-dev uses
`local-postgres + dev-overrides`; CI uses `local-postgres +
ci-overrides`; staging uses `local-postgres +
staging-overrides`.  Same suite of tests, four profiles,
three differences.

## What flows from the controller to the host

A connection profile leads to a *connection from the
controller to the data store*.  The flow:

- Controller resolves the profile via the
  `DataProfileManager`.
- Controller's data-store ProtocolExtension uses the profile's
  host / port / database / schema to construct a connection.
- Controller authenticates using the credential (chapter 11
  describes this side).
- Connection is established; queries run from the controller
  to the store.

The data profile's *content* is not sensitive (the same way
credentials are).  Hosts under test that need their own
connections to the data store typically do so through their
*own* configuration — the framework's job is to give the
*controller* a connection for *its* test code, not to push
connection profiles to the hosts.

If a test exercises a host's connection to a data store
(e.g., "does this app server connect to the database?"), the
test:

- Uses the controller's connection (via the data profile) to
  inspect the database state.
- Uses the host's API or admin interface to check the host's
  connection state.
- Compares.

The framework's data profiles are *the controller's*, not the
host's.  The host has its own configuration story, owned by
the project.

## Per-test isolation through profile selection

A common pattern: a test wants its *own* sandbox connection.
The framework supports this through profile-name selection
at the test level:

```python
@testbase.attach.parameter(orders_db_profile, identifier="db")
def test_can_query_orders(db):
    """
        Verify a query against the orders database.
    """
    rtnval = db.execute("SELECT count(*) FROM orders").rowcount
    assert rtnval >= 0
    return
```

The `orders_db_profile` here is the originator that yields a
profile-bound connection.  The Volume 1 chapter 3
resource-factory model applies: per-test scoping means each
test gets a fresh connection (with whatever transaction
isolation the database provides) and the connection is
released at test end.

A project that wants per-test isolation more strongly (e.g., a
fresh schema per test) writes a custom originator:

```python
@testbase.factories.resource_factory(scope=ResourceLifespan.Test)
@testbase.attach.parameter(orders_db_profile, identifier="profile")
def isolated_db_session(profile):
    """
        Yields a per-test isolated DB session with a fresh schema.
    """
    schema = f"test_schema_{uuid.uuid4().hex[:8]}"
    conn = profile.connect()
    conn.execute(f"CREATE SCHEMA {schema}")
    conn.execute(f"SET search_path TO {schema}")
    try:
        yield conn
    finally:
        conn.execute(f"DROP SCHEMA {schema} CASCADE")
        conn.close()
    return
```

The originator is per-test (lifespan Test), gets the profile
from the framework, creates a fresh schema, yields the
connection, drops the schema on cleanup.  The test sees a
clean isolated database.  The Volume 1 chapter 3 dependency
graph composes the `orders_db_profile` (yields a profile) →
`isolated_db_session` (yields a per-test connection) chain
correctly.

## Streaming-data profiles for queues and event buses

Data profiles aren't database-specific.  Message queues, event
buses, key-value stores, time-series databases all use the
same shape:

```yaml
# A Kafka topic profile
- serviceType: messaging/kafka
  name: orders-events
  connection_profile: orders-events-profile
  credentials:
      - kafka-consumer
```

```yaml
# data-profiles/orders-events-profile.yaml
name: orders-events-profile
type: messaging/kafka
brokers:
    - kafka-1.staging-east.internal:9092
    - kafka-2.staging-east.internal:9092
topic: orders.events.v2
consumer_group: ci-test-runner
auto_offset_reset: latest
```

The framework's discipline is uniform: declare by name,
resolve at activation, ProtocolExtension wires the connection,
credential authenticates.  The test uses
`service.kafka.consume(...)` exactly as it uses
`service.db.execute(...)`.

## What the framework does NOT do

A short list of things the framework deliberately stays out
of, leaving them to the test or to the data store's own
machinery:

- **Connection pool management at the data-store level.**  The
  framework gives the test a connection (or a pool, depending
  on the ProtocolExtension); how the data store handles the
  pool is the data store's concern.
- **Transaction boundaries.**  A test that wants explicit
  transaction control uses the connection's transaction API;
  the framework doesn't impose a per-test transaction
  envelope.
- **Schema migrations.**  Bringing the database to a known
  state is *test fixture* territory.  The framework can yield
  a connection; it doesn't run migrations.

These are project / domain concerns, not framework concerns.
The framework's job is to *connect*; the project's job is to
*use*.

## What the next chapter does

Chapter 13 walks result aggregation — the JSOS streams from N
workers across the Pod arriving at one coordinator, the
preservation of per-host timelines, the failure-mode where one
worker's stream stops mid-run.  Where Volume 1 chapter 5
walked single-host streaming, chapter 13 walks the
multi-host aggregation.
