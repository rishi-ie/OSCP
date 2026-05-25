# OSCP Linux Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The Linux platform driver wraps AT-SPI2 and X11 into a real-time, streaming, error-resilient interface.

**Key Insight:** AT-SPI2 and X11 already extract the semantic tree. OSCP adds streaming, unification, and error handling.

---

## Wrapped Technologies

### AT-SPI2 (D-Bus Accessibility)

```python
# Native Linux accessibility:
# AT-SPI2 is the standard D-Bus accessibility API
# Used by: Orca screen reader, Accerciser, dogtail

# Existing wrappers:
# - dogtail (Python)
# - pyatspi (PyAT-SPI2)
# - ldtp (Linux Desktop Testing Project)
# - at-spi2-core (system daemon)
```

### What AT-SPI2 Provides

```python
struct AT_SPIElement:
    role: String           # ROLE_BUTTON, ROLE_MENU, etc.
    name: String           # Accessible name
    bounds: Rect           # Position and size
    states: [State]       # STATE_ENABLED, etc.
    children: [AT_SPIElement]
```

### X11 (Fallback)

```python
# XQueryTree, XGetWindowProperty, XGetWindowAttributes
# Covers: X11 desktops + Xwayland apps (~85% of Linux)
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  AT-SPI2         │  │   X11           │  │   Input      │  │
│  │  (wrapped)       │  │   (fallback)    │  │   Engine     │  │
│  │  EXISTING        │  │  EXISTING       │  │  (/dev/uinput)│  │
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬──────┘  │
│           │                     │                   │          │
│           └──────────┬──────────┘                   │          │
│                      │                              │          │
│               ┌──────▼──────┐                       │          │
│               │   Stream    │◄──────────────────────┘          │
│               │   + Error   │                              │
│               │   Handler   │                              │
│               └──────┬──────┘                              │
│                      │                                     │
│               ┌──────▼──────┐                              │
│               │  Protocol   │                              │
│               │  Server     │                              │
│               │  (Unix)     │                              │
│               └─────────────┘                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Fallback Hierarchy

### Level 1: AT-SPI2 (Primary)

```python
# Wrapped from existing tools (dogtail, pyatspi)
def capture_atspi() -> ElementTree:
    # Connect to at-spi-bus
    # Enumerate applications
    # Extract element tree
```

**Coverage: ~85%**

### Level 2: X11 (X11 + Xwayland)

```python
# XQueryTree, XGetWindowProperty
def capture_x11() -> PartialTree:
    # Window hierarchy
    # Metadata extraction
```

**Coverage: +5% (X11 desktops, Xwayland apps)**

### Level 3: CDP Bridge (Electron/Browser)

```python
# Chrome, Firefox, Electron apps
def capture_cdp() -> ElementTree:
    # CDP port connection
    # DOM extraction
```

### Level 4: Heuristics + Position-Only

```python
# For SDL/GL games and custom renderers
def position_only(window: Window) -> EmptyTree:
    return WindowBounds(window)
```

### Level 5: Human Handoff

```python
def human_handoff(reason: String) -> HandoffRequest:
    return HandoffRequest(...)
```

---

## Wayland Compositor Support

### Detection

```python
def detect_compositor() -> WaylandCompositor:
    # GNOME (Mutter), KDE (KWin), Sway, etc.
```

### Per-Compositor Bridge (Optional)

```python
# For native Wayland apps, per-compositor IPC:
# - GNOME: org.gnome.Shell.Introspect (dbus)
# - KDE: KWin scripting
# - Sway: swaymsg IPC
```

---

## Error Detection

### Tree Quality Analysis

```python
struct TreeAnalysis:
    coverageScore: Float
    namedElements: Int
    unlabeledElements: Int
    avgDepth: Float
    confidence: Confidence  # HIGH, MEDIUM, LOW, NONE
```

### Detection Rules

```python
if coverageScore < 0.3: triggerFallback()
if namedRatio < 0.5: triggerLowConfidence()
if avgDepth < 2: triggerCustomRenderer()
```

---

## Revised Complexity

| Component | Complexity | Notes |
|-----------|------------|-------|
| **AT-SPI2 wrapping** | Medium | D-Bus complexity |
| **X11 fallback** | Low | Already exists |
| **Real-time streaming** | Medium | 2-3 weeks |
| **Wayland support** | High | Per-compositor IPC |
| **Error handling** | Low | 1 week |
| **Input engine** | Medium | /dev/uinput |
| **Testing** | High | Multiple distros |

**Time Estimate: 6-8 weeks**

---

## Installation

```bash
# apt
sudo apt install oscp

# Requires:
# - at-spi2-core package
# - Accessibility enabled in desktop environment
```

---

## Status

🚧 **V1 Target:** Wrapped AT-SPI2 + X11 + streaming + fallbacks
⏱️ **Time:** 6-8 weeks

---

## References

- [AT-SPI2 Specification](https://www.freedesktop.org/wiki/Accessibility/)
- [dogtail](https://github.com/nick96/dogtail) - Python AT-SPI2 library
- [pyatspi](https://github.com/python-atspi/pyatspi) - PyAT-SPI2 bindings