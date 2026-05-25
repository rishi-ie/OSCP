# OSCP Windows Platform Driver Specification

**Version:** 0.1.0
**Status:** Draft

---

## Overview

The Windows platform driver provides OSCP protocol implementation for Windows 10/11. It intercepts DirectX render operations and delivers raw geometry to agents.

**Principle:** Intercept at the source. Deliver the geometry. Agent provides the meaning.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌───────────────┐           ┌───────────────┐                  │
│  │  DXGI Hook    │           │  Input Engine │                  │
│  │  (C++ DLL)     │           │  (Rust)       │                  │
│  └───────┬───────┘           └───────┬───────┘                  │
│          │                           │                          │
│          └──────────┬────────────────┘                          │
│                     │                                          │
│              ┌──────▼──────┐                                   │
│              │  Decoder    │                                   │
│              │  Service     │                                   │
│              │  (Rust)      │                                   │
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

### 1. DXGI Hook (C++)

**Purpose:** Intercept DirectX/Direct3D render operations at the graphics API level.

**Injection Method:**
- DLL injected into target processes via `CreateRemoteThread` + `LoadLibrary`
- Hook targets: `IDXGISwapChain::Present`, `ID3D11DeviceContext::DrawIndexed`

**Shared Memory:**
- Ring buffer in shared memory (named `Global\\OSCP_SharedBuffer_{PID}`)
- Producer: DXGI hook (in target process)
- Consumer: Decoder service

**Data Captured:**
- Draw operations (texture_id, destination rect, z_index, HWND)
- Frame timing (timestamp, frame_number)
- Window context (HWND mapping)

**Render Operations Output:**
```json
{
  "id": "op_001",
  "bounds": {"x": 10, "y": 10, "w": 100, "h": 30},
  "z": 1,
  "texture_id": "0xAA01",
  "clip_bounds": {"x": 15, "y": 15, "w": 90, "h": 20}
}
```

### 2. Input Engine (Rust)

**Purpose:** Execute agent actions on Windows.

**Methods:**
- `SendInput`: Primary for keyboard and mouse events
- `PostMessage`: Targeted messages to specific windows

**Supported Actions:**
- Mouse: click, double-click, right-click, move, drag, scroll
- Keyboard: type, key press, key combo

---

## Capture Pipeline

### Frame Capture Flow

```
1. DXGI Hook captures Present() call
   ↓
2. Draw operations extracted (texture, rect, z)
   ↓
3. HWND determined from swapchain
   ↓
4. Frame written to ring buffer
   ↓
5. Decoder reads frame from shared memory
   ↓
6. Frame assembled with window context
   ↓
7. Raw frame sent to agent via protocol
```

### Rendering Interception Points

| Interface | Method | Purpose |
|-----------|--------|---------|
| `IDXGISwapChain` | `Present()` | Frame boundary detection |
| `ID3D11DeviceContext` | `DrawIndexed()` | Individual draw operations |
| `ID3D11DeviceContext` | `PSSetShaderResources()` | Texture binding |

---

## Window Management

### Surface Discovery

- Enumerate all windows via `EnumWindows`
- Track by `HWND` value
- Map `HWND` → surface ID
- Handle: hidden, minimized, maximized, restored

### Surface Data

```json
{
  "id": "surface_0x12345",
  "title": "Visual Studio Code",
  "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
  "position": {"x": 100, "y": 50},
  "focused": true
}
```

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
  "driver": "oscp-windows-v0.1",
  "capabilities": ["frames", "actions", "events"],
  "features": {
    "dxgi_hook": true,
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
3. Driver sends `welcome` message
4. Frame streaming begins

---

## Configuration

```json
{
  "windows": {
    "capture": {
      "frame_rate": 30
    },
    "input": {
      "method": "sendinput",
      "action_delay_ms": 0
    },
    "hooks": {
      "auto_hook": true,
      "target_processes": ["*"],
      "excluded_processes": ["explorer.exe"]
    }
  }
}
```

---

## Limitations

| Limitation | Description |
|------------|-------------|
| DXGI only | DirectX apps only. OpenGL/Vulkan excluded. |
| User-mode hook | Detectable by some applications. |
| Games | Some block hooks (anti-cheat). |

---

## Status

🚧 **V1 Target:** DXGI hook + raw geometry + basic actions

---

## References

- [DXGI Overview](https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/dxgi-landing)
- [SendInput](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput)