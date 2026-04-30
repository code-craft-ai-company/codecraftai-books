# Chapter 15 — The Local-Dev Tax and How Typed Configs Cancel It

> *The `.env.local` file is a confession.  Every team that has one
> is admitting that their pipeline depends on configuration the
> codebase doesn't name.*

The previous chapter built the proposed profile mechanism.  This
chapter applies it to a specific operational symptom: the
*local-dev tax* — the accumulated friction every developer pays on
a fresh checkout to get the project actually running.  Every piece
of that friction is a missing typed configuration.  This chapter
walks through the common pieces, names what each one is hiding,
and shows the migration path under the chapter-14 profile system.

The end state the chapter describes is the *boring* fresh
checkout: clone the repository, run one command, the test suite
passes.  No tribal knowledge, no README's "you might also need
to..." section, no half-day onboarding ritual.  Just code, profile,
results.

## The shape of the tax

A fresh checkout of a moderately-mature Python project typically
exposes a developer to seven or eight distinct sources of local-dev
tax.  Each has its own recognisable name and its own characteristic
failure mode.

### `.env.local` files

The most common form.  A file (committed or not) named `.env`,
`.env.local`, `.env.example` that contains a list of `export
NAME=value` lines.  The application reads them via a library
(`python-dotenv`, `direnv`'s shell hook, the developer's manual
`source .env.local`).

```bash
# .env.example
DATABASE_URL=postgresql://localhost:5432/dev
REDIS_URL=redis://localhost:6379/0
SECRET_KEY=changeme
LOG_LEVEL=DEBUG
GITHUB_TOKEN=
SLACK_WEBHOOK_URL=
```

What the file is *hiding*:

- **A landscape entry** for the local-dev database (`localhost:5432`).
- **A landscape entry** for the local-dev Redis.
- **A credential** for the database (the password, currently empty
  because the local Postgres is set up with trust auth).
- **A runtime configuration** for the log level.
- **A credential reference** for `GITHUB_TOKEN` (with no
  resolution discipline — the developer is expected to paste their
  PAT into the file).
- **An infrastructure-service entry** for Slack notifications.

The migration: each piece of content moves into the appropriate
typed configuration file from chapters 4–9.  The file disappears.

### `direnv` and `.envrc`

A more sophisticated cousin of `.env.local`.  The developer
installs `direnv`; when they `cd` into the project directory, a
`.envrc` script auto-runs and exports a set of environment
variables for the shell:

```bash
# .envrc
export DATABASE_URL=postgresql://localhost:5432/dev
export REDIS_URL=redis://localhost:6379/0
source_up_if_exists  # cascade up the directory tree
use python 3.11      # auto-activate the project's Python
```

The same content as `.env.local`, plus a Python-version selection
on top.  The migration is the same: every `export` becomes a typed
configuration entry.  The Python version is a runtime configuration
field (not currently shipped in the runtime YAML schema, but a
natural addition).

### `.tool-versions` (asdf, mise)

A file declaring which versions of which tools the project uses:

```
# .tool-versions
python 3.11.7
nodejs 20.10.0
poetry 1.7.1
```

Tool-version selection is *legitimately* a local-dev concern —
it is about how the developer's *machine* is set up, not about
how the orchestration runs.  Most teams should keep this file
checked in.

What it is *not* hiding: nothing significant.  A `.tool-versions`
file is closer to a `pyproject.toml`'s Python-version constraint
than to an environment-variable export — it informs the developer
of the expected toolchain rather than parameterising the
orchestration.

The migration in this case is partial: keep the file as a
toolchain manifest; do not let it grow into an environment-variable
substitute.  A `.tool-versions` file with five tool versions is
fine.  A `.tool-versions` file with `DATABASE_URL` in it is the
same `.env.local` mistake in fresh costume.

### `make setup`

An imperative setup script the developer runs once after checkout:

```makefile
# Makefile
setup:
	./scripts/install-system-deps.sh
	poetry install
	./scripts/create-local-db.sh
	./scripts/seed-test-data.sh
	./scripts/generate-ssh-keys.sh
	cp .env.example .env.local
	@echo "Now edit .env.local with your GitHub token"
```

What `make setup` is hiding:

- **A specification** for the local environment (which system
  packages, which Python version, which database, which seed
  data).
- **A landscape entry** that says "the local-dev landscape includes
  these databases, these mock services."
- **A credential bootstrap** (the SSH keys, the `.env.local`
  template).
- **A README** the developer is expected to read after the
  imperative script completes.

The migration: the system-package and Python-version steps stay
imperative (they're machine-setup concerns).  Everything else
becomes typed configuration plus the chapter-14 profile machinery.

### The README's "you might also need to..." section

The plaintext resting-place of every workaround the team has
accumulated:

> *Fresh checkout?  You'll need:*
> *— Python 3.11 (use pyenv or asdf)*
> *— Postgres running on port 5432 (we use Postgres.app on
> macOS; Linux folks use Docker)*
> *— A GitHub PAT with `read:repo` scope*
> *— An entry in `/etc/hosts` for `dev.local`*
> *— Trust the local SSL cert (instructions below)*
> *— Run `make setup` (might fail on Apple Silicon; if so, see
>   issue #234)*
> *— Edit `.env.local` to add your PAT*
> *— Sign in to AWS via SSO (token expires daily)*
> *— Connect the VPN (only required if you're testing the
>   payments service)*

This section is the *manifest of tribal knowledge*.  Every line
is a piece of state the codebase requires but doesn't *name*.  A
new hire works through it for a half-day; a returning developer
works through it on each major OS upgrade; a senior engineer
swears at it when the seventh dependency in the list has shifted
without anyone updating the README.

Each line is a missing typed configuration:

| Line | What it really is |
|---|---|
| Python 3.11 (use pyenv) | Toolchain manifest (`.tool-versions`) |
| Postgres on port 5432 | Landscape entry: a local-dev `staging-db` infrastructure service |
| GitHub PAT with `read:repo` | Credential entry: `local-github-pat`, scope tag `environment:local-dev` |
| `/etc/hosts` entry for `dev.local` | Specification: the local-dev environment requires this name resolution; provisioning step ensures it |
| Trust local SSL cert | Specification: the local-dev environment requires this cert to be in the system trust store |
| `make setup` (might fail on Apple Silicon) | Specification: machine-setup imperative; bug to fix in issue #234 |
| Edit `.env.local` | Migrating already; remove this step |
| AWS SSO sign-in (daily) | Credential resolution: the `local-dev` AWS credential reference resolves to a fetched-on-demand AWS SSO token |
| VPN if testing payments | Topology: the payments topology requires VPN connectivity; document the dependency in the topology file |

Once each line has a typed home, the README's section *empties
out*.  The remaining entries are the genuine machine-setup
imperatives (toolchain, system packages) — short, stable, and
honest about being one-time work.

## The cumulative cost

Every piece of local-dev tax has the same general failure mode:
*it works on the developer who set it up; it fails on the
next developer*.

The cumulative cost across a team:

- **Onboarding time.**  A new hire spends a half-day to a full
  day getting the project actually running.  Multiply by team
  growth rate.
- **OS-upgrade churn.**  When macOS releases a new major version,
  every developer pays the local-dev tax again because some part
  of `make setup` doesn't work on the new OS.
- **Debugging time.**  When a test fails locally but passes in
  CI, the developer's first few hours of debugging are spent
  ruling out local-dev environmental differences.  When the
  difference is in the README's tribal knowledge, the bug is
  *only reproducible by people who set up their environment the
  same way*.
- **CI ↔ local divergence.**  As the README grows, local-dev
  diverges from CI.  Eventually some classes of failure can only
  be reproduced in CI; the developer has to push commits to
  iterate.  The inner-loop velocity collapses.

The discipline of this chapter cancels each of these costs by
*moving the configuration into the codebase*.  Once the local-dev
profile names every piece of state the project depends on, a
fresh checkout is `clone + automojo profile activate local-dev`.

## The migration walk-through

The walk-through that follows takes the fictitious project from
the section above through migration.  Each step adds one typed
configuration; each step is independently committable; each step
makes the README's section shorter.

### Step 1 — Inventory

Walk every file that participates in local-dev configuration:
`.env.local`, `.envrc`, the README, the `Makefile`, the project's
`docker-compose.yaml`, every `setup-*.sh`.  Catalogue every
distinct configuration value.  Group them by category:

| Value | Source | Category |
|---|---|---|
| `DATABASE_URL=postgresql://localhost:5432/dev` | `.env.local` | Data Profile + Credential + Landscape |
| `REDIS_URL=redis://localhost:6379/0` | `.env.local` | Data Profile + Landscape |
| `SECRET_KEY=changeme` | `.env.local` | Credential |
| `LOG_LEVEL=DEBUG` | `.env.local` | Runtime |
| `GITHUB_TOKEN=...` | `.env.local` | Credential |
| `SLACK_WEBHOOK_URL=...` | `.env.local` | Credential + Landscape (infrastructure service) |
| `python 3.11` | `.tool-versions` | Toolchain manifest (stays) |
| Postgres on port 5432 | README | Landscape (infrastructure service) |
| `/etc/hosts` for `dev.local` | README | Specification |
| Local SSL cert trusted | README | Specification |
| AWS SSO daily token | README | Credential |
| VPN for payments | README | Topology |

The inventory is the chapter's most important step.  An
inventory that misses something will leave the README with a
residual line, which is the same kind of tribal knowledge the
discipline aims to eliminate.

### Step 2 — Author the local-dev configurations

For each typed configuration, author the file:

**`config/landscape/landscape-local-dev.yaml`** — the local-dev
landscape:

```yaml
environment:
    label: local-dev

apod:
    clients:
        - deviceType: network/client-localhost
          name: developer-laptop
          host: 127.0.0.1
          role: runner
          credentials:
              - local-runner
          features:
              parallelism_max: 4
          skip: false

infrastructure:
    services:
        - serviceType: database/postgres
          name: staging-db          # same identifier as production
          host: 127.0.0.1
          port: 5432
          database: dev
          credentials:
              - local-postgres

        - serviceType: cache/redis
          name: staging-redis        # same identifier as production
          host: 127.0.0.1
          port: 6379
          credentials:
              - local-redis

        - serviceType: web/webhook
          name: slack-notifications  # same identifier as production
          endpoint: ${PROMPT:Slack webhook URL}
          credentials:
              - local-slack-webhook
```

**`config/dataprofiles/dataprofiles-local-dev.yaml`** — the
profiles:

```yaml
dataprofiles:
    - identifier: staging-db-rw       # same identifier as production
      category: basic-tcp
      host: 127.0.0.1
      port: 5432
      dbtype: postgres
      dbname: dev
      credential: local-postgres

    - identifier: staging-redis        # same identifier as production
      category: basic-tcp
      host: 127.0.0.1
      port: 6379
      dbtype: redis
      credential: local-redis
```

**`config/credentials/credentials.yaml`** — local-dev section
appended:

```yaml
credentials:
    # ... existing CI / staging / prod entries ...

    - identifier: local-postgres
      categories: [basic, environment:local-dev]
      username: dev
      password: ${KEYCHAIN:com.company.app/local-postgres}

    - identifier: local-redis
      categories: [basic, environment:local-dev]
      username: default
      password: ${PROMPT:Local Redis password}

    - identifier: local-runner
      categories: [ssh, environment:local-dev]
      username: $USER
      keyfile: ${FILE:~/.ssh/id_ed25519}
      keypasswd: ${KEYCHAIN:com.company.app/ssh-id-ed25519}

    - identifier: local-github-pat
      categories: [github-token, environment:local-dev]
      username: $USER
      token: ${KEYCHAIN:com.company.app/github-pat}

    - identifier: local-aws-sso
      categories: [aws-access-key, environment:local-dev, ttl:3600]
      keyid: ${AWSSSO:role/Developer/AccessKey#keyid}
      secret: ${AWSSSO:role/Developer/AccessKey#secret}

    - identifier: local-slack-webhook
      categories: [api-token, environment:local-dev]
      token: ${KEYCHAIN:com.company.app/slack-webhook}

    - identifier: local-app-secret-key
      categories: [api-token, environment:local-dev]
      token: ${LITERAL:changeme}     # explicit; not a real secret
```

**`config/runtime/runtime-local-dev.yaml`** — the runtime
configuration:

```yaml
breakpoints: []
debugger: debugpy
logging:
    levels:
        console: DEBUG
        logfile: DEBUG
parallelism:
    max_workers: 4
timeouts:
    default_seconds: 300       # generous for breakpoints
property_test_caps:
    max_examples: 5            # fast feedback
paths:
    output: ./.output
version: 1.0.0
```

**`config/specs/spec-local-dev.yaml`** — the specification for the
local environment:

```yaml
environment:
    venues:
        - name: localhost
          provider: local
          credential: null

groups:
    - name: local-dev
      description: "Local-dev environment requirements"

profiles:
    - name: local-postgres
      description: "Local Postgres install"
      parameters:
          provider: brew
          formula: postgresql@15
          port: 5432
          database: dev

    - name: local-redis
      description: "Local Redis install"
      parameters:
          provider: brew
          formula: redis
          port: 6379

    - name: local-hosts-entry
      description: "Local /etc/hosts entries"
      parameters:
          entries:
              - hostname: dev.local
                address: 127.0.0.1

    - name: local-trust-cert
      description: "Local SSL cert in system trust store"
      parameters:
          cert_path: ./certs/local-dev.crt

apod:
    devices:
        - identifier: local-postgres
          category: brew-service
          group: local-dev
          profile: local-postgres
          role: database
        - identifier: local-redis
          category: brew-service
          group: local-dev
          profile: local-redis
          role: cache
        - identifier: local-hosts
          category: hosts-file
          group: local-dev
          profile: local-hosts-entry
          role: dns
        - identifier: local-cert
          category: trust-store
          group: local-dev
          profile: local-trust-cert
          role: tls
```

The specification is consumed by `automojo provisioning apply`,
which (per chapter 4's discipline) is idempotent.  The first run
installs Postgres, Redis, edits `/etc/hosts`, and trusts the
cert.  Subsequent runs are no-ops if the state is already correct.

**`config/profiles/local-dev.yaml`** — the profile that ties
everything together:

```yaml
profile:
    extends: base
    name: local-dev
    description: Single-host developer workstation

    select:
        landscape: local-dev
        topology: local-dev
        runtime: local-dev
        credentials: local-dev
        dataprofiles: local-dev
```

### Step 3 — Wire the application code

Code that previously read environment variables now reads typed
configurations:

```python
# Before
import os
DATABASE_URL = os.environ["DATABASE_URL"]

# After
from automojo.dataprofiles.wellknown import DataProfileManagerSingleton
from automojo.credentials.wellknown import CredentialManagerSingleton

profile = DataProfileManagerSingleton().lookup_profile("staging-db-rw")
cred = CredentialManagerSingleton().lookup_credential(profile.credential)
DATABASE_URL = f"postgresql://{cred.username}:{cred.password}@{profile.host}:{profile.port}/{profile.dbname}"
```

The migration is incremental: one configuration value at a time.
Until every reader has migrated, the team can keep the env-var
fallback in place via the chapter-10 `EnvBackend` (which lets a
credential reference resolve from `$DATABASE_URL` if it isn't
otherwise resolvable).  This is the same migration shim from
chapter 7.

### Step 4 — Replace `make setup` with `automojo provisioning apply`

The imperative setup script becomes a wrapper around the spec:

```makefile
# Makefile (after migration)
setup:
	# Toolchain (the legitimate machine-setup imperative)
	asdf install
	poetry install

	# Everything else is declarative
	AM_PROFILE=local-dev automojo provisioning apply --spec config/specs/spec-local-dev.yaml
```

Two lines of toolchain installation; one line of declarative
provisioning.  The previous five-line imperative chain — install
deps, create db, seed data, generate keys, copy `.env.example` —
is gone, replaced by the spec-driven equivalent.

### Step 5 — Activate the profile

The new fresh-checkout sequence is two commands:

```bash
git clone <repo>
cd <repo>
make setup
AM_PROFILE=local-dev automojo testbase test run
```

That's it.  No `.env.local`, no README scavenger hunt, no
half-day onboarding ritual.  The developer's machine has the
toolchain (from `asdf install` and `poetry install`); the local
infrastructure is provisioned (from the spec); the configurations
are typed and per-environment (from the profile); the test suite
runs.

### Step 6 — Empty the README

The README's "you might also need to..." section, after migration:

> *Fresh checkout?*
> *— Install asdf or mise (one-time machine setup)*
> *— Run `make setup`*
> *— Run `automojo testbase test run --profile local-dev`*

Three lines.  None of them are tribal knowledge.  None of them
will silently rot when the seventh dependency shifts.  Each line
is *small enough to keep correct*.

## The pattern is universal

The migration above is concrete to one project, but the pattern
generalises:

1. **Inventory** every piece of configuration state the project
   currently reads from environment variables, `.env` files,
   shell scripts, the README, or imperative setup logic.
2. **Group** the inventory by category (Specifications,
   Landscapes, Topologies, Runtimes, Credentials, Data Profiles).
3. **Author** the local-dev configuration for each category
   under `config/<category>/<file>-local-dev.yaml`.
4. **Author** the local-dev profile that selects them as a
   bundle.
5. **Author** the local-dev specification that captures the
   one-time machine-setup imperatives.
6. **Migrate** application code to read typed configurations
   instead of environment variables.  Use the chapter-10
   `EnvBackend` as a transitional shim during the migration.
7. **Replace** `make setup`'s imperative steps with the
   specification-driven equivalent.
8. **Empty** the README of tribal knowledge; what remains is the
   genuine one-time machine-setup imperative (toolchain plus
   `make setup`).

The pattern is the same regardless of project size.  A small
project might compress steps 3 and 4 into one combined
configuration; a large project might split each category into
multiple files for clarity.  The discipline is the inventory and
the elimination of tribal-knowledge prose, not the file-count
pedantry.

## What about existing developers' workflows?

A common objection: *"the migration breaks every developer's
existing setup."*  Three responses to this objection.

### Existing setups keep working during migration

The chapter-10 `EnvBackend` and the chapter-14 profile system are
both *opt-in*.  When `AM_PROFILE` is unset, the system runs with
the existing shipped behaviour — every existing `os.environ.get(...)`
in the codebase still works.  The migration can run alongside
the existing setup for as long as needed.  Each developer migrates
to the profile system on their own schedule.

### The migration produces parity, not divergence

The local-dev profile *replaces* the `.env.local` content.  The
values in the profile are the same values the developer had in
`.env.local` (or in their direnv, or in their shell history).  No
behaviour change — only a behaviour *naming* change.  The first
run of `AM_PROFILE=local-dev automojo testbase test run` produces
the same results as the previous `pytest` invocation against
`.env.local`.

### The benefits accrue to *new* developers immediately

Existing developers see incremental improvement as the migration
progresses.  *New* developers see the full benefit on day one —
they never set up the old way at all.  A team that's hiring rapidly
sees the steepest payoff: every new hire's onboarding time drops
by a half-day, and the team's senior engineers stop being asked
the same setup questions every week.

## What this discipline is *not*

Three boundaries to keep clear:

### Not a replacement for tools that work

The chapter is critical of `.env.local` as a *content* mechanism.
It is not critical of file-based configuration as a *concept* —
the typed configuration files this chapter proposes are
themselves files in version control.  The difference is
*structure*: `.env.local` is untyped key-value pairs; a typed
configuration file is a structured document with a known schema.

Tools like `direnv` and `asdf` that solve real local-dev problems
(automatic Python-version selection, project-scoped tool
shimming) remain useful and complementary.  The discipline asks
those tools to stop carrying *application configuration* in their
files; the toolchain-management uses are unaffected.

### Not "one tool to rule them all"

The chapter has not proposed that every team adopt
`automojo-*` packages exclusively.  The *discipline* of typed
configuration is portable; teams that prefer different
implementations of the same pattern (Pydantic Settings, dynaconf,
Hydra) can apply the same discipline using their preferred
toolkit.  What matters is that configuration is structured, named,
and per-environment-scoped — not whose YAML loader does the
loading.

### Not a security boundary

The local-dev profile loads credentials and runs them through the
chapter-10 resolver.  This is *not* a security boundary against a
compromised developer machine.  A developer whose laptop has been
compromised can read every secret that resolves into their local
profile; the chapter's `${KEYCHAIN:...}` and `${PROMPT:...}`
backends are *operational* conveniences, not security defences.

For real protection of production secrets, the discipline of
chapter 12 applies: production secrets are never resolvable from
local-dev profiles, because the resolver in local-dev does not
have the production Vault token.  The local-dev profile *exists*
so developers don't accidentally try to use production
credentials; it is not a fence around them if the developer
attempts to.

## Summary

- The **local-dev tax** is the friction every developer pays on
  fresh checkout.  It manifests as `.env.local` files, `direnv`
  configs, `make setup` scripts, and the README's "you might
  also need to..." section.
- Every piece of the tax is a missing typed configuration.  The
  inventory groups each piece into one of the six categories.
- Migration produces a fresh-checkout flow of three commands:
  install the toolchain, run `make setup` (now declarative),
  activate the local-dev profile.  No tribal knowledge, no
  scavenger hunt, no half-day ritual.
- The migration is incremental: each configuration value can
  migrate independently using the chapter-10 `EnvBackend` as a
  transitional shim.  Existing developer workflows keep working
  during the migration; new developers see the full benefit on
  day one.
- The pattern is universal: inventory, group, author per category,
  author the profile, replace imperative setup with declarative
  specification, migrate code, empty the README.
- The discipline is *not* a replacement for tools that work
  (direnv, asdf for what they're good at), *not* "one toolkit
  exclusively," and *not* a security boundary against a
  compromised local machine.

The next chapter is the operational payoff of this discipline:
when a CI failure shows up that doesn't reproduce locally, the
captured profile from the failing CI run lets the developer
replay the exact configuration on their laptop and debug the
difference in their IDE.
