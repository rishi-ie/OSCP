# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It is a **wrapper + streaming layer** on top of existing OS accessibility APIs.

**Key Insight:** The hard parts are already built. OSCP adds real-time streaming, unified protocol, and error handling.

**Principle:** "Wrap existing tools. Add real-time streaming. Handle errors gracefully. Agent provides meaning."

## Core Architecture

```
OSCP = WRAPPER + STREAMING LAYER
        │              │
        │              ├── Real-time 30fps
        │              ├── Unified protocol (all platforms)
        │              ├── Error handling (5-level fallback)
        │              └── Agent-friendly output (confidence scores)
        │
        └── Existing OS Accessibility APIs
             ├── AXUIElement (macOS) — 90% coverage
             ├── AT-SPI2 (Linux) — 85% coverage
             └── UIAutomation (Windows) — 85% coverage
```

## What OSCP Wraps

| Platform | Wraps | Coverage | Time |
|----------|-------|----------|------|
| **macOS** | AXUIElement + pyax/ax-element | 95% | 4-6 weeks |
| **Linux** | AT-SPI2 + X11 + dogtail | 90-95% | 6-8 weeks |
| **Windows** | UIAutomation + pywinauto | 90% | 6-8 weeks |

## What OSCP Adds

| Component | Description |
|-----------|-------------|
| **Streaming Engine** | 30fps real-time updates (existing APIs are one-shot) |
| **Unified Protocol** | Same JSON format on all platforms |
| **Error Handler** | 5-level fallback hierarchy for empty/unhelpful trees |
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
| **HIGH** | > 0.8 coverage | Execute immediately |
| **MEDIUM** | 0.5-0.8 coverage | Execute with monitoring |
| **LOW** | 0.3-0.5 coverage | Explore candidates first |
| **NONE** | < 0.3 coverage | Explore + confirm or handoff |

## Agent Success Rate

| Workload | Agent Success | Human Involvement |
|----------|--------------|-------------------|
| Standard desktop (VS Code, Chrome, Terminal) | 95% | 5% |
| Standard web (DOM-accessible) | 90% | 10% |
| Custom apps (learns as goes) | 80% | 20% |
| Games (custom renderers) | 50% (exploration) | 50% |
| DRM video | 0% (blocked) | 100% |
| Canvas apps (Figma, CAD) | 0% | 100% |
| Remote desktop | 0% | 100% |

**Overall success rate: 85-90% for typical desktop tasks.**

## Implementation Order

1. **macOS** first (simplest, AXUIElement is well-documented)
2. **Linux** second (more complex, D-Bus, Wayland)
3. **Windows** third (similar to macOS)

## Project Status

- [x] Architecture finalized
- [x] Existing tools identified
- [x] Per-platform approach defined
- [ ] Implementation pending

## Key Files

| File | Content |
|------|---------|
| `SPEC.md` | Full architecture diagram |
| `MEMORY/architecture.md` | Architecture reference |
| `platforms/macos/SPEC.md` | macOS approach |
| `platforms/linux/SPEC.md` | Linux approach |
| `platforms/windows/SPEC.md` | Windows approach |
| `protocol/SPEC.md` | Protocol format |

## References

- GitHub: github.com/rishi-ie/OSCP
- Protocol: OSCP/protocol/SPEC.md
- Architecture: OSCP/MEMORY/architecture.md