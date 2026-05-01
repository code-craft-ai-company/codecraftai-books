# Chapter 11 — Cross-Platform UI Automation

> *Where the previous chapter walked a scaffolding-stage
> package, this one walks a mature one.  `automojo-uia`
> implements UI automation across Linux (AT-SPI / X11 /
> Wayland), macOS (Accessibility / scripting bridges), and
> Windows (UIA — currently stubbed with NotImplementedError).
> The class hierarchy is small, the platform wrappers are
> substantial, and a recent 20-bug audit hardened the
> package.*

The framework's UI automation package is the answer to a
specific kind of automation question: *the system under test
is a graphical application*.  Volume 1 chapter 1 listed test
shapes the framework supports; UI tests are one of them, and
`automojo-uia` is the substrate.

This chapter walks the platform-neutral hierarchy, the
per-platform wrappers, the factory dispatch model, and the
recent audit history.  Chapter 12 walks the declarative
application mappings on top of this substrate.

## The hierarchy

**File:** `automojo-uia/source/packages/automojo/uia/platforms/base.py`

```python
# Lines 13–33
class UIObject(ABC):
    """
        Base class for all UI objects.
    """

# Lines 35–72
class DisplayElement(UIObject):
    """
        Foundation class for any displayed UI object.
    """
    @property
    @abstractmethod
    def bounds(self) -> Tuple[int, int, int, int]: ...
    @property
    @abstractmethod
    def position(self) -> Tuple[int, int]: ...
    @property
    @abstractmethod
    def size(self) -> Tuple[int, int]: ...
    @property
    @abstractmethod
    def is_visible(self) -> bool: ...

# Lines 74–107
class ScreenElement(DisplayElement):
    """
        Foundation class for screen-level objects like windows and applications.
    """
    def __init__(self, parent: Optional['UIElement'] = None):
        self._parent = weakref.ref(parent) if parent is not None else None
    # Includes bring_to_front() and send_to_back()

# Lines 109–196
class UIElement(DisplayElement):
    """
        Abstract base class for UI elements (controls/widgets).
    """
    # Includes find_element, find_elements, wait_for_element, wait_until_gone

# Lines 496–536
class UIWindow(ScreenElement):
    """
        Abstract base class for windows.
    """
    @property
    @abstractmethod
    def name(self) -> str: ...
    @property
    @abstractmethod
    def title(self) -> str: ...

# Lines 560–601
class UIApplication(ScreenElement):
    """
        Abstract base class for applications.
    """
    def __init__(self, parent: Optional['UIElement'] = None):
        super().__init__(parent)
        self._declarative_model = None
```

Six classes.  Each inherits from the one above:

- `UIObject` — the abstract root.  Has `__repr__` and
  `exists`.
- `DisplayElement` — anything visible.  Adds `bounds`,
  `position`, `size`, `is_visible`.
- `ScreenElement` — top-level visible objects (windows,
  applications).  Adds `bring_to_front`, `send_to_back`.
- `UIElement` — interactive controls (buttons, text fields).
  Adds `find_element`, `find_elements`, `wait_for_element`,
  `wait_until_gone`.
- `UIWindow` — a window.  Adds `name`, `title`.
- `UIApplication` — a running application.  Adds
  `_declarative_model` (the linkage to chapter 12's
  declarative model).

The hierarchy is *small enough to memorise*.  Six classes;
two parallel branches (UIWindow and UIApplication both inherit
from ScreenElement; UIElement is its own branch).  A reader
of any UI test code should be able to locate every class
they encounter in this hierarchy.

## The platform wrappers

Each platform has its own per-class wrapper that
*implements* the abstract methods using the platform's
native UI APIs.

### macOS

**File:**
`automojo-uia/source/packages/automojo/uia/platforms/macos/wrappers/uiaapplication.py:29`

```python
class MacApplication(WrappedObject, UIApplication):
    """
        Wrapper for an application process. Use this object to attach to and
        communicate with the application process; window objects are exposed
        separately via window wrappers.
    """
    APPLICATION_NAME = "NotSet"
    BUNDLE_IDENTIFIER = "NotSet"

    def __init__(self, controller: "MacOSAutomation", native_obj: Any):
        super().__init__(controller, native_obj)
        self._sb_process = None  # AppleScript scripting-bridge process
        self._ax_app = None      # NSAccessibility app
```

`MacApplication` wraps two macOS-native interfaces:

- **`_sb_process`** — `appscript` / scripting-bridge for
  AppleScript-driven automation (file menus, scriptable
  applications).
- **`_ax_app`** — NSAccessibility's `AXUIElement` for the
  accessibility-tree-driven automation (buttons, text fields,
  menus that are AX-accessible).

The macOS layer has 51 source files and provides per-app
wrapper classes for 27 system applications (Calculator,
Calendar, Contacts, Excel, Mail, Safari, etc.).  Each
per-app class sets `APPLICATION_NAME` and
`BUNDLE_IDENTIFIER` for the framework's lookup.

### Linux

**File:**
`automojo-uia/source/packages/automojo/uia/platforms/linux/wrappers/uiaapplication.py:17`

```python
class LinuxApplication(LinuxElement, UIApplication):
    """
        Wrapper for Linux applications using Agent Architecture.
    """

    def __init__(self, native_obj: Any, access_agent: AccessibilityAgent,
                 wm_agent: WindowManagerAgent, parent: Optional[UIElement] = None):
        super().__init__(native_obj, access_agent, wm_agent, parent=parent)
```

`LinuxApplication` uses an *agent* architecture:

- **`access_agent`** — the accessibility agent.  The shipped
  implementation is `PyAtspiAccessibilityAgent` using AT-SPI
  (`pyatspi`) over D-Bus.
- **`wm_agent`** — the window-manager agent.  Two
  implementations: `X11WindowManagerAgent` for X11,
  `WaylandGnomeWindowManagerAgent` for GNOME on Wayland.

The agent split exists because Linux's UI stack is *not
unified*: AT-SPI handles accessibility (which buttons /
menus / text fields exist and what they say); the window
manager handles window lifecycle (which window is on top,
how to move a window, how to close one).  The two systems
talk different protocols.  The agent split keeps each
agent's responsibility clean.

### Windows

**File:**
`automojo-uia/source/packages/automojo/uia/platforms/windows/`

Currently a stub.  The factory's dispatch:

```python
elif system == "Windows":
    raise NotImplementedError("Windows automation is not yet implemented.")
```

Windows UI Automation (UIA) is the intended substrate when
the implementation lands.  The stub means projects that need
Windows UI tests today cannot use the framework's UI
automation; they fall back to other tools.  The chapter is
honest about this gap.

## The factory

**File:** `automojo-uia/source/packages/automojo/uia/factory.py`

```python
class AutomationFactory:
    """
        Factory class for creating automation engines based on the current platform.
    """

    @staticmethod
    def get_automation_engine() -> AutomationEngine:
        """
            Returns the automation engine for the current platform.
        """
        system = platform.system()

        if system == "Darwin":
            from automojo.uia.platforms.macos.automation import MacOSAutomation
            return MacOSAutomation()
        elif system == "Windows":
             raise NotImplementedError("Windows automation is not yet implemented.")
        elif system == "Linux":
             from automojo.uia.platforms.linux.automation import LinuxAutomation
             return LinuxAutomation()
        else:
            raise NotImplementedError(f"Unsupported platform: {system}")
```

The factory dispatches on `platform.system()`.  Three
branches: macOS, Linux, Windows (stubbed).  Other platforms
raise.

A note on the design: this is *not* a factory protocol like
the Landscape's (chapter 7).  It's a hardcoded
`if`/`elif`/`else`.  Adding a new platform requires *editing
the factory's source*, not registering through a namespace
package.

This is a small wart on an otherwise mature package.
Future work might lift the factory to use the same protocol
pattern as `LandscapingFactoryProtocol`, but the present
implementation is what ships.

## What test code sees

A test using UI automation:

```python
from automojo.uia.factory import AutomationFactory

engine = AutomationFactory.get_automation_engine()
calc = engine.get_application_by_name("Calculator")
calc.bring_to_front()

main_window = calc.find_element(role="window", name="Calculator")
button_5 = main_window.find_element(role="button", name="Five")
button_5.click()
```

The flow:

1. `AutomationFactory` returns the platform-specific engine.
2. `engine.get_application_by_name(...)` returns a
   platform-specific `UIApplication` subclass instance
   (`MacApplication` on macOS, `LinuxApplication` on Linux).
3. `calc.bring_to_front()` calls into the platform-specific
   implementation.
4. `calc.find_element(...)` returns a platform-specific
   `UIElement` subclass instance.
5. `.click()` performs the click using the platform's
   native click machinery (mouse events for X11; `AXPress`
   for macOS Accessibility; etc.).

The test code is *platform-neutral* in the sense that the
same code works against macOS or Linux.  The behaviour is
platform-specific in that the underlying machinery differs.

In practice, projects often pair UI tests with
**declarative application mappings** (chapter 12) for
better readability and reuse.

## The 20-bug audit

The package recently went through an audit pass that fixed
20 bugs across 17 actionable defects + 3 medium-severity
follow-ups, plus a 31-file docstring typo cleanup.

**Commits:**

- `79172d0` — "Fix audit-surfaced bugs across uia platform layers" (17 defects).
- `2479151` — 31-file docstring typo fix ("Represents and instance" → "Represents an instance").
- `e5e4cad` — three more medium-severity bugs.

**Bug classes caught:**

1. **Model layer:** `DeclarativeApplication.attach()`
   referenced an undefined `APPLICATION_NAME` class
   variable; `ScreenElement` had no `__init__`, causing
   `TypeError` on `super().__init__(parent)` from `UIApplication`;
   four bare `except:` clauses replaced with `except Exception:`.
2. **Linux platform:** Broken import path
   (`agents.wm.x11windowmanageragent` →
   `agents.x11.x11windowmanageragent`); `AccessibilityAgent`
   missing abstract `get_value` / `set_value` methods;
   `WaylandGnomeWindowManagerAgent`'s `_last_x` / `_last_y`
   uninitialised when sync=False; non-compliant JSON-RPC
   (hardcoded `req_id=1`, checking for non-existent "ok"
   field); debug `print()` polluting stdout on every
   keystroke; `MenuElement.close()` sending invalid
   `<Esc>` instead of `{ESC}`.
3. **macOS platform:** Bare `except:` and stdout pollution
   on optional HIServices import failure; `SbWindow.groups()`
   debug print loop on every call.
4. **Test suite:** Imports of nonexistent symbols; missing
   Linux platform guards on Linux-specific tests; missing
   defaults on `uia_controller` parameter.

**Test status post-audit:** 16/16 macOS tests passing
(was 9/15 before the first pass).

### The four bundle-identifier verifications

A subtle point: the audit *verified* — and chose not to
change — four legacy Apple bundle identifiers that initially
looked stale:

- **Calendar:** `com.apple.iCal` (the legacy iCal name).
- **Contacts:** `com.apple.AddressBook` (the legacy AddressBook name).
- **Numbers:** `com.apple.iWork.Numbers`.
- **Pages:** `com.apple.iWork.Pages`.

Live macOS still uses these IDs; updating to a "Calendar"
or "Contacts" name would have *broken* the lookup.  The
audit's discipline included verifying against live macOS
before "fixing" what wasn't broken.  The chapter calls this
out because it's a useful reminder: not every plausible
correction is correct.

## What the next chapter does

Chapter 12 walks the **declarative application mappings** —
the project-side mechanism for naming UI elements
semantically.  `DeclarativeApplication`, `ElementHint`, and
the per-app classes (`Calculator`, `MicrosoftExcel`,
`LibreOfficeCalc`) that ship in source.  The chapter shows
how a project declares an application's UI structure once
and writes test code against the *declared names* rather
than against role / role / hint chains.
