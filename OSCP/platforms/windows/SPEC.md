# OSCP Windows Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The Windows platform driver provides OSCP protocol implementation for Windows 10/11. It uses UI Automation (UIA) and Win32 APIs to provide semantic desktop data to agents.

**Principle:** Intercept the compositor. Decode the render tree. Agent provides the meaning.

**Note:** Windows lacks documented APIs to extract render operations from DWM. UIA + Win32 provides semantic data (element types, names, states) with 90% coverage. Render operations approach deferred to V2.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐ │
│  │     Win32       │  │       UIA       │  │ Input Engine  │ │
│  │   Window Enum   │  │   Element Tree  │  │   (Rust)      │ │
│  │   (Rust)        │  │   (Rust)       │  │               │ │
│  └────────┬────────┘  └────────┬────────┘  └───────┬───────┘ │
│           │                    │                    │         │
│           └──────────┬─────────┘                    │         │
│                      │                              │         │
│               ┌──────▼──────┐                       │         │
│               │   Tree      │◄──────────────────────┘         │
│               │   Builder   │                                   │
│               │   (Rust)    │                                   │
│               └──────┬──────┘                                   │
│                      │                                          │
│               ┌──────▼──────┐                                   │
│               │  Protocol    │                                   │
│               │  Server      │                                   │
│               │  (TCP)       │                                   │
│               └──────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Approach

### Why Not Render Operations?

Windows DWM (Desktop Window Manager) does not expose render operations via documented APIs. The options are:

| Approach | Coverage | Complexity | Stability |
|----------|----------|------------|-----------|
| Per-app DXGI Hook | 85% | High | Low |
| DWM Hook | 100% | Very High | Unstable |
| Kernel Driver | 100% | Extreme | Stable (if signed) |
| **UIA + Win32** | **90%** | **Low** | **High** |

UIA + Win32 is the practical choice for V1. Render operations approach (DWM hook) is planned for V2.

---

## Components

### 1. Win32 Window Enumeration (Rust)

**Purpose:** Enumerate all top-level windows and their basic properties.

**What it captures:**
- Window handles (HWND)
- Window titles
- Window positions and sizes
- Z-order (stacking order)
- Process information

**Implementation:**
```rust
// Win32 APIs used:
fn enum_windows() -> Vec<WindowInfo> {
    // 1. EnumWindows - enumerate all top-level windows
    // 2. GetWindowRect - window position and size
    // 3. GetWindowText - window title
    // 4. GetWindowZOrder - window z-order via GetWindow()
    // 5. GetWindowThreadProcessId - owning process
}

// Output:
struct WindowInfo {
    hwnd: isize,
    title: String,
    bounds: Rect,
    z_order: u32,
    process_id: u32,
    process_name: String,
}
```

### 2. UI Automation Bridge (Rust)

**Purpose:** Extract semantic element tree for each window.

**What it captures:**
- Element types (button, menu, text_field, etc.)
- Element names
- Element states (enabled, focused, etc.)
- Bounding rectangles
- Parent-child relationships

**Implementation:**
```rust
// UIA COM API used:
fn extract_element_tree(hwnd: HWND) -> ElementTree {
    // 1. CUIAutomation::ElementFromHandle(hwnd)
    // 2. IUIAutomationElement::FindAll() for children
    // 3. IUIAutomationElement::get_CurrentName()
    // 4. IUIAutomationElement::get_CurrentControlType()
    // 5. IUIAutomationElement::get_CurrentBoundingRectangle()
    // 6. IUIAutomationElement::get_CurrentIsEnabled()
}

// Element types mapped:
enum ElementType {
    Button, Menu, MenuItem,
    TextField, TextArea, Label,
    CheckBox, RadioButton,
    List, ListItem,
    Tree, TreeItem,
    Tab, TabItem,
    Custom, Unknown,
}
```

### 3. Input Engine (Rust)

**Purpose:** Execute agent actions on Windows.

**Methods:**
- `SendInput`: Primary for keyboard and mouse events
- `PostMessage`: Targeted messages to specific windows

**Supported Actions:**
- Mouse: click, double-click, right-click, move, drag, scroll
- Keyboard: type, key press, key combo

---

## Protocol Output

### Render Tree Message

```json
{
  "type": "render_tree",
  "frame_id": 12345,
  "timestamp": 1716576000000,
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x12345",
      "title": "Visual Studio Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "position": {"x": 100, "y": 50},
      "z_order": 2,
      "focused": true,
      "process": "Code.exe",
      "elements": [
        {
          "id": "e_001",
          "type": "menu_bar",
          "name": "File",
          "bounds": {"x": 0, "y": 30, "w": 1920, "h": 25},
          "state": ["enabled", "visible"],
          "children": [
            {
              "id": "e_002",
              "type": "menu_item",
              "name": "File",
              "bounds": {"x": 0, "y": 30, "w": 50, "h": 25}
            },
            {
              "id": "e_003",
              "type": "menu_item",
              "name": "Edit",
              "bounds": {"x": 50, "y": 30, "w": 50, "h": 25}
            }
          ]
        },
        {
          "id": "e_010",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "state": ["enabled", "visible"]
        }
      ]
    }
  ],
  "mouse": {
    "x": 540,
    "y": 320,
    "hovered_element_id": "e_010"
  }
}
```

### Field Mapping

| UIA Property | OSCP Field |
|--------------|------------|
| `CurrentName` | `name` |
| `CurrentControlType` | `type` |
| `CurrentBoundingRectangle` | `bounds` |
| `CurrentIsEnabled` | `state` (enabled/disabled) |
| `CurrentHasKeyboardFocus` | `state` (focused) |

---

## Coverage

| App Type | Works? | Method |
|----------|--------|--------|
| Win32 apps | ✅ Yes | UIA + Win32 |
| WPF apps | ✅ Yes | UIA |
| UWP apps (modern) | ✅ Yes | UIA |
| WinForms | ⚠️ Partial | UIA (basic) |
| Electron (Chromium) | ✅ Yes | UIA |
| DirectX apps | ✅ Yes | UIA (chrome frame) |
| OpenGL/Vulkan apps | ✅ Yes | UIA (chrome frame) |
| Legacy GDI apps | ⚠️ Limited | Win32 only |
| Java Swing | ⚠️ Variable | UIA (if enabled) |

**Coverage: ~90% of desktop apps**

---

## Limitations

| Limitation | Description |
|------------|-------------|
| Non-UIA apps | Legacy apps without accessibility support |
| Secure input | UAC dialogs, password fields (OS protected) |
| DRM content | Protected content not accessible |
| Remote Desktop | App runs on remote machine |

---

## Protocol Implementation

### Transport

Default: `tcp://localhost:9876`

### Message Framing

Length-prefixed JSON:
```
[4-byte big-endian length][JSON payload]
```

### Capabilities

```json
{
  "platform": "windows",
  "driver": "oscp-windows-v0.2",
  "capture_methods": ["win32", "uia"],
  "input_methods": ["sendinput", "postmessage"],
  "capabilities": ["render_tree", "actions", "events"],
  "features": {
    "element_tree": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Installation & Startup

### Installation

```powershell
# Single command install (V1 target)
winget install OSCP.Windows
```

### Startup

1. OSCP service starts (Windows service or auto-launch)
2. Agent connects via TCP
3. Driver sends `welcome` message
4. Element tree streaming begins

---

## Configuration

```json
{
  "windows": {
    "capture": {
      "frame_rate": 30,
      "methods": ["uia", "win32"],
      "poll_uia": true,
      "poll_win32": true
    },
    "input": {
      "method": "sendinput",
      "action_delay_ms": 0
    }
  }
}
```

---

## V2: Render Operations (Future)

For Windows V2, we plan to explore DWM DirectX hook:

**Approach:** Inject into DWM.exe, hook D3D11 device methods

**Challenges:**
- DWM is a protected system process
- Internal structures undocumented
- May break on Windows updates

**Status:** Experimental, deferred from V1.

---

## Status

🚧 **V1 Target:** UIA + Win32 for element tree + basic actions

---

## References

- [UI Automation](https://learn.microsoft.com/en-us/windows/win32/winauto/ui-automation)
- [Win32 Window Enumeration](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumwindows)
- [SendInput](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput)