# OSCP — Operating System Context Protocol

**Version:** 0.4.0
**Status:** Specs Complete. Ready for Implementation.

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP is a **wrapper + on-demand layer** on top of existing OS accessibility APIs. Agent requests screen state when needed; OSCP responds with semantic tree.

---

## Principle

> "Wrap existing tools. Respond on-demand. Handle errors gracefully. Agent provides meaning."

---

## Architecture

```
AGENT                                    OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (I need to see the screen)               │
 │                     │ ├─► Query OS APIs
 │                     │ ├─► Fallback chain
 │                     │ ├─► Tree analysis
 │  ◄────────────────────────────────────────│
 │  { windows, elements, confidence }          │
```

---

## Specs Complete

### Protocol Specification
**File:** `protocol/SPEC.md`
- All message types defined
- Element models specified  
- Error codes defined
- **Status: Ready for implementation**

### macOS Platform Spec
**File:** `platforms/macos/SPEC.md`
- AXUIElement integration details
- CGEvent input engine
- CDP bridge implementation
- **Time: 4-5 weeks**

### Linux Platform Spec
**File:** `platforms/linux/SPEC.md`
- AT-SPI2 integration details
- X11 fallback implementation
- /dev/uinput input engine
- **Time: 6-7 weeks**

### Windows Platform
**File:** `platforms/windows/SPEC.md`
- **Deferred until after macOS/Linux**

---

## What OSCP Wraps

| Platform | Wraps | Coverage |
|----------|-------|----------|
| **macOS** | AXUIElement | 95% |
| **Linux** | AT-SPI2 + X11 | 90-95% |

---

## Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90%)
  └── AXUIElement / AT-SPI2

LEVEL 2: CDP Bridge (Browser)
  └── Chrome DevTools Protocol

LEVEL 3: Structural Heuristics
  └── Position-based inference

LEVEL 4: Position-Only Mode
  └── Agent explores

LEVEL 5: Human Handoff
  └── Graceful degradation
```

---

## Agent SDK Example

```python
import oscp

client = oscp.connect("unix:///tmp/oscp.sock")

async def agent_task():
    # Agent requests when needed
    frame = await client.getFrame()
    
    if frame.tree_analysis.confidence == "HIGH":
        save_button = frame.find(name="Save", type="button")
        if save_button:
            await client.click(save_button.bounds)
    
    elif frame.tree_analysis.confidence == "LOW":
        # Explore candidates
        for c in frame.explore(save_button.bounds):
            result = await client.click(c.bounds)
            if result.success:
                break
```

---

## Implementation Timeline

```
WEEK 1-5:    macOS implementation
WEEK 6-12:   Linux implementation
WEEK 13+:    macOS + Linux polish + testing
```

**Total: 10-12 weeks (macOS + Linux)**

---

## Directory Structure

```
OSCP/
├── SPEC.md                      # This file
├── protocol/
│   └── SPEC.md                 # Protocol specification
├── platforms/
│   ├── macos/
│   │   └── SPEC.md             # macOS implementation
│   ├── linux/
│   │   └── SPEC.md             # Linux implementation
│   └── windows/
│       └── SPEC.md             # Deferred
├── agents/
│   └── SPEC.md                # Agent SDK guidelines
└── MEMORY/                     # Project context
```

---

## Status

- [x] Protocol specification complete
- [x] macOS platform detailed spec complete
- [x] Linux platform detailed spec complete
- [ ] macOS implementation pending
- [ ] Linux implementation pending
- [ ] Windows implementation deferred

---

*OSCP — On-demand desktop awareness for agents.*