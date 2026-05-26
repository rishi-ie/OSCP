# OSCP Agent Integration Specification

**Version:** 0.4.0
**Status:** Specs Complete

---

## Overview

OSCP provides agents with on-demand, deterministic desktop interaction. Agent requests screen state when needed; OSCP responds with semantic tree.

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
  │  (I need to see the screen)                │
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

## Agent SDK Example

### Python SDK

```python
import oscp

# Connect to OSCP service
client = oscp.connect("unix:///tmp/oscp.sock")

# Agent workflow
async def agent_task():
    # 1. Agent requests screen state
    frame = await client.getFrame()
    
    # 2. Check quality
    if frame.tree_analysis.confidence == "HIGH":
        # Find element
        save_button = frame.find(name="Save", type="button")
        if save_button and save_button.confidence > 0.8:
            # 3. Act immediately for high confidence
            result = await client.click(save_button.bounds)
            
            # 4. Verify
            verify_frame = await client.getFrame()
            if verify_frame.tree_analysis.confidence == "HIGH":
                # Continue with next task
                pass
    
    elif frame.tree_analysis.confidence == "LOW":
        # Explore candidates for low confidence
        candidates = frame.explore(save_button.bounds)
        for c in candidates:
            result = await client.click(c.bounds)
            if result.success:
                break
        else:
            # All failed, request human handoff
            await client.request_handoff()
    
    else:
        # Human handoff required
        await client.request_handoff(
            reason="Cannot identify target element",
            options=["Try exploring", "Human takes over"]
        )
```

### TypeScript SDK

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
                    const result = await client.click(element.bounds);
                    if (result.success) {
                        // Continue
                    }
                }
            }
        }
    } else if (frame.treeAnalysis.confidence === 'LOW') {
        // Explore candidates
        const candidates = frame.explore({x: 1750, y: 5});
        for (const c of candidates) {
            const result = await client.click(c.bounds);
            if (result.success) break;
        }
    }
}
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

## Error Handling

### Action Failure with Alternatives

```python
result = await client.click(element.bounds)

if not result.success:
    # Check error for alternatives
    if result.error.alternatives:
        for alt in result.error.alternatives:
            if alt.confidence > 0.3:
                retry = await client.click(bounds=alt.bounds)
                if retry.success:
                    break
```

### Tree Quality Response

```python
frame = await client.getFrame()

if frame.tree_analysis.confidence == "NONE":
    # Handle empty/custom renderer
    await client.explore_and_confirm(element)
```

---

## Human Handoff Protocol

```json
{
  "type": "handoff_request",
  "reason": "Cannot identify target element",
  "reasoning": "Custom renderer detected, no element names",
  "attempts": 3,
  "alternatives": [
    {"bounds": {"x": 1700, "y": 5}, "confidence": 0.3},
    {"bounds": {"x": 960, "y": 540}, "confidence": 0.2}
  ],
  "options": [
    "Agent explores at alternative positions",
    "Human completes this specific task",
    "Skip this task"
  ]
}
```

### Handoff Response

```python
await client.respond_handoff(
    resolution="human_completed",
    human_click={"x": 1750, "y": 5}
)
```

---

## Best Practices

### 1. Request Strategically

```python
# Good: Request when needed
frame = await client.getFrame()  # Before task
# ... perform actions ...
verify_frame = await client.getFrame()  # After action

# Bad: Unnecessary requests
for i in range(100):
    frame = await client.getFrame()  # Wasteful
```

### 2. Always Check Confidence

```python
if element.confidence > 0.8:
    await execute()  # High confidence
else:
    await explore_first()  # Lower confidence
```

### 3. Handle Failures Gracefully

```python
result = await execute(element)
if not result.success:
    for alt in result.error.alternatives:
        if alt.confidence > 0.3:
            if await client.click(alt.bounds).success:
                break
    else:
        await client.request_handoff()
```

### 4. Monitor State Changes

```python
async def execute_with_monitoring(element):
    before = await client.getFrame()
    result = await execute(element)
    after = await client.getFrame()
    
    if not state_changed(before, after):
        # Action didn't have visible effect
        return await explore_and_confirm(element)
    
    return result
```

---

## Principle

> "Agent requests when needed. OSCP responds with coordinates and types. Agent provides meaning. Handle errors gracefully."

---

## Status

- [x] Agent integration fully specified
- [x] SDK examples provided
- [x] Error handling documented
- [ ] SDK implementation pending