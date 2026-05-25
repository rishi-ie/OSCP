# OSCP — Operating System Context Protocol

**Version:** 0.3.0
**Status:** Architecture Finalized

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP is a **wrapper + streaming layer** on top of existing OS accessibility APIs. The hard parts are already built — OSCP adds streaming, unification, and error handling.

---

## Principle

> "Wrap existing tools. Add real-time streaming. Handle errors gracefully. Agent provides meaning."

---

## Architecture

```
AGENT HARNESS
     │
OSCP Protocol
     │
┌────────────────────────────────────────────────────────────┐
│              OSCP PLATFORM DRIVER                        │
│                                                            │
│   ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│   │  STREAMING  │  │    ERROR     │  │      INPUT      │ │
│   │   ENGINE    │  │   HANDLER    │  │     ENGINE      │ │
│   └────────┬─────┘  └──────┬──────┘  └────────┬────────┘ │
│           │                │                  │           │
│           │         ┌───────▼───────┐          │           │
│           │         │    TREE       │◄─────────┘           │
│           │         │   ANALYZER    │                      │
│           │         └───────┬───────┘                      │
│           │                 │                              │
│           │    ┌────────────┴────────────┐                 │
│           │    │                         │                 │
│           │    ▼                         ▼                 │
│           │   ▼                           ▼                 │
│   ┌───────┴──────────────────┐  ┌─────────────────────────┐│
│   │   PRIMARY CAPTURE         │  │    FALLBACK METHODS      ││
│   │   (AXUIElement/AT-SPI2)   │  │    (CDP/X11/Position)    ││
│   └────────────────────────────┘  └────────────────────────┘│
└────────────────────────────────────────────────────────────┘
     │
Native OS APIs
```

---

## What OSCP Wraps

| Platform | Wraps | Coverage |
|----------|-------|----------|
| **macOS** | AXUIElement | 95% |
| **Linux** | AT-SPI2 + X11 | 90-95% |
| **Windows** | UIAutomation | 90% |

---

## What OSCP Adds

| Feature | Existing Tools | OSCP |
|---------|----------------|------|
| **Real-time streaming** | One-shot queries | 30fps updates |
| **Unified protocol** | Per-platform APIs | Same format everywhere |
| **Error handling** | Fail silently | Fallback hierarchy |
| **Confidence scoring** | No quality metrics | Per-element confidence |
| **Human handoff** | Not supported | Escalation protocol |

---

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

---

## Confidence & Agent Decision

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore first |
| **NONE** | < 0.2 | Explore + handoff |

---

## How It Works

```python
# Agent connects to OSCP service
client = oscp.connect("unix:///tmp/oscp.sock")

# Real-time 30fps stream
async for frame in client.stream():
    if frame.tree_analysis.confidence == "HIGH":
        # Standard desktop work - full speed
        for element in frame.find("button"):
            if element.name == "Save":
                await client.click(element.bounds)
    
    elif frame.tree_analysis.confidence == "LOW":
        # Edge case - explore candidates
        candidates = frame.explore(element)
        for c in candidates:
            result = await client.click(c)
            if result.success:
                break
```

---

## Implementation Stack

| Platform | Wraps | Streaming | Input | Time |
|----------|-------|-----------|-------|------|
| **macOS** | AXUIElement (pyax) | AXObserver 30fps | CGEvent | 4-6 weeks |
| **Linux** | AT-SPI2 (dogtail) | D-Bus events 30fps | /dev/uinput | 6-8 weeks |
| **Windows** | UIAutomation (pywinauto) | UIA events 30fps | SendInput | 6-8 weeks |

**Total: 12-16 weeks for all three platforms**

---

## Update Note

Windows implementation follows macOS and Linux. Start order: macOS → Linux → Windows.

---

## Directory Structure

```
OSCP/
├── protocol/              # Protocol specification
│
├── platforms/
│   ├── macos/            # AXUIElement wrapper + streaming
│   ├── linux/            # AT-SPI2 wrapper + streaming
│   └── windows/          # UIA wrapper + streaming
│
└── agents/               # Agent SDK guidelines
```

---

## Status

- [x] Architecture finalized
- [x] Existing tools identified
- [x] Per-platform approach defined
- [ ] Implementation pending

---

*OSCP — First-class access for agents.*