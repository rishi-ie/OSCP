# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It is a **wrapper + streaming layer** on top of existing OS accessibility APIs.

**Key Insight:** The hard parts are already built. OSCP adds real-time streaming, unified protocol, and error handling.

## Core Architecture

```
OSCP = WRAPPER + STREAMING LAYER
        │              │
        │              ├── Real-time 30fps
        │              ├── Unified protocol
        │              ├── Error handling
        │              └── Agent-friendly output
        │
        └── Existing OS Accessibility APIs
             ├── AXUIElement (macOS)
             ├── AT-SPI2 (Linux)
             └── UIAutomation (Windows)
```

## What OSCP Wraps

| Platform | Existing API | Wrappers Available |
|----------|-------------|-------------------|
| **macOS** | AXUIElement | pyax, ax-element |
| **Linux** | AT-SPI2 + X11 | dogtail, pyatspi, ldtp |
| **Windows** | UIAutomation | pywinauto |

## What OSCP Adds

| Component | Description |
|-----------|-------------|
| **Streaming Engine** | 30fps real-time updates (existing APIs are one-shot) |
| **Unified Protocol** | Same format on all platforms |
| **Error Handler** | Fallback hierarchy for empty/unhelpful trees |
| **Tree Analyzer** | Coverage scores, confidence metrics |
| **Input Engine** | Hardware-level actuation (undetectable) |

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
| **LOW** | 0.3-0.5 | Explore first |
| **NONE** | < 0.2 | Human handoff |

## Coverage & Time

| Platform | Wraps | Coverage | Time |
|----------|-------|----------|------|
| **macOS** | AXUIElement | 95% | 4-6 weeks |
| **Linux** | AT-SPI2 + X11 | 90-95% | 6-8 weeks |

## Project Status

Phase 0 complete. Architecture finalized.

## References

- GitHub: github.com/rishi-ie/OSCP
- Protocol: OSCP/protocol/SPEC.md
- Architecture: OSCP/MEMORY/architecture.md
- Specs: OSCP/platforms/{macos,linux,windows}/SPEC.md