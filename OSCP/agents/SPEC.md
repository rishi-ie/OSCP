# OSCP Agent Integration Specification

**Version:** 0.1.0
**Status:** Draft

---

## Overview

OSCP provides agents with raw render geometry — positions, z-order, texture IDs — enabling agents to see and interact with the desktop just like humans. Agent provides the meaning.

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
  │  Receive: frame stream         │
  │ ◄───────────────────────────── │
  │                                │
  │  Send: actions                 │
  │ ─────────────────────────────► │
```

---

## Data Received by Agent

### Frame Structure

Every frame sent by OSCP to the agent contains:

```json
{
  "type": "frame",
  "frame_id": 12345,
  "timestamp": 1716576000000,
  "surfaces": [
    {
      "id": "surface_abc123",
      "title": "Visual Studio Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "focused": true,
      "render_ops": [
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
    "x": 150,
    "y": 25,
    "hovered_op_id": "op_002"
  }
}
```

### Surface

A surface represents a window or visual region.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `title` | string | Window title |
| `bounds` | object | Window size (w, h) |
| `position` | object | Window position (x, y) |
| `focused` | boolean | Is window focused |
| `render_ops` | array | Draw operations from render pipeline |

### Render Operation

A render operation represents a single draw call from the graphics pipeline.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier within frame |
| `bounds` | object | Position (x, y) and size (w, h) |
| `z` | number | Z-index (render order, higher = on top) |
| `texture_id` | string | Texture identifier |
| `clip_bounds` | object | Optional clipping rectangle |

### Mouse

| Field | Type | Description |
|-------|------|-------------|
| `x` | number | Cursor X position |
| `y` | number | Cursor Y position |
| `hovered_op_id` | string | Render op under cursor (if any) |

---

## What Agent Infers

Agent receives raw geometry. Agent provides:

| What Agent Deduces | How |
|-------------------|-----|
| Element type (button, label, input) | From position, size, z-order patterns |
| Clickable regions | From z-order and texture hints |
| Text content | From texture patterns and surrounding elements |
| Element state | From mouse position and recent actions |
| Layout structure | From bounds and z-order relationships |

---

## Actions Sent by Agent

### Click

```json
{
  "type": "action",
  "action_id": "act_001",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "click",
    "x": 50,
    "y": 25,
    "button": "left",
    "click_type": "single"
  }
}
```

### Move

```json
{
  "type": "action",
  "action_id": "act_002",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "move",
    "x": 100,
    "y": 200
  }
}
```

### Drag

```json
{
  "type": "action",
  "action_id": "act_003",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "drag",
    "start_x": 100,
    "start_y": 200,
    "end_x": 300,
    "end_y": 400,
    "button": "left"
  }
}
```

### Scroll

```json
{
  "type": "action",
  "action_id": "act_004",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "scroll",
    "delta_x": 0,
    "delta_y": -3
  }
}
```

### Type

```json
{
  "type": "action",
  "action_id": "act_005",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "type",
    "text": "hello world"
  }
}
```

### Key Press

```json
{
  "type": "action",
  "action_id": "act_006",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "key_press",
    "key": "enter"
  }
}
```

### Key Combo

```json
{
  "type": "action",
  "action_id": "act_007",
  "surface_id": "surface_abc123",
  "action": {
    "kind": "key_combo",
    "key": "s",
    "modifiers": ["ctrl"]
  }
}
```

---

## Event Types

Agents receive system events.

| Event | Description |
|-------|-------------|
| `window_opened` | New surface created |
| `window_closed` | Surface removed |
| `window_focused` | Surface gained focus |
| `window_unfocused` | Surface lost focus |
| `window_moved` | Surface position changed |
| `window_resized` | Surface size changed |
| `key_pressed` | Keyboard input received |
| `mouse_moved` | Mouse moved |
| `mouse_clicked` | Mouse button pressed |
| `scroll` | Scroll event |

---

## Agent SDKs

### Supported Agents

| Agent | Language | SDK Location |
|-------|----------|---------------|
| OpenClaw | Python | `agents/openclaw/` |
| HermesAgent | TypeScript | `agents/hermes/` |
| PyAgent | Python | `agents/pyagent/` |

### SDK Interface

```python
import oscp

# Connect to OSCP driver
client = oscp.connect("tcp://localhost:9876")  # Windows
# or
client = oscp.connect("unix:///tmp/oscp.sock")  # macOS/Linux

# Get current frame
frame = client.get_frame()

# Iterate over surfaces
for surface in frame.surfaces:
    print(f"Window: {surface.title}")
    
    # Iterate over render operations (raw geometry)
    for op in surface.render_ops:
        print(f"  rect: {op.bounds}, z: {op.z}, texture: {op.texture_id}")

# Perform action
result = client.click(x=100, y=200)
```

```typescript
import { createOSCPClient } from 'oscp-sdk';

const client = createOSCPClient('tcp://localhost:9876');

const frame = await client.getFrame();

for (const surface of frame.surfaces) {
    console.log(`Window: ${surface.title}`);
    
    for (const op of surface.render_ops) {
        console.log(`  rect: ${op.bounds}, z: ${op.z}`);
    }
}

const result = await client.click({ x: 100, y: 200 });
```

---

## Error Handling

### Action Failures

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": false,
  "error": {
    "code": "SURFACE_NOT_FOUND",
    "message": "Surface surface_0x99999 not found",
    "recoverable": true
  }
}
```

### Recovery Strategies

| Error | Recovery |
|-------|----------|
| `SURFACE_NOT_FOUND` | Refresh surface list, retry |
| `ACTION_FAILED` | Retry with exponential backoff |
| `TRANSPORT_ERROR` | Reconnect |
| `TIMEOUT` | Retry |

---

## Transport Options

### TCP (Windows default)

```python
client = oscp.connect("tcp://localhost:9876")
```

### Unix Socket (macOS/Linux default)

```python
client = oscp.connect("unix:///tmp/oscp.sock")
```

### Stdio (embedded mode)

```python
client = oscp.connect("stdio:")
```

### WebSocket

```python
client = oscp.connect("ws://localhost:9876")
```

---

## Best Practices

### 1. Frame Rate

Match frame rate to use case:
- **Interactive UI:** 30fps
- **Static content:** 10fps
- **Video/animation:** 60fps

### 2. Element Caching

Cache element references between frames. Elements are stable unless window changes.

### 3. Action Batching

For complex sequences, batch actions and use `action_delay_ms` to ensure visual feedback.

### 4. Error Recovery

Always implement retry logic with exponential backoff. OSCP is designed for reliability, but network issues can occur.

---

## Principle

> "Intercept at the source. Deliver the geometry. Agent provides the meaning."

Agent receives positions, z-order, and texture IDs. Agent figures out what it all means.

---

## Status

🚧 Agent SDKs under development.