# OSCP Core Protocol Specification

**Version:** 0.4.0
**Status:** Design Updated

---

## Overview

OSCP protocol defines request-response communication between agent harnesses and platform drivers. Agent requests screen state on-demand; OSCP responds with semantic tree.

**Principle:** Request-response. Deterministic. Low-latency. Error-resilient.

---

## Transport

### Connection Model

```
Agent Harness  ←→  Platform Driver
     (client)       (server)
```

### Default Ports

| Platform | Default |
|----------|---------|
| macOS | `unix:///tmp/oscp.sock` |
| Linux | `unix:///tmp/oscp.sock` |
| Windows | `tcp://localhost:9876` |

### Framing

Each message is a JSON frame with newline delimiter:

```
{"type": "...", ...}\n
```

---

## Message Types

### 1. Agent → OSCP: Get Frame

```json
{
  "type": "get_frame",
  "request_id": "req_001",
  "timestamp": 1716576000000
}
```

### 2. OSCP → Agent: Frame Response

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
      "pid": 1234,
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "description": "",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "states": ["enabled", "visible"],
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

### 3. Agent → OSCP: Click Action

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

### 4. OSCP → Agent: Action Result

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "source": "axuielement",
  "error": null
}
```

### 5. Agent → OSCP: Type Action

```json
{
  "type": "action",
  "action_id": "act_002",
  "action": {
    "kind": "type",
    "text": "hello world"
  }
}
```

### 6. Agent → OSCP: Key Combo Action

```json
{
  "type": "action",
  "action_id": "act_003",
  "action": {
    "kind": "key_combo",
    "key": "s",
    "modifiers": ["ctrl"]
  }
}
```

### 7. OSCP → Agent: Error Response

```json
{
  "type": "error",
  "request_id": "req_001",
  "code": "EMPTY_TREE",
  "message": "Semantic tree empty or low coverage",
  "reasoning": "SDL renderer detected, no AT-SPI support",
  "fallback_method": "position_only",
  "recommended_action": "explore_or_handoff"
}
```

---

## Tree Analysis

### Confidence Levels

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | coverage > 0.8, named > 50% | Execute immediately |
| **MEDIUM** | coverage 0.5-0.8 | Execute with monitoring |
| **LOW** | coverage 0.3-0.5 | Explore candidates first |
| **NONE** | coverage < 0.3 | Explore + handoff |

### Coverage Score

```json
{
  "coverage_score": 0.9,
  "named_elements": 150,
  "unlabeled_elements": 12
}
```

Coverage score = area covered by named elements / total window area.

---

## Element Source

```json
{
  "source": "axuielement"
}
```

| Source | Description |
|--------|-------------|
| `axuielement` | macOS AXUIElement API |
| `atspi` | Linux AT-SPI2 API |
| `uia` | Windows UIAutomation API |
| `x11` | X11 fallback |
| `cdp` | Chrome DevTools Protocol |
| `position` | Position-only mode |
| `human` | Human handoff required |

---

## Fallback Frame

When tree quality is low:

```json
{
  "type": "frame",
  "request_id": "req_002",
  "platform": "linux",
  "windows": [
    {
      "id": "win_game",
      "title": "Game",
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
    "confidence": "NONE",
    "fallback_method": "position_only",
    "recommended_action": "explore_or_handoff"
  }
}
```

---

## Human Handoff Protocol

When all fallbacks fail:

```json
{
  "type": "handoff_request",
  "request_id": "req_003",
  "reason": "Cannot identify target element",
  "reasoning": "Custom renderer detected, no element names",
  "attempts": 3,
  "alternatives": [
    {"bounds": {"x": 1700, "y": 5}, "confidence": 0.3},
    {"bounds": {"x": 960, "y": 540}, "confidence": 0.2}
  ],
  "options": [
    "Agent explores and learns positions",
    "Human clicks and agent learns from it",
    "Skip this task"
  ]
}
```

---

## Latency Expectations

| Platform | Typical Latency |
|----------|-----------------|
| macOS | 20-50ms |
| Linux | 50-100ms |
| Windows | 30-80ms |

Latency measured from `get_frame` request to frame response.

---

## Capabilities

### Agent → OSCP

| Capability | Description |
|------------|-------------|
| `get_frame` | Request current screen state |
| `click` | Mouse click at coordinates |
| `type` | Type text |
| `key_combo` | Key combination (Ctrl+S, etc.) |
| `drag` | Mouse drag |
| `scroll` | Scroll action |

### OSCP → Agent

| Capability | Description |
|------------|-------------|
| `frame` | Semantic tree of current screen |
| `action_result` | Success/failure of action |
| `error` | Error with fallback info |
| `handoff_request` | Human intervention required |

---

## Status

- [x] Protocol updated (on-demand, not streaming)
- [x] Request-response model defined
- [ ] Implementation pending