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
│   └── SPEC.md                 # Complete protocol specification
│                                 (with SDK examples: Python, Swift)
│
├── platforms/
│   ├── macos/
│   │   └── SPEC.md             # Complete macOS implementation
│   │                             (every component: Models, Capture,
│   │                              Input, Server, with full code)
│   ├── linux/
│   │   └── SPEC.md             # Detailed Linux implementation
│   └── windows/
│       └── SPEC.md             # Deferred
│
├── agents/
│   └── SPEC.md                 # Agent SDK guidelines
│
├── MEMORY/
│   ├── architecture.md
│   ├── DECISIONS.md
│   └── project-overview.md
│
└── docs/
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

## Specs Complete

### Protocol Specification
**File:** `protocol/SPEC.md` (42KB)
- Complete message types with JSON examples
- Element reference (roles, states, sources)
- Error codes with agent actions
- State machine diagram
- **Python SDK** (full implementation)
- **Swift SDK** (full implementation)
- Connection examples (C, Swift, Python, TypeScript)

### macOS Platform Spec
**File:** `platforms/macos/SPEC.md` (145KB)
- **Complete Models:** Element, Window, Frame, Action, Errors
- **AXUIElement Capture:** Full Swift implementation
- **CDP Bridge:** Complete Swift implementation
- **Tree Builder:** Full implementation
- **Tree Analyzer:** Full implementation
- **Fallback Manager:** Complete 5-level fallback
- **CGEvent Input Engine:** Full implementation
- **Protocol Server:** Complete Unix socket server
- **Main Entry Point:** Ready-to-run main.swift
- **Testing Templates:** Unit test examples
- **Common Pitfalls:** Troubleshooting guide

### Linux Platform Spec
**File:** `platforms/linux/SPEC.md` (31KB)
- AT-SPI2 integration details
- X11 fallback implementation
- /dev/uinput input engine
- **Time: 6-7 weeks**

---

## What OSCP Wraps

| Platform | Wraps | Coverage |
|----------|-------|----------|
| **macOS** | AXUIElement + CDP | 95% |
| **Linux** | AT-SPI2 + X11 + CDP | 90-95% |

---

## Protocol

Full protocol specification in `protocol/SPEC.md`:
- Request-response model (no streaming)
- JSON over newline-delimited Unix socket
- 15 message types
- Complete SDK examples

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
   └── Agent explores

LEVEL 5: Human Handoff
   └── Graceful degradation
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

## Implementation Status

| Spec | Status | Ready? |
|------|--------|--------|
| Protocol | ✅ Complete (42KB) | Yes |
| macOS | ✅ Complete (145KB) | Yes |
| Linux | ✅ Detailed | Yes |
| Windows | ⏸️ Deferred | No |

**All specs are implementation-ready with complete code examples.**

---

## Next Steps

1. Implement macOS driver (4-5 weeks)
2. Test with real applications
3. Publish as MCP server for Cline/Cursor/Windsurf
4. Implement Linux driver (6-7 weeks)

---

## Status

- [x] Protocol: Complete (with SDKs)
- [x] macOS: Complete (full code)
- [x] Linux: Detailed spec ready
- [ ] macOS implementation: Pending
- [ ] Linux implementation: Pending
- [ ] Windows: Deferred