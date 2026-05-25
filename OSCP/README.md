# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP intercepts the compositor's render pipeline at the OS level and delivers raw geometry to agents. No pixels. No screenshots. No VLM dependency.

---

## Principle

> "Intercept the compositor. Decode the render tree. Agent provides the meaning."

---

## Scope (V1)

| Platform | Approach | Coverage | Time | Risk |
|----------|----------|----------|------|------|
| **macOS** | CGWindowList (Window Server) | 95% | 2-3 mo | Low |
| **Linux** | X11 + Compositor Hook | 90-95% | 3-4 mo | Medium |
| **Windows** | UIA + Win32 | 90% | 1-2 mo | Very Low |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                           │
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
└─────────────────────────────────────────────────────────────┘
                             │
                       OSCP Protocol
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
    ▼                        ▼                        ▼
┌──────────┐          ┌──────────┐          ┌──────────┐
│  macOS   │          │  Linux   │          │ Windows  │
│ Platform │          │ Platform │          │ Platform │
│  Driver  │          │  Driver  │          │  Driver  │
└──────────┘          └──────────┘          └──────────┘
     │                     │                     │
     ▼                     ▼                     ▼
 Window Server        X11/Compositor          UIA + Win32
```

---

## Per-Platform Approaches

### macOS

**Method:** `CGWindowListCopyWindowInfo` (Window Server)

- Window Server is the compositor
- Layer tree accessible via Core Graphics APIs
- No GPU hooks needed
- Official, stable API

### Linux

**Method:** X11 APIs + Compositor OpenGL Hook

**Tier 1:** X11 (primary)
- `XQueryTree` for window hierarchy
- `XGetWindowProperty` for metadata
- Covers X11 desktops + Xwayland apps (~85%)

**Tier 2:** Compositor Hook (Wayland native)
- Hooks: Mutter, KWin, Sway OpenGL/EGL
- One hook = full Wayland scene
- Adds ~10% coverage

### Windows

**Method:** UIA + Win32 APIs

**Tier 1:** Win32 APIs
- `EnumWindows` for window list
- `GetWindowRect` for positions
- `GetWindowText` for titles

**Tier 2:** UI Automation
- Full element tree per window
- Element types, names, states
- Bounding rectangles

---

## Quick Start

### Install

```bash
# macOS
brew install oscp

# Linux
sudo apt install oscp

# Windows (V1)
winget install OSCP.Windows
```

### Connect

```python
import oscp

client = oscp.connect("unix:///tmp/oscp.sock")  # macOS/Linux
# or
client = oscp.connect("tcp://localhost:9876")    # Windows

async for tree in client.stream():
    for window in tree.windows:
        for op in window.ops:
            print(f"rect: {op.bounds}, z: {op.z}")

client.click(x=100, y=200)
```

---

## Coverage

| Platform | Coverage | Gaps |
|----------|----------|------|
| **macOS** | 95% | Screen sharing, DRM, sandboxed apps |
| **Linux** | 90-95% | Some Wayland compositors, TTY, KMS apps |
| **Windows** | 90% | Non-UIA apps, legacy Win32, protected content |

---

## Gaps After V1

| Gap | Severity | Fillable? |
|-----|----------|-----------|
| Element semantics (macOS/Linux) | Moderate | ✅ Agent skills |
| WebGL/Canvas content | Moderate | ⚠️ Partially |
| Color semantics | Minor | ✅ Protocol extension |
| Audio | Minor | ✅ Separate API later |
| Protected content | Minor | ❌ OS restriction |
| Physical hardware | Minor | ❌ Separate API later |

---

## Documentation

| Document | Description |
|----------|-------------|
| [SPEC.md](SPEC.md) | Project overview |
| [protocol/SPEC.md](protocol/SPEC.md) | Core protocol |
| [platforms/macos/SPEC.md](platforms/macos/SPEC.md) | macOS driver |
| [platforms/linux/SPEC.md](platforms/linux/SPEC.md) | Linux driver |
| [platforms/windows/SPEC.md](platforms/windows/SPEC.md) | Windows driver |
| [agents/SPEC.md](agents/SPEC.md) | Agent integration |

---

## Project Structure

```
OSCP/
├── README.md
├── SPEC.md
├── protocol/SPEC.md           # Core protocol
├── platforms/
│   ├── macos/SPEC.md         # Window Server (CGWindowList)
│   ├── linux/SPEC.md         # X11 + Compositor Hook
│   └── windows/SPEC.md       # UIA + Win32
├── agents/SPEC.md
└── MEMORY/
    ├── project-overview.md
    └── DECISIONS.md
```

---

## Status

🚧 **V1 Development** — macOS, Linux, Windows drivers
🚧 **V2 Planning** — Render ops for Windows (DWM hook)

---

## Core Principle

> Humans interact through interfaces.
> Agents interact through meaning.

OSCP provides the geometry. Agent provides the meaning. Execution remains deterministic.

---

*OSCP — First-class access for agents.*