# OSCP macOS Platform Driver Specification

**Version:** 0.1.0
**Status:** Draft

---

## Overview

The macOS platform driver provides OSCP protocol implementation for macOS 12+ (Monterey and later). It captures window content at the graphics level and delivers raw geometry to agents.

**Principle:** Intercept at the source. Deliver the geometry. Agent provides the meaning.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌───────────────┐           ┌───────────────┐                  │
│  │  Window List  │           │  Input Engine │                  │
│  │  Monitor      │           │  (Swift)       │                  │
│  │  (Swift)      │           │               │                  │
│  └───────┬───────┘           └───────┬───────┘                  │
│          │                           │                          │
│          └──────────┬────────────────┘                          │
│                     │                                          │
│              ┌──────▼──────┐                                   │
│              │  Frame      │                                   │
│              │  Compositor │                                   │
│              │  (Rust)     │                                   │
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

## Capture Approach

### Window Capture

macOS provides APIs for window content access:

| API | Description |
|-----|-------------|
| `CGWindowListCreateImage` | Capture individual windows |
| `CGWindowListCopyWindowInfo` | Enumerate windows with metadata |

### Render Operation Extraction

For each window:
1. Enumerate window via `CGWindowListCopyWindowInfo`
2. Capture window image via `CGWindowListCreateImage`
3. Extract render operations from window content
4. Map operations to surface geometry

**Data Captured:**
- Window bounds and position
- Window title and owner
- Render operations with texture and position

---

## Components

### 1. Window List Monitor (Swift)

**Purpose:** Track all windows and their properties.

```swift
CGWindowListCopyWindowInfo([.optionOnScreenOnly, .excludeDesktopElements], kCGNullWindowID)
// Returns: windowID, ownerName, windowName, bounds, layer
```

**Tracking:**
- Window created → add to surface list
- Window destroyed → remove from surface list
- Window moved/resized → update bounds
- Window title changed → update title

### 2. Render Operations

Window content is analyzed to extract render operations:

```json
{
  "id": "op_001",
  "bounds": {"x": 10, "y": 10, "w": 100, "h": 30},
  "z": 1,
  "texture_id": "0xAA01"
}
```

- `bounds`: Position and size within window
- `z`: Render order (higher = on top)
- `texture_id`: Content identifier

### 3. Input Engine (Swift)

**Purpose:** Execute agent actions on macOS.

**Methods:**
- `CGEvent`: Mouse and keyboard (global)

**Supported Actions:**
- Mouse: click, double-click, move, drag, scroll
- Keyboard: type, key press, key combo

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
  "driver": "oscp-macos-v0.1",
  "capabilities": ["frames", "actions", "events"],
  "features": {
    "window_monitor": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Permission Model

### Required Permissions

For OSCP to work on macOS:

1. **Screen Recording** — Required for `CGWindowListCreateImage`
   - System Preferences → Privacy & Security → Screen Recording

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
4. Frame streaming begins

---

## Configuration

```json
{
  "macos": {
    "capture": {
      "frame_rate": 30
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
| Screen Recording | Requires user permission |
| Accessibility | Requires user permission |
| Some apps | May block capture (DRM, anti-cheat) |

---

## Status

🚧 **V1 Target:** Window list capture + raw geometry + basic actions

---

## References

- [CGWindowListCreateImage](https://developer.apple.com/documentation/application-services/1503048-cgwindowlistcreateimage)
- [CGEvent](https://developer.apple.com/documentation/coregraphics/cgevent)