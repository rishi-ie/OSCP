# OSCP Architecture Specification

**Version:** 0.4.0
**Status:** Design Updated

---

## Overview

OSCP is a **wrapper + request-response layer** on top of existing OS accessibility APIs. Agent requests screen state on-demand; OSCP responds with semantic tree. No continuous streaming.

**Key Changes from v0.3:**
- ~~30fps streaming~~ → On-demand calls
- Agent controls when to see the screen
- Simpler architecture, lower overhead

---

## Architecture Pattern (All Platforms)

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS                                 │
│                                                                 │
│   Agent decides when to request:                                 │
│   {                                                             │
│     "await oscp.getFrame()"                                     │
│   }                                                            │
└─────────────────────────────────────────────────────────────────┘
                             │
                        OSCP Protocol
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
    ▼                        ▼                        ▼
┌─────────────────────────────────────────────────────────────┐
│                 OSCP PLATFORM DRIVER                         │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │   REQUEST       │  │   ERROR HANDLER │  │   INPUT    │ │
│  │   HANDLER       │  │   + FALLBACKS   │  │   ENGINE   │ │
│  └────────┬────────┘  └────────┬────────┘  └─────┬──────┘ │
│           │                    │                 │          │
│           │              ┌──────▼──────┐         │          │
│           │              │   TREE      │◄────────┘          │
│           │              │   ANALYZER  │                    │
│           │              └──────┬──────┘                    │
│           │                    │                            │
│           │    ┌──────────────┴──────────────┐            │
│           │    │                              │            │
│           │    ▼                              ▼            │
│  ┌────────┴──┴──────────────┐    ┌────────────┴─────────┐  │
│  │   PRIMARY CAPTURE       │    │   FALLBACK METHODS    │  │
│  │   (AXUIElement/AT-SPI2) │    │   (CDP/X11/Position)  │  │
│  └──────────────────────────┘    └──────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                             │
                    Native OS APIs
```

---

## How It Works

```
AGENT                                     OSCP SERVICE
 │                                          │
 │  oscp.getFrame() ──────────────────────────►│
 │                                          ├─► AXUIElement query
 │                                          ├─► Tree analysis
 │                                          ├─► Coverage check
 │                                          │   (fallback if needed)
 │  ◄────────────────────────────────────────│
 │  {                                        │
 │    "frame_id": 12345,                     │
 │    "platform": "macOS",                   │
 │    "windows": [...],                      │
 │    "tree_analysis": {...}                 │
 │  }                                        │
 │                                          │
 │  oscp.click(element.bounds) ─────────────► │
 │                                          ├─► CGEvent injection
 │  ◄────────────────────────────────────────│
 │  { success: true, confidence: 0.95 }        │
```

---

## Request Types

### Agent → OSCP: Get Frame

```json
{
  "type": "get_frame",
  "request_id": "req_001",
  "timestamp": 1716576000000
}
```

### OSCP → Agent: Frame Response

```json
{
  "type": "frame",
  "request_id": "req_001",
  "frame_id": 12345,
  "platform": "macOS",
  "latency_ms": 45,
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "confidence": 0.95,
          "source": "axuielement"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH",
    "fallback_method": null
  },
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_element_id": "e_042"
  }
}
```

### Agent → OSCP: Perform Action

```json
{
  "type": "action",
  "action_id": "act_001",
  "action": {
    "kind": "click",
    "x": 1750,
    "y": 17
  }
}
```

### OSCP → Agent: Action Result

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "error": null
}
```

---

## No Streaming = Simpler

| Aspect | Old (Streaming) | New (On-Demand) |
|--------|----------------|-----------------|
| **Data flow** | Continuous push | Request-response |
| **Frequency** | 30fps always | On-demand |
| **Agent complexity** | Handle stream | Simple call/response |
| **Server complexity** | Stream management | Request handler |
| **Resource usage** | Higher (always on) | Lower (idle when not called) |
| **Latency** | <33ms per frame | ~50-100ms per call |
| **Use case fit** | Real-time monitoring | Task-oriented agents |

---

## When Agent Requests

```
AGENT DECIDES WHEN TO CALL:

1. Before a task
   "Let me see what's on screen"
   └── oscp.getFrame()

2. After an action (verification)
   "Did the dialog appear?"
   └── oscp.getFrame()

3. When uncertain
   "I don't see the button I clicked"
   └── oscp.getFrame()

4. Periodic for long tasks
   "Track progress of file save"
   └── Periodic oscp.getFrame()

Agent keeps state in memory. OSCP responds on-demand.
```

---

## Speed Comparison

| Feature | Streaming (Old) | On-Demand (New) |
|---------|----------------|-----------------|
| Frame latency | <33ms | ~50-100ms |
| Agent overhead | Constant stream processing | Call when needed |
| Network | Continuous | Request-response |
| Battery impact | Higher | Lower |
| Suitability | Real-time games, fast UI | Task-oriented work |

---

## Fallback Hierarchy (Same)

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

## Complexity Comparison

| Component | Old (Streaming) | New (On-Demand) |
|-----------|----------------|-----------------|
| Server | Complex (stream management) | Simple (request handler) |
| Client | Complex (stream consumer) | Simple (call/response) |
| Protocol | Streaming frames | Request-response messages |
| Implementation | 4-6 weeks | 3-4 weeks |

**Reduced complexity = Faster to build.**

---

## macOS Architecture (Updated)

```
┌─────────────────────────────────────────────────────────────┐
│                 macOS Platform Driver                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              REQUEST HANDLER                         │  │
│  │                                                      │  │
│  │   getFrame() ──► AXUIElement query ──► JSON response │
│  │                                                      │  │
│  │   onDemand:                                         │  │
│  │   ├── Query focused app                              │  │
│  │   ├── Get element hierarchy                          │  │
│  │   ├── Extract bounds, names, states                  │  │
│  │   └── Analyze coverage                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                             │
│  ┌───────────────────────────┼───────────────────────────┐ │
│  │                    TREE ANALYZER                      │ │
│  │   coverage_score, named_elements, confidence           │ │
│  └───────────────────────────┬───────────────────────────┘ │
│                              │                             │
│                 ┌───────────┴───────────┐                  │
│                 ▼                       ▼                  │
│  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│  │   PRIMARY: AXUIElement  │  │   FALLBACKS:             │ │
│  │   Coverage: 90%          │  │   • CDP Bridge           │ │
│  │                          │  │   • Position-Only        │ │
│  │                          │  │   • Human Handoff        │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                INPUT ENGINE                          │ │
│  │   click(), type(), key_combo() via CGEvent          │ │
│  └──────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Linux Architecture (Updated)

```
┌─────────────────────────────────────────────────────────────┐
│                 Linux Platform Driver                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              REQUEST HANDLER                         │  │
│  │                                                      │  │
│  │   onDemand:                                         │  │
│  │   ├── Query at-spi-bus                               │  │
│  │   ├── Get element hierarchy                          │  │
│  │   ├── Fallback to X11 if needed                      │  │
│  │   └── Analyze coverage                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                             │
│  ┌───────────────────────────┼───────────────────────────┐ │
│  │                    TREE ANALYZER                      │ │
│  │   coverage_score, named_elements, confidence           │ │
│  └───────────────────────────┬───────────────────────────┘ │
│                              │                             │
│                 ┌───────────┴───────────┐                  │
│                 ▼                       ▼                  │
│  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│  │   PRIMARY: AT-SPI2        │  │   FALLBACKS:             │ │
│  │   + X11 (fallback)       │  │   • CDP Bridge           │ │
│  │   Coverage: 90%          │  │   • Heuristics           │ │
│  │                          │  │   • Human Handoff        │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                INPUT ENGINE                          │ │
│  │   click(), type() via /dev/uinput                    │ │
│  └──────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Agent SDK Example

```python
import oscp

# Connect
client = oscp.connect("unix:///tmp/oscp.sock")

# Agent working on task:
async def agent_task():
    # Before action
    frame = await client.getFrame()
    
    # Agent decides
    save_button = frame.find(name="Save", type="button")
    
    # If found with high confidence
    if save_button and save_button.confidence > 0.8:
        await client.click(save_button.bounds)
    
    # After action, verify
    new_frame = await client.getFrame()
    if new_frame.tree.confidence == "HIGH":
        # Continue
    else:
        # Fallback strategy
```

---

## Revised Time Estimates

| Component | Complexity | Time |
|-----------|------------|------|
| **Request handler** | Low | 1 week |
| **API wrapping** | Low | 2 weeks |
| **Fallback handling** | Low | 1 week |
| **Input engine** | Low | 1 week |
| **Testing** | Medium | 2 weeks |
| **Total macOS** | | **4-5 weeks** |

| Component | Complexity | Time |
|-----------|------------|------|
| **Request handler** | Medium | 1 week |
| **AT-SPI2 wrapping** | Medium | 2 weeks |
| **X11 fallback** | Low | 1 week |
| **Fallback handling** | Low | 1 week |
| **Input engine** | Medium | 2 weeks |
| **Testing** | High | 2 weeks |
| **Total Linux** | | **6-7 weeks** |

**Total: 10-12 weeks (down from 12-16)**

---

## Status

- [x] Architecture updated (no streaming)
- [x] Request-response model defined
- [x] On-demand pattern documented
- [ ] Implementation pending

---

## Summary

```
OSCP v0.4 = ON-DEMAND, NOT STREAMING

- Agent requests when needed
- OSCP responds immediately
- Simpler architecture
- Lower overhead
- Faster to build
- Still covers 85-90% of tasks
```