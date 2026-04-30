# Chapter 12 — Credentials Across Environments

> *The same credential vocabulary works in local-dev, in CI, in
> staging, in prod.  What changes is the backend that resolves it.
> The pipeline does not know which backend resolved its
> credentials.*

This chapter closes Part III and bridges to Part IV.  Chapter 9
established the vocabulary, chapter 10 designed the resolver,
chapter 11 made the credential lifecycle operational.  This chapter
asks the question those three chapters set up: *how does the same
credential discipline survive the trip from a developer's laptop
through CI through staging into production?*

The answer is the discipline of this chapter: the credential file
is identical in every environment.  The credential identifiers are
identical.  The resolver interface is identical.  The audit logger
is identical.  What changes — *and only what changes* — is the
backend that materialises the secret values.

This is the credential-specific application of a broader principle.
Part IV will generalise it to every configuration category: same
files, same identifiers, same code paths, different *resolution*.
But credentials are the right place to introduce the discipline,
because credentials are where the security stakes are highest and
the cost of getting it wrong is most concrete.

## The four environments

A piece of automation visits four environments in its lifetime.

| Environment | Audience | Backend choices |
|---|---|---|
| **local-dev** | The engineer writing or debugging the code on their workstation | Keychain, file-on-disk, prompt |
| **integration-CI** | The CI pipeline running on every merge request | Vault, AWS Secrets Manager, file-from-K8s-Secret |
| **staging** | A pre-production environment running the same orchestration against real services | Vault with stricter access controls, audit logging always on |
| **prod-readonly** | An on-call engineer reading production state for incident debugging | Vault with read-only and audit-only credentials, mandatory time-bound access |

The engineer's debugging session and the CI pipeline and the
staging deployment and the production debugger all run *the same
code*.  They invoke the same orchestration with the same
credential references.  The difference is which `CredentialResolver`
the orchestration's bootstrap built, and that difference is one
profile-keyed function.

## What stays the same

Five things do not change between environments:

### The credentials YAML file

The same `credentials.yaml` — same identifiers, same categories,
same field names, same `${SCHEME:path}` references — is the
canonical credential declaration in every environment.  A new
credential added for a local-dev test run automatically becomes
available in CI and staging, because the file is the same file.

This is structurally identical to the landscape and topology
discipline from chapters 5 and 6: the *file* is the source of
truth; the *resolution* is the per-environment customisation.

In practice this often means the credentials file is checked into
the repository — not the secret values, only the credential
*identifiers and references*.  The references say `${VAULT:/secret/...}`
or `${FILE:/run/secrets/...}`; the actual secrets live in the
backends those references point at.

### The credential identifiers

Once an identifier is declared (`staging-db-rw`,
`ci-runner-ssh`, `staging-snowflake-rw`), it has the same meaning
in every environment.  Code that calls
`CredentialManagerSingleton().lookup_credential("staging-db-rw")`
gets the same *type* of credential everywhere.  The username, the
field shape, the role — all identical.  Only the secret value
varies.

Identifier stability is the single most important property of the
credential discipline.  When an identifier means one thing in
local-dev and a different thing in CI, the discipline has been
broken — and the next bug from that breakage will be hard to
reproduce.

### The resolver interface

The `CredentialResolver` class from chapter 10 is the same class
in every environment.  It takes a list of `CredentialBackend`
instances and walks the credentials dict.  The walking is identical;
the parsing of `${SCHEME:path}` references is identical; the
dispatch of references to backends is identical.

What changes is *which backends are passed in*.  The resolver
itself is environment-agnostic.

### The audit logger

The audit trail format from chapter 11 — the structured event with
identifier, calling context, resolution status — is the same shape
in every environment.  The audit *sink* may differ (a file in
local-dev, a centralised logging service in CI, a SIEM in
staging/prod) but the events flow into them with the same fields.

This means a developer reproducing a CI failure locally sees the
same event types they'd see in CI; the post-failure investigation
narrative reads consistently across environments.

### The "never log" defences

The four-layer defence from chapter 11 — `__repr__` redaction,
logging filter, exception hook, CI guardrails — runs in every
environment.  Local-dev is *especially* a place these matter: a
developer's laptop is not a hardened machine, the `~/.zsh_history`
is forever, and the casual `print(cred)` someone added during
debugging is the most common credential leak in any codebase.

The defences run in local-dev with the same vigilance as in
production.  They are not "production hardening" — they are
*always-on hygiene*.

## What changes

The differences between environments are confined to two places.

### The backend roster

The set of `CredentialBackend` instances passed to the resolver
differs per environment.  This is `build_resolver_for_environment`
from chapter 10:

```python
def build_resolver_for_environment(profile: str) -> CredentialResolver:
    if profile == "local-dev":
        return CredentialResolver([
            KeychainBackend(),
            FileBackend(),
            PromptBackend(),
        ])
    elif profile == "integration-ci":
        return CredentialResolver([
            VaultBackend(
                vault_addr=os.environ["VAULT_ADDR"],
                vault_token=os.environ["VAULT_TOKEN"],
            ),
            FileBackend(),  # for K8s-mounted secret files
        ])
    elif profile == "staging":
        return CredentialResolver([
            VaultBackend(
                vault_addr=STAGING_VAULT_ADDR,
                vault_token=fetch_vault_token_via_aws_iam(),
            ),
        ])
    elif profile == "prod-readonly":
        return CredentialResolver([
            VaultBackend(
                vault_addr=PROD_VAULT_ADDR,
                vault_token=fetch_short_lived_readonly_vault_token(
                    operator_email=os.environ["OPERATOR_EMAIL"],
                    ticket_id=os.environ["INCIDENT_TICKET_ID"],
                ),
            ),
        ])
    else:
        raise ConfigurationError(f"Unknown profile {profile!r}")
```

Three observations from this code:

- **The number of backends decreases** as the environment becomes
  more sensitive.  Local-dev has three backends with permissive
  fall-through; production has one backend with strict access
  control.  The discipline is to remove backends as you climb the
  ladder, not to add them.
- **The backend's *configuration* hardens** with the environment.
  The local-dev `VaultBackend` (if used) might point at a personal
  Vault instance; the production `VaultBackend` requires
  short-lived tokens fetched from AWS IAM, with operator identity
  and ticket ID required.  The same class; very different security
  posture.
- **The selection lives in one function.**  The rest of the
  orchestration code never branches on `profile`.  When the
  environment changes, only `build_resolver_for_environment`
  changes.

### Per-environment policy on the credential's category tags

Chapter 10 introduced host-scoping via category tags — a credential
tagged `[host-scope:staging-east]` only resolves successfully when
the active landscape is `staging-east`.  The same machinery
expresses environment-scoping:

```yaml
- identifier: prod-db-readonly
  categories:
    - basic
    - environment:prod
    - role:readonly
    - ttl:300
  username: ${VAULT:/secret/prod/db/readonly#username}
  password: ${VAULT:/secret/prod/db/readonly#password}
```

The `environment:prod` tag tells the resolver to refuse the
credential in any environment other than prod.  When local-dev
accidentally references `prod-db-readonly`, the resolver fails
loudly: *"Credential prod-db-readonly is scoped to environment 'prod'
but active environment is 'local-dev'."*

This is the most important environment-specific defence in the
chapter.  It is the structural answer to the question *"how do we
prevent a developer from accidentally connecting to production?"*
The answer is: the credential file declares the boundary, the
resolver enforces it, and the discipline is checkable in code review.

## The promotion path

A new credential added in local-dev should become available in CI
and staging without rewriting code.  The promotion path:

1. **Day 1 (local-dev).**  A developer adds an entry to
   `credentials.yaml`:
   ```yaml
   - identifier: local-redis
     categories: [basic, environment:local-dev]
     username: dev
     password: ${KEYCHAIN:com.company.app/local-redis}
   ```
   The credential resolves in their local-dev environment because
   the keychain backend is wired.  Other environments would refuse
   to resolve `${KEYCHAIN:...}` because they don't have a Keychain
   backend; the `environment:local-dev` tag prevents accidental
   activation.

2. **Day 7 (CI).**  The same credential is needed for CI testing.
   The developer adds a *parallel* entry — same identifier, but
   different `${SCHEME:path}` and different scope tag:
   ```yaml
   - identifier: ci-redis
     categories: [basic, environment:integration-ci]
     username: ci
     password: ${VAULT:/secret/ci/redis#password}
   ```
   And a CI-side Vault entry is created to hold the secret.

3. **Day 14 (staging).**  Same pattern, with staging's Vault path
   and `environment:staging` tag.

The same code references the credential by identifier-prefix or by
role tag; the resolver picks the right one based on the active
environment.  At no point did the application code change.  At no
point did the plumbing (Dockerfile, CI YAML, K8s manifest) change —
the AM_CREDENTIAL_NAMES environment variable still points at the
same credentials.yaml file.

This is the chapter 7 plumbing-vs-content principle applied to
credentials specifically: the plumbing is small and fixed (one
env var, one resolver-builder, one credentials file path), and
the content (the credentials and their environment-scoped scopes)
grows freely.

## Cross-environment isolation

The most important security property of the discipline is that a
credential from one environment can never accidentally work in
another.  Three layers enforce this:

### Backend-level isolation

The backend instances themselves are environment-scoped.  The
`VaultBackend` for local-dev points at a personal Vault.  The
`VaultBackend` for CI points at the team's CI Vault.  The two
Vaults have different network reachability, different access
controls, and (in well-run setups) different root keys.

A reference like `${VAULT:/secret/prod/db/readonly}` issued from a
local-dev resolver goes to the local-dev Vault and *misses* —
either with a "not found" error or a "permission denied" error.
The local-dev Vault doesn't have the production secrets.

### Token-level isolation

Even within a single Vault, the *token* the backend was instantiated
with limits what it can read.  A staging token cannot read prod
paths; a CI token cannot read on-call-bound paths.  The credentials
file can declare every credential the orchestration might want; the
backend's token controls which ones the current environment can
actually fetch.

This means a leaky credentials file (one that mentions production
secret paths) is not by itself catastrophic.  An attacker who
obtains the credentials file in local-dev still cannot read the
production secrets — they would also need a production Vault
token.

### Tag-level isolation

The `environment:<name>` category tag, enforced by the resolver, is
the third layer.  Even if the backend permitted the lookup and the
token had access, the resolver refuses to return the credential when
the tag mismatches.  This is defence-in-depth: a misconfigured
backend or a token with too much access is *still* caught by the
resolver before the credential reaches the application code.

The three layers are independent.  Defence-in-depth means that any
two failing simultaneously still leaves the third holding the line.

## Migration patterns

Four common operational patterns the discipline supports:

### Adding a new credential

The procedure described above.  Add the credential to the file
with a per-environment scope tag.  Create the backend entry
(Vault path, K8s Secret, etc.) for the environments that need it.
Test in local-dev first; promote up the ladder.

The migration is *additive* in every environment.  Removing a
credential is structurally similar — drop the file entry, drop the
backend entry, ensure no code still references the identifier.

### Rotating across environments

A credential rotation in production is a backend operation: change
the secret value at the Vault path, leave the credentials file
unchanged.  The resolver's TTL cache (chapter 11) picks up the new
value at the next resolution window.

When a credential rotates *and* the rotation needs to be applied to
multiple environments (e.g., a third-party API key rotated by the
provider — same key value applies everywhere), the procedure is to
update each environment's backend independently.  The credentials
file's references don't change.

### Decommissioning an environment

When a staging environment is retired, four things happen:

1. The credentials file entries with `environment:staging` are
   removed.
2. The Vault paths for staging are deleted.
3. The `build_resolver_for_environment("staging")` branch is
   removed.
4. CI YAML that set `AM_PROFILE=staging` (chapter 14) is updated.

Steps 2-4 are operational and reversible.  Step 1 is the *only*
code change — and it is a deletion of redundant entries, not a
restructuring of any active environment.

### Adding a new environment

Promotion of `staging` to `staging-east` and `staging-west` (one
environment becoming two regional siblings) is the inverse:

1. Duplicate the relevant entries in the credentials file with
   the new scope tags.
2. Provision the Vault paths for each new environment.
3. Add the new branches to `build_resolver_for_environment`.
4. The CI YAML grows two new profile values.

The application code remains untouched.  The orchestration code
remains untouched.  The plumbing layer was already prepared for
multiple values of `AM_PROFILE` per the chapter 7 fixed-interface
principle.

## What credentials across environments is *not*

Three boundaries to keep clear:

### Not multi-tenant

The discipline assumes the same *team* owns the credentials
across environments.  When a credential is shared across teams
(an inherited service account, a shared admin login), the
discipline does not address the team-vs-team coordination — that
is an organisational concern that lives outside the orchestration
layer.

The pattern that does work for multi-tenant credentials is to
treat each tenant as a separate environment with its own credential
identifiers — `tenant-acme-db-rw` and `tenant-globex-db-rw`
rather than a shared `tenant-db-rw`.  Each tenant's credentials
remain isolated through the same backend-and-tag-scoping machinery.

### Not shared accounts

The discipline strongly *discourages* shared accounts (one
credential identifier whose value is reused by every member of a
team).  Shared accounts undermine the audit trail — the calling
context records *who* invoked the lookup, but the underlying
credential records *only* "the team."

When a shared account is unavoidable (third-party SaaS that
charges per seat, legacy systems with no per-user credential
support), the credential file should declare the sharing
explicitly via a category tag (`shared:platform-team`) so the
audit trail at least records the team boundary.  The discipline
acknowledges the constraint without pretending it isn't a
weakness.

### Not migration off legacy environment variables on day one

Chapter 7 made the case for runtime profiles over environment
variables.  The transition is not instant.  During the transition,
the credentials file may have entries that resolve via the
`EnvBackend` — pulling values from existing CI environment
variables — alongside entries that resolve via Vault.  This is
acceptable as a *transitional* state.  The discipline is to mark
the env-backed entries with a category tag (`legacy:env-var`) so
they show up in regular audits and the team has visibility into
the remaining migration work.

The end-state has zero `legacy:env-var` tags.  Reaching it is a
function of the team's bandwidth, not the discipline's correctness.

## A complete worked example

Building on the credentials file from chapter 9 (the eight
staging-east credentials) and the resolver builder from chapter 10,
here is the complete cross-environment setup for the same project:

```yaml
# credentials.yaml — checked into the repository
credentials:

    # ============== local-dev =================
    - identifier: local-redis
      categories: [basic, environment:local-dev]
      username: dev
      password: ${KEYCHAIN:com.company.app/local-redis}

    - identifier: local-postgres
      categories: [basic, environment:local-dev]
      username: dev
      password: ${PROMPT:Local postgres password}

    # ============== integration-ci =================
    - identifier: ci-runner-ssh
      categories: [ssh, environment:integration-ci, runner-fleet, ttl:3600]
      username: runner
      keyfile: ${FILE:/run/secrets/ssh/ci-runner-rsa}
      keypasswd: ${VAULT:/secret/ci/ssh/ci-runner-rsa-passphrase}

    - identifier: ci-runner-aws
      categories: [aws-access-key, environment:integration-ci, ttl:900]
      keyid: ${VAULT:/secret/ci/aws/ci-runner#keyid}
      secret: ${VAULT:/secret/ci/aws/ci-runner#secret}

    # ============== staging-east =================
    - identifier: staging-db-rw
      categories: [basic, environment:staging-east, app-tier, ttl:600]
      username: app_user
      password: ${VAULT:/secret/staging-east/db/app-rw}

    - identifier: staging-db-readonly
      categories: [basic, environment:staging-east, reporting-tier, role:readonly]
      username: app_readonly
      password: ${VAULT:/secret/staging-east/db/app-ro}

    - identifier: staging-snowflake-rw
      categories: [basic, environment:staging-east, analytics-tier, ttl:1800]
      username: ci_loader
      password: ${VAULT:/secret/staging-east/snowflake/ci-loader}

    # ============== prod-readonly =================
    - identifier: prod-db-readonly
      categories: [basic, environment:prod, role:readonly, audit:required, ttl:300]
      username: ${VAULT:/secret/prod/db/readonly#username}
      password: ${VAULT:/secret/prod/db/readonly#password}
```

Three things visible in the file structure:

- **Identifier prefixes mirror the environment** (`local-`, `ci-`,
  `staging-`, `prod-`).  This is convention, not enforcement — the
  `environment:<name>` tag is what the resolver enforces — but the
  prefix makes the file readable at a glance.
- **TTL varies by environment.**  Local credentials have effectively
  no TTL (Keychain doesn't rotate).  CI credentials have short TTLs
  (15-60 minutes) because they roll on every job.  Staging
  credentials have longer TTLs (10-30 minutes).  Prod credentials
  have the shortest TTLs (5 minutes) because the audit cost of a
  stale lookup is highest there.
- **Prod credentials require `audit:required`.**  This tag is
  enforced by the resolver in prod environments — every prod
  credential lookup emits an audit event with operator-identity and
  ticket-id metadata.  Failing to log is a hard error.

The same `build_resolver_for_environment` from chapter 10 wires
each profile to its appropriate backends.  The orchestration code
that consumes credentials is unchanged across all four environments.

## Summary — and end of Part III

- Same credentials file, same identifiers, same resolver
  interface, same audit logger, same "never log" defences across
  every environment.  Only two things change: the backend roster
  and the per-credential scope tags.
- `environment:<name>` category tags express the per-credential
  scope; the resolver enforces them at lookup time as defence-in-
  depth alongside backend-level and token-level isolation.
- Cross-environment isolation has three layers: the backend's
  reachability, the token's permission scope, and the resolver's
  tag enforcement.  Any two failing leaves the third holding the
  line.
- The promotion path for a new credential is additive: declare the
  identifier in local-dev with a scope tag, add the per-environment
  parallel entries as the credential is needed in CI / staging /
  prod.  No application-code change at any step.
- Common operational patterns — adding, rotating, decommissioning,
  splitting an environment — are all supported as additive or
  deletional changes to the credentials file plus their backend
  counterparts.  The plumbing layer never changes.
- The discipline is *not* multi-tenant coordination, *not* a fix
  for shared-account weaknesses, and *not* an instant migration
  off env-var-based credentials.  Each is a separate concern.

This is the last chapter of Part III.  The credentials story now
has its full shape: a vocabulary (chapter 9), a resolver discipline
(chapter 10), a lifecycle (chapter 11), and an environment-spanning
strategy (chapter 12).

The next part of the volume — Part IV — generalises the
environment-ladder discipline to *every* configuration category.
What this chapter said about credentials specifically (same
vocabulary, different backends, scope tags for isolation, additive
promotion) becomes a uniform pattern across Specifications,
Landscapes, Topologies, Runtimes, Data Profiles, *and* Credentials.
The credential discipline was the right place to introduce the
pattern because the security stakes made it concrete; Part IV is
where the pattern earns its keep across the rest of the platform.
