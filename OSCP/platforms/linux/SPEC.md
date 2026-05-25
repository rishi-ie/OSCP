# OSCP Linux Platform Driver Specification

**Version:** 0.3.0
**Status:** Architecture Finalized

---

## Overview

The Linux platform driver wraps AT-SPI2 and X11 with a streaming layer.

**Key Insight:** AT-SPI2 and X11 already extract the semantic tree. OSCP adds streaming, unified protocol, and error handling.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Platform Driver                             │
│                                                             │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────┐ │
│  │  STREAMING      │  │  ERROR HANDLER   │  │   INPUT   │ │
│  │   ENGINE        │  │  + FALLBACKS     │  │  ENGINE   │ │
│  │  (30fps)        │  │                  │  │           │ │
│  └────────┬────────┘  └────────┬─────────┘  └─────┬─────┘ │
│           │                    │                  │         │
│           │             ┌──────▼──────┐          │         │
│           │             │   TREE      │◄─────────┘         │
│           │             │  ANALYZER   │                    │
│           │             └──────┬──────┘                    │
│           │                    │                          │
│           │    ┌───────────────┴────────────┐            │
│           │    │                            │            │
│           │    ▼                            ▼            │
│           │   ▼                              ▼           │
│  ┌────────┴──────────────┐  ┌──────────────────────────┐ │
│  │   PRIMARY CAPTURE      │  │    FALLBACK METHODS      │ │
│  │                        │  │                          │ │
│  │   AT-SPI2              │  │    CDP Bridge             │ │
│  │   ──────────           │  │    (Chrome/Electron)     │ │
│  │   Coverage: 85%        │  │                          │ │
│  │                        │  │    X11                   │ │
│  │                        │  │    (XQueryTree)          │ │
│  │                        │  │    — X11 desktops        │ │
│  │                        │  │    — Xwayland apps       │ │
│  │                        │  │                          │ │
│  │                        │  │    Position-Only Mode     │ │
│  │                        │  │                          │ │
│  │                        │  │    Human Handoff          │ │
│  └────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 INPUT ENGINE                         │  │
│  │  /dev/uinput + XTest fallback                        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Wrapped Technology

### AT-SPI2 (Primary)

```
EXISTING: AT-SPI2 (D-Bus accessibility)
WRAPPERS: dogtail, pyatspi, ldtp
COVERAGE: 85%

Provides:
├── Element role (button, menu, etc.)
├── Accessible name
├── Bounds (position, size)
├── States (enabled, visible, etc.)
├── Application enumeration
└── Window hierarchy
```

### X11 (Fallback for X11 + Xwayland)

```
EXISTING: X11 Protocol
APIS: XQueryTree, XGetWindowProperty, XGetWindowAttributes
COVERAGE: +5% (X11 desktops, Xwayland apps)

Provides:
├── Window list
├── Window hierarchy
├── Window metadata (title, class)
└── Window bounds
```

### Input Engine

```
PRIMARY: /dev/uinput (Linux kernel interface)
FALLBACK: XTest (X11 desktops)

Provides:
├── Mouse click/move/drag
├── Keyboard input
└── Key combinations
```

---

## Streaming Engine

```
IMPLEMENTATION: AT-SPI2 Observer + Poll Interval

AT-SPI2 Events (preferred):
├── Accessible event bus
├── D-Bus signals for changes
└── Real-time notifications

Poll Interval (fallback):
├── 33ms interval (30fps)
├── Query all accessible apps
└── Extract changed elements

FALLBACK: X11 polling
└── When AT-SPI2 unavailable
```

---

## Fallback Hierarchy

```
LEVEL 1: AT-SPI2 (Primary)
├── GTK apps
├── Qt apps
├── Java Swing (if AT-SPI enabled)
└── Coverage: 85%

LEVEL 2: CDP Bridge
├── Chrome, Firefox
├── Electron apps (VS Code, Slack, Discord)
└── Coverage: +5%

LEVEL 3: X11
├── X11 desktops
├── Xwayland apps on Wayland compositors
└── Coverage: +5%

LEVEL 4: Position-Only
├── SDL/GL games
├── Custom renderers
└── Works when tree is empty

LEVEL 5: Human Handoff
├── Wayland-native apps (no accessibility)
└── Unrecoverable cases
```

---

## Tree Analyzer

```python
def analyze_tree(tree):
    coverage = tree.covered_area / tree.window_area
    named_ratio = tree.named_count / tree.total_count
    
    if coverage < 0.3:
        confidence = "NONE"      # Empty tree
    elif coverage < 0.5:
        confidence = "LOW"      # Minimal tree
    elif coverage < 0.8:
        confidence = "MEDIUM"   # Partial tree
    else:
        confidence = "HIGH"     # Full tree
    
    return {
        "coverage_score": coverage,
        "named_elements": tree.named_count,
        "unlabeled_elements": tree.unlabeled_count,
        "confidence": confidence,
        "recommended_action": recommended_action[confidence]
    }
```

---

## Error Handling

```
EMPTY TREE DETECTED:
├── Check if app supports AT-SPI2
├── Try CDP bridge (if browser/Electron)
├── Try X11 (if X11/Xwayland)
├── Fall back to position-only mode
└── Report to agent with confidence

STRUCTURAL HEURISTICS:
├── Even if tree is minimal, infer positions
├── Toolbar pattern: top bar, small height
├── Sidebar pattern: left or right, vertical
└── Modal pattern: centered, high z-order
```

---

## Output Format

```json
{
  "type": "render_tree",
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "states": ["enabled", "visible"],
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
    "confidence": "HIGH",
    "recommended_action": "execute"
  }
}
```

---

## Implementation Stack

| Layer | Technology |
|-------|------------|
| **Driver** | Python or Rust |
| **AT-SPI2 Wrapper** | dogtail or pyatspi |
| **Protocol Server** | Unix socket + JSON |
| **Input** | /dev/uinput (C/Python) |

---

## System Requirements

```
REQUIRED:
├── at-spi2-core package
├── at-spi-bus running
└── Accessibility enabled in desktop environment

OPTIONAL:
├── X11 (for X11 fallback)
├── Chrome/Firefox (for CDP bridge)
└── Accessibility permissions granted
```

---

## Installation

```bash
# apt
sudo apt install oscp

# dnf
sudo dnf install oscp

# Requires at-spi2-core
sudo apt install at-spi2-core
```

---

## Time Estimate

| Component | Complexity | Time |
|-----------|-----------|------|
| AT-SPI2 wrapping | Medium | 2-3 weeks |
| X11 fallback | Low | 1 week |
| Streaming engine | Medium | 2-3 weeks |
| Error handling | Low | 1 week |
| Input engine | Medium | 2 weeks |
| Testing | High | 2-3 weeks |
| **Total Linux** | | **6-8 weeks** |

---

## Status

✅ Architecture documented
✅ Existing tools identified
⏳ Implementation pending