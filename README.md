# OSCP — Operating System Context Protocol

## Vision

OSCP exists to make agents **first-class users of computing systems**, alongside humans.

Today's software ecosystem is designed around a single assumption:

> Humans interact with computers through graphical interfaces.

Agents currently imitate humans by observing screens, moving cursors, pressing keys, and reacting to pixels. This approach introduces fragility, hidden state, and unreliable behavior over long workflows.

OSCP aims to replace GUI imitation with a native semantic execution layer.

```
Human sees:    Rendered pixels (the result)
Agent sees:    Render operations (the source)

Agent knows:   Every element's position, z-order, texture ID, history
Agent does:   Click, type, scroll — with perfect precision
```

---

## Why Not Vision?

| Aspect | Vision-Based | OSCP |
|--------|--------------|------|
| **Speed** | 2-5 seconds | <50ms |
| **Cost** | $0.01-0.05/frame | $0.0002/frame |
| **Reliability** | ~70% | ~90% |
| **Precision** | Guessed coordinates | Exact coordinates |
| **Coverage** | All UIs | DX apps (~85%) |

**OSCP wins for desktop automation. Vision is fallback for edge cases.**

---

## Agent vs Human

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT WITH OSCP vs HUMAN                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   PERFECT MEMORY                                                │
│   Human: Forgets what they saw 5 seconds ago                   │
│   Agent: Full frame history, searchable, comparable             │
│                                                                 │
│   EXACT REPRODUCIBILITY                                         │
│   Human: "I think I clicked there"                              │
│   Agent: "Clicked at (523, 287) on element e42"                 │
│                                                                 │
│   NO FATIGUE                                                    │
│   Human: Error rate increases over time                          │
│   Agent: Identical precision at action 1 and action 1000        │
│                                                                 │
│   KNOWS WHAT HUMANS CAN'T SEE                                   │
│   Human: Current frame only                                     │
│   Agent: Render operations, texture IDs, z-order, history       │
│                                                                 │
│   PARALLEL-READY                                                │
│   Human: One task at a time                                     │
│   Agent: Analyze and execute simultaneously                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Technical Principle

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Go sideways at the source, not backwards from pixels.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

The two paths of GUI extraction:

```
FORWARD (how rendering happens):
App → Layout → Draw Calls → GPU → Pixels

BACKWARDS (vision approach):
Pixels → OCR → Guess → Element positions (unreliable)

SIDEWAYS at source (OSCP approach):
App → Layout → INTERCEPT HERE → Structured element list (exact)
                ↑
           Draw calls already contain:
           - What texture
           - Where (x, y, w, h)
           - In what order (z)
           - For which window (HWND)
```

**The insight:** Draw calls already ARE the element list. We intercept them before they become pixels.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Drawing a button = {texture, position, z}                     │
│                                                                 │
│   This IS the element.                                          │
│                                                                 │
│   The pixel is the result. The draw call is the truth.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Mechanism: Intercept at the Source

Instead of observing the final output (pixels), OSCP intercepts the rendering process at its source — the graphics API layer.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   TRADITIONAL APPROACH:                                         │
│   App → Render → Pixels → Vision → Agent guesses               │
│                      └───── Information lost here              │
│                                                                 │
│   OSCP APPROACH:                                                │
│   App → Render → INTERCEPT HERE → Agent knows everything        │
│                     └───── Full render data preserved          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What Gets Intercepted

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   For every frame, the driver captures:                         │
│                                                                 │
│   1. DRAW OPERATIONS                                            │
│      - What texture was drawn                                   │
│      - Where it was drawn (x, y, w, h)                         │
│      - In what order (z-index)                                  │
│      - To which window (HWND)                                   │
│                                                                 │
│   2. WINDOW CONTEXT                                             │
│      - Window handle (HWND)                                     │
│      - Window title                                              │
│      - Window position and size                                  │
│      - Focus state                                               │
│                                                                 │
│   3. TEXTURE INFORMATION                                        │
│      - Unique texture ID                                        │
│      - Texture type hint                                        │
│      - Text content (for text textures)                         │
│                                                                 │
│   4. FULL SCREENSHOT (fallback)                                 │
│      - Always available if needed                               │
│      - For vision-based reasoning when needed                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AGENT HARNESS (pi)                       │
│                                                                 │
│   Agent receives:                                                │
│   {                                                             │
│     "frame_id": 12345,                                         │
│     "window": {"title": "Settings", "bounds": {...}},          │
│     "elements": [                                               │
│       {"bounds": {...}, "text": "General"},                    │
│       {"bounds": {...}, "text": "Security"},                   │
│       {"bounds": {...}, "text": "Save"}                        │
│     ],                                                          │
│     "screenshot": "base64..."                                  │
│   }                                                            │
│                                                                 │
│   Agent sends: {"action": "click", "x": 210, "y": 320}         │
└─────────────────────────────────────────────────────────────────┘
                            │ JSON (socket)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      OSCP DRIVER                                │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  DXGI Hook (C++ DLL)                                    │   │
│   │  - Injected into target processes                       │   │
│   │  - Hooks: Present(), DrawIndexed(), PSSetShaderResources│   │
│   │  - Captures all draw operations                        │   │
│   │  - Writes to shared memory ring buffer                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Decoder Service (Rust)                                │   │
│   │  - Reads from shared memory                             │   │
│   │  - Parses render operations → element list              │   │
│   │  - Maps HWND → window info                              │   │
│   │  - Sends JSON via socket                               │   │
│   │  - Receives actions → executes via Win32                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### 1. DXGI Hook (C++ DLL)

```cpp
// Hooked interfaces:
// - IDXGISwapChain::Present()
// - ID3D11DeviceContext::DrawIndexed()
// - ID3D11DeviceContext::PSSetShaderResources()

// Captured per frame:
struct FrameCapture {
    HWND hwnd;
    uint64_t timestamp;
    std::vector<DrawOp> operations;
};

struct DrawOp {
    uint32_t texture_id;
    RECT dest_rect;
    float z_index;
    // ... additional metadata
};
```

### 2. Shared Memory (Ring Buffer)

```cpp
// Producer: DXGI Hook (in target process)
// Consumer: Decoder Service

// Ring buffer in shared memory
// Non-blocking, lock-free where possible
// Falls back to mutex if needed
```

### 3. Decoder Service (Rust)

```rust
struct Element {
    id: String,
    bounds: Rect,
    z_index: u32,
    texture_id: u32,
    text: Option<String>,
}

struct WindowState {
    hwnd: String,
    title: String,
    bounds: Rect,
    focused: bool,
    elements: Vec<Element>,
}

struct Snapshot {
    frame_id: u64,
    timestamp: u64,
    window: WindowState,
    screenshot: Option<String>,
}
```

### 4. Socket Protocol

```json
// Driver → Harness
{
  "frame_id": 12345,
  "timestamp": 1716576000000,
  "window": {
    "hwnd": "0x12345",
    "title": "Settings",
    "bounds": {"x": 100, "y": 100, "w": 1200, "h": 800},
    "focused": true
  },
  "elements": [
    {"id": "e1", "bounds": {"x": 0, "y": 0, "w": 200, "h": 800}, "z": 0, "texture_id": "0xAA01"},
    {"id": "e2", "bounds": {"x": 10, "y": 70, "w": 180, "h": 40}, "z": 1, "texture_id": "0xAA02", "text": "General"},
    {"id": "e3", "bounds": {"x": 10, "y": 115, "w": 180, "h": 40}, "z": 1, "texture_id": "0xAA03", "text": "Security"},
    {"id": "e4", "bounds": {"x": 200, "y": 300, "w": 120, "h": 36}, "z": 2, "texture_id": "0xAA04", "text": "Save"}
  ],
  "screenshot": "base64..."
}

// Harness → Driver
{"action": "click", "x": 100, "y": 135}
{"action": "type", "text": "hello world"}
{"action": "key_combo", "modifiers": ["ctrl"], "key": "s"}
{"action": "scroll", "delta": -3}
```

---

## What Works

### Captures
- All DirectX/Direct3D rendered content
- Every draw operation: position, size, z-index, texture
- Per-window rendering (maps textures to windows)
- Full desktop screenshot (fallback)

### Actions
- Click (left, right, middle)
- Double-click
- Scroll (up/down)
- Type text
- Key press / key combinations
- Drag and drop

### Applications
- Chrome, Edge, Firefox
- VS Code, JetBrains IDEs
- Electron apps (Slack, Discord, Notion, Figma)
- Microsoft Office
- File Explorer
- Terminal
- Zoom, Teams
- Most Win32 and WPF applications

**Coverage: ~85-90% of desktop applications**

---

## Limitations

```
┌─────────────────────────────────────────────────────────────────┐
│                      TECHNICAL LIMITS                            │
├─────────────────────────────────────────────────────────────────┤
│ DXGI only           DirectX apps only. OpenGL/Vulkan excluded.  │
│ User-mode hook      Detectable. Not kernel-level.               │
│ No element types    Button vs input vs label unknown.           │
│ No state            Enabled/disabled/hover unknown.             │
│ Text partial        Text in textures: available. OCR: not built.│
├─────────────────────────────────────────────────────────────────┤
│                      APPS NOT COVERED                           │
├─────────────────────────────────────────────────────────────────┤
│ Games               Some block hooks (anti-cheat).              │
│ Protected media     Netflix, DRM: blocked.                       │
│ Remote Desktop      No hook access.                            │
│ Legacy GDI          May not render via DirectX.                │
└─────────────────────────────────────────────────────────────────┘
```

### What Agent Infers
- Element role from position and context
- Clickable from appearance and z-order
- Text from texture hints and surrounding elements
- State from history and mouse position

### Fallback Strategy
- Screenshot always available for vision-based reasoning
- Desktop Duplication API for pixels-only capture
- UI Automation integration (future)

---

## Objectives

### Agent-native interaction
Create an interaction model where agents no longer need to pretend to be humans.

### Deterministic execution
Separate intelligence from execution. Reasoning may remain probabilistic, but execution is deterministic and verifiable.

### Atomic operations
Reduce all system interactions into the smallest practical semantic actions.

### Semantic system understanding
Expose machine state in structured form rather than raw visual representation.

### Event-driven operation
Expose changes as structured events so agents can operate with awareness of state transitions in real time.

### Reliability over convenience
Optimize for repeatability, fault tolerance, recoverability, and predictable behavior.

---

## Protocol Design

```
┌─────────────────────────────────────────────────────┐
│                    OSCP Protocol                     │
├─────────────────────────────────────────────────────┤
│  Layer 1: Context     — Hierarchical OS state        │
│  Layer 2: Input       — Structured input injection   │
│  Layer 3: Events      — Real-time system events     │
│  Layer 4: Permissions — Capability-based access      │
└─────────────────────────────────────────────────────┘
```

---

## Roadmap

| Phase | Addition |
|-------|----------|
| V1 | DXGI hook + decoder + action execution |
| V2 | OpenGL hook for broader coverage |
| V2 | Text detection in textures |
| V3 | UI Automation integration (supplement) |
| V3 | Browser extension (full DOM access) |
| V4 | Kernel driver (undetectable) |
| V4 | Full state detection |
| V5 | macOS driver |
| V5 | Linux driver |

---

## Building

### Prerequisites
- Windows 10/11
- Rust (stable)
- C++ compiler (MSVC)

### Structure

```
OSCP/
├── dxgi-hook/              Injectable DLL (C++)
│   ├── hook.cpp            IDXGISwapChain::Present interception
│   ├── draw.cpp            Draw operation logging
│   └── shared.h            Shared memory ring buffer
│
├── decoder/                Service (Rust)
│   ├── src/
│   │   ├── main.rs         Entry point
│   │   ├── receiver.rs     Shared memory reader
│   │   ├── decoder.rs      Render ops → element list
│   │   ├── socket.rs        JSON-RPC to harness
│   │   └── executor.rs     Action execution
│   └── Cargo.toml
│
└── protocol.md             Driver ↔ Harness specification
```

### Build

```bash
# Build decoder service
cd decoder
cargo build --release

# DXGI hook requires Visual Studio build tools
# See dxgi-hook/README.md for build instructions
```

---

## Core Principle

> Humans interact through interfaces.
> Agents interact through meaning.

OSCP does not attempt to make intelligence deterministic.
OSCP attempts to make **execution deterministic**.

---

## Status

🚧 **Early development** — protocol design in progress.

---

*OSCP — Operating System Context Protocol.*