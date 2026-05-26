# OSCP Architecture Specification

**Version:** 0.4.0
**Status:** Specs Complete. Ready for Implementation.

---

## Overview

OSCP is a **wrapper + on-demand layer** on top of existing OS accessibility APIs. Agent requests screen state when needed; OSCP responds with semantic tree.

## Repository Structure

```
OSCP/
├── README.md                    # Project overview
├── SPEC.md                     # This file
│
├── protocol/
│   └── SPEC.md                 # Protocol specification
│
├── platforms/
│   ├── macos/
│   │   └── SPEC.md             # macOS implementation
│   ├── linux/
│   │   └── SPEC.md             # Linux implementation
│   └── windows/
│       └── SPEC.md             # Deferred
│
├── agents/
│   └── SPEC.md                 # Agent SDK guidelines
│
├── MEMORY/                     # Project context
│   ├── architecture.md
│   ├── DECISIONS.md
│   └── project-overview.md
│
└── docs/                       # Documentation
```

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
```

---

## Per-Platform Specs

| Platform | Spec | Status | Time |
|----------|------|--------|------|
| **macOS** | `platforms/macos/SPEC.md` | ✅ Ready | 4-5 weeks |
| **Linux** | `platforms/linux/SPEC.md` | ✅ Ready | 6-7 weeks |
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

---

## Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90%)
LEVEL 2: CDP Bridge (Browser/Electron)
LEVEL 3: Structural Heuristics
LEVEL 4: Position-Only Mode
LEVEL 5: Human Handoff
```

---

## Confidence

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore first |
| **NONE** | < 0.2 | Human handoff |

---

## Implementation Order

1. **macOS** (4-5 weeks)
2. **Linux** (6-7 weeks)
3. **Windows** (deferred)

---

## Status

- [x] Protocol: Complete
- [x] macOS: Detailed spec ready
- [x] Linux: Detailed spec ready
- [ ] macOS implementation: Pending
- [ ] Linux implementation: Pending
- [ ] Windows: Deferred