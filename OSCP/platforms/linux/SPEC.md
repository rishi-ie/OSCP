# OSCP Linux Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The Linux platform driver provides OSCP protocol implementation for Linux distributions. It intercepts the X11/Wayland compositor at the OS level and delivers raw render operations to agents.

**Principle:** Intercept the compositor. Decode the render tree. Agent provides the meaning.

---

## Compositor Architecture

### X11

```
┌─────────────────────────────────────────────────────────────────┐
│                     APPS                                        │
│                                                                 │
│   App A (GTK) ─────┐                                            │
│   App B (Qt)      ─┼─ RENDER ─► X11 REQUESTS ─►                 │
│   App C (SDL)     ─┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      X11 SERVER                                 │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    WINDOW TREE                          │   │
│   │                                                         │   │
│   │   Window 1 ──► Properties ──► Compositor                 │   │
│   │   Window 2 ──► Properties ──► Compositor                 │   │
│   │   Window 3 ──► Properties ──► Compositor                 │   │
│   │                                                         │   │
│   │            ┌──────────────────────┐                      │   │
│   │            │   WINDOW TREE        │                      │   │
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

### Wayland

```
┌─────────────────────────────────────────────────────────────────┐
│                     APPS                                        │
│                                                                 │
│   App A (GTK4) ────┐                                           │
│   App B (Qt)      ─┼─ RENDER ─► WAYLAND PROTOCOL ─►            │
│   App C (SDL)     ─┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   WAYLAND COMPOSITOR                           │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    SURFACE TREE                         │   │
│   │                                                         │   │
│   │   Surface 1 ──► wl_surface ──► Compositor               │   │
│   │   Surface 2 ──► wl_surface ──► Compositor               │   │
│   │   Surface 3 ──► wl_surface ──► Compositor               │   │
│   │                                                         │   │
│   │            ┌──────────────────────┐                      │   │
│   │            │   SURFACE TREE       │                      │   │
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

Per-app OpenGL/Vulkan hook misses:
- GTK/Qt apps (X11/Wayland rendering)
- Games with different renderers
- Legacy X11 apps

**OSCP intercepts the X11/Wayland compositor instead — one point, all apps.**

### X11 Intercept Methods

| Method | Description | Coverage |
|--------|-------------|----------|
| **XQueryTree** | Window hierarchy | Metadata only |
| **XDamage** | Change tracking | Efficient updates |
| **XFixes** | Region operations | Clip bounds |
| **Shared memory pixmap** | Texture content | Partial render data |

### Wayland Intercept Methods

| Method | Description | Coverage |
|--------|-------------|----------|
| **xdg-decoration** | Window decorations | Metadata |
| **wl_viewport** | Surface regions | Clip bounds |
| **wl_subcompositor** | Surface hierarchy | Z-order |
| **DMABUF** | Buffer interception | Texture data |

### Approach: Compositor Scene Graph

The compositor maintains a scene graph of all surfaces. We intercept this:

**X11:**
```rust
// Conceptual approach
struct X11CompositorInterceptor {
    // Hook into X11 server's window tree
    // Extract window properties and regions
    // Decode render operations from damage/compositor
    
    fn capture_scene() -> RenderTree {
        // Query all top-level windows
        // Get window properties (bounds, name, class)
        // Track damage regions for changes
        // Build render tree with positions and z-order
    }
}
```

**Wayland:**
```rust
// Conceptual approach
struct WaylandCompositorInterceptor {
    // Hook into Wayland compositor protocols
    // Extract surface hierarchy
    // Decode render operations from compositor
    
    fn capture_scene() -> RenderTree {
        // Get all surfaces from compositor
        // Query surface properties (bounds, role, parent)
        // Track buffer commits for changes
        // Build render tree with positions and z-order
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
│  │  X11/Wayland  │           │  Input Engine │                  │
│  │  Compositor   │           │  (Rust)       │                  │
│  │  Interceptor  │           │               │                  │
│  │  (Rust)       │           │               │                  │
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

### 1. X11/Wayland Compositor Interceptor (Rust)

**Purpose:** Intercept the compositor's scene graph.

**What it captures:**
- All visible windows/surfaces
- Window metadata (title, bounds, class)
- Compositor z-order and transforms
- Render regions and damage

**X11 Output:**
```json
{
  "windows": [
    {
      "id": "win_0x1000001",
      "title": "Visual Studio Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "focused": true,
      "ops": [
        {
          "id": "op_001",
          "bounds": {"x": 10, "y": 10, "w": 100, "h": 30},
          "z": 1,
          "texture_id": "pixmap_0x100"
        }
      ]
    }
  ]
}
```

### 2. Render Decoder (Rust)

**Purpose:** Decode the compositor scene into render operations.

**Pipeline:**
```
X11:
1. Query root window tree
   ↓
2. Enumerate all top-level windows
   ↓
3. Get window properties and regions
   ↓
4. Track damage for changes
   ↓
5. Build render tree
   ↓
6. Output render operations

Wayland:
1. Get all surfaces from compositor
   ↓
2. Query surface properties
   ↓
3. Track buffer commits
   ↓
4. Build render tree
   ↓
5. Output render operations
```

### 3. Input Engine (Rust)

**Purpose:** Execute agent actions on Linux.

**X11 Input:**
```rust
// XTest extension
XTestFakeKeyEvent(display, keycode, is_press, 0);
XFlush(display);
```

**Wayland Input:**
```rust
// Via wl_seat keyboard/mouse
keyboard.add_key(key, state);
seat.pointer.motion(x, y);
```

**Supported Actions:**
- Mouse: click, double-click, move, drag, scroll
- Keyboard: type, key press, key combo

---

## Multi-Environment Support

### Desktop Environment Detection

```rust
enum DesktopEnvironment {
    GNOME,      // Mutter compositor
    KDE,        // KWin compositor
    Xfce,       // Xfwm compositor
    LXDE,       // Openbox
    MATE,       // Marco compositor
    Cinnamon,   // Muffin compositor
    Unknown,
}

fn detect_des() -> DesktopEnvironment {
    // Check DESKTOP_SESSION, XDG_CURRENT_DESKTOP
}
```

### Wayland Compositors

| Compositor | Environment | Intercept Method |
|------------|-------------|------------------|
| Mutter | GNOME | Extended API |
| KWin | KDE | DBus / scripting |
| Sway | i3-based | i3ipc |
| Weston | Generic | Protocol introspection |
| Hyprland | Custom | Custom API |

---

## Coverage

| App Type | Works? | Notes |
|----------|--------|-------|
| X11 apps (GTK, Qt) | ✅ Yes | Via X11 server |
| Wayland native apps | ✅ Yes | Via compositor |
| OpenGL/Vulkan games | ✅ Yes | Captured at compositor |
| Xwayland apps | ✅ Yes | Bridged via X11 |
| Terminal apps | ✅ Yes | Via X11/Wayland |
| TTY | ❌ No | No compositor |
| Wayland (no protocol) | ❌ No | Requires compositor support |

**Coverage: ~90% of desktop apps**

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
  "platform": "linux",
  "driver": "oscp-linux-v0.2",
  "compositor": "x11_wayland",
  "capabilities": ["render_tree", "actions", "events"],
  "features": {
    "compositor_intercept": true,
    "x11": true,
    "wayland": true,
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
# apt
sudo apt install oscp

# dnf
sudo dnf install oscp

# Arch
yay -S oscp
```

### Startup

1. OSCP driver starts as systemd user service
2. Driver detects display server (X11/Wayland)
3. Agent connects via Unix socket
4. Render tree streaming begins

---

## Configuration

```json
{
  "linux": {
    "capture": {
      "frame_rate": 30,
      "display_server": "auto",
      "compositor": "x11_wayland"
    },
    "input": {
      "method": "xtest",
      "action_delay_ms": 0
    },
    "compatibility": {
      "desktop_environment": "auto",
      "fallback_to_x11": true
    }
  }
}
```

---

## Limitations

| Limitation | Description |
|------------|-------------|
| TTY | No compositor access |
| Wayland | Depends on compositor support |
| Some games | DRM or anti-cheat |
| Nested Wayland | Limited support |

---

## Status

🚧 **V1 Target:** X11 compositor interception + render tree + actions

---

## References

- [X11 Protocol](https://www.x.org/docs/X11/x11protocol.pdf)
- [X Damage Extension](https://www.x.org/releases/X11R7.7/doc/kb/xwd.pdf)
- [Wayland Protocols](https://wayland.freedesktop.org/)
- [wlroots](https://gitlab.freedesktop.org/wlroots/wlroots)