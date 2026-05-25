# OSCP Linux Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The Linux platform driver provides OSCP protocol implementation for Linux distributions. It uses AT-SPI2 with X11 fallback and a comprehensive fallback hierarchy for problematic apps.

**Principle:** Deterministic. Error-resilient. Graceful degradation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   AT-SPI2    │  │     X11      │  │   Input Engine   │   │
│  │  Accessibility │  │   Window Enum │  │   (/dev/uinput) │   │
│  └───────┬──────┘  └───────┬──────┘  └────────┬─────────┘   │
│          │                 │                  │               │
│          └──────────┬──────┘                  │               │
│                     │                         │               │
│              ┌──────▼──────┐                   │               │
│              │    Tree     │◄──────────────────┘               │
│              │   Builder   │                                   │
│              │   + Error   │                                   │
│              │   Handler   │                                   │
│              └──────┬──────┘                                   │
│                     │                                          │
│              ┌──────▼──────┐                                   │
│              │  Protocol   │                                   │
│              │  Server     │                                   │
│              │  (Unix)     │                                   │
│              └─────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Primary Capture: AT-SPI2

### What It Captures

```rust
// AT-SPI2 via D-Bus:
struct AT_SPIElement {
    role: String,           // ROLE_BUTTON, ROLE_MENU, etc.
    name: String,           // Accessible name
    description: String,    // Accessible description
    bounds: Rect,           // Position and size
    states: Vec<State>,     // STATE_ENABLED, etc.
    relations: Vec<Relation>, // Labeled-by, etc.
    children: Vec<AT_SPIElement>,
}
```

### Coverage

| App Type | Works? | Notes |
|----------|--------|-------|
| GTK apps | ✅ | Full AT-SPI2 support |
| Qt apps | ✅ | Full AT-SPI2 support |
| Java Swing | ⚠️ | If AT-SPI enabled |
| Electron | ⚠️ | Basic support |
| SDL/GL apps | ❌ | No AT-SPI |

**Primary Coverage: ~85%**

---

## Fallback Hierarchy

### Level 1: AT-SPI2 (Primary)

```rust
fn capture_atspi() -> Result<ElementTree> {
    // Connect to at-spi-bus
    // Enumerate all applications
    // Extract element tree for each window
    // Returns full semantic tree
}
```

### Level 2: X11 (X11 Desktops + Xwayland)

```rust
fn capture_x11() -> PartialTree {
    // XQueryTree for window hierarchy
    // XGetWindowProperty for metadata
    // XGetWindowAttributes for bounds
    // Covers: X11 desktops, Xwayland apps
    // ~85% of Wayland desktops via Xwayland
}
```

### Level 3: CDP Bridge (Electron/Browser)

```rust
async fn capture_cdp(app: &str) -> Result<ElementTree> {
    // Connect to Chrome/Electron CDP port
    // Extract DOM tree with bounding boxes
    // Covers: Chrome, Firefox, Electron apps
}
```

### Level 4: Structural Heuristics

```rust
fn heuristics(window: &Window) -> InferredElements {
    // Even if tree is empty, we get window bounds
    // Infer common UI patterns:
    // - Toolbar at top, small height
    // - Sidebar at left, vertical items
    // - Modal centered, high z-order
    
    // Works for: Custom UIs with standard layouts
}
```

### Level 5: Position-Only Mode

```rust
fn position_only(window: &Window) -> EmptyTree {
    // Return window bounds only
    // Agent must learn through exploration
    // Triggered when coverage_score < 0.3
}
```

### Level 6: Human Handoff

```rust
fn human_handoff(reason: &str, attempts: u32) -> HandoffRequest {
    // Report failure
    // Ask for guidance
}
```

---

## X11 + Wayland Dual Strategy

### X11 (Universal)

```rust
fn capture_x11(display: &X11Display) -> Vec<WindowInfo> {
    // XQueryTree - window hierarchy
    // XGetWindowProperty - titles, classes
    // XGetWindowAttributes - positions, sizes
    // XDamage - change tracking
    // Works on: X11 desktops (70% of Linux)
    // Works on: Xwayland apps on Wayland desktops (95% of apps)
}
```

### Wayland Compositor Detection

```rust
enum WaylandCompositor {
    Gnome,  // Mutter
    KDE,    // KWin
    Sway,
    River,
    Hyprland,
    Weston,
    Unknown,
}

fn detect_compositor() -> WaylandCompositor {
    // Check XDG_CURRENT_DESKTOP
    // Check WAYLAND_DISPLAY
    // Return compositor type
}
```

### Wayland Native Support (Per Compositor)

```rust
async fn capture_wayland_native(compositor: WaylandCompositor) 
    -> Option<ElementTree> 
{
    match compositor {
        Gnome => capture_gnome_shell().await,
        KDE => capture_kwin().await,
        Sway => capture_sway_ipc().await,
        _ => None,  // Fall back to X11/Xwayland
    }
}
```

**Wayland Native Coverage: ~5-10% additional**

---

## Error Detection

### Tree Quality Analysis

```rust
struct TreeAnalysis {
    coverage_score: f32,      // 0.0-1.0
    named_elements: u32,
    unlabeled_elements: u32,
    avg_depth: f32,
    confidence: Confidence,
}

enum Confidence {
    HIGH,    // > 0.8
    MEDIUM,  // 0.5-0.8
    LOW,     // 0.3-0.5
    NONE,    // < 0.2
}
```

### Detection Triggers

```rust
fn analyze_tree(tree: &ElementTree) -> TreeAnalysis {
    let named_ratio = tree.named_count as f32 / tree.total_count as f32;
    let coverage = tree.covered_area / tree.window_area;
    
    if coverage < 0.3 {
        return TreeAnalysis {
            confidence: Confidence::NONE,
            recommended_fallback: "position_only"
        };
    }
    
    if named_ratio < 0.5 {
        return TreeAnalysis {
            confidence: Confidence::LOW,
            recommended_fallback: "heuristics"
        };
    }
    
    // Custom renderer detection
    if tree.avg_depth < 2.0 && tree.named_count < 5 {
        return TreeAnalysis {
            confidence: Confidence::NONE,
            recommended_fallback: "x11_position_only"
        };
    }
    
    // ...
}
```

---

## Output Format

### Standard Frame (AT-SPI2)

```json
{
  "type": "render_tree",
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "state": ["enabled", "visible"],
          "confidence": 0.95,
          "source": "atspi"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH"
  }
}
```

### Fallback Frame

```json
{
  "type": "render_tree",
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x500001",
      "title": "CustomGame",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "elements": [],
      "fallback_active": true,
      "fallback_method": "position_only",
      "fallback_reason": "custom_renderer_detected"
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.05,
    "named_elements": 0,
    "confidence": "NONE",
    "recommended_action": "human_handoff"
  }
}
```

---

## Action Result with Confidence

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "source": "atspi",
  "error": null
}
```

```json
{
  "type": "action_result",
  "action_id": "act_002",
  "success": false,
  "confidence": 0.2,
  "source": "heuristic",
  "error": {
    "code": "EMPTY_TREE",
    "message": "Semantic tree empty",
    "reasoning": "SDL renderer detected, no AT-SPI",
    "alternatives": [
      {"bounds": {"x": 960, "y": 540}, "confidence": 0.3}
    ],
    "recommended_action": "explore_and_confirm"
  }
}
```

---

## Protocol Implementation

### Transport

Default: `unix:///tmp/oscp.sock`

### Capabilities

```json
{
  "platform": "linux",
  "driver": "oscp-linux-v0.2",
  "semantic_apis": ["atspi2", "x11"],
  "fallback_methods": ["cdp", "heuristics", "position_only", "human_handoff"],
  "compositor_support": {
    "x11": true,
    "gnome": true,
    "kde": true,
    "sway": true,
    "other": false
  },
  "capabilities": ["render_tree", "actions", "events", "error_handling"],
  "features": {
    "element_tree": true,
    "cdp_bridge": true,
    "heuristics": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Installation

```bash
# apt
sudo apt install oscp

# dnf
sudo dnf install oscp

# Arch
yay -S oscp
```

---

## System Requirements

- AT-SPI2 daemon running (`at-spi2-arc` package)
- X11 or Wayland compositor
- Accessibility enabled in desktop environment

---

## Status

🚧 **V1 Target:** AT-SPI2 + X11 + fallback hierarchy

---

## References

- [AT-SPI2 Specification](https://www.freedesktop.org/wiki/Accessibility/)
- [X11 Protocol](https://www.x.org/docs/X11/x11protocol.pdf)
- [wlroots](https://gitlab.freedesktop.org/wlroots/wlroots)