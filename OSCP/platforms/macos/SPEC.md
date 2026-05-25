# OSCP macOS Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The macOS platform driver provides OSCP protocol implementation for macOS 12+ (Monterey and later). It intercepts the Window Server compositor at the OS level and delivers raw render operations to agents.

**Principle:** Intercept the compositor. Decode the render tree. Agent provides the meaning.

---

## Compositor Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     APPS                                        │
│                                                                 │
│   App A (AppKit) ─┐                                            │
│   App B (Metal)   ─┼─ RENDER ─► LAYERS ─►                       │
│   App C (Quartz)  ─┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  WINDOW SERVER (Quartz Compositor)               │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    LAYER TREE                          │   │
│   │                                                         │   │
│   │   Window 1 ──► CALayer ──► Compositor                   │   │
│   │   Window 2 ──► CALayer ──► Compositor                   │   │
│   │   Window 3 ──► CALayer ──► Compositor                   │   │
│   │                                                         │   │
│   │            ┌──────────────────────┐                      │   │
│   │            │   LAYER TREE         │                      │   │
│   │            │   (intercept here)   │                      │   │
│   │            └──────────────────────┘                      │   │
│   │                                                         │   │
│   │            INTERCEPT ─► DECODE ─► AGENT                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│                      DISPLAY OUTPUT                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Intercept Approach

### Why Not Per-App Hook?

Per-app Metal/OpenGL hook misses:
- AppKit apps (uses Quartz/Core Animation)
- Games with Metal
- Some legacy Carbon apps

**OSCP intercepts the Window Server layer tree instead — one point, all apps.**

### How macOS Compositor Works

1. Every window has a `CALayer` tree
2. Window Server composites all layers into final output
3. Layers contain textures, transforms, and properties
4. Intercepting the layer tree gives us all render operations

### Intercept Methods

| Method | Description | Coverage |
|--------|-------------|----------|
| **CGWindowListCreateImage** | ❌ Gives pixels, not ops | - |
| **CALayer tree inspection** | Extract layer properties | Window metadata |
| **IOSurface interception** | Intercept shared surfaces | Texture IDs |
| **Window Server plugin** | Hook into compositor | Full scene |

### Approach: Window Server Layer Tree

The Window Server maintains a layer tree for all windows. We intercept this at the compositor level:

```swift
// Conceptual approach
class WindowServerInterceptor {
    // Hook into Window Server's layer tree
    // Extract CALayer properties from each window
    // Decode render operations from layer hierarchy
    
    func captureScene() -> RenderTree {
        // Get all windows from Window Server
        // For each window, extract CALayer tree
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
│  │  Window       │           │  Input Engine │                  │
│  │  Server       │           │  (Swift)      │                  │
│  │  Interceptor  │           │               │                  │
│  │  (Swift)      │           │               │                  │
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
│              │  (Unix/TCP) │                                   │
│              └─────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Window Server Interceptor (Swift)

**Purpose:** Intercept the Window Server compositor's layer tree.

**What it captures:**
- All visible windows and their metadata
- CALayer hierarchy per window
- Layer properties (position, size, z-order, texture)
- Compositor transforms and effects

**Output:**
```json
{
  "windows": [
    {
      "id": "win_12345",
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

**Purpose:** Decode the layer tree into render operations.

**Pipeline:**
```
1. Capture Window Server layer tree
   ↓
2. Extract window list and metadata
   ↓
3. For each window, traverse CALayer hierarchy
   ↓
4. Extract layer positions, sizes, z-order, textures
   ↓
5. Build render tree
   ↓
6. Output render operations
```

### 3. Input Engine (Swift)

**Purpose:** Execute agent actions on macOS.

**Methods:**
- `CGEvent`: Mouse and keyboard (global)

**Supported Actions:**
- Mouse: click, double-click, move, drag, scroll
- Keyboard: type, key press, key combo

---

## Coverage

| App Type | Works? | Notes |
|----------|--------|-------|
| AppKit apps (Finder, Safari) | ✅ Yes | Via Window Server |
| Metal apps (games) | ✅ Yes | Via Window Server |
| AppKit + Metal (modern) | ✅ Yes | Via Window Server |
| Legacy Carbon apps | ⚠️ Limited | Partial support |
| DRM content | ❌ No | Protected at source |
| Screen Sharing | ❌ No | Different compositor |

**Coverage: ~95% of desktop apps**

---

## Permission Model

### Required Permissions

For OSCP to work on macOS:

1. **Screen Recording** — Required for Window Server access
   - System Preferences → Privacy & Security → Screen Recording
   - Needed for layer tree interception

2. **Accessibility** — Required for input injection
   - System Preferences → Privacy & Security → Accessibility

### Permission Check

```swift
// Check screen recording permission
let hasScreenRecording = CGPreflightScreenCaptureAccess()

// Request screen recording permission
CGRequestScreenCaptureAccess()

// Check accessibility permission
let hasAccessibility = AXIsProcessTrusted()
```

---

## Protocol Implementation

### Transport

Default: `unix:///tmp/oscp.sock`
Alternative: `tcp://localhost:9876`

### Message Framing

Length-prefixed JSON:
```
[4-byte big-endian length][JSON payload]
```

### Capabilities

```json
{
  "platform": "macos",
  "driver": "oscp-macos-v0.2",
  "compositor": "window_server",
  "capabilities": ["render_tree", "actions", "events"],
  "features": {
    "compositor_intercept": true,
    "layer_tree": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Installation & Startup

### Installation

```bash
# Single command install (V1 target)
brew install oscp
```

### Startup

1. User grants Screen Recording and Accessibility permissions
2. OSCP driver starts as background process
3. Agent connects via Unix socket
4. Render tree streaming begins

---

## Configuration

```json
{
  "macos": {
    "capture": {
      "frame_rate": 30,
      "compositor": "window_server"
    },
    "input": {
      "method": "cgevent",
      "action_delay_ms": 0
    },
    "permissions": {
      "auto_request": true,
      "require_all": true
    }
  }
}
```

---

## Limitations

| Limitation | Description |
|------------|-------------|
| Screen Recording permission | Required for layer tree access |
| Accessibility permission | Required for input injection |
| DRM content | Protected at source |
| Screen Sharing | Different compositor |

---

## Status

🚧 **V1 Target:** Window Server layer tree interception + render tree + actions

---

## References

- [Core Animation Layer Tree](https://developer.apple.com/documentation/quartzcore)
- [CGWindowListCreateImage](https://developer.apple.com/documentation/application-services/1503048-cgwindowlistcreateimage)
- [CGEvent](https://developer.apple.com/documentation/coregraphics/cgevent)