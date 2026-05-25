# OSCP macOS Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The macOS platform driver provides OSCP protocol implementation for macOS 12+ (Monterey and later). It intercepts the Window Server compositor's layer tree at the OS level and delivers raw render operations to agents.

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
│   │            INTERCEPT ─► DECODE ─► AGENT                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│                      DISPLAY OUTPUT                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Intercept Approach

### How macOS Compositor Works

1. Every window has a `CALayer` tree
2. Window Server composites all layers into final output
3. Layers contain textures, transforms, and properties
4. Intercepting the layer tree gives us all render operations

### Intercept Methods

| Method | Description | Provides |
|--------|-------------|----------|
| `CGWindowListCopyWindowInfo` | Window enumeration | Metadata, bounds, z-order |
| `CGWindowListCreateImage` | ❌ Gives pixels | NOT USED |
| `CGWindowID` | Window identifier | Stable window ID |
| Window Server layer tree | Layer properties | Positions, sizes, hierarchy |

**No pixels. No screenshots. Just layer properties and geometry.**

### Layer Properties We Extract

For each window's layer tree:

```swift
struct LayerInfo {
    id: String              // Layer identifier
    bounds: CGRect          // Position and size
    z_index: Int            // Render order
    content: LayerContent   // What the layer contains
    children: [LayerInfo]   // Child layers
}

enum LayerContent {
    image(texture_id: String, size: CGSize)
    text(content: String, font: String?, size: CGFloat?)
    solid(color: String)
    composited(children: [LayerInfo])
}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌───────────────────┐          ┌───────────────────┐          │
│  │  Window Server    │          │  Input Engine     │          │
│  │  Layer Interceptor│          │  (Swift/Rust)     │          │
│  │  (Swift)          │          │                   │          │
│  └─────────┬─────────┘          └─────────┬─────────┘          │
│            │                               │                     │
│            └───────────────┬───────────────┘                     │
│                            │                                     │
│                    ┌───────▼───────┐                             │
│                    │   Render      │                             │
│                    │   Decoder     │                             │
│                    │   (Rust)      │                             │
│                    └───────┬───────┘                             │
│                            │                                     │
│                    ┌───────▼───────┐                             │
│                    │   Protocol    │                             │
│                    │   Server      │                             │
│                    │   (Unix)      │                             │
│                    └───────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Window Server Layer Interceptor (Swift)

**Purpose:** Extract layer tree from Window Server.

**What it captures:**
- All visible windows via `CGWindowListCopyWindowInfo`
- Layer hierarchy for each window
- Layer properties: bounds, z-order, content type

**Implementation:**
```swift
func captureScene() -> [WindowLayers] {
    // Get all on-screen windows
    let windowList = CGWindowListCopyWindowInfo([.optionOnScreenOnly, .excludeDesktopElements], kCGNullWindowID)!
    
    for window in windowList as! [CFDictionary] {
        let windowID = window[kCGWindowNumber]!
        let bounds = window[kCGWindowBounds] as! CGRect
        let layer = window[kCGWindowLayer]!
        
        // Extract layer properties
        extractLayerTree(for: windowID)
    }
}
```

### 2. Render Decoder (Rust)

**Purpose:** Convert layer tree to render operations.

**Pipeline:**
```
1. Capture Window Server layer tree
   ↓
2. Extract window metadata
   ↓
3. For each window, traverse layer hierarchy
   ↓
4. Flatten layers into render operations
   ↓
5. Output ops with bounds, z, texture_id
```

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
          "texture_id": "layer_0x1001"
        }
      ]
    }
  ]
}
```

### 3. Input Engine (Swift/Rust)

**Purpose:** Execute agent actions on macOS.

**Methods:**
- `CGEvent`: Mouse and keyboard

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
| Screen Sharing | ❌ No | Different compositor |

**Coverage: ~95% of desktop apps**

---

## Permission Model

### Required Permissions

**"Screen Recording" (misleading name)** — Actually enables:
- Window Server layer tree access
- CGWindowListCopyWindowInfo with full info
- CALayer property extraction

**Accessibility** — For input injection.

### Permission Flow

```
1. Driver starts
2. Check CGPreflightScreenCaptureAccess()
3. If false → notify user, open System Preferences
4. User enables OSCP in Privacy & Security
5. Driver proceeds
```

---

## Protocol Implementation

### Transport

Default: `unix:///tmp/oscp.sock`

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
    "layer_tree": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Installation

```bash
brew install oscp
```

### Startup

1. User grants permission (one-time)
2. Driver starts as launchd agent
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
    }
  }
}
```

---

## Status

🚧 **V1 Target:** Window Server layer tree extraction + render ops + actions

---

## References

- [CGWindowListCopyWindowInfo](https://developer.apple.com/documentation/application-services/1503048-cgwindowlistcopywindowinfo)
- [Core Animation Layer Tree](https://developer.apple.com/documentation/quartzcore/calayer)
- [CGEvent](https://developer.apple.com/documentation/coregraphics/cgevent)