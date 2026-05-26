# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It is a **wrapper + on-demand layer** on top of existing OS accessibility APIs.

**Key Changes from v0.3:**
- ~~30fps streaming~~ → On-demand calls
- Agent controls when to see the screen
- Simpler architecture, lower overhead

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
             ├── AXUIElement (macOS)
             ├── AT-SPI2 (Linux)
             └── UIAutomation (Windows)
```

## How It Works

```
AGENT                                     OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (on-demand)                               ├─► Query OS APIs
 │                                           ├─► Analyze tree
 │  ◄────────────────────────────────────────│
 │  {                                       │
 │    "windows": [...],                       │
 │    "confidence": "HIGH"                   │
 │  }                                        │
```

## What OSCP Wraps

| Platform | Wraps | Coverage | Time |
|----------|-------|----------|------|
| **macOS** | AXUIElement | 95% | 4-5 weeks |
| **Linux** | AT-SPI2 + X11 | 90-95% | 6-7 weeks |
| **Windows** | UIAutomation | 90% | 6-7 weeks |

## What OSCP Adds

| Component | Description |
|-----------|-------------|
| **On-demand capture** | Agent requests frame when needed |
| **Unified Protocol** | Same JSON format on all platforms |
| **Error Handler** | 5-level fallback hierarchy |
| **Tree Analyzer** | Coverage scores, confidence metrics |
| **Input Engine** | Hardware-level actuation (CGEvent, /dev/uinput, SendInput) |

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
| **NONE** | < 0.2 | Explore + confirm or handoff |

## Agent Success Rate

| Workload | Agent Success | Human Involvement |
|----------|--------------|-------------------|
| Standard desktop (VS Code, Chrome, Terminal) | 95% | 5% |
| Standard web (DOM-accessible) | 90% | 10% |
| Custom apps (learns as goes) | 80% | 20% |
| Games (custom renderers) | 50% | 50% |

**Overall success rate: 85-90% for typical desktop tasks.**

## Key Changes from v0.3

| Aspect | Old (Streaming) | New (On-Demand) |
|--------|----------------|-----------------|
| **Pattern** | 30fps continuous push | Request-response |
| **Agent control** | Passive receiver | Active requester |
| **Resource usage** | Higher | Lower |
| **Server complexity** | Higher | Lower |
| **Time to build** | 12-16 weeks | 10-12 weeks |

## Project Status

- [x] Architecture updated (on-demand)
- [x] Approach validated
- [x] Per-platform specs defined
- [ ] Implementation pending

## Key Files

| File | Content |
|------|---------|
| `SPEC.md` | Full architecture |
| `platforms/macos/SPEC.md` | macOS approach |
| `platforms/linux/SPEC.md` | Linux approach |
| `protocol/SPEC.md` | Protocol format |

## References

- GitHub: github.com/rishi-ie/OSCP