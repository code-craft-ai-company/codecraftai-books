# Chapter 6 — REST Client Foundations

> *REST is request / response — a different shape from SSH's
> stream orientation.  But Measured Interoperability applies
> the same way: every method takes an `AspectsRest` parameter
> carrying its persistence criteria; the client honours the
> contract; control reverts via typed exception when the
> contract is exceeded.*

`automojo-restclient-base` is the REST foundation that
`automojo-restclient-github-public` (chapter 7) and
Volume 6's `automojo-gitlab` (chapter 13 of Volume 6)
build on.  This chapter walks the foundation classes —
`RestClientBase`, `AspectsRest`, `RestApiGroup`,
`ItemsResult` — and the auto-generation infrastructure that
turns OpenAPI specs into Python clients.

## The package layout

```
automojo-restclient-base/source/packages/automojo/restclient/base/
├── restclientbase.py         # the core client
├── restclientgenerator.py    # the OpenAPI generator
├── aspectsrest.py            # AspectsRest + ActionPattern
├── restapigroup.py           # base for operation groupings
├── itemsresult.py            # lazy paged container
├── pagedlist.py              # older paging container
├── schemaobject.py           # generated schema types base
├── schemafield.py            # generated schema field base
├── restclientutils.py        # helpers
└── __init__.py
```

Ten modules.  Each handles one concern.  The package is
small enough to read in a single sitting.

## `RestClientBase` — the core client

**File:** `restclientbase.py:41–88`

```python
class RestClientBase:
    """
        Generic REST client base.  Subclasses provide the
        per-API specifics; this class provides authentication,
        retry, and the perform_call pipeline.
    """

    AUTH_LEAF = ""
    VERIFY_CERTIFICATE = True
    AUTHENTICATION_FAILURES = (HttpForbiddenError,)

    def __init__(self, host: str, credential: BaseCredential,
                 port: Optional[int] = None,
                 scheme: Optional[str] = "https",
                 headers: Optional[Dict[str, str]] = None,
                 verify_certificate: Optional[bool] = None,
                 aspects: AspectsRest = DEFAULT_REST_ASPECTS):
        self._host = host
        self._credential = credential
        self._headers = headers if headers is not None else {}
        if verify_certificate is None:
            self._verify_certificate = self.VERIFY_CERTIFICATE
        else:
            self._verify_certificate = verify_certificate
        self._aspects = aspects

        self._host_url = f"{scheme}://{self._host}"
        if port is not None:
            self._host_url = f"{self._host_url}:{port}"

        self._session: Session = None
        self._logger = logging.getLogger()
        return
```

The class accepts the standard set of REST connection
parameters: host, credential, optional port, scheme (default
HTTPS), default headers, TLS verification flag, and the
default `AspectsRest` for all calls.

A few class-level conventions:

- **`AUTH_LEAF`** — the URL leaf used for an authentication
  preflight.  Default empty (no preflight); GitHub-shaped
  clients override to e.g. `"/user"` to verify the token
  against a known endpoint.
- **`VERIFY_CERTIFICATE`** — the default TLS-verification
  setting.  Most projects keep True; lab-only tests against
  self-signed services flip to False per call (or per client
  instance).
- **`AUTHENTICATION_FAILURES`** — tuple of exception classes
  that are interpreted as authentication failures.  The
  client catches these and surfaces them through the
  delegation contract (principle 4).

The `_session: Session = None` is the lazy-init slot.  The
client doesn't open a `requests.Session` at construction; it
opens one the first time `perform_call` runs.  This avoids
work for clients that are constructed but never called.

## `AspectsRest` — REST-specific persistence parameters

**File:** `aspectsrest.py:24–116`

`AspectsRest` is the REST-shaped Aspects subclass.  It lives
*here* (in `automojo-restclient-base`), not in
`automojo-interop`'s aspects module.  This is by design —
REST has specific concerns the base `Aspects` doesn't carry,
and the REST foundation owns those.

```python
class ActionPattern(IntEnum):
    """
        REST-shaped action patterns.  Same five values as
        automojo-interop's ActionPattern.
    """
    SINGLE_CALL = 0
    SINGLE_CONNECTED_CALL = 1
    DO_UNTIL_SUCCESS = 2
    DO_WHILE_SUCCESS = 3
    DO_UNTIL_CONNECTION_FAILURE = 4


class AspectsRest:
    """
        Aspects container specialised for REST operations.
    """

    def __init__(self, *,
                 action_pattern: ActionPattern = ActionPattern.SINGLE_CALL,
                 completion_timeout: float = DEFAULT_COMPLETION_TIMEOUT,
                 completion_interval: float = DEFAULT_COMPLETION_INTERVAL,
                 connection_timeout: float = DEFAULT_CONNECTION_TIMEOUT,
                 inactivity_timeout: float = DEFAULT_INACTIVITY_TIMEOUT,
                 monitor_delay: float = DEFAULT_MONITOR_DELAY,
                 logging_pattern: LoggingPattern = DEFAULT_LOGGING_PATTERN,
                 retry_logging_interval: int = DEFAULT_RETRY_LOGGING_INTERVAL,
                 allowed_error_codes: Tuple[int, ...] = DEFAULT_ALLOWED_ERROR_CODES,
                 allowed_connection_exceptions: Tuple[BaseException, ...] = DEFAULT_ALLOWED_CONNECTION_EXCEPTIONS,
                 must_connect: bool = DEFAULT_MUST_CONNECT,
                 logger: Optional[logging.Logger] = None,
                 session: Optional[Session] = None):
        object.__setattr__(self, "__frozen__", False)

        self.action_pattern = action_pattern
        self.completion_timeout = completion_timeout
        self.completion_interval = completion_interval
        self.connection_timeout = connection_timeout
        self.inactivity_timeout = inactivity_timeout
        self.monitor_delay = monitor_delay
        self.logging_pattern = logging_pattern
        self.retry_logging_interval = retry_logging_interval
        self.allowed_error_codes = tuple(allowed_error_codes)
        self.allowed_connection_exceptions = tuple(allowed_connection_exceptions)
        self.must_connect = must_connect
        self.logger = logger
        self.session = session

        object.__setattr__(self, "__frozen__", True)
        return
```

Three REST-specific extensions over the base `Aspects`:

- **`retry_logging_interval`** — when retrying, log every Nth
  attempt rather than every one.  Useful for chatty long
  retries where every-attempt logging would spam.
- **`session`** — optional explicit `requests.Session`.  When
  provided, the call uses that session; when None, the client
  uses its own.  Useful for test code that wants to share a
  session across calls or inject a mocked session.
- **`allowed_error_codes`** — at the *HTTP-status* level,
  not the exit-code level.  E.g., `(404,)` says "treat 404
  as success-equivalent" — useful for "delete this resource
  if it exists" patterns where 404 is fine.

A point of attention: `AspectsRest` does *not* inherit from
`Freezable`.  It implements its own freeze pattern via
`object.__setattr__("__frozen__", True)` plus a
`__setattr__` override.  Functionally equivalent to the
`Freezable` mixin; the implementation is duplicated rather
than shared.

A reader noticing the duplication is right that it's a small
inconsistency — `Aspects` (in interop) inherits from
`Freezable`, `AspectsRest` (in restclient-base) re-implements
the same pattern.  Future work might unify them.  The
chapter is honest about this; for the present, both work.

## `perform_call` — the central dispatcher

The `RestClientBase.perform_call` method (the actual line
range varies; the method is the heart of the client) is the
function every generated method calls.  Its shape:

```python
def perform_call(self, *,
                 leaf: str,
                 method: str,
                 params: Optional[Dict] = None,
                 body: Any = None,
                 exp_status: List[int],
                 headers: Optional[Dict] = None,
                 context_msg: Optional[str] = None,
                 aspects: Optional[AspectsRest] = None) -> Response:
    """
        Issue an HTTP call.  Honour the chosen ActionPattern;
        return a requests.Response on success; raise on
        contract failure.
    """
    if aspects is None:
        aspects = self._aspects

    if self._session is None:
        self._session = Session()
        self.authenticate()

    url = self._host_url + leaf

    if aspects.action_pattern == ActionPattern.SINGLE_CALL:
        # Single attempt with completion_timeout enforcement.
        with MonitoredScope(...):
            response = self._session.request(
                method=method, url=url, params=params, json=body,
                headers=headers, timeout=aspects.completion_timeout,
                verify=self._verify_certificate)
        if response.status_code not in exp_status \
                and response.status_code not in aspects.allowed_error_codes:
            self._raise_for_status(response, exp_status)
        return response

    elif aspects.action_pattern == ActionPattern.DO_UNTIL_SUCCESS:
        # Retry loop honoring completion_toctx.
        completion_toctx = TimeoutContext(aspects.completion_timeout,
                                          aspects.completion_interval)
        while not completion_toctx.final_attempt:
            try:
                with MonitoredScope(...):
                    response = self._session.request(...)
                if response.status_code in exp_status:
                    return response
                if response.status_code in aspects.allowed_error_codes:
                    return response
                # ... not yet success; retry after interval ...
            except aspects.allowed_connection_exceptions:
                pass  # transient connection error; retry
            completion_toctx.next_attempt(aspects.completion_interval)

        raise completion_toctx.create_timeout(what_for="HTTP success")

    elif aspects.action_pattern == ActionPattern.DO_WHILE_SUCCESS:
        # ... mirror loop ...
```

Same shape as `SshBase._run_cmd` from chapter 4 — different
underlying mechanism, identical persistence-behaviour logic.
The five `ActionPattern` modes are uniform across protocols.

## Authentication

`RestClientBase.authenticate` is the method subclasses
override to apply credentials to the session:

```python
def authenticate(self) -> None:
    """
        Apply the credential to self._session.

        Override per-credential-type.
    """
    if isinstance(self._credential, BasicAuthCredential):
        self._session.auth = (self._credential.username,
                              self._credential.password)
    elif isinstance(self._credential, BearerTokenCredential):
        self._session.headers["Authorization"] = \
            f"Bearer {self._credential.token}"
    elif isinstance(self._credential, ApiKeyCredential):
        self._session.headers[self._credential.header_name] = \
            self._credential.value
    else:
        raise ConfigurationError(
            f"Unsupported credential type: {type(self._credential).__name__}")
    return
```

Three credential types covered out-of-the-box: basic auth,
bearer tokens, API keys.  Subclasses for specific APIs
override (e.g., GitHub's client adds the `Accept:
application/vnd.github.v3+json` header at construction;
GitLab's client adds the project token's specific header).

The authentication is applied to the *session*, not per-call.
Once the session has the auth, every subsequent request
inherits it.  This is what makes the auto-generated methods
(chapter 7) lightweight — they don't think about
authentication.

## `RestApiGroup` — the operation grouping base

**File:** `restapigroup.py:1–43`

```python
class RestApiGroup:
    """
        Base class for REST API operation groups.

        A typical client has many groups (e.g., GitHub has 42:
        repos, issues, pulls, actions, etc.).  Each group is
        a separate class with the methods for its endpoints.
    """

    def __init__(self, client: "RestClientBase"):
        self._client_ref = weakref.ref(client)
        return

    @property
    def client(self) -> "RestClientBase":
        return self._client_ref()

    def _parse_item(self, data: Dict[str, Any], item_type: type[T]) -> T:
        """
            Parse a dict into a SchemaObject instance.
        """
        try:
            if hasattr(item_type, 'from_dict'):
                return item_type.from_dict(data)
            else:
                return item_type(**data)
        except Exception:
            return data
```

Three things:

- **Weak reference to the client.**  The group holds a *weak*
  reference to its parent client.  This avoids reference
  cycles when the client owns multiple groups and each group
  holds a reference back.
- **`_parse_item` deserialiser.**  Generated methods call
  this to turn API response dicts into typed
  `SchemaObject` instances.  The fallback (return raw dict
  on parse failure) is for forward-compatibility when the
  API returns fields the client's schema doesn't know about.
- **No public methods.**  All actual operations live in
  *generated* subclass methods.  The base just provides the
  client reference and the parser.

A worked example in chapter 7 walks a real `ReposApiGroup`
with its 100+ generated methods.

## `ItemsResult` and `Paging` — lazy pagination

**File:** `itemsresult.py:210–275`

REST APIs paginate large collections.  GitHub's "list all
repos" returns 30 at a time; the response includes links to
next / previous pages.  `ItemsResult` is the framework's lazy
container for paged responses.

```python
class ItemsResult(Generic[CT]):
    """
        Holds a list of returned items and optional paging
        information.

        - Non-paged: paging is None; just a list.
        - Paged: paging is a Paging object with next/prev links.
    """

    def __init__(self, items: List[CT], paging: Optional[Paging] = None):
        self._items = items
        self._paging = paging

    @property
    def is_paged(self) -> bool:
        rtnval = True if self._paging is not None else False
        return rtnval

    @property
    def items(self) -> List[CT]:
        return self._items

    @property
    def paging(self) -> Union[Paging, None]:
        return self._paging

    def next_page_result(self) -> "ItemsResult[CT]":
        """
            Retrieve the next page lazily.
        """
        result = None
        if self._paging is not None and self._paging.request_next is not None:
            result = self._paging.request_next.perform_call()
        return result

    def prev_page_result(self) -> "ItemsResult[CT]":
        """
            Retrieve the previous page lazily.
        """
        result = None
        if self._paging is not None and self._paging.request_prev is not None:
            result = self._paging.request_prev.perform_call()
        return result
```

The container holds:

- The current page's items.
- An optional `Paging` object with `request_next` and
  `request_prev` *pre-canned* requests.

`next_page_result` invokes the pre-canned next request,
returns a fresh `ItemsResult`.  No HTTP traffic is made
until `next_page_result` is called — *lazy*.

A test that wants the first page only:

```python
result = client.repos.list_user_repos(username="alice")
for repo in result.items:
    print(repo.name)
```

A test that wants all pages:

```python
result = client.repos.list_user_repos(username="alice")
while result is not None:
    for repo in result.items:
        print(repo.name)
    result = result.next_page_result()
```

The lazy pattern means tests can stop iterating partway
through without paying for unfetched pages.

## `RestClientGenerator` — OpenAPI to Python

**File:** `restclientgenerator.py:77–100`

```python
class RestClientGenerator:
    """
        Shapes the patterns for the REST API generation.
    """

    REST_CLIENT_BASE_CLASS: Type[RestClientBase] = RestClientBase
    GROUP_BASE_TYPE: Type[RestApiGroup] = RestApiGroup
    SCHEMA_BASE_TYPE: Type[SchemaObject] = SchemaObject
    SCHEMA_FIELD_TYPE: Type[Field] = Field
    ASPECTS_TYPE: Type[AspectsRest] = AspectsRest
    PAGEDLIST_TYPE: Type[PagedList] = PagedList

    SCHEMA_MODULE_ROOT = "automojo.github.rest.clients.public.schemas"
    GROUP_MODULE_ROOT = "automojo.github.rest.clients.public.groups"
    CLIENT_MODULE_ROOT = "automojo.github.rest.clients.public"

    DEFAULT_PAGINATION_DETECTION_HINTS = {
        "detection_rules": {
            "query_parameters": {
                "parameters": ["page", "per_page"],
                "enabled": True,
            }
        },
        # ...
    }
```

The generator class is *configurable*: subclasses override
`*_BASE_CLASS` / `*_BASE_TYPE` attributes to customise the
generated code's parent classes (e.g., the GitHub generator
overrides `REST_CLIENT_BASE_CLASS` to a GitHub-aware subclass
that adds the `Accept:` header).

The generator's job:

1. Read the OpenAPI JSON spec.
2. For each schema definition, generate a `SchemaObject`
   subclass.
3. For each operation (path + method), generate a method on
   the appropriate group class.
4. For each tag (logical group), generate a `RestApiGroup`
   subclass.
5. Generate a top-level client class that instantiates each
   group as an attribute.
6. Detect pagination (using `DEFAULT_PAGINATION_DETECTION_HINTS`).

The output is a tree of generated Python modules.  Every
generated file starts with the auto-generated banner from
chapter 7's worked example (`# Auto-generated by RestClientGenerator. DO NOT EDIT.`).

## How the generated code calls back to the foundation

A generated method (chapter 7 walks a real one):

```python
def list_user_repos(self, username: str, *,
                    exp_status: Optional[List[int]] = None,
                    headers: Optional[Dict[str, str]] = None,
                    aspects: Optional[AspectsRest] = None) -> ItemsResult[Repository]:
    client = self.client
    if aspects is None:
        aspects = client.aspects

    leaf_template = "/users/{username}/repos"
    path_params = {'username': username}
    call_leaf = leaf_template.format(**path_params)

    if exp_status is None:
        exp_status = [200]

    resp = client.perform_call(
        leaf=call_leaf,
        method="GET",
        body=None,
        exp_status=exp_status,
        headers=headers,
        context_msg=None,
        aspects=aspects,
    )

    raw_items = resp.json()
    items = [self._parse_item(item, Repository) for item in raw_items]

    paging = self._extract_paging(resp)
    result = ItemsResult(items, paging=paging)
    return result
```

Five steps:

1. Resolve aspects (use caller's, fall back to client's
   default).
2. Format the URL path with caller-supplied parameters.
3. Determine expected statuses.
4. Call `client.perform_call` (the chapter-2-style
   delegated persistence behaviour).
5. Parse the response and wrap in `ItemsResult` if paged.

The same shape repeats across hundreds of methods.  The
foundation classes carry all the variation; the generated
methods are templated.

## What this chapter set up

By the end:

- A reader knows the four foundation classes
  (`RestClientBase`, `AspectsRest`, `RestApiGroup`,
  `ItemsResult`) and can construct one of each by hand.
- The Measured Interoperability discipline applied to REST
  is clear: every method takes Aspects; the client follows
  the contract; control reverts via exception when the
  contract is exceeded.
- The generator's role is understood as a *tool* that
  produces methods following the foundation classes'
  conventions.

The next chapter walks `automojo-restclient-github-public` —
the auto-generated GitHub client built on this foundation.
A 571-module package; the chapter shows what's worth
testing and what to trust by construction.
