# Chapter 4 — Device Discovery and Friendly Identifiers

> *Names matter.  Devices in a Landscape have full identifiers
> (often IPs or DNS names) that are correct but unwieldy, and
> short hints (often human-friendly names) that are convenient
> but ambiguous.  `FriendlyIdentifier` is the framework's
> answer — pair them, optionally bind them with a regex, and
> implement careful equality semantics.*

A `LandscapeDevice` has a name.  The Landscape's YAML
declared `name: casey-01`.  But the device also has a host
address (`192.168.1.21`), and a fully-qualified DNS name
(`casey-01.lab.internal`), and possibly more identifiers
depending on the project's discovery patterns.  A test might
want to look up the device by any of these.

The framework's answer is `FriendlyIdentifier` — a small
class that pairs a *full identifier* with a *hint* and
optionally a *regex* for canonical-identity extraction.  The
class also implements careful equality semantics: two
`FriendlyIdentifier`s are equal when their full_identifiers
match; a `FriendlyIdentifier` compared against a string
returns equality if the string is a substring of the full
identifier.

This chapter walks the class with code anchors and explains
the design.  It also notes a real bug in the class's history
— the audit-pass-4 `__eq__` bug that always returned `True`,
caught and fixed in commit `1545f67`.

## The class

**File:** `codecraftai-landscaping/source/packages/ccai/landscaping/friendlyidentifier.py:24`

```python
class FriendlyIdentifier:
    """
        Associates a short identifier with a full identifier string and an
        optional identity match pattern.
    """

    def __init__(self, full_identifier: str, hint: str,
                 identity_match: Optional[Union[re.Pattern, str]]=None):
        """
            :param full_identifier: The full identifier string.
            :param hint: A short name used for quick reference.
            :param identity_match: Optional regular expression used to extract an identity value.
        """
        self._full_identifier = full_identifier
        self._hint = hint
        self._identity_match = identity_match
        if self._identity_match is not None and isinstance(identity_match, str):
            self._identity_match = re.compile(identity_match)

        # If a short_match expression is specified, it must have a match in the full identifier.
        if self._identity_match is not None:
            mobj = self._identity_match.match(self._full_identifier)
            if mobj is None:
                errmsg = "The specified identifier is not compatible with the short match expression provided."
                raise SemanticError(errmsg)
```

Three instance variables.  Three properties:

- `_full_identifier` — the canonical, unambiguous string for
  the device.  Often a fully-qualified hostname or an IP.
- `_hint` — a short, human-friendly name.  Often the same
  as the device's `name:` in the Landscape.
- `_identity_match` — optional regex.  When present, the
  pattern must match the full identifier at construction
  time; otherwise `SemanticError` raises.

The `SemanticError` at construction is the Volume 0 chapter
3 discipline applied here: building a `FriendlyIdentifier`
with an incompatible regex is API misuse, not a runtime
condition; raise `SemanticError` and let it bubble.

## The `identity` property

**File:** `friendlyidentifier.py:68–107`

```python
@property
def identity(self) -> str:
    """
        Returns the canonical identity for this object.
        - If identity_match was specified, returns the first capturing group.
        - Otherwise, returns the hint.
        - Otherwise, returns the full identifier.
    """
    if self._identity_match is not None:
        mobj = self._identity_match.match(self._full_identifier)
        if mobj is None:
            errmsg = "The full_identifier no longer matches the identity_match pattern."
            raise SemanticError(errmsg)

        groups = mobj.groups()
        if len(groups) > 0:
            rtnval = groups[0]
        else:
            errmsg = "The identity_match pattern has no capturing group."
            raise SemanticError(errmsg)
    elif self._hint is not None:
        rtnval = self._hint
    else:
        rtnval = self._full_identifier

    return rtnval
```

The `identity` property returns the *canonical extracted name*
when a regex is set.  Three failure cases the property guards
against:

1. The regex no longer matches the full identifier — surfaces
   a `SemanticError`.  This happens if `_full_identifier` is
   later modified inconsistently with the pattern.
2. The pattern has no capturing group — surfaces a
   `SemanticError`.  The pattern's job is to *extract* the
   identity; a pattern with no group is incomplete.
3. Neither pattern nor hint — fall back to the full
   identifier itself.

The fallback chain (regex → hint → full) is what makes
`identity` always return *something*.  The most-canonical name
the object knows about; the property's contract is that the
return value uniquely identifies the device within its scope.

## The `match` method

**File:** `friendlyidentifier.py:109–116`

```python
def match(self, hint: str) -> bool:
    """
        Returns True if the given hint substring is found in this object's
        full_identifier and the hint exists in the hint string.
    """
    rtnval = (self._full_identifier.find(hint) > -1) and (self._hint.find(hint) > -1)
    return rtnval
```

The `match` method answers "does this `FriendlyIdentifier`
match the given hint?"  The answer is yes if the hint appears
as a substring of *both* the full identifier and the hint.

This is *bidirectional substring*, not equality.  Useful when
test code says `"casey"` and wants to find any device whose
full identifier and hint both contain `"casey"`.

The dual-substring requirement prevents false matches.  If a
device has `_full_identifier = "casey-01.lab.internal"` and
`_hint = "casey-01"`, asking `match("casey")` returns True.
But if a different device has
`_full_identifier = "casey-01.lab.internal"` and
`_hint = "node-7"`, the same `match("casey")` returns False
— the hint doesn't contain the requested substring, so the
test author's "casey" doesn't match this device.

## The `__eq__` method — and its history

**File:** `friendlyidentifier.py:137–165`

```python
def __eq__(self, other):
    """
        Compares two identifiers for equivalence.
        - FriendlyIdentifier vs FriendlyIdentifier: full_identifier equality.
        - FriendlyIdentifier vs str: substring match (full_identifier.find(other) > -1).
        - Other types: returns NotImplemented.
    """
    if isinstance(other, FriendlyIdentifier):
        result = self._full_identifier == other._full_identifier
    elif isinstance(other, str):
        result = self._full_identifier.find(other) > -1
    else:
        result = NotImplemented
    return result
```

Three cases:

1. **Two `FriendlyIdentifier`s.**  Equal if their full
   identifiers match exactly.
2. **`FriendlyIdentifier` vs a string.**  Equal if the string
   is a substring of the full identifier.  Asymmetric — this
   lets `device == "casey"` work in test code without
   constructing a comparison `FriendlyIdentifier`.
3. **Other types.**  Returns `NotImplemented` to let Python
   fall back to other comparisons.

The asymmetry (substring match for strings, exact match for
other `FriendlyIdentifier`s) is *intentional* but worth
calling out — readers expecting `==` to be exact will be
surprised by `device == "casey"` returning True for any
device whose full identifier contains "casey".  The
discipline: the framework's preference is for *test code to
be permissive*; tests that need strict equality use the
`identity` property and compare strings.

### The audit-pass-4 bug

The current implementation is correct.  The audit history
isn't — pass 4 of the six-pass landscaping audit (commit
`1545f67`) caught a version of `__eq__` that always returned
True:

```python
# BUG (caught and fixed)
def __eq__(self, other):
    if isinstance(other, FriendlyIdentifier):
        result = self._full_identifier == self._full_identifier  # ← wrong
    # ...
```

The bug compared `self._full_identifier` to itself rather
than to `other._full_identifier`.  Two `FriendlyIdentifier`s
were *always* equal, regardless of their full identifiers.

The bug was *silent* — it never raised, and tests that
happened to use `==` between `FriendlyIdentifier`s would
incidentally pass when they should have failed.  The audit
caught it through code review, not through a test failure.

The lesson worth pulling out for the volume's reader: silent
bugs in comparison code are particularly dangerous because
they break the *invariant* that comparison reflects equality.
Volume 2 chapter 15 walked the audit-diary discipline that
caught this and 14 other silent bugs across the six-pass
landscaping audit.

## Using `FriendlyIdentifier` in code

A test that wants a specific device:

```python
device = LandscapeSingleton().device("casey-01")
```

The `device(name)` method looks up the device whose
`FriendlyIdentifier.identity` matches the name string.

A test that wants any device matching a pattern:

```python
candidates = LandscapeSingleton().get_devices(
    include_filters=["casey-*"],
    exclude_filters=[],
)
```

The framework walks every device's `FriendlyIdentifier` and
keeps the ones whose identity matches the include filter.

A test that wants devices by role:

```python
workers = LandscapeSingleton().devices_by_role("worker")
```

Different lookup, different mechanism — the role is a
top-level field on the device declaration, not part of the
`FriendlyIdentifier`.  But the result is a list of devices,
each with its own `FriendlyIdentifier`.

## Project-specific identity patterns

A project might want a specific identity-extraction pattern.
For example, a project using devices named like
`region-az-shard-replica` (e.g., `us-east-1-2-3`) might want
the identity to be only the shard-replica part:

```python
fid = FriendlyIdentifier(
    full_identifier="us-east-1-2-3",
    hint="2-3",
    identity_match=r"^[a-z]+-[a-z]+-\d+-(\d+-\d+)$",
)

print(fid.identity)  # → "2-3"
```

The regex captures just the last two number-dash-number
segment; that's what `identity` returns.  Test code working
with shards uses the canonical short form; logging and
debugging see the full identifier.

The project doesn't subclass `FriendlyIdentifier` — it just
constructs instances with the right pattern.  The framework's
class is sufficient.

## Why a class instead of plain strings

A reader's natural question: why not just use strings?  A
device has a name; the name is a string; why wrap it?

Three reasons:

1. **Multi-name lookup.**  A device has a full identifier
   *and* a hint.  Test code might use either; the
   `FriendlyIdentifier` provides both as separate fields and
   a unified equality.
2. **Identity extraction.**  The regex pattern lets the
   project define a canonical identity that's a *function* of
   the full identifier, not just one of its substrings.  This
   matters for projects with structured naming conventions.
3. **Type safety.**  A function that takes a
   `FriendlyIdentifier` is asking for a *device identity*,
   not just any string.  Type checkers catch the difference;
   readers see the intent.

The cost is a class allocation per device.  A landscape with
50 devices has 50 `FriendlyIdentifier`s.  At 8 instance
variables each, that's negligible.

## What the next chapter does

Chapter 5 walks Topology — the logical organisation overlay
on the Landscape's physical inventory.  The canonical
`clusters` shape (with `name`/`group`/`nodes`/`spares`),
the verification contract from Volume 2 chapter 7 walked at
the schema level, the relationship to Roles, and the
distinction from Landscape that the previous chapters set up
for the reader.
