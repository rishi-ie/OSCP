# OSCP — Operating System Context Protocol

**Version:** 0.2.0 (Draft)
**Status:** Design Finalized

---

## Purpose

OSCP is a foundational protocol for agent-native OS interaction. It intercepts the compositor's render pipeline at the OS level and delivers raw render operations to agents — enabling them to see and interact with the full desktop just like humans.

**Goal:** Replace unreliable VLM-based screen observation with OS-level compositor interception. No pixels. No screenshots. Just decoded geometry.

---

## Scope (V1)

**Platforms:** macOS and Linux only

| Platform | Compositor | Intercept Method |
|----------|-----------|------------------|
| **macOS** | Window Server | Layer tree extraction |
| **Linux** | X11/Wayland | Scene graph |

Windows support deferred to V2 (UIA + Win32 hybrid approach).

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
│                                                                 │
│            ┌────────────────┬────────────────┐                 │
│            │                │                │                  │
│            ▼                ▼                ▼                  │
│     ┌──────────┐      ┌──────────┐                                 │
│     │  macOS   │      │  Linux   │                                 │
│     │ Platform │      │ Platform │                                 │
│     │  Driver  │      │  Driver  │                                 │
│     └──────────┘      └──────────┘                                 │
│            │                │                                      │
│            └────────────────┴────────────────┘                    │
│                         │                                        │
│                    Protocol Layer                                │
└─────────────────────────────────────────────────────────────────┘
```

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
| `bounds` | object | Window bounds |
| `focused` | boolean | Window focus state |
| `ops` | array | Render operations in this window |

### Render Operation

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique operation identifier |
| `bounds` | object | Position (x, y) and size (w, h) |
| `z` | number | Z-index (render order) |
| `texture_id` | string | Texture identifier |
| `clip_bounds` | object | Optional clipping rect |

---

## Platform-Specific Approaches

### macOS

**Compositor:** Window Server
**Intercept:** Layer tree extraction via Core Graphics
**Input:** CGEvent

### Linux

**Compositor:** X11 / Wayland
**Intercept:** X11 window tree + Wayland compositor protocols
**Input:** XTest

---

## Why macOS and Linux?

### macOS
- Window Server is introspectable
- Layer tree accessible via Core Graphics APIs
- CGWindowListCopyWindowInfo provides metadata + structure

### Linux (X11)
- X11 was designed for introspection
- Full window tree, properties, and region tracking
- No permissions needed (same user)

### Linux (Wayland)
- Compositor protocols expose surface hierarchy
- DMABUF for texture tracking
- Fallback to X11 if needed

### Windows (Deferred to V2)
- No documented APIs expose render ops from DWM
- Would require undocumented hooks or kernel driver
- UIA + Win32 hybrid possible but not true render ops

---

## Implementation Roadmap

| Phase | Description |
|-------|-------------|
| **V0** | Protocol design and spec finalization |
| **V1** | macOS platform driver (Window Server) |
| **V1** | Linux platform driver (X11/Wayland) |
| **V2** | Windows platform driver (UIA + Win32 hybrid) |
| **V3** | Agent SDKs and integration |

---

## Directory Structure

```
OSCP/
├── protocol/           # Protocol specification
│   └── SPEC.md        # Core protocol
│
├── platforms/          # OS-specific drivers
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

🚧 **Phase 0** — Design complete. macOS and Linux V1 scope defined. Windows deferred.

---

*OSCP — First-class access for agents.*