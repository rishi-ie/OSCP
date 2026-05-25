# OSCP Core Protocol Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The OSCP protocol defines how platform drivers communicate with agent harnesses. It is transport-agnostic and can run over stdio, WebSocket, Unix domain sockets, TCP, or any bidirectional channel.

**Principle:** Deterministic. Low-latency. Error-resilient. No visual parsing.

---

## Transport

### Connection Model

```
Agent Harness  ←→  Platform Driver
     (client)       (server)

The driver is the server. The harness connects as a client.
```

### Default Ports

| Platform | Default |
|----------|---------|
| Windows | `tcp://localhost:9876` |
| macOS | `unix:///tmp/oscp.sock` |
| Linux | `unix:///tmp/oscp.sock` |

### Framing

Each message is a length-prefixed JSON frame:

```
[4-byte big-endian length][JSON payload]
```

---

## Message Types

### 1. Driver → Harness: Render Tree

Sent continuously at target frame rate (default 30fps, configurable).

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
        },
        {
          "id": "e_002",
          "type": "menu_item",
          "name": "File",
          "bounds": {"x": 0, "y": 30, "w": 50, "h": 25},
          "state": ["enabled"],
          "confidence": 0.95,
          "source": "atspi",
          "children": [
            {
              "id": "e_003",
              "type": "menu_item",
              "name": "New File",
              "bounds": {"x": 0, "y": 55, "w": 150, "h": 25}
            }
          ]
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

### Fallback Render Tree (Empty Tree Scenario)

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
      "focused": true,
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
    "avg_depth": 1,
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

### 2. Harness → Driver: Action

Sent when the agent wants to interact with the system.

```json
{
  "type": "action",
  "action_id": "act_001",
  "action": {
    "kind": "click",
    "x": 1750,
    "y": 17,
    "button": "left"
  }
}
```

### 3. Driver → Harness: Action Result

Confirms or reports failure of an action with confidence scoring.

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "source": "uia",
  "error": null
}
```

```json
{
  "type": "action_result",
  "action_id": "act_002",
  "success": false,
  "confidence": 0.2,
  "source": "heuristic",
  "error": {
    "code": "EMPTY_TREE",
    "message": "Semantic tree empty, fallback attempted",
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

### 4. Harness → Driver: Request

For explicit queries.

```json
{
  "type": "request",
  "request_id": "req_001",
  "method": "get_element_at",
  "params": {
    "x": 1750,
    "y": 17
  }
}
```

### 5. Driver → Harness: Response

Response to a request.

```json
{
  "type": "response",
  "request_id": "req_001",
  "confidence": 0.8,
  "result": {
    "element_id": "e_001",
    "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
    "source": "uia"
  }
}
```

### 6. Driver → Harness: Event

System-level events.

```json
{
  "type": "event",
  "event": {
    "kind": "window_opened",
    "window_id": "win_0x67890",
    "title": "New Window"
  }
}
```

### 7. Harness ↔ Driver: Handshake

Initial connection and capability negotiation.

```json
// Harness → Driver: Hello
{
  "type": "hello",
  "version": "0.2.0",
  "agent": "pi-agent",
  "capabilities": ["render_tree", "actions", "events", "error_handling"]
}
```

```json
// Driver → Harness: Welcome
{
  "type": "welcome",
  "version": "0.2.0",
  "driver": "oscp-linux",
  "platform": "linux",
  "semantic_api": "atspi2",
  "capabilities": ["render_tree", "actions", "events", "cdp_fallback", "heuristics"],
  "settings": {
    "frame_rate": 30
  }
}
```

---

## Tree Analysis

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `coverage_score` | float | 0.0-1.0, percentage of UI covered |
| `named_elements` | int | Elements with text labels |
| `unlabeled_elements` | int | Elements without text labels |
| `avg_depth` | float | Average tree depth |
| `confidence` | enum | HIGH, MEDIUM, LOW, NONE |

### Confidence Thresholds

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| HIGH | > 0.8 | Execute immediately |
| MEDIUM | 0.5-0.8 | Execute with monitoring |
| LOW | 0.3-0.5 | Explore first |
| NONE | < 0.2 | Human handoff |

### Fallback Triggers

| Trigger | Detection |
|---------|-----------|
| `coverage_score < 0.3` | Low confidence |
| `named_elements / total < 0.5` | Many unlabeled |
| `root_has_no_children` | Custom renderer |
| `avg_depth < 2` | Possible canvas app |

---

## Element

Represents a UI element in the semantic tree.

```json
{
  "id": "e_001",
  "type": "button",
  "name": "Save",
  "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
  "state": ["enabled", "visible"],
  "confidence": 0.95,
  "source": "uia",
  "children": []
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `type` | enum | Element type |
| `name` | string | Display text (may be empty) |
| `bounds` | object | Position and size (x, y, w, h) |
| `state` | array | States (enabled, disabled, etc.) |
| `confidence` | float | 0.0-1.0 |
| `source` | string | Data source (uia, atspi, heuristic, etc.) |
| `children` | array | Child elements (if any) |

### Element Types

| Type | Description |
|------|-------------|
| `window` | Top-level window |
| `button` | Push button |
| `menu` | Menu bar |
| `menu_item` | Menu item |
| `text_field` | Text input |
| `text_area` | Multi-line text |
| `checkbox` | Checkbox |
| `radio_button` | Radio button |
| `combo_box` | Dropdown |
| `list` | List container |
| `list_item` | List item |
| `tab` | Tab |
| `tab_group` | Tab container |
| `toolbar` | Toolbar |
| `toolbar_item` | Toolbar button/item |
| `canvas` | Custom rendered area |
| `unknown` | Unrecognized type |

---

## Error Codes

| Code | Description |
|------|-------------|
| `EMPTY_TREE` | Semantic tree empty or useless |
| `ELEMENT_NOT_FOUND` | Target element doesn't exist |
| `ELEMENT_OBSCURED` | Target behind other elements |
| `WINDOW_NOT_FOUND` | Target window doesn't exist |
| `ACTION_FAILED` | Action execution failed |
| `PERMISSION_DENIED` | OS blocked access |
| `CONNECTION_LOST` | Transport connection lost |
| `TIMEOUT` | Operation timed out |

---

## Action Types

### Click

```json
{
  "kind": "click",
  "x": 1750,
  "y": 17,
  "button": "left",
  "click_type": "single"
}
```

### Move

```json
{
  "kind": "move",
  "x": 100,
  "y": 200
}
```

### Drag

```json
{
  "kind": "drag",
  "start_x": 100,
  "start_y": 200,
  "end_x": 300,
  "end_y": 400,
  "button": "left"
}
```

### Scroll

```json
{
  "kind": "scroll",
  "delta_x": 0,
  "delta_y": -3
}
```

### Type

```json
{
  "kind": "type",
  "text": "hello world"
}
```

### Key Press

```json
{
  "kind": "key_press",
  "key": "enter"
}
```

### Key Combo

```json
{
  "kind": "key_combo",
  "key": "s",
  "modifiers": ["ctrl"]
}
```

---

## Event Types

| Event | Description |
|-------|-------------|
| `window_opened` | New window appeared |
| `window_closed` | Window removed |
| `window_focused` | Window gained focus |
| `window_unfocused` | Window lost focus |
| `element_added` | New element appeared |
| `element_removed` | Element removed |
| `element_changed` | Element properties changed |
| `tree_quality_changed` | Coverage or confidence changed |

---

## Configuration

```json
{
  "config": {
    "frame_rate": 30,
    "action_delay_ms": 0,
    "retry_on_failure": true,
    "max_retries": 3,
    "fallback_enabled": true,
    "human_handoff_threshold": 0.2
  }
}
```

- `frame_rate`: Target frames per second (1-60)
- `action_delay_ms`: Delay between action and next frame
- `retry_on_failure`: Retry failed actions
- `max_retries`: Maximum retry attempts
- `fallback_enabled`: Enable fallback hierarchy
- `human_handoff_threshold`: Confidence below which to handoff

---

## Appendix: Data Sources

| Source | Platform | Description |
|--------|----------|-------------|
| `uia` | Windows | UI Automation |
| `win32` | Windows | Win32 API |
| `atspi` | Linux | AT-SPI2 D-Bus |
| `x11` | Linux | X11 protocol |
| `axui` | macOS | AXUIElement |
| `cdp` | All | Chrome DevTools Protocol |
| `heuristic` | All | Position-based inference |
| `position` | All | Coordinates only |

---

## Appendix: Key Names

Standard key names for keyboard actions:

```
a-z, 0-9, enter, tab, escape, space, backspace, delete,
up, down, left, right,
home, end, page_up, page_down,
f1-f12,
ctrl, alt, shift, meta
```