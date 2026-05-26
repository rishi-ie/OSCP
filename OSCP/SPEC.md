# OSCP Architecture Specification

**Version:** 0.4.0
**Status:** Specs Complete. Ready for Implementation.

---

## Overview

OSCP is a **wrapper + on-demand layer** on top of existing OS accessibility APIs. Agent requests screen state when needed; OSCP responds with semantic tree.

---

## Architecture

```
AGENT                                    OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (on-demand)                               ├─► Query OS APIs
 │                                           ├─► Fallback chain
 │                                           ├─► Tree analysis
 │  ◄────────────────────────────────────────│
 │  { windows, elements, confidence }          │
 │                                           │
 │  oscp.click(bounds) ─────────────────────►│
 │                                           ├─► Input injection
 │  ◄────────────────────────────────────────│
 │  { success: true }                        │
```

---

## Per-Platform Specs

| Platform | Spec | Status | Time |
|----------|------|--------|------|
| **macOS** | `platforms/macos/SPEC.md` | ✅ Detailed | 4-5 weeks |
| **Linux** | `platforms/linux/SPEC.md` | ✅ Detailed | 6-7 weeks |
| **Windows** | `platforms/windows/SPEC.md` | ⏸️ Deferred | - |

---

## What OSCP Wraps

| Platform | Wraps | Coverage |
|----------|-------|----------|
| **macOS** | AXUIElement + CDP | 95% |
| **Linux** | AT-SPI2 + X11 + CDP | 90-95% |

---

## Protocol

Full protocol specification in `protocol/SPEC.md`:
- All message types defined
- Element models specified
- Error codes defined
- Framing specified
- Request-response model

---

## Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90%)
   └── AXUIElement / AT-SPI2

LEVEL 2: CDP Bridge (Browser/Electron)
   └── Chrome DevTools Protocol

LEVEL 3: Structural Heuristics
   └── Position-based inference

LEVEL 4: Position-Only Mode
   └── Window bounds only

LEVEL 5: Human Handoff
   └── Escalation for edge cases
```

---

## Confidence & Agent Decision

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore first |
| **NONE** | < 0.2 | Explore + handoff |

---

## Implementation Order

1. **macOS** (4-5 weeks)
2. **Linux** (6-7 weeks)
3. **Windows** (deferred)

---

## Status

- [x] Protocol: Complete
- [x] macOS: Detailed spec complete
- [x] Linux: Detailed spec complete
- [ ] Implementation: Pending