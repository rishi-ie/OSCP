# OSCP — Operating System Context Protocol

**Version:** 0.2.0 (Draft)
**Status:** Design finalization complete

---

## Purpose

OSCP is a foundational protocol for agent-native OS interaction. It intercepts the compositor's render pipeline at the OS level and delivers raw render operations to agents — enabling them to see and interact with the full desktop just like humans.

**Goal:** Replace unreliable VLM-based screen observation with OS-level compositor interception. No pixels. No screenshots. Just decoded geometry from one interception point covering all apps.

---

## Principle

> "Intercept the compositor. Decode the render tree. Agent provides the meaning."

```
VLM approach:
Compositor ─► Pixels ─► VLM guesses ─► Agent guesses

OSCP approach:
Compositor ─► Render tree ─► Decode ─► Agent knows exact positions
             (intercept here)
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS (pi)                          │
│                         │                                      │
│            ┌────────────┼────────────┐                          │
│            │            │            │                          │
│            ▼            ▼            ▼                          │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│     │ Windows  │  │  macOS   │  │  Linux   │                    │
│     │ Platform │  │ Platform │  │ Platform │                    │
│     │   Driver │  │   Driver │  │   Driver │                    │
│     └──────────┘  └──────────┘  └──────────┘                    │
│            │            │            │                          │
│            └────────────┼────────────┘                          │
│                         │                                      │
│                    Protocol Layer                               │
└─────────────────────────────────────────────────────────────────┘
```

### The Intercept Point

```
App A (OpenGL) ─┐
App B (Vulkan)  ─┼─► Compositor ─► INTERCEPT ─► Decode ─► Agent
App C (DXGI)    ─┘       ↑
                    One point. All apps.
```

### Layers

| Layer | Description |
|-------|-------------|
| **Platform Driver** | OS-level compositor interception |
| **Protocol** | Transport, framing, message types |
| **Agent Interface** | Unified API for any agent |

---

## What Agent Receives

| Field | Type | Description |
|-------|------|-------------|
| `render_tree` | object | Full compositor scene tree |
| `windows` | array | All visible windows |
| `ops` | array | Render operations (bounds, z, texture) |
| `mouse` | object | Cursor position and hovered element |

### Window

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Window identifier |
| `title` | string | Window title |
| `bounds` | object | Window bounds (x, y, w, h) |
| `focused` | boolean | Window focus state |
| `ops` | array | Render operations in this window |

### Render Operation

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique operation identifier |
| `bounds` | object | Position (x, y) and size (w, h) |
| `z` | number | Z-index (render order, higher = on top) |
| `texture_id` | string | Texture identifier |
| `clip_bounds` | object | Optional clipping rect |

### Mouse

| Field | Type | Description |
|-------|------|-------------|
| `x` | number | Cursor X position |
| `y` | number | Cursor Y position |
| `hovered_op_id` | string | Render op under cursor |

---

## Platform-Specific Approaches

### Windows

**Compositor:** Desktop Window Manager (DWM)
**Intercept:** Graphics Capture API / DWM scene capture
**Input:** SendInput, PostMessage

### macOS

**Compositor:** Window Server
**Intercept:** Layer tree extraction via Core Graphics
**Input:** CGEvent

### Linux

**Compositor:** X11 / Wayland
**Intercept:** X11 damage tracking / Wayland compositor protocols
**Input:** XTest

---

## Why Compositor Interception?

### Per-App Hook vs Compositor Interception

| Aspect | Per-App Hook | Compositor Interception |
|--------|--------------|-------------------------|
| **Coverage** | 85% | 100% |
| **Interception point** | Each app | One global |
| **OpenGL** | ❌ No | ✅ Yes |
| **Vulkan** | ❌ No | ✅ Yes |
| **Games** | ❌ No | ✅ Yes |
| **Anti-cheat** | ❌ Blocked | ✅ Captures output |
| **Complexity** | High | Lower |
| **Data** | Per-app ops | Global scene tree |

### Screen Recording vs Compositor Interception

| Aspect | Screen Recording | Compositor Interception |
|--------|-----------------|------------------------|
| **Output** | Pixels | Render ops |
| **VLM needed** | Yes | No |
| **Speed** | Slow (2-5 sec) | Fast (<50ms) |
| **Cost** | High (tokens) | Low (raw data) |
| **Precision** | Fuzzy | Exact |
| **Structural data** | None | Full |

---

## Implementation Roadmap

| Phase | Description |
|-------|-------------|
| **V0** | Protocol design and spec finalization |
| **V1** | Windows compositor driver (DWM) |
| **V2** | macOS compositor driver |
| **V3** | Linux compositor driver |
| **V4** | Cross-platform unification and agent SDK |

---

## Directory Structure

```
OSCP/
├── protocol/           # Protocol specification
│   └── SPEC.md        # Core protocol
│
├── platforms/          # OS-specific drivers
│   ├── windows/       # Windows compositor intercept
│   ├── macos/         # macOS Window Server intercept
│   └── linux/         # Linux compositor intercept
│
├── agents/             # Agent integration
│   └── SPEC.md       # Agent SDK guidelines
│
└── README.md          # Project overview
```

---

## Status

🚧 **Phase 0** — Protocol design complete. Compositor interception approach finalized.

---

*OSCP — First-class access for agents.*