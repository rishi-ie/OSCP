# OSCP macOS Platform Driver Specification

**Version:** 0.4.0
**Status:** Ready for Implementation

---

## Overview

The macOS platform driver wraps AXUIElement and responds to on-demand requests. Agent calls `getFrame()`; OSCP queries AXUIElement and returns semantic tree.

---

## System Requirements

- macOS 12.0+ (Monterey or later)
- Screen Recording permission (for AXUIElement access)
- Accessibility permission (for input injection)

### Permissions

```
REQUIRED PERMISSIONS:
━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Screen Recording
   Purpose: Enables AXUIElement access
   
2. Accessibility
   Purpose: Enables CGEvent input injection
   
How to grant:
1. System Preferences → Privacy & Security → Screen Recording
   → Add: OSCP
   
2. System Preferences → Privacy & Security → Accessibility
   → Add: OSCP
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    macOS Platform Driver                                │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    REQUEST HANDLER                               │  │
│  │                                                                    │  │
│  │   Receives: getFrame(), click(), type(), key_combo()              │  │
│  │   Routes to: capture systems or error handler                     │  │
│  │   Returns: JSON frame or action result                            │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                               │                                        │
│                               │ 1. Query AXUIElement                  │
│                               ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    FALLBACK CHAIN                                 │  │
│  │                                                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   AXUIElement (Primary)                                     │  │  │
│  │   │   ━━━━━━━━━━━━━━━━━━━━━                                     │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: 90%                                             │  │  │
│  │   │   AppKit, SwiftUI, standard macOS controls                  │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              │ coverage < 0.3?                   │  │
│  │                              ▼                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   CDP Bridge (Safari / Chrome / Electron)                   │  │  │
│  │   │   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                    │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: +4%                                             │  │  │
│  │   │   Safari WebKit, Chrome DevTools, VS Code, Slack             │  │  │
│  │   │                                                             │  │  │
│  │   └─────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              │ all fallbacks exhausted?            │  │
│  │                              ▼                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────┐  │  │
│  │   │                                                             │  │  │
│  │   │   Position-Only Mode                                        │  │  │
│  │   │   ━━━━━━━━━━━━━━━━━━━                                       │  │  │
│  │   │                                                             │  │  │
│  │   │   Coverage: Window bounds only                              │  │  │
│  │   │   Metal/OpenGL apps, custom renderers                       │  │  │
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
│                               │ 2. Tree Analysis                      │
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
│                    │  {                       │                          │
│                    │    "windows": [...],     │                          │
│                    │    "confidence": "HIGH"  │                          │
│                    │  }                       │                          │
│                    └────────────────────────┘                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ 4. Input Injection
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         INPUT ENGINE                                    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    CGEvent (CGEventtap)                           │  │
│  │                                                                    │  │
│  │   click() ───────► CGEventCreateMouseEvent()                      │  │
│  │                              │                                    │  │
│  │   type() ───────► CGEventCreateKeyboardEvent()                   │  │
│  │                              │                                    │  │
│  │   key_combo() ──► CGEventCreateKeyboardEvent()                   │  │
│  │                              │                                    │  │
│  │   CGEventPost(kCGHIDEventTarget, event)                          │  │
│  │                              │                                    │  │
│  │   ━━━━━━━━━━━━━━━━━━━━━━━━━                                     │  │
│  │                                                                    │  │
│  │   OS cannot distinguish from human input                          │  │
│  │   Works even with Accessibility permissions denied (for click)   │  │
│  │   Requires Accessibility permission for AXUIElement capture       │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## AXUIElement Integration

### Overview

AXUIElement is Apple's native accessibility API. It provides:
- Full element tree for running applications
- Element roles, names, descriptions, values
- Bounds (position and size)
- States (enabled, focused, etc.)
- Application and window enumeration

### How to Query

```c
// 1. Get system accessibility state
AXUIElementSetEnabled(kAXTrustedAccessibilityLayer(), true);

// 2. Get system-wide element
AXUIElementRef systemWide = AXUIElementCreateSystemWide();

// 3. Get focused application
pid_t pid;
AXUIElementCopyAttributeValue(systemWide, kAXFocusedApplicationAttribute, &appRef);

// 4. Get window hierarchy
AXUIElementCopyAttributeValue(appRef, kAXWindowsAttribute, &windowsRef);

// 5. Get element children recursively
AXUIElementCopyAttributeValue(element, kAXChildrenAttribute, &childrenRef);
```

### AXUIElement Attributes

| Attribute | Returns | OSCP Field |
|-----------|---------|------------|
| `kAXRoleAttribute` | Role string | `role` |
| `kAXSubroleAttribute` | Subrole string | `subrole` |
| `kAXTitleAttribute` | Title string | `name` |
| `kAXDescriptionAttribute` | Description | `description` |
| `kAXValueAttribute` | Value string | `value` |
| `kAXPositionAttribute` | CGPoint | `bounds.x, bounds.y` |
| `kAXSizeAttribute` | CGSize | `bounds.w, bounds.h` |
| `kAXChildrenAttribute` | Children array | `children` |
| `kAXEnabledAttribute` | Boolean | `states: enabled/disabled` |
| `kAXFocusedAttribute` | Boolean | `states: focused` |
| `kAXVisibleAttribute` | Boolean | `states: visible/invisible` |

### Role Mapping

| AXUIElement Role | OSCP Role |
|------------------|-----------|
| `AXButton` | `button` |
| `AXTextField` | `text_field` |
| `AXTextArea` | `text_area` |
| `AXMenuItem` | `menu_item` |
| `AXMenu` | `menu` |
| `AXMenuBar` | `menu_bar` |
| `AXStaticText` | `static_text` |
| `AXLink` | `link` |
| `AXCheckbox` | `check_box` |
| `AXRadioButton` | `radio_button` |
| `AXComboBox` | `combo_box` |
| `AXList` | `list` |
| `AXListItem` | `list_item` |
| `AXTabGroup` | `tab_group` |
| `AXTab` | `tab` |
| `AXTable` | `table` |
| `AXTableRow` | `row` |
| `AXTableCell` | `tabular_cell` |
| `AXGroup` | `group` |
| `AXWindow` | `window` |
| `AXSheet` | `sheet` |
| `AXDialog` | `dialog` |
| `AXPopover` | `popover` |
| `AXToolbar` | `toolbar` |
| `AXStaticText` | `static_text` |
| `AXImage` | `image` |
| `AXUnknown` | `unknown` |

### Bounds Calculation

```c
// AXUIElement gives position + size separately
CGPoint position;
CGSize size;

AXUIElementCopyAttributeValue(element, kAXPositionAttribute, &position);
AXUIElementCopyAttributeValue(element, kAXSizeAttribute, &size);

// OSCP bounds format
{
  "x": position.x,
  "y": position.y,
  "w": size.width,
  "h": size.height
}
```

### Element ID Generation

```c
// Generate unique, stable element IDs
// Format: "e_{appPid}_{elementHash}"

// Hash based on element's unique attributes
element_hash = hash(role + name + position + size)

// Example:
// e_1234_a1b2c3d4
```

---

## CDP Bridge

### When Used

- Safari browser windows
- Google Chrome windows
- Electron apps (VS Code, Slack, Discord, etc.)
- When AXUIElement coverage is below 0.3

### Implementation

```swift
// Connect to Safari WebKit debugger
// Local CDP port for Safari: 9222
// Chrome/Electron: typically 9222 + offset per profile

// CDP Runtime.evaluate to get DOM tree
// Parse bounding boxes from CDP.Rect

// Map CDP nodes to OSCP element format
// Use "cdp" as source
```

### CDP vs AXUIElement

| Aspect | AXUIElement | CDP Bridge |
|--------|-------------|------------|
| Coverage | 90% | Browser DOM |
| Speed | ~20ms | ~50ms |
| Data | Full element tree | DOM nodes |
| Scroll position | Yes | Yes (via CDP) |
| Form values | Yes | Yes |

---

## Input Engine

### CGEvent Overview

CGEvent is Core Graphics Event API. It injects input at the lowest level:
- Mouse movements, clicks, drags
- Keyboard input, key combinations
- Scroll wheel events

### Click Implementation

```c
// Create mouse click event
CGEventRef clickEvent = CGEventCreateMouseEvent(
    NULL,                         // Event source (NULL = current)
    kCGEventLeftMouseDown,         // Event type
    CGPointMake(x, y),           // Position
    kCGMouseButtonLeft            // Button
);

// Set click type (down only, or move+down+up for single click)
CGEventSetIntegerValueField(clickEvent, kCGMouseEventClickState, 1);

// Post the event
CGEventPost(kCGHIDEventTarget, clickEvent);

// Release
CFRelease(clickEvent);
```

### Type Implementation

```c
// Create keyboard event for character
CGEventRef keyEvent = CGEventCreateKeyboardEvent(
    NULL,
    keyCode,                     // Virtual key code
    true                          // Key down (false for key up)
);

// Set character value (for text input)
CGEventKeyboardSetUnicodeString(keyEvent, length, characters);

// Post event
CGEventPost(kCGHIDEventTarget, keyEvent);
```

### Key Code Mapping

```swift
// macOS virtual key codes (partial)
let keyCodes: [String: Int] = [
    "a": 0x00, "s": 0x01, "d": 0x02, "f": 0x03,
    "h": 0x04, "g": 0x05, "z": 0x06, "x": 0x07,
    "c": 0x08, "v": 0x09, "b": 0x0B, "q": 0x0C,
    "w": 0x0D, "e": 0x0E, "r": 0x0F, "y": 0x10,
    "t": 0x11, "1": 0x12, "2": 0x13, "3": 0x14,
    "4": 0x15, "6": 0x16, "5": 0x17, "9": 0x19,
    "7": 0x1A, "8": 0x1C, "0": 0x1D, "o": 0x1F,
    "u": 0x20, "i": 0x22, "p": 0x23, "l": 0x25,
    "j": 0x26, "k": 0x28, "n": 0x2D, "m": 0x2E,
    "return": 0x24, "tab": 0x30, "space": 0x31,
    "delete": 0x33, "escape": 0x35, "command": 0x37,
    "shift": 0x38, "caps_lock": 0x39, "option": 0x3A,
    "ctrl": 0x3B, "right_shift": 0x3C, "right_option": 0x3D,
    "right_ctrl": 0x3E, "right_command": 0x36,
    "up": 0x7E, "down": 0x7D, "left": 0x7B, "right": 0x7C,
    "f1": 0x7A, "f2": 0x78, "f3": 0x76, "f4": 0x60,
    "f5": 0x61, "f6": 0x62, "f7": 0x63, "f12": 0x6F
]
```

### Key Combo Implementation

```swift
// Ctrl+S
let modifiers = CGEventFlags.maskControl
let keyCode = 0x0D // 's'

// Create key down with modifier
let keyDown = CGEventCreateKeyboardEvent(NULL, keyCode, true)!
CGEventSetFlags(keyDown, modifiers)

// Create key up
let keyUp = CGEventCreateKeyboardEvent(NULL, keyCode, false)!

// Post both
CGEventPost(kCGHIDEventTarget, keyDown)
CGEventPost(kCGHIDEventTarget, keyUp)
```

### Modifiers Mapping

```swift
CGEventFlags:
- maskCommand = Cmd key (macOS)
- maskControl = Ctrl key
- maskAlternate = Option/Alt key
- maskShift = Shift key
- maskSecondaryFn = Function key
```

---

## Tree Building

### Process

```
1. Get list of running applications
   → NSWorkspace.shared.runningApplications
   
2. For each application:
   a. Get AXUIElement for app
   
3. For each window:
   a. Get AXUIElement for window
   b. Extract window attributes (title, bounds, focused)
   
4. For each window, traverse element tree:
   a. Get children from kAXChildrenAttribute
   b. For each child element:
      - Extract role, name, description, value
      - Extract bounds (position + size)
      - Extract states (enabled, visible, focused)
      - Recurse into children
      
5. Generate element IDs

6. Build tree structure

7. Calculate tree analysis metrics
```

### Depth Handling

```swift
// Limit depth to prevent runaway recursion
let maxDepth = 20

func traverse(element: AXUIElement, depth: Int) -> Element? {
    guard depth < maxDepth else { return nil }
    
    // Process element...
    
    // Recurse children
    for child in children {
        traverse(element: child, depth: depth + 1)
    }
}

// Skip certain container types (scroll areas, etc.)
let skipRoles = ["ruler", "layout_area", "growth_area"]
```

### Filter Elements

```swift
// Filter out invisible or non-interactive elements
func shouldInclude(_ element: AXUIElement) -> Bool {
    // Check visible state
    guard isVisible(element) else { return false }
    
    // Check if actionable (has role + can be interacted with)
    guard isActionable(element) else { return false }
    
    return true
}
```

---

## Tree Analysis

### Metrics

```swift
struct TreeAnalysis {
    coverageScore: Float      // named_area / window_area
    namedElements: Int       // Count of named elements
    unlabeledElements: Int   // Count without names
    totalElements: Int       // Total element count
    avgDepth: Float          // Average tree depth
    confidence: Confidence   // HIGH/MEDIUM/LOW/NONE
}

enum Confidence {
    case HIGH    // > 0.8 coverage, > 0.5 named ratio
    case MEDIUM  // 0.5-0.8 coverage
    case LOW     // 0.3-0.5 coverage
    case NONE    // < 0.3 coverage
}
```

### Coverage Calculation

```swift
func calculateCoverage(elements: [Element], windowArea: Float) -> Float {
    var namedArea: Float = 0
    
    for element in elements where element.name != nil {
        let area = element.bounds.w * element.bounds.h
        namedArea += area
    }
    
    return namedArea / windowArea
}
```

---

## Error Handling

### Axel Hierarchy Implementation

```swift
func handleGetFrame() -> FrameResponse {
    // Try 1: AXUIElement primary
    let tree = captureAXUIElement()
    
    if tree.analysis.coverageScore >= 0.3 {
        return FrameResponse(tree: tree)
    }
    
    // Try 2: CDP Bridge (if browser window)
    if let cdpTree = tryCDPBridge() {
        if cdpTree.analysis.coverageScore >= 0.3 {
            return FrameResponse(tree: cdpTree)
        }
    }
    
    // Try 3: Position-only mode
    let positionTree = createPositionOnlyTree()
    return FrameResponse(tree: positionTree, fallbackActive: true)
}
```

### Element Not Found

```swift
// When target element not in tree
if let element = findElement(byName: "Save") {
    return element
} else {
    // Search nearby positions
    let candidates = findByProximity(lastKnownPosition)
    
    if candidates.isEmpty {
        return ErrorResponse(
            code: "ELEMENT_NOT_FOUND",
            message: "Could not find Save button",
            reasoning: "Element may have moved or app state changed"
        )
    }
}
```

---

## Performance

### Latency Targets

| Operation | Target | Notes |
|-----------|--------|-------|
| getFrame() | <50ms | Including tree building |
| click() | <10ms | Input injection |
| type() | <10ms per key | Single key injection |
| key_combo() | <20ms | Modifier + key |

### Optimization

```swift
// Cache AXUIElement references (expensive to create)
var elementCache: [String: AXUIElement] = [:]

// Limit children traversal depth
let maxDepth = 20

// Skip non-actionable elements early
if !isActionable(element) {
    return nil
}

// Parallel window processing
let windows = application.windows
let processedWindows = windows.map { processWindow($0) }
```

---

## Implementation Stack

| Layer | Technology |
|-------|------------|
| **Driver** | Swift or Objective-C |
| **AXUIElement** | Native C API via Swift bridging |
| **CDP Bridge** | WebSocket + CDP protocol |
| **Protocol Server** | Unix socket + JSON/stdlib |
| **Input Engine** | CGEvent (Core Graphics) |

---

## Time Estimate

| Component | Complexity | Time |
|-----------|-----------|------|
| AXUIElement basic capture | Medium | 1 week |
| Tree building + element mapping | Medium | 1 week |
| Tree analysis | Low | 3-4 days |
| CDP bridge | Medium | 1 week |
| Error handling + fallbacks | Low | 3-4 days |
| CGEvent input engine | Low | 1 week |
| Protocol server | Low | 3-4 days |
| Permissions handling | Low | 2-3 days |
| Testing + polish | Medium | 1 week |
| **Total macOS** | | **4-5 weeks** |

---

## Dependencies

### Swift Package Manager

```
dependencies: [
    .package(url: "https://github.com/apple/swift-nio", from: "2.60.0"),
]
```

### System Frameworks

```swift
import ApplicationServices  // AXUIElement
import CoreGraphics       // CGEvent
import Cocoa              // App/Window enumeration
import Foundation         // Networking
```

---

## Testing

### Test Applications

| App | What to Test |
|-----|-------------|
| Finder | All standard controls |
| Safari | Browser + CDP |
| VS Code | Text editing + tree |
| Terminal | Text field + scrolling |
| Xcode | Complex hierarchy |
| Slack | Electron + CDP |

### Permission Testing

```swift
// Check permissions
let hasAX = AXIsProcessTrusted()
let hasScreenRecording = hasScreenCapturePermission()

// If not authorized, guide user
if !hasAX {
    presentPermissionGuide()
}
```

---

## File Structure

```
oscp-macos/
├── Sources/
│   ├── main.swift                     # Entry point
│   ├── Driver/
│   │   ├── Driver.swift               # Main driver class
│   │   ├── RequestHandler.swift       # Request routing
│   │   └── ProtocolServer.swift       # Unix socket server
│   │
│   ├── Capture/
│   │   ├── AXUIElementCapture.swift   # AXUIElement queries
│   │   ├── CDPBridge.swift             # Safari/Chrome CDP
│   │   └── TreeBuilder.swift           # Element tree building
│   │
│   ├── Analysis/
│   │   ├── TreeAnalyzer.swift         # Coverage analysis
│   │   └── FallbackManager.swift      # Fallback chain
│   │
│   ├── Input/
│   │   ├── CGEventEngine.swift        # Input injection
│   │   ├── MouseInput.swift           # Mouse actions
│   │   └── KeyboardInput.swift        # Keyboard actions
│   │
│   └── Models/
│       ├── Frame.swift                # Frame response model
│       ├── Element.swift             # Element model
│       └── Action.swift              # Action model
│
├── Resources/
│   └── Info.plist
│
└── Tests/
    └── OSCPTests/
        ├── CaptureTests.swift
        ├── InputTests.swift
        └── IntegrationTests.swift
```

---

## Status

- [x] Detailed spec written
- [x] AXUIElement integration detailed
- [x] CDP bridge detailed
- [x] CGEvent input documented
- [x] Error handling specified
- [ ] Implementation pending

---

## References

- [AXUIElement Documentation](https://developer.apple.com/documentation/application-services)
- [CGEvent Reference](https://developer.apple.com/documentation/coregraphics/cgevent)
- [pyax](https://github.com/pyax/pyax) - Python reference implementation
- [AXUIElement C Header](https://developer.apple.com/reference/application-services/1462335-element_ref)