# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It delivers deterministic, pixel-perfect interaction via semantic tree extraction and hardware-level actuation — enabling agents to see and interact with the full desktop just like humans.

**Principle:** "Intercept the semantic tree. Deliver coordinates. Agent provides meaning. Handle errors gracefully."

## Core Concept

OSCP makes agents first-class citizens. No VLMs. No screenshots. No guessing.

```
┌─────────────────────────────────────────────────────────────┐
│                 FIRST-CLASS AGENT = OSCP + SKILLS          │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  OSCP           │ +  │  Agent Skills   │  = First-Class  │
│  │  (Deterministic)│    │  (Inference)    │    Citizen      │
│  └─────────────────┘    └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### Key Properties
- **Deterministic** — Semantic trees are mathematical facts, not guesses
- **Low-latency** — <50ms per action vs 1-5s for VLMs
- **Zero visual parsing** — No screenshots, no pixel analysis
- **Error-resilient** — Fallback hierarchy ensures agents never get stuck
- **Hardware-level** — OS cannot distinguish agent from human

## Per-Platform Approach

### macOS: AXUIElement + Fallbacks

| Level | Method | Coverage |
|-------|--------|----------|
| 1 | AXUIElement | 90% |
| 2 | CDP (Browser/Electron) | 95% |
| 3 | Position-only | 95% |

### Linux: AT-SPI2 + X11 + Fallbacks

| Level | Method | Coverage |
|-------|--------|----------|
| 1 | AT-SPI2 | 85% |
| 2 | X11 (XQueryTree) | 90% |
| 3 | CDP | 95% |
| 4 | Heuristics | 95% |

### Windows: UIA + Win32 + Fallbacks

| Level | Method | Coverage |
|-------|--------|----------|
| 1 | UIAutomation | 85% |
| 2 | CDP (Electron/Browser) | 90% |
| 3 | Win32 (EnumWindows) | 90% |
| 4 | Position-only | 90% |

## Error Handling: The Empty Tree Problem

When semantic tree is empty or unhelpful:

```
LEVEL 1: Native Semantic Tree (90% of apps)
   ↓ works
LEVEL 2: CDP Bridge (Electron/Browser)
   ↓ works
LEVEL 3: Structural Heuristics
   ↓ works
LEVEL 4: Position-Only Mode
   ↓ works
LEVEL 5: Human Handoff
```

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

## Coverage

| Platform | Coverage | Blind Spots |
|----------|----------|-------------|
| **macOS** | 95% | Screen sharing, DRM |
| **Linux** | 90-95% | Some Wayland, TTY |
| **Windows** | 90% | Non-UIA apps, protected |

## Gaps After V1

| Gap | Fillable? |
|-----|-----------|
| Element semantics (macOS/Linux) | ✅ Agent skills |
| WebGL/Canvas content | ⚠️ Partially via CDP |
| Color semantics | ✅ Protocol extension |
| Custom renderers | ✅ Position-only mode |
| Protected content | ❌ OS restriction |
| Audio | ✅ Separate API later |

## Status

Phase 0 complete. V1 implementation starting for all three platforms with error handling.

## References

- GitHub: github.com/rishi-ie/OSCP
- Protocol: OSCP/protocol/SPEC.md
- Platforms: OSCP/platforms/{macos,linux,windows}/SPEC.md