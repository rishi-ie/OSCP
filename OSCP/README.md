# OSCP — Operating System Context Protocol

**Version:** 0.4.0
**Status:** Design Updated

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP is a **wrapper + on-demand layer** on top of existing OS accessibility APIs. Agent requests screen state when needed; OSCP responds with semantic tree. No continuous streaming.

**Key Change:** No 30fps streaming. Agent controls when to see the screen.

---

## Principle

> "Wrap existing tools. Respond on-demand. Handle errors gracefully. Agent provides meaning."

---

## Architecture

```
AGENT                                    OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │                                           ├─► Query APIs
 │                                           ├─► Analyze tree
 │  ◄────────────────────────────────────────│
 │  { windows, elements, confidence }          │
 │                                           │
 │  oscp.click(bounds) ─────────────────────►│
 │                                           ├─► Input injection
 │  ◄────────────────────────────────────────│
 │  { success: true }                        │
```

---

## No Streaming = On-Demand

| Aspect | Old | New |
|--------|-----|-----|
| **Pattern** | 30fps continuous push | Request-response |
| **Agent control** | Passive (receives stream) | Active (requests when needed) |
| **Resource usage** | Higher | Lower |
| **Latency** | <33ms | ~50-100ms |
| **Complexity** | Higher | Lower |

---

## What OSCP Wraps

| Platform | Wraps | Coverage |
|----------|-------|----------|
| **macOS** | AXUIElement | 95% |
| **Linux** | AT-SPI2 + X11 | 90-95% |
| **Windows** | UIAutomation | 90% |

---

## What OSCP Adds

| Feature | Description |
|---------|-------------|
| **On-demand capture** | Agent requests, OSCP responds |
| **Unified protocol** | Same JSON format on all platforms |
| **Error handling** | 5-level fallback hierarchy |
| **Confidence scoring** | Per-element and per-tree confidence |
| **Input engine** | Hardware-level actuation |

---

## How It Works

```python
# Agent requests when needed
frame = await oscp.getFrame()

# Agent decides
save_button = frame.find(name="Save", type="button")

# Act
if save_button and save_button.confidence > 0.8:
    await oscp.click(save_button.bounds)

# Verify
verify_frame = await oscp.getFrame()
```

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

## Confidence Decision Table

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore first |
| **NONE** | < 0.2 | Explore + handoff |

---

## Implementation Stack

| Platform | Wraps | Input | Time |
|----------|-------|-------|------|
| **macOS** | AXUIElement | CGEvent | 4-5 weeks |
| **Linux** | AT-SPI2 + X11 | /dev/uinput | 6-7 weeks |
| **Windows** | UIAutomation | SendInput | 6-7 weeks |

**Total: 10-12 weeks**

---

## Status

- [x] Architecture updated (on-demand, not streaming)
- [x] Request-response model defined
- [x] Specifications updated
- [ ] Implementation pending

---

*OSCP — On-demand desktop awareness for agents.*