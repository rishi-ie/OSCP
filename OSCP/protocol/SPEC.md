# OSCP Core Protocol Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The OSCP protocol defines how platform drivers communicate with agent harnesses. It is transport-agnostic and can run over stdio, WebSocket, Unix domain sockets, TCP, or any bidirectional channel.

**Principle:** Intercept the compositor. Decode the render tree. Agent provides the meaning.

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
  "windows": [
    {
      "id": "win_0x12345",
      "title": "Visual Studio Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "focused": true,
      "ops": [
        {
          "id": "op_001",
          "bounds": {"x": 10, "y": 10, "w": 100, "h": 30},
          "z": 1,
          "texture_id": "0xAA01"
        },
        {
          "id": "op_002",
          "bounds": {"x": 120, "y": 10, "w": 80, "h": 30},
          "z": 2,
          "texture_id": "0xAA02"
        }
      ]
    }
  ],
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_op_id": "op_042"
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
    "x": 50,
    "y": 25,
    "button": "left"
  }
}
```

### 3. Driver → Harness: Action Result

Confirms or reports failure of an action.

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "error": null
}
```

```json
{
  "type": "action_result",
  "action_id": "act_002",
  "success": false,
  "error": "Window not focused"
}
```

### 4. Harness → Driver: Request

For explicit queries (e.g., get render op at position).

```json
{
  "type": "request",
  "request_id": "req_001",
  "method": "get_op_at",
  "params": {
    "x": 100,
    "y": 200
  }
}
```

### 5. Driver → Harness: Response

Response to a request.

```json
{
  "type": "response",
  "request_id": "req_001",
  "result": {
    "op_id": "op_042",
    "window_id": "win_0x12345",
    "bounds": {"x": 90, "y": 190, "w": 50, "h": 20}
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

```json
{
  "type": "event",
  "event": {
    "kind": "window_focused",
    "window_id": "win_0x12345"
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
  "capabilities": ["render_tree", "actions", "events"]
}
```

```json
// Driver → Harness: Welcome
{
  "type": "welcome",
  "version": "0.2.0",
  "driver": "oscp-windows",
  "platform": "windows",
  "compositor": "dwm",
  "capabilities": ["render_tree", "actions", "events"],
  "settings": {
    "frame_rate": 30
  }
}
```

---

## Render Tree

The render tree represents the full compositor scene — all windows and their render operations.

### Structure

```json
{
  "render_tree": {
    "windows": [...],
    "mouse": {...}
  }
}
```

### Window

Represents a visible window in the compositor scene.

```json
{
  "id": "win_0x12345",
  "title": "Visual Studio Code",
  "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
  "position": {"x": 100, "y": 50},
  "focused": true,
  "ops": [...]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique window identifier |
| `title` | string | Window title |
| `bounds` | object | Window size (w, h) |
| `position` | object | Window position (x, y) |
| `focused` | boolean | Is window focused |
| `ops` | array | Render operations in this window |

### Render Operation

A render operation represents a single draw call from the compositor.

```json
{
  "id": "op_001",
  "bounds": {"x": 10, "y": 10, "w": 100, "h": 30},
  "z": 1,
  "texture_id": "0xAA01",
  "clip_bounds": {"x": 15, "y": 15, "w": 90, "h": 20}
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier within frame |
| `bounds` | object | Destination rectangle (x, y, w, h) |
| `z` | number | Z-index (render order, higher = on top) |
| `texture_id` | string | Texture identifier from GPU |
| `clip_bounds` | object | Optional clipping rectangle |

**Coordinates:**
- All coordinates are in physical pixels
- Origin is top-left of primary monitor
- Bounds are relative to the window

---

## Action Types

### Click

```json
{
  "kind": "click",
  "x": 100,
  "y": 200,
  "button": "left",
  "click_type": "single"
}
```

- `button`: `left`, `right`, `middle`
- `click_type`: `single`, `double`, `triple`

### Move

Move cursor to absolute position.

```json
{
  "kind": "move",
  "x": 100,
  "y": 200
}
```

### Drag

Drag from start to end position.

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

- Positive values: scroll down/right
- Negative values: scroll up/left

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
| `window_moved` | Window position changed |
| `window_resized` | Window size changed |
| `key_pressed` | Keyboard input received |
| `mouse_moved` | Mouse moved |
| `mouse_clicked` | Mouse button pressed |
| `scroll` | Scroll event |

---

## Configuration

```json
{
  "config": {
    "frame_rate": 30,
    "action_delay_ms": 0,
    "retry_on_failure": true,
    "max_retries": 3
  }
}
```

- `frame_rate`: Target frames per second (1-60)
- `action_delay_ms`: Delay between action and next frame
- `retry_on_failure`: Retry failed actions
- `max_retries`: Maximum retry attempts

---

## Error Handling

```json
{
  "type": "error",
  "code": "WINDOW_NOT_FOUND",
  "message": "Window win_0x99999 not found",
  "recoverable": true
}
```

| Code | Description |
|------|-------------|
| `TRANSPORT_ERROR` | Connection lost or broken |
| `SERIALIZATION_ERROR` | Invalid JSON |
| `WINDOW_NOT_FOUND` | Target window doesn't exist |
| `ACTION_FAILED` | Action execution failed |
| `TIMEOUT` | Operation timed out |

---

## Appendix: Coordinate System

All coordinates are in **physical pixels** relative to the **primary display's top-left corner**.

- Origin (0, 0) is top-left of primary monitor
- X increases to the right
- Y increases downward
- Multi-monitor: coordinates extend beyond primary bounds
- Element bounds are relative to their window

---

## Appendix: Key Names

Standard key names for keyboard actions:

```
a-z, 0-9, enter, tab, escape, space, backspace, delete,
up, down, left, right,
home, end, page_up, page_down,
f1-f12,
ctrl, alt, shift, meta/super/win
```

Modifiers: `ctrl`, `alt`, `shift`, `meta`