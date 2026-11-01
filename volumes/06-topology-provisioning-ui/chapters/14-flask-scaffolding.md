# Chapter 14 — Web-Service Scaffolding with Flask

> **Note (October 2026).** `codecraftai-webservice-flask` is no longer part of the portfolio.  The Code Craft AI website moved to
> [`codecraftai-webworks`](https://github.com/code-craft-ai-company/codecraftai-webworks), which replaces the Flask patterns
> described here.  The chapter is kept for the design it records: security-aware blueprints, role-based route protection and
> multi-provider OAuth.  For new work, use `codecraftai-webworks`.

> *Where GitLab integration was the *consumer* side of REST,
> Flask scaffolding is the *server* side.
> `codecraftai-webservice-flask` provides the patterns the
> portfolio's website front-end uses: security-aware
> blueprints, role-based route protection, OAuth integration
> with eight providers, and SQLAlchemy ORM mappings.*

The portfolio includes a Flask scaffolding package.  It's
not a "framework" in the `codecraftai-*` sense — it's a
*pattern library* for building Flask web services that
follow the portfolio's conventions: security-aware routing,
multi-provider OAuth, SQLAlchemy ORM with auditable
timestamps.

This chapter is a *survey* rather than a deep walk.  A reader
building a web service against the portfolio's conventions
follows the patterns; a reader who just wants to *understand*
what's in the package can skim the chapter.

## What the package provides

```
codecraftai-webservice-flask/source/packages/ccai/webservice/xflask/
├── basesecurity.py         ← RouteSecurityManager
├── securityblueprint.py    ← SecurityBlueprint(Blueprint)
├── baseauthentication.py   ← OAuth provider enum + auth-check helpers
├── baseendpoints.py        ← signup / login / logout / OAuth callbacks
├── baseoperations.py       ← user CRUD, token management
├── baseorm.py              ← SQLAlchemy ORM models
├── basestate.py            ← Flask app state and configuration
├── basemessages.py         ← localised messages
├── basevalidation.py       ← input validation helpers
└── ...
```

Ten modules.  Each addresses one concern: security routing,
auth, endpoints, ORM, app state.  A project building a
web service uses some or all.

## `SecurityBlueprint` — security-aware route registration

**File:**
`codecraftai-webservice-flask/source/packages/ccai/webservice/xflask/securityblueprint.py:17`

```python
class SecurityBlueprint(Blueprint):
    """
        A Flask Blueprint subclass that integrates with the route security manager.

        This blueprint automatically registers routes with the security manager when they
        are added, ensuring proper access control enforcement. Routes must be explicitly
        marked as public or secure using the provided decorators.
    """

    def __init__(self, name: str, import_name: str, route_security: RouteSecurityManager,
                 static_folder: Optional[str] = None, static_url_path: Optional[str] = None,
                 template_folder: Optional[str] = None, url_prefix: Optional[str] = None,
                 # ... other Blueprint parameters ...
                 ):
        super().__init__(
            name=name,
            import_name=import_name,
            static_folder=static_folder,
            # ...
        )

        self._route_security = route_security
        self._pending_security_registrations = []
```

`SecurityBlueprint` extends Flask's `Blueprint` to
automatically register every route with a
`RouteSecurityManager`.  When a route is added (via
`@blueprint.route("/foo")`), the blueprint records the
registration; the manager later evaluates which roles can
reach the route.

The pattern eliminates a common security bug: a developer
adds a new endpoint and forgets to add it to the security
table.  `SecurityBlueprint` makes registration *automatic*;
the developer marks each route as public or secure, but
can't forget to register it.

## `RouteSecurityManager` — the access-control decision point

**File:**
`codecraftai-webservice-flask/source/packages/ccai/webservice/xflask/basesecurity.py:24`

```python
class RouteSecurityManager:
    """
        Manage route metadata that associates endpoints with access control rules.

        * Public APIs
        * Token Authenticated
        * Domain Authenticated
        * Role Authentication
        * Group Authentication
    """

    def __init__(self, app: Flask, route_security_table: Optional[Dict[str, Set[str]]] = None):
        if route_security_table is None:
            route_security_table = {}

        self._app = app
        self._api_prefixes = set()
        self._static_prefixes = set()
        self._endpoint_security_table = route_security_table
        self._public_endpoints = set()

        self._endpoint_login = "login"
        self._endpoint_logout = "logout"
        self._endpoint_landing = "landing"
```

The manager holds:

- **The endpoint → required-roles map**
  (`_endpoint_security_table`).
- **The set of public endpoints** (no auth required).
- **API and static prefixes** for distinguishing endpoint
  classes (API routes return 401; UI routes redirect to
  login).
- **The conventional endpoints** for login / logout /
  landing.

At request time, the manager:

1. Looks up the requested endpoint.
2. If public, allow.
3. If authenticated, check roles against the required set.
4. If unauthenticated:
   - For an API route, return 401 / 403.
   - For a UI route, redirect to login.

The five-class taxonomy ("Public / Token / Domain / Role /
Group") is the manager's vocabulary; projects extend with
their own taxonomy by subclassing.

## OAuth integration — eight providers

**File:**
`codecraftai-webservice-flask/source/packages/ccai/webservice/xflask/baseauthentication.py:16`

```python
class OauthProviderNames(str, Enum):
    """
        Enumerates the supported OAuth providers for the automation site.
    """
    Amazon = 'amazon'
    Apple = 'apple'
    Google = 'google'
    GitHub = 'github'
    Microsoft = 'microsoft'
    Okta = 'okta'
    Facebook = 'facebook'
    Gitlab = 'gitlab'


OAUTH_PROVIDER_LOOKUP = {member.value: member.name for member in OauthProviderNames}
```

Eight named providers.  Each has its own callback endpoint
in `baseendpoints.py` and its own token-handling code in
`baseoperations.py`.  A project building a web service that
authenticates users typically uses some subset:

- A consumer-facing site uses Google, Apple, Facebook,
  Amazon.
- A developer-facing site uses GitHub, GitLab, Microsoft,
  Okta.
- An enterprise-facing site uses Microsoft, Okta plus the
  consumer ones.

The framework's role is to provide *the integration shape*
— callbacks, token storage, refresh, revocation on logout —
not to dictate which providers a project uses.  Projects
disable the providers they don't want by *not* registering
the endpoints; the framework supports them all.

## SQLAlchemy ORM — `Credential` and friends

**File:**
`codecraftai-webservice-flask/source/packages/ccai/webservice/xflask/baseorm.py:35`

```python
class Credential(WebServiceBase, SerializableMixIn):
    """
        Represents a user's credentials for authentication.
        This model is used for password-based authentication and is separate from OAuth user records.
    """

    __tablename__ = "credentials"

    id = Column('cred_id', VARCHAR(40), primary_key=True,
                doc="Unique identifier for the credential record.")
    password_hash = Column('cred_password_hash', VARCHAR(255), nullable=False,
                           doc="Hashed password for the user account")
    created_at = Column('cred_created_at', DateTime, default=utc_now, nullable=False,
                        doc="Timestamp when the credential record was created")
    updated_at = Column('cred_updated_at', DateTime, default=utc_now, onupdate=utc_now, nullable=False,
                        doc="Timestamp when the credential record was last updated")


class CredentialResetToken(WebServiceBase, SerializableMixIn):
    """
        Represents a token used for resetting a user's password.
    """

    __tablename__ = "credential_reset_tokens"

    id = Column('reset_token_id', VARCHAR(40), primary_key=True)
    user_id = Column('reset_token_user_id', VARCHAR(40), nullable=False, index=True,
                     doc="ID of the user for whom the reset token is issued")
    token_hash = Column('reset_token_hash', VARCHAR(255), nullable=False, unique=True,
                        doc="Hash of a unique token string")
    created_at = Column('reset_token_created_at', DateTime, default=utc_now, nullable=False)
    expires_at = Column('reset_token_expires_at', DateTime, default=reset_token_expiration, nullable=False)
```

The ORM patterns:

- **VARCHAR(40) primary keys.**  Keys are typically UUIDs or
  similar 40-char strings.
- **Hashed passwords.**  `password_hash` is the storage
  column; the framework's auth code never stores plaintext.
- **`created_at` / `updated_at` columns** with default
  `utc_now` and `onupdate=utc_now`.  Every table tracks its
  own lifecycle.
- **`SerializableMixIn`.**  Provides `to_dict()` /
  `from_dict()` for JSON serialisation across the wire.

A project subclassing these conventions for new tables gets
the same auditing for free.  The base mixin handles the
boilerplate.

## Distinction from `codecraftai-credentials`

A vocabulary point: this chapter's `Credential` ORM model
in the Flask web-service is *for storing the website's
own user passwords*.  It is *not* the same thing as the
`codecraftai-credentials` package (Volume 0 chapter 10, Volume
2 chapter 11), which is the runtime credential manager for
test jobs.

The two coexist:

- The website stores its users' credentials in its database
  (this chapter's `Credential` model).
- The website's *backend* might call into other services
  using credentials managed by `codecraftai-credentials` (the
  framework's runtime credentials).

A reader confused by overlapping vocabulary should remember:
"`Credential` ORM" is for the website's user; "`Credential`
in `codecraftai-credentials`" is for the test job's
authentication.  Different stores, different lifecycles,
different concerns.

## What the Flask scaffolding does NOT do

Three things the package deliberately leaves to the project:

- **The actual UI.**  The package provides the *backend*
  scaffolding (security, auth, ORM); the project provides
  the templates, the JavaScript, the CSS.
- **The deployment.**  Gunicorn / uWSGI / Docker / Kubernetes
  — the project's choice.  The package provides a Flask app;
  the project deploys it.
- **The database choice.**  SQLAlchemy supports many
  backends; the project picks Postgres / MySQL / SQLite based
  on its needs.

The package is opinionated about *patterns* (security
routing, auth flow, ORM shape) and unopinionated about
*deployment*.

## When a project should use this scaffolding

The scaffolding is a fit when:

- The project needs a Flask web service with role-based
  access control.
- The project authenticates users through any of the eight
  supported OAuth providers.
- The project's data model fits SQLAlchemy ORM.
- The project values consistency with the portfolio's
  conventions.

The scaffolding is overkill when:

- The project needs only a single-route public API.
- The project uses a different web framework (FastAPI,
  Django).
- The project's auth is a single API key, not user-level.

For overkill cases, project-specific scaffolding is the right
shape.  For appropriate cases, this scaffolding eliminates a
month of boilerplate.

## What the next chapter does

Chapter 15 is the volume's case study — a worked walk-through
of provisioning a Pod, activating it, running a UI test
against an application inside the Pod, integrating with
GitLab to record results, and exposing a status dashboard
through the Flask scaffolding.  The case study assembles the
chapters into one example.
