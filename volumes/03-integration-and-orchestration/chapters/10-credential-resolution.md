# Chapter 10 — Credential Resolution

> *The `codecraftai-credentials` package gives you the vocabulary.  The
> package does not give you a way to fetch the secret value.  This
> chapter is about that gap and how to close it.*

The previous chapter introduced the eleven shipped credential
classes and showed worked examples in which secret values appeared
as `${VAULT:/secret/...}` references rather than literal strings.
That syntax was *not* part of the shipped package — it was a
preview.  This chapter is the discipline that makes the syntax
meaningful.

The chapter is the most substantial design exercise in the volume.
The reason is that the portfolio's deliberate decision to ship a
vocabulary layer with no resolution backends puts a real engineering
load on the reader: you will build the resolver yourself, and the
choices you make will shape your team's security posture for the
rest of the project's life.  The chapter makes those choices
concrete by proposing an architecture, the interfaces it exposes,
and a recipe for the most common backends.

## The gap to close

Recall the shape of the load path through the shipped package:

```
credentials.yaml ──► YAML parser ──► dict ──►
   CredentialManager.load_credentials(credential_info=dict) ──►
      validate() per credential ──►
      dispatch by category tag ──►
      construct typed credential object ──►
      register in self._credentials[identifier]
```

The
[`CredentialManager`](../../../codecraftai-credentials/source/packages/ccai/credentials/credentialmanager.py)
takes a *parsed dictionary* — `credential_info` — and treats every
string value as opaque.  When the YAML contains
`password: hunter2`, the resulting `BasicCredential.password` is
the literal string `"hunter2"`.  When the YAML contains
`password: ${VAULT:/secret/staging/db/rw}`, the resulting password
is the literal string `"${VAULT:/secret/staging/db/rw}"`.  The
package does not interpret the syntax; it merely stores the value.

This is the gap.  Between the literal `${VAULT:...}` reference and
a usable secret, *something* has to fetch the value from a real
secrets store.  That something is the resolver layer.  It is not
shipped.  It is the reader's responsibility.

Three reasons for the choice:

1. **Backends are environment-specific.**  A team using HashiCorp
   Vault has different needs from a team using AWS Secrets Manager,
   from a team using a static encrypted file.  Shipping one would
   privilege one environment over others; shipping ten would bloat
   the package with dependencies most users don't need.
2. **Resolution is the security boundary.**  The point at which a
   credential reference becomes a credential value is the point at
   which audit, rotation, and access control happen.  Forcing the
   reader to design that boundary deliberately produces a better
   security posture than handing them a black-box default.
3. **The vocabulary stays portable.**  A credentials YAML written
   against the eleven shipped types and the `${SCHEME:path}` syntax
   is the same in every environment.  The resolver swaps; the
   credentials file does not.

## The reference syntax

The proposed convention is `${SCHEME:path}` — a bash-style
parameter-expansion form, where the `SCHEME` names a backend and
the `path` is the backend-specific lookup key.  Examples:

| Reference | Meaning |
|---|---|
| `${VAULT:/secret/staging/db/rw}` | Fetch from HashiCorp Vault at the given path |
| `${AWSSM:staging/db/rw}` | Fetch from AWS Secrets Manager by secret name |
| `${AZUREKV:staging-db-rw}` | Fetch from Azure Key Vault |
| `${GCPSM:projects/p/secrets/staging-db-rw/versions/latest}` | Fetch from GCP Secret Manager |
| `${FILE:/run/secrets/staging-db-rw}` | Read the value from a file path |
| `${ENV:STAGING_DB_PASSWORD}` | Read from a process env var (legacy bridge) |
| `${KEYCHAIN:com.company.app/staging-db-rw}` | Read from macOS Keychain (local dev) |
| `${PROMPT:Enter staging DB password}` | Prompt the user (local dev only) |
| `${DERIVED:other-credential-id/field}` | Use a field of another credential |
| `${LITERAL:hunter2}` | Literal value; the syntax is explicit (rare; for tests) |

Three things the syntax gets right:

- **The scheme is explicit.**  A grep for `${VAULT:` finds every
  Vault reference in a project.  No ambiguity, no implicit defaults.
- **The path is opaque to the parser.**  The resolver decides how
  to interpret it.  The reference syntax does not constrain backend
  semantics.
- **The reference is recognizable in logs.**  When a `${VAULT:...}`
  string accidentally appears in an error message or a debug dump,
  it is immediately distinguishable from a literal secret.  This
  is a security defence-in-depth — the syntax is loud, so a leak of
  a reference is catastrophically less bad than a leak of a value.

The syntax is bash-style on purpose.  Shell users recognise the
shape immediately.  YAML keeps the string intact (the curly braces
do not require escaping).  Most editors will not "helpfully" parse
the reference into something else.

## The Resolver interface

The resolver is the piece you build.  It has a small interface:

```python
from typing import Optional


class CredentialReference:
    """
        A parsed credential reference of the form ${SCHEME:path}.
    """
    def __init__(self, scheme: str, path: str, raw: str):
        self.scheme = scheme    # e.g. "VAULT"
        self.path = path        # e.g. "/secret/staging/db/rw"
        self.raw = raw          # the original "${VAULT:/secret/...}" string
        return


class CredentialBackend:
    """
        Abstract interface for a backend that resolves credential references.
    """

    SCHEME: str = None  # subclasses override; the dispatch key

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        """
            Given a credential reference whose scheme matches this backend,
            return the materialised secret value as a string.

            Return None if the reference cannot be resolved (the resolver
            then tries the next backend, or raises ConfigurationError if
            no backend resolves the reference).
        """
        raise NotImplementedError("CredentialBackend subclasses must implement resolve")


class CredentialResolver:
    """
        Walks the credential configuration data, finds ${SCHEME:path}
        references, and replaces each with the value the matching backend
        returns.
    """

    def __init__(self, backends: list):
        # backends are tried in order; the first whose SCHEME matches the
        # reference is asked to resolve.  If it returns None, the resolver
        # raises rather than falling through to a different backend with the
        # same scheme.
        self._backends = backends
        self._scheme_to_backend = {b.SCHEME: b for b in backends}
        return

    def resolve_dict(self, data: dict) -> dict:
        """
            Walk a parsed YAML dict, replace every ${SCHEME:path} string
            with the resolved value.  Return a new dict; do not mutate the
            input.
        """
        return self._walk(data)

    # ... _walk, _resolve_string implementation in the worked example below
```

The interface is small for a reason: every method has a single
responsibility, and the responsibilities don't grow.  Adding a new
backend means writing a new `CredentialBackend` subclass, not
modifying the resolver.

## The shipped backends (none) and the ones you build

The shipped scope of `codecraftai-credentials` is zero backends.  The
following sections walk through the most common backends a reader
will build.  Each subsection is a recipe — interface contract,
implementation sketch, when to use it, when not to.

### File backend

The simplest backend.  Reads the secret value from a file on disk.
Useful for:

- Containerised deployments where secrets are mounted as files
  (Kubernetes `Secret` mounted to `/run/secrets/...`, Docker
  Swarm secrets, AWS ECS secrets-as-files).
- Local development where the secret is in `~/.ccai/secrets/`
  protected by filesystem permissions.
- CI where the runner has been pre-populated with secret files at a
  well-known path.

```python
class FileBackend(CredentialBackend):
    SCHEME = "FILE"

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        path = os.path.abspath(
            os.path.expanduser(os.path.expandvars(reference.path))
        )
        if not os.path.exists(path):
            return None
        with open(path, "r") as fh:
            value = fh.read().rstrip("\n")
        return value
```

Two design decisions worth noting:

- **`expanduser` and `expandvars`** are applied to the path so
  `${FILE:~/.ccai/secrets/db}` and
  `${FILE:$XDG_RUNTIME_DIR/secrets/db}` both work.
- **Trailing newline stripping** because most tools that write
  secrets to files (heredocs, `echo`, vault's `vault read -field=...`)
  leave a trailing newline that breaks the credential value.

When *not* to use the file backend: when the secret needs to rotate
without a process restart.  A file value is read once at resolution
time; rotation requires re-resolution.  Chapter 11 covers rotation.

### Environment-variable backend

Reads the secret value from a process environment variable.  Useful
for:

- Migrating from existing env-var-based pipelines.  Lets you migrate
  the *configuration* to typed credentials before migrating the
  *backend*.
- CI vendors whose secret store exposes secrets as environment
  variables and not as files (some older Jenkins setups).
- Tests, where you want the secret to come from a controlled
  fixture.

```python
class EnvBackend(CredentialBackend):
    SCHEME = "ENV"

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        return os.environ.get(reference.path)
```

One design decision:

- **Returns None on missing var, not an empty string.**  A missing
  environment variable should fall through to the next backend (if
  any) or raise a clear "no backend resolved this reference" error.
  An empty string would be silently accepted as the secret value,
  which is almost certainly a bug.

When *not* to use the env-var backend: as the *primary* backend in
production.  It exists to bridge the gap from a previous discipline
to the new one.  Once migration is complete, the env-var backend
can be removed.

### Prompt backend (local-dev only)

Prompts the developer for the secret value at process start.
Useful for local development where the developer would rather type
a password than copy a token into a file.

```python
class PromptBackend(CredentialBackend):
    SCHEME = "PROMPT"

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        if not sys.stdin.isatty():
            return None  # not interactive; let other backends try
        return getpass.getpass(prompt=f"{reference.path}: ")
```

Two design decisions:

- **Returns None on non-TTY** so the same credentials YAML can be
  used in CI without the prompt firing.  A `${PROMPT:...}` reference
  in CI silently falls through to the next backend; a missing
  fall-through resolves to a clear error.
- **`getpass.getpass`** rather than `input()` so the secret is not
  echoed to the terminal.

When *not* to use prompt: in any non-interactive context.  The
fall-through above is the safety net.

### HashiCorp Vault backend

The most common production secrets backend.  Uses the `hvac`
library (or your team's wrapper).

```python
class VaultBackend(CredentialBackend):
    SCHEME = "VAULT"

    def __init__(self, vault_addr: str, vault_token: str):
        import hvac
        self._client = hvac.Client(url=vault_addr, token=vault_token)
        return

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        # path format: /secret/path/to/key#field
        path, field = reference.path.split("#", 1) if "#" in reference.path else (reference.path, "value")
        try:
            response = self._client.secrets.kv.v2.read_secret_version(
                path=path.lstrip("/")
            )
        except Exception:
            return None
        return response["data"]["data"].get(field)
```

Three design decisions:

- **The Vault path supports a `#field` suffix** so a single Vault
  secret can hold multiple fields and the credential reference
  picks one.  `${VAULT:/secret/staging/db#username}` and
  `${VAULT:/secret/staging/db#password}` resolve to the two fields
  of the same Vault entry.
- **The Vault client is constructed at backend creation time**, not
  per-reference.  Vault auth happens once; subsequent reads are
  cheap.
- **Exceptions are swallowed and converted to None.**  A real
  resolver would log the exception (with the reference path but
  *not* the response body) so the engineer can see why a Vault
  lookup failed without leaking the secret if it did succeed.

When *not* to use Vault: when your team doesn't have a Vault
deployment.  Vault is excellent but operationally non-trivial; for
teams that already have one, this backend is the right answer.  For
teams that don't, file + a separate secrets-distribution channel
(Kubernetes Secrets, cloud-native KMS) is often simpler.

### Cloud-native secret-manager backends

AWS Secrets Manager, GCP Secret Manager, Azure Key Vault — each
has its own client library and its own quirks, but the backend
shape is the same:

```python
class AwsSecretsManagerBackend(CredentialBackend):
    SCHEME = "AWSSM"

    def __init__(self, region: str):
        import boto3
        self._client = boto3.client("secretsmanager", region_name=region)
        return

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        # path format: secret-name#field (for JSON secrets) or just secret-name
        secret_name, field = reference.path.split("#", 1) if "#" in reference.path else (reference.path, None)
        try:
            response = self._client.get_secret_value(SecretId=secret_name)
        except Exception:
            return None
        if "SecretString" not in response:
            return None
        secret_string = response["SecretString"]
        if field is None:
            return secret_string
        try:
            data = json.loads(secret_string)
        except json.JSONDecodeError:
            return None
        return data.get(field)
```

Same pattern for GCP and Azure with the relevant SDK calls.  When
your environment is single-cloud, one backend is enough.  When
your environment is multi-cloud, ship both, and let the
`${SCHEME:path}` syntax route each reference to the right one.

### Keychain backend (macOS local dev)

For developers on macOS who want to keep secrets in the system
Keychain instead of in plain files:

```python
class KeychainBackend(CredentialBackend):
    SCHEME = "KEYCHAIN"

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        # path format: service/account
        try:
            service, account = reference.path.split("/", 1)
        except ValueError:
            return None
        try:
            result = subprocess.run(
                ["security", "find-generic-password", "-s", service, "-a", account, "-w"],
                capture_output=True, text=True, check=True,
            )
        except subprocess.CalledProcessError:
            return None
        return result.stdout.rstrip("\n")
```

When the developer hasn't yet stored the secret in Keychain, the
resolver falls through to the next backend (typically prompt).
The Keychain entry then gets created by hand or via a setup script.

### Derived backend

For credentials that compose from other credentials.  A worked
example: a database connection where the password is the same as
the SSH password for the host the database runs on:

```yaml
- identifier: legacy-db-on-runner-1
  categories: [basic]
  username: app
  password: ${DERIVED:ci-runner-ssh#password}
```

The `DERIVED` backend looks up another credential in the
already-loaded set and returns one of its fields:

```python
class DerivedBackend(CredentialBackend):
    SCHEME = "DERIVED"

    def __init__(self, manager):
        # the manager parameter must be the partially-loaded credential
        # manager.  The derived backend is special: it can only resolve
        # references to credentials that have already been resolved.
        self._manager = manager
        return

    def resolve(self, reference: CredentialReference) -> Optional[str]:
        try:
            cred_id, field = reference.path.split("#", 1)
        except ValueError:
            return None
        try:
            cred = self._manager.lookup_credential(cred_id)
        except ConfigurationError:
            return None
        return getattr(cred, field, None)
```

The derived backend is order-sensitive: the credential it derives
*from* must already be loaded when the derivation is resolved.
The resolver enforces this by performing two passes (resolve all
non-derived references first, then resolve derived ones).

The derived backend is also the one place the resolver discipline
gets *cycle-prone*: a `cred-A` derived from `cred-B` derived from
`cred-A` is a cycle that would loop forever.  A real resolver
detects cycles and raises a clear error.

## The resolution-order recipe

When a `${SCHEME:path}` reference's scheme matches more than one
backend (which the proposed dispatch model prevents — exactly one
backend per scheme — but more generally, when a reference *could*
be resolved by multiple paths), the discipline follows a strict
preference order:

1. **Vault / cloud secret manager** preferred over everything.  If a
   secret can be in a centralised audited store, that's where it
   should be.
2. **File backend** preferred over env or prompt.  Files have
   filesystem permissions; environment variables don't.
3. **Environment variable** as a *bridge* during migration.  Should
   fade out of production over time.
4. **Prompt** only in interactive contexts.  Never the production
   answer.
5. **Derived** as a special case, resolved in a second pass.

The proposed dispatch model — exactly one backend per scheme —
makes this less an "order" question and more a "default scheme"
question.  Most credentials should reference the centralised store
directly.  The order matters when a credential *could* be looked up
multiple ways, in which case the discipline says: prefer the form
your secret-handling auditor would prefer.

## Pre-load vs post-lookup resolution

The resolver pattern can fit at two seams:

### Pre-load resolution (recommended default)

Resolve all references *before* calling `CredentialManager.load_credentials`.
The materialised values become the credential field values; the
manager and the rest of the orchestration never see the references.

```python
# orchestration startup
resolver = build_resolver(...)
with open("config/credentials/credentials-staging-east.yaml", "r") as fh:
    raw = yaml.safe_load(fh)

# Resolve every ${SCHEME:path} in the raw dict.  resolved_credentials
# now contains literal secret strings.
resolved = resolver.resolve_dict(raw)

# Hand the resolved dict to the manager.  The manager doesn't know
# that the values were ever references.
mgr = CredentialManagerSingleton(
    credential_info=resolved,
    credential_uris=["config/credentials/credentials-staging-east.yaml"],
)
```

Pre-load resolution is the recommended default because:

- The manager and the rest of the orchestration are simpler.
- Failure to resolve fails the orchestration at *startup*, not at
  the moment of first credential use.
- The audit log of "which secrets were fetched at startup" is a
  single resolved-batch event, not a stream of per-lookup events.

### Post-lookup resolution

Resolve references *on demand* the first time a credential is
looked up.  Wrap the manager:

```python
class LazyResolvingManager:
    def __init__(self, raw_manager: CredentialManager, resolver: CredentialResolver):
        self._raw = raw_manager
        self._resolver = resolver
        self._resolved_cache = {}
        return

    def lookup_credential(self, key: str):
        if key in self._resolved_cache:
            return self._resolved_cache[key]
        cred = self._raw.lookup_credential(key)
        # walk cred.as_dict(), resolve ${SCHEME:path} fields, rebuild
        materialised = self._materialise(cred)
        self._resolved_cache[key] = materialised
        return materialised
```

Post-lookup resolution is the right answer when:

- The orchestration uses only a small fraction of declared
  credentials per run.  Resolving everything at startup is wasted
  work.
- Secrets are short-lived and need to be re-fetched periodically
  (the cache becomes a TTL cache; the next chapter on rotation
  builds on this).
- The startup time budget is tight and the resolver is slow (some
  vault backends have noticeable latency at first connect).

Most teams should start with pre-load resolution and only move to
post-lookup if a specific need surfaces.

## Host scoping

The shipped `codecraftai-credentials` package has **no notion of host
scope** — a credential applies wherever its identifier is referenced.
This is sometimes too permissive.  A credential that lets you SSH to
the staging runner pool should not silently work against the
production runner pool just because both apods reference it by the
same identifier.

The proposed discipline: extend the categories tag with host-scope
annotations, and have the resolver enforce them at lookup time.

```yaml
- identifier: ci-runner-ssh
  categories:
    - ssh
    - host-scope:staging-east
    - runner-fleet
```

The resolver wraps `lookup_credential` to check that the active
landscape's label matches the host-scope annotation:

```python
def lookup_credential(self, key: str):
    cred = self._raw.lookup_credential(key)
    scopes = [c.split(":", 1)[1] for c in cred.categories if c.startswith("host-scope:")]
    if scopes:
        active_landscape = LandscapeSingleton().label
        if active_landscape not in scopes:
            raise ConfigurationError(
                f"Credential {key} is scoped to {scopes} but active landscape is {active_landscape}"
            )
    return cred
```

This is a discipline addition the reader builds.  It is one example
of why having the resolver layer be *yours* matters: the
host-scoping rule that fits your environment is not the same as the
rule that fits a different team's environment.

## Putting it together: a complete resolver

A complete `CredentialResolver` plus a small set of backends, wired
into orchestration startup, looks like this:

```python
import os
import re
import yaml
from typing import Any, Dict, List, Optional

# ... CredentialBackend, FileBackend, EnvBackend, VaultBackend
# ... defined as in the recipes above

REFERENCE_RE = re.compile(r"^\$\{([A-Z]+):(.+)\}$")


class CredentialResolver:
    def __init__(self, backends: List[CredentialBackend]):
        self._scheme_to_backend = {b.SCHEME: b for b in backends}
        return

    def resolve_dict(self, data: Dict[str, Any]) -> Dict[str, Any]:
        return self._walk(data)

    def _walk(self, value):
        if isinstance(value, dict):
            return {k: self._walk(v) for k, v in value.items()}
        if isinstance(value, list):
            return [self._walk(v) for v in value]
        if isinstance(value, str):
            return self._resolve_string(value)
        return value

    def _resolve_string(self, raw: str) -> str:
        m = REFERENCE_RE.match(raw)
        if not m:
            return raw  # not a reference, leave alone
        scheme, path = m.group(1), m.group(2)
        backend = self._scheme_to_backend.get(scheme)
        if backend is None:
            raise ConfigurationError(
                f"No backend registered for scheme {scheme} in reference {raw}"
            )
        ref = CredentialReference(scheme=scheme, path=path, raw=raw)
        value = backend.resolve(ref)
        if value is None:
            raise ConfigurationError(
                f"Backend {scheme} could not resolve reference {raw}"
            )
        return value


def build_resolver_for_environment(profile: str) -> CredentialResolver:
    """
        Build the resolver for a particular environment profile.
        The set of backends differs per profile — this is the
        environment-specific bit (chapter 12 expands on this).
    """
    if profile == "ci":
        return CredentialResolver([
            VaultBackend(vault_addr=os.environ["VAULT_ADDR"],
                         vault_token=os.environ["VAULT_TOKEN"]),
            FileBackend(),
            EnvBackend(),
        ])
    elif profile == "local-dev":
        return CredentialResolver([
            KeychainBackend(),
            FileBackend(),
            PromptBackend(),
        ])
    elif profile == "staging":
        return CredentialResolver([
            VaultBackend(vault_addr=...,
                         vault_token=...),
            FileBackend(),
        ])
    else:
        raise ConfigurationError(f"Unknown profile {profile!r}")


# Wired into orchestration startup
def bootstrap_credentials(profile: str, credentials_path: str) -> None:
    with open(credentials_path, "r") as fh:
        raw = yaml.safe_load(fh)

    resolver = build_resolver_for_environment(profile)
    resolved = resolver.resolve_dict(raw)

    CredentialManagerSingleton(
        credential_info=resolved,
        credential_uris=[credentials_path],
    )
    return
```

Sixty lines — including the reference-parsing regex, the recursive
walk, the per-backend dispatch, and the per-environment configuration
— for a complete resolver.  The same sixty lines work for every
project in your portfolio.

## What the resolver does *not* do

Three things to keep out of the resolver:

### Not authentication

The resolver fetches secrets.  It does *not* perform authentication
against systems using those secrets.  A `BasicCredential` whose
password the resolver materialised is still inert; the consumer's
HTTP client is what authenticates.

This matters because the resolver runs *once* per orchestration
startup, but authentication runs every time a credential is used.
Conflating the two would force authentication failures at the wrong
seam.

### Not access-control policy

The resolver fetches secrets.  It does *not* decide whether the
caller is *allowed* to fetch a particular secret.  That is a
property of the backend's own access control (Vault policies, IAM
roles, KV store ACLs).

A team's credential discipline should rely on the backend's
permission system to enforce who-can-read-what, not on the
resolver.  When the resolver fails to fetch (because the caller
lacks permission), the backend returns the failure; the resolver
surfaces it as a clear error.

### Not rotation

The resolver fetches the *current* value at the time it runs.  When
a credential rotates, the cached resolved value goes stale.  The
chapter on credentials in flight (chapter 11) is the rotation
discipline; the resolver is one of the building blocks but not the
whole story.

For long-running orchestrations (anything over an hour), the
post-lookup resolution model with TTL caching becomes important;
chapter 11 expands on the pattern.

## The migration path

For a team adopting the discipline against an existing pipeline:

1. **Stand up one backend.**  Pick the one that matches your
   environment — Vault if you have it, AWS Secrets Manager if
   you're on AWS, file backend backed by Kubernetes Secrets if you
   run on K8s.  Do not try to ship six backends on day one.
2. **Migrate the credential file** to use the
   `${SCHEME:path}` syntax for one credential at a time.  Keep
   the old environment variables as a fallback during migration
   by wiring the EnvBackend.
3. **Validate at every step.**  Each migrated credential is a
   separate change; each change is reviewable; each change can be
   rolled back independently.
4. **Remove the EnvBackend** once every credential has migrated.
5. **Add host-scoping annotations** as a second pass once the
   primary backends are stable.

The migration takes weeks, not days.  This is fine — the discipline
compounds over time.  A single rotated production credential that
didn't leak through environment variables to a CI log is, by itself,
worth the migration.

## Summary

- The `codecraftai-credentials` package treats secret values as opaque
  strings.  Resolution — turning `${VAULT:/path}` into the actual
  secret — is the consumer's responsibility, deliberately so.
- The proposed reference syntax is `${SCHEME:path}`, with each
  scheme corresponding to a backend.  The `SCHEME` is greppable, the
  `path` is opaque to the parser, and the form is loud enough to
  notice if it leaks into a log.
- The backend interface is small: one `resolve(reference)` method
  returning the materialised value or None.  Adding a backend is a
  new subclass and a new dispatch entry.
- Common backends a reader will build: File, Env (legacy bridge),
  Prompt (local dev), Vault, AWS Secrets Manager, Azure Key Vault,
  GCP Secret Manager, Keychain (macOS local dev), Derived
  (cred-from-cred).
- Pre-load resolution (resolve before `load_credentials`) is the
  recommended default; post-lookup resolution is the answer when
  rotation or sparse usage matters.
- Host-scoping is a discipline addition the reader builds on top of
  the categories vocabulary; the resolver enforces it at lookup
  time.
- The complete resolver — regex, recursive walk, per-environment
  backend selection, orchestration wiring — is roughly sixty lines
  and is reusable across projects.
- The resolver is not authentication, not access-control policy,
  and not rotation.  Each is a distinct concern with its own seam.

The next chapter is credentials in flight — rotation during long
runs, encryption at rest of credential files, audit trails, and the
"never log a credential" discipline made operational.
