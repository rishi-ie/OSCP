# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Early Design

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

Today, agents attempt to interact with computers using VLMs — observing pixels, guessing element positions, and reacting to visual output. This approach is **fragile, slow, and unreliable**.

OSCP intercepts the compositor's render pipeline at the OS level — not per-app — and delivers raw render operations to agents. No pixels. No screenshots. Just decoded geometry.

---

## Principle

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Intercept the compositor. Decode the render tree.             │
│   Agent provides the meaning.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**VLM approach:**
```
Compositor ─► Pixels ─► VLM guesses ─► Agent guesses
```

**OSCP approach:**
```
Compositor ─► Render tree ─► Decode ─► Agent knows
             (intercept here)  (ops)     (exact positions)
```

**Per-app hook approach:**
```
App A ─► DXGI ─► Shared memory ─► Decoder ─► Agent (85% coverage)
App B ─► OpenGL ─► ❌ missed
App C ─► Vulkan ─► ❌ missed
```

**OSCP approach:**
```
App A ─┐
App B ─┼─► Compositor ─► INTERCEPT ─► Decode ─► Agent (100% coverage)
App C ─┘        ↑
              One point. All apps.
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                               │
│                                                                 │
│   Agent receives:                                               │
│   {                                                             │
│     "render_tree": {                                            │
│       "windows": [{                                             │
│         "id": "win_1",                                          │
│         "title": "VS Code",                                     │
│         "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},       │
│         "ops": [{"id": "op_1", "bounds": {...}, "z": 0}]         │
│       }]                                                        │
│     },                                                          │
│     "mouse": {"x": 540, "y": 320}                               │
│   }                                                            │
│                                                                 │
│   Agent sends: {"action": "click", "x": 100, "y": 200}         │
└─────────────────────────────────────────────────────────────────┘
                             │
                       OSCP Protocol
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
    ▼                        ▼                        ▼
┌──────────┐          ┌──────────┐          ┌──────────┐
│ Windows  │          │  macOS   │          │  Linux   │
│ Platform │          │ Platform │          │ Platform │
│  Driver  │          │  Driver  │          │  Driver  │
└──────────┘          └──────────┘          └──────────┘
     │                     │                     │
     ▼                     ▼                     ▼
 DWM/Compositor      Window Server         X11/Wayland
   (global)           (global)            Compositor
```

---

## Key Differences from Per-App Hook

| Aspect | Per-App Hook | OSCP (Compositor) |
|--------|--------------|-------------------|
| **Coverage** | 85% (DirectX only) | 100% (all apps) |
| **Interception point** | Each app's DXGI | Compositor output |
| **OpenGL** | ❌ No | ✅ Yes |
| **Vulkan** | ❌ No | ✅ Yes |
| **Games** | ❌ No | ✅ Yes |
| **Complexity** | High (inject each app) | Lower (one global point) |
| **Data** | Per-app render ops | Global scene tree |

---

## What Agent Receives

| Field | Description |
|-------|-------------|
| `render_tree` | Full compositor scene tree |
| `windows` | All visible windows with metadata |
| `ops` | Render operations per window (bounds, z, texture) |
| `mouse` | Cursor position and hovered element |

**Agent provides:**
- Element semantics (button vs label vs input)
- Interaction logic (what's clickable)
- State inference (enabled, disabled, checked)
- Visual reasoning (from geometry patterns)

---

## Platforms

| Platform | Compositor | Intercept Method | Status |
|----------|-----------|------------------|--------|
| **Windows** | DWM | Graphics Capture API / DWM internals | Draft |
| **macOS** | Window Server | Layer tree extraction | Draft |
| **Linux** | X11/Wayland | Scene graph / compositor protocols | Draft |

---

## Quick Start

### Install

```bash
# Windows
winget install OSCP.Windows

# macOS
brew install oscp

# Linux
sudo apt install oscp
```

### Connect

```python
import oscp

client = oscp.connect("tcp://localhost:9876")  # Windows
# or
client = oscp.connect("unix:///tmp/oscp.sock")  # macOS/Linux

# Receive render tree
async for tree in client.stream():
    for window in tree.windows:
        print(f"Window: {window.title}")
        for op in window.ops:
            print(f"  rect: {op.bounds}, z: {op.z}")

# Perform action
client.click(x=100, y=200)
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [SPEC.md](SPEC.md) | Project overview and structure |
| [protocol/SPEC.md](protocol/SPEC.md) | Core protocol: messages, transport, types |
| [platforms/windows/SPEC.md](platforms/windows/SPEC.md) | Windows compositor intercept |
| [platforms/macos/SPEC.md](platforms/macos/SPEC.md) | macOS Window Server intercept |
| [platforms/linux/SPEC.md](platforms/linux/SPEC.md) | Linux compositor intercept |
| [agents/SPEC.md](agents/SPEC.md) | Agent integration guidelines |

---

## Project Structure

```
OSCP/
├── README.md                 # Project overview
├── SPEC.md                   # Main specification
├── protocol/                 # Protocol specification
│   └── SPEC.md              # Core protocol
├── platforms/               # OS-specific drivers
│   ├── windows/             # Windows compositor intercept
│   ├── macos/               # macOS Window Server intercept
│   └── linux/               # Linux compositor intercept
├── agents/                   # Agent integration
│   └── SPEC.md             # Agent SDK guidelines
└── MEMORY/                   # Project context
    ├── project-overview.md
    └── DECISIONS.md
```

---

## Why Not Per-App Hook?

Per-app hooking (DXGI hook) misses:
- OpenGL applications
- Vulkan applications
- Games with anti-cheat
- Some legacy GDI apps
- Java Swing with software rendering

OSCP intercepts at the compositor — one point, all apps.

---

## Why Not Screenshots?

Screenshots reintroduce VLM dependency:
- Slow (2-5 seconds per frame)
- Expensive (VLM token cost)
- Unreliable (VLM guessing)
- No structural data (just pixels)

OSCP delivers render operations — exact geometry, not fuzzy pixels.

---

## Roadmap

| Phase | Milestone |
|-------|-----------|
| **V0.1** | Protocol and spec finalization |
| **V0.2** | Windows compositor driver |
| **V0.3** | macOS Window Server driver |
| **V0.4** | Linux compositor driver |
| **V1.0** | Cross-platform unification |
| **V1.1** | Agent SDKs (OpenClaw, HermesAgent, PyAgent) |

---

## Core Principle

> Humans interact through interfaces.
> Agents interact through meaning.

OSCP provides the geometry from the compositor's render tree. Agent provides the meaning. Execution remains deterministic.

---

## Status

🚧 **Phase 0** — Specifications updated. Compositor interception approach.

---

*OSCP — First-class access for agents.*