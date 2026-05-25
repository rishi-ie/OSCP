# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Purpose

OSCP is a foundational protocol for agent-native OS interaction. It intercepts the compositor's render pipeline at the OS level and delivers raw render operations to agents — enabling them to see and interact with the full desktop just like humans.

**Goal:** Replace unreliable VLM-based screen observation with OS-level compositor interception. No pixels. No screenshots. Just decoded geometry.

---

## Principle

> "Intercept the compositor. Decode the render tree. Agent provides the meaning."

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS (pi)                       │
│                         │                                      │
│            ┌────────────┼────────────┐                          │
│            │            │            │                          │
│            ▼            ▼            ▼                          │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│     │  macOS   │  │  Linux   │  │ Windows  │                    │
│     │ Platform │  │ Platform │  │ Platform │                    │
│     │  Driver  │  │  Driver  │  │  Driver  │                    │
│     └──────────┘  └──────────┘  └──────────┘                    │
│            │            │            │                          │
│            └────────────┼────────────┘                          │
│                         │                                      │
│                    Protocol Layer                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Per-Platform Implementation

| Platform | Method | Coverage | Complexity |
|----------|--------|----------|------------|
| **macOS** | CGWindowListCopyWindowInfo (Window Server) | 95% | Low |
| **Linux** | X11 APIs + Compositor OpenGL Hook | 90-95% | Medium |
| **Windows** | UIA + Win32 APIs | 90% | Low |

### macOS: Window Server Layer Tree

The Window Server is the compositor. It maintains a complete layer tree of all windows. We query this via Core Graphics APIs — no GPU hooks needed.

### Linux: X11 + Compositor Hook

**Tier 1:** X11 window enumeration for X11 desktops and Xwayland apps (~85%)
**Tier 2:** Compositor OpenGL hook for native Wayland apps (~5-10% additional)

### Windows: UIA + Win32

**Tier 1:** Win32 APIs for window enumeration and positions
**Tier 2:** UI Automation for semantic element tree

*Note: Windows lacks documented APIs for render operation extraction. UIA + Win32 provides semantic data (element types, names, states) with 90% coverage.*

---

## What Agent Receives

### macOS/Linux (Render Operations)

```json
{
  "type": "render_tree",
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "focused": true,
      "ops": [
        {
          "id": "op_001",
          "bounds": {"x": 10, "y": 10, "w": 100, "h": 30},
          "z": 1,
          "texture_id": "0xAA01"
        }
      ]
    }
  ],
  "mouse": {"x": 150, "y": 25}
}
```

### Windows (Element Tree)

```json
{
  "type": "render_tree",
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x12345",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25}
        }
      ]
    }
  ],
  "mouse": {"x": 150, "y": 25}
}
```

**Note:** Agent receives same unified format regardless of platform. macOS/Linux get render ops, Windows gets semantic elements. Agent skills fill the meaning gap.

---

## Why These Approaches?

### macOS
- Window Server layer tree is introspectable via CGWindowListCopyWindowInfo
- No GPU hooks needed
- Official, stable API

### Linux
- X11 was designed for introspection (complete window tree)
- Compositor hook captures native Wayland windows
- Unified output hides implementation details

### Windows
- No documented API exposes render operations from DWM
- UIA + Win32 gives semantic data with 90% coverage
- Stable, official, reliable
- Render ops approach deferred to V2 (DWM hook)

---

## Coverage

| Platform | Coverage | Gaps |
|----------|----------|------|
| **macOS** | 95% | Screen sharing, DRM, sandboxed apps |
| **Linux** | 90-95% | Some Wayland compositors, TTY, KMS apps |
| **Windows** | 90% | Non-UIA apps, legacy Win32, protected content |

---

## Implementation Roadmap

| Phase | Description | Timeline |
|-------|-------------|----------|
| **V1** | macOS driver (CGWindowList) | 2-3 months |
| **V1** | Linux driver (X11 + Compositor Hook) | 3-4 months |
| **V1** | Windows driver (UIA + Win32) | 1-2 months |
| **V2** | Windows render ops (DWM hook) | 5-9 months |

---

## Directory Structure

```
OSCP/
├── protocol/           # Protocol specification
│   └── SPEC.md        # Core protocol
│
├── platforms/          # OS-specific drivers
│   ├── macos/         # Window Server (CGWindowList)
│   ├── linux/         # X11 + Compositor Hook
│   └── windows/       # UIA + Win32
│
├── agents/             # Agent integration
│   └── SPEC.md       # Agent SDK guidelines
│
└── README.md          # Project overview
```

---

## Status

🚧 **Phase 0** — Design complete. V1 scope finalized.
🚧 **V1 Development** — All three platforms.

---

*OSCP — First-class access for agents.*