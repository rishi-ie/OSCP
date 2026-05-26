# OSCP Core Protocol Specification - Implementation Reference

**Version:** 0.4.0
**Status:** Ready for Implementation

---

## Table of Contents

1. [Overview](#1-overview)
2. [Connection](#2-connection)
3. [Message Types](#3-message-types)
4. [Element Reference](#4-element-reference)
5. [Error Handling](#5-error-handling)
6. [Protocol State Machine](#6-protocol-state-machine)
7. [Implementation Examples](#7-implementation-examples)

---

## 1. Overview

### 1.1 Protocol Design

OSCP uses a simple request-response pattern over a Unix domain socket:

- **Transport:** Unix domain socket (`/tmp/oscp.sock`)
- **Format:** JSON over newline-delimited frames
- **Pattern:** One request → One response → Next request

### 1.2 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| JSON over newline | Simple parsing, human-readable |
| Unix socket | Fast local communication, no TCP overhead |
| Request-response | Simple state machine, easy error handling |
| No streaming | Agent controls timing, lower resource usage |

### 1.3 Message Flow

```
Agent                              OSCP Service
  │                                    │
  │ ──── connect() ──────────────────► │
  │ ──── hello ─────────────────────► │
  │ ◄─── welcome ──────────────────── │
  │                                    │
  │ ──── get_frame ─────────────────► │
  │ ◄─── frame ───────────────────── │
  │                                    │
  │ ──── action(click) ─────────────► │
  │ ◄─── action_result ───────────── │
  │                                    │
  │ ──── action(type) ─────────────► │
  │ ◄─── action_result ───────────── │
  │                                    │
  │ ──── disconnect ────────────────► │
  │ ◄─── goodbye ──────────────────── │
```

---

## 2. Connection

### 2.1 Unix Socket Setup (macOS/Linux)

```c
#include <sys/socket.h>
#include <sys/un.h>

// Socket path
const char* SOCKET_PATH = "/tmp/oscp.sock";

// Create socket
int fd = socket(AF_UNIX, SOCK_STREAM, 0);

// Bind to path
struct sockaddr_un addr;
memset(&addr, 0, sizeof(addr));
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

bind(fd, (struct sockaddr*)&addr, sizeof(addr));

// Listen
listen(fd, 5);
```

### 2.2 Swift Implementation

```swift
import Foundation

class ProtocolServer {
    private let socketPath: String
    private var serverFd: Int32 = -1
    
    func start() throws {
        // Remove existing socket
        unlink(socketPath)
        
        // Create socket
        serverFd = socket(AF_UNIX, SOCK_STREAM, 0)
        guard serverFd >= 0 else {
            throw OSCPError.systemError(message: "Failed to create socket")
        }
        
        // Bind
        var addr = sockaddr_un()
        addr.sun_family = sa_family_t(AF_UNIX)
        
        socketPath.withCString { ptr in
            withUnsafeMutablePointer(to: &addr.sun_path) { pathPtr in
                let pathBuf = UnsafeMutableRawPointer(pathPtr)
                    .assumingMemoryBound(to: CChar.self)
                strncpy(pathBuf, ptr, Int(MAXPATHLEN) - 1)
            }
        }
        
        let bindResult = withUnsafePointer(to: &addr) { ptr in
            ptr.withMemoryRebound(to: sockaddr.self, capacity: 1) { sockaddrPtr in
                bind(self.serverFd, sockaddrPtr,
                     socklen_t(MemoryLayout<sockaddr_un>.size))
            }
        }
        
        guard bindResult == 0 else {
            close(serverFd)
            throw OSCPError.systemError(message: "Failed to bind socket")
        }
        
        // Listen
        guard listen(serverFd, 5) == 0 else {
            close(serverFd)
            throw OSCPError.systemError(message: "Failed to listen")
        }
    }
}
```

### 2.3 Client Connection (Python)

```python
import socket
import json

class OSCPClient:
    def __init__(self, socket_path="/tmp/oscp.sock"):
        self.socket_path = socket_path
        self.sock = None
        self.request_id = 0
        
    def connect(self):
        self.sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.sock.connect(self.socket_path)
        
        # Send hello
        hello = {"type": "hello", "version": "0.4"}
        self._send(hello)
        
        # Receive welcome
        welcome = self._receive()
        if welcome["type"] != "welcome":
            raise Exception("Expected welcome, got", welcome["type"])
        
        return welcome
    
    def _send(self, message):
        data = json.dumps(message) + "\n"
        self.sock.sendall(data.encode())
    
    def _receive(self):
        data = b""
        while b"\n" not in data:
            chunk = self.sock.recv(4096)
            if not chunk:
                break
            data += chunk
        
        return json.loads(data.decode().strip())
```

### 2.4 Client Connection (TypeScript)

```typescript
import net from 'net';

class OSCPClient {
    private socket: net.Socket;
    private requestId: number = 0;
    
    constructor(private socketPath: string = '/tmp/oscp.sock') {
        this.socket = net.createConnection(this.socketPath);
    }
    
    async connect(): Promise<WelcomeMessage> {
        return new Promise((resolve, reject) => {
            this.socket.write(JSON.stringify({
                type: 'hello',
                version: '0.4'
            }) + '\n');
            
            let buffer = '';
            this.socket.on('data', (data) => {
                buffer += data.toString();
                if (buffer.includes('\n')) {
                    const welcome = JSON.parse(buffer.trim());
                    resolve(welcome);
                }
            });
        });
    }
    
    async send(message: object): Promise<any> {
        return new Promise((resolve, reject) => {
            const requestId = `req_${++this.requestId}`;
            const fullMessage = { ...message, request_id: requestId };
            
            this.socket.write(JSON.stringify(fullMessage) + '\n');
            
            let buffer = '';
            this.socket.on('data', (data) => {
                buffer += data.toString();
                if (buffer.includes('\n')) {
                    resolve(JSON.parse(buffer.trim()));
                }
            });
        });
    }
}
```

### 2.5 Hello/Welcome Exchange

#### Hello Message (Agent → OSCP)

```json
{
  "type": "hello",
  "version": "0.4"
}
```

#### Welcome Message (OSCP → Agent)

```json
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

## 3. Message Types

### 3.1 Get Frame

#### Request: get_frame

```json
{
  "type": "get_frame",
  "request_id": "req_001"
}
```

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be "get_frame" |
| `request_id` | string | Yes | Unique request identifier |

#### Response: frame

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
        "x": 0,
        "y": 0
      },
      "focused": true,
      "minimized": false,
      "elements": [
        {
          "id": "e_1234_a1b2c3d4",
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
          "states": ["enabled", "visible", "actionable"],
          "attributes": {
            "default_button": false
          },
          "confidence": 0.95,
          "source": "axuielement",
          "children": []
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

**Frame Response Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `type` | string | "frame" |
| `request_id` | string | Matches request |
| `frame_id` | int | Incremental frame counter |
| `platform` | string | "macOS", "linux", or "windows" |
| `latency_ms` | int | Capture latency in milliseconds |
| `timestamp` | int64 | Unix timestamp (milliseconds) |
| `windows` | array | List of windows |
| `tree_analysis` | object | Tree quality analysis |
| `mouse` | object | Current mouse state |
| `keyboard` | object | Current keyboard modifiers |

---

### 3.2 Click Action

#### Request: action (click)

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

**Click Action Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | Yes | Must be "click" |
| `x` | number | Yes | X coordinate |
| `y` | number | Yes | Y coordinate |
| `button` | string | No | "left" (default), "right", "middle" |
| `click_type` | string | No | "single", "double", "triple", "right" |

#### Response: action_result

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "timestamp": 1716576000051,
  "latency_ms": 1,
  "confidence": 0.95,
  "target": {
    "element_id": "e_001",
    "element_name": "Save",
    "element_role": "button"
  }
}
```

**Action Result Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `type` | string | "action_result" |
| `action_id` | string | Matches request |
| `success` | boolean | Whether action succeeded |
| `timestamp` | int64 | Completion timestamp |
| `latency_ms` | int | Execution time |
| `confidence` | number | Confidence score (0-1) |
| `target` | object | Target element info (if available) |
| `error` | object | Error details (if failed) |

#### Failure Response

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": false,
  "timestamp": 1716576000051,
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

### 3.3 Type Action

#### Request: action (type)

```json
{
  "type": "action",
  "action_id": "act_002",
  "action": {
    "kind": "type",
    "text": "hello world",
    "typing_delay_ms": 50
  }
}
```

**Type Action Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | Yes | Must be "type" |
| `text` | string | Yes | Text to type |
| `typing_delay_ms` | int | No | Delay between keystrokes (default: 50) |

---

### 3.4 Key Combo Action

#### Request: action (key_combo)

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

**Key Combo Action Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | Yes | Must be "key_combo" |
| `key` | string | Yes | Key to press |
| `modifiers` | array | No | Modifier keys to hold |

**Modifier Keys:**
| Value | Description |
|-------|-------------|
| `ctrl` | Control key |
| `alt` | Option/Alt key |
| `shift` | Shift key |
| `cmd` | Command (macOS) / Win (Windows) / Super (Linux) |
| `fn` | Function key |

**Key Values:**
```
Letters:      a-z
Numbers:      0-9
Special:      return, tab, space, delete, escape
Arrows:       up, down, left, right
Function:     f1-f12
Navigation:   home, end, page_up, page_down
```

**Common Key Combos:**
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

// Escape
{"kind": "key_combo", "key": "escape"}

// Tab
{"kind": "key_combo", "key": "tab"}
```

---

### 3.5 Scroll Action

#### Request: action (scroll)

```json
{
  "type": "action",
  "action_id": "act_004",
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

**Scroll Action Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | Yes | Must be "scroll" |
| `x` | number | No | X anchor point |
| `y` | number | No | Y anchor point |
| `delta_x` | number | Yes | Horizontal scroll delta |
| `delta_y` | number | Yes | Vertical scroll delta |
| `scroll_type` | string | No | "precise" (default), "line", "page" |

---

### 3.6 Drag Action

#### Request: action (drag)

```json
{
  "type": "action",
  "action_id": "act_005",
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

**Drag Action Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | Yes | Must be "drag" |
| `start_x` | number | Yes | Start X coordinate |
| `start_y` | number | Yes | Start Y coordinate |
| `end_x` | number | Yes | End X coordinate |
| `end_y` | number | Yes | End Y coordinate |
| `button` | string | No | "left" (default), "right" |
| `duration_ms` | int | No | Duration in milliseconds (default: 500) |

---

### 3.7 Move Action

#### Request: action (move)

```json
{
  "type": "action",
  "action_id": "act_006",
  "action": {
    "kind": "move",
    "x": 960,
    "y": 540
  }
}
```

---

### 3.8 Mouse Position

#### Request: mouse_position

```json
{
  "type": "mouse_position",
  "request_id": "req_002"
}
```

#### Response: mouse_position_result

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

### 3.9 Error

#### Response: error

```json
{
  "type": "error",
  "request_id": "req_001",
  "code": "EMPTY_TREE",
  "message": "Semantic tree is empty",
  "timestamp": 1716576000046,
  "details": {
    "coverage_score": 0.05,
    "reasoning": "Custom renderer detected, no AT-SPI support",
    "fallback_method": "position_only"
  }
}
```

---

### 3.10 Human Handoff

#### Request: handoff_request (OSCP → Agent)

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

#### Response: handoff_response (Agent → OSCP)

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

**Handoff Resolutions:**
| Value | Description |
|-------|-------------|
| `agent_continue` | Agent will try alternative positions |
| `human_completed` | Human completed the task |
| `skip_task` | Task should be skipped |
| `learn_from_human` | Agent learns from human's click |

---

### 3.11 Keep-Alive

#### Request: ping

```json
{
  "type": "ping",
  "timestamp": 1716576000000
}
```

#### Response: pong

```json
{
  "type": "pong",
  "timestamp": 1716576000000,
  "latency_ms": 1
}
```

---

### 3.12 Disconnect

#### Request: disconnect

```json
{
  "type": "disconnect"
}
```

#### Response: goodbye

```json
{
  "type": "goodbye"
}
```

---

## 4. Element Reference

### 4.1 Element Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique ID within frame (format: "e_{pid}_{hash}") |
| `role` | string | ARIA-style role |
| `subrole` | string | More specific role variant |
| `name` | string | Accessible name |
| `description` | string | Accessible description |
| `value` | string | Current value |
| `bounds` | object | {x, y, w, h} in screen coordinates |
| `states` | array | Active states |
| `attributes` | object | Additional role-specific attributes |
| `confidence` | number | Confidence score (0.0-1.0) |
| `source` | string | API source |
| `children` | array | Child elements |

### 4.2 Role Values

```json
// Interactive elements
"button"
"check_box"
"radio_button"
"text_field"
"text_area"
"combo_box"
"slider"
"spin_button"
"list_item"
"menu_item"
"tab"
"link"

// Static elements
"static_text"
"image"
"icon"
"graphic"
"group"
"label"

// Containers
"window"
"dialog"
"alert"
"menu"
"menu_bar"
"toolbar"
"tab_group"
"list"
"table"
"row"
"column"
"tabular_cell"
"tree"
"tree_item"
"split_group"
"form"
"frame"

// Specialized
"scroll_bar"
"scroll_area"
"color_well"
"date_picker"
"time_picker"
"tree_grid"
"description_list"
"term"
"definition"
"separator"
"paragraph"
"heading"
"blockquote"
"caption"
"note"
"content_info"
"definition"
"math"
"paragraph"
"span"
"line"
"word"
"character"
"unknown"
```

### 4.3 State Values

```json
"enabled"          // Element is enabled
"disabled"         // Element is disabled
"visible"          // Element is visible
"invisible"        // Element is hidden
"focused"          // Element has focus
"unfocused"        // Element does not have focus
"selected"         // Element is selected
"unselected"       // Element is not selected
"checked"          // Checkbox/radio is checked
"unchecked"       // Checkbox/radio is unchecked
"indeterminate"   // Checkbox is indeterminate
"pressed"         // Button is pressed
"expanded"        // Expandable is expanded
"collapsed"       // Expandable is collapsed
"actionable"      // Element can be interacted with
"updating"        // Element is updating/loading
"readonly"        // Element is read-only
"busy"            // Element is busy
"linked"          // Link has been visited
"visited"         // (synonym for linked)
```

### 4.4 Source Values

```json
"axuielement"   // macOS AXUIElement API
"atspi"         // Linux AT-SPI2 API
"uia"           // Windows UIAutomation API
"x11"           // X11 fallback (Linux)
"cdp"           // Chrome DevTools Protocol
"heuristic"     // Position-based inference
"position"      // Position-only mode
"unknown"       // Unknown source
```

### 4.5 Bounds

```json
{
  "x": 100,   // X coordinate (from left)
  "y": 50,    // Y coordinate (from top)
  "w": 200,    // Width
  "h": 30     // Height
}
```

**Coordinate System:**
- Origin: Top-left corner of screen
- X: Increases to the right
- Y: Increases downward

### 4.6 Element Example

```json
{
  "id": "e_1234_a1b2c3d4",
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
  "states": ["enabled", "visible", "actionable"],
  "attributes": {
    "default_button": false,
    "cancel_button": false
  },
  "confidence": 0.95,
  "source": "axuielement",
  "children": []
}
```

---

## 5. Error Handling

### 5.1 Error Codes

| Code | Description | Agent Action |
|------|-------------|--------------|
| `PERMISSION_DENIED` | Accessibility permissions not granted | Guide user to grant permission |
| `ELEMENT_NOT_FOUND` | Target element not found | Re-capture frame, try alternatives |
| `ELEMENT_DISABLED` | Target element is disabled | Wait for element to be enabled |
| `ELEMENT_MOVED` | Target element moved since last frame | Re-capture frame, find new position |
| `ACTION_FAILED` | Action did not produce expected result | Try alternatives or handoff |
| `EMPTY_TREE` | Semantic tree is empty | Use fallback methods |
| `LOW_COVERAGE` | Tree coverage below threshold | Apply heuristics or handoff |
| `TIMEOUT` | Operation timed out | Retry or skip |
| `CONNECTION_LOST` | Socket connection lost | Reconnect |
| `INVALID_REQUEST` | Malformed request JSON | Fix request format |
| `UNSUPPORTED_ACTION` | Action type not supported | Use different action |
| `PLATFORM_ERROR` | Platform-specific error | Log error, skip action |

### 5.2 Error Response Structure

```json
{
  "type": "error",
  "request_id": "req_001",
  "code": "ERROR_CODE",
  "message": "Human-readable message",
  "timestamp": 1716576000046,
  "details": {
    "coverage_score": 0.05,
    "reasoning": "Optional explanation",
    "fallback_method": "position_only",
    "recommended_action": "explore_or_handoff"
  }
}
```

### 5.3 Error Handling Flow

```
1. OSCP sends error response
   │
   ▼
2. Agent checks error.code
   │
   ├─ PERMISSION_DENIED → Request user permission
   ├─ ELEMENT_NOT_FOUND → Re-capture frame, retry
   ├─ ACTION_FAILED → Try alternatives or handoff
   ├─ LOW_COVERAGE → Apply heuristics or handoff
   └─ CONNECTION_LOST → Reconnect
```

---

## 6. Protocol State Machine

### 6.1 States

```
┌─────────────┐
│  CONNECTING │ ← Initial state
└──────┬──────┘
       │ socket connect
       ▼
┌──────┴──────┐
│ WELCOMING  │ ← Awaiting hello
└──────┬──────┘
       │ receive hello
       ▼
┌──────┴──────┐
│   READY    │ ← Normal operation
└──────┬──────┘
       │
       ├─ get_frame ──► FRAME_SENT ──► READY
       │
       ├─ action ─────► ACTION_EXECUTED ──► READY
       │
       ├─ handoff ───► HANDOVER_IN_PROGRESS ──► READY
       │
       ├─ ping ──────► PONG_SENT ──► READY
       │
       └─ disconnect ──► CLOSING ──► CLOSED
```

### 6.2 State Transitions

| State | Event | Next State | Action |
|-------|-------|------------|--------|
| CONNECTING | connect | WELCOMING | - |
| WELCOMING | hello | READY | send welcome |
| READY | get_frame | FRAME_SENT | capture + send frame |
| READY | action | ACTION_EXECUTED | execute + send result |
| READY | handoff_response | READY | process handoff |
| READY | ping | PONG_SENT | send pong |
| READY | disconnect | CLOSING | send goodbye, close |
| FRAME_SENT | (response sent) | READY | - |
| ACTION_EXECUTED | (result sent) | READY | - |
| CLOSING | (close complete) | CLOSED | cleanup |

---

## 7. Implementation Examples

### 7.1 Python Agent SDK

```python
import socket
import json
import asyncio
from typing import Optional, Callable, List, Dict, Any

class OSCPClient:
    """Python SDK for OSCP protocol."""
    
    def __init__(self, socket_path: str = "/tmp/oscp.sock"):
        self.socket_path = socket_path
        self.sock: Optional[socket.socket] = None
        self.request_id = 0
        self.frame_id = 0
        self.connected = False
        
    def connect(self) -> dict:
        """Connect to OSCP service."""
        self.sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.sock.connect(self.socket_path)
        
        # Send hello
        self._send({"type": "hello", "version": "0.4"})
        
        # Receive welcome
        welcome = self._receive()
        if welcome["type"] != "welcome":
            raise Exception("Expected welcome, got", welcome["type"])
        
        self.connected = True
        return welcome
    
    def disconnect(self):
        """Disconnect from OSCP service."""
        if self.sock:
            self._send({"type": "disconnect"})
            self._receive()  # Receive goodbye
            self.sock.close()
            self.sock = None
            self.connected = False
    
    async def get_frame(self) -> dict:
        """Request current frame."""
        request_id = f"req_{self._next_id()}"
        self._send({"type": "get_frame", "request_id": request_id})
        response = self._receive()
        
        if response["type"] == "error":
            raise OSCPError(response["code"], response["message"])
        
        self.frame_id = response.get("frame_id", 0)
        return response
    
    async def click(
        self,
        x: float,
        y: float,
        button: str = "left",
        click_type: str = "single"
    ) -> dict:
        """Click at position."""
        return await self._action({
            "kind": "click",
            "x": x,
            "y": y,
            "button": button,
            "click_type": click_type
        })
    
    async def type_text(
        self,
        text: str,
        typing_delay_ms: int = 50
    ) -> dict:
        """Type text."""
        return await self._action({
            "kind": "type",
            "text": text,
            "typing_delay_ms": typing_delay_ms
        })
    
    async def key_combo(
        self,
        key: str,
        modifiers: List[str] = None
    ) -> dict:
        """Press key combination."""
        return await self._action({
            "kind": "key_combo",
            "key": key,
            "modifiers": modifiers or []
        })
    
    async def scroll(
        self,
        delta_x: float = 0,
        delta_y: float = 0,
        x: float = None,
        y: float = None
    ) -> dict:
        """Scroll."""
        return await self._action({
            "kind": "scroll",
            "delta_x": delta_x,
            "delta_y": delta_y,
            "x": x,
            "y": y,
            "scroll_type": "precise"
        })
    
    async def drag(
        self,
        start_x: float,
        start_y: float,
        end_x: float,
        end_y: float,
        duration_ms: int = 500
    ) -> dict:
        """Drag from start to end."""
        return await self._action({
            "kind": "drag",
            "start_x": start_x,
            "start_y": start_y,
            "end_x": end_x,
            "end_y": end_y,
            "duration_ms": duration_ms
        })
    
    async def move_mouse(self, x: float, y: float) -> dict:
        """Move mouse to position."""
        return await self._action({
            "kind": "move",
            "x": x,
            "y": y
        })
    
    async def mouse_position(self) -> dict:
        """Get current mouse position."""
        request_id = f"req_{self._next_id()}"
        self._send({"type": "mouse_position", "request_id": request_id})
        return self._receive()
    
    async def handoff_response(
        self,
        resolution: str,
        human_click: dict = None
    ) -> None:
        """Respond to handoff request."""
        response = {
            "type": "handoff_response",
            "resolution": resolution
        }
        if human_click:
            response["human_click"] = human_click
        self._send(response)
    
    # Private methods
    
    def _send(self, message: dict):
        """Send JSON message."""
        if not self.sock:
            raise Exception("Not connected")
        
        data = json.dumps(message) + "\n"
        self.sock.sendall(data.encode())
    
    def _receive(self) -> dict:
        """Receive JSON message."""
        if not self.sock:
            raise Exception("Not connected")
        
        buffer = b""
        while b"\n" not in buffer:
            chunk = self.sock.recv(4096)
            if not chunk:
                raise Exception("Connection closed")
            buffer += chunk
        
        line = buffer.decode().split("\n")[0]
        return json.loads(line)
    
    def _next_id(self) -> int:
        """Generate next request ID."""
        self.request_id += 1
        return self.request_id
    
    async def _action(self, action: dict) -> dict:
        """Execute action and return result."""
        action_id = f"act_{self._next_id()}"
        message = {
            "type": "action",
            "action_id": action_id,
            "action": action
        }
        self._send(message)
        response = self._receive()
        
        if response["type"] == "error":
            raise OSCPError(response["code"], response["message"])
        
        return response


class OSCPError(Exception):
    """OSCP protocol error."""
    
    def __init__(self, code: str, message: str):
        self.code = code
        self.message = message
        super().__init__(f"{code}: {message}")


# Example usage
async def example():
    client = OSCPClient("/tmp/oscp.sock")
    
    try:
        # Connect
        welcome = client.connect()
        print(f"Connected to OSCP v{welcome['version']} on {welcome['platform']}")
        
        # Get frame
        frame = await client.get_frame()
        print(f"Frame #{frame['frame_id']} - Confidence: {frame['tree_analysis']['confidence']}")
        
        # Find and click a button
        for window in frame["windows"]:
            for element in window["elements"]:
                if element["role"] == "button" and element.get("name") == "Save":
                    result = await client.click(
                        x=element["bounds"]["x"] + 5,
                        y=element["bounds"]["y"] + 5
                    )
                    print(f"Click result: {result['success']}")
                    break
        
        # Type some text
        result = await client.type_text("Hello, World!")
        print(f"Type result: {result['success']}")
        
        # Press Ctrl+S
        result = await client.key_combo("s", modifiers=["ctrl"])
        print(f"Key combo result: {result['success']}")
        
    finally:
        client.disconnect()


if __name__ == "__main__":
    asyncio.run(example())
```

### 7.2 Swift Client SDK

```swift
import Foundation

/// Swift SDK for OSCP protocol
public actor OSCPClient {
    
    // MARK: - Properties
    
    private let socketPath: String
    private var inputStream: InputStream?
    private var outputStream: OutputStream?
    private var requestId: Int = 0
    private var frameId: Int = 0
    private var isConnected = false
    
    private var receiveTask: Task<Void, Never>?
    private var messageHandler: ((Data) -> Void)?
    
    // MARK: - Initialization
    
    public init(socketPath: String = "/tmp/oscp.sock") {
        self.socketPath = socketPath
    }
    
    // MARK: - Connection
    
    public func connect() async throws -> WelcomeMessage {
        // Create socket pair
        var readFd: Int32 = 0
        var writeFd: Int32 = 0
        
        let socketPair = socket(AF_UNIX, SOCK_STREAM, 0)
        guard socketPair >= 0 else {
            throw OSCPClientError.socketCreationFailed
        }
        
        // Connect to server
        var addr = sockaddr_un()
        addr.sun_family = sa_family_t(AF_UNIX)
        
        socketPath.withCString { ptr in
            withUnsafeMutablePointer(to: &addr.sun_path) { pathPtr in
                let pathBuf = UnsafeMutableRawPointer(pathPtr)
                    .assumingMemoryBound(to: CChar.self)
                strncpy(pathBuf, ptr, Int(MAXPATHLEN) - 1)
            }
        }
        
        let connectResult = withUnsafePointer(to: &addr) { ptr in
            ptr.withMemoryRebound(to: sockaddr.self, capacity: 1) { sockaddrPtr in
                connect(socketPair, sockaddrPtr,
                       socklen_t(MemoryLayout<sockaddr_un>.size))
            }
        }
        
        guard connectResult == 0 else {
            close(socketPair)
            throw OSCPClientError.connectionFailed
        }
        
        // Create streams
        inputStream = InputStream(fileDescriptor: socketPair, closeDealloc: true)
        outputStream = OutputStream(fileDescriptor: socketPair, closeDealloc: false)
        
        inputStream?.open()
        outputStream?.open()
        
        // Send hello
        try await send(["type": "hello", "version": "0.4"])
        
        // Receive welcome
        guard let welcomeData = try await receive(),
              let welcome = try? JSONDecoder().decode(WelcomeMessage.self, from: welcomeData) else {
            throw OSCPClientError.invalidWelcome
        }
        
        isConnected = true
        
        // Start receive loop
        receiveTask = Task {
            await self.receiveLoop()
        }
        
        return welcome
    }
    
    public func disconnect() async {
        guard isConnected else { return }
        
        try? await send(["type": "disconnect"])
        try? await Task.sleep(nanoseconds: 100_000_000)  // 100ms
        
        receiveTask?.cancel()
        inputStream?.close()
        outputStream?.close()
        isConnected = false
    }
    
    // MARK: - Commands
    
    public func getFrame() async throws -> Frame {
        let requestId = nextRequestId()
        try await send(["type": "get_frame", "request_id": requestId])
        
        guard let data = try await receive() else {
            throw OSCPClientError.noResponse
        }
        
        let decoder = JSONDecoder()
        let response = try decoder.decode(FrameResponse.self, from: data)
        
        if let error = response.error {
            throw OSCPError(code: error.code, message: error.message)
        }
        
        self.frameId = response.frame?.frameId ?? 0
        return response.frame!
    }
    
    public func click(
        x: Double,
        y: Double,
        button: String = "left",
        clickType: String = "single"
    ) async throws -> ActionResult {
        return try await action([
            "kind": "click",
            "x": x,
            "y": y,
            "button": button,
            "click_type": clickType
        ])
    }
    
    public func typeText(_ text: String, delayMs: Int = 50) async throws -> ActionResult {
        return try await action([
            "kind": "type",
            "text": text,
            "typing_delay_ms": delayMs
        ])
    }
    
    public func keyCombo(_ key: String, modifiers: [String] = []) async throws -> ActionResult {
        return try await action([
            "kind": "key_combo",
            "key": key,
            "modifiers": modifiers
        ])
    }
    
    public func scroll(deltaX: Double = 0, deltaY: Double = 0, x: Double? = nil, y: Double? = nil) async throws -> ActionResult {
        var action: [String: Any] = [
            "kind": "scroll",
            "delta_x": deltaX,
            "delta_y": deltaY,
            "scroll_type": "precise"
        ]
        if let x = x { action["x"] = x }
        if let y = y { action["y"] = y }
        return try await action(action)
    }
    
    public func drag(
        startX: Double,
        startY: Double,
        endX: Double,
        endY: Double,
        durationMs: Int = 500
    ) async throws -> ActionResult {
        return try await action([
            "kind": "drag",
            "start_x": startX,
            "start_y": startY,
            "end_x": endX,
            "end_y": endY,
            "duration_ms": durationMs
        ])
    }
    
    public func moveMouse(x: Double, y: Double) async throws -> ActionResult {
        return try await action([
            "kind": "move",
            "x": x,
            "y": y
        ])
    }
    
    // MARK: - Private Methods
    
    private func send(_ message: [String: Any]) async throws {
        let data = try JSONSerialization.data(withJSONObject: message)
        var line = data
        line.append(contentsOf: "\n".utf8)
        
        guard let outputStream = outputStream else {
            throw OSCPClientError.notConnected
        }
        
        let bytes = [UInt8](line)
        let written = outputStream.write(bytes, maxLength: bytes.count)
        
        guard written == bytes.count else {
            throw OSCPClientError.sendFailed
        }
    }
    
    private func receive() async throws -> Data? {
        return try await withCheckedThrowingContinuation { continuation in
            var buffer = [UInt8](repeating: 0, count: 4096)
            
            guard let inputStream = inputStream else {
                continuation.resume(throwing: OSCPClientError.notConnected)
                return
            }
            
            inputStream.schedule(in: .main, forMode: .default)
            
            inputStream.setHandler { [weak self] stream in
                guard stream == self?.inputStream else { return }
                
                let bytesRead = inputStream.read(&buffer, maxLength: buffer.count)
                
                if bytesRead > 0 {
                    let data = Data(buffer.prefix(bytesRead))
                    continuation.resume(returning: data)
                } else if bytesRead < 0 {
                    continuation.resume(throwing: OSCPClientError.receiveFailed)
                }
            }
        }
    }
    
    private func receiveLoop() async {
        while isConnected {
            do {
                if let _ = try await receive() {
                    // Handle incoming messages
                }
            } catch {
                break
            }
        }
    }
    
    private func action(_ action: [String: Any]) async throws -> ActionResult {
        let actionId = "act_\(nextRequestId())"
        
        try await send([
            "type": "action",
            "action_id": actionId,
            "action": action
        ])
        
        guard let data = try await receive() else {
            throw OSCPClientError.noResponse
        }
        
        let response = try JSONDecoder().decode(ActionResultResponse.self, from: data)
        
        if let error = response.error {
            throw OSCPError(code: error.code, message: error.message)
        }
        
        return response.result!
    }
    
    private func nextRequestId() -> Int {
        requestId += 1
        return requestId
    }
}

// MARK: - Types

public struct WelcomeMessage: Codable {
    public let type: String
    public let version: String
    public let platform: String
    public let capabilities: [String]
    public let inputMethods: [String]
    
    enum CodingKeys: String, CodingKey {
        case type, version, platform, capabilities
        case inputMethods = "input_methods"
    }
}

public struct Frame: Codable {
    public let frameId: Int
    public let platform: String
    public let latencyMs: Int
    public let timestamp: Int64
    public let windows: [Window]
    public let treeAnalysis: TreeAnalysis
    public let mouse: MouseState
    public let keyboard: KeyboardState
}

public struct Window: Codable {
    public let id: String
    public let title: String
    public let pid: Int32
    public let app: String?
    public let bounds: Bounds
    public let position: CGPoint
    public let focused: Bool
    public let minimized: Bool
    public let elements: [Element]
}

public struct Bounds: Codable {
    public let x: Double
    public let y: Double
    public let w: Double
    public let h: Double
}

public struct Element: Codable {
    public let id: String
    public let role: String
    public let subrole: String?
    public let name: String?
    public let description: String?
    public let value: String?
    public let bounds: Bounds
    public let states: [String]
    public let attributes: [String: String]
    public let confidence: Double
    public let source: String
}

public struct ActionResult: Codable {
    public let actionId: String
    public let success: Bool
    public let latencyMs: Int
    public let confidence: Double
    public let error: ActionError?
}

// MARK: - Response Types

struct FrameResponse: Codable {
    let type: String?
    let frameId: Int?
    let frame: Frame?
    let error: ErrorResponse?
}

struct ActionResultResponse: Codable {
    let type: String?
    let actionId: String?
    let success: Bool?
    let result: ActionResult?
    let error: ErrorResponse?
}

struct ErrorResponse: Codable {
    let code: String
    let message: String
}

// MARK: - Errors

public enum OSCPClientError: Error {
    case socketCreationFailed
    case connectionFailed
    case invalidWelcome
    case notConnected
    case sendFailed
    case receiveFailed
    case noResponse
}

public struct OSCPError: Error {
    public let code: String
    public let message: String
}
```

---

## Status

- [x] Protocol fully specified
- [x] All message types defined
- [x] Element reference complete
- [x] Error codes documented
- [x] State machine documented
- [x] Python SDK example
- [x] Swift SDK example
- [ ] Implementation pending

---

## Appendix: Complete Message Type Index

| Message Type | Direction | Description |
|--------------|-----------|-------------|
| `hello` | Agent → OSCP | Initial handshake |
| `welcome` | OSCP → Agent | Handshake response |
| `get_frame` | Agent → OSCP | Request screen capture |
| `frame` | OSCP → Agent | Screen capture response |
| `action` | Agent → OSCP | Execute action |
| `action_result` | OSCP → Agent | Action result |
| `mouse_position` | Agent → OSCP | Request mouse position |
| `mouse_position_result` | OSCP → Agent | Mouse position response |
| `error` | OSCP → Agent | Error notification |
| `handoff_request` | OSCP → Agent | Human handoff request |
| `handoff_response` | Agent → OSCP | Human handoff response |
| `ping` | Agent → OSCP | Keep-alive ping |
| `pong` | OSCP → Agent | Keep-alive pong |
| `disconnect` | Agent → OSCP | Initiate disconnect |
| `goodbye` | OSCP → Agent | Confirm disconnect |