# Chapter 12 — Declarative UI Mappings

> *Test code that says "click button five" is more readable
> than test code that says `find_element(role="button",
> name="Five").click()`.  Declarative application mappings are
> the framework's mechanism for assigning semantic names to
> UI elements once and using them throughout test code.
> `DeclarativeApplication` plus `ElementHint` plus a per-app
> class is the pattern.*

The previous chapter established the platform-neutral UI
automation hierarchy.  Test code calling `find_element(...)`
with role and name arguments works, but it's verbose and
brittle: the same `(role, name)` tuple appears in every test
that touches the same element.  A button rename means
editing N tests.

The framework's answer: **declare the application's UI
structure once**, and write test code against the
declarations.  A button rename means editing one declaration.

## The pattern

Three classes are involved:

- **`DeclarativeApplication`** — the base for project-side
  application classes.
- **`ElementHint`** (and specialised subclasses
  `WindowHint`, `ButtonHint`, `TextHint`, `MenuHint`, etc.) —
  declares one named UI element with its locating
  parameters.
- **A per-app subclass** that lists the application's named
  elements as class attributes.

A test then attaches to the application and uses the
declared names:

```python
calc = Calculator.attach()
calc.Button5.click()
calc.ButtonAdd.click()
calc.Button7.click()
calc.ButtonEqual.click()
result_text = calc.DisplayTextField.read_text()
```

Compare to the equivalent without declarative mappings:

```python
engine = AutomationFactory.get_automation_engine()
calc = engine.get_application_by_name("Calculator")

main_window = calc.find_element(role="window", name="Calculator")
btn_5 = main_window.find_element(role="button", name="Five")
btn_5.click()
btn_add = main_window.find_element(role="button", name="Add")
btn_add.click()
# ... and so on
```

The first form names what the test does.  The second names
how the test finds things.  Both achieve the same outcome;
the first is the readable one.

## `ElementHint`

**File:**
`codecraftai-uia/source/packages/ccai/uia/model/declarative.py:14`

```python
class ElementHint(Generic[T]):
    """
        Declarative hint to locate a UI element of type T.

        :param type_class: The expected class of the element (e.g. UIButton).
        :param name: The name of the element to match.
        :param role: The role of the element to match.
        :param access_name: Alternate accessibility name.
    """
    def __init__(self, type_class: Type[T] = UIElement, name: str = None, role: str = None,
                 access_name: str = None, path: str = None, identifier: str = None):
        self.type_class = type_class
        self.name = name
        self.role = role
        self.access_name = access_name
        self.path = path
        self.identifier = identifier

    def resolve(self, parent: UIElement) -> Optional[T]:
        """
            Resolves the hint to a concrete UI element within the parent context.
        """
        # ... dispatches to parent's find capability ...

    def matches(self, element: UIElement) -> bool:
        """
            Checks if the given element matches this hint's criteria.
        """
        # ... checks role / name / identifier ...
```

A `ElementHint` packages the *finding strategy* for a UI
element:

- **`type_class`** — the expected element type (e.g.,
  `UIButton`).  Used for type inference and optional runtime
  validation.
- **`name`** — the displayed name.
- **`role`** — the platform-specific role (e.g.,
  `AXButton` on macOS, `push button` on Linux AT-SPI,
  `Button` on Windows UIA).
- **`access_name`** — alternate accessibility name.
- **`path`** — XPath-like locator (`"//{name:Calc*}"`).
- **`identifier`** — platform-specific identifier (the
  `AXIdentifier` value on macOS, etc.).

The hint can use any combination.  A button uniquely named
within its window can use just `name`; a button shared by
name across windows might use `name` + `path` to specify
which window's button.

Specialised subclasses tighten the type:

```python
class WindowHint(ElementHint[UIWindow]):
    def __init__(self, ...):
        super().__init__(type_class=UIWindow, ...)

class ButtonHint(ElementHint[UIButton]):
    def __init__(self, ...):
        super().__init__(type_class=UIButton, ...)

# Similar for TextHint, MenuHint, MenuItemHint, MenuBarHint, TableHint, ...
```

The specialised hints make the test code's type inference
work — a `ButtonHint` resolves to a `UIButton`; a
`WindowHint` resolves to a `UIWindow`.  IDEs and type
checkers know the right methods.

## `DeclarativeApplication`

**File:**
`codecraftai-uia/source/packages/ccai/uia/model/declarative.py:489`

```python
class DeclarativeApplication:
    """
        Base class for declarative application models.
        Note: This is a factory/proxy, not the actual application instance.
        It returns a concrete UIApplication when valid.
    """

    APPLICATION_NAME: str = None
    APPLICATION_COMMAND: str = None

    @classmethod
    def attach(cls, automation_engine: Optional[AutomationEngine] = None) -> Optional[UIApplication]:
        """
            Attaches to the running application matching the hint.
        """
        if not automation_engine:
             automation_engine = AutomationFactory.get_automation_engine()

        if cls.APPLICATION_NAME:
            app = automation_engine.get_application_by_name(cls.APPLICATION_NAME)
            if app:
                # Inject the declarative model into the runtime object
                app.declarative_model = cls
            return app

        return None

    @classmethod
    def launch_or_attach(cls, automation_engine: Optional[AutomationEngine] = None) -> Optional[UIApplication]:
        """
            Launches the application if not running, or attaches to it.
        """
        app = cls.attach(automation_engine)
        if app:
            return app

        if cls.APPLICATION_COMMAND:
            import subprocess
            import shlex
            import time

            subprocess.Popen(shlex.split(cls.APPLICATION_COMMAND))

            for _ in range(20):
                time.sleep(1)
                app = cls.attach(automation_engine)
                if app:
                    return app

        return None
```

Two class methods:

- **`attach(engine=None)`** — looks up a running application
  by `APPLICATION_NAME` and binds the declarative model
  class to the returned `UIApplication` instance.  Returns
  None if the app isn't running.
- **`launch_or_attach(engine=None)`** — tries `attach` first;
  if not running, spawns the app via `APPLICATION_COMMAND`
  and waits up to 20 seconds for it to appear.

The "binding" step (`app.declarative_model = cls`) is what
makes the per-element accessors work.  When test code
references `calc.Button5`, the framework's
`UIApplication.__getattr__` (or equivalent) looks up
`Button5` on the bound declarative model class, finds the
`ButtonHint`, and resolves the hint to a concrete element.

## A worked per-app class — Calculator (macOS)

**File:**
`codecraftai-uia/source/packages/ccai/uia/platforms/macos/mappings/calculator.py:9`

```python
class Calculator(DeclarativeApplication):
    """
        Declarative mapping for the MacOS Calculator application.
    """

    APPLICATION_NAME = "Calculator"
    APPLICATION_COMMAND = "open -a Calculator"

    MainWindow = WindowHint(role=AX_WINDOW_ROLE, name="Calculator")
    """The main Calculator window."""

    DisplayTextField = TextHint(role=AX_TEXT_ROLE)
    DisplayStaticText = TextHint(role=AX_STATIC_TEXT_ROLE)
    """Display control(s) showing the current input or result."""

    # Digits (AXIdentifier values)
    Button0 = ButtonHint(name="Zero", role=AX_BUTTON_ROLE)
    Button1 = ButtonHint(name="One", role=AX_BUTTON_ROLE)
    # ... through Button9 ...

    # Operators
    ButtonAdd = ButtonHint(name="Add", role=AX_BUTTON_ROLE)
    # ... ButtonSubtract, ButtonMultiply, ButtonDivide, ButtonEqual ...

    ButtonClear = ButtonHint(name="AllClear", role=AX_BUTTON_ROLE)
    ButtonDelete = ButtonHint(name="Delete", role=AX_BUTTON_ROLE)
```

The class is a flat list of named hints.  `Button0` through
`Button9`, the four arithmetic operators, equals, clear,
delete.  Each is a `ButtonHint` with the platform-specific
role (`AX_BUTTON_ROLE`) and the macOS-specific name (`Zero`,
`One`, etc.) drawn from the live UI's `AXTitle` / `AXName`
values.

A test using this:

```python
from ccai.uia.platforms.macos.mappings.calculator import Calculator

def test_calculator_addition():
    """
        Verify 5 + 7 = 12 in the macOS Calculator.
    """
    calc = Calculator.launch_or_attach()
    if calc is None:
        raise SemanticError("Could not launch or attach to Calculator.")

    calc.Button5.click()
    calc.ButtonAdd.click()
    calc.Button7.click()
    calc.ButtonEqual.click()

    result_text = calc.DisplayTextField.read_text()
    rtnval = result_text == "12"
    assert rtnval
    return
```

The test reads naturally.  Each `calc.X` is a hint resolved
on-demand.  A button rename in macOS upgrades that need
*one* edit: the `ButtonHint` declaration in the per-app
class.

## Per-platform mapping — same shape, different names

The Linux LibreOffice Calc mapping:

**File:**
`codecraftai-uia/source/packages/ccai/uia/platforms/linux/mappings/libreoffice_calc.py:37`

```python
class LibreOfficeCalc(DeclarativeApplication):
    """
        Declarative mapping for LibreOffice Calc.
    """
    APPLICATION_NAME = "soffice"
    APPLICATION_COMMAND = "libreoffice --calc --norestore"

    MainWindow = WindowHint(role="frame", name="~LibreOffice Calc")

    class MenuBar(MenuBarHint):
        """Application Menu Bar."""
        File = MenuHint(name="File")

    SaveAsMenuItem = MenuItemHint(name="Save As...")
    SaveAs = SaveAsDialog(name="Save As")

    # Spreadsheet
    Spreadsheet = TableHint(role="table")
```

Same shape (`DeclarativeApplication` subclass, hints as
class attributes) but the *contents* are platform-specific:

- `APPLICATION_NAME` is `"soffice"` (the LibreOffice
  process name) rather than `"Calculator"`.
- The role values are AT-SPI roles (`"frame"`, `"table"`)
  rather than macOS AX roles.
- Nested classes (`MenuBar`) group related hints.

A project that needs the same logical app on multiple
platforms writes one per-platform mapping.  The test code
imports the right one for the platform it's running on; the
shape of the test is the same regardless.

## Composition with the test framework

Declarative UI mappings *integrate* with the rest of the
framework cleanly:

- A test attaches to an application and exercises it.  The
  test is the actor (Volume 2 chapter 2's commitment).
- A *tandem validator* (Volume 2 chapter 5) can attach to
  observe UI state during the test — e.g., a
  `ScreenshotValidator` that captures screenshots at
  intervals.
- The application launch / attach happens in fixture-style
  code (Volume 1 chapter 3's resource factory model) — a
  per-test fixture that yields the app and kills it on
  cleanup.
- UI test results land in the JSOS stream like any other
  test results (Volume 1 chapter 5).

The declarative mapping is a *substrate* for tests; the rest
of the framework wraps it as it would any other system under
test.

## Limitations and rough edges

A reader using the package today should know about a few
limitations:

- **Windows is stubbed.**  Chapter 11 noted this; it bears
  repeating.  Windows-only UI tests don't work today.
- **The factory is hardcoded.**  No registration of
  custom platforms.
- **The hint resolution dispatches to platform-specific
  finder methods.**  A hint with a `path` works on macOS
  through XPath-like resolution but the same path may need a
  different syntax on Linux.  Mappings for the same logical
  app on different platforms typically have different hint
  parameters.
- **Audit history.**  The 20 bugs caught in the recent audit
  (chapter 11) all surfaced from manual testing.  An SDET
  using the package today should still expect occasional
  rough edges — the framework is mature but not battle-
  hardened across every UI variation.

## What the next chapter does

Chapter 13 walks **GitLab integration** — a worked example
of a real-world REST integration in the portfolio.  The
distinction between auto-generated and hand-written code,
the OpenAPI source spec, the `RestClientBase` foundation,
and the Aspects-pattern integration for retry / polling
behaviour.  The chapter is short because the package is
narrowly-scoped, but the patterns it illustrates apply more
broadly.
