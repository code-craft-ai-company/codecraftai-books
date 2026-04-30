# Chapter 9 — Credential Vocabularies

> *There are eleven kinds of credential a pipeline ever needs.  This
> chapter names all eleven and explains why the right answer to "what
> about credential type X?" is almost always "use one of the eleven
> and add a `categories` tag."*

Welcome to Part III.  The five categories of Part II
(Specifications, Landscapes, Topologies, Runtimes, Data Profiles)
are introduced and finished.  Credentials — the sixth category —
gets four chapters of its own because the credential discipline has
more moving parts than the other five combined.  This chapter
establishes the vocabulary; chapter 10 is the resolution discipline
(how to materialise secrets at runtime); chapter 11 is credentials
in flight (rotation, encryption, audit); chapter 12 is credentials
across environments.

The portfolio's
[`automojo-credentials`](../../../automojo-credentials/source/packages/automojo/credentials/)
package is the substantive subject of Part III.  An important
finding from the research that grounded this volume is worth stating
up front: **`automojo-credentials` is a pure vocabulary layer**.
It defines the typed credential classes and a registry that loads
them from configuration data.  It does not ship a single
resolution backend — no vault integration, no file backend, no
prompt-on-startup, no rotation API.  The shipped scope is
deliberately narrow.  This chapter is the catalogue of what *is*
shipped; chapter 10 is the discipline for everything that's not.

That narrowness is a feature.  A vocabulary with no opinion about
backends translates to any environment a reader is in.  The eleven
shipped credential classes cover the credential types every team
needs.  The backend layer is yours to build, and the chapter that
follows is the recipe for building it well.

## The three universal fields

Every credential is an instance of a class inheriting from
[`BaseCredential`](../../../automojo-credentials/source/packages/automojo/credentials/basecredential.py).
The base class declares three universal fields that every concrete
credential carries:

| Field | Purpose |
|---|---|
| `identifier` | Unique string used by other configurations to reference this credential |
| `categories` | List of category tags telling the framework what this credential can be used for |
| `role` | Optional role label (default `"priv"`) used to distinguish privilege levels |

A credential YAML entry looks like this in raw form:

```yaml
- identifier: staging-db-rw
  category: basic        # singular form; the loader translates to a list
  categories:            # plural form is also supported and preferred
    - basic
    - rest-basic
  role: priv
  username: app_user
  password: ${VAULT:/secret/staging/db/rw}
```

Three deliberate choices visible in the universals.

### `identifier` — the reference

The identifier is what every other configuration uses to reach a
credential.  A Landscape entry's `credentials: [pi-cluster]` field;
a Data Profile's `credential: staging-db-rw` field; a CloudVenue's
`credential: ci-runner-aws` field.  All three are *references* —
strings that the credential resolution layer (chapter 10) maps to
a concrete credential instance at runtime.

The identifier discipline is the same as for every other category:
unique, stable, descriptive, environment-agnostic.  A credential
called `staging-db-rw` exists in every environment that needs it —
the *same* identifier resolves to a *different* credential value
per environment.  This is the architecture chapter 12 is about.

### `categories` — the type-tag list

The categories field is a list of strings.  Each entry is a category
tag the framework recognises.  Every concrete credential class
*requires* a specific category tag to be present in the list and
raises `ConfigurationError` at construction time if it's missing.
For example,
[`SshCredential`](../../../automojo-credentials/source/packages/automojo/credentials/sshcredential.py)
requires the `ssh` category:

```python
if "ssh" not in categories:
    raise ConfigurationError(
        "The SshCredential should only be given credentials of category 'ssh'."
    )
```

The list form (rather than a single category string) lets a
credential carry *both* the framework-required category tag *and*
application-defined tags that orchestration code uses for grouping
and filtering.  A real credential entry typically looks like this:

```yaml
- identifier: ci-runner-ssh
  categories:
    - ssh                    # framework-required: enables SshCredential
    - runner-fleet           # application-defined: groups runner creds
    - rotated-quarterly      # application-defined: lifecycle metadata
  username: runner
  keyfile: /run/secrets/ssh/ci-runner
```

Orchestration code can ask for "all credentials in the
`runner-fleet` category" and get exactly the SSH and AWS and Azure
credentials the fleet uses — without the framework knowing what
`runner-fleet` means.  The category vocabulary is yours; the
framework only enforces the type-tag.

### `role` — the privilege label

The role field carries one string label, defaulting to `"priv"` for
*privileged*.  Other typical values are `"readonly"`, `"audit"`,
`"admin"`, or any application-defined role term.

Two credentials pointing at the same logical resource with different
roles is the common pattern — a `staging-db-rw` and a
`staging-db-readonly`, the same database, different access levels:

```yaml
- identifier: staging-db-rw
  categories: [basic]
  role: priv
  username: app_user
  password: ${VAULT:/secret/staging/db/rw}

- identifier: staging-db-readonly
  categories: [basic]
  role: readonly
  username: app_readonly
  password: ${VAULT:/secret/staging/db/ro}
```

A Data Profile (chapter 8) selects between them by *identifier*; the
`role` field is metadata that lets reporting and audit code
recognise the privilege tier without parsing the identifier name.

## The category-as-discriminator pattern

The `categories` list serves a second function beyond filtering:
the *first matching tag* tells the
[`CredentialManager`](../../../automojo-credentials/source/packages/automojo/credentials/credentialmanager.py)
which concrete credential class to instantiate.  This is the
category-as-discriminator pattern shared with Data Profiles (chapter
8) and CloudDevices (chapter 4).

Each shipped credential class declares the category tag it claims:

| Class | Category tag claimed |
|---|---|
| `BasicCredential` | `basic` (also accepts `rest-basic`) |
| `SshCredential` | `ssh` |
| `ApiTokenCredential` | `api-token` |
| `PersonalApiTokenCredential` | `personal-api-token` |
| `SecretTokenCredential` | `secret-token` |
| `AwsAccessKeyCredential` | `aws-access-key` |
| `AzureClientSecretCredential` | `azure-client-secret` |
| `PublicKeyCredential` | `public-key` |
| `GithubTokenCredential` | `github-token` |
| `GithubAppCredential` | `github-app` |
| `WifiChoiceCredential` | `wifi-choice` |

The `CredentialManager` walks the list, dispatches to the matching
class, instantiates the credential, and registers it.  Application-
defined tags (`runner-fleet`, `rotated-quarterly`) are kept in the
list but not used for dispatch.

## The eleven shipped classes

Each class is a thin typed container.  None of them perform I/O,
none of them resolve secrets from a backend, none of them rotate or
expire — they hold the values they were constructed with and expose
them as properties.  That is by design: chapter 10 explains why the
backend layer is separate.

### Basic auth

[`BasicCredential`](../../../automojo-credentials/source/packages/automojo/credentials/basiccredential.py)
— the workhorse type for username/password authentication.  Used
for HTTP Basic, REST Basic, database `BasicCredential`-driven
auth.  Two category tags accepted: `basic` (general) and
`rest-basic` (REST-specific).

```yaml
- identifier: staging-api-basic
  categories: [basic]
  username: api_user
  password: ${VAULT:/secret/staging/api/basic}
```

Required fields: `username`, `password`.

### SSH

[`SshCredential`](../../../automojo-credentials/source/packages/automojo/credentials/sshcredential.py)
— SSH credential with multiple authentication modes.  The
constructor accepts any of: password, keyfile, raw key, or
SSH-agent flag, and validates that at least one mechanism is
provided.

```yaml
- identifier: ci-runner-ssh
  categories: [ssh]
  username: runner
  keyfile: /run/secrets/ssh/ci-runner-rsa
  keypasswd: ${VAULT:/secret/ssh/ci-runner-rsa-passphrase}
  allow_agent: false
  primitive: false
```

Required fields: `username` plus *one of* (`password`, `keyfile`,
`keyraw`, `allow_agent: true`).  `keyfile` and `keyraw` are
mutually exclusive.  `primitive: true` switches the consuming
SSH client into "use raw `ssh` commands instead of SFTP/exec
RPC" mode for environments that don't support the full SSH
subsystem.

### API tokens

Three flavours of token-based auth.

[`ApiTokenCredential`](../../../automojo-credentials/source/packages/automojo/credentials/apitokencredential.py)
— a single opaque token.  Used for service tokens that don't have a
notion of a per-user identity.

```yaml
- identifier: staging-jira-token
  categories: [api-token]
  token: ${VAULT:/secret/staging/jira/service-token}
```

[`PersonalApiTokenCredential`](../../../automojo-credentials/source/packages/automojo/credentials/personalapitokencredential.py)
— a token bound to a username.  Used for personal access tokens
where the username appears in audit logs (Atlassian, Jira, Bitbucket,
many SaaS APIs).

```yaml
- identifier: staging-jira-personal
  categories: [personal-api-token]
  username: ci-bot@company.com
  token: ${VAULT:/secret/staging/jira/ci-bot-pat}
```

[`SecretTokenCredential`](../../../automojo-credentials/source/packages/automojo/credentials/secrettokencredential.py)
— a token paired with the verification *endpoint* the consumer
must call.  Used for third-party verification services (hCaptcha,
reCAPTCHA, similar challenge-response APIs) where the token only
makes sense in the context of a particular service URL.

```yaml
- identifier: hcaptcha-secret
  categories: [secret-token]
  endpoint: https://hcaptcha.com/siteverify
  token: ${VAULT:/secret/services/hcaptcha-secret}
```

### Cloud-provider credentials

Two cloud-specific shapes.

[`AwsAccessKeyCredential`](../../../automojo-credentials/source/packages/automojo/credentials/awsaccesskeycredential.py)
— the AWS access-key-id + secret-access-key pair.

```yaml
- identifier: ci-runner-aws
  categories: [aws-access-key]
  keyid: AKIA...
  secret: ${VAULT:/secret/aws/ci-runner-secret}
```

[`AzureClientSecretCredential`](../../../automojo-credentials/source/packages/automojo/credentials/azureclientsecretcredential.py)
— Azure application credential (tenant id + client id + client
secret), used for non-interactive Azure access.

```yaml
- identifier: staging-azure-client
  categories: [azure-client-secret]
  tenant_id: 11111111-1111-1111-1111-111111111111
  client_id: 22222222-2222-2222-2222-222222222222
  client_secret: ${VAULT:/secret/azure/staging-app-secret}
```

GCP service-account credentials don't have a shipped class today —
this is one of the natural extensions a reader will likely add.
The pattern is the same: a new class inheriting from
`BaseCredential`, a new category tag, a new dispatch branch in
the `CredentialManager`.

### Generic asymmetric keys

[`PublicKeyCredential`](../../../automojo-credentials/source/packages/automojo/credentials/publickeycredential.py)
— a generic public/private keypair, not tied to any particular
protocol.  Used for application-level signing (JWT signing,
artifact signing, code-signing keypairs).

```yaml
- identifier: jwt-signing-key
  categories: [public-key]
  public: |
    -----BEGIN PUBLIC KEY-----
    MIIBIjANBgkqhkiG9w0BAQ...
    -----END PUBLIC KEY-----
  private: ${VAULT:/secret/jwt/staging-private-key}
```

When the keypair is specifically for SSH, use `SshCredential`
instead — `SshCredential` understands the SSH-specific concerns
(agent, keypasswd, primitive mode) that `PublicKeyCredential`
doesn't model.

### GitHub-specific credentials

GitHub has two shipped classes because it has two distinct
authentication flows.

[`GithubTokenCredential`](../../../automojo-credentials/source/packages/automojo/credentials/githubtokencredential.py)
— a personal access token (PAT) with a username.  GitHub doesn't
verify the username for PATs, so the field defaults to `"git"`; it
exists for logging and tracking.

```yaml
- identifier: ci-bot-github
  categories: [github-token]
  username: ci-bot
  token: ${VAULT:/secret/github/ci-bot-pat}
```

[`GithubAppCredential`](../../../automojo-credentials/source/packages/automojo/credentials/githubappcredential.py)
— GitHub Apps authenticate using a private key (or list of keys
for rotation) signed against the App's identifier.  The credential
holds either keyfiles, raw keys, or both, with optional passphrase.

```yaml
- identifier: ci-github-app
  categories: [github-app]
  keyfiles:
    - /run/secrets/github/app-current.pem
    - /run/secrets/github/app-previous.pem      # for grace-period rotation
  keypasswd:
    - ${VAULT:/secret/github/app-current-passphrase}
    - ${VAULT:/secret/github/app-previous-passphrase}
```

The keyfiles are read at construction time and held in
`_loaded_keys`; the constructor raises `ConfigurationError` if a
declared keyfile is missing on disk.  This is the only credential
class that does *I/O at construction time* — a deliberate exception
because a GitHub App that can't read its key is a credential that
can never authenticate, and failing loudly at load is preferable to
failing silently at first use.

### WiFi networks

[`WifiChoiceCredential`](../../../automojo-credentials/source/packages/automojo/credentials/wifichoicecredential.py)
— a list of SSID+password pairs the consumer can iterate through,
trying each in turn.  Used for IoT / embedded / mobile-device
automation where the client may move through several networks
during a run.

```yaml
- identifier: lab-wifi
  categories: [wifi-choice]
  networks:
    - ssid: lab-floor-3
      password: ${VAULT:/secret/wifi/lab-floor-3}
    - ssid: lab-floor-4
      password: ${VAULT:/secret/wifi/lab-floor-4}
    - ssid: guest
      password: ${VAULT:/secret/wifi/guest}
```

This is an unusual shape — most credentials are single-value, this
one is a list of value-sets.  It exists because the alternative
(eleven separate credentials, one per SSID) loses the *intent*
that they're alternatives.  The consumer's logic is "try each in
order until one connects"; a single credential captures that
semantics directly.

## A complete worked example

Here is a credentials file matching the staging-east scenario from
Chapters 4–8.  Every reference these earlier chapters made — to
`ci-runner-aws`, `ci-runner-ssh`, `staging-db-rw`,
`staging-snowflake-rw`, etc. — resolves to an entry below:

```yaml
# credentials-staging-east.yaml
credentials:

    # SSH credential for the runner pool (referenced by Landscape
    # apod entries and by the Specification's CloudVenue)
    - identifier: ci-runner-ssh
      categories:
        - ssh
        - runner-fleet
      username: runner
      keyfile: /run/secrets/ssh/ci-runner-rsa
      keypasswd: ${VAULT:/secret/ssh/ci-runner-rsa-passphrase}
      role: priv

    # AWS credential for provisioning the runner pool (referenced by
    # the CloudVenue in chapter 4's Specification)
    - identifier: ci-runner-aws
      categories:
        - aws-access-key
        - cloud-provisioning
      keyid: AKIA...
      secret: ${VAULT:/secret/aws/ci-runner-secret}
      role: priv

    # Database credentials (referenced by chapter 8's Data Profiles)
    - identifier: staging-db-rw
      categories:
        - basic
        - app-tier
      username: app_user
      password: ${VAULT:/secret/staging/db/app-rw}
      role: priv

    - identifier: staging-db-readonly
      categories:
        - basic
        - reporting-tier
      username: app_readonly
      password: ${VAULT:/secret/staging/db/app-ro}
      role: readonly

    - identifier: staging-redis-default
      categories:
        - basic
        - app-tier
      username: default
      password: ${VAULT:/secret/staging/redis/default}
      role: priv

    # Snowflake credential (Snowflake auth uses a basic-shape credential)
    - identifier: staging-snowflake-rw
      categories:
        - basic
        - analytics-tier
      username: ci_loader
      password: ${VAULT:/secret/staging/snowflake/ci-loader}
      role: priv

    # CouchDB credential
    - identifier: staging-couchdb-rw
      categories:
        - basic
        - app-tier
      username: app_couchdb
      password: ${VAULT:/secret/staging/couchdb/app-rw}
      role: priv

    # GitHub PAT for fetching repos as part of the orchestration
    - identifier: ci-bot-github
      categories:
        - github-token
        - source-control
      username: ci-bot
      token: ${VAULT:/secret/github/ci-bot-pat}
      role: priv
```

Eight credential entries; six distinct credential types
represented (`SshCredential`, `AwsAccessKeyCredential`,
`BasicCredential` × 5, `GithubTokenCredential`); every secret value
is a `${VAULT:...}` reference rather than an inline literal.

The `${VAULT:...}` syntax is **not part of the shipped
`automojo-credentials` package**.  The shipped package treats the
right-hand-side strings as opaque values — so a literal
`password: hunter2` would load and work.  The `${VAULT:...}`
expansion is a *resolution discipline* the next chapter establishes:
the consumer-supplied resolver inspects each loaded credential and
materialises any reference syntax against a real backend.  This
chapter shows the syntax in worked examples to anchor the discipline
in the form chapter 10 will give it.

## Composition with the rest of the categories

Credentials are referenced — never embedded — by every other category:

- **Specifications** (chapter 4): `CloudVenue.credential` and
  `CloudResourceProfile.credential` carry credential identifiers.
- **Landscapes** (chapter 5): each apod client and infrastructure
  service carries a `credentials` list of identifiers.
- **Data Profiles** (chapter 8): every profile carries a
  `credential` identifier.

Three different layers, all reaching into the same identifier
namespace.  This means a single credential record — say,
`ci-runner-aws` — appears once in the credentials file and is
referenced from the venue (for provisioning) and from the runtime
(for accessing the resulting EC2 instances).  Rotating the AWS
secret is a one-place change.

The Topology category (chapter 6) is the exception — it doesn't
reference credentials directly, because Topology is a relationship
layer rather than an access layer.  The credentials needed to act
on a Topology come from the Landscape entries the Topology references.

## The `CredentialManager` and lookup-by-identifier

The runtime equivalent of the Data Profiles' `DataProfileManager` is
the
[`CredentialManager`](../../../automojo-credentials/source/packages/automojo/credentials/credentialmanager.py).
Same pattern, different domain:

```python
from automojo.credentials.wellknown import CredentialManagerSingleton

manager = CredentialManagerSingleton(...)
ssh_cred = manager.lookup_credential("ci-runner-ssh")
# ssh_cred is now an SshCredential with typed properties
# ssh_cred.username, ssh_cred.keyfile, ssh_cred.keypasswd
```

The lookup returns the typed object.  Lookups against missing
identifiers raise `ConfigurationError` with the source URIs in the
message, so the engineer can see where to add the missing
credential.  No silent `None` returns; loading and lookup both fail
loudly.

The dispatch logic inside `CredentialManager.load_credentials`
mirrors the Data Profiles dispatch from chapter 8 — walk the list,
match the category tag, instantiate the right concrete class.
Application-defined category tags (`runner-fleet`, `app-tier`) are
preserved on the instance so orchestration code can filter by them
later.

## Adding a new credential type

When the eleven shipped types don't cover a new authentication
shape, the extension recipe is short:

1. **Add a new category tag** the rest of the system can recognise.
   For a hypothetical `gcp-service-account` credential, the tag is
   `gcp-service-account`.

2. **Subclass `BaseCredential`.**  Add a class attribute or
   constructor check that requires the new category tag.  Define
   the new fields the credential needs (the GCP service-account
   JSON, the project ID, etc.).  Implement a `validate` classmethod
   for load-time checks.

3. **Add a dispatch branch in `CredentialManager.load_credentials`.**
   This is the one place in the package that knows about the new
   class.

4. **Document the new shape** in the project's credential schema
   reference.

The new type lives alongside the eleven shipped ones with no
special treatment.  Categories are open; the framework does not
constrain you to the shipped set.

## What credentials is *not*

Three things to keep out of credentials, every one of which a team
will be tempted to put in:

### Not access-control policy

A credential says "here is the material to authenticate."  It does
*not* say "this credential may access service X but not service Y."
Access-control policy is a property of the *resource* (or of an
external policy engine like AWS IAM, OPA, Azure RBAC), not of the
credential.

A credential that's tagged `[basic, readonly-tier]` is metadata
that helps your code *select* the right credential; it is not
enforcement.  The actual readonly access is enforced by the
database, not by the credential file.

### Not a secrets store

A credentials file with `password: hunter2` inline is a credentials
file pretending to be a secrets store.  It is not.  The shipped
`automojo-credentials` package will load such a file — the
constructor accepts any non-empty string for a password — but the
result is the same as if you had committed the password to the
repository.

The discipline is: every secret value is a *reference* to a
backend.  The format of the reference is your choice (the
`${VAULT:/path}` form used in this chapter is one convention; per-
backend formats like `vault://...` or `aws-secrets://...` work
equally well).  The chapter that follows is the discipline for
building the resolver that turns those references into materialised
values.

### Not a cache

Credentials are not the right place to cache short-lived auth
artefacts (OAuth access tokens, AWS STS tokens, Kerberos tickets).
The credential is the *long-lived* identity material; the
short-lived artefact is the *result of using* the credential.
Conflating the two leaks short-lived tokens into long-lived
storage and complicates rotation.

The pattern that works is: the credential is the long-lived seed;
the consumer derives short-lived auth artefacts at the point of
use; the artefacts live only in the consumer's process memory.
Chapter 11 addresses the rotation side of this.

## The "never log a credential" rule

Every credential class implements an `as_dict()` method that
serialises the credential to a dictionary — *including the secret
material*.  This is a deliberate trade-off: serialisation is
necessary for some legitimate operations (writing a credential
back to disk, transmitting it to a remote worker over an authenticated
channel), but it also means the secret can be accidentally exposed
through naive logging.

The discipline is unambiguous: **a credential never goes through a
logger, a print statement, an `f"{cred}"`, or a generic
exception's traceback.**  Two practices keep this in place:

1. **Override `__repr__` to elide the secret.**  Each shipped
   credential class should redact secret material in its `__repr__`
   so that `f"{cred}"` produces something safe.  (This is an audit
   finding from the credentials package research — most shipped
   classes do *not* currently override `__repr__`; an upcoming
   audit pass should fix this.)
2. **Audit the codebase for credential logging.**  Every
   `logger.info(cred)` or `print(cred)` is a candidate leak.
   Chapter 11 covers the audit pattern in detail.

## Summary

- `automojo-credentials` is a **pure vocabulary layer** — eleven
  typed credential classes with three universal fields
  (`identifier`, `categories`, `role`) and category-specific
  extensions.
- Every credential is dispatched at load time from the *first
  matching tag* in its `categories` list.  Application-defined
  tags coexist with the framework-required type tag.
- The eleven shipped types are: `BasicCredential`, `SshCredential`,
  `ApiTokenCredential`, `PersonalApiTokenCredential`,
  `SecretTokenCredential`, `AwsAccessKeyCredential`,
  `AzureClientSecretCredential`, `PublicKeyCredential`,
  `GithubTokenCredential`, `GithubAppCredential`,
  `WifiChoiceCredential`.  Adding a new type is a four-step recipe.
- All other categories — Specifications, Landscapes, Data Profiles
  — reference credentials by identifier; the credential file is the
  single place a credential's material lives.
- The shipped package treats secret values as opaque strings.  The
  *resolution* of a `${VAULT:...}`-style reference into a real
  secret is the next chapter's subject; it is the most substantial
  design exercise in the volume because the portfolio leaves it to
  the reader.
- Credentials are not access-control policy, not a secrets store,
  and not a cache for short-lived auth artefacts.  Each has its
  own place in the discipline.
- The "never log a credential" rule is the security discipline that
  keeps the typed-vocabulary approach safe.  Every credential class
  needs `__repr__` redaction; every codebase needs an audit
  guardrail against credential logging.

The next chapter is the resolution discipline — how to take the
`${VAULT:...}` references the chapter has shown and turn them into
materialised credentials at runtime, with backends the reader
designs to fit their environment.
