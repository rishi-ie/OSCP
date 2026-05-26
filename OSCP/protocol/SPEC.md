# OSCP Core Protocol Specification

**Version:** 0.4.0
**Status:** Ready for Implementation

---

## Overview

OSCP is a request-response protocol for agent-native OS interaction. Agent requests screen state on-demand; OSCP responds with semantic tree. No streaming.

**Transport:** Unix domain socket (macOS/Linux) or TCP (Windows)
**Format:** JSON over newline-delimited frames

---

## Connection

### Unix Domain Socket (macOS/Linux)

```
Endpoint: unix:///tmp/oscp.sock
```

### TCP (Windows)

```
Endpoint: tcp://localhost:9876
```

### Connection Flow

```
Agent                              OSCP Service
  │                                    │
  │  Connect (socket)                   │
  │ ─────────────────────────────────► │
  │                                    │
  │  Send: hello                       │
  │  {"type": "hello", "version": "0.4"}│
  │ ─────────────────────────────────► │
  │                                    │
  │  Receive: welcome                  │
  │  {"type": "welcome", "version": "0.4",│
  │   "capabilities": [...]}          │
  │ ◄───────────────────────────────── │
  │                                    │
  │  [Connected]                       │
```

### Hello Message

```json
// Agent → OSCP
{
  "type": "hello",
  "version": "0.4"
}
```

### Welcome Message

```json
// OSCP → Agent
{
  "type": "welcome",
  "version": "0.4",
  "platform": "macOS",
  "capabilities": [
    "get_frame",
    "click",
    "type",
    "key_combo",
    "scroll",
    "drag",
    "mouse_position"
  ],
  "input_methods": ["cg_event"]
}
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

---

### 2. OSCP → Agent: Frame Response

```json
{
  "type": "frame",
  "request_id": "req_001",
  "frame_id": 12345,
  "platform": "macOS",
  "latency_ms": 45,
  "timestamp": 1716576000045,
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "pid": 1234,
      "app": "com.microsoft.VSCode",
      "bounds": {
        "x": 0,
        "y": 0,
        "w": 1920,
        "h": 1080
      },
      "position": {
        "x": 100,
        "y": 50
      },
      "focused": true,
      "minimized": false,
      "elements": [
        {
          "id": "e_001",
          "role": "button",
          "subrole": "push_button",
          "name": "Save",
          "description": "Save the current file",
          "value": "",
          "bounds": {
            "x": 1750,
            "y": 5,
            "w": 80,
            "h": 25
          },
          "states": [
            "enabled",
            "visible",
            "actionable"
          ],
          "attributes": {
            "default_button": false,
            "focused": false
          },
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
    "total_elements": 162,
    "avg_depth": 4.2,
    "confidence": "HIGH",
    "fallback_method": null,
    "recommended_action": "execute"
  },
  "mouse": {
    "x": 540,
    "y": 320,
    "button_state": "none",
    "hovered_element_id": "e_042"
  },
  "keyboard": {
    "modifiers": []
  }
}
```

---

### 3. Agent → OSCP: Click Action

```json
{
  "type": "action",
  "action_id": "act_001",
  "timestamp": 1716576000050,
  "action": {
    "kind": "click",
    "x": 1750,
    "y": 17,
    "button": "left",
    "click_type": "single",
    "element_id": "e_001"
  }
}
```

### Click Types

| `click_type` | Description |
|---------------|-------------|
| `single` | Single click |
| `double` | Double click |
| `triple` | Triple click |
| `right` | Right click |

---

### 4. Agent → OSCP: Type Action

```json
{
  "type": "action",
  "action_id": "act_002",
  "timestamp": 1716576000100,
  "action": {
    "kind": "type",
    "text": "hello world",
    "typing_delay_ms": 50
  }
}
```

---

### 5. Agent → OSCP: Key Combo Action

```json
{
  "type": "action",
  "action_id": "act_003",
  "timestamp": 1716576000150,
  "action": {
    "kind": "key_combo",
    "key": "s",
    "modifiers": ["ctrl"]
  }
}
```

### Key Constants

```json
// Modifier keys
"modifiers": [
  "ctrl",    // Control
  "alt",     // Option/Alt
  "shift",   // Shift
  "cmd",     // Command (macOS) / Win (Windows) / Super (Linux)
  "fn"       // Function key
]

// Common keys
"key": "s"           // Character key
"key": "return"      // Enter
"key": "escape"      // Esc
"key": "tab"         // Tab
"key": "space"       // Space
"key": "delete"      // Backspace/Delete
"key": "forward_delete"  // Forward Delete
"key": "up"          // Arrow keys
"key": "down"
"key": "left"
"key": "right"
"key": "f1"          // Function keys
"key": "home"        // Navigation
"key": "end"
"key": "page_up"
"key": "page_down"
```

### Common Key Combos

```json
// Ctrl+S (Save)
{"kind": "key_combo", "key": "s", "modifiers": ["ctrl"]}

// Cmd+S (macOS Save)
{"kind": "key_combo", "key": "s", "modifiers": ["cmd"]}

// Ctrl+Z (Undo)
{"kind": "key_combo", "key": "z", "modifiers": ["ctrl"]}

// Alt+Tab (Switch window)
{"kind": "key_combo", "key": "tab", "modifiers": ["alt"]}

// Ctrl+Shift+T (Reopen closed tab)
{"kind": "key_combo", "key": "t", "modifiers": ["ctrl", "shift"]}
```

---

### 6. Agent → OSCP: Scroll Action

```json
{
  "type": "action",
  "action_id": "act_004",
  "timestamp": 1716576000200,
  "action": {
    "kind": "scroll",
    "x": 540,
    "y": 320,
    "delta_x": 0,
    "delta_y": -3,
    "scroll_type": "precise"
  }
}
```

### Scroll Types

| `scroll_type` | Description |
|----------------|-------------|
| `precise` | Smooth scroll (pixel units) |
| `line` | Line-based scroll |
| `page` | Page-up/page-down |

---

### 7. Agent → OSCP: Drag Action

```json
{
  "type": "action",
  "action_id": "act_005",
  "timestamp": 1716576000250,
  "action": {
    "kind": "drag",
    "start_x": 100,
    "start_y": 200,
    "end_x": 500,
    "end_y": 200,
    "button": "left",
    "duration_ms": 500
  }
}
```

---

### 8. Agent → OSCP: Move Mouse Action

```json
{
  "type": "action",
  "action_id": "act_006",
  "timestamp": 1716576000300,
  "action": {
    "kind": "move",
    "x": 960,
    "y": 540
  }
}
```

---

### 9. OSCP → Agent: Action Success

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "timestamp": 1716576000051,
  "latency_ms": 1,
  "confidence": 0.95,
  "source": "axuielement",
  "target": {
    "element_id": "e_001",
    "element_name": "Save",
    "element_role": "button"
  },
  "error": null
}
```

---

### 10. OSCP → Agent: Action Failure

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": false,
  "timestamp": 1716576000051,
  "confidence": 0.2,
  "error": {
    "code": "ACTION_FAILED",
    "message": "Action did not produce expected result",
    "reasoning": "Element may have moved or changed state",
    "alternatives": [
      {
        "x": 1730,
        "y": 5,
        "confidence": 0.3,
        "element_name": "Save (alt)"
      },
      {
        "x": 1750,
        "y": 7,
        "confidence": 0.25,
        "element_name": "Toolbar area"
      }
    ]
  }
}
```

---

### 11. OSCP → Agent: Error

```json
{
  "type": "error",
  "request_id": "req_001",
  "code": "EMPTY_TREE",
  "message": "Semantic tree empty or insufficient coverage",
  "timestamp": 1716576000046,
  "details": {
    "coverage_score": 0.05,
    "reasoning": "Custom renderer detected, no AT-SPI support",
    "fallback_method": "position_only"
  },
  "recommended_action": "explore_or_handoff"
}
```

---

### 12. OSCP → Agent: Human Handoff Request

```json
{
  "type": "handoff_request",
  "request_id": "handoff_001",
  "timestamp": 1716576000050,
  "reason": "Cannot identify target element",
  "reasoning": "Custom renderer detected, no element names available",
  "attempts": 3,
  "failed_attempts": [
    {"x": 1700, "y": 5, "success": false, "reason": "no_reaction"},
    {"x": 1750, "y": 5, "success": false, "reason": "no_reaction"},
    {"x": 1800, "y": 5, "success": false, "reason": "no_reaction"}
  ],
  "window": {
    "id": "win_game",
    "title": "Game",
    "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080}
  },
  "alternatives": [
    {
      "bounds": {"x": 1700, "y": 0, "w": 100, "h": 50},
      "confidence": 0.3,
      "strategy": "Assuming toolbar at top"
    },
    {
      "bounds": {"x": 960, "y": 540, "w": 100, "h": 50},
      "confidence": 0.2,
      "strategy": "Assuming center click"
    }
  ],
  "options": [
    "Agent explores at provided alternative positions",
    "Human clicks and agent learns from it for future",
    "Human completes this specific task",
    "Skip this task entirely"
  ]
}
```

---

### 13. Agent → OSCP: Handoff Response

```json
{
  "type": "handoff_response",
  "response_to": "handoff_001",
  "resolution": "human_completed",
  "human_click": {
    "x": 1750,
    "y": 5
  }
}
```

### Handoff Resolutions

```json
"resolution": "agent_continue"     // Agent will try alternatives
"resolution": "human_completed"     // Human completed task
"resolution": "skip_task"          // Task skipped
"resolution": "learn_from_human"    // Agent learns position
```

---

### 14. Agent → OSCP: Mouse Position

```json
{
  "type": "mouse_position",
  "request_id": "req_002"
}
```

### OSCP → Agent: Mouse Position Response

```json
{
  "type": "mouse_position_result",
  "request_id": "req_002",
  "x": 540,
  "y": 320,
  "button_state": "none",
  "hovered_element_id": "e_042"
}
```

---

## Element Reference

### Element Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique element ID within frame |
| `role` | string | ARIA role or AX role |
| `subrole` | string | More specific role |
| `name` | string | Accessible name |
| `description` | string | Accessible description |
| `value` | string | Current value |
| `bounds` | object | {x, y, w, h} |
| `states` | array | Active states |
| `attributes` | object | Additional attributes |
| `confidence` | float | 0.0-1.0 confidence |
| `source` | string | API source |

### Role Values

```json
// Standard roles
"role": "button"
"role": "check_box"
"role": "text_field"
"role": "text_area"
"role": "menu"
"role": "menu_item"
"role": "menu_bar"
"role": "link"
"role": "tab"
"role": "tabular_cell"
"role": "row"
"role": "column"
"role": "toolbar"
"role": "group"
"role": "window"
"role": "dialog"
"role": "alert"
"role": "scroll_bar"
"role": "slider"
"role": "combo_box"
"role": "list"
"role": "list_item"
"role": "tree"
"role": "tree_item"
"role": "static_text"
"role": "image"
"role": "note"
"role": "unknown"
```

### State Values

```json
"states": [
  "enabled",        // Element is enabled
  "disabled",       // Element is disabled
  "visible",        // Element is visible
  "invisible",      // Element is hidden
  "focused",        // Element has focus
  "selected",       // Element is selected
  "checked",        // Checkbox/radio is checked
  "unchecked",      // Checkbox/radio is unchecked
  "indeterminate",  // Checkbox is indeterminate
  "pressed",       // Button is pressed
  "expanded",       // Expandable is expanded
  "collapsed",      // Expandable is collapsed
  "actionable",     // Element can be interacted with
  "updating"        // Element is updating/loading
]
```

### Source Values

```json
"source": "axuielement"  // macOS AXUIElement
"source": "atspi"        // Linux AT-SPI2
"source": "uia"          // Windows UIAutomation
"source": "x11"          // X11 fallback
"source": "x11_window"   // X11 window info only
"source": "cdp"          // Chrome DevTools Protocol
"source": "heuristic"    // Position heuristics
"source": "position"     // Position-only mode
"source": "unknown"      // Unknown source
```

---

## Error Codes

| Code | Description |
|------|-------------|
| `PERMISSION_DENIED` | Accessibility permissions not granted |
| `ELEMENT_NOT_FOUND` | Target element not found |
| `ELEMENT_DISABLED` | Target element is disabled |
| `ELEMENT_MOVED` | Target element moved since last frame |
| `ACTION_FAILED` | Action did not produce expected result |
| `EMPTY_TREE` | Semantic tree is empty |
| `LOW_COVERAGE` | Tree coverage below threshold |
| `TIMEOUT` | Operation timed out |
| `CONNECTION_LOST` | Socket connection lost |
| `INVALID_REQUEST` | Malformed request JSON |
| `UNSUPPORTED_ACTION` | Action type not supported |
| `PLATFORM_ERROR` | Platform-specific error |

---

## Confidence Thresholds

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| **HIGH** | coverage > 0.8, named ratio > 0.5 | Execute immediately |
| **MEDIUM** | coverage 0.5-0.8 | Execute with monitoring |
| **LOW** | coverage 0.3-0.5 | Explore candidates first |
| **NONE** | coverage < 0.3 | Human handoff |

---

## Latency Expectations

| Platform | Typical Latency |
|----------|-----------------|
| macOS | 20-50ms |
| Linux | 50-100ms |
| Windows | 30-80ms |

Latency = time from request to response completion.

---

## Request ID Format

```
Format: {type}_{timestamp}_{random}

Examples:
- req_1716576000000_abc123
- act_00000001_xyz789
- handoff_00000001_def456
```

Request IDs must be unique per session.

---

## Framing

### Frame Format

```
{JSON message}\n
```

Each message is a single-line JSON object followed by newline.

### Message Boundaries

No framing headers needed. Each newline-terminated JSON is one complete message.

### Request-Response Matching

```python
# Agent sends request with request_id
{"type": "action", "action_id": "act_001", ...}

# OSCP responds with same action_id
{"type": "action_result", "action_id": "act_001", ...}
```

---

## Protocol State Machine

```
┌─────────────┐
│  CONNECTED  │
└──────┬──────┘
       │ hello
       ▼
┌──────┴──────┐
│ WELCOMING   │
└──────┬──────┘
       │ welcome received
       ▼
┌──────┴──────┐
│   READY    │◄──────────────────┐
│            │                   │
│ get_frame  │                   │
│    │       │                   │
│    ▼       │                   │
│ FRAME_SENT │                   │
│    │       │                   │
│    ▼       │                   │
│ get_frame returns               │
│            │                   │
│ action     │                   │
│    │       │                   │
│    ▼       │                   │
│ ACTION_EXECUTED               │
│    │       │                   │
│    ▼       │                   │
│ action_result returns          │
│            │                   │
│ handoff    │                   │
│    │       │                   │
│    ▼       │                   │
│ HANDOVER_IN_PROGRESS          │
│    │       │                   │
│    ▼       │                   │
│ handoff_response returns      │
│            │                   │
│    └───────┴───────────────────┘
```

---

## Keep-Alive

### Ping

```json
// Agent → OSCP
{"type": "ping", "timestamp": 1716576000000}
```

### Pong

```json
// OSCP → Agent
{"type": "pong", "timestamp": 1716576000000, "latency_ms": 1}
```

---

## Disconnect

```json
// Agent → OSCP
{"type": "disconnect"}

// OSCP → Agent
{"type": "goodbye"}
```

---

## Implementation Notes

### Connection Timeout
- Connect timeout: 5 seconds
- If welcome not received, disconnect and retry

### Request Timeout
- Default request timeout: 30 seconds
- After timeout, return error with code `TIMEOUT`

### Reconnection
- Agent should implement exponential backoff
- Max reconnection attempts: 5
- Backoff: 1s, 2s, 4s, 8s, 16s

---

## Status

- [x] Protocol fully specified
- [x] Message types defined
- [x] Error codes defined
- [x] Framing specified
- [ ] Implementation pending