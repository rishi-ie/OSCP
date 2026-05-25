# OSCP Linux Platform Driver Specification

**Version:** 0.1.0
**Status:** Draft

---

## Overview

The Linux platform driver provides OSCP protocol implementation for Linux distributions. It supports multiple desktop environments (GNOME, KDE, Xfce, etc.) via X11 and Wayland.

**Principle:** Intercept at the source. Deliver the geometry. Agent provides the meaning.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌───────────────┐           ┌───────────────┐                  │
│  │  Window       │           │  Input Engine │                  │
│  │  Monitor      │           │  (Rust)       │                  │
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

## Display Server Support

Linux has multiple display servers. OSCP targets them in priority order:

| Priority | Display Server | Coverage |
|----------|---------------|----------|
| 1 | **X11** | ~90% of desktops |
| 2 | **Wayland** | Growing |
| 3 | **TTY** | None (no GUI capture) |

### X11 Approach

**Window Enumeration:**
```c
XQueryTree(display, RootWindow(display, screen),
           &root, &parent, &children, &nchildren);
```

**Window Properties:**
```c
XGetWindowProperty(display, win, XA_WM_NAME, ...);
XGetWindowProperty(display, win, XA_WM_CLASS, ...);
```

**Render Operation Extraction:**
- Use `XShmGetImage` for window content capture
- Use `XDamage` for efficient change detection
- Map textures and positions from window content

### Wayland Approach

**Compositor Introspection:**
- Weston: weston-info, custom protocols
- GNOME: gnome-shell built-in
- KDE: KWin introspection

**Frame Capture:**
```c
wl_shm + pixman for window capture
```

---

## Components

### 1. Window Monitor (Rust)

**Purpose:** Track all windows across display servers.

**X11 Implementation:**
```rust
struct X11WindowMonitor {
    display: XConnection,
    root: Window,
}

impl X11WindowMonitor {
    fn get_windows(&self) -> Vec<WindowInfo> {
        // Query tree, get properties, track changes
    }
}
```

**Wayland Implementation:**
```rust
struct WaylandWindowMonitor {
    // Use xdg-foreign for window tracking
}
```

**Tracking:**
- `_NET_CLIENT_LIST` for all windows
- `_NET_ACTIVE_WINDOW` for focused
- `PropertyNotify` for changes

### 2. Render Operations

Window content analyzed to extract render operations:

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

### 3. Input Engine (Rust)

**Purpose:** Execute agent actions on Linux.

**X11 Input:**
```rust
// XTest extension
XTestFakeKeyEvent(display, keycode, is_press, 0);
XFlush(display);

// XSendEvent for targeted input
XSendEvent(display, window, propagate, event_mask, &event);
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
    GNOME,
    KDE,
    Xfce,
    LXDE,
    MATE,
    Cinnamon,
    Unknown,
}

fn detect_des() -> DesktopEnvironment {
    // Check DESKTOP_SESSION, XDG_CURRENT_DESKTOP
}
```

### Environment-Specific Features

| Environment | Capture Method | Notes |
|-------------|---------------|-------|
| GNOME | X11/Wayland | Shell extensions available |
| KDE | X11/Wayland | KWin scripts |
| Xfce | X11 | Mature |
| Cinnamon | X11 | Based on GNOME |

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
  "driver": "oscp-linux-v0.1",
  "capabilities": ["frames", "actions", "events"],
  "features": {
    "window_monitor": true,
    "multi_window": true,
    "multi_monitor": true,
    "display_server": ["x11", "wayland"]
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
2. Driver detects display server
3. Agent connects via Unix socket
4. Frame streaming begins

---

## Configuration

```json
{
  "linux": {
    "capture": {
      "frame_rate": 30,
      "display_server": "auto"
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

## Permissions

**X11:**
- No special permissions for window listing
- Screen capture requires X11 access control

**Wayland:**
- Screenshot portal (xdg-desktop-portal)
- Screen cast API (pipewire)

---

## Limitations

| Limitation | Description |
|------------|-------------|
| X11 only | No TTY capture |
| Wayland | Compositor-dependent support |
| Some apps | May block capture |

---

## Status

🚧 **V1 Target:** X11 window capture + raw geometry + basic actions

---

## References

- [X11 Protocol](https://www.x.org/docs/X11/x11protocol.pdf)
- [wlroots](https://gitlab.freedesktop.org/wlroots/wlroots) - Wayland compositor library