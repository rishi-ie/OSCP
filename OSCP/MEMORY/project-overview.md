# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It unifies existing OS accessibility APIs into a real-time, deterministic, agent-native interface for desktop automation.

**Principle:** "Unify existing tools. Add real-time streaming. Handle errors gracefully. Agent provides meaning."

## Core Concept

```
┌─────────────────────────────────────────────────────────────┐
│                 OSCP: Integration + Streaming                 │
│                                                             │
│  Existing Tools:          OSCP adds:                        │
│  ├── AXUIElement    ───► │ Real-time streaming (30fps)    │
│  ├── AT-SPI2       ───► │ Unified protocol (all OS)      │
│  ├── UIA           ───► │ Error handling (fallbacks)     │
│  └── Input APIs    ───► │ Agent-friendly output          │
└─────────────────────────────────────────────────────────────┘
```

**The hard parts (semantic tree extraction) are already built.**
OSCP's contribution is unifying, streaming, and error handling.

## What Already Exists

| Platform | Existing Tool | What It Does |
|----------|--------------|--------------|
| **macOS** | `pyax`, `ax-element`, `accessibility-service` | Extract AXUIElement tree |
| **Linux** | `dogtail`, `pyatspi`, `ldtp`, `at-spi2-core` | Extract AT-SPI2 tree |
| **Windows** | `pywinauto`, `uiautomation`, `UIAutomationCore` | Extract UIA tree |

**The semantic tree extraction is already done by existing tools.**

## What OSCP Adds

| Feature | Description |
|---------|-------------|
| **Real-time streaming** | 30fps render tree updates (existing tools are one-shot) |
| **Unified protocol** | Same format on macOS/Linux/Windows (existing tools are per-platform) |
| **Error handling** | Fallback hierarchy, tree quality analysis, human handoff |
| **Input engine** | Hardware-level input (well-documented) |
| **Agent output** | Confidence scores, tree analysis metrics |

## Per-Platform Approach

### macOS

**Existing:** AXUIElement (Apple's accessibility API)
**Wrap:** `pyax`, `ax-element`, or Rust bindings

**Fallback Hierarchy:**
```
1. AXUIElement (90%)
2. CDP Bridge (Safari, Chrome, Electron)
3. Position-Only Mode
4. Human Handoff
```

### Linux

**Existing:** AT-SPI2 (D-Bus accessibility)
**Wrap:** `dogtail`, `pyatspi`, `ldtp`

**Fallback Hierarchy:**
```
1. AT-SPI2 (85%)
2. X11 (XQueryTree)
3. CDP Bridge (Chrome, Firefox, Electron)
4. Heuristics
5. Human Handoff
```

### Windows

**Existing:** UIAutomation + Win32
**Wrap:** `pywinauto`, `UIAutomationCore`

**Fallback Hierarchy:**
```
1. UIAutomation (85%)
2. CDP Bridge (Chrome, Edge, VS Code)
3. Win32 EnumWindows
4. Position-Only Mode
5. Human Handoff
```

## Revised Complexity

| Component | Complexity | Notes |
|-----------|------------|-------|
| **Semantic tree extraction** | Already done | 0 weeks |
| **Real-time streaming** | Medium | 2-3 weeks |
| **Unified protocol** | Low | 1 week |
| **Error handling** | Low | 1 week |
| **Input engine** | Low | 1 week |
| **Testing** | Medium | 2 weeks |

## Revised Time Estimates

| Platform | Original | Revised |
|----------|----------|---------|
| **macOS** | 6-8 weeks | 4-6 weeks |
| **Linux** | 10-14 weeks | 6-8 weeks |
| **Windows** | 7-9 weeks | 4-6 weeks |

**Total: 12-16 weeks (not 14-17)**

## Coverage

| Platform | Coverage | Blind Spots |
|----------|----------|-------------|
| **macOS** | 95% | Screen sharing, DRM |
| **Linux** | 90-95% | Some Wayland, TTY |
| **Windows** | 90% | Non-UIA apps, protected |

## Error Handling

### Tree Quality Metrics

```json
{
  "coverage_score": 0.85,
  "named_elements": 45,
  "unlabeled_elements": 3,
  "confidence": "HIGH"
}
```

### Confidence Thresholds

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| HIGH | > 0.8 | Execute immediately |
| MEDIUM | 0.5-0.8 | Execute with monitoring |
| LOW | 0.3-0.5 | Explore first |
| NONE | < 0.2 | Human handoff |

## Status

Phase 0 complete. V1 implementation starting.
Key insight: The hard parts are already built. OSCP is integration work.

## References

- GitHub: github.com/rishi-ie/OSCP
- Existing tools: dogtail, pywinauto, pyax, pyatspi
- Protocol: OSCP/protocol/SPEC.md
- Platforms: OSCP/platforms/{macos,linux,windows}/SPEC.md