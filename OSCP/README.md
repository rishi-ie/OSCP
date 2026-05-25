# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP intercepts the OS's semantic tree and delivers deterministic, pixel-perfect interaction to agents. No VLMs. No screenshots. No guessing.

---

## Principle

> "Intercept the semantic tree. Deliver coordinates. Agent provides meaning. Handle errors gracefully."

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                           │
│                                                                 │
│   Agent receives:                                               │
│   {                                                             │
│     "windows": [{                                               │
│       "id": "win_1",                                            │
│       "title": "VS Code",                                       │
│       "elements": [{                                             │
│         "type": "button",                                        │
│         "name": "Save",                                         │
│         "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},       │
│         "confidence": 0.95                                      │
│       }]                                                        │
│     }],                                                         │
│     "mouse": {"x": 540, "y": 320}                              │
│   }                                                            │
│                                                                 │
│   Agent sends: {"action": "click", "x": 1750, "y": 20}         │
└─────────────────────────────────────────────────────────────┘
                             │
                       OSCP Protocol
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
    ▼                        ▼                        ▼
┌──────────┐          ┌──────────┐          ┌──────────┐
│  macOS   │          │  Linux   │          │ Windows  │
│ Platform │          │ Platform │          │ Platform │
│  Driver  │          │  Driver  │          │  Driver  │
└──────────┘          └──────────┘          └──────────┘
     │                     │                     │
     ▼                     ▼                     ▼
  AXUIElement           AT-SPI2              UIA + WMI
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
                    Error Handling + Fallback
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
              CDP Bridge      Heuristics Engine
              (Electron)      (Custom UIs)
```

---

## Per-Platform Implementation

| Platform | Primary | Fallback | Coverage |
|----------|---------|----------|----------|
| **macOS** | AXUIElement | Position-only + CDP | 95% |
| **Linux** | AT-SPI2 | X11 + Heuristics | 90-95% |
| **Windows** | UIAutomation | CDP + WMI | 90% |

---

## Error Handling Architecture

### The Empty Tree Problem

When the semantic tree is empty or unhelpful (custom renderers, unlabeled icons, DRM), OSCP provides a fallback hierarchy:

```
LEVEL 1: Native Semantic Tree (90% of apps)
   ↓ works
LEVEL 2: CDP Bridge (Electron/Browser apps)
   ↓ works
LEVEL 3: Structural Heuristics (Custom UIs)
   ↓ works
LEVEL 4: Position-Only Mode (Games, Custom renderers)
   ↓ works
LEVEL 5: Human Handoff (Escalation)
```

### Tree Quality Analysis

```json
{
  "tree_analysis": {
    "coverage_score": 0.85,
    "named_elements": 45,
    "unlabeled_elements": 3,
    "avg_depth": 4,
    "confidence": "HIGH"
  }
}
```

- `coverage_score < 0.3` → Trigger fallback
- `named_elements / total < 0.5` → Low confidence
- `root_has_no_children` → Custom renderer detected

### Confidence Scoring

| Confidence | Action |
|------------|--------|
| > 0.8 | Execute immediately |
| 0.5-0.8 | Execute with monitoring |
| < 0.5 | Explore first |
| < 0.2 | Human handoff |

---

## CDP Bridge (Electron/Browser Apps)

For Electron apps (VS Code, Slack, Discord) and browsers:

```json
{
  "action": "cdp_snapshot",
  "target": "electron_app",
  "returns": {
    "dom_tree": {...},
    "bounding_boxes": [...],
    "element_labels": [...]
  }
}
```

Covers: Chrome, Edge, Firefox, VS Code, Slack, Discord, Notion, Figma.

---

## Supported Capabilities

### Read (Semantic Tree)
- Window enumeration
- Element types, names, states
- Bounding rectangles
- Parent-child relationships
- Z-order

### Context (System State)
- Active windows
- Focus state
- Mouse position
- Keyboard modifiers

### Actuate (Input)
- Mouse: click, move, drag, scroll
- Keyboard: type, key press, key combo
- Hardware-level (kernel signals)

---

## Coverage

| Platform | Coverage | Blind Spots |
|----------|----------|-------------|
| **macOS** | 95% | Screen sharing, DRM |
| **Linux** | 90-95% | Some Wayland, TTY |
| **Windows** | 90% | Non-UIA apps, protected |

---

## Status

🚧 **V1 Development** — All three platforms with error handling
🚧 **V2** — Windows render ops (DWM hook)

---

*OSCP — First-class access for agents.*