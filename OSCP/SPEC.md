# OSCP Architecture Specification

**Version:** 0.3.0
**Status:** Architecture Finalized

---

## Overview

OSCP is a **wrapper + streaming layer** on top of existing OS accessibility APIs. The hard parts are already built. OSCP adds real-time streaming, unified protocol, and error handling.

---

## Architecture Pattern (All Platforms)

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                                 │
│                                                                 │
│   Agent receives:                                               │
│   {                                                             │
│     "frame_id": 12345,                                         │
│     "platform": "macOS",                                        │
│     "windows": [...],                                           │
│     "tree_analysis": {...},                                    │
│     "mouse": {...}                                             │
│   }                                                            │
└─────────────────────────────────────────────────────────────────┘
                             │
                    OSCP Protocol
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
    ▼                        ▼                        ▼
┌─────────────────────────────────────────────────────────────┐
│                 OSCP PLATFORM DRIVER                         │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │    STREAMING    │  │   ERROR HANDLER │  │   INPUT    │ │
│  │    ENGINE       │  │   + FALLBACKS   │  │   ENGINE   │ │
│  └────────┬────────┘  └────────┬────────┘  └─────┬──────┘ │
│           │                    │                 │          │
│           │              ┌──────▼──────┐         │          │
│           │              │   TREE      │◄────────┘          │
│           │              │   ANALYZER  │                    │
│           │              └──────┬──────┘                    │
│           │                    │                            │
│           │    ┌──────────────┴──────────────┐            │
│           │    │                              │            │
│           │    ▼                              ▼            │
│           │   ▼                                ▼           │
│  ┌────────┴──┴──────────────┐    ┌────────────┴─────────┐  │
│  │   PRIMARY CAPTURE       │    │   FALLBACK METHODS    │  │
│  │   (AXUIElement/AT-SPI2) │    │   (CDP/X11/Position)  │  │
│  └──────────────────────────┘    └──────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                             │
                    Native OS APIs
                             │
                    OS LEVEL ACCESSIBILITY
```

---

## macOS Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 macOS Platform Driver                            │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    STREAMING ENGINE                        │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ AX Observer (30fps) — watches for AXUIElement changes │ │ │
│  │  │ Poll Interval: 33ms                                    │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼────────────────────────────────┐│
│  │                    TREE ANALYZER                            │ │
│  │  • coverage_score                                  │ │
│  │  • named_elements vs unlabeled_elements            │ │
│  │  • confidence: HIGH/MEDIUM/LOW/NONE                │ │
│  └───────────────────────────┬────────────────────────────────┘│
│                              │                                  │
│                 ┌───────────┴───────────┐                     │
│                 ▼                       ▼                       │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐│
│  │    PRIMARY CAPTURE       │  │    FALLBACK METHODS          ││
│  │    ──────────────────    │  │    ─────────────────────     ││
│  │                          │  │                              ││
│  │    AXUIElement           │  │    CDP Bridge                ││
│  │    ─────────────         │  │    (Safari/Chrome/Electron)  ││
│  │    AppKit, SwiftUI       │  │                              ││
│  │    Standard controls    │  │    Screen Region (OBSOLETE)  ││
│  │    50-100ms capture      │  │    (Not used — no VLMs)      ││
│  │                          │  │                              ││
│  │    Coverage: 90%        │  │    Position-Only Mode        ││
│  │                          │  │    (Metal/OpenGL games)      ││
│  │                          │  │                              ││
│  │                          │  │    Human Handoff             ││
│  └──────────────────────────┘  └──────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    INPUT ENGINE                           │ │
│  │  CGEventPost — click, type, key combos                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Wrapped Technology

| Component | Source | OSCP Role |
|-----------|--------|-----------|
| **AXUIElement** | Native macOS API | Extract semantic tree |
| **AXObserver** | Native macOS API | Real-time change notifications |
| **AXUIElementCopyElementAtPosition** | Native macOS API | Hit testing |
| **CGEvent** | Native macOS API | Input injection |

### Existing Libraries

```
WRAPPERS (OSCP uses these):
├── pyax — Python AXUIElement wrapper
├── ax-element — Rust AXUIElement bindings
└── accessibility-service — System-level API

OSCP DOES NOT REIMPLEMENT:
├── How to query AXUIElement attributes
├── How to traverse element hierarchy
└── How to get bounds from AXUIElement
```

### What OSCP Adds on macOS

```
WHAT OSCP ADDS:
├── Real-time streaming (AXObserver loop at 30fps)
├── Tree quality analysis
├── Unified error handling with fallbacks
├── Protocol server (Unix socket)
├── CDP bridge for Safari/Chrome
├── CGEvent input engine
└── Agent-friendly JSON output

WHAT OSCP DOES NOT REIMPLEMENT:
├── AXUIElement querying
├── Element tree traversal
└── Attribute extraction
```

---

## Linux Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 Linux Platform Driver                            │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    STREAMING ENGINE                        │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ AT-SPI2 Observer — watches for accessibility events  │ │ │
│  │  │ + Poll Interval for D-Bus (fallback)                │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼────────────────────────────────┐│
│  │                    TREE ANALYZER                            │ │
│  │  • coverage_score                                  │ │
│  │  • named_elements vs unlabeled_elements            │ │
│  │  • confidence: HIGH/MEDIUM/LOW/NONE                │ │
│  └───────────────────────────┬────────────────────────────────┘│
│                              │                                  │
│                 ┌───────────┴───────────┐                     │
│                 ▼                       ▼                       │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐│
│  │    PRIMARY CAPTURE       │  │    FALLBACK METHODS          ││
│  │                          │  │                              ││
│  │    AT-SPI2              │  │    CDP Bridge                ││
│  │    ──────────           │  │    (Chrome/Firefox/Electron) ││
│  │    GTK, Qt, Swing       │  │                              ││
│  │    Standard controls    │  │    X11                       ││
│  │    ~50ms capture        │  │    (XQueryTree)              ││
│  │    Coverage: 85%        │  │    — X11 desktops            ││
│  │                          │  │    — Xwayland apps           ││
│  │                          │  │                              ││
│  │                          │  │    Position-Only Mode        ││
│  │                          │  │                              ││
│  │                          │  │    Human Handoff             ││
│  └──────────────────────────┘  └──────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    INPUT ENGINE                             │ │
│  │  /dev/uinput — click, type, key combos                     │ │
│  │  XTest fallback (X11 desktops)                             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Wrapped Technology

| Component | Source | OSCP Role |
|-----------|--------|-----------|
| **AT-SPI2** | D-Bus accessibility bus | Extract semantic tree |
| **at-spi-bus** | System daemon | Application enumeration |
| **org.a11y.Bus** | D-Bus | AT-SPI2 connection |
| **X11** | X Window System | X11 + Xwayland fallback |
| **/dev/uinput** | Linux kernel | Input injection |

### Existing Libraries

```
WRAPPERS (OSCP uses these):
├── dogtail — Python AT-SPI2 library
├── pyatspi — PyAT-SPI2 bindings
├── ldtp — Linux Desktop Testing Project
├── at-spi2-core — System accessibility daemon

OSCP DOES NOT REIMPLEMENT:
├── How to connect to at-spi-bus
├── How to query AT-SPI2 objects
├── How to get bounds from AT-SPI2
└── How to handle D-Bus events
```

### What OSCP Adds on Linux

```
WHAT OSCP ADDS:
├── Real-time streaming (AT-SPI2 events + polling)
├── Tree quality analysis
├── X11 fallback for non-AT-SPI2 apps
├── Unified error handling with fallbacks
├── Protocol server (Unix socket)
├── CDP bridge for Chrome/Firefox
├── /dev/uinput input engine
└── Agent-friendly JSON output

WHAT OSCP DOES NOT REIMPLEMENT:
├── AT-SPI2 D-Bus connection
├── Element tree traversal
├── Application enumeration
└── Bounds extraction
```

---

## OSCP Layer (Shared)

```
┌─────────────────────────────────────────────────────────────────┐
│                 OSCP LAYER (Cross-Platform)                     │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 PROTOCOL SERVER                            │ │
│  │                 ─────────────────                          │ │
│  │  Transport: Unix socket (/tmp/oscp.sock)                │ │
│  │  Protocol: JSON over TCP-like stream                     │ │
│  │  Framing: 30fps render tree frames                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 TREE BUILDER                              │ │
│  │                 ────────────                               │ │
│  │  Converts native elements → OSCP element format          │ │
│  │  { id, type, name, bounds, states, confidence, source }   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 ERROR HANDLER                              │ │
│  │                 ──────────────                              │ │
│  │  Fallback hierarchy:                                       │ │
│  │  1. Native tree (primary)                                  │ │
│  │  2. CDP bridge (browser support)                          │ │
│  │  3. X11 | Screen region (platform fallback)               │ │
│  │  4. Position-only mode                                     │ │
│  │  5. Human handoff                                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 TREE ANALYZER                               │ │
│  │                 ─────────────                               │ │
│  │  coverage_score: 0.0-1.0                                  │ │
│  │  named_elements: count                                      │ │
│  │  unnamed_elements: count                                    │ │
│  │  confidence: HIGH (>0.8) | MEDIUM | LOW | NONE (<0.2)     │ │
│  │  recommended_action: execute | explore | handoff          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

```
1. NATIVE CAPTURE (AXUIElement or AT-SPI2)
   │
   │ Raw element hierarchy
   ▼
2. TREE BUILDER (OSCP layer)
   │
   │ Standardized element format
   ▼
3. TREE ANALYZER
   │
   │ Quality metrics
   ▼
4. IF QUALITY < THRESHOLD:
   │
   │ Primary tree empty/unhelpful
   ▼
   FALLBACK CHAIN (try next method)
   │
5. PROTOCOL SERVER
   │
   │ JSON format
   ▼
6. AGENT CLIENTS
```

---

## Output Format

### Standard Frame

```json
{
  "type": "render_tree",
  "frame_id": 12345,
  "platform": "macOS",
  "timestamp": 1716576000000,
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "pid": 1234,
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "description": "",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "states": ["enabled", "visible"],
          "confidence": 0.95,
          "source": "axuielement"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "avg_depth": 4.2,
    "confidence": "HIGH",
    "recommended_action": "execute"
  },
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_element_id": "e_042"
  }
}
```

### Fallback Frame

```json
{
  "type": "render_tree",
  "frame_id": 12346,
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x500001",
      "title": "Game",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "elements": [],
      "fallback_active": true,
      "fallback_method": "position_only",
      "fallback_reason": "custom_renderer_detected"
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.05,
    "named_elements": 0,
    "unlabeled_elements": 1,
    "avg_depth": 1,
    "confidence": "NONE",
    "recommended_action": "explore_or_handoff"
  }
}
```

---

## Confidence Decision Table

| Tree Confidence | Threshold | Agent Action |
|-----------------|-----------|--------------|
| **HIGH** | > 0.8 coverage, >50% named | Execute immediately |
| **MEDIUM** | 0.5-0.8 coverage | Execute + monitor state |
| **LOW** | 0.3-0.5 coverage | Explore candidates first |
| **NONE** | < 0.3 coverage | Explore + confirm or handoff |

---

## Implementation Stack

| Layer | Technology |
|-------|------------|
| **Platform Driver** | C/Rust (macOS), Python/Rust (Linux) |
| **Protocol Server** | Any language with Unix socket + JSON |
| **Agent SDK** | Python, TypeScript (or existing pi SDK) |
| **Input Engine** | CGEvent (macOS), /dev/uinput (Linux) |

---

## File Structure

```
OSCP/
├── protocol/
│   └── SPEC.md                    # Protocol format
│
├── platforms/
│   ├── macos/
│   │   ├── SPEC.md               # macOS approach
│   │   ├── driver/               # AXUIElement wrapper + streaming
│   │   └── input/                # CGEvent engine
│   │
│   └── linux/
│       ├── SPEC.md               # Linux approach
│       ├── driver/               # AT-SPI2 wrapper + streaming
│       └── input/                # /dev/uinput engine
│
├── agents/
│   └── SPEC.md                   # Agent SDK guidelines
│
└── README.md
```

---

## Complexity Summary

| Platform | Wraps | OSCP Adds | Total Time |
|----------|-------|-----------|------------|
| **macOS** | AXUIElement (existing) | Streaming + Protocol + Fallbacks | 4-6 weeks |
| **Linux** | AT-SPI2 + X11 (existing) | Streaming + Protocol + Fallbacks | 6-8 weeks |

---

## Status

- [x] Architecture documented
- [x] Existing tools identified
- [x] OSCP layer defined
- [x] Fallback hierarchy defined
- [ ] Implementation begins