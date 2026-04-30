# Chapter 11 — Credentials in Flight

> *A credential at rest is a file you encrypt.  A credential in
> flight is a process attribute that lives only in the process's
> memory and never touches disk again.  Most credential bugs happen
> at the boundary between the two.*

The previous chapter showed how to fetch a secret from a backend
and turn it into a typed credential object.  The chapter ended
with the credential alive in process memory, ready to be used.
This chapter is about the lifecycle of that in-memory value: how
it stays safe while the process holds it, how it survives rotation
without breaking the run, how it leaves an audit trail, and — most
importantly — how it never accidentally leaks into a place that
outlives the process.

The chapter covers three bands of the lifecycle:

1. **Encryption at rest** — when the credential file itself is
   stored as ciphertext rather than plaintext.  The Fernet machinery
   shipped in `automojo-configuration` is the right tool here and
   the chapter explains how to use it correctly.
2. **Rotation in flight** — what happens when a credential changes
   while the orchestration is running.  Long-running pipelines (a
   nightly suite, a multi-hour data migration) outlive the lifetime
   of many cloud-native credentials, so rotation is operational, not
   theoretical.
3. **Audit and the "never log" rule** — the discipline that keeps
   typed credentials safe from the most common leak vector
   (accidental serialisation in logs and exception messages).

## Encryption at rest

The shipped cryptography module is
[`automojo-configuration/source/packages/automojo/configuration/cryptography.py`](../../../automojo-configuration/source/packages/automojo/configuration/cryptography.py).
Four functions, all using Fernet symmetric encryption:

```python
generate_fernet_key(keyphrase: str) -> str
encrypt_content(key: str, plain_content: str) -> str
decrypt_content(key: str, encrypted_content: str) -> str
create_encrypted_configuration(key: str, plain_configuration: str,
                               format: str = ConfigurationFormat.JSON) -> dict
```

The convention they implement: an encrypted credential file is a
*configuration document* with three top-level fields:

```yaml
version: "1.0"
format: yaml
encrypted_content: "BASE64-CIPHERTEXT-HERE"
```

The `automojo-configuration` loader recognises the
`encrypted_content` key at load time, asks the consumer for a key
or keyphrase, decrypts the ciphertext, and parses the resulting
plaintext as the file's actual content.  The encrypted file looks
identical from the outside whether the inner content is a
credentials YAML, a runtime profile, or a landscape — the same
machinery encrypts any of the six categories.

### The encryption flow

To encrypt an existing credentials file:

```python
import yaml
from automojo.configuration.cryptography import (
    create_encrypted_configuration,
    generate_fernet_key,
)
from automojo.configuration.configurationformat import ConfigurationFormat

# Read the plaintext credentials file
with open("credentials-staging-east.yaml", "r") as fh:
    plain_yaml = fh.read()

# Derive the Fernet key from a keyphrase
key = generate_fernet_key("the-team's-shared-keyphrase-2026q2")

# Wrap the plaintext into an encrypted document
encrypted_doc = create_encrypted_configuration(
    key=key,
    plain_configuration=plain_yaml,
    format=ConfigurationFormat.YAML,
)

# Write the encrypted document
with open("credentials-staging-east.encrypted.yaml", "w") as fh:
    yaml.safe_dump(encrypted_doc, fh)
```

The output file looks like:

```yaml
version: "1.0"
format: yaml
encrypted_content: "Z0FBQUFBQm5UZ0NWUUI4Z0lqdjcyTWVfWWNlcEFTeFRtRC4uLg=="
```

To decrypt at orchestration startup, the loader is given the same
keyphrase (or directly the Fernet key) and the original plaintext
is reconstructed transparently — by the time the credentials reach
`CredentialManager.load_credentials`, the file's content is already
plain.

### A note on the keyphrase

The shipped
[`generate_fernet_key`](../../../automojo-configuration/source/packages/automojo/configuration/cryptography.py)
is **deterministic**: the same keyphrase always produces the same
key.  The implementation is `random.Random(keyphrase).randbytes(32)`
followed by URL-safe base64.

Determinism is a feature, not a bug, in this context.  It means a
team that shares a keyphrase out-of-band can decrypt files without
distributing key material separately.  The keyphrase is the
shared secret; the key derivation is reproducible.

It also means the keyphrase is the entire security story.  Two
implications worth being explicit about:

1. **A weak keyphrase is a weak key.**  `random.Random(keyphrase)`
   is *not* a key-derivation function — there is no salt, no
   iteration count, no resistance against offline brute force.  A
   keyphrase like `"changeme"` produces a key an attacker can
   reproduce in milliseconds if they get the encrypted file.  Use
   keyphrases with the entropy of a real password (long randomly-
   generated strings, password-manager output, or human-chosen
   passphrases of four-or-more random words).
2. **Keyphrase compromise is total.**  A keyphrase that an
   attacker learns gives them access to every file ever encrypted
   with it.  Rotate keyphrases on a schedule (chapter 12 expands
   on rotation across environments) and re-encrypt files with
   the new keyphrase.

For teams whose security posture requires stronger key derivation,
the Fernet primitives accept any 32-byte URL-safe base64 key — the
team is free to derive that key with PBKDF2, scrypt, Argon2, or a
hardware HSM and pass the result directly to `encrypt_content`.
The shipped `generate_fernet_key` is a convenience for the common
case; the underlying machinery is general.

### When to encrypt at rest

The discipline is straightforward:

- **Encrypt** when the credentials file lives in a place where
  unauthorised access is plausible: a shared filesystem, a Git
  repository (even private), an artifact store, a backup.
- **Don't encrypt** when the file lives behind a stronger access
  boundary than encryption-at-rest would provide — Vault, AWS
  Secrets Manager, K8s Secrets with the K8s-side encryption-at-
  rest enabled.  Double-encrypting adds operational complexity
  without security gain.
- **Always encrypt** when the credentials file is committed to a
  repository, even temporarily.  The Git history is forever; an
  unencrypted credential committed once is leaked forever.

The right answer for most teams: keep credentials *out* of the
repository entirely (Vault, AWS Secrets Manager, K8s Secrets), use
the resolver layer from chapter 10 to fetch them at runtime, and
keep the encryption-at-rest machinery as a fallback for the cases
where in-repo storage is unavoidable.

## Rotation in flight

The shipped credential classes are immutable after construction.
A `BasicCredential` instantiated with `password="hunter2"` will
return `"hunter2"` from `cred.password` for the rest of its life;
there is no setter.  This is a deliberate API choice.

Rotation, then, is not a property of the credential object — it is
a property of the *resolution layer* that produces credential
objects.  A long-running orchestration with rotating credentials
needs the resolver to re-fetch the current value periodically.

### When rotation matters

Three patterns make rotation operational:

1. **Long-running pipelines.**  An orchestration that runs for more
   than the credential's TTL must re-fetch.  Cloud-native
   credentials (AWS STS tokens, Azure AAD tokens) typically have
   1-12 hour lifetimes.  Database credentials with HashiCorp Vault
   dynamic-secret leases default to one hour.
2. **Distributed orchestrations.**  When the orchestration spawns
   workers that outlive the original resolver invocation, each
   worker may need its own rotation strategy.  Volume 2's
   distributed-test-job chapters address this; the pattern reduces
   to "each worker re-resolves at startup, plus periodically."
3. **Mid-run rotation events.**  An on-call engineer who rotates a
   compromised credential during an active orchestration shouldn't
   have to abort the run.  The orchestration should pick up the
   new value at the next resolution.

### The TTL cache pattern

Chapter 10 introduced post-lookup resolution as one of the two
seams the resolver fits at.  Rotation is the canonical reason to
choose it.  The shape:

```python
import time
from typing import Optional


class TTLCachingResolvingManager:
    """
        Wraps a CredentialManager and a CredentialResolver.  Caches
        resolved credentials with a per-credential TTL.  Beyond the
        TTL, the next lookup re-resolves from the backend.
    """

    DEFAULT_TTL_SECONDS = 300   # 5 minutes; per-credential override below

    def __init__(self, raw_manager, resolver):
        self._raw = raw_manager
        self._resolver = resolver
        self._cache = {}   # ident -> (resolved_credential, expires_at)
        return

    def lookup_credential(self, key: str):
        now = time.time()
        if key in self._cache:
            cred, expires_at = self._cache[key]
            if now < expires_at:
                return cred

        # Cache miss or expired — re-resolve.
        raw_cred = self._raw.lookup_credential(key)
        ttl = self._ttl_for(raw_cred)
        materialised = self._resolver.materialise(raw_cred)
        self._cache[key] = (materialised, now + ttl)
        return materialised

    def _ttl_for(self, cred) -> int:
        # Per-credential TTL via the categories tag.  E.g. a
        # category tag of "ttl:60" overrides the default to 60s.
        for cat in cred.categories:
            if cat.startswith("ttl:"):
                try:
                    return int(cat.split(":", 1)[1])
                except ValueError:
                    pass
        return self.DEFAULT_TTL_SECONDS
```

Three design decisions:

- **The cache is keyed by identifier**, not by credential object
  identity.  An updated value with the same identifier replaces the
  old one transparently.
- **Per-credential TTL via the categories tag.**  The shipped
  categories field doubles for routing and for metadata; adding a
  `ttl:<seconds>` tag is the natural place for the TTL.  Default
  five minutes; explicit override per credential.
- **No eager refresh.**  The cache invalidates lazily on the next
  lookup.  A credential that is never looked up is never re-fetched.
  This is the right behaviour for the common case (sparse access);
  for hot credentials, a background refresh thread is a small
  extension.

### Handling rotation failures

Rotation can fail for the same reasons resolution can: the
backend is unreachable, the credential has been revoked, the
caller has lost permission.  The discipline:

- **Old value remains usable until rotation succeeds.**  A failed
  re-fetch should *not* invalidate the cached value — keep using
  the old one until the new one is available.  Rotation that
  brittlely interrupts the run is worse than a slightly-stale
  credential.
- **Rotation failures emit a clear log event.**  The next chapter
  on auditing covers what to log; rotation failure is one of the
  events that should always reach the audit channel.
- **Cascading failures are bounded.**  When the secrets backend is
  down for an hour, every cached credential will eventually try to
  rotate and fail.  The TTL pattern naturally bounds this — failed
  re-fetches set a short retry interval rather than blocking, and
  the orchestration continues with stale credentials rather than
  cascading the backend outage into a pipeline outage.

A real production resolver implements rotation as a small state
machine per credential: `fresh → expiring → refreshing → fresh` or
`fresh → expiring → refresh-failed → fresh-but-stale`.  The
state-machine details are environment-specific; the pattern is
universal.

## The audit trail

Credentials in flight need an audit trail for two reasons.  The
first is operational: when an orchestration fails, "which
credential did we use?" should have an immediate answer.  The
second is regulatory: many environments require a log of every
credential access, with the *identifier* (not the value) and a
timestamp and a calling context.

### What to log

Three fields per audit event:

- **Credential identifier.**  Always logged.
- **Calling context.**  The orchestration step or job that
  requested the credential.  This is what makes the audit trail
  *useful* for debugging — "credential `staging-db-rw` was used by
  step `run-migrations` at 14:32 UTC" is actionable.
- **Resolution status.**  Whether the lookup hit the cache or
  re-fetched, whether re-fetch succeeded or failed, the latency.

Three fields per audit event that are **never logged**:

- **The credential value itself.**  No exceptions.
- **Anything derived from the credential value** (the resulting
  HTTP headers, the connection string, the SSH session token).
- **The credential's `as_dict()` output.**  The shipped
  `as_dict()` includes the secret material — see the next section.

A simple audit logger:

```python
class CredentialAuditLog:
    def __init__(self, sink):
        # sink is a logger or a structured-log writer; this class
        # never decides what backend the audit goes to, only what
        # fields to emit.
        self._sink = sink
        return

    def record_lookup(self, identifier: str, calling_context: str,
                      cache_hit: bool, latency_ms: float):
        event = {
            "type": "credential.lookup",
            "identifier": identifier,
            "context": calling_context,
            "cache_hit": cache_hit,
            "latency_ms": latency_ms,
        }
        self._sink.info(json.dumps(event))
        return

    def record_rotation(self, identifier: str, success: bool,
                        old_value_age_seconds: int):
        event = {
            "type": "credential.rotation",
            "identifier": identifier,
            "success": success,
            "old_value_age_seconds": old_value_age_seconds,
        }
        self._sink.info(json.dumps(event))
        return

    def record_resolution_failure(self, identifier: str, error_class: str,
                                  reference_path: str):
        event = {
            "type": "credential.resolution_failure",
            "identifier": identifier,
            "error_class": error_class,
            # Reference path is the BACKEND path (e.g.
            # "/secret/staging/db/rw"), not the credential value.
            "reference_path": reference_path,
        }
        self._sink.error(json.dumps(event))
        return
```

The audit logger is wired into the resolving manager — every
`lookup_credential` call emits an event, and the resolver emits an
event on every backend resolution and every failure.  The wiring is
a one-line addition per call site; the discipline is to never let a
credential lookup go unaudited.

## The "never log a credential" rule, made operational

Chapter 9 introduced the rule.  This is where it becomes an
operational discipline.  Three practices keep it in place.

### `__repr__` redaction

Every credential class should override `__repr__` to elide the
secret material.  An audit finding from chapter 9 noted that the
shipped classes mostly do *not* do this today — that's a defect
worth fixing in the package.  The pattern:

```python
class BasicCredential(BaseCredential):
    # ...

    def __repr__(self) -> str:
        return (
            f"BasicCredential(identifier={self._identifier!r}, "
            f"username={self._username!r}, password=<redacted>)"
        )

    __str__ = __repr__
```

After the override, `f"{cred}"` produces
`BasicCredential(identifier='staging-db-rw', username='app_user', password=<redacted>)`.
The same protection applies in tracebacks: when a `BasicCredential`
appears in a function's local scope and an exception is raised,
the variable's representation in the traceback no longer leaks the
password.

The same override is needed for every credential class that has
secret material — that is, all eleven shipped classes.

### Logging filters

Even with `__repr__` redaction, structured logging frameworks
sometimes serialise objects via `__dict__` or `as_dict()` rather
than `__repr__`.  A logging filter at the application boundary
catches this:

```python
import logging


class CredentialRedactingFilter(logging.Filter):
    """
        Inspects log records for fields that look like credential
        material (password, secret, token, key, keypasswd, private,
        client_secret) and redacts them.
    """

    REDACT_KEYS = frozenset({
        "password", "secret", "token", "keypasswd",
        "private", "client_secret", "keyraw",
    })

    def filter(self, record: logging.LogRecord) -> bool:
        for k in list(record.__dict__.keys()):
            if k.lower() in self.REDACT_KEYS:
                record.__dict__[k] = "<redacted>"
        # Also walk the args; same treatment.
        if isinstance(record.args, dict):
            for k in list(record.args.keys()):
                if k.lower() in self.REDACT_KEYS:
                    record.args[k] = "<redacted>"
        return True


logging.getLogger().addFilter(CredentialRedactingFilter())
```

The filter is defence-in-depth.  It runs on every log record,
catches accidental leaks the `__repr__` override missed, and
imposes essentially zero performance cost.

### Exception traceback handling

Python's default exception handling can include large amounts of
local-variable context in tracebacks (especially with libraries
like `rich` or `better-exceptions`).  When a credential is in
scope at the point of an exception, the credential's `__repr__`
goes into the traceback — which is why the redaction in
`__repr__` is the first line of defence.

The second line of defence is to install a custom exception hook
that strips local-variable contexts from credential-bearing frames:

```python
import sys
import types


def _make_credential_aware_excepthook(original):
    def _hook(exc_type, exc_value, exc_traceback):
        # Walk the traceback; for any frame whose locals contain a
        # BaseCredential, blank the locals before the original hook
        # serialises them.
        from automojo.credentials.basecredential import BaseCredential
        tb = exc_traceback
        while tb is not None:
            for var, val in list(tb.tb_frame.f_locals.items()):
                if isinstance(val, BaseCredential):
                    tb.tb_frame.f_locals[var] = "<credential redacted>"
            tb = tb.tb_next
        return original(exc_type, exc_value, exc_traceback)
    return _hook


sys.excepthook = _make_credential_aware_excepthook(sys.excepthook)
```

This is paranoid.  Many teams will not need it.  But for teams
with strict regulatory requirements (PCI, HIPAA, SOC 2 environments
where credential leakage is a reportable incident), the paranoia is
worth the small operational cost.

### The audit guardrail

Code review and static checks catch the patterns that runtime
filtering cannot.  Two guardrails worth adopting:

1. **Grep guardrail in CI.**  A pre-commit hook or CI step that
   greps for patterns like `print(*cred*)`, `logger.*\(*cred*\)`,
   `f".*{cred*}.*"` and fails the build if any match.  False
   positives are common (the patterns also match harmless variable
   names) and the hook needs an allowlist; that's the cost of the
   guardrail.
2. **AST-level lint.**  A custom lint rule that flags any `print`,
   `logger`, or f-string that takes an argument whose type
   annotation is `BaseCredential` or a subclass.  More accurate than
   grep, more work to set up.  Worth it for projects above ~10 KLOC.

The guardrails do not need to be perfect.  The redaction at
`__repr__` and the logging filter catch most leaks at runtime; the
guardrails catch the patterns the runtime defences miss before they
ever ship.

## The `as_dict` paradox

Every shipped credential class has an `as_dict()` method that
returns a dictionary representation of the credential — *including
the secret material*.  This is a deliberate trade-off:

- **For:** serialisation is necessary for some legitimate
  operations.  Re-encrypting a credentials file requires reading,
  modifying, and writing the credential records.  Sending a
  credential over an authenticated channel to a remote worker
  requires serialising it.  The shipped `as_dict` is the
  serialisation primitive that supports those use cases.
- **Against:** `as_dict` is *also* what naive serialisation paths
  call by accident.  Many JSON encoders, structured loggers, and
  introspection libraries call `as_dict` if it exists, on the
  reasonable assumption that a method named `as_dict` produces a
  representation appropriate for serialisation.  The result is a
  log entry containing the password.

The discipline:

- **`as_dict` is for *deliberate* serialisation only.**  Code that
  calls it should be rare, named clearly, and reviewed.  The places
  that legitimately call `as_dict` are: the credential file writer,
  the encrypted-config builder, the inter-process credential
  transport (rare).
- **Do not let auto-serialising libraries discover `as_dict`.**  If
  your structured-logging library serialises any object with an
  `as_dict` method, configure it not to.  If your JSON encoder
  treats `as_dict` as a hook, override that hook for credential
  classes specifically.  The redacting filter from the previous
  section is a good defence-in-depth here too.
- **Consider `to_safe_dict()` as a future addition.**  An audit
  finding from chapter 9 noted the absence of `__repr__` redaction;
  a parallel audit finding here is that the shipped credential
  classes would benefit from a `to_safe_dict()` method that returns
  the same shape as `as_dict()` with all secret fields elided.
  That would give the structured-logger a safe option that
  superficially resembles `as_dict()`, reducing the chance that a
  developer reaches for the wrong one.

## What credentials in flight is *not*

Three boundaries to keep clear:

### Not authentication

The chapter has talked about credentials in process memory, but
*using* the credential to actually authenticate is the consumer's
job — the HTTP client's, the database driver's, the SSH library's.
The credential object is passive material.  Authentication
failures (wrong password, expired token, revoked key) are detected
by the consumer at use time, not by the credential class.

This is also why rotation lives at the resolver layer rather than
inside the credential class: the credential class doesn't know
when authentication failed.  The orchestration's authentication-
aware code catches the failure and asks the resolver to re-fetch.

### Not access-control policy

Whether a particular credential is *allowed* to access a particular
resource is enforced by the resource (the database, the API
endpoint, the SSH server) and by the credential's backend access
controls (Vault policies, IAM roles).  The credential class does
not enforce policy; the resolver does not enforce policy; the
orchestration does not enforce policy.  Policy lives at the system
of record.

The discipline of "audit the credential identifier and calling
context" gives the policy enforcement points the information they
need to refuse improper accesses.  But the refusal happens at the
backend, not at the orchestration.

### Not a secret store

A credentials file with redacted `__repr__` and a logging filter
and `as_dict` discipline is *still not a secret store*.  It is a
fetched-and-cached projection of the secret store.  The system of
record for the secret is Vault (or AWS Secrets Manager, or your
chosen backend); the credential object is an in-process copy.

When you need to rotate a secret, rotate it at the backend.  When
you need to revoke access, revoke at the backend.  The credential
object's TTL cache will pick up the change at the next resolution
window.  Treating the credential object as the source of truth
inverts the right architecture.

## Putting it together

A complete in-flight credential discipline has six pieces:

1. **Encryption at rest** when the credentials file is in a
   plausible-leak location.  Fernet from `automojo-configuration`,
   keyphrase with real entropy, files re-encrypted on keyphrase
   rotation.
2. **TTL caching with rotation** at the resolver layer.  Per-
   credential TTL via the `categories` tag.  Stale-on-failure
   semantics; never cascade backend outages into pipeline outages.
3. **`__repr__` redaction** on every credential class.  The first
   line of defence against accidental logging.
4. **A logging filter** at the application boundary.  Defence in
   depth; catches what `__repr__` missed.
5. **An audit trail** that emits a structured event per lookup,
   per rotation, per failure.  Records the identifier and calling
   context; never the value.
6. **CI-level guardrails** (grep or AST-based) that catch
   credential-logging patterns before they ship.

Together, these form a credential discipline that is operationally
robust against the most common failure modes (leak through logging,
brittle rotation, credential drift between the backend and the
process) without imposing a heavy ceremony on the team's daily
workflow.

## Summary

- **Encryption at rest** uses the Fernet machinery in
  `automojo-configuration/cryptography.py`.  An encrypted file is
  a configuration document with `version`, `format`, and
  `encrypted_content` fields.  The shipped `generate_fernet_key` is
  deterministic from a keyphrase — strong-enough keyphrases are
  the entire security story.  For PBKDF2/scrypt/Argon2 derivation,
  pass a custom 32-byte URL-safe-base64 key directly to
  `encrypt_content`.
- **Rotation in flight** is a property of the resolver layer, not
  the credential object.  The TTL caching pattern from chapter 10
  is the natural place to implement rotation; per-credential TTL
  via the `categories` tag (e.g., `ttl:60`) lets short-lived
  credentials co-exist with long-lived ones.  Stale-on-failure
  semantics keep backend outages from cascading.
- **The audit trail** logs three fields per event: identifier,
  calling context, resolution status.  It never logs the value or
  anything derived from it.  Lookups, rotations, and failures all
  become structured audit events.
- **The "never log" rule** has four layers of defence:
  1. `__repr__` redaction on every credential class.
  2. A logging filter at the application boundary.
  3. Custom exception-hook handling for credential-bearing frames.
  4. CI guardrails (grep and/or AST lint) for credential-logging
     patterns.
- **The `as_dict` paradox** is a real trade-off — serialisation is
  necessary, but auto-serialising libraries make the method
  dangerous.  A `to_safe_dict()` parallel method is the proposed
  long-term fix; in the meantime, use the redacting filter as
  defence-in-depth.
- The discipline is *not* authentication, *not* access-control
  policy, *not* a secret store.  Each is the responsibility of a
  different layer; the credential object is passive in-process
  material.

The next chapter is the last in Part III: credentials across
environments — how the same credential vocabulary survives the
trip from local-dev to CI to staging to prod, with backends that
swap out beneath it.
