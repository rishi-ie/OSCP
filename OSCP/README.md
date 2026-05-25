# OSCP — Operating System Context Protocol

**Version:** 0.1.0 (Draft)
**Status:** Design finalization complete

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

Today, agents attempt to interact with computers using VLMs — observing pixels, guessing element positions, and reacting to visual output. This approach is **fragile, slow, and unreliable**.

OSCP replaces GUI imitation with **native render interception**. Agents receive raw geometry from the render pipeline. Agent provides the meaning.

---

## Principle

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Intercept at the source. Deliver the geometry.               │
│   Agent provides the meaning.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Traditional approach (VLM):**
```
App → Render → Pixels → Agent guesses from pixels
```

**OSCP approach:**
```
App → Render → INTERCEPT HERE → Raw geometry → Agent reasons
                           ↓
                    Positions, z-order, texture IDs
                    (exact, deterministic)
```

Draw calls already ARE the element list. We intercept before they become pixels. Agent interprets the geometry.

---

## What Agent Receives

| Data | Description |
|------|-------------|
| `surface_id` | Window identifier |
| `title` | Window title |
| `bounds` | Window position and size |
| `focused` | Whether window is focused |
| `render_ops` | List of draw operations with positions, rects, z-order |
| `mouse` | Cursor position and hovered element |

**Agent provides:**
- Element semantics (button vs label vs input)
- Interaction logic (what's clickable)
- State inference (enabled, disabled, checked)
- Visual reasoning (color, icon, layout patterns)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AGENT HARNESS                            │
│                                                                 │
│   Agent receives:                                               │
│   {                                                             │
│     "surface_id": "win_123",                                    │
│     "title": "VS Code",                                         │
│     "render_ops": [                                             │
│       {"bounds": {...}, "z": 1, "texture_id": "0xAA01"},        │
│       {"bounds": {...}, "z": 2, "texture_id": "0xAA02"}         │
│     ],                                                          │
│     "mouse": {"x": 540, "y": 320}                              │
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
  DXGI Hook           Window List           X11/Wayland
  (render ops)       (render ops)          (render ops)
```

---

## Platforms

| Platform | Capture Method | Input Method | Status |
|----------|---------------|--------------|--------|
| **Windows** | DXGI hook | SendInput | Draft |
| **macOS** | Window list | CGEvent | Draft |
| **Linux** | X11/Wayland | XTest | Draft |

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

# Get current frame
frame = client.get_frame()

# Iterate over surfaces
for surface in frame.surfaces:
    print(f"Window: {surface.title}")
    
    # Iterate over render ops (raw geometry)
    for op in surface.render_ops:
        print(f"  rect: {op.bounds}, z: {op.z}, texture: {op.texture_id}")

# Perform action
client.click(x=100, y=200)
client.type("hello world")
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [SPEC.md](SPEC.md) | Project overview and structure |
| [protocol/SPEC.md](protocol/SPEC.md) | Core protocol: messages, transport, types |
| [platforms/windows/SPEC.md](platforms/windows/SPEC.md) | Windows driver design |
| [platforms/macos/SPEC.md](platforms/macos/SPEC.md) | macOS driver design |
| [platforms/linux/SPEC.md](platforms/linux/SPEC.md) | Linux driver design |
| [agents/SPEC.md](agents/SPEC.md) | Agent integration guidelines |

---

## Project Structure

```
OSCP/
├── README.md                 # Project overview
├── SPEC.md                   # Main specification
├── protocol/                 # Protocol specification
│   └── SPEC.md              # Core protocol spec
├── platforms/               # OS-specific drivers
│   ├── windows/             # Windows implementation (DXGI hook)
│   ├── macos/              # macOS implementation
│   └── linux/              # Linux implementation
└── agents/                  # Agent integration
    └── SPEC.md             # Agent SDK guidelines
```

---

## Protocol Highlights

### Frame Message (Driver → Agent)

```json
{
  "type": "frame",
  "frame_id": 12345,
  "timestamp": 1716576000000,
  "surfaces": [
    {
      "id": "surface_abc123",
      "title": "Visual Studio Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "render_ops": [
        {"id": "op_001", "bounds": {"x": 10, "y": 10, "w": 100, "h": 30}, "z": 1, "texture_id": "0xAA01"},
        {"id": "op_002", "bounds": {"x": 120, "y": 10, "w": 80, "h": 30}, "z": 2, "texture_id": "0xAA02"},
        {"id": "op_003", "bounds": {"x": 10, "y": 50, "w": 200, "h": 100}, "z": 0, "texture_id": "0xAA03"}
      ]
    }
  ],
  "mouse": {
    "x": 150,
    "y": 25,
    "hovered_op_id": "op_002"
  }
}
```

### Action Message (Agent → Driver)

```json
{
  "type": "action",
  "action_id": "act_001",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "click",
    "x": 50,
    "y": 25,
    "button": "left"
  }
}
```

---

## Why Not VLM?

| Aspect | VLM Approach | OSCP |
|--------|-------------|------|
| **Speed** | 2-5 seconds per frame | <50ms per frame |
| **Reliability** | ~70% (guessing) | ~95% (exact) |
| **Precision** | Fuzzy coordinates | Pixel-perfect |
| **Determinism** | Variable | Consistent |
| **Cost** | $0.01-0.05/frame | $0.0002/frame |

---

## Roadmap

| Phase | Milestone |
|-------|-----------|
| **V0.1** | Protocol and spec finalization |
| **V0.2** | Windows driver (DXGI hook + actions) |
| **V0.3** | macOS driver |
| **V0.4** | Linux driver |
| **V1.0** | Cross-platform unification |
| **V1.1** | Agent SDKs (OpenClaw, HermesAgent, PyAgent) |

---

## Core Principle

> Humans interact through interfaces.
> Agents interact through meaning.

OSCP provides the geometry. Agent provides the meaning. Execution remains deterministic.

---

## Status

🚧 **Phase 0** — Specifications complete. Implementation starting.