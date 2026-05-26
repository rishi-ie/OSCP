# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It is a **wrapper + on-demand layer** on top of existing OS accessibility APIs. Agent requests screen state when needed; OSCP responds with semantic tree.

**Version:** 0.4.0
**Status:** Specs complete. Ready for implementation.

**Principle:** "Wrap existing tools. Respond on-demand. Handle errors gracefully. Agent provides meaning."

## Core Architecture

```
OSCP = WRAPPER + ON-DEMAND LAYER
        │              │
        │              ├── On-demand request-response
        │              ├── Unified protocol (all platforms)
        │              ├── Error handling (5-level fallback)
        │              └── Agent-friendly output (confidence scores)
        │
        └── Existing OS Accessibility APIs
             ├── AXUIElement (macOS) — 90% coverage
             ├── AT-SPI2 (Linux) — 85% coverage
             └── UIAutomation (Windows) — 85% coverage
```

## How It Works

```
AGENT                                     OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (I need to see the screen)               │
 │                                           ├─► Query OS APIs
 │                                           ├─► Fallback chain
 │                                           ├─► Tree analysis
 │  ◄────────────────────────────────────────│
 │  {                                       │
 │    "windows": [...],                       │
 │    "confidence": "HIGH"                   │
 │  }                                        │
 │                                           │
 │  oscp.click(bounds) ─────────────────────►│
 │  ◄────────────────────────────────────────│
 │  { success: true }                         │
```

## Spec Status

| Spec | Status |
|------|--------|
| **Protocol** | ✅ Complete |
| **macOS** | ✅ Complete (detailed) |
| **Linux** | ✅ Complete (detailed) |
| **Windows** | ⏸️ Deferred |

## Implementation Stack

### macOS

- **API:** AXUIElement (native)
- **Wrapper:** Swift/Objective-C with C bridging
- **Input:** CGEvent
- **Time:** 4-5 weeks

### Linux

- **API:** AT-SPI2 (primary) + X11 (fallback)
- **Wrapper:** Python (pyatspi) or Rust
- **Input:** /dev/uinput + XTest
- **Time:** 6-7 weeks

### Windows

- **API:** UIAutomation
- **Deferred** until after macOS/Linux

## Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90%)
└── AXUIElement / AT-SPI2 / UIA

LEVEL 2: CDP Bridge (Electron/Browser)
└── Chrome DevTools Protocol

LEVEL 3: Structural Heuristics
└── Position-based inference

LEVEL 4: Position-Only Mode
└── Window bounds only

LEVEL 5: Human Handoff
└── Escalation for edge cases
```

## Confidence Decision Table

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore candidates first |
| **NONE** | < 0.2 | Human handoff |

## Agent Success Rate

| Workload | Agent Success | Human Involvement |
|----------|--------------|-------------------|
| Standard desktop (VS Code, Chrome, Terminal) | 95% | 5% |
| Standard web (DOM-accessible) | 90% | 10% |
| Custom apps (learns as goes) | 80% | 20% |
| Games (custom renderers) | 50% | 50% |

**Overall success rate: 85-90% for typical desktop tasks.**

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| Windows | Deferred |
| **Total** | **10-12 weeks** |

## Key Files

| File | Content |
|------|---------|
| `SPEC.md` | Architecture overview |
| `protocol/SPEC.md` | Complete protocol (ready for impl) |
| `platforms/macos/SPEC.md` | Detailed macOS implementation spec |
| `platforms/linux/SPEC.md` | Detailed Linux implementation spec |
| `agents/SPEC.md` | Agent SDK guidelines |
| `MEMORY/DECISIONS.md` | Architectural decisions |

## References

- GitHub: github.com/rishi-ie/OSCP