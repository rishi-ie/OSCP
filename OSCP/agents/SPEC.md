# OSCP Agent Integration Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

OSCP provides agents with deterministic, pixel-perfect desktop interaction via semantic tree extraction and hardware-level actuation. Agent provides the meaning; protocol provides coordinates.

**Key Features:**
- Deterministic (not probabilistic like VLMs)
- Error-resilient (fallback hierarchy)
- Confidence scoring on every element and action
- Human handoff for edge cases

---

## Connection Flow

```
Agent                          OSCP Driver
  │                                │
  │  Connect (TCP/Unix/stdio)      │
  │ ─────────────────────────────► │
  │                                │
  │  Send: hello                   │
  │ ─────────────────────────────► │
  │                                │
  │  Receive: welcome              │
  │ ◄───────────────────────────── │
  │                                │
  │  Receive: render tree stream   │
  │ ◄───────────────────────────── │
  │                                │
  │  Send: actions                 │
  │ ─────────────────────────────► │
```

---

## Data Received by Agent

### Standard Render Tree

```json
{
  "type": "render_tree",
  "frame_id": 12345,
  "timestamp": 1716576000000,
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "state": ["enabled", "visible"],
          "confidence": 0.95,
          "source": "atspi"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH"
  },
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_element_id": "e_042"
  }
}
```

### Fallback Render Tree (Empty Tree)

```json
{
  "type": "render_tree",
  "frame_id": 12346,
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x500001",
      "title": "CustomGame",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "elements": [],
      "fallback_active": true,
      "fallback_method": "position_only",
      "fallback_reason": "custom_renderer_detected"
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.05,
    "named_elements": 0,
    "unlabeled_elements": 1,
    "confidence": "NONE",
    "recommended_action": "human_handoff"
  },
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_element_id": null
  }
}
```

---

## Confidence-Based Decision Making

### Tree Quality Thresholds

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | > 0.8 | Execute immediately |
| **MEDIUM** | 0.5-0.8 | Execute with monitoring |
| **LOW** | 0.3-0.5 | Explore first, then execute |
| **NONE** | < 0.2 | Human handoff |

### Agent Decision Logic

```python
async def click_element(element):
    if element.confidence > 0.8:
        # HIGH confidence - execute immediately
        return await execute_click(element.bounds)
    
    elif element.confidence > 0.5:
        # MEDIUM confidence - execute with monitoring
        before_state = capture_state()
        result = await execute_click(element.bounds)
        after_state = capture_state()
        
        if state_changed(before_state, after_state):
            return result  # Success
        else:
            return await explore_and_retry(element)  # Failed, explore
    
    elif element.confidence > 0.2:
        # LOW confidence - explore first
        candidates = find_similar_elements(element)
        for candidate in candidates:
            result = await try_element(candidate)
            if success(result):
                return result
        return human_handoff("All candidates failed")
    
    else:
        # NONE confidence - human handoff
        return await human_handoff(
            reason="No confidence in element location",
            alternatives=element.alternatives
        )
```

---

## Actions Sent by Agent

### Click

```json
{
  "type": "action",
  "action_id": "act_001",
  "action": {
    "kind": "click",
    "x": 1750,
    "y": 17,
    "button": "left",
    "click_type": "single"
  }
}
```

### Key Combo

```json
{
  "type": "action",
  "action_id": "act_002",
  "action": {
    "kind": "key_combo",
    "key": "s",
    "modifiers": ["ctrl"]
  }
}
```

### Type

```json
{
  "type": "action",
  "action_id": "act_003",
  "action": {
    "kind": "type",
    "text": "hello world"
  }
}
```

---

## Action Results

### Success

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "source": "atspi",
  "error": null
}
```

### Failure with Alternatives

```json
{
  "type": "action_result",
  "action_id": "act_002",
  "success": false,
  "confidence": 0.2,
  "source": "heuristic",
  "error": {
    "code": "EMPTY_TREE",
    "message": "Semantic tree empty",
    "reasoning": "Custom renderer detected, no element names",
    "alternatives": [
      {"bounds": {"x": 1700, "y": 5}, "confidence": 0.3},
      {"bounds": {"x": 1750, "y": 5}, "confidence": 0.2},
      {"bounds": {"x": 1800, "y": 5}, "confidence": 0.1}
    ],
    "recommended_action": "explore_and_confirm"
  }
}
```

---

## Error Handling

### Tree Analysis Response

Agent receives `tree_analysis` with every frame:

```json
{
  "tree_analysis": {
    "coverage_score": 0.05,
    "named_elements": 0,
    "unlabeled_elements": 1,
    "avg_depth": 1,
    "confidence": "NONE",
    "recommended_action": "human_handoff"
  }
}
```

### Agent Response to Poor Tree Quality

```python
async def handle_low_quality(tree_analysis):
    if tree_analysis.confidence == "NONE":
        # Custom renderer or protected content
        if tree_analysis.fallback_method == "position_only":
            # Agent must explore and learn
            await enter_exploration_mode()
        else:
            # Escalate to human
            await human_handoff(
                reason="Cannot access UI elements",
                alternatives=[]
            )
    
    elif tree_analysis.confidence == "LOW":
        # Many unlabeled elements
        # Agent uses heuristics and position
        await.enter_heuristic_mode()
```

---

## Agent SDKs

### Python SDK

```python
import oscp

# Connect
client = oscp.connect("unix:///tmp/oscp.sock")

# Stream with quality checking
async for tree in client.stream():
    # Check tree quality
    if tree.tree_analysis.confidence == "HIGH":
        # Safe to execute immediately
        for window in tree.windows:
            for element in window.elements:
                if element.name == "Save" and element.confidence > 0.8:
                    await client.click(element.bounds)
    
    elif tree.tree_analysis.confidence == "LOW":
        # Use heuristics
        await client.explore_and_confirm(element)
    
    else:
        # Human handoff
        await client.human_handoff("Cannot identify target")
```

### TypeScript SDK

```typescript
import { createOSCPClient } from 'oscp-sdk';

const client = createOSCPClient('unix:///tmp/oscp.sock');

client.on('render_tree', (tree) => {
    if (tree.tree_analysis.confidence === 'HIGH') {
        for (const window of tree.windows) {
            for (const element of window.elements) {
                if (element.name === 'Save' && element.confidence > 0.8) {
                    client.click(element.bounds);
                }
            }
        }
    }
});
```

---

## Best Practices

### 1. Always Check Confidence

```python
# Don't assume element is clickable
if element.confidence > 0.8:
    await execute()
else:
    await explore_first()
```

### 2. Use Alternatives on Failure

```python
result = await execute(element)
if not result.success:
    for alternative in result.error.alternatives:
        if alternative.confidence > 0.3:
            result = await execute(alternative)
            if result.success:
                break
```

### 3. Learn from Failures

```python
async def on_action_failure(result):
    if result.source == "heuristic":
        # Record what didn't work
        await record_failed_position(result.target)
        # Try human handoff if multiple failures
        if failure_count > 3:
            await human_handoff()
```

### 4. Monitor State Changes

```python
async def execute_with_monitoring(element):
    before = capture_state()
    result = await execute(element)
    after = capture_state()
    
    if not state_changed(before, after):
        # Action didn't have visible effect
        # Consider alternative or explore
        return await explore_and_confirm(element)
    
    return result
```

---

## Human Handoff Protocol

```json
{
  "type": "handoff_request",
  "reason": "Cannot identify Save button",
  "reasoning": "Custom renderer detected, no element names",
  "attempts": 3,
  "options": [
    "Click at position (1750, 5) based on toolbar pattern",
    "Click at position (960, 540) center of window",
    "Skip this step",
    "Click manually and I'll learn from it"
  ]
}
```

---

## Principle

> "Intercept the semantic tree. Deliver coordinates. Agent provides meaning. Handle errors gracefully."

Agent receives deterministic coordinates with confidence. Agent supplies semantics. Fallback hierarchy ensures agent never gets stuck.

---

## Status

🚧 Agent SDKs under development.