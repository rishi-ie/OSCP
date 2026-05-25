# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP intercepts the compositor's render pipeline at the OS level and delivers raw render operations to agents. No pixels. No screenshots. No VLM.

---

## Scope (V1)

**Platforms:** macOS and Linux only

| Platform | Compositor | Intercept Method | Status |
|----------|-----------|------------------|--------|
| **macOS** | Window Server | Layer tree extraction | V1 Target |
| **Linux** | X11/Wayland | Scene graph | V1 Target |
| **Windows** | DWM | Deferred | V2 |

---

## Principle

```
Intercept the compositor. Decode the render tree. Agent provides the meaning.
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                               │
│                                                                 │
│   Agent receives:                                               │
│   {                                                             │
│     "windows": [{                                               │
│       "id": "win_1",                                            │
│       "title": "VS Code",                                       │
│       "ops": [{"id": "op_1", "bounds": {...}, "z": 0}]         │
│     }],                                                         │
│     "mouse": {"x": 540, "y": 320}                              │
│   }                                                            │
│                                                                 │
│   Agent sends: {"action": "click", "x": 100, "y": 200}         │
└─────────────────────────────────────────────────────────────────┘
                             │
                       OSCP Protocol
                             │
         ┌───────────────────┴───────────────────┐
         │                                       │
         ▼                                       ▼
┌──────────────────┐                 ┌──────────────────┐
│      macOS       │                 │      Linux       │
│     Platform     │                 │     Platform     │
│      Driver      │                 │      Driver      │
└──────────────────┘                 └──────────────────┘
         │                                       │
         ▼                                       ▼
 Window Server                     X11 / Wayland
  Layer Tree                      Compositor
```

---

## Coverage

| Platform | Coverage | Notes |
|----------|----------|-------|
| **macOS** | ~95% | AppKit, Metal, all renderers |
| **Linux** | ~90% | X11 + Wayland compositors |
| **Windows** | Deferred | UIA + Win32 for V2 |

---

## Quick Start

### Install

```bash
# macOS
brew install oscp

# Linux
# apt
sudo apt install oscp

# dnf
sudo dnf install oscp
```

### Connect

```python
import oscp

# macOS: Unix socket
client = oscp.connect("unix:///tmp/oscp.sock")

# Linux: Unix socket
client = oscp.connect("unix:///tmp/oscp.sock")

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

## Project Structure

```
OSCP/
├── README.md
├── SPEC.md
├── protocol/SPEC.md           # Core protocol
├── platforms/
│   ├── macos/SPEC.md         # Window Server intercept
│   └── linux/SPEC.md         # X11/Wayland compositor
├── agents/SPEC.md
└── MEMORY/
    ├── project-overview.md
    └── DECISIONS.md
```

---

## Core Principle

> Humans interact through interfaces.
> Agents interact through meaning.

OSCP provides geometry from the compositor's render tree. Agent provides the meaning.

---

## Status

🚧 **V1 Development** — macOS and Linux drivers
🚧 **V2 Planning** — Windows (UIA + Win32 hybrid)

---

*OSCP — First-class access for agents.*