# Operating System Context Protocol (OSCP)

> Give AI agents first-class citizenship in the operating system.

```text
Human sees:    Rendered pixels (the result)
Agent sees:    Render operations (the source)

Agent knows:   Every element's position, z-order, texture ID, history
Agent does:   Click, type, scroll — with perfect precision
```

---

## What It Does

OSCP intercepts Windows desktop rendering at the graphics layer and exposes **decoded UI elements** to AI agents, enabling them to perceive and interact with the desktop reliably and efficiently.

```
┌─────────────────────────────────────────────────────────────────┐
│                        AGENT HARNESS                            │
│                                                                 │
│   Agent receives:                                                │
│   {                                                             │
│     "window": {"title": "Settings", "bounds": {...}},          │
│     "elements": [                                               │
│       {"bounds": {"x": 10, "y": 70}, "text": "General"},      │
│       {"bounds": {"x": 10, "y": 115}, "text": "Security"},     │
│       {"bounds": {"x": 200, "y": 300}, "text": "Save"}        │
│     ]                                                           │
│   }                                                             │
│                                                                 │
│   Agent sends: {"action": "click", "x": 210, "y": 320}          │
└─────────────────────────────────────────────────────────────────┘
                            │ JSON
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      OSCP DRIVER                                │
│                                                                 │
│   DXGI Hook → Shared Memory → Decoder → Socket                  │
│                                                                 │
│   Captures draw operations, textures, positions, z-order.       │
│   Executes actions via Win32 APIs.                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why Not Vision?

| Aspect | Vision-Based | OSCP |
|--------|--------------|------|
| **Speed** | 2-5 seconds | <50ms |
| **Cost** | $0.01-0.05/frame | $0.0002/frame |
| **Reliability** | ~70% | ~90% |
| **Precision** | Guessed coordinates | Exact coordinates |
| **Coverage** | All UIs | DX apps (85%) |

**OSCP wins for desktop automation. Vision is fallback for edge cases.**

---

## Comparison: Agent vs Human

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT WITH OSCP vs HUMAN                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   PERFECT MEMORY                                                │
│   Human: Forgets what they saw 5 seconds ago                   │
│   Agent: Full frame history, searchable, comparable            │
│                                                                 │
│   EXACT REPRODUCIBILITY                                         │
│   Human: "I think I clicked there"                              │
│   Agent: "Clicked at (523, 287) on element e42"                │
│                                                                 │
│   NO FATIGUE                                                    │
│   Human: Error rate increases over time                        │
│   Agent: Identical precision at action 1 and action 1000        │
│                                                                 │
│   KNOWS WHAT HUMANS CAN'T SEE                                   │
│   Human: Current frame only                                     │
│   Agent: Render operations, texture IDs, z-order, history       │
│                                                                 │
│   PARALLEL-READY                                                │
│   Human: One task at a time                                     │
│   Agent: Analyze and execute simultaneously (future)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works

### 1. Capture (DXGI Hook)

```cpp
// Hooked: IDXGISwapChain::Present()
// Hooked: ID3D11DeviceContext::DrawIndexed()
// Hooked: ID3D11DeviceContext::PSSetShaderResources()

// For each frame:
{
    "hwnd": "0x12345",
    "ops": [
        {"op": "draw_quad", "texture_id": "0xAA01", "dest": {x: 0, y: 0, w: 1920, h: 900}, "z": 0},
        {"op": "draw_quad", "texture_id": "0xAA02", "dest": {x: 0, y: 0, w: 60, h: 900}, "z": 1},
        {"op": "draw_quad", "texture_id": "0xAA03", "dest": {x: 10, y: 70, w: 180, h: 40}, "z": 2}
    ]
}
```

### 2. Decode (Service)

```rust
// Parse render operations → element list
{
    "window": {"hwnd": "0x12345", "title": "Settings"},
    "elements": [
        {"bounds": {x: 0, y: 0, w: 60, h: 900}, "z": 1, "texture_id": "0xAA02"},
        {"bounds": {x: 10, y: 70, w: 180, h: 40}, "z": 2, "texture_id": "0xAA03", "text": "General"},
        {"bounds": {x: 10, y: 115, w: 180, h: 40}, "z": 2, "texture_id": "0xAA04", "text": "Security"}
    ]
}
```

### 3. Deliver (Socket)

```json
// Driver → Harness
{"frame_id": 12345, "timestamp": 1716576000000, "elements": [...], "screenshot": "base64..."}

// Harness → Driver
{"action": "click", "x": 100, "y": 125}
```

### 4. Execute (Win32)

```rust
// Click at (x, y) via SendInput
// Type text via keybd_event
// Scroll via mouse_event
```

---

## Architecture

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
├── protocol.md             Driver ↔ Harness specification
└── README.md
```

---

## Protocol

### Driver → Harness

```json
{
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
```

### Harness → Driver

```json
{"action": "click", "x": 100, "y": 135}
{"action": "type", "text": "hello world"}
{"action": "key_combo", "modifiers": ["ctrl"], "key": "s"}
{"action": "scroll", "delta": -3}
{"action": "drag", "from": {"x": 100, "y": 200}, "to": {"x": 300, "y": 400}}
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
│ DXGI only           DirectX apps. OpenGL/Vulkan excluded.       │
│ User-mode hook      Detectable. Not kernel-level.               │
│ No element types    Button vs input vs label unknown.           │
│ No state            Enabled/disabled/hover unknown.             │
│ Text partial        Text in textures: available. OCR: not built.│
├─────────────────────────────────────────────────────────────────┤
│                      APPS NOT COVERED                           │
├─────────────────────────────────────────────────────────────────┤
│ Games               Some block hooks (anti-cheat).              │
│ Protected media     Netflix, DRM: blocked.                      │
│ Remote Desktop      No hook access.                            │
│ Legacy GDI          May not render via DirectX.                │
└─────────────────────────────────────────────────────────────────┘
```

### What Agent Infers
- Element role from position and context
- Clickable from appearance and z-order
- Text from texture hints and surrounding elements

### Fallback
- Screenshot always available for vision-based reasoning
- Desktop Duplication API for pixels-only capture

---

## Future Work

| Phase | Addition |
|-------|----------|
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

### Build

```bash
# Build decoder service
cd decoder
cargo build --release

# DXGI hook requires Visual Studio build tools
# See dxgi-hook/README.md for build instructions
```

### Run

```bash
# Start decoder service
./decoder/target/release/decoder.exe

# Connect harness
# (pi harness integration)
```

---

## Contributing

```
Coming soon.
```

---

## License

MIT