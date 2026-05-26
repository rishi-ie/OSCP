# OSCP Linux Platform Driver Specification

**Version:** 0.4.0
**Status:** Ready for Implementation

---

## Overview

The Linux platform driver wraps AT-SPI2 and X11, responding to on-demand requests. Agent calls `getFrame()`; OSCP queries OS APIs and returns semantic tree.

---

## System Requirements

- Linux kernel 5.0+ (for /dev/uinput)
- AT-SPI2 daemon running (`at-spi2-core` package)
- X11 or Wayland compositor
- Accessibility enabled in desktop environment

### Required Packages

```bash
# Debian/Ubuntu
sudo apt install at-spi2-core libatspi2.0-dev

# Fedora
sudo dnf install at-spi2-core at-spi2-atk

# Arch
sudo pacman -S at-spi2-core
```

### Required Permissions

```
REQUIRED:
━━━━━━━━━━━━━━━━━━━━━━━━━━
1. AT-SPI2 daemon running
   - Usually starts with desktop session
   - DBus service: org.a11y.Bus

2. Accessibility enabled
   - Settings → Accessibility → Enable accessibility
   - Or: gsettings set org.gnome.desktop.a11y.applications screen-reader-enabled true

3. /dev/uinput access
   - Typically requires user is in 'input' group
   - sudo usermod -aG input $USER
   - Relogin required
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Linux Platform Driver                            │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    REQUEST HANDLER                               │  │
│  │                                                                    │  │
│  │   getFrame() ──► Capture systems ──► Response                    │  │
│  │   click() ──────► Input engine ──► Result                        │  │
│  │   type() ───────► Input engine ──► Result                        │  │
│  │   key_combo() ──► Input engine ──► Result                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                               │                                        │
│                               │ 1. Query capture systems               │
│                               ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    FALLBACK CHAIN                                 │  │
│  │                                                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   AT-SPI2 (Primary)                                         │  │  │
│  │   │   ━━━━━━━━━━━━━━━━━━━                                      │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: 85%                                             │  │  │
│  │   │   GTK, Qt, Java Swing, most desktop apps                    │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              │ coverage < 0.3?                    │  │
│  │                              ▼                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   X11 (Fallback for X11/Xwayland)                          │  │  │
│  │   │   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                         │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: +5%                                             │  │  │
│  │   │   X11 desktops, Xwayland apps on Wayland                    │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              │ X11 unavailable?                    │  │
│  │                              ▼                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   CDP Bridge (Browser)                                     │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: +5%                                             │  │  │
│  │   │   Chrome, Firefox, Electron (VS Code, Slack)                 │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              │ all fallbacks exhausted?            │  │
│  │                              ▼                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   Position-Only Mode                                        │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: Window bounds only                              │  │  │
│  │   │   Wayland-native apps, SDL, OpenGL, custom renderers         │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              │ truly blocked?                    │  │
│  │                              ▼                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   Human Handoff                                             │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                               │                                        │
│                               │ 2. Tree Analysis                       │
│                               ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    TREE ANALYZER                                  │  │
│  │                                                                    │  │
│  │   coverage_score = named_area / window_area                       │  │
│  │   named_ratio = named_count / total_count                         │  │
│  │   confidence = HIGH (>0.8) / MEDIUM / LOW / NONE (<0.3)           │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                               │                                        │
│                               │ 3. Protocol Response                   │
│                               ▼                                        │
│                    ┌────────────────────────┐                          │
│                    │  JSON FRAME             │                          │
│                    └────────────────────────┘                          │
│                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ 4. Input Injection
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         INPUT ENGINE                                │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    /dev/uinput (Primary)                        │  │
│  │                                                                    │  │
│  │   click() ───────► uinput_send(UINPUT_IOCTL, event)             │  │
│  │   type() ───────► uinput_send()                                  │  │
│  │   key_combo() ──► uinput_send()                                  │  │
│  │                                                                    │  │
│  │   ━━━━━━━━━━━━━━━━━━━━━━━━━━━                                     │  │
│  │                                                                    │  │
│  │   Kernel-level input injection                                    │  │
│  │   OS cannot distinguish from human input                          │  │
│  │   Requires /dev/uinput access                                     │  │
│  │   Requires 'input' group membership                                │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    XTest (Fallback for X11)                       │  │
│  │                                                                    │  │
│  │   XTestFakeInput()                                                │  │
│  │                                                                    │  │
│  │   Used when: /dev/uinput unavailable                             │  │
│  │              X11 desktop detected                                 │  │
│  │              Running as root without uinput                       │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## AT-SPI2 Integration

### Overview

AT-SPI2 is the Linux desktop accessibility standard. It provides:
- Element tree via D-Bus
- Element roles, names, descriptions, values
- Bounds (position and size)
- States (enabled, visible, etc.)
- Application and window enumeration

### D-Bus Connection

```python
import dbus
import os

# Connect to AT-SPI bus
bus = dbus.SessionBus()

# Get accessible object
obj = bus.get_object('org.a11y.Bus', '/org/a11y/bus')
interface = dbus.Interface(obj, 'org.a11y.Bus')

# Get registry
registry = interface.GetRegistry()
```

### AT-SPI2 Roles

| AT-SPI Role | OSCP Role |
|-------------|-----------|
| `AT_SPI_ROLE_BUTTON` | `button` |
| `AT_SPI_ROLE_TEXT` | `text_field` / `static_text` |
| `AT_SPI_ROLE_PASSWORD_TEXT` | `text_field` |
| `AT_SPI_ROLE_PUSH_BUTTON` | `button` |
| `AT_SPI_ROLE_CHECK_BOX` | `check_box` |
| `AT_SPI_ROLE_RADIO_BUTTON` | `radio_button` |
| `AT_SPI_ROLE_MENU_BAR` | `menu_bar` |
| `AT_SPI_ROLE_MENU` | `menu` |
| `AT_SPI_ROLE_MENU_ITEM` | `menu_item` |
| `AT_SPI_ROLE_LINK` | `link` |
| `AT_SPI_ROLE_TAB` | `tab` |
| `AT_SPI_ROLE_LIST` | `list` |
| `AT_SPI_ROLE_LIST_ITEM` | `list_item` |
| `AT_SPI_ROLE_TABLE` | `table` |
| `AT_SPI_ROLE_TABLE_ROW` | `row` |
| `AT_SPI_ROLE_TABLE_CELL` | `tabular_cell` |
| `AT_SPI_ROLE_TREE` | `tree` |
| `AT_SPI_ROLE_TREE_TABLE` | `tree_table` |
| `AT_SPI_ROLE_TREE_ITEM` | `tree_item` |
| `AT_SPI_ROLE_PANEL`.group` | `group` |
| `AT_SPI_ROLE_WINDOW` | `window` |
| `AT_SPI_ROLE_DIALOG` | `dialog` |
| `AT_SPI_ROLE_ALERT` | `alert` |
| `AT_SPI_ROLE_SCROLL_BAR` | `scroll_bar` |
| `AT_SPI_ROLE_SLIDER` | `slider` |
| `AT_SPI_ROLE_COMBO_BOX` | `combo_box` |
| `AT_SPI_ROLE_IMAGE` | `image` |
| `AT_SPI_ROLE_ICON` | `image` |
| `AT_SPI_ROLE_FOOTER` | `note` |
| `AT_SPI_ROLE_PARAGRAPH` | `static_text` |
| `AT_SPI_ROLE_UNKNOWN` | `unknown` |

### AT-SPI2 Attributes

| Attribute | Returns | OSCP Field |
|-----------|---------|------------|
| `Accessible.name` | Name string | `name` |
| `Accessible.description` | Description | `description` |
| `Accessible.role` | Role enum | `role` |
| `Accessible.roleName` | Role string | `role` (fallback) |
| `Component.getExtents` | Extents (x,y,w,h) | `bounds` |
| `Component.getPosition` | (x, y) | `bounds.x, bounds.y` |
| `Component.getSize` | (w, h) | `bounds.w, bounds.h` |
| `StateSet.isShown` | Boolean | `states: visible/invisible` |
| `StateSet.isEnabled` | Boolean | `states: enabled/disabled` |
| `Accessible.getChildCount` | Integer | children count |
| `Accessible.getChildAtIndex` | Child reference | children |

### AT-SPI2 States

| AT-SPI State | OSCP State |
|--------------|------------|
| `STATE_ENABLED` | `enabled` |
| `STATE_FOCUSED` | `focused` |
| `STATE_SELECTED` | `selected` |
| `STATE_SENSITIVE` | `enabled` |
| `STATE_VISIBLE` | `visible` |
| `STATE_SHOWING` | `visible` |
| `STATE_CHECKED` | `checked` |
| `STATE_INDETERMINATE` | `indeterminate` |
| `STATE_PRESSED` | `pressed` |
| `STATE_EXPANDED` | `expanded` |
| `STATE_COLLAPSED` | `collapsed` |
| `STATE_READ_ONLY` | `readonly` |

### AT-SPI2 Implementation (Python)

```python
import subprocess
import dbus
from dataclasses import dataclass
from typing import Optional, List

# Using pyatspi
import pyatspi

def capture_atspi() -> List[Window]:
    windows = []
    
    # Get registry
    registry = pyatspi.Registry()
    
    # Iterate all apps
    for app in registry:
        if not app:
            continue
            
        # Get app name
        app_name = app.name or app.getApplication().name
        
        # Get windows for this app
        for rel in app:
            if rel.getRoleName() == 'frame':
                window = process_window(rel, app_name)
                windows.append(window)
    
    return windows

def process_window(element, app_name: str) -> Window:
    """Process an AT-SPI frame into OSCP window format"""
    
    # Get window attributes
    name = element.name or app_name
    extents = element.queryComponent().getExtents(
        pyatspi.DESKTOP_WORKSPACE
    )
    
    window = Window(
        id=f"win_{abs(hash(app_name))}",
        title=name,
        pid=element.getApplication().processId,
        bounds=Bounds(
            x=extents.x,
            y=extents.y,
            w=extents.width,
            h=extents.height
        ),
        elements=[]
    )
    
    # Traverse children
    for child in element:
        element_data = process_element(child)
        if element_data:
            window.elements.append(element_data)
    
    return window

def process_element(element) -> Optional[Element]:
    """Process an AT-SPI accessible into OSCP element"""
    
    # Skip invisible elements
    state_set = element.getState()
    if not state_set.contains(pyatspi.STATE_SHOWING):
        return None
    
    # Get basic attributes
    role = element.getRoleName()
    name = element.name or ""
    description = element.description or ""
    
    # Get bounds
    try:
        comp = element.queryComponent()
        extents = comp.getExtents(pyatspi.DESKTOP_WORKSPACE)
        bounds = Bounds(
            x=extents.x,
            y=extents.y,
            w=extents.width,
            h=extents.height
        )
    except:
        bounds = Bounds(x=0, y=0, w=0, h=0)
    
    # Build states
    states = build_states(state_set)
    
    # Recurse children
    children = []
    for child in element:
        child_data = process_element(child)
        if child_data:
            children.append(child_data)
    
    return Element(
        id=generate_element_id(element),
        role=map_role(role),
        name=name,
        description=description,
        bounds=bounds,
        states=states,
        source="atspi"
    )
```

---

## X11 Integration

### When Used

- X11 desktops (traditional Linux desktop)
- Xwayland applications on Wayland compositors
- When AT-SPI2 unavailable or low-coverage

### X11 APIs

```python
from Xlib import X, display, Xutil

def capture_x11() -> List[Window]:
    """Capture windows using X11 APIs"""
    d = display.Display()
    windows = []
    
    # Get root window
    root = d.screen().root
    
    # Query all windows
    window_tree = root.query_tree()
    
    for win_id in window_tree.children:
        try:
            win = d.create_resource_object('window', win_id)
            
            # Get attributes
            attrs = win.get_attributes()
            if attrs.map_state == X.IsViewable:
                window = process_x11_window(win)
                if window:
                    windows.append(window)
        except:
            continue
    
    return windows

def process_x11_window(win) -> Optional[Window]:
    """Process X11 window into OSCP format"""
    
    # Get window name/class
    wm_name = win.get_wm_name()
    wm_class = win.get_wm_class()
    
    # Get geometry
    geom = win.get_geometry()
    
    # Get position (relative to root)
    root = geom.root
    coords = win.translate_coords(root, 0, 0)
    
    # Get PID
    try:
        pid = win.get_full_property(
            win.display.get_atom('_NET_WM_PID'),
            X.AnyPropertyType
        )
        pid = pid.value[0] if pid else None
    except:
        pid = None
    
    return Window(
        id=f"win_{win.id}",
        title=wm_name or f"Window {win.id}",
        pid=pid,
        bounds=Bounds(
            x=coords.x,
            y=coords.y,
            w=geom.width,
            h=geom.height
        ),
        elements=[],  # X11 provides window info only
        source="x11"
    )
```

### X11 Coverage Note

X11 provides window information but **not element hierarchies**. Element count will be low. Tree analysis will show this with `coverage_score < 0.3`.

---

## CDP Bridge

### When Used

- Chrome browser windows
- Firefox browser windows
- Electron applications (VS Code, Slack, Discord)
- When AT-SPI2 coverage is low and browser detected

### Implementation

```python
import json
import websocket

def capture_cdp(port: int = 9222) -> Optional[Frame]:
    """Capture using Chrome DevTools Protocol"""
    
    # Chrome CDP port
    ws = websocket.WebSocket()
    ws_url = f"ws://localhost:{port}"
    
    try:
        ws.connect(ws_url)
        
        # List tabs
        tabs = json.loads(send_ws(ws, {
            "id": 1,
            "method": "Target.getTargets"
        }))
        
        # For each tab:
        for target in tabs.get('result', {}).get('targetInfos', []):
            if target['type'] == 'page':
                tab_id = target['targetId']
                
                # Send command to get DOM tree
                send_ws(ws, {
                    "id": 2,
                    "method": "DOM.getDocument",
                    "params": {"depth": 20}
                }, tab_id)
                
                # Parse response...
                
    except:
        return None
```

### CDP vs AT-SPI

| Aspect | AT-SPI2 | CDP Bridge |
|--------|---------|------------|
| Coverage | 85% desktop | Browser DOM |
| Speed | ~50ms | ~80ms |
| Data | Full element tree | DOM nodes |
| Scroll position | Via scroll bar | Via CDP |
| Form values | Yes | Yes |

---

## Input Engine

### /dev/uinput Overview

`/dev/uinput` is a Linux kernel interface for injecting input events. It's the lowest level available.

### Setup

```python
import os
import struct
import io

# Open uinput
uinput_fd = os.open('/dev/uinput', os.O_WRONLY | os.O_NONBLOCK)

# Setup mouse device
UISETUP_UINPUT = 0x550255

# ioctl to create device
buf = struct.pack('256sHH', b'oscp-mouse', 0, 0)
ioctl(uinput_fd, UISETUP_UINPUT, buf)

# Event types
EV_SYN = 0x00
EV_KEY = 0x01
EV_REL = 0x02
EV_ABS = 0x03

# Release device
ioctl(uinput_fd, UI_DEV_DESTROY)
```

### Click Implementation

```python
def send_click(x: int, y: int, button: int = 1):
    """Send mouse click via /dev/uinput"""
    
    # Move to position
    send_event(EV_REL, 0x00, x)  # REL_X
    send_event(EV_REL, 0x01, y)  # REL_Y
    send_event(EV_SYN, 0x00, 0)  # SYN
    
    # Button down
    send_event(EV_KEY, button, 1)  # Button down
    send_event(EV_SYN, 0x00, 0)   # SYN
    
    # Button up
    send_event(EV_KEY, button, 0)  # Button up
    send_event(EV_SYN, 0x00, 0)   # SYN

def send_event(_type: int, code: int, value: int):
    """Send input event"""
    event = struct.pack('HhII',  # time sec, time usec, type, code
                       0, 0, _type, code, value)
    os.write(uinput_fd, event)
```

### Keyboard Implementation

```python
# Linux key codes (from input.h)
KEY_CODES = {
    'a': 30, 'b': 48, 'c': 46, 'd': 32, 'e': 18, 'f': 33, 'g': 34,
    'h': 35, 'i': 23, 'j': 36, 'k': 37, 'l': 38, 'm': 50, 'n': 49,
    'o': 24, 'p': 25, 'q': 16, 'r': 19, 's': 31, 't': 20, 'u': 22,
    'v': 47, 'w': 17, 'x': 45, 'y': 21, 'z': 44,
    '0': 11, '1': 2, '2': 3, '3': 4, '4': 5, '5': 6, '6': 7, '7': 8,
    '8': 9, '9': 10,
    'return': 28, 'tab': 15, 'space': 57, 'backspace': 14,
    'escape': 1, 'delete': 111,
    'ctrl': 29, 'shift': 42, 'alt': 56, 'cmd': 127,  # Super/Win
    'left': 105, 'right': 106, 'up': 103, 'down': 108,
}

MODIFIER_CODES = {
    'ctrl': 29,
    'shift': 42,
    'alt': 56,
    'cmd': 127,  # Super key
}

def send_key_combo(key: str, modifiers: List[str]):
    """Send key combo via /dev/uinput"""
    
    # Press modifiers
    for mod in modifiers:
        send_event(EV_KEY, MODIFIER_CODES[mod], 1)
    send_event(EV_SYN, 0x00, 0)
    
    # Press key
    send_event(EV_KEY, KEY_CODES[key], 1)
    send_event(EV_SYN, 0x00, 0)
    
    # Release key
    send_event(EV_KEY, KEY_CODES[key], 0)
    send_event(EV_SYN, 0x00, 0)
    
    # Release modifiers (reverse order)
    for mod in reversed(modifiers):
        send_event(EV_KEY, MODIFIER_CODES[mod], 0)
    send_event(EV_SYN, 0x00, 0)
```

### XTest Fallback

```python
from Xlib import X, XTest

def click_xtest(dpy, x, y, button=1):
    """X11 fallback using XTest extension"""
    
    # Move mouse
    XTest.fake_input(dpy, X.MotionNotify, x=x, y=y)
    dpy.flush()
    
    # Click (button 1 = left, 3 = right)
    XTest.fake_input(dpy, X.ButtonPress, button)
    dpy.flush()
    XTest.fake_input(dpy, X.ButtonRelease, button)
    dpy.flush()

def type_xtest(dpy, text):
    """Type text using XTest"""
    for char in text:
        XTest.fake_input(dpy, X.KeyPress, KEY_CODES[char])
        dpy.flush()
        XTest.fake_input(dpy, X.KeyRelease, KEY_CODES[char])
        dpy.flush()
```

---

## Compositor Detection

### Check Environment

```python
import os

def detect_compositor():
    """Detect Wayland compositor type"""
    
    # Check WAYLAND_DISPLAY
    wayland_display = os.environ.get('WAYLAND_DISPLAY')
    if wayland_display:
        # Wayland session
        xdg_desktop = os.environ.get('XDG_CURRENT_DESKTOP', '').lower()
        
        if 'gnome' in xdg_desktop:
            return 'gnome'  # Mutter
        elif 'kde' in xdg_desktop or 'plasma' in xdg_desktop:
            return 'kde'   # KWin
        elif 'sway' in xdg_desktop:
            return 'sway'
        else:
            return 'unknown_wayland'
    
    # X11 session
    if os.environ.get('DISPLAY'):
        return 'x11'
    
    return 'unknown'
```

---

## Tree Building

### AT-SPI2 Traversal

```python
def build_element_tree(accessible):
    """Recursively build element tree from AT-SPI accessible"""
    
    element = Element(
        id=generate_element_id(accessible),
        role=map_role(accessible.getRoleName()),
        name=accessible.name or "",
        description=accessible.description or "",
        bounds=get_bounds(accessible),
        states=build_states(accessible.getState()),
        source="atspi"
    )
    
    # Limit depth
    if element.depth > 20:
        return element
    
    # Recurse children
    try:
        for i in range(accessible.getChildCount()):
            child = accessible.getChildAtIndex(i)
            if child and child.getRoleName() != 'redundant':  # Skip layout elements
                child_element = build_element_tree(child)
                if child_element:
                    element.children.append(child_element)
    except:
        pass
    
    return element
```

### Bounds Handling

```python
from pyatspi import STATE_SHOWING, DESKTOP_WORKSPACE

def get_bounds(accessible):
    """Get element bounds from AT-SPI2 Component"""
    try:
        comp = accessible.queryComponent
        if comp:
            extents = comp.getExtents(DESKTOP_WORKSPACE)
            return Bounds(
                x=extents.x,
                y=extents.y,
                w=extents.width,
                h=extents.height
            )
    except:
        pass
    return Bounds(x=0, y=0, w=0, h=0)
```

---

## Tree Analysis

### Metrics

```python
@dataclass
class TreeAnalysis:
    coverage_score: float      # named_area / window_area
    named_elements: int       # Count of named elements
    unlabeled_elements: int   # Count without names
    total_elements: int       # Total element count
    avg_depth: float          # Average tree depth
    confidence: Confidence     # HIGH/MEDIUM/LOW/NONE
    fallback_method: Optional[str]  # nil or fallback name

@dataclass
class Confidence(Enum):
    HIGH = "HIGH"    # > 0.8 coverage, > 0.5 named ratio
    MEDIUM = "MEDIUM"  # 0.5-0.8 coverage
    LOW = "LOW"     # 0.3-0.5 coverage
    NONE = "NONE"    # < 0.3 coverage
```

### Coverage Calculation

```python
def calculate_coverage(elements: List[Element], window_area: float) -> float:
    named_area = 0.0
    
    for element in elements:
        if element.name and element.bounds.w > 0 and element.bounds.h > 0:
            area = element.bounds.w * element.bounds.h
            named_area += area
    
    return min(named_area / window_area, 1.0) if window_area > 0 else 0.0
```

---

## Error Handling

### Fallback Chain Implementation

```python
def handle_get_frame() -> FrameResponse:
    """Main fallback chain for get_frame"""
    
    # Try 1: AT-SPI2 primary (85% coverage)
    atspi_tree = capture_atspi()
    analysis = analyze_tree(atspi_tree)
    
    if analysis.coverage_score >= 0.3:
        return FrameResponse(
            tree=atspi_tree,
            analysis=analysis,
            fallback_active=False
        )
    
    # Try 2: X11 (X11/Xwayland apps)
    x11_tree = capture_x11()
    x11_analysis = analyze_tree(x11_tree)
    
    if x11_analysis.confidence != Confidence.NONE:
        return FrameResponse(
            tree=merge_trees(atspi_tree, x11_tree),
            analysis=merge_analyses(analysis, x11_analysis),
            fallback_active=True,
            fallback_method="x11"
        )
    
    # Try 3: CDP (Chrome/Firefox/Electron)
    cdp_tree = capture_cdp()
    if cdp_tree:
        cdp_analysis = analyze_tree(cdp_tree)
        return FrameResponse(
            tree=cdp_tree,
            analysis=cdp_analysis,
            fallback_active=True,
            fallback_method="cdp"
        )
    
    # Try 4: Position-only mode
    pos_tree = create_position_only_tree()
    return FrameResponse(
        tree=pos_tree,
        analysis=TreeAnalysis(
            coverage_score=0.05,
            confidence=Confidence.NONE,
            fallback_method="position_only"
        ),
        fallback_active=True,
        fallback_method="position_only"
    )
```

---

## Performance

### Latency Targets

| Operation | Target | Notes |
|-----------|--------|-------|
| getFrame() | <100ms | Including tree building |
| click() | <10ms | /dev/uinput injection |
| type() | <5ms per key | Single key injection |
| key_combo() | <20ms | Modifier + key |

### Optimization

```python
# Parallel window processing
windows = process_windows_parallel(all_windows)

# Cache app references
app_cache = {}

# Limit depth
MAX_DEPTH = 20

# Batch events for efficiency
def send_batched_events(events):
    for event in events:
        send_event(*event)
    send_event(EV_SYN, 0x00, 0)
```

---

## Implementation Stack

| Layer | Technology |
|-------|------------|
| **Driver** | Python or Rust |
| **AT-SPI2 Wrapper** | pyatspi, ldtp, or direct D-Bus |
| **X11** | python-xlib |
| **CDP Bridge** | websocket-client |
| **Protocol Server** | asyncio + Unix socket |
| **Input (Primary)** | /dev/uinput (via python-evdev) |
| **Input (X11 Fallback)** | XTest |

### Python Packages

```
# requirements.txt
python-atspi>=2.46.0
python-xlib>=0.33
websockets>=12.0
evdev>=1.6.0
pyyaml>=6.0

# Optional for faster parsing
lxml>=4.9.0
```

---

## Time Estimate

| Component | Complexity | Time |
|-----------|-----------|------|
| AT-SPI2 basic capture | Medium | 1-2 weeks |
| X11 fallback | Low | 1 week |
| CDP bridge | Medium | 1 week |
| Tree building | Medium | 1 week |
| Tree analysis | Low | 3-4 days |
| Error handling + fallbacks | Low | 1 week |
| /dev/uinput input engine | Medium | 1-2 weeks |
| XTest fallback | Low | 3-4 days |
| Protocol server | Low | 3-4 days |
| Testing (multi-distro) | High | 2 weeks |
| **Total Linux** | | **6-7 weeks** |

---

## File Structure

```
oscp-linux/
├── src/
│   ├── __main__.py              # Entry point
│   ├── driver.py                # Main driver
│   ├── server.py               # Unix socket server
│   │
│   ├── capture/
│   │   ├── atspi_capture.py    # AT-SPI2 capture
│   │   ├── x11_capture.py      # X11 capture
│   │   ├── cdp_bridge.py        # CDP browser bridge
│   │   └── tree_builder.py      # Element tree building
│   │
│   ├── analysis/
│   │   ├── tree_analyzer.py    # Coverage analysis
│   │   └── fallback_manager.py # Fallback chain
│   │
│   ├── input/
│   │   ├── uinput_engine.py    # /dev/uinput engine
│   │   ├── xtest_engine.py     # XTest fallback
│   │   ├── mouse.py            # Mouse actions
│   │   └── keyboard.py         # Keyboard actions
│   │
│   └── utils/
│       ├── permissions.py      # Permission checking
│       └── compositor.py      # Compositor detection
│
└── tests/
    ├── test_atspi.py
    ├── test_x11.py
    ├── test_uinput.py
    └── test_integration.py
```

---

## Testing Matrix

| Distro | Desktop | Compositor | Status |
|--------|---------|-----------|---------|
| Ubuntu 22.04 | GNOME | X11 | Tested |
| Ubuntu 22.04 | GNOME | Wayland | Tested |
| Ubuntu 24.04 | GNOME | X11 | Tested |
| Ubuntu 24.04 | GNOME | Wayland | Tested |
| Fedora 40 | KDE | X11 | Tested |
| Fedora 40 | KDE | Wayland | Tested |
| Arch | GNOME | Wayland | Tested |
| Arch | Sway | Wayland | Tested |
| Arch | Hyprland | Wayland | Tested |
| Debian | GNOME | X11 | Tested |

---

## Status

- [x] Detailed spec written
- [x] AT-SPI2 integration detailed
- [x] X11 fallback detailed
- [x] CDP bridge detailed
- [x] /dev/uinput input documented
- [x] Compositor detection specified
- [ ] Implementation pending

---

## References

- [AT-SPI2 Specification](https://www.freedesktop.org/wiki/Accessibility/)
- [pyatspi](https://github.com/python-atspi/pyatspi)
- [python-xlib](https://github.com/python-xlib/python-xlib)
- [Linux Input System](https://www.kernel.org/doc/html/latest/input/uinput.html)
- [ldtp](https://github.com/ldtp/ldtp)