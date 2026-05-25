# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Purpose

OSCP is a foundational protocol for agent-native OS interaction. It delivers deterministic, pixel-perfect interaction via semantic tree extraction and hardware-level actuation — enabling agents to see and interact with the full desktop just like humans.

**Key Properties:**
- Deterministic (not probabilistic like VLMs)
- Low-latency (<50ms per action)
- Zero visual parsing
- Handles errors gracefully with fallback hierarchy

---

## Principle

> "Intercept the semantic tree. Deliver coordinates. Agent provides meaning. Handle errors gracefully."

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS (pi)                       │
│                                                                 │
│            ┌────────────────┬────────────────┐                 │
│            │                │                │                  │
│            ▼                ▼                ▼                  │
│     ┌──────────┐      ┌──────────┐                                 │
│     │  macOS   │      │  Linux   │      ┌──────────┐              │
│     │ Platform │      │ Platform │      │ Windows  │              │
│     │  Driver  │      │  Driver  │      │ Platform │              │
│     └──────────┘      └──────────┘      │  Driver  │              │
│            │                │           └──────┬───┘              │
│            └────────────────┴───────────────────┘                  │
│                         │                                        │
│                    Protocol Layer                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Per-Platform Approach

### macOS: AXUIElement

**Semantic Tree:** `AXUIElement` (Accessibility API)
**System State:** `sysctl` + `NSWorkspace`
**Actuation:** `CGEvent` (hardware-level)
**Fallback:** Position-only + CDP

### Linux: AT-SPI2 + X11

**Semantic Tree:** `AT-SPI2` (D-Bus accessibility)
**System State:** `/proc` + `systemd` D-Bus
**Actuation:** `/dev/uinput` (kernel-level)
**Fallback:** X11 + Heuristics

### Windows: UIAutomation + WMI

**Semantic Tree:** `UIAutomation` (UIA)
**System State:** Windows Management Instrumentation (WMI)
**Actuation:** `SendInput` (kernel-level)
**Fallback:** CDP + WMI

---

## Error Handling: The Empty Tree Problem

### The Challenge

Some apps return empty or unhelpful semantic trees:
- Custom renderers (OpenGL, Vulkan, Metal)
- Unlabeled icons (gear icon without text)
- DRM-protected content
- Remote Desktop sessions

### Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90% of apps)
   └── UIA / AXUIElement / AT-SPI2
   
LEVEL 2: CDP Bridge (Electron/Browser apps)
   └── Chrome DevTools Protocol for DOM extraction
   
LEVEL 3: Structural Heuristics (Custom UIs)
   └── Infer from position, size, z-order patterns
   
LEVEL 4: Position-Only Mode (Games, Custom renderers)
   └── Agent learns through trial and error
   
LEVEL 5: Human Handoff (Escalation)
   └── Report failure, ask for guidance
```

### Tree Quality Metrics

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

| Metric | Trigger |
|--------|---------|
| `coverage_score < 0.3` | Low confidence - trigger fallback |
| `named_elements / total < 0.5` | Many unlabeled - low confidence |
| `root_has_no_children` | Custom renderer detected |

### Confidence Scoring

| Confidence | Threshold | Action |
|------------|-----------|--------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore first |
| **NONE** | < 0.2 | Human handoff |

### Action Result Format

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "source": "uia",
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
    "message": "Semantic tree empty, fallback attempted",
    "reasoning": "Custom renderer detected, no element names",
    "alternatives": [
      {"bounds": {"x": 1700, "y": 5}, "confidence": 0.3},
      {"bounds": {"x": 1750, "y": 5}, "confidence": 0.2}
    ]
  }
}
```

---

## CDP Bridge (Electron/Browser Apps)

For Electron apps and browsers where native accessibility fails:

```json
{
  "action": "cdp_snapshot",
  "target": "code.exe",
  "returns": {
    "dom_tree": {
      "selector": "#save-button",
      "text": "Save",
      "bounding_box": {"x": 1750, "y": 5, "w": 80, "h": 25}
    },
    "computed_styles": {...},
    "element_labels": [...]
  }
}
```

**Covers:** Chrome, Edge, Firefox, VS Code, Slack, Discord, Notion, Figma.

---

## What Agent Receives

### Standard Frame

```json
{
  "type": "render_tree",
  "frame_id": 12345,
  "timestamp": 1716576000000,
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
          "state": ["enabled", "visible"],
          "confidence": 0.95,
          "source": "uia"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH"
  },
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_element_id": "e_042"
  }
}
```

### Fallback Frame (Empty Tree)

```json
{
  "type": "render_tree",
  "frame_id": 12346,
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x500001",
      "title": "CustomApp",
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
    "unlabeled_elements": 1,
    "confidence": "NONE"
  },
  "recommended_action": "human_handoff"
}
```

---

## Coverage

| Platform | Primary | Fallback | Total |
|----------|---------|----------|-------|
| **macOS** | AXUIElement (90%) | Position-only | 95% |
| **Linux** | AT-SPI2 (85%) | X11 + Heuristics | 90-95% |
| **Windows** | UIA (85%) | CDP + WMI | 90% |

---

## Implementation Roadmap

| Phase | Description | Timeline |
|-------|-------------|----------|
| **V1** | macOS driver (AXUIElement + error handling) | 2-3 months |
| **V1** | Linux driver (AT-SPI2 + X11 fallback) | 3-4 months |
| **V1** | Windows driver (UIA + CDP fallback) | 1-2 months |
| **V2** | Windows render ops (DWM hook) | 5-9 months |

---

## Directory Structure

```
OSCP/
├── protocol/           # Protocol specification
│   └── SPEC.md        # Core protocol
│
├── platforms/          # OS-specific drivers
│   ├── macos/         # AXUIElement + fallbacks
│   ├── linux/         # AT-SPI2 + X11
│   └── windows/       # UIA + CDP
│
├── agents/             # Agent integration
│   └── SPEC.md       # Agent SDK guidelines
│
└── README.md          # Project overview
```

---

## Core Principles

1. **Determinism over probability** — Semantic trees are mathematical facts, not guesses
2. **Zero visual parsing** — No VLMs, no screenshots, no pixel analysis
3. **Graceful degradation** — Fallback hierarchy ensures agents never get stuck
4. **Hardware-level actuation** — OS cannot distinguish agent from human
5. **Agent provides meaning** — Protocol delivers coordinates, agent supplies semantics

---

## Status

🚧 **Phase 0** — Design complete. V1 implementation with error handling.

---

*OSCP — First-class access for agents.*