# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It is a **wrapper + on-demand layer** on top of existing OS accessibility APIs.

**Version:** 0.4.0
**Status:** Specs complete. Ready for implementation.

**Principle:** "Wrap existing tools. Respond on-demand. Handle errors gracefully. Agent provides meaning."

## Repository Structure

```
OSCP/
├── README                     # Project overview
├── SPEC.md                    # Architecture summary
│
├── protocol/
│   └── SPEC.md                # Protocol specification (ready for impl)
│
├── platforms/
│   ├── macos/
│   │   └── SPEC.md            # macOS implementation spec (ready for impl)
│   ├── linux/
│   │   └── SPEC.md            # Linux implementation spec (ready for impl)
│   └── windows/
│       └── SPEC.md            # Deferred
│
├── agents/
│   └── SPEC.md                # Agent SDK guidelines
│
├── MEMORY/                    # Project context for AI agents
│   ├── architecture.md        # Architecture diagrams
│   ├── DECISIONS.md          # Key decisions
│   └── project-overview.md   # This file
│
└── docs/
```

## How OSCP Works

```
AGENT                                     OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (on-demand)                              ├─► Query OS APIs
 │                                           ├─► Fallback chain
 │  ◄────────────────────────────────────────│
 │  { windows, elements, confidence }          │
```

## Spec Status

| Spec | Status | Ready for Impl? |
|------|--------|----------------|
| **Protocol** | ✅ Complete | Yes |
| **macOS** | ✅ Complete | Yes |
| **Linux** | ✅ Complete | Yes |
| **Windows** | ⏸️ Deferred | No |

## Implementation Stack

### macOS
- **API:** AXUIElement (native C API)
- **Language:** Swift/Objective-C + C bridging
- **Input:** CGEvent
- **Time:** 4-5 weeks

### Linux
- **API:** AT-SPI2 (D-Bus) + X11 (fallback)
- **Language:** Python (pyatspi) or Rust
- **Input:** /dev/uinput + XTest fallback
- **Time:** 6-7 weeks

## Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90%)
   └── AXUIElement / AT-SPI2

LEVEL 2: CDP Bridge (Browser/Electron)
   └── Chrome DevTools Protocol

LEVEL 3: Structural Heuristics
   └── Position-based inference

LEVEL 4: Position-Only Mode
   └── Agent explores

LEVEL 5: Human Handoff
   └── Graceful degradation
```

## Confidence Decision Table

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore candidates first |
| **NONE** | < 0.2 | Human handoff |

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| Total (macOS + Linux) | 10-12 weeks |

## References

- GitHub: github.com/rishi-ie/OSCP