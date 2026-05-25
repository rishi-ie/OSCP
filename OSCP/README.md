# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP unifies existing OS accessibility APIs into a real-time, deterministic, agent-native interface. The hard parts are already built — OSCP adds streaming, unification, and error handling.

---

## Principle

> "Wrap existing tools. Add real-time streaming. Handle errors gracefully. Agent provides meaning."

---

## Key Insight

```
EXISTING TOOLS:
├── AXUIElement (macOS) — already extracts semantic tree
├── AT-SPI2 (Linux) — already extracts semantic tree
├── UIAutomation (Windows) — already extracts semantic tree
└── These tools work, they're proven

OSCP'S CONTRIBUTION:
├── Real-time streaming (30fps) — existing tools are one-shot
├── Unified protocol — existing tools are per-platform
├── Error handling — existing tools fail silently
└── Agent-native output — existing tools are human-centric
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                           │
│                                                                 │
│   Agent receives:                                               │
│   {                                                             │
│     "windows": [{                                               │
│       "id": "win_1",                                            │
│       "title": "VS Code",                                       │
│       "elements": [...],                                       │
│       "confidence": 0.95                                       │
│     }],                                                         │
│     "tree_analysis": {...}                                    │
│   }                                                            │
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
  EXISTING              EXISTING              EXISTING
  AXUIElement          AT-SPI2               UIAutomation
  (already built)     (already built)      (already built)
```

---

## Per-Platform Implementation

| Platform | Wraps | Coverage | Time |
|----------|-------|----------|------|
| **macOS** | AXUIElement | 95% | 4-6 weeks |
| **Linux** | AT-SPI2 + X11 | 90-95% | 6-8 weeks |
| **Windows** | UIAutomation | 90% | 4-6 weeks |

---

## What OSCP Adds

| Feature | Description |
|---------|-------------|
| **Real-time streaming** | 30fps render tree updates |
| **Unified protocol** | Same format on all platforms |
| **Error handling** | Fallback hierarchy for empty trees |
| **Confidence scoring** | Per-element confidence metrics |
| **Human handoff** | Escalation for edge cases |

---

## Error Handling

```
LEVEL 1: Native Semantic Tree (90% of apps)
LEVEL 2: CDP Bridge (Electron/Browser)
LEVEL 3: Structural Heuristics
LEVEL 4: Position-Only Mode
LEVEL 5: Human Handoff
```

---

## Coverage

| Platform | Coverage | Blind Spots |
|----------|----------|-------------|
| **macOS** | 95% | Screen sharing, DRM |
| **Linux** | 90-95% | Some Wayland, TTY |
| **Windows** | 90% | Non-UIA apps, protected |

---

## Quick Start

```python
import oscp

client = oscp.connect("unix:///tmp/oscp.sock")

async for tree in client.stream():
    if tree.tree_analysis.confidence == "HIGH":
        for element in tree.find_all("button"):
            if element.name == "Save":
                await client.click(element.bounds)
```

---

## Status

🚧 **V1 Development** — Integration work, not new development
🚧 **V2** — Windows render ops (DWM hook)

---

*OSCP — First-class access for agents.*