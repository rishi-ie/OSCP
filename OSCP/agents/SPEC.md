# OSCP Agent Integration Specification

**Version:** 0.4.0
**Status:** Design Updated

---

## Overview

OSCP provides agents with on-demand, deterministic desktop interaction. Agent requests screen state when needed; OSCP responds with semantic tree. No continuous streaming.

**Key Features:**
- On-demand request-response (not streaming)
- Deterministic (not probabilistic like VLMs)
- Error-resilient (fallback hierarchy)
- Confidence scoring on every element
- Human handoff for edge cases

---

## Request-Response Flow

```
Agent                                    OSCP Service
  │                                          │
  │  getFrame() ─────────────────────────────► │
  │  (I need to see the screen)               │
  │                                          │
  │  ◄───────────────────────────────────────│
  │  {                                        │
  │    "windows": [...],                       │
  │    "elements": [...],                      │
  │    "confidence": "HIGH"                    │
  │  }                                        │
  │                                          │
  │  click(bounds) ──────────────────────────►│
  │  (Click at x,y)                            │
  │                                          │
  │  ◄───────────────────────────────────────│
  │  { success: true }                         │
```

---

## Connection Flow

```
Agent                          OSCP Service
  │                                │
  │  Connect (Unix/TCP)           │
  │ ─────────────────────────────► │
  │                                │
  │  Send: hello                   │
  │ ─────────────────────────────► │
  │                                │
  │  Receive: welcome              │
  │ ◄───────────────────────────── │
  │                                │
  │  Request: get_frame            │
  │ ─────────────────────────────► │
  │                                │
  │  Receive: frame                │
  │ ◄───────────────────────────── │
  │                                │
  │  Send: actions                 │
  │ ─────────────────────────────► │
```

---

## Agent SDK Example

### Python

```python
import oscp

# Connect
client = oscp.connect("unix:///tmp/oscp.sock")

# Agent workflow
async def agent_task():
    # Agent decides when to request
    frame = await client.getFrame()
    
    # Check quality
    if frame.tree_analysis.confidence == "HIGH":
        # Find element
        save_button = frame.find(name="Save", type="button")
        if save_button:
            await client.click(save_button.bounds)
    
    elif frame.tree_analysis.confidence == "LOW":
        # Explore candidates
        candidates = frame.explore(position=save_button.bounds)
        for c in candidates:
            result = await client.click(c.bounds)
            if result.success:
                break
    
    else:
        # Human handoff
        await client.request_handoff("Cannot find Save button")
```

### TypeScript

```typescript
import { createOSCPClient } from 'oscp-sdk';

const client = createOSCPClient('unix:///tmp/oscp.sock');

async function agentTask() {
    // Agent controls when to request
    const frame = await client.getFrame();
    
    if (frame.treeAnalysis.confidence === 'HIGH') {
        for (const window of frame.windows) {
            for (const element of window.elements) {
                if (element.name === 'Save' && element.confidence > 0.8) {
                    client.click(element.bounds);
                }
            }
        }
    }
}
```

---

## Best Practices

### 1. Request When Needed

```python
# DON'T: Request blindly
for i in range(1000):
    frame = await client.getFrame()  # Unnecessary calls

# DO: Request strategically
frame = await client.getFrame()  # Before task
# ... perform actions ...
frame = await client.getFrame()  # After action, to verify
```

### 2. Always Check Confidence

```python
# DON'T: Assume element is clickable
await client.click(element.bounds)

# DO: Check confidence first
if element.confidence > 0.8:
    await execute()
else:
    await explore_first()
```

### 3. Handle Errors Gracefully

```python
# DO: Use fallbacks
result = await execute(element)
if not result.success:
    for alternative in result.error.alternatives:
        if alternative.confidence > 0.3:
            result = await execute(alternative)
            if result.success:
                break
```

---

## Confidence Decision Table

| Tree Confidence | Threshold | Agent Action |
|-----------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore candidates first |
| **NONE** | < 0.2 | Human handoff |

---

## Human Handoff Protocol

```json
{
  "type": "handoff_request",
  "reason": "Cannot identify Save button",
  "reasoning": "Custom renderer detected, no element names",
  "attempts": 3,
  "options": [
    "Agent explores and learns positions",
    "Human clicks and agent learns from it",
    "Skip this task"
  ]
}
```

---

## Principle

> "Agent requests when needed. OSCP responds with coordinates and types. Agent provides meaning. Handle errors gracefully."

---

## Status

- [x] Agent integration updated (on-demand model)
- [ ] SDK implementation pending