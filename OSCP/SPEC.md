# OSCP — Operating System Context Protocol

**Version:** 0.1.0 (Draft)
**Status:** Design finalization complete

---

## Purpose

OSCP is a foundational protocol for agent-native OS interaction. It provides agents with raw render geometry from the desktop interface — positions, z-order, texture IDs — enabling them to see, interact with, and manage GUI applications just like humans.

**Goal:** Replace unreliable VLM-based screen observation with native render interception. Agent receives geometry. Agent provides meaning.

---

## Principle

> "Intercept at the source. Deliver the geometry. Agent provides the meaning."

```
Traditional (VLM):
App → Render → Pixels → Agent guesses from visual patterns

OSCP:
App → Render → INTERCEPT HERE → Raw geometry → Agent reasons
                     ↓
              Positions, z-order, texture IDs
              (exact, deterministic)
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

### Layers

| Layer | Description |
|-------|-------------|
| **Platform Driver** | OS-specific render interception |
| **Protocol** | Transport, framing, message types |
| **Agent Interface** | Unified API for any agent |

---

## What Agent Receives

| Field | Type | Description |
|-------|------|-------------|
| `frame_id` | number | Unique frame identifier |
| `timestamp` | number | Frame timestamp (ms) |
| `surfaces` | array | List of windows/surfaces |
| `mouse` | object | Cursor position and hovered element |

### Surface

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Surface identifier |
| `title` | string | Window title |
| `bounds` | object | Window bounds (x, y, w, h) |
| `focused` | boolean | Window focus state |
| `render_ops` | array | Draw operations from render pipeline |

### Render Operation

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique operation identifier |
| `bounds` | object | Position (x, y) and size (w, h) |
| `z` | number | Z-index (render order) |
| `texture_id` | string | Texture identifier |
| `clip_bounds` | object | Optional clipping rect |

### Mouse

| Field | Type | Description |
|-------|------|-------------|
| `x` | number | Cursor X position |
| `y` | number | Cursor Y position |
| `hovered_op_id` | string | Render op under cursor (if any) |

---

## Platform-Specific Approaches

### Windows

**Capture:** DXGI hook intercepts DirectX render operations
**Input:** SendInput, PostMessage

### macOS

**Capture:** CGWindowList API for window content
**Input:** CGEvent

### Linux

**Capture:** X11 window introspection (XRecord)
**Input:** XTest

---

## Implementation Roadmap

| Phase | Description |
|-------|-------------|
| **V0** | Protocol design and spec finalization |
| **V1** | Windows platform driver (DXGI hook + actions) |
| **V2** | macOS platform driver |
| **V3** | Linux platform driver |
| **V4** | Cross-platform unification and agent SDK |

---

## Directory Structure

```
OSCP/
├── protocol/           # Protocol specification
│   └── SPEC.md        # Core protocol spec
│
├── platforms/          # OS-specific drivers
│   ├── windows/       # Windows implementation (DXGI hook)
│   ├── macos/         # macOS implementation
│   └── linux/         # Linux implementation
│
├── agents/             # Agent integration adapters
│   └── SPEC.md       # Agent SDK guidelines
│
└── README.md          # Project overview
```

---

## Status

🚧 **Phase 0** — Protocol design complete. Specs finalized.

---

*OSCP — First-class access for agents.*