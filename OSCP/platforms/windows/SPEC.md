# OSCP Windows Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The Windows platform driver provides OSCP protocol implementation for Windows 10/11. It intercepts the Desktop Window Manager (DWM) compositor at the OS level — not per-app — and delivers raw render operations to agents.

**Principle:** Intercept the compositor. Decode the render tree. Agent provides the meaning.

---

## Compositor Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     APPS                                        │
│                                                                 │
│   App A (DXGI) ─┐                                               │
│   App B (OpenGL) ─┼─ RENDER ─► TEXTURES ─►                      │
│   App C (Vulkan) ─┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  DWM (Desktop Window Manager)                    │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    SCENE MANAGER                        │   │
│   │                                                         │   │
│   │   Window 1 ──► Layer ──► Compositor                     │   │
│   │   Window 2 ──► Layer ──► Compositor                     │   │
│   │   Window 3 ──► Layer ──► Compositor                     │   │
│   │                                                         │   │
│   │            ┌──────────────────────┐                      │   │
│   │            │   RENDER TREE        │                      │   │
│   │            │   (intercept here)   │                      │   │
│   │            └──────────────────────┘                      │   │
│   │                                                         │   │
│   │            INTERCEPT ─► DECODE ─► AGENT                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│                      DISPLAY OUTPUT                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Intercept Approach

### Why Not Per-App Hook?

Per-app DXGI hook misses:
- OpenGL applications
- Vulkan applications
- Games with anti-cheat
- Some legacy GDI apps

**OSCP intercepts the DWM compositor instead — one point, all apps.**

### Intercept Methods

| Method | Description | Coverage |
|--------|-------------|----------|
| **Graphics Capture API** | Windows.Graphics.Capture | Win32 windows |
| **DWM Shared Surface** | DWM internal surfaces | All windows |
| **Mirror Driver** | Kernel-level mirror | Full desktop |
| **DXGI Output** | DXGI swapchain output | DirectX apps |

### Approach: DWM Scene Capture

DWM maintains a render tree of all visible windows. We intercept this at the compositor level:

```rust
// Conceptual approach
struct DWMInterceptor {
    // Hook into DWM's internal scene
    // Extract window layers and their textures
    // Decode render operations from each layer
    
    fn capture_scene() -> RenderTree {
        // Get all visible windows from DWM
        // For each window, extract its layers
        // Build render tree with positions and z-order
        // Return decoded render ops
    }
}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌───────────────┐           ┌───────────────┐                  │
│  │  DWM Scene     │           │  Input Engine │                  │
│  │  Interceptor   │           │  (Rust)       │                  │
│  │  (Rust)        │           │               │                  │
│  └───────┬───────┘           └───────┬───────┘                  │
│          │                           │                          │
│          └──────────┬────────────────┘                          │
│                     │                                          │
│              ┌──────▼──────┐                                   │
│              │  Render    │                                   │
│              │  Decoder   │                                   │
│              │  (Rust)    │                                   │
│              └──────┬──────┘                                   │
│                     │                                          │
│              ┌──────▼──────┐                                   │
│              │  Protocol   │                                   │
│              │  Server     │                                   │
│              │  (TCP)      │                                   │
│              └─────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. DWM Scene Interceptor (Rust)

**Purpose:** Intercept the DWM compositor's render tree.

**What it captures:**
- All visible windows and their metadata
- Window layers and textures
- Compositor z-order and transforms
- Render operations per window

**Output:**
```json
{
  "windows": [
    {
      "id": "win_0x12345",
      "title": "Visual Studio Code",
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
  ]
}
```

### 2. Render Decoder (Rust)

**Purpose:** Decode the DWM scene into render operations.

**Pipeline:**
```
1. Capture DWM scene (shared surfaces)
   ↓
2. Extract window list and metadata
   ↓
3. For each window, extract layers and textures
   ↓
4. Build render tree with positions, z-order
   ↓
5. Output render operations with bounds and texture IDs
```

### 3. Input Engine (Rust)

**Purpose:** Execute agent actions on Windows.

**Methods:**
- `SendInput`: Primary for keyboard and mouse events
- `PostMessage`: Targeted messages to specific windows

**Supported Actions:**
- Mouse: click, double-click, right-click, move, drag, scroll
- Keyboard: type, key press, key combo

---

## Coverage

| App Type | Works? | Notes |
|----------|--------|-------|
| DirectX apps (Chrome, VS Code) | ✅ Yes | Via DWM layer capture |
| OpenGL apps | ✅ Yes | Via DWM layer capture |
| Vulkan apps | ✅ Yes | Via DWM layer capture |
| GDI apps | ✅ Yes | Via DWM layer capture |
| Games | ✅ Yes | Captured at compositor |
| Anti-cheat games | ✅ Yes | Output captured, not hooked |
| DRM content | ❌ No | Protected at source |
| Remote Desktop | ❌ No | App runs on remote machine |

**Coverage: ~95% of desktop apps**

---

## Protocol Implementation

### Transport

Default: `tcp://localhost:9876`

### Message Framing

Length-prefixed JSON:
```
[4-byte big-endian length][JSON payload]
```

### Capabilities

```json
{
  "platform": "windows",
  "driver": "oscp-windows-v0.2",
  "compositor": "dwm",
  "capabilities": ["render_tree", "actions", "events"],
  "features": {
    "compositor_intercept": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Installation & Startup

### Installation

```powershell
# Single command install (V1 target)
winget install OSCP.Windows
```

### Startup

1. OSCP service starts (Windows service or auto-launch)
2. Agent connects via TCP
3. Driver sends `welcome` message with compositor info
4. Render tree streaming begins

---

## Configuration

```json
{
  "windows": {
    "capture": {
      "frame_rate": 30,
      "compositor": "dwm"
    },
    "input": {
      "method": "sendinput",
      "action_delay_ms": 0
    }
  }
}
```

---

## Limitations

| Limitation | Description |
|------------|-------------|
| DRM content | Netflix, etc. protected at source |
| Remote Desktop | No access to remote machine compositor |
| Protected apps | Some UWP apps may restrict capture |

---

## Status

🚧 **V1 Target:** DWM compositor interception + render tree + actions

---

## References

- [Desktop Window Manager](https://learn.microsoft.com/en-us/windows/win32/dwm/wm-dwm--desktop-window-manager-)
- [Graphics Capture API](https://learn.microsoft.com/en-us/windows/win32graphics/capture/graphics-capture-api)
- [DWM Overview](https://learn.microsoft.com/en-us/windows/win32/dwm/dwm-overview)