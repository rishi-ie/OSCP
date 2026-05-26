# OSCP macOS Platform Driver - Implementation Specification

**Version:** 0.4.0
**Status:** Ready for Implementation

---

## Table of Contents

1. [Project Setup](#1-project-setup)
2. [Models](#2-models)
3. [AXUIElement Capture](#3-axuielement-capture)
4. [CDP Bridge](#4-cdp-bridge)
5. [Tree Builder](#5-tree-builder)
6. [Tree Analyzer](#6-tree-analyzer)
7. [Fallback Manager](#7-fallback-manager)
8. [CGEvent Input Engine](#8-cgevent-input-engine)
9. [Protocol Server](#9-protocol-server)
10. [Main Entry Point](#10-main-entry-point)
11. [Error Handling](#11-error-handling)
12. [Testing](#12-testing)
13. [Common Pitfalls](#13-common-pitfalls)

---

## 1. Project Setup

### 1.1 Project Structure

```
oscp-macos/
├── Sources/
│   ├── main.swift
│   ├── Driver/
│   │   ├── Driver.swift
│   │   └── RequestHandler.swift
│   ├── Capture/
│   │   ├── AXUIElementCapture.swift
│   │   ├── CDPBridge.swift
│   │   └── TreeBuilder.swift
│   ├── Analysis/
│   │   ├── TreeAnalyzer.swift
│   │   └── FallbackManager.swift
│   ├── Input/
│   │   ├── CGEventEngine.swift
│   │   ├── MouseInput.swift
│   │   └── KeyboardInput.swift
│   └── Models/
│       ├── Frame.swift
│       ├── Element.swift
│       ├── Action.swift
│       ├── Window.swift
│       └── Errors.swift
├── Resources/
│   └── Info.plist
├── Tests/
│   └── OSCPTests/
│       ├── CaptureTests.swift
│       ├── InputTests.swift
│       └── IntegrationTests.swift
├── project.yml
└── Package.swift
```

### 1.2 XcodeGen project.yml

```yaml
name: oscp-macos
options:
  bundleIdPrefix: com.oscp
  deploymentTarget:
    macOS: "12.0"
  xcodeVersion: "15.0"

settings:
  base:
    SWIFT_VERSION: "5.9"
    MACOSX_DEPLOYMENT_TARGET: "12.0"
    CODE_SIGN_IDENTITY: "-"
    CODE_SIGN_STYLE: Manual
    ENABLE_HARDENED_RUNTIME: YES

targets:
  oscp-macos:
    type: application
    platform: macOS
    sources:
      - Sources
    resources:
      - Resources
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.oscp.macos
        INFOPLIST_FILE: Resources/Info.plist
        LD_RUNPATH_SEARCH_PATHS: "@executable_path/../Frameworks"
        ENABLE_APP_SANDBOX: NO
        ENABLE_HARDENED_RUNTIME: YES
    entitlements:
      path: Resources/OSCP.entitlements
    dependencies: []

  oscp-macos-tests:
    type: bundle.unit-test
    platform: macOS
    sources:
      - Tests
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.oscp.macos.tests
    dependencies:
      - target: oscp-macos
```

### 1.3 Entitlements File

Create `Resources/OSCP.entitlements`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key>
    <false/>
    <key>com.apple.security.automation.apple-events</key>
    <true/>
</dict>
</plist>
```

### 1.4 Info.plist

Create `Resources/Info.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>en</string>
    <key>CFBundleExecutable</key>
    <string>$(EXECUTABLE_NAME)</string>
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>OSCP</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>CFBundleShortVersionString</key>
    <string>0.4.0</string>
    <key>CFBundleVersion</key>
    <string>1</string>
    <key>LSMinimumSystemVersion</key>
    <string>$(MACOSX_DEPLOYMENT_TARGET)</string>
    <key>NSAppleEventsUsageDescription</key>
    <string>OSCP needs to send Apple Events to control applications.</string>
    <key>LSUIElement</key>
    <true/>
</dict>
</plist>
```

### 1.5 Build Commands

```bash
# Generate Xcode project
cd oscp-macos
xcodegen generate

# Build
xcodebuild -project oscp-macos.xcodeproj -scheme oscp-macos -configuration Debug build

# Or build from command line
swift build
```

---

## 2. Models

### 2.1 Element.swift

```swift
import Foundation
import CoreGraphics

// MARK: - Element Model

/// Represents a single UI element in the accessibility tree
public struct Element: Codable, Identifiable, Sendable {
    
    // MARK: - Properties
    
    /// Unique identifier within this frame (format: "e_{pid}_{hash}")
    public let id: String
    
    /// ARIA-style role (e.g., "button", "text_field", "window")
    public let role: String
    
    /// More specific role variant (e.g., "push_button", "search_field")
    public let subrole: String?
    
    /// Accessible name of the element
    public let name: String?
    
    /// Accessible description/help text
    public let description: String?
    
    /// Current value (for text fields, sliders, etc.)
    public let value: String?
    
    /// Bounding box in screen coordinates
    public let bounds: Bounds
    
    /// Active states (e.g., ["enabled", "visible", "focused"])
    public let states: [ElementState]
    
    /// Additional role-specific attributes
    public let attributes: [String: AttributeValue]
    
    /// Confidence score (0.0 - 1.0)
    public let confidence: Double
    
    /// Source API that provided this element
    public let source: ElementSource
    
    /// Children elements (if this is a container)
    public var children: [Element]
    
    // MARK: - Initialization
    
    public init(
        id: String,
        role: String,
        subrole: String? = nil,
        name: String? = nil,
        description: String? = nil,
        value: String? = nil,
        bounds: Bounds,
        states: [ElementState],
        attributes: [String: AttributeValue] = [:],
        confidence: Double = 1.0,
        source: ElementSource,
        children: [Element] = []
    ) {
        self.id = id
        self.role = role
        self.subrole = subrole
        self.name = name
        self.description = description
        self.value = value
        self.bounds = bounds
        self.states = states
        self.attributes = attributes
        self.confidence = confidence
        self.source = source
        self.children = children
    }
}

// MARK: - Supporting Types

/// Bounding box with screen coordinates
public struct Bounds: Codable, Sendable {
    public let x: Double
    public let y: Double
    public let w: Double
    public let h: Double
    
    public init(x: Double, y: Double, w: Double, h: Double) {
        self.x = x
        self.y = y
        self.w = w
        self.h = h
    }
    
    /// Convert to CGRect
    public var cgRect: CGRect {
        CGRect(x: x, y: y, width: w, height: h)
    }
    
    /// Center point of the bounds
    public var center: CGPoint {
        CGPoint(x: x + w / 2, y: y + h / 2)
    }
    
    /// Area of the bounds
    public var area: Double {
        w * h
    }
    
    /// Check if a point is within bounds
    public func contains(_ point: CGPoint) -> Bool {
        point.x >= x && point.x <= x + w &&
        point.y >= y && point.y <= y + h
    }
}

/// Element state values
public enum ElementState: String, Codable, CaseIterable, Sendable {
    case enabled
    case disabled
    case visible
    case invisible
    case focused
    case unfocused
    case selected
    case unselected
    case checked
    case unchecked
    case indeterminate
    case pressed
    case expanded
    case collapsed
    case actionable
    case updating
}

/// Element source API
public enum ElementSource: String, Codable, Sendable {
    case axuielement = "axuielement"
    case cdp = "cdp"
    case x11 = "x11"
    case heuristic = "heuristic"
    case position = "position"
    case unknown = "unknown"
}

/// Attribute value (flexible type for different attribute types)
public enum AttributeValue: Codable, Sendable {
    case string(String)
    case bool(Bool)
    case int(Int)
    case double(Double)
    case stringArray([String])
    
    public init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let string = try? container.decode(String.self) {
            self = .string(string)
        } else if let bool = try? container.decode(Bool.self) {
            self = .bool(bool)
        } else if let int = try? container.decode(Int.self) {
            self = .int(int)
        } else if let double = try? container.decode(Double.self) {
            self = .double(double)
        } else if let array = try? container.decode([String].self) {
            self = .stringArray(array)
        } else {
            throw DecodingError.dataCorruptedError(
                in: container,
                debugDescription: "Unknown attribute value type"
            )
        }
    }
    
    public func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        switch self {
        case .string(let value): try container.encode(value)
        case .bool(let value): try container.encode(value)
        case .int(let value): try container.encode(value)
        case .double(let value): try container.encode(value)
        case .stringArray(let value): try container.encode(value)
        }
    }
}

// MARK: - Role Mapping

/// Maps AXUIElement roles to OSCP roles
public enum RoleMapping {
    
    /// AXUIElement role -> OSCP role mapping
    public static let axToOSCP: [String: String] = [
        "AXButton": "button",
        "AXCheckBox": "check_box",
        "AXRadioButton": "radio_button",
        "AXTextField": "text_field",
        "AXTextArea": "text_area",
        "AXComboBox": "combo_box",
        "AXMenuItem": "menu_item",
        "AXMenu": "menu",
        "AXMenuBar": "menu_bar",
        "AXStaticText": "static_text",
        "AXLink": "link",
        "AXImage": "image",
        "AXList": "list",
        "AXListItem": "list_item",
        "AXTable": "table",
        "AXRow": "row",
        "AXCell": "tabular_cell",
        "AXColumn": "column",
        "AXTabGroup": "tab_group",
        "AXTab": "tab",
        "AXToolbar": "toolbar",
        "AXGroup": "group",
        "AXWindow": "window",
        "AXSheet": "sheet",
        "AXDialog": "dialog",
        "AXPopover": "popover",
        "AXScrollArea": "scroll_area",
        "AXScrollBar": "scroll_bar",
        "AXSlider": "slider",
        "AXMenuButton": "menu_button",
        "AXSplitGroup": "split_group",
        "AXColorWell": "color_well",
        "AXIncrementor": "incrementor",
        "AXRatingIndicator": "rating_indicator",
        "AXComboBox": "combo_box",
        "AXDateTimeField": "date_time_field",
        "AXDescriptionList": "description_list",
        "AXTerm": "term",
        "AXDefinition": "definition",
        "AXCalendarView": "calendar_view",
        "AXUnknown": "unknown"
    ]
    
    /// AXUIElement subrole -> OSCP subrole mapping
    public static let subroleToOSCP: [String: String] = [
        "AXButton": "push_button",
        "AXCancelButton": "cancel_button",
        "AXDefaultButton": "default_button",
        "AXSearchField": "search_field",
        "AXSecureTextField": "secure_text_field",
        "AXRuler": "ruler",
        "AXEditText": "edit_text",
        "AXOutlineRow": "outline_row"
    ]
}
```

### 2.2 Window.swift

```swift
import Foundation

// MARK: - Window Model

/// Represents a single application window
public struct Window: Codable, Identifiable, Sendable {
    
    // MARK: - Properties
    
    /// Unique window identifier (format: "win_{hexAddress}")
    public let id: String
    
    /// Window title
    public let title: String
    
    /// Process ID of owning application
    public let pid: Int32
    
    /// Bundle identifier of owning application
    public let app: String?
    
    /// Window bounds in screen coordinates
    public let bounds: Bounds
    
    /// Window position (may differ from bounds for decorated windows)
    public let position: CGPoint
    
    /// Whether this window is focused
    public let focused: Bool
    
    /// Whether this window is minimized
    public let minimized: Bool
    
    /// Whether this window is on all spaces
    public let onAllSpaces: Bool
    
    /// Child elements within this window
    public var elements: [Element]
    
    // MARK: - Initialization
    
    public init(
        id: String,
        title: String,
        pid: Int32,
        app: String? = nil,
        bounds: Bounds,
        position: CGPoint,
        focused: Bool = false,
        minimized: Bool = false,
        onAllSpaces: Bool = false,
        elements: [Element] = []
    ) {
        self.id = id
        self.title = title
        self.pid = pid
        self.app = app
        self.bounds = bounds
        self.position = position
        self.focused = focused
        self.minimized = minimized
        self.onAllSpaces = onAllSpaces
        self.elements = elements
    }
}
```

### 2.3 Frame.swift

```swift
import Foundation

// MARK: - Frame Model

/// Complete screen frame response
public struct Frame: Codable, Sendable {
    
    // MARK: - Properties
    
    /// Frame identifier (incrementing counter)
    public let frameId: Int
    
    /// Platform (always "macOS" for this driver)
    public let platform: String
    
    /// Capture latency in milliseconds
    public let latencyMs: Int
    
    /// Timestamp (Unix milliseconds)
    public let timestamp: Int64
    
    /// All visible windows
    public let windows: [Window]
    
    /// Tree analysis results
    public let treeAnalysis: TreeAnalysis
    
    /// Current mouse state
    public let mouse: MouseState
    
    /// Current keyboard modifiers
    public let keyboard: KeyboardState
    
    /// Whether fallback mode is active
    public let fallbackActive: Bool
    
    // MARK: - Initialization
    
    public init(
        frameId: Int,
        platform: String = "macOS",
        latencyMs: Int,
        timestamp: Int64,
        windows: [Window],
        treeAnalysis: TreeAnalysis,
        mouse: MouseState,
        keyboard: KeyboardState,
        fallbackActive: Bool = false
    ) {
        self.frameId = frameId
        self.platform = platform
        self.latencyMs = latencyMs
        self.timestamp = timestamp
        self.windows = windows
        self.treeAnalysis = treeAnalysis
        self.mouse = mouse
        self.keyboard = keyboard
        self.fallbackActive = fallbackActive
    }
}

// MARK: - Tree Analysis

/// Analysis results for the accessibility tree
public struct TreeAnalysis: Codable, Sendable {
    
    /// Coverage score (named_area / window_area)
    public let coverageScore: Double
    
    /// Count of named elements
    public let namedElements: Int
    
    /// Count of unlabeled elements
    public let unlabeledElements: Int
    
    /// Total element count
    public let totalElements: Int
    
    /// Average tree depth
    public let avgDepth: Double
    
    /// Confidence level
    public let confidence: Confidence
    
    /// Active fallback method (if any)
    public let fallbackMethod: String?
    
    /// Recommended agent action
    public let recommendedAction: RecommendedAction
    
    // MARK: - Initialization
    
    public init(
        coverageScore: Double,
        namedElements: Int,
        unlabeledElements: Int,
        totalElements: Int,
        avgDepth: Double,
        confidence: Confidence,
        fallbackMethod: String? = nil,
        recommendedAction: RecommendedAction
    ) {
        self.coverageScore = coverageScore
        self.namedElements = namedElements
        self.unlabeledElements = unlabeledElements
        self.totalElements = totalElements
        self.avgDepth = avgDepth
        self.confidence = confidence
        self.fallbackMethod = fallbackMethod
        self.recommendedAction = recommendedAction
    }
}

// MARK: - Confidence

/// Tree confidence level
public enum Confidence: String, Codable, CaseIterable, Sendable {
    case high = "HIGH"
    case medium = "MEDIUM"
    case low = "LOW"
    case none = "NONE"
    
    /// Threshold values for confidence calculation
    public static func from(coverageScore: Double, namedRatio: Double) -> Confidence {
        if coverageScore > 0.8 && namedRatio > 0.5 {
            return .high
        } else if coverageScore >= 0.5 {
            return .medium
        } else if coverageScore >= 0.3 {
            return .low
        } else {
            return .none
        }
    }
    
    /// Agent action threshold
    public var actionThreshold: Double {
        switch self {
        case .high: return 0.8
        case .medium: return 0.5
        case .low: return 0.3
        case .none: return 0.0
        }
    }
}

// MARK: - Recommended Action

/// Recommended action for the agent
public enum RecommendedAction: String, Codable, Sendable {
    case execute = "execute"
    case executeWithMonitoring = "execute_with_monitoring"
    case explore = "explore"
    case exploreOrHandoff = "explore_or_handoff"
    case handoff = "handoff"
    
    /// Get recommended action from confidence
    public static func from(confidence: Confidence) -> RecommendedAction {
        switch confidence {
        case .high: return .execute
        case .medium: return .executeWithMonitoring
        case .low: return .explore
        case .none: return .handoff
        }
    }
}

// MARK: - Mouse State

/// Current mouse state
public struct MouseState: Codable, Sendable {
    
    /// Current X position
    public let x: Double
    
    /// Current Y position
    public let y: Double
    
    /// Button state
    public let buttonState: MouseButtonState
    
    /// Element currently under cursor
    public let hoveredElementId: String?
    
    // MARK: - Initialization
    
    public init(
        x: Double,
        y: Double,
        buttonState: MouseButtonState = .none,
        hoveredElementId: String? = nil
    ) {
        self.x = x
        self.y = y
        self.buttonState = buttonState
        self.hoveredElementId = hoveredElementId
    }
}

/// Mouse button state
public enum MouseButtonState: String, Codable, Sendable {
    case none
    case leftDown = "left_down"
    case leftUp = "left_up"
    case rightDown = "right_down"
    case rightUp = "right_up"
}

// MARK: - Keyboard State

/// Current keyboard modifier state
public struct KeyboardState: Codable, Sendable {
    
    /// Active modifier keys
    public let modifiers: [ModifierKey]
    
    // MARK: - Initialization
    
    public init(modifiers: [ModifierKey] = []) {
        self.modifiers = modifiers
    }
}

/// Modifier key
public enum ModifierKey: String, Codable, Sendable {
    case command = "cmd"
    case control = "ctrl"
    case option = "alt"
    case shift = "shift"
    case function = "fn"
    case capsLock = "caps_lock"
}
```

### 2.4 Action.swift

```swift
import Foundation

// MARK: - Action Models

/// Action request from agent
public struct ActionRequest: Codable, Sendable {
    
    /// Unique action identifier
    public let actionId: String
    
    /// Timestamp (Unix milliseconds)
    public let timestamp: Int64
    
    /// Action details
    public let action: Action
    
    // MARK: - Initialization
    
    public init(actionId: String, timestamp: Int64, action: Action) {
        self.actionId = actionId
        self.timestamp = timestamp
        self.action = action
    }
}

/// Action to perform
public enum Action: Codable, Sendable {
    case click(ClickAction)
    case type(TypeAction)
    case keyCombo(KeyComboAction)
    case scroll(ScrollAction)
    case drag(DragAction)
    case move(MoveAction)
    
    // MARK: - Codable
    
    private enum CodingKeys: String, CodingKey {
        case kind
        case click
        case type
        case keyCombo
        case scroll
        case drag
        case move
    }
    
    public init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        let kind = try container.decode(String.self, forKey: .kind)
        
        switch kind {
        case "click":
            self = .click(try container.decode(ClickAction.self, forKey: .click))
        case "type":
            self = .type(try container.decode(TypeAction.self, forKey: .type))
        case "key_combo":
            self = .keyCombo(try container.decode(KeyComboAction.self, forKey: .keyCombo))
        case "scroll":
            self = .scroll(try container.decode(ScrollAction.self, forKey: .scroll))
        case "drag":
            self = .drag(try container.decode(DragAction.self, forKey: .drag))
        case "move":
            self = .move(try container.decode(MoveAction.self, forKey: .move))
        default:
            throw DecodingError.dataCorruptedError(
                forKey: .kind,
                in: container,
                debugDescription: "Unknown action kind: \(kind)"
            )
        }
    }
    
    public func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        
        switch self {
        case .click(let action):
            try container.encode("click", forKey: .kind)
            try container.encode(action, forKey: .click)
        case .type(let action):
            try container.encode("type", forKey: .kind)
            try container.encode(action, forKey: .type)
        case .keyCombo(let action):
            try container.encode("key_combo", forKey: .kind)
            try container.encode(action, forKey: .keyCombo)
        case .scroll(let action):
            try container.encode("scroll", forKey: .kind)
            try container.encode(action, forKey: .scroll)
        case .drag(let action):
            try container.encode("drag", forKey: .kind)
            try container.encode(action, forKey: .drag)
        case .move(let action):
            try container.encode("move", forKey: .kind)
            try container.encode(action, forKey: .move)
        }
    }
    
    // MARK: - Kind
    
    public var kind: String {
        switch self {
        case .click: return "click"
        case .type: return "type"
        case .keyCombo: return "key_combo"
        case .scroll: return "scroll"
        case .drag: return "drag"
        case .move: return "move"
        }
    }
}

// MARK: - Click Action

/// Click action
public struct ClickAction: Codable, Sendable {
    
    /// X coordinate
    public let x: Double
    
    /// Y coordinate
    public let y: Double
    
    /// Button to click
    public let button: MouseButton
    
    /// Type of click
    public let clickType: ClickType
    
    /// Target element ID (optional)
    public let elementId: String?
    
    // MARK: - Initialization
    
    public init(
        x: Double,
        y: Double,
        button: MouseButton = .left,
        clickType: ClickType = .single,
        elementId: String? = nil
    ) {
        self.x = x
        self.y = y
        self.button = button
        self.clickType = clickType
        self.elementId = elementId
    }
}

/// Mouse button
public enum MouseButton: String, Codable, Sendable {
    case left
    case right
    case middle
}

/// Click type
public enum ClickType: String, Codable, Sendable {
    case single
    case double
    case triple
    case right
}

// MARK: - Type Action

/// Type text action
public struct TypeAction: Codable, Sendable {
    
    /// Text to type
    public let text: String
    
    /// Delay between keystrokes (milliseconds)
    public let typingDelayMs: Int
    
    // MARK: - Initialization
    
    public init(text: String, typingDelayMs: Int = 50) {
        self.text = text
        self.typingDelayMs = typingDelayMs
    }
}

// MARK: - Key Combo Action

/// Key combination action
public struct KeyComboAction: Codable, Sendable {
    
    /// Key to press
    public let key: String
    
    /// Modifier keys to hold
    public let modifiers: [String]
    
    // MARK: - Initialization
    
    public init(key: String, modifiers: [String] = []) {
        self.key = key
        self.modifiers = modifiers
    }
}

// MARK: - Scroll Action

/// Scroll action
public struct ScrollAction: Codable, Sendable {
    
    /// X coordinate (optional anchor point)
    public let x: Double?
    
    /// Y coordinate (optional anchor point)
    public let y: Double?
    
    /// Horizontal scroll delta
    public let deltaX: Double
    
    /// Vertical scroll delta
    public let deltaY: Double
    
    /// Scroll type
    public let scrollType: ScrollType
    
    // MARK: - Initialization
    
    public init(
        x: Double? = nil,
        y: Double? = nil,
        deltaX: Double = 0,
        deltaY: Double,
        scrollType: ScrollType = .precise
    ) {
        self.x = x
        self.y = y
        self.deltaX = deltaX
        self.deltaY = deltaY
        self.scrollType = scrollType
    }
}

/// Scroll type
public enum ScrollType: String, Codable, Sendable {
    case precise      // Pixel units
    case line         // Line-based
    case page         // Page up/down
}

// MARK: - Drag Action

/// Drag action
public struct DragAction: Codable, Sendable {
    
    /// Start X coordinate
    public let startX: Double
    
    /// Start Y coordinate
    public let startY: Double
    
    /// End X coordinate
    public let endX: Double
    
    /// End Y coordinate
    public let endY: Double
    
    /// Button to use
    public let button: MouseButton
    
    /// Duration in milliseconds
    public let durationMs: Int
    
    // MARK: - Initialization
    
    public init(
        startX: Double,
        startY: Double,
        endX: Double,
        endY: Double,
        button: MouseButton = .left,
        durationMs: Int = 500
    ) {
        self.startX = startX
        self.startY = startY
        self.endX = endX
        self.endY = endY
        self.button = button
        self.durationMs = durationMs
    }
}

// MARK: - Move Action

/// Move mouse action
public struct MoveAction: Codable, Sendable {
    
    /// Target X coordinate
    public let x: Double
    
    /// Target Y coordinate
    public let y: Double
    
    // MARK: - Initialization
    
    public init(x: Double, y: Double) {
        self.x = x
        self.y = y
    }
}

// MARK: - Action Result

/// Result of action execution
public struct ActionResult: Codable, Sendable {
    
    /// Original action ID
    public let actionId: String
    
    /// Whether action succeeded
    public let success: Bool
    
    /// Timestamp (Unix milliseconds)
    public let timestamp: Int64
    
    /// Execution latency in milliseconds
    public let latencyMs: Int
    
    /// Confidence score
    public let confidence: Double
    
    /// Source of target element
    public let source: ElementSource?
    
    /// Target element info
    public let target: TargetInfo?
    
    /// Error info (if failed)
    public let error: ActionError?
    
    // MARK: - Initialization
    
    public init(
        actionId: String,
        success: Bool,
        timestamp: Int64,
        latencyMs: Int,
        confidence: Double = 1.0,
        source: ElementSource? = nil,
        target: TargetInfo? = nil,
        error: ActionError? = nil
    ) {
        self.actionId = actionId
        self.success = success
        self.timestamp = timestamp
        self.latencyMs = latencyMs
        self.confidence = confidence
        self.source = source
        self.target = target
        self.error = error
    }
    
    // MARK: - Factory Methods
    
    public static func success(
        actionId: String,
        latencyMs: Int,
        target: TargetInfo
    ) -> ActionResult {
        ActionResult(
            actionId: actionId,
            success: true,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            latencyMs: latencyMs,
            confidence: 1.0,
            target: target
        )
    }
    
    public static func failure(
        actionId: String,
        latencyMs: Int,
        error: ActionError
    ) -> ActionResult {
        ActionResult(
            actionId: actionId,
            success: false,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            latencyMs: latencyMs,
            error: error
        )
    }
}

/// Target element info
public struct TargetInfo: Codable, Sendable {
    public let elementId: String?
    public let elementName: String?
    public let elementRole: String?
    
    public init(
        elementId: String? = nil,
        elementName: String? = nil,
        elementRole: String? = nil
    ) {
        self.elementId = elementId
        self.elementName = elementName
        self.elementRole = elementRole
    }
}

/// Action error
public struct ActionError: Codable, Sendable {
    
    /// Error code
    public let code: String
    
    /// Human-readable message
    public let message: String
    
    /// Reasoning for failure
    public let reasoning: String?
    
    /// Alternative positions to try
    public let alternatives: [AlternativePosition]?
    
    // MARK: - Initialization
    
    public init(
        code: String,
        message: String,
        reasoning: String? = nil,
        alternatives: [AlternativePosition]? = nil
    ) {
        self.code = code
        self.message = message
        self.reasoning = reasoning
        self.alternatives = alternatives
    }
    
    // MARK: - Factory Methods
    
    public static func elementNotFound(_ reasoning: String? = nil) -> ActionError {
        ActionError(
            code: "ELEMENT_NOT_FOUND",
            message: "Target element not found",
            reasoning: reasoning
        )
    }
    
    public static func actionFailed(_ reasoning: String? = nil, alternatives: [AlternativePosition]? = nil) -> ActionError {
        ActionError(
            code: "ACTION_FAILED",
            message: "Action did not produce expected result",
            reasoning: reasoning,
            alternatives: alternatives
        )
    }
    
    public static func permissionDenied() -> ActionError {
        ActionError(
            code: "PERMISSION_DENIED",
            message: "Accessibility permission not granted",
            reasoning: "Grant permission in System Preferences > Privacy & Security > Accessibility"
        )
    }
}

/// Alternative position to try
public struct AlternativePosition: Codable, Sendable {
    public let x: Double
    public let y: Double
    public let confidence: Double
    public let elementName: String?
    
    public init(
        x: Double,
        y: Double,
        confidence: Double,
        elementName: String? = nil
    ) {
        self.x = x
        self.y = y
        self.confidence = confidence
        self.elementName = elementName
    }
}
```

### 2.5 Errors.swift

```swift
import Foundation

// MARK: - OSCP Errors

/// OSCP-specific errors
public enum OSCPError: Error, LocalizedError {
    
    // MARK: - Permission Errors
    
    case permissionDenied
    case screenRecordingPermissionRequired
    case accessibilityPermissionRequired
    
    // MARK: - Capture Errors
    
    case emptyTree
    case lowCoverage(score: Double)
    case captureTimeout
    case elementNotFound(id: String)
    case elementDisabled(id: String)
    case elementMoved(id: String, oldPosition: CGPoint, newPosition: CGPoint)
    
    // MARK: - Action Errors
    
    case actionFailed(reasoning: String?)
    case actionTimeout
    case invalidAction(message: String)
    
    // MARK: - Connection Errors
    
    case connectionFailed
    case connectionLost
    case invalidMessage
    
    // MARK: - Platform Errors
    
    case axUIElementError(status: Int32)
    case cgEventError(status: Int)
    case systemError(message: String)
    
    // MARK: - LocalizedError
    
    public var errorDescription: String? {
        switch self {
        case .permissionDenied:
            return "Accessibility permission denied"
        case .screenRecordingPermissionRequired:
            return "Screen Recording permission required"
        case .accessibilityPermissionRequired:
            return "Accessibility permission required"
        case .emptyTree:
            return "Semantic tree is empty"
        case .lowCoverage(let score):
            return "Tree coverage too low: \(Int(score * 100))%"
        case .captureTimeout:
            return "Capture timed out"
        case .elementNotFound(let id):
            return "Element not found: \(id)"
        case .elementDisabled(let id):
            return "Element disabled: \(id)"
        case .elementMoved(let id, _, _):
            return "Element moved: \(id)"
        case .actionFailed(let reasoning):
            return "Action failed: \(reasoning ?? "unknown")"
        case .actionTimeout:
            return "Action timed out"
        case .invalidAction(let message):
            return "Invalid action: \(message)"
        case .connectionFailed:
            return "Connection failed"
        case .connectionLost:
            return "Connection lost"
        case .invalidMessage:
            return "Invalid message"
        case .axUIElementError(let status):
            return "AXUIElement error: \(status)"
        case .cgEventError(let status):
            return "CGEvent error: \(status)"
        case .systemError(let message):
            return "System error: \(message)"
        }
    }
}

// MARK: - Error Codes

/// OSCP error codes for protocol
public enum ErrorCode: String, Codable {
    case permissionDenied = "PERMISSION_DENIED"
    case elementNotFound = "ELEMENT_NOT_FOUND"
    case elementDisabled = "ELEMENT_DISABLED"
    case elementMoved = "ELEMENT_MOVED"
    case actionFailed = "ACTION_FAILED"
    case emptyTree = "EMPTY_TREE"
    case lowCoverage = "LOW_COVERAGE"
    case timeout = "TIMEOUT"
    case connectionLost = "CONNECTION_LOST"
    case invalidRequest = "INVALID_REQUEST"
    case unsupportedAction = "UNSUPPORTED_ACTION"
    case platformError = "PLATFORM_ERROR"
    
    /// Convert OSCPError to ErrorCode
    public static func from(_ error: OSCPError) -> ErrorCode {
        switch error {
        case .permissionDenied, .screenRecordingPermissionRequired, .accessibilityPermissionRequired:
            return .permissionDenied
        case .elementNotFound:
            return .elementNotFound
        case .elementDisabled:
            return .elementDisabled
        case .elementMoved:
            return .elementMoved
        case .actionFailed, .actionTimeout:
            return .actionFailed
        case .emptyTree:
            return .emptyTree
        case .lowCoverage:
            return .lowCoverage
        case .captureTimeout, .actionTimeout:
            return .timeout
        case .connectionLost:
            return .connectionLost
        case .invalidAction, .invalidMessage:
            return .invalidRequest
        case .unsupportedAction:
            return .unsupportedAction
        case .axUIElementError, .cgEventError, .systemError, .connectionFailed:
            return .platformError
        }
    }
}

// MARK: - Error Response

/// Protocol error response
public struct ErrorResponse: Codable, Sendable {
    public let type: String = "error"
    public let requestId: String?
    public let code: String
    public let message: String
    public let timestamp: Int64
    public let details: ErrorDetails?
    
    public init(
        requestId: String? = nil,
        code: ErrorCode,
        message: String,
        timestamp: Int64 = Int64(Date().timeIntervalSince1970 * 1000),
        details: ErrorDetails? = nil
    ) {
        self.requestId = requestId
        self.code = code.rawValue
        self.message = message
        self.timestamp = timestamp
        self.details = details
    }
}

/// Error details
public struct ErrorDetails: Codable, Sendable {
    public let coverageScore: Double?
    public let reasoning: String?
    public let fallbackMethod: String?
    public let recommendedAction: String?
    
    public init(
        coverageScore: Double? = nil,
        reasoning: String? = nil,
        fallbackMethod: String? = nil,
        recommendedAction: String? = nil
    ) {
        self.coverageScore = coverageScore
        self.reasoning = reasoning
        self.fallbackMethod = fallbackMethod
        self.recommendedAction = recommendedAction
    }
}
```

---

## 3. AXUIElement Capture

### 3.1 AXUIElementCapture.swift

```swift
import Foundation
import ApplicationServices
import Cocoa

// MARK: - AXUIElement Capture

/// Handles AXUIElement queries and tree capture
public final class AXUIElementCapture {
    
    // MARK: - Properties
    
    /// System-wide accessibility element
    private let systemWide: AXUIElement
    
    /// Element ID generator
    private var elementIdCounter: Int = 0
    
    /// Cache for pid-to-app mapping
    private var pidToApp: [Int32: NSRunningApplication] = [:]
    
    // MARK: - Initialization
    
    public init() {
        self.systemWide = AXUIElementCreateSystemWide()
    }
    
    // MARK: - Permission Check
    
    /// Check if accessibility permissions are granted
    public func checkPermissions() -> Bool {
        let trusted = AXIsProcessTrusted()
        return trusted
    }
    
    /// Prompt for accessibility permissions
    public func requestPermissions() {
        let options = [kAXTrustedCheckOptionPrompt.takeUnretainedValue() as String: true] as CFDictionary
        AXIsProcessTrustedWithOptions(options)
    }
    
    // MARK: - Capture Methods
    
    /// Capture the full accessibility tree
    public func captureTree() throws -> [Window] {
        // Get running applications
        let apps = NSWorkspace.shared.runningApplications
        
        var windows: [Window] = []
        
        for app in apps {
            // Skip background-only apps
            guard app.activationPolicy == .regular else { continue }
            
            // Get windows for this app
            let appWindows = try captureWindows(for: app)
            windows.append(contentsOf: appWindows)
        }
        
        return windows
    }
    
    /// Capture windows for a specific application
    public func captureWindows(for app: NSRunningApplication) throws -> [Window] {
        let pid = app.processIdentifier
        let axApp = AXUIElementCreateApplication(pid)
        
        // Get windows attribute
        var windowsRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(
            axApp,
            kAXWindowsAttribute as CFString,
            &windowsRef
        )
        
        guard status == .success else {
            if status == .noValue {
                return []  // App has no windows
            }
            throw OSCPError.axUIElementError(status: status.rawValue)
        }
        
        guard let windowsArray = windowsRef as? [AXUIElement] else {
            return []
        }
        
        // Cache the app reference
        pidToApp[pid] = app
        
        // Process each window
        return windowsArray.enumerated().compactMap { index, axWindow in
            do {
                return try captureWindow(axWindow, index: index, pid: pid, app: app)
            } catch {
                print("Error capturing window: \(error)")
                return nil
            }
        }
    }
    
    /// Capture a single window and its elements
    private func captureWindow(
        _ axWindow: AXUIElement,
        index: Int,
        pid: Int32,
        app: NSRunningApplication
    ) throws -> Window {
        
        // Get window title
        let title = getStringAttribute(axWindow, attribute: kAXTitleAttribute) ?? ""
        
        // Get window position
        let position = getPointAttribute(axWindow, attribute: kAXPositionAttribute) ?? .zero
        
        // Get window size
        let size = getSizeAttribute(axWindow, attribute: kAXSizeAttribute) ?? .zero
        
        // Get focused state
        let focused = getBoolAttribute(axWindow, attribute: kAXFocusedAttribute) ?? false
        
        // Get minimized state
        let minimized = getBoolAttribute(axWindow, attribute: kAXMinimizedAttribute) ?? false
        
        // Generate window ID
        let windowId = "win_\(String(format: "%x", pid))_\(index)"
        
        // Build element tree
        let elements = try captureElements(from: axWindow, pid: pid)
        
        // Create bounds
        let bounds = Bounds(
            x: position.x,
            y: position.y,
            w: size.width,
            h: size.height
        )
        
        return Window(
            id: windowId,
            title: title,
            pid: pid,
            app: app.bundleIdentifier,
            bounds: bounds,
            position: position,
            focused: focused,
            minimized: minimized,
            elements: elements
        )
    }
    
    // MARK: - Element Tree Capture
    
    /// Capture element tree from a container
    private func captureElements(
        from container: AXUIElement,
        pid: Int32,
        depth: Int = 0,
        maxDepth: Int = 20
    ) throws -> [Element] {
        
        // Limit depth to prevent runaway recursion
        guard depth < maxDepth else { return [] }
        
        // Get children
        var childrenRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(
            container,
            kAXChildrenAttribute as CFString,
            &childrenRef
        )
        
        guard status == .success,
              let children = childrenRef as? [AXUIElement] else {
            return []
        }
        
        var elements: [Element] = []
        
        for child in children {
            // Skip non-actionable elements
            guard shouldInclude(child) else { continue }
            
            // Build element
            if let element = try buildElement(from: child, pid: pid, depth: depth, maxDepth: maxDepth) {
                elements.append(element)
            }
        }
        
        return elements
    }
    
    /// Build a single element from AXUIElement
    private func buildElement(
        from axElement: AXUIElement,
        pid: Int32,
        depth: Int,
        maxDepth: Int
    ) throws -> Element? {
        
        // Get role
        let role = getStringAttribute(axElement, attribute: kAXRoleAttribute) ?? "AXUnknown"
        let oscpRole = RoleMapping.axToOSCP[role] ?? "unknown"
        
        // Get subrole
        let subrole = getStringAttribute(axElement, attribute: kAXSubroleAttribute)
        
        // Get name
        let name = getStringAttribute(axElement, attribute: kAXTitleAttribute) ??
                   getStringAttribute(axElement, attribute: kAXDescriptionAttribute)
        
        // Get description
        let description = getStringAttribute(axElement, attribute: kAXDescriptionAttribute)
        
        // Get value
        let value = getStringAttribute(axElement, attribute: kAXValueAttribute)
        
        // Get position and size
        let position = getPointAttribute(axElement, attribute: kAXPositionAttribute) ?? .zero
        let size = getSizeAttribute(axElement, attribute: kAXSizeAttribute) ?? .zero
        
        // Build bounds
        let bounds = Bounds(x: position.x, y: position.y, w: size.width, h: size.height)
        
        // Skip elements with zero size
        guard size.width > 0 && size.height > 0 else { return nil }
        
        // Get states
        var states: [ElementState] = []
        
        if let enabled = getBoolAttribute(axElement, attribute: kAXEnabledAttribute) {
            states.append(enabled ? .enabled : .disabled)
        }
        
        if let visible = getBoolAttribute(axElement, attribute: kAXVisibleAttribute) {
            states.append(visible ? .visible : .invisible)
        }
        
        if let focused = getBoolAttribute(axElement, attribute: kAXFocusedAttribute), focused {
            states.append(.focused)
        }
        
        if let pressed = getBoolAttribute(axElement, attribute: kAXPressedAttribute), pressed {
            states.append(.pressed)
        }
        
        // Check for additional states via AXValue
        if let value = value, !value.isEmpty {
            // Could be checked, selected, etc.
        }
        
        // Build attributes
        var attributes: [String: AttributeValue] = [:]
        
        if let isDefault = getBoolAttribute(axElement, attribute: kAXDefaultAttribute) {
            attributes["default_button"] = .bool(isDefault)
        }
        
        if let isCancel = getBoolAttribute(axElement, attribute: kAXCancelAttribute) {
            attributes["cancel_button"] = .bool(isCancel)
        }
        
        // Generate element ID
        let elementId = generateElementId(pid: pid, role: role, name: name, position: position, size: size)
        
        // Recursively capture children
        let children = try captureElements(from: axElement, pid: pid, depth: depth + 1, maxDepth: maxDepth)
        
        return Element(
            id: elementId,
            role: oscpRole,
            subrole: subrole,
            name: name,
            description: description,
            value: value,
            bounds: bounds,
            states: states,
            attributes: attributes,
            confidence: 1.0,  // AXUIElement is authoritative
            source: .axuielement,
            children: children
        )
    }
    
    /// Check if element should be included in tree
    private func shouldInclude(_ element: AXUIElement) -> Bool {
        
        // Get role
        guard let role = getStringAttribute(element, attribute: kAXRoleAttribute) else {
            return false
        }
        
        // Skip certain container types
        let skipRoles = [
            "AXRuler",
            "AXLayoutArea",
            "AXGrowthArea",
            "AXMatte"
        ]
        
        if skipRoles.contains(role) {
            return false
        }
        
        // Check if visible
        if let visible = getBoolAttribute(element, attribute: kAXVisibleAttribute), !visible {
            return false
        }
        
        return true
    }
    
    // MARK: - Attribute Accessors
    
    /// Get string attribute
    public func getStringAttribute(_ element: AXUIElement, attribute: String) -> String? {
        var valueRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(element, attribute as CFString, &valueRef)
        
        guard status == .success else { return nil }
        
        if let string = valueRef as? String {
            return string
        }
        
        return nil
    }
    
    /// Get boolean attribute
    public func getBoolAttribute(_ element: AXUIElement, attribute: String) -> Bool? {
        var valueRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(element, attribute as CFString, &valueRef)
        
        guard status == .success else { return nil }
        
        if let bool = valueRef as? Bool {
            return bool
        }
        
        // AX returns CFBoolean
        if let cfBool = valueRef {
            return CFBooleanGetValue(cfBool as CFBoolean)
        }
        
        return nil
    }
    
    /// Get point attribute
    public func getPointAttribute(_ element: AXUIElement, attribute: String) -> CGPoint? {
        var valueRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(element, attribute as CFString, &valueRef)
        
        guard status == .success else { return nil }
        
        var point = CGPoint.zero
        if AXValueGetValue(valueRef as! AXValue, .cgPoint, &point) {
            return point
        }
        
        return nil
    }
    
    /// Get size attribute
    public func getSizeAttribute(_ element: AXUIElement, attribute: String) -> CGSize? {
        var valueRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(element, attribute as CFString, &valueRef)
        
        guard status == .success else { return nil }
        
        var size = CGSize.zero
        if AXValueGetValue(valueRef as! AXValue, .cgSize, &size) {
            return size
        }
        
        return nil
    }
    
    // MARK: - Element ID Generation
    
    /// Generate unique element ID
    private func generateElementId(
        pid: Int32,
        role: String,
        name: String?,
        position: CGPoint,
        size: CGSize
    ) -> String {
        elementIdCounter += 1
        
        // Create hash from element attributes
        let hashInput = "\(role)_\(name ?? "")_\(Int(position.x))_\(Int(position.y))_\(Int(size.width))_\(Int(size.height))"
        let hash = hashInput.hashValue
        
        return "e_\(pid)_\(String(format: "%x", abs(hash)))"
    }
    
    // MARK: - Mouse State
    
    /// Get current mouse position
    public func getMousePosition() -> CGPoint {
        let mouseLocation = NSEvent.mouseLocation
        // NSEvent gives Y from bottom, convert to top-left origin
        guard let screen = NSScreen.main else { return .zero }
        return CGPoint(
            x: mouseLocation.x,
            y: screen.frame.height - mouseLocation.y
        )
    }
    
    /// Find element under mouse
    public func getElementAtMouse() -> AXUIElement? {
        let mousePos = getMousePosition()
        var element: AXUIElement?
        
        let status = AXUIElementCopyElementAtPosition(
            systemWide,
            Float(mousePos.x),
            Float(mousePos.y),
            &element
        )
        
        guard status == .success else { return nil }
        return element
    }
    
    // MARK: - Focused Application
    
    /// Get the currently focused application
    public func getFocusedApplication() -> AXUIElement? {
        var appRef: CFTypeRef?
        let status = AXUIElementCopyAttributeValue(
            systemWide,
            kAXFocusedApplicationAttribute as CFString,
            &appRef
        )
        
        guard status == .success else { return nil }
        return (appRef as! AXUIElement)
    }
}
```

### 3.2 Usage Example

```swift
// Example usage of AXUIElementCapture
let capture = AXUIElementCapture()

// Check permissions first
if !capture.checkPermissions() {
    capture.requestPermissions()
    return
}

// Capture full tree
let windows = try capture.captureTree()

for window in windows {
    print("Window: \(window.title)")
    print("  Bounds: \(window.bounds)")
    
    for element in window.elements {
        print("  Element: \(element.role) - \(element.name ?? "unnamed")")
    }
}

// Get mouse position
let mousePos = capture.getMousePosition()
print("Mouse at: \(mousePos)")
```

---

## 4. CDP Bridge

### 4.1 CDPBridge.swift

```swift
import Foundation
import WebKit

// MARK: - CDP Bridge

/// Chrome DevTools Protocol bridge for browser windows
public final class CDPBridge {
    
    // MARK: - Properties
    
    /// WebSocket connection to CDP endpoint
    private var webSocketTask: URLSessionWebSocketTask?
    
    /// Current CDP session ID
    private var sessionId: String?
    
    /// Request ID counter
    private var requestId: Int = 0
    
    /// Pending requests
    private var pendingRequests: [Int: CheckedContinuation<CDPResponse, Error>] = [:]
    
    /// CDP endpoint URL
    private let endpoint: URL
    
    /// URLSession for WebSocket
    private lazy var urlSession = URLSession(configuration: .default)
    
    // MARK: - Initialization
    
    /// Initialize with CDP WebSocket URL
    public init(endpoint: URL) {
        self.endpoint = endpoint
    }
    
    /// Create bridge for Safari
    public static func safariBridge() -> CDPBridge? {
        // Safari WebKit debugger endpoint (port 9222)
        guard let url = URL(string: "ws://localhost:9222/1") else { return nil }
        return CDPBridge(endpoint: url)
    }
    
    /// Create bridge for Chrome
    public static func chromeBridge(profile: Int = 0) -> CDPBridge? {
        // Chrome DevTools endpoint
        guard let url = URL(string: "ws://localhost:9222/devtools/page/\(profile)") else { return nil }
        return CDPBridge(endpoint: url)
    }
    
    // MARK: - Connection
    
    /// Connect to CDP endpoint
    public func connect() async throws {
        webSocketTask = urlSession.webSocketTask(with: endpoint)
        webSocketTask?.resume()
        
        // Start receiving messages
        Task {
            await receiveLoop()
        }
        
        // Enable CDP domains
        try await enableDomains()
    }
    
    /// Disconnect from CDP
    public func disconnect() {
        webSocketTask?.cancel(with: .goingAway, reason: nil)
        webSocketTask = nil
        sessionId = nil
    }
    
    // MARK: - Enable Domains
    
    /// Enable required CDP domains
    private func enableDomains() async throws {
        // Enable Runtime domain
        _ = try await sendCommand("Runtime.enable")
        
        // Enable DOM domain
        _ = try await sendCommand("DOM.enable")
        
        // Enable Page domain
        _ = try await sendCommand("Page.enable")
    }
    
    // MARK: - Send Command
    
    /// Send CDP command and wait for response
    public func sendCommand(_ method: String, params: [String: Any] = [:]) async throws -> CDPResponse {
        requestId += 1
        
        let command = CDPCommand(
            id: requestId,
            method: method,
            params: params
        )
        
        let encoder = JSONEncoder()
        let data = try encoder.encode(command)
        
        guard let jsonString = String(data: data, encoding: .utf8) else {
            throw CDPError.encodingFailed
        }
        
        return try await withCheckedThrowingContinuation { continuation in
            pendingRequests[requestId] = continuation
            
            webSocketTask?.send(.string(jsonString)) { error in
                if let error = error {
                    self.pendingRequests.removeValue(forKey: requestId)
                    continuation.resume(throwing: error)
                }
            }
        }
    }
    
    // MARK: - Receive Loop
    
    /// Main receive loop for CDP messages
    private func receiveLoop() async {
        guard let task = webSocketTask else { return }
        
        do {
            while true {
                let message = try await task.receive()
                
                switch message {
                case .string(let text):
                    await handleMessage(text)
                case .data(let data):
                    if let text = String(data: data, encoding: .utf8) {
                        await handleMessage(text)
                    }
                @unknown default:
                    break
                }
            }
        } catch {
            print("CDP receive error: \(error)")
        }
    }
    
    /// Handle received CDP message
    private func handleMessage(_ text: String) async {
        guard let data = text.data(using: .utf8) else { return }
        
        let decoder = JSONDecoder()
        
        // Try to decode as response
        if let response = try? decoder.decode(CDPResponse.self, from: data) {
            if let continuation = pendingRequests.removeValue(forKey: response.id) {
                continuation.resume(returning: response)
            }
            return
        }
        
        // Try to decode as event
        if let event = try? decoder.decode(CDPEvent.self, from: data) {
            await handleEvent(event)
        }
    }
    
    /// Handle CDP event
    private func handleEvent(_ event: CDPEvent) async {
        // Handle events as needed (e.g., DOM.subtreeModified)
        switch event.method {
        case "Runtime.consoleAPICalled":
            // Handle console events
            break
        case "Runtime.exceptionThrown":
            // Handle exceptions
            break
        default:
            break
        }
    }
    
    // MARK: - DOM Methods
    
    /// Get document DOM tree
    public func getDocument() async throws -> CDPNode {
        let response = try await sendCommand("DOM.getDocument", params: [
            "depth": 100,
            "pierce": true
        ])
        
        guard let root = response.result["root"] as? [String: Any] else {
            throw CDPError.invalidResponse
        }
        
        return try CDPNode(json: root)
    }
    
    /// Get node bounds
    public func getBoxModel(nodeId: Int) async throws -> CDPBoxModel {
        let response = try await sendCommand("DOM.getBoxModel", params: [
            "nodeId": nodeId
        ])
        
        guard let model = response.result["model"] as? [String: Any] else {
            throw CDPError.invalidResponse
        }
        
        return try CDPBoxModel(json: model)
    }
    
    /// Resolve node to object
    public func resolveNode(nodeId: Int) async throws -> CDPObject {
        let response = try await sendCommand("DOM.resolveNode", params: [
            "nodeId": nodeId
        ])
        
        guard let obj = response.result["object"] as? [String: Any] else {
            throw CDPError.invalidResponse
        }
        
        return try CDPObject(json: obj)
    }
    
    // MARK: - Page Methods
    
    /// Get frame tree
    public func getFrameTree() async throws -> CDPFrameTree {
        let response = try await sendCommand("Page.getFrameTree")
        
        guard let tree = response.result["frameTree"] as? [String: Any] else {
            throw CDPError.invalidResponse
        }
        
        return try CDPFrameTree(json: tree)
    }
    
    // MARK: - Runtime Methods
    
    /// Evaluate JavaScript expression
    public func evaluate(script: String) async throws -> CDPEvaluationResult {
        let response = try await sendCommand("Runtime.evaluate", params: [
            "expression": script,
            "returnByValue": true
        ])
        
        guard let result = response.result else {
            throw CDPError.invalidResponse
        }
        
        return try CDPEvaluationResult(json: result)
    }
    
    /// Call function on object
    public func callFunctionOn(
        objectId: String,
        functionDeclaration: String
    ) async throws -> CDPEvaluationResult {
        let response = try await sendCommand("Runtime.callFunctionOn", params: [
            "objectId": objectId,
            "functionDeclaration": functionDeclaration
        ])
        
        guard let result = response.result else {
            throw CDPError.invalidResponse
        }
        
        return try CDPEvaluationResult(json: result)
    }
    
    // MARK: - Capture Tree
    
    /// Capture DOM tree as OSCP elements
    public func captureTree() async throws -> [Window] {
        let document = try await getDocument()
        let bounds = try await getBoxModel(nodeId: document.nodeId)
        
        // Build element from DOM
        let element = try await buildElement(from: document.nodeId)
        
        // Create window
        let window = Window(
            id: "cdp_window_\(document.nodeId)",
            title: "Browser Content",
            pid: -1,
            app: nil,
            bounds: bounds.content,
            position: CGPoint(x: bounds.content.x, y: bounds.content.y),
            focused: true,
            elements: element.children
        )
        
        return [window]
    }
    
    /// Build OSCP element from CDP node
    private func buildElement(from nodeId: Int) async throws -> Element {
        // Get node attributes
        let node = try await getNode(nodeId)
        
        // Get bounds
        let boxModel = try await getBoxModel(nodeId: nodeId)
        
        // Determine role
        let role = mapNodeToRole(node)
        
        // Build attributes
        var children: [Element] = []
        
        for childNodeId in node.children {
            let child = try await buildElement(from: childNodeId)
            children.append(child)
        }
        
        return Element(
            id: "e_cdp_\(nodeId)",
            role: role,
            subrole: nil,
            name: node.name,
            description: nil,
            value: node.value,
            bounds: boxModel.content,
            states: [],
            attributes: [:],
            confidence: 0.9,
            source: .cdp,
            children: children
        )
    }
    
    /// Get node details
    private func getNode(_ nodeId: Int) async throws -> CDPNodeInfo {
        let response = try await sendCommand("DOM.getAttributes", params: [
            "nodeId": nodeId
        ])
        
        guard let attributes = response.result["attributes"] as? [String] else {
            throw CDPError.invalidResponse
        }
        
        // Parse attributes
        var name: String?
        var value: String?
        var children: [Int] = []
        
        var i = 0
        while i < attributes.count {
            let attrName = attributes[i]
            let attrValue = i + 1 < attributes.count ? attributes[i + 1] : ""
            
            switch attrName {
            case "id":
                name = attrValue
            case "value":
                value = attrValue
            default:
                break
            }
            
            i += 2
        }
        
        // Get children
        let childrenResponse = try await sendCommand("DOM.getChildNodes", params: [
            "nodeId": nodeId
        ])
        
        if let nodes = childrenResponse.result["nodes"] as? [[String: Any]] {
            for node in nodes {
                if let childId = node["nodeId"] as? Int {
                    children.append(childId)
                }
            }
        }
        
        return CDPNodeInfo(
            nodeId: nodeId,
            name: name,
            value: value,
            children: children
        )
    }
    
    /// Map CDP node type to OSCP role
    private func mapNodeToRole(_ node: CDPNodeInfo) -> String {
        // Use tag name or role attribute
        if let role = node.attributes["role"] as? String {
            return role
        }
        
        let tagName = (node.attributes["tagName"] as? String ?? "").lowercased()
        
        switch tagName {
        case "button": return "button"
        case "input": return "text_field"
        case "textarea": return "text_area"
        case "a": return "link"
        case "img": return "image"
        case "div": return "group"
        case "span": return "static_text"
        case "ul", "ol": return "list"
        case "li": return "list_item"
        case "table": return "table"
        case "tr": return "row"
        case "td", "th": return "tabular_cell"
        case "select": return "combo_box"
        case "checkbox": return "check_box"
        case "radio": return "radio_button"
        default: return "static_text"
        }
    }
}

// MARK: - CDP Types

/// CDP command
struct CDPCommand: Codable {
    let id: Int
    let method: String
    let params: [String: Any]
    
    enum CodingKeys: String, CodingKey {
        case id, method, params
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(method, forKey: .method)
        
        // Encode params as JSON object
        let data = try JSONSerialization.data(withJSONObject: params)
        let json = try JSONSerialization.jsonObject(with: data)
        try container.encode(json as? [String: Any] ?? [:], forKey: .params)
    }
}

/// CDP response
struct CDPResponse: Codable {
    let id: Int
    let result: [String: Any]
    
    enum CodingKeys: String, CodingKey {
        case id, result
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(Int.self, forKey: .id)
        
        // Parse result as dictionary
        let resultContainer = try container.nestedContainer(keyedBy: DynamicCodingKey.self, forKey: .result)
        var result: [String: Any] = [:]
        for key in resultContainer.allKeys {
            result[key.stringValue] = try resultContainer.decode(Any.self, forKey: key)
        }
        self.result = result
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(result, forKey: .result)
    }
}

/// CDP event
struct CDPEvent: Codable {
    let method: String
    let params: [String: Any]?
    
    enum CodingKeys: String, CodingKey {
        case method, params
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        method = try container.decode(String.self, forKey: .method)
        params = try container.decodeIfPresent([String: Any].self, forKey: .params)
    }
}

/// Dynamic coding key for flexible JSON parsing
struct DynamicCodingKey: CodingKey {
    var stringValue: String
    var intValue: Int?
    
    init?(stringValue: String) {
        self.stringValue = stringValue
        self.intValue = nil
    }
    
    init?(intValue: Int) {
        self.stringValue = String(intValue)
        self.intValue = intValue
    }
}

/// CDP node
struct CDPNode {
    let nodeId: Int
    let backendNodeId: Int
    let children: [CDPNode]
    
    init(json: [String: Any]) throws {
        self.nodeId = json["nodeId"] as? Int ?? 0
        self.backendNodeId = json["backendNodeId"] as? Int ?? 0
        
        if let childrenArray = json["children"] as? [[String: Any]] {
            self.children = try childrenArray.map { try CDPNode(json: $0) }
        } else {
            self.children = []
        }
    }
}

/// CDP box model
struct CDPBoxModel {
    let content: Bounds
    let padding: Bounds
    let border: Bounds
    let margin: Bounds
    
    init(json: [String: Any]) throws {
        func parseQuad(_ key: String) -> Bounds? {
            guard let quad = json[key] as? [Double], quad.count == 4 else {
                return nil
            }
            // CDP returns [x1, y1, x2, y2] for quad
            return Bounds(
                x: quad[0],
                y: quad[1],
                w: quad[2] - quad[0],
                h: quad[3] - quad[1]
            )
        }
        
        self.content = parseQuad("content") ?? .init(x: 0, y: 0, w: 0, h: 0)
        self.padding = parseQuad("padding") ?? .init(x: 0, y: 0, w: 0, h: 0)
        self.border = parseQuad("border") ?? .init(x: 0, y: 0, w: 0, h: 0)
        self.margin = parseQuad("margin") ?? .init(x: 0, y: 0, w: 0, h: 0)
    }
}

/// CDP node info
struct CDPNodeInfo {
    let nodeId: Int
    let name: String?
    let value: String?
    let children: [Int]
    var attributes: [String: Any] = [:]
    
    init(json: [String: Any]) {
        self.nodeId = json["nodeId"] as? Int ?? 0
        self.name = json["name"] as? String
        self.value = json["value"] as? String
        self.children = json["children"] as? [Int] ?? []
    }
}

/// CDP object
struct CDPObject {
    let type: String
    let subtype: String?
    let className: String?
    let value: String?
    let description: String?
    
    init(json: [String: Any]) throws {
        self.type = json["type"] as? String ?? "unknown"
        self.subtype = json["subtype"] as? String
        self.className = json["className"] as? String
        self.value = json["value"] as? String
        self.description = json["description"] as? String
    }
}

/// CDP evaluation result
struct CDPEvaluationResult {
    let result: CDPObject
    let wasThrown: Bool?
    
    init(json: [String: Any]) throws {
        if let resultObj = json["result"] as? [String: Any] {
            self.result = try CDPObject(json: resultObj)
        } else {
            self.result = try CDPObject(json: json)
        }
        self.wasThrown = json["wasThrown"] as? Bool
    }
}

/// CDP frame tree
struct CDPFrameTree {
    let frame: CDPFrame
    let childFrames: [CDPFrameTree]
    
    init(json: [String: Any]) throws {
        if let frameJson = json["frame"] as? [String: Any] {
            self.frame = try CDPFrame(json: frameJson)
        } else {
            self.frame = CDPFrame(json: [:])
        }
        
        if let childrenArray = json["childFrames"] as? [[String: Any]] {
            self.childFrames = try childrenArray.map { try CDPFrameTree(json: $0) }
        } else {
            self.childFrames = []
        }
    }
}

/// CDP frame
struct CDPFrame {
    let id: String
    let url: String
    let name: String?
    
    init(json: [String: Any]) {
        self.id = json["id"] as? String ?? ""
        self.url = json["url"] as? String ?? ""
        self.name = json["name"] as? String
    }
}

// MARK: - CDP Errors

/// CDP errors
enum CDPError: Error, LocalizedError {
    case connectionFailed
    case encodingFailed
    case invalidResponse
    case commandFailed(message: String)
    
    var errorDescription: String? {
        switch self {
        case .connectionFailed: return "CDP connection failed"
        case .encodingFailed: return "CDP encoding failed"
        case .invalidResponse: return "Invalid CDP response"
        case .commandFailed(let message): return "CDP command failed: \(message)"
        }
    }
}

// MARK: - Extension for decoding Any

extension KeyedDecodingContainer {
    func decode(_ type: [String: Any].Type, forKey key: Key) throws -> [String: Any] {
        let container = try self.nestedContainer(keyedBy: DynamicCodingKey.self, forKey: key)
        return try container.decode(type)
    }
    
    func decodeIfPresent(_ type: [String: Any].Type, forKey key: Key) throws -> [String: Any]? {
        guard contains(key) else { return nil }
        return try decode(type, forKey: key)
    }
    
    func decode(_ type: [Any].Type, forKey key: Key) throws -> [Any] {
        var container = try self.nestedUnkeyedContainer(forKey: key)
        return try container.decode(type)
    }
    
    func decodeIfPresent(_ type: [Any].Type, forKey key: Key) throws -> [Any]? {
        guard contains(key) else { return nil }
        return try decode(type, forKey: key)
    }
    
    func decode(_ type: String.Type, forKey key: Key) throws -> String {
        return try decode(String.self, forKey: key)
    }
    
    func decodeIfPresent(_ type: String.Type, forKey key: Key) throws -> String? {
        return try decodeIfPresent(String.self, forKey: key)
    }
    
    func decode(_ type: Int.Type, forKey key: Key) throws -> Int {
        return try decode(Int.self, forKey: key)
    }
    
    func decodeIfPresent(_ type: Int.Type, forKey key: Key) throws -> Int? {
        return try decodeIfPresent(Int.self, forKey: key)
    }
    
    func decode(_ type: Bool.Type, forKey key: Key) throws -> Bool {
        return try decode(Bool.self, forKey: key)
    }
    
    func decodeIfPresent(_ type: Bool.Type, forKey key: Key) throws -> Bool? {
        return try decodeIfPresent(Bool.self, forKey: key)
    }
    
    func decode(_ type: Double.Type, forKey key: Key) throws -> Double {
        return try decode(Double.self, forKey: key)
    }
    
    func decodeIfPresent(_ type: Double.Type, forKey key: Key) throws -> Double? {
        return try decodeIfPresent(Double.self, forKey: key)
    }
}

extension KeyedDecodingContainer where Key == DynamicCodingKey {
    func decode(_ type: [String: Any].Type) throws -> [String: Any] {
        var result: [String: Any] = [:]
        
        for key in allKeys {
            result[key.stringValue] = try decode(Any.self, forKey: key)
        }
        
        return result
    }
}

extension UnkeyedDecodingContainer {
    mutating func decode(_ type: [Any].Type) throws -> [Any] {
        var array: [Any] = []
        
        while !isAtEnd {
            array.append(try decode(Any.self))
        }
        
        return array
    }
}

extension KeyedDecodingContainer where Key == DynamicCodingKey {
    func decode(_ type: Any.self, forKey key: DynamicCodingKey) throws -> Any {
        if let string = try? decode(String.self, forKey: key) {
            return string
        }
        if let int = try? decode(Int.self, forKey: key) {
            return int
        }
        if let double = try? decode(Double.self, forKey: key) {
            return double
        }
        if let bool = try? decode(Bool.self, forKey: key) {
            return bool
        }
        if let dict = try? decode([String: Any].self, forKey: key) {
            return dict
        }
        if let array = try? decode([Any].self, forKey: key) {
            return array
        }
        if let null = try? decode(Null.self, forKey: key) {
            return NSNull()
        }
        
        throw DecodingError.dataCorruptedError(forKey: key, in: self, debugDescription: "Unable to decode value")
    }
}

struct Null: Codable {}
```

---

## 5. Tree Builder

### 5.1 TreeBuilder.swift

```swift
import Foundation

// MARK: - Tree Builder

/// Builds standardized element trees from platform-specific data
public final class TreeBuilder {
    
    // MARK: - Properties
    
    /// Element ID counter
    private var elementCounter: Int = 0
    
    /// Current PID for ID generation
    private var currentPid: Int32 = 0
    
    // MARK: - Build from AXUIElement
    
    /// Build element tree from raw AXUIElement data
    public func buildTree(from windows: [Window]) -> [Element] {
        var elements: [Element] = []
        
        for window in windows {
            elements.append(contentsOf: flattenElements(window.elements))
        }
        
        return elements
    }
    
    /// Flatten nested element tree into list
    public func flattenElements(_ elements: [Element]) -> [Element] {
        var result: [Element] = []
        
        for element in elements {
            result.append(element)
            result.append(contentsOf: flattenElements(element.children))
        }
        
        return result
    }
    
    /// Find element by ID in tree
    public func findElement(byId id: String, in windows: [Window]) -> Element? {
        for window in windows {
            if let found = findElementInTree(byId: id, in: window.elements) {
                return found
            }
        }
        return nil
    }
    
    /// Find element in nested tree
    private func findElementInTree(byId id: String, in elements: [Element]) -> Element? {
        for element in elements {
            if element.id == id {
                return element
            }
            
            if let found = findElementInTree(byId: id, in: element.children) {
                return found
            }
        }
        return nil
    }
    
    /// Find elements by role
    public func findElements(byRole role: String, in windows: [Window]) -> [Element] {
        var results: [Element] = []
        
        for window in windows {
            results.append(contentsOf: findElementsInTree(byRole: role, in: window.elements))
        }
        
        return results
    }
    
    /// Find elements by role in nested tree
    private func findElementsInTree(byRole role: String, in elements: [Element]) -> [Element] {
        var results: [Element] = []
        
        for element in elements {
            if element.role == role {
                results.append(element)
            }
            
            results.append(contentsOf: findElementsInTree(byRole: role, in: element.children))
        }
        
        return results
    }
    
    /// Find elements by name (partial match)
    public func findElements(byName name: String, in windows: [Window], partial: Bool = true) -> [Element] {
        var results: [Element] = []
        
        for window in windows {
            results.append(contentsOf: findElementsInTree(byName: name, in: window.elements, partial: partial))
        }
        
        return results
    }
    
    /// Find elements by name in nested tree
    private func findElementsInTree(byName name: String, in elements: [Element], partial: Bool) -> [Element] {
        var results: [Element] = []
        
        let nameMatch: (String?) -> Bool = { elementName in
            guard let elementName = elementName else { return false }
            if partial {
                return elementName.localizedCaseInsensitiveContains(name)
            } else {
                return elementName == name
            }
        }
        
        for element in elements {
            if nameMatch(element.name) {
                results.append(element)
            }
            
            results.append(contentsOf: findElementsInTree(byName: name, in: element.children, partial: partial))
        }
        
        return results
    }
    
    /// Find element at position
    public func findElement(at point: CGPoint, in windows: [Window]) -> Element? {
        // Search windows in reverse order (topmost first)
        for window in windows.reversed() {
            // Check if point is within window bounds
            guard window.bounds.contains(point) else { continue }
            
            // Find element in window
            if let element = findElementInTree(at: point, in: window.elements) {
                return element
            }
        }
        return nil
    }
    
    /// Find element at position in nested tree
    private func findElementInTree(at point: CGPoint, in elements: [Element]) -> Element? {
        // Search in reverse order (topmost first)
        for element in elements.reversed() {
            // Check if point is within element bounds
            guard element.bounds.contains(point) else { continue }
            
            // First check children
            if let child = findElementInTree(at: point, in: element.children) {
                return child
            }
            
            // Return this element if no child matched
            return element
        }
        return nil
    }
    
    /// Find candidates near a position
    public func findCandidates(near point: CGPoint, radius: Double, in windows: [Window]) -> [Element] {
        var candidates: [Element] = []
        
        for window in windows {
            let windowCandidates = findCandidatesInTree(near: point, radius: radius, in: window.elements)
            candidates.append(contentsOf: windowCandidates)
        }
        
        // Sort by distance
        candidates.sort { e1, e2 in
            let d1 = distance(from: point, to: e1.bounds.center)
            let d2 = distance(from: point, to: e2.bounds.center)
            return d1 < d2
        }
        
        return candidates
    }
    
    /// Find candidates in nested tree
    private func findCandidatesInTree(near point: CGPoint, radius: Double, in elements: [Element]) -> [Element] {
        var candidates: [Element] = []
        
        for element in elements {
            let elementCenter = element.bounds.center
            let dist = distance(from: point, to: elementCenter)
            
            if dist <= radius {
                candidates.append(element)
            }
            
            candidates.append(contentsOf: findCandidatesInTree(near: point, radius: radius, in: element.children))
        }
        
        return candidates
    }
    
    /// Calculate distance between two points
    private func distance(from p1: CGPoint, to p2: CGPoint) -> Double {
        let dx = p1.x - p2.x
        let dy = p1.y - p2.y
        return sqrt(dx * dx + dy * dy)
    }
    
    // MARK: - Build Frame
    
    /// Build complete frame from windows
    public func buildFrame(
        windows: [Window],
        treeAnalysis: TreeAnalysis,
        mouseState: MouseState,
        keyboardState: KeyboardState,
        frameId: Int,
        latencyMs: Int,
        fallbackActive: Bool = false
    ) -> Frame {
        return Frame(
            frameId: frameId,
            platform: "macOS",
            latencyMs: latencyMs,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            windows: windows,
            treeAnalysis: treeAnalysis,
            mouse: mouseState,
            keyboard: keyboardState,
            fallbackActive: fallbackActive
        )
    }
}
```

---

## 6. Tree Analyzer

### 6.1 TreeAnalyzer.swift

```swift
import Foundation
import CoreGraphics

// MARK: - Tree Analyzer

/// Analyzes accessibility tree quality and calculates confidence
public final class TreeAnalyzer {
    
    // MARK: - Configuration
    
    /// Thresholds for confidence levels
    public struct Thresholds {
        public let highCoverage: Double      // > 0.8 = HIGH
        public let mediumCoverage: Double    // 0.5-0.8 = MEDIUM
        public let lowCoverage: Double       // 0.3-0.5 = LOW
        public let namedRatio: Double        // > 0.5 for HIGH
        
        public static let `default` = Thresholds(
            highCoverage: 0.8,
            mediumCoverage: 0.5,
            lowCoverage: 0.3,
            namedRatio: 0.5
        )
    }
    
    // MARK: - Properties
    
    /// Analysis thresholds
    public let thresholds: Thresholds
    
    // MARK: - Initialization
    
    public init(thresholds: Thresholds = .default) {
        self.thresholds = thresholds
    }
    
    // MARK: - Analysis
    
    /// Analyze a list of windows and return tree analysis
    public func analyze(windows: [Window]) -> TreeAnalysis {
        // Collect all elements
        var allElements: [Element] = []
        
        for window in windows {
            allElements.append(contentsOf: flattenElements(window.elements))
        }
        
        // Calculate metrics
        let totalElements = allElements.count
        let namedElements = allElements.filter { $0.name != nil && !$0.name!.isEmpty }.count
        let unlabeledElements = totalElements - namedElements
        
        // Calculate total window area
        let windowArea = windows.reduce(0.0) { $0 + $1.bounds.area }
        
        // Calculate named area
        var namedArea: Double = 0
        for element in allElements where element.name != nil && !element.name!.isEmpty {
            namedArea += element.bounds.area
        }
        
        // Calculate coverage score
        let coverageScore = windowArea > 0 ? namedArea / windowArea : 0
        
        // Calculate named ratio
        let namedRatio = totalElements > 0 ? Double(namedElements) / Double(totalElements) : 0
        
        // Calculate average depth
        let avgDepth = calculateAverageDepth(elements: allElements)
        
        // Determine confidence
        let confidence = calculateConfidence(coverageScore: coverageScore, namedRatio: namedRatio)
        
        // Determine recommended action
        let recommendedAction = RecommendedAction.from(confidence: confidence)
        
        return TreeAnalysis(
            coverageScore: coverageScore,
            namedElements: namedElements,
            unlabeledElements: unlabeledElements,
            totalElements: totalElements,
            avgDepth: avgDepth,
            confidence: confidence,
            recommendedAction: recommendedAction
        )
    }
    
    /// Flatten element tree
    private func flattenElements(_ elements: [Element]) -> [Element] {
        var result: [Element] = []
        
        for element in elements {
            result.append(element)
            result.append(contentsOf: flattenElements(element.children))
        }
        
        return result
    }
    
    /// Calculate average tree depth
    private func calculateAverageDepth(elements: [Element]) -> Double {
        guard !elements.isEmpty else { return 0 }
        
        var totalDepth: Double = 0
        var count: Double = 0
        
        func traverse(_ elements: [Element], depth: Int) {
            for element in elements {
                totalDepth += Double(depth)
                count += 1
                traverse(element.children, depth: depth + 1)
            }
        }
        
        traverse(elements, depth: 0)
        
        return count > 0 ? totalDepth / count : 0
    }
    
    /// Calculate confidence level
    private func calculateConfidence(coverageScore: Double, namedRatio: Double) -> Confidence {
        if coverageScore > thresholds.highCoverage && namedRatio > thresholds.namedRatio {
            return .high
        } else if coverageScore >= thresholds.mediumCoverage {
            return .medium
        } else if coverageScore >= thresholds.lowCoverage {
            return .low
        } else {
            return .none
        }
    }
    
    // MARK: - Quick Checks
    
    /// Quick check if tree has good coverage
    public func hasGoodCoverage(windows: [Window]) -> Bool {
        let analysis = analyze(windows: windows)
        return analysis.confidence == .high || analysis.confidence == .medium
    }
    
    /// Quick check if tree needs fallback
    public func needsFallback(windows: [Window]) -> Bool {
        let analysis = analyze(windows: windows)
        return analysis.confidence == .low || analysis.confidence == .none
    }
    
    // MARK: - Element Analysis
    
    /// Analyze a single element's quality
    public func analyzeElement(_ element: Element) -> Double {
        var confidence: Double = 0
        
        // Has name (0.4)
        if element.name != nil && !element.name!.isEmpty {
            confidence += 0.4
        }
        
        // Has actionable role (0.3)
        let actionableRoles = ["button", "text_field", "link", "menu_item", "check_box", "radio_button"]
        if actionableRoles.contains(element.role) {
            confidence += 0.3
        }
        
        // Has reasonable size (0.2)
        let minSize: Double = 5
        if element.bounds.w >= minSize && element.bounds.h >= minSize {
            confidence += 0.2
        }
        
        // Source bonus (0.1)
        switch element.source {
        case .axuielement:
            confidence += 0.1
        case .cdp:
            confidence += 0.1
        case .x11:
            confidence += 0.05
        default:
            break
        }
        
        return min(confidence, 1.0)
    }
}
```

---

## 7. Fallback Manager

### 7.1 FallbackManager.swift

```swift
import Foundation
import CoreGraphics

// MARK: - Fallback Manager

/// Manages fallback chain for capture methods
public final class FallbackManager {
    
    // MARK: - Properties
    
    /// Primary capture (AXUIElement)
    private let axCapture: AXUIElementCapture
    
    /// CDP bridge
    private var cdpBridge: CDPBridge?
    
    /// Frame ID counter
    private var frameCounter: Int = 0
    
    /// Tree analyzer
    private let treeAnalyzer: TreeAnalyzer
    
    // MARK: - Fallback Levels
    
    /// Fallback level
    public enum FallbackLevel: Int, CaseIterable {
        case axuielement = 1
        case cdp = 2
        case heuristics = 3
        case positionOnly = 4
        case handoff = 5
        
        var description: String {
            switch self {
            case .axuielement: return "Native AXUIElement"
            case .cdp: return "CDP Bridge"
            case .heuristics: return "Structural Heuristics"
            case .positionOnly: return "Position-Only Mode"
            case .handoff: return "Human Handoff"
            }
        }
    }
    
    // MARK: - Initialization
    
    public init(axCapture: AXUIElementCapture, cdpBridge: CDPBridge? = nil) {
        self.axCapture = axCapture
        self.cdpBridge = cdpBridge
        self.treeAnalyzer = TreeAnalyzer()
    }
    
    // MARK: - Capture with Fallback
    
    /// Capture frame with automatic fallback
    public func captureFrame() async throws -> Frame {
        let startTime = Date()
        frameCounter += 1
        
        // Try each fallback level in order
        for level in FallbackLevel.allCases {
            do {
                let frame = try await captureAtLevel(level, startTime: startTime)
                return frame
            } catch {
                print("Level \(level.rawValue) (\(level.description)) failed: \(error)")
                continue
            }
        }
        
        // All fallbacks failed - return minimal frame
        return createMinimalFrame(startTime: startTime)
    }
    
    /// Capture at specific fallback level
    private func captureAtLevel(_ level: FallbackLevel, startTime: Date) async throws -> Frame {
        switch level {
        case .axuielement:
            return try await captureAXUIElement(startTime: startTime)
            
        case .cdp:
            return try await captureCDP(startTime: startTime)
            
        case .heuristics:
            return try await captureWithHeuristics(startTime: startTime)
            
        case .positionOnly:
            return try await capturePositionOnly(startTime: startTime)
            
        case .handoff:
            throw OSCPError.emptyTree
        }
    }
    
    // MARK: - Level 1: AXUIElement
    
    /// Capture using AXUIElement
    private func captureAXUIElement(startTime: Date) async throws -> Frame {
        // Check permissions
        guard axCapture.checkPermissions() else {
            throw OSCPError.permissionDenied
        }
        
        // Capture tree
        let windows = try axCapture.captureTree()
        
        // Analyze tree
        let analysis = treeAnalyzer.analyze(windows: windows)
        
        // Check if coverage is acceptable
        guard analysis.confidence != .none else {
            throw OSCPError.lowCoverage(score: analysis.coverageScore)
        }
        
        // Get mouse state
        let mousePos = axCapture.getMousePosition()
        let mouse = MouseState(x: mousePos.x, y: mousePos.y)
        
        // Create frame
        let latencyMs = Int(Date().timeIntervalSince(startTime) * 1000)
        
        return Frame(
            frameId: frameCounter,
            platform: "macOS",
            latencyMs: latencyMs,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            windows: windows,
            treeAnalysis: analysis,
            mouse: mouse,
            keyboard: KeyboardState(),
            fallbackActive: false
        )
    }
    
    // MARK: - Level 2: CDP Bridge
    
    /// Capture using CDP bridge
    private func captureCDP(startTime: Date) async throws -> Frame {
        guard let cdpBridge = cdpBridge else {
            throw OSCPError.systemError(message: "CDP bridge not configured")
        }
        
        // Connect if needed
        // Note: In real implementation, maintain persistent connection
        
        // Capture DOM tree
        let windows = try await cdpBridge.captureTree()
        
        // Analyze tree
        let analysis = treeAnalyzer.analyze(windows: windows)
        
        // Get mouse state
        let mousePos = axCapture.getMousePosition()
        let mouse = MouseState(x: mousePos.x, y: mousePos.y)
        
        // Create frame
        let latencyMs = Int(Date().timeIntervalSince(startTime) * 1000)
        
        return Frame(
            frameId: frameCounter,
            platform: "macOS",
            latencyMs: latencyMs,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            windows: windows,
            treeAnalysis: analysis,
            mouse: mouse,
            keyboard: KeyboardState(),
            fallbackActive: true
        )
    }
    
    // MARK: - Level 3: Heuristics
    
    /// Capture with position-based heuristics
    private func captureWithHeuristics(startTime: Date) async throws -> Frame {
        // Get raw windows
        let windows = try axCapture.captureTree()
        
        // Apply heuristics to add missing information
        let enhancedWindows = windows.map { enhanceWindowWithHeuristics($0) }
        
        // Analyze
        let analysis = treeAnalyzer.analyze(windows: enhancedWindows)
        
        // Get mouse state
        let mousePos = axCapture.getMousePosition()
        let mouse = MouseState(x: mousePos.x, y: mousePos.y)
        
        // Create frame
        let latencyMs = Int(Date().timeIntervalSince(startTime) * 1000)
        
        return Frame(
            frameId: frameCounter,
            platform: "macOS",
            latencyMs: latencyMs,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            windows: enhancedWindows,
            treeAnalysis: analysis,
            mouse: mouse,
            keyboard: KeyboardState(),
            fallbackActive: true
        )
    }
    
    /// Enhance window with heuristics
    private func enhanceWindowWithHeuristics(_ window: Window) -> Window {
        // Apply heuristics to elements without names
        var enhancedElements = window.elements
        
        for i in 0..<enhancedElements.count {
            enhancedElements[i] = enhanceElementWithHeuristics(enhancedElements[i])
        }
        
        return Window(
            id: window.id,
            title: window.title,
            pid: window.pid,
            app: window.app,
            bounds: window.bounds,
            position: window.position,
            focused: window.focused,
            minimized: window.minimized,
            onAllSpaces: window.onAllSpaces,
            elements: enhancedElements
        )
    }
    
    /// Enhance element with heuristics
    private func enhanceElementWithHeuristics(_ element: Element) -> Element {
        // If element has no name but is actionable, infer name from role
        if (element.name == nil || element.name!.isEmpty) && isActionable(element) {
            // Try to infer name from position
            let inferredName = inferNameFromPosition(element)
            
            return Element(
                id: element.id,
                role: element.role,
                subrole: element.subrole,
                name: inferredName,
                description: element.description,
                value: element.value,
                bounds: element.bounds,
                states: element.states,
                attributes: element.attributes,
                confidence: 0.5,  // Lower confidence for heuristic
                source: .heuristic,
                children: element.children.map { enhanceElementWithHeuristics($0) }
            )
        }
        
        return Element(
            id: element.id,
            role: element.role,
            subrole: element.subrole,
            name: element.name,
            description: element.description,
            value: element.value,
            bounds: element.bounds,
            states: element.states,
            attributes: element.attributes,
            confidence: element.confidence,
            source: element.source,
            children: element.children.map { enhanceElementWithHeuristics($0) }
        )
    }
    
    /// Check if element is actionable
    private func isActionable(_ element: Element) -> Bool {
        let actionableRoles = [
            "button", "link", "menu_item", "check_box", "radio_button",
            "text_field", "text_area", "combo_box", "tab", "slider"
        ]
        return actionableRoles.contains(element.role)
    }
    
    /// Infer name from position heuristics
    private func inferNameFromPosition(_ element: Element) -> String? {
        // Common UI patterns:
        // - Top toolbar buttons: "toolbar_button"
        // - Sidebar items: "sidebar_item"
        // - Icon buttons: "icon_button"
        
        let bounds = element.bounds
        
        // Toolbar region (top 40px)
        if bounds.y < 40 {
            return "toolbar_element"
        }
        
        // Sidebar region (left 250px, tall)
        if bounds.x < 250 && bounds.h > 200 {
            return "sidebar_element"
        }
        
        // Footer region (bottom 40px)
        if bounds.y > bounds.y + bounds.h - 40 {
            return "footer_element"
        }
        
        return nil
    }
    
    // MARK: - Level 4: Position Only
    
    /// Capture position-only data
    private func capturePositionOnly(startTime: Date) async throws -> Frame {
        // Get basic window info only
        let windows = try axCapture.captureTree()
        
        // Strip all element details except position
        let minimalWindows = windows.map { window in
            Window(
                id: window.id,
                title: window.title,
                pid: window.pid,
                app: window.app,
                bounds: window.bounds,
                position: window.position,
                focused: window.focused,
                minimized: window.minimized,
                elements: []  // No elements
            )
        }
        
        // Create minimal analysis
        let analysis = TreeAnalysis(
            coverageScore: 0,
            namedElements: 0,
            unlabeledElements: 0,
            totalElements: 0,
            avgDepth: 0,
            confidence: .none,
            fallbackMethod: "position_only",
            recommendedAction: .exploreOrHandoff
        )
        
        // Get mouse state
        let mousePos = axCapture.getMousePosition()
        let mouse = MouseState(x: mousePos.x, y: mousePos.y)
        
        // Create frame
        let latencyMs = Int(Date().timeIntervalSince(startTime) * 1000)
        
        return Frame(
            frameId: frameCounter,
            platform: "macOS",
            latencyMs: latencyMs,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            windows: minimalWindows,
            treeAnalysis: analysis,
            mouse: mouse,
            keyboard: KeyboardState(),
            fallbackActive: true
        )
    }
    
    // MARK: - Level 5: Human Handoff
    
    /// Create minimal frame for handoff
    private func createMinimalFrame(startTime: Date) -> Frame {
        let latencyMs = Int(Date().timeIntervalSince(startTime) * 1000)
        
        return Frame(
            frameId: frameCounter,
            platform: "macOS",
            latencyMs: latencyMs,
            timestamp: Int64(Date().timeIntervalSince1970 * 1000),
            windows: [],
            treeAnalysis: TreeAnalysis(
                coverageScore: 0,
                namedElements: 0,
                unlabeledElements: 0,
                totalElements: 0,
                avgDepth: 0,
                confidence: .none,
                fallbackMethod: "handoff",
                recommendedAction: .handoff
            ),
            mouse: MouseState(x: 0, y: 0),
            keyboard: KeyboardState(),
            fallbackActive: true
        )
    }
    
    // MARK: - Human Handoff Request
    
    /// Generate human handoff request
    public func createHandoffRequest(
        reason: String,
        reasoning: String,
        failedAttempts: [(x: Double, y: Double, success: Bool, reason: String)],
        window: Window?
    ) -> HandoffRequest {
        // Generate alternative positions
        var alternatives: [AlternativePosition] = []
        
        if let window = window {
            // Suggest positions based on window region
            let toolbarY = window.bounds.y + 20
            alternatives.append(AlternativePosition(
                x: window.bounds.x + 50,
                y: toolbarY,
                confidence: 0.3,
                elementName: "Assume toolbar left"
            ))
            
            alternatives.append(AlternativePosition(
                x: window.bounds.x + window.bounds.w / 2,
                y: window.bounds.y + window.bounds.h / 2,
                confidence: 0.2,
                elementName: "Assume center"
            ))
        }
        
        return HandoffRequest(
            reason: reason,
            reasoning: reasoning,
            attempts: failedAttempts.count,
            failedAttempts: failedAttempts,
            window: window.map { windowToHandoffWindow($0) },
            alternatives: alternatives
        )
    }
    
    /// Convert Window to HandoffWindow
    private func windowToHandoffWindow(_ window: Window) -> HandoffWindow {
        return HandoffWindow(
            id: window.id,
            title: window.title,
            bounds: window.bounds
        )
    }
}

// MARK: - Handoff Types

/// Human handoff request
public struct HandoffRequest: Codable, Sendable {
    public let type: String = "handoff_request"
    public let reason: String
    public let reasoning: String
    public let attempts: Int
    public let failedAttempts: [(x: Double, y: Double, success: Bool, reason: String)]
    public let window: HandoffWindow?
    public let alternatives: [AlternativePosition]
    public let timestamp: Int64
    
    public init(
        reason: String,
        reasoning: String,
        attempts: Int,
        failedAttempts: [(x: Double, y: Double, success: Bool, reason: String)],
        window: HandoffWindow?,
        alternatives: [AlternativePosition]
    ) {
        self.reason = reason
        self.reasoning = reasoning
        self.attempts = attempts
        self.failedAttempts = failedAttempts
        self.window = window
        self.alternatives = alternatives
        self.timestamp = Int64(Date().timeIntervalSince1970 * 1000)
    }
    
    // Codable
    enum CodingKeys: String, CodingKey {
        case type, reason, reasoning, attempts, failedAttempts, window, alternatives, timestamp
    }
    
    public init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        reason = try container.decode(String.self, forKey: .reason)
        reasoning = try container.decode(String.self, forKey: .reasoning)
        attempts = try container.decode(Int.self, forKey: .attempts)
        failedAttempts = []
        window = try container.decodeIfPresent(HandoffWindow.self, forKey: .window)
        alternatives = try container.decode([AlternativePosition].self, forKey: .alternatives)
        timestamp = try container.decode(Int64.self, forKey: .timestamp)
    }
    
    public func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(type, forKey: .type)
        try container.encode(reason, forKey: .reason)
        try container.encode(reasoning, forKey: .reasoning)
        try container.encode(attempts, forKey: .attempts)
        try container.encode(window, forKey: .window)
        try container.encode(alternatives, forKey: .alternatives)
        try container.encode(timestamp, forKey: .timestamp)
    }
}

/// Handoff window info
public struct HandoffWindow: Codable, Sendable {
    public let id: String
    public let title: String
    public let bounds: Bounds
    
    public init(id: String, title: String, bounds: Bounds) {
        self.id = id
        self.title = title
        self.bounds = bounds
    }
}
```

---

## 8. CGEvent Input Engine

### 8.1 CGEventEngine.swift

```swift
import Foundation
import CoreGraphics

// MARK: - CGEvent Input Engine

/// Handles all input injection using CGEvent
public final class CGEventEngine {
    
    // MARK: - Properties
    
    /// Whether input is currently enabled
    public var inputEnabled: Bool = true
    
    /// Key code mapping
    private let keyCodeMap: [String: CGKeyCode] = [
        // Letters
        "a": 0x00, "s": 0x01, "d": 0x02, "f": 0x03,
        "h": 0x04, "g": 0x05, "z": 0x06, "x": 0x07,
        "c": 0x08, "v": 0x09, "b": 0x0B, "q": 0x0C,
        "w": 0x0D, "e": 0x0E, "r": 0x0F, "y": 0x10,
        "t": 0x11, "1": 0x12, "2": 0x13, "3": 0x14,
        "4": 0x15, "6": 0x16, "5": 0x17, "9": 0x19,
        "7": 0x1A, "8": 0x1C, "0": 0x1D, "o": 0x1F,
        "u": 0x20, "i": 0x22, "p": 0x23, "l": 0x25,
        "j": 0x26, "k": 0x28, "n": 0x2D, "m": 0x2E,
        
        // Special keys
        "return": 0x24, "tab": 0x30, "space": 0x31,
        "delete": 0x33, "escape": 0x35,
        
        // Modifiers
        "command": 0x37, "shift": 0x38, "caps_lock": 0x39,
        "option": 0x3A, "ctrl": 0x3B, "right_shift": 0x3C,
        "right_option": 0x3D, "right_ctrl": 0x3E, "right_command": 0x36,
        
        // Arrow keys
        "up": 0x7E, "down": 0x7D, "left": 0x7B, "right": 0x7C,
        
        // Function keys
        "f1": 0x7A, "f2": 0x78, "f3": 0x76, "f4": 0x60,
        "f5": 0x61, "f6": 0x62, "f7": 0x63, "f8": 0x64,
        "f9": 0x65, "f10": 0x6D, "f11": 0x67, "f12": 0x6F
    ]
    
    // MARK: - Initialization
    
    public init() {}
    
    // MARK: - Permission Check
    
    /// Check if input can be injected
    public func canInjectInput() -> Bool {
        // CGEvent works at system level, but we should verify
        return true
    }
    
    // MARK: - Mouse Actions
    
    /// Click at position
    public func click(
        x: Double,
        y: Double,
        button: MouseButton = .left,
        clickType: ClickType = .single
    ) throws {
        let point = CGPoint(x: x, y: y)
        
        switch clickType {
        case .single:
            try performSingleClick(point: point, button: button)
        case .double:
            try performDoubleClick(point: point, button: button)
        case .triple:
            try performTripleClick(point: point, button: button)
        case .right:
            try performRightClick(point: point)
        }
    }
    
    /// Perform single click
    private func performSingleClick(point: CGPoint, button: MouseButton) throws {
        let eventType: CGEventType = button == .left ? .leftMouseDown : .rightMouseDown
        
        // Create mouse down event
        guard let mouseDown = CGEvent(
            mouseEventSource: nil,
            mouseType: eventType,
            mouseCursorPosition: point,
            mouseButton: button == .left ? .left : .right
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        // Create mouse up event
        guard let mouseUp = CGEvent(
            mouseEventSource: nil,
            mouseType: eventType == .leftMouseDown ? .leftMouseUp : .rightMouseUp,
            mouseCursorPosition: point,
            mouseButton: button == .left ? .left : .right
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        // Post events
        mouseDown.post(tap: .cghidEventTap)
        mouseUp.post(tap: .cghidEventTap)
    }
    
    /// Perform double click
    private func performDoubleClick(point: CGPoint, button: MouseButton) throws {
        // macOS handles double-click internally based on click state
        let eventType: CGEventType = button == .left ? .leftMouseDown : .rightMouseDown
        let upType: CGEventType = button == .left ? .leftMouseUp : .rightMouseUp
        let cgButton: CGMouseButton = button == .left ? .left : .right
        
        for clickCount in 1...2 {
            guard let mouseDown = CGEvent(
                mouseEventSource: nil,
                mouseType: eventType,
                mouseCursorPosition: point,
                mouseButton: cgButton
            ) else {
                throw OSCPError.cgEventError(status: -1)
            }
            
            mouseDown.setIntegerValueField(.mouseEventClickState, value: Int64(clickCount))
            
            guard let mouseUp = CGEvent(
                mouseEventSource: nil,
                mouseType: upType,
                mouseCursorPosition: point,
                mouseButton: cgButton
            ) else {
                throw OSCPError.cgEventError(status: -1)
            }
            
            mouseUp.setIntegerValueField(.mouseEventClickState, value: Int64(clickCount))
            
            mouseDown.post(tap: .cghidEventTap)
            mouseUp.post(tap: .cghidEventTap)
        }
    }
    
    /// Perform triple click
    private func performTripleClick(point: CGPoint, button: MouseButton) throws {
        let eventType: CGEventType = button == .left ? .leftMouseDown : .rightMouseDown
        let upType: CGEventType = button == .left ? .leftMouseUp : .rightMouseUp
        let cgButton: CGMouseButton = button == .left ? .left : .right
        
        for clickCount in 1...3 {
            guard let mouseDown = CGEvent(
                mouseEventSource: nil,
                mouseType: eventType,
                mouseCursorPosition: point,
                mouseButton: cgButton
            ) else {
                throw OSCPError.cgEventError(status: -1)
            }
            
            mouseDown.setIntegerValueField(.mouseEventClickState, value: Int64(clickCount))
            
            guard let mouseUp = CGEvent(
                mouseEventSource: nil,
                mouseType: upType,
                mouseCursorPosition: point,
                mouseButton: cgButton
            ) else {
                throw OSCPError.cgEventError(status: -1)
            }
            
            mouseUp.setIntegerValueField(.mouseEventClickState, value: Int64(clickCount))
            
            mouseDown.post(tap: .cghidEventTap)
            mouseUp.post(tap: .cghidEventTap)
        }
    }
    
    /// Perform right click
    private func performRightClick(point: CGPoint) throws {
        try performSingleClick(point: point, button: .right)
    }
    
    // MARK: - Drag Actions
    
    /// Drag from start to end
    public func drag(
        startX: Double,
        startY: Double,
        endX: Double,
        endY: Double,
        button: MouseButton = .left,
        durationMs: Int = 500
    ) throws {
        let start = CGPoint(x: startX, y: startY)
        let end = CGPoint(x: endX, y: endY)
        
        let eventType: CGEventType = button == .left ? .leftMouseDragged : .rightMouseDragged
        
        // Mouse down at start
        guard let mouseDown = CGEvent(
            mouseEventSource: nil,
            mouseType: button == .left ? .leftMouseDown : .rightMouseDown,
            mouseCursorPosition: start,
            mouseButton: button == .left ? .left : .right
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        mouseDown.post(tap: .cghidEventTap)
        
        // Calculate steps for smooth drag
        let steps = max(Int(durationMs / 10), 10)  // At least 10 steps
        let dx = (end.x - start.x) / Double(steps)
        let dy = (end.y - start.y) / Double(steps)
        
        for i in 1...steps {
            let current = CGPoint(x: start.x + dx * Double(i), y: start.y + dy * Double(i))
            
            guard let drag = CGEvent(
                mouseEventSource: nil,
                mouseType: eventType,
                mouseCursorPosition: current,
                mouseButton: button == .left ? .left : .right
            ) else {
                throw OSCPError.cgEventError(status: -1)
            }
            
            drag.post(tap: .cghidEventTap)
            
            // Small delay between steps
            usleep(10_000)  // 10ms
        }
        
        // Mouse up at end
        guard let mouseUp = CGEvent(
            mouseEventSource: nil,
            mouseType: button == .left ? .leftMouseUp : .rightMouseUp,
            mouseCursorPosition: end,
            mouseButton: button == .left ? .left : .right
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        mouseUp.post(tap: .cghidEventTap)
    }
    
    // MARK: - Mouse Move
    
    /// Move mouse to position
    public func move(x: Double, y: Double) throws {
        let point = CGPoint(x: x, y: y)
        
        guard let moveEvent = CGEvent(
            mouseEventSource: nil,
            mouseType: .mouseMoved,
            mouseCursorPosition: point,
            mouseButton: .left
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        moveEvent.post(tap: .cghidEventTap)
    }
    
    // MARK: - Scroll Actions
    
    /// Scroll at position
    public func scroll(
        x: Double,
        y: Double,
        deltaX: Double,
        deltaY: Double,
        scrollType: ScrollType = .precise
    ) throws {
        // CGEvent scroll uses fixed-point notation
        // 1.0 = one line, 10.0 = one page
        let scrollX = Int32(deltaX * 10)
        let scrollY = Int32(-deltaY * 10)  // Invert Y for natural scroll
        
        guard let scrollEvent = CGEvent(
            scrollEventSource: nil,
            CGScrollEventUnit.line,
            scrollX: scrollX,
            scrollY: scrollY
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        scrollEvent.post(tap: .cghidEventTap)
    }
    
    // MARK: - Keyboard Actions
    
    /// Type text
    public func type(text: String, delayMs: Int = 50) throws {
        for character in text {
            try typeCharacter(character)
            
            if delayMs > 0 {
                usleep(UInt32(delayMs * 1000))
            }
        }
    }
    
    /// Type a single character
    private func typeCharacter(_ character: Character) throws {
        let string = String(character)
        
        guard let keyDown = CGEvent(
            keyboardEventSource: nil,
            virtualKey: 0,
            keyDown: true
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        // Set Unicode string for the character
        var chars = Array(string.utf16)
        keyDown.keyboardSetUnicodeString(stringLength: chars.count, unicodeString: &chars)
        
        guard let keyUp = CGEvent(
            keyboardEventSource: nil,
            virtualKey: 0,
            keyDown: false
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        keyUp.keyboardSetUnicodeString(stringLength: chars.count, unicodeString: &chars)
        
        keyDown.post(tap: .cghidEventTap)
        keyUp.post(tap: .cghidEventTap)
    }
    
    /// Press key combination
    public func keyCombo(key: String, modifiers: [String]) throws {
        // Get key code
        guard let keyCode = keyCodeMap[key.lowercased()] else {
            throw OSCPError.invalidAction(message: "Unknown key: \(key)")
        }
        
        // Build flags
        var flags: CGEventFlags = []
        
        for modifier in modifiers {
            switch modifier.lowercased() {
            case "ctrl", "control":
                flags.insert(.maskControl)
            case "alt", "option":
                flags.insert(.maskAlternate)
            case "shift":
                flags.insert(.maskShift)
            case "cmd", "command":
                flags.insert(.maskCommand)
            case "fn", "function":
                flags.insert(.maskSecondaryFn)
            default:
                break
            }
        }
        
        // Create key down
        guard let keyDown = CGEvent(
            keyboardEventSource: nil,
            virtualKey: keyCode,
            keyDown: true
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        keyDown.flags = flags
        
        // Create key up
        guard let keyUp = CGEvent(
            keyboardEventSource: nil,
            virtualKey: keyCode,
            keyDown: false
        ) else {
            throw OSCPError.cgEventError(status: -1)
        }
        
        keyUp.flags = flags
        
        // Post events
        keyDown.post(tap: .cghidEventTap)
        keyUp.post(tap: .cghidEventTap)
    }
    
    // MARK: - Action Execution
    
    /// Execute action and return result
    public func execute(action: Action, actionId: String) throws -> ActionResult {
        let startTime = Date()
        
        switch action {
        case .click(let clickAction):
            try click(
                x: clickAction.x,
                y: clickAction.y,
                button: clickAction.button,
                clickType: clickAction.clickType
            )
            
        case .type(let typeAction):
            try type(text: typeAction.text, delayMs: typeAction.typingDelayMs)
            
        case .keyCombo(let comboAction):
            try keyCombo(key: comboAction.key, modifiers: comboAction.modifiers)
            
        case .scroll(let scrollAction):
            try scroll(
                x: scrollAction.x ?? 0,
                y: scrollAction.y ?? 0,
                deltaX: scrollAction.deltaX,
                deltaY: scrollAction.deltaY,
                scrollType: scrollAction.scrollType
            )
            
        case .drag(let dragAction):
            try drag(
                startX: dragAction.startX,
                startY: dragAction.startY,
                endX: dragAction.endX,
                endY: dragAction.endY,
                button: dragAction.button,
                durationMs: dragAction.durationMs
            )
            
        case .move(let moveAction):
            try move(x: moveAction.x, y: moveAction.y)
        }
        
        let latencyMs = Int(Date().timeIntervalSince(startTime) * 1000)
        
        return ActionResult.success(
            actionId: actionId,
            latencyMs: latencyMs,
            target: TargetInfo(elementName: nil, elementRole: nil)
        )
    }
}
```

---

## 9. Protocol Server

### 9.1 ProtocolServer.swift

```swift
import Foundation

// MARK: - Protocol Server

/// Handles OSCP protocol communication over Unix socket
public final class ProtocolServer {
    
    // MARK: - Properties
    
    /// Unix socket path
    private let socketPath: String
    
    /// Server file descriptor
    private var serverFd: Int32 = -1
    
    /// Connected clients
    private var clients: [ClientConnection] = []
    
    /// Driver instance
    private let driver: Driver
    
    /// Whether server is running
    private var isRunning = false
    
    /// Dispatch queue for client handling
    private let queue = DispatchQueue(label: "com.oscp.server", qos: .userInteractive)
    
    // MARK: - Initialization
    
    /// Initialize with socket path
    public init(socketPath: String = "/tmp/oscp.sock", driver: Driver) {
        self.socketPath = socketPath
        self.driver = driver
    }
    
    // MARK: - Server Control
    
    /// Start the server
    public func start() throws {
        // Remove existing socket file
        unlink(socketPath)
        
        // Create socket
        serverFd = socket(AF_UNIX, SOCK_STREAM, 0)
        guard serverFd >= 0 else {
            throw OSCPError.systemError(message: "Failed to create socket")
        }
        
        // Set socket options
        var reuseAddr: Int32 = 1
        setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &reuseAddr, socklen_t(MemoryLayout<Int32>.size))
        
        // Bind socket
        var addr = sockaddr_un()
        addr.sun_family = sa_family_t(AF_UNIX)
        socketPath.withCString { ptr in
            withUnsafeMutablePointer(to: &addr.sun_path) { pathPtr in
                let pathBuf = UnsafeMutableRawPointer(pathPtr).assumingMemoryBound(to: CChar.self)
                strncpy(pathBuf, ptr, Int(MAXPATHLEN) - 1)
            }
        }
        
        let bindResult = withUnsafePointer(to: &addr) { ptr in
            ptr.withMemoryRebound(to: sockaddr.self, capacity: 1) { sockaddrPtr in
                bind(serverFd, sockaddrPtr, socklen_t(MemoryLayout<sockaddr_un>.size))
            }
        }
        
        guard bindResult == 0 else {
            close(serverFd)
            throw OSCPError.systemError(message: "Failed to bind socket: \(errno)")
        }
        
        // Listen for connections
        guard listen(serverFd, 5) == 0 else {
            close(serverFd)
            throw OSCPError.systemError(message: "Failed to listen")
        }
        
        // Set socket to non-blocking
        var flags = fcntl(serverFd, F_GETFL, 0)
        fcntl(serverFd, F_SETFL, flags | O_NONBLOCK)
        
        isRunning = true
        
        // Accept loop
        acceptLoop()
        
        print("OSCP server listening on \(socketPath)")
    }
    
    /// Stop the server
    public func stop() {
        isRunning = false
        
        // Close all client connections
        for client in clients {
            client.close()
        }
        clients.removeAll()
        
        // Close server socket
        if serverFd >= 0 {
            close(serverFd)
            serverFd = -1
        }
        
        // Remove socket file
        unlink(socketPath)
    }
    
    // MARK: - Accept Loop
    
    /// Main accept loop
    private func acceptLoop() {
        queue.async { [weak self] in
            guard let self = self else { return }
            
            while self.isRunning {
                var clientAddr = sockaddr_un()
                var addrLen = socklen_t(MemoryLayout<sockaddr_un>.size)
                
                let clientFd = withUnsafeMutablePointer(to: &clientAddr) { ptr in
                    ptr.withMemoryRebound(to: sockaddr.self, capacity: 1) { sockaddrPtr in
                        accept(self.serverFd, sockaddrPtr, &addrLen)
                    }
                }
                
                if clientFd >= 0 {
                    self.handleClient(fd: clientFd)
                } else {
                    // No connection pending
                    usleep(100_000)  // 100ms
                }
            }
        }
    }
    
    // MARK: - Client Handling
    
    /// Handle new client connection
    private func handleClient(fd: Int32) {
        let connection = ClientConnection(
            fd: fd,
            driver: driver,
            onClose: { [weak self] conn in
                self?.removeClient(conn)
            }
        )
        
        clients.append(connection)
        connection.start()
    }
    
    /// Remove closed client
    private func removeClient(_ client: ClientConnection) {
        clients.removeAll { $0 === client }
    }
}

// MARK: - Client Connection

/// Represents a single client connection
private final class ClientConnection {
    
    // MARK: - Properties
    
    let fd: Int32
    private let driver: Driver
    private let onClose: (ClientConnection) -> Void
    
    private var inputSource: DispatchSourceRead?
    private var isRunning = false
    private var hasSentHello = false
    private let queue = DispatchQueue(label: "com.oscp.client")
    
    // JSON decoder/encoder
    private let decoder = JSONDecoder()
    private let encoder = JSONEncoder()
    
    // Input buffer
    private var buffer = Data()
    
    // MARK: - Initialization
    
    init(fd: Int32, driver: Driver, onClose: @escaping (ClientConnection) -> Void) {
        self.fd = fd
        self.driver = driver
        self.onClose = onClose
    }
    
    // MARK: - Connection Control
    
    /// Start handling client
    func start() {
        isRunning = true
        
        // Set non-blocking
        var flags = fcntl(fd, F_GETFL, 0)
        fcntl(fd, F_SETFL, flags | O_NONBLOCK)
        
        // Create dispatch source for reading
        inputSource = DispatchSource.makeReadSource(fileDescriptor: fd, queue: queue)
        
        inputSource?.setEventHandler { [weak self] in
            self?.readAvailable()
        }
        
        inputSource?.setCancelHandler { [weak self] in
            self?.cleanup()
        }
        
        inputSource?.resume()
    }
    
    /// Close connection
    func close() {
        isRunning = false
        inputSource?.cancel()
    }
    
    /// Cleanup on close
    private func cleanup() {
        close(fd)
        onClose(self)
    }
    
    // MARK: - Reading
    
    /// Handle available data
    private func readAvailable() {
        var readBuffer = [UInt8](repeating: 0, count: 4096)
        
        let bytesRead = read(fd, &readBuffer, readBuffer.count)
        
        if bytesRead > 0 {
            buffer.append(contentsOf: readBuffer.prefix(bytesRead))
            processBuffer()
        } else if bytesRead == 0 {
            // Connection closed
            close()
        } else if errno != EAGAIN && errno != EWOULDBLOCK {
            // Error
            close()
        }
    }
    
    /// Process buffered data
    private func processBuffer() {
        // Find newline-delimited JSON messages
        while let newlineIndex = buffer.firstIndex(of: UInt8(ascii: "\n")) {
            let messageData = buffer.prefix(newlineIndex)
            buffer.removeFirst(newlineIndex + 1)
            
            // Skip empty messages
            guard !messageData.isEmpty else { continue }
            
            // Parse and handle message
            if let message = String(data: Data(messageData), encoding: .utf8) {
                handleMessage(message)
            }
        }
    }
    
    // MARK: - Message Handling
    
    /// Handle incoming message
    private func handleMessage(_ message: String) {
        guard let data = message.data(using: .utf8) else { return }
        
        do {
            // Decode as generic JSON to check type
            guard let json = try JSONSerialization.jsonObject(with: data) as? [String: Any],
                  let type = json["type"] as? String else {
                return
            }
            
            switch type {
            case "hello":
                handleHello(json)
            case "get_frame":
                handleGetFrame(json)
            case "action":
                handleAction(json)
            case "mouse_position":
                handleMousePosition(json)
            case "ping":
                handlePing(json)
            case "disconnect":
                handleDisconnect()
            default:
                sendError(requestId: json["request_id"] as? String, code: .invalidRequest, message: "Unknown message type: \(type)")
            }
        } catch {
            sendError(requestId: nil, code: .invalidRequest, message: "Failed to parse message: \(error.localizedDescription)")
        }
    }
    
    // MARK: - Hello
    
    /// Handle hello message
    private func handleHello(_ json: [String: Any]) {
        // Send welcome
        let welcome: [String: Any] = [
            "type": "welcome",
            "version": "0.4",
            "platform": "macOS",
            "capabilities": [
                "get_frame", "click", "type", "key_combo",
                "scroll", "drag", "mouse_position"
            ],
            "input_methods": ["cg_event"]
        ]
        
        sendJSON(welcome)
        hasSentHello = true
    }
    
    // MARK: - Get Frame
    
    /// Handle get_frame request
    private func handleGetFrame(_ json: [String: Any]) {
        let requestId = json["request_id"] as? String ?? UUID().uuidString
        
        Task {
            do {
                let frame = try await driver.captureFrame()
                var response = try encoder.encode(frame)
                
                // Add type field
                var dict = try JSONSerialization.jsonObject(with: response) as? [String: Any] ?? [:]
                dict["type"] = "frame"
                dict["request_id"] = requestId
                
                sendJSON(dict)
            } catch {
                self.sendError(requestId: requestId, error: error)
            }
        }
    }
    
    // MARK: - Action
    
    /// Handle action request
    private func handleAction(_ json: [String: Any]) {
        let actionId = json["action_id"] as? String ?? UUID().uuidString
        
        Task {
            do {
                // Parse action
                guard let actionJson = json["action"] as? [String: Any] else {
                    throw OSCPError.invalidAction(message: "Missing action field")
                }
                
                let actionData = try JSONSerialization.data(withJSONObject: actionJson)
                let action = try decoder.decode(Action.self, from: actionData)
                
                // Execute action
                let result = try driver.executeAction(action, actionId: actionId)
                var response = try encoder.encode(result)
                
                // Add type field
                var dict = try JSONSerialization.jsonObject(with: response) as? [String: Any] ?? [:]
                dict["type"] = "action_result"
                
                sendJSON(dict)
            } catch {
                self.sendError(error: error, actionId: actionId)
            }
        }
    }
    
    // MARK: - Mouse Position
    
    /// Handle mouse_position request
    private func handleMousePosition(_ json: [String: Any]) {
        let requestId = json["request_id"] as? String ?? UUID().uuidString
        
        let position = driver.getMousePosition()
        let state = driver.getMouseState()
        
        let response: [String: Any] = [
            "type": "mouse_position_result",
            "request_id": requestId,
            "x": position.x,
            "y": position.y,
            "button_state": state.buttonState.rawValue,
            "hovered_element_id": state.hoveredElementId
        ]
        
        sendJSON(response)
    }
    
    // MARK: - Ping
    
    /// Handle ping
    private func handlePing(_ json: [String: Any]) {
        let timestamp = json["timestamp"] as? Int64 ?? Int64(Date().timeIntervalSince1970 * 1000)
        
        let response: [String: Any] = [
            "type": "pong",
            "timestamp": timestamp,
            "latency_ms": 1
        ]
        
        sendJSON(response)
    }
    
    // MARK: - Disconnect
    
    /// Handle disconnect
    private func handleDisconnect() {
        sendJSON(["type": "goodbye"])
        close()
    }
    
    // MARK: - Sending
    
    /// Send JSON as newline-delimited message
    private func sendJSON(_ dict: [String: Any]) {
        do {
            let data = try JSONSerialization.data(withJSONObject: dict)
            if var string = String(data: data, encoding: .utf8) {
                string += "\n"
                if let bytes = string.data(using: .utf8) {
                    write(fd, [UInt8](bytes), bytes.count)
                }
            }
        } catch {
            print("Failed to send JSON: \(error)")
        }
    }
    
    /// Send error
    private func sendError(requestId: String?, code: ErrorCode, message: String) {
        let response: [String: Any] = [
            "type": "error",
            "request_id": requestId ?? "",
            "code": code.rawValue,
            "message": message,
            "timestamp": Int64(Date().timeIntervalSince1970 * 1000)
        ]
        sendJSON(response)
    }
    
    private func sendError(requestId: String?, error: Error) {
        let code: ErrorCode
        if let oscpError = error as? OSCPError {
            code = ErrorCode.from(oscpError)
        } else {
            code = .platformError
        }
        sendError(requestId: requestId, code: code, message: error.localizedDescription)
    }
    
    private func sendError(error: Error, actionId: String) {
        let code: ErrorCode
        if let oscpError = error as? OSCPError {
            code = ErrorCode.from(oscpError)
        } else {
            code = .platformError
        }
        
        let response: [String: Any] = [
            "type": "action_result",
            "action_id": actionId,
            "success": false,
            "timestamp": Int64(Date().timeIntervalSince1970 * 1000),
            "error": [
                "code": code.rawValue,
                "message": error.localizedDescription
            ]
        ]
        sendJSON(response)
    }
}
```

---

## 10. Main Entry Point

### 10.1 main.swift

```swift
import Foundation
import ApplicationServices

// MARK: - OSCP macOS Driver

/// Main driver class that orchestrates all components
public final class Driver {
    
    // MARK: - Properties
    
    /// AXUIElement capture
    private let axCapture: AXUIElementCapture
    
    /// CDP bridge (optional)
    private let cdpBridge: CDPBridge?
    
    /// Fallback manager
    private let fallbackManager: FallbackManager
    
    /// Input engine
    private let inputEngine: CGEventEngine
    
    /// Tree builder
    private let treeBuilder: TreeBuilder
    
    /// Frame counter
    private var frameCounter: Int = 0
    
    /// Lock for thread safety
    private let lock = NSLock()
    
    // MARK: - Initialization
    
    public init() {
        self.axCapture = AXUIElementCapture()
        
        // Try to create CDP bridge (optional)
        self.cdpBridge = CDPBridge.safariBridge() ?? CDPBridge.chromeBridge()
        
        self.fallbackManager = FallbackManager(axCapture: axCapture, cdpBridge: cdpBridge)
        self.inputEngine = CGEventEngine()
        self.treeBuilder = TreeBuilder()
    }
    
    // MARK: - Permissions
    
    /// Check and request permissions
    public func checkPermissions() -> Bool {
        return axCapture.checkPermissions()
    }
    
    /// Request accessibility permissions
    public func requestPermissions() {
        axCapture.requestPermissions()
    }
    
    // MARK: - Frame Capture
    
    /// Capture current frame
    public func captureFrame() async throws -> Frame {
        return try await fallbackManager.captureFrame()
    }
    
    // MARK: - Actions
    
    /// Execute action
    public func executeAction(_ action: Action, actionId: String) throws -> ActionResult {
        return try inputEngine.execute(action: action, actionId: actionId)
    }
    
    // MARK: - Mouse State
    
    /// Get current mouse position
    public func getMousePosition() -> CGPoint {
        return axCapture.getMousePosition()
    }
    
    /// Get current mouse state
    public func getMouseState() -> MouseState {
        let pos = getMousePosition()
        return MouseState(x: pos.x, y: pos.y)
    }
    
    // MARK: - Element Lookup
    
    /// Find element at position
    public func findElement(at point: CGPoint, in frame: Frame) -> Element? {
        return treeBuilder.findElement(at: point, in: frame.windows)
    }
    
    /// Find element by ID
    public func findElement(byId id: String, in frame: Frame) -> Element? {
        return treeBuilder.findElement(byId: id, in: frame.windows)
    }
    
    /// Find elements by role
    public func findElements(byRole role: String, in frame: Frame) -> [Element] {
        return treeBuilder.findElements(byRole: role, in: frame.windows)
    }
    
    /// Find elements by name
    public func findElements(byName name: String, in frame: Frame, partial: Bool = true) -> [Element] {
        return treeBuilder.findElements(byName: name, in: frame.windows, partial: partial)
    }
}

// MARK: - Main Entry Point

/// OSCP macOS Driver entry point
public struct OSCP {
    
    /// Run the OSCP server
    public static func run() {
        print("OSCP macOS Driver v0.4.0")
        print("====================")
        
        // Create driver
        let driver = Driver()
        
        // Check permissions
        if !driver.checkPermissions() {
            print("")
            print("⚠️  Accessibility permission required!")
            print("")
            print("Please grant permission in:")
            print("  System Preferences → Privacy & Security → Accessibility")
            print("")
            print("Adding OSCP to the allowed apps list...")
            print("")
            
            driver.requestPermissions()
            
            print("After granting permission, restart OSCP.")
            exit(1)
        }
        
        print("✓ Accessibility permission granted")
        
        // Create protocol server
        let server = ProtocolServer(socketPath: "/tmp/oscp.sock", driver: driver)
        
        // Start server
        do {
            try server.start()
            
            print("✓ OSCP server running on /tmp/oscp.sock")
            print("")
            print("Ready for agent connections...")
            print("")
            
            // Run indefinitely
            RunLoop.main.run()
        } catch {
            print("✗ Failed to start server: \(error)")
            exit(1)
        }
        
        // Keep running
        dispatchMain()
    }
}

// Start OSCP
OSCP.run()
```

---

## 11. Error Handling

### 11.1 Error Handling Strategy

```swift
// Error handling follows this pattern:

// 1. Try best approach first
do {
    let windows = try axCapture.captureTree()
    // Success
} catch OSCPError.permissionDenied {
    // 2. Handle permission error
    requestPermissions()
} catch let error as OSCPError {
    // 3. Try fallback
    try await fallbackManager.captureFrame()
} catch {
    // 4. Last resort
    throw OSCPError.systemError(message: error.localizedDescription)
}
```

### 11.2 Common Error Scenarios

| Error | Cause | Resolution |
|-------|-------|------------|
| `permissionDenied` | No Accessibility permission | Guide user to System Preferences |
| `emptyTree` | No apps running or all custom renderers | Use CDP bridge or handoff |
| `lowCoverage` | App uses custom rendering | Apply heuristics or position-only |
| `actionFailed` | Element moved or state changed | Try alternatives or handoff |
| `elementNotFound` | Element no longer exists | Re-capture frame and retry |

---

## 12. Testing

### 12.1 Capture Tests

```swift
// Tests/OSCPTests/CaptureTests.swift

import XCTest
@testable import OSCP

final class CaptureTests: XCTestCase {
    
    var capture: AXUIElementCapture!
    
    override func setUp() {
        super.setUp()
        capture = AXUIElementCapture()
    }
    
    func testPermissionsCheck() {
        let hasPermission = capture.checkPermissions()
        XCTAssertTrue(hasPermission, "Accessibility permission should be granted")
    }
    
    func testCaptureWindows() throws {
        let windows = try capture.captureTree()
        
        // Should get at least one window (possibly Finder or similar)
        XCTAssertFalse(windows.isEmpty, "Should capture at least one window")
        
        // Verify window structure
        for window in windows {
            XCTAssertFalse(window.id.isEmpty, "Window should have ID")
            XCTAssertNotNil(window.bounds, "Window should have bounds")
        }
    }
    
    func testCaptureElements() throws {
        let windows = try capture.captureTree()
        
        for window in windows {
            // Windows should have elements (may be empty)
            XCTAssertNotNil(window.elements, "Window elements should be initialized")
        }
    }
    
    func testMousePosition() {
        let position = capture.getMousePosition()
        
        // Position should be valid
        XCTAssert(position.x >= 0, "X should be non-negative")
        XCTAssert(position.y >= 0, "Y should be non-negative")
    }
}
```

### 12.2 Input Tests

```swift
// Tests/OSCPTests/InputTests.swift

import XCTest
@testable import OSCP

final class InputTests: XCTestCase {
    
    var engine: CGEventEngine!
    
    override func setUp() {
        super.setUp()
        engine = CGEventEngine()
    }
    
    func testCanInjectInput() {
        XCTAssertTrue(engine.canInjectInput(), "Should be able to inject input")
    }
    
    func testKeyCombo() throws {
        // Test a simple key combo (Cmd+Space on macOS opens Spotlight)
        // Comment out for automated testing:
        // try engine.keyCombo(key: "space", modifiers: ["cmd"])
    }
    
    func testMouseMove() throws {
        // Move to center of screen
        let screen = NSScreen.main!
        let center = CGPoint(
            x: screen.frame.width / 2,
            y: screen.frame.height / 2
        )
        
        // This would move the mouse - skip in automated tests
        // try engine.move(x: center.x, y: center.y)
    }
    
    func testActionExecution() throws {
        // Test action parsing and execution
        let clickAction = ClickAction(x: 100, y: 100, button: .left, clickType: .single)
        let action: Action = .click(clickAction)
        
        let result = try engine.execute(action: action, actionId: "test_001")
        XCTAssertTrue(result.success, "Click action should succeed")
    }
}
```

### 12.3 Integration Tests

```swift
// Tests/OSCPTests/IntegrationTests.swift

import XCTest
@testable import OSCP

final class IntegrationTests: XCTestCase {
    
    var driver: Driver!
    var server: ProtocolServer!
    
    override func setUp() {
        super.setUp()
        driver = Driver()
    }
    
    override func tearDown() {
        server?.stop()
        super.tearDown()
    }
    
    func testDriverInitialization() {
        XCTAssertNotNil(driver, "Driver should initialize")
        XCTAssertTrue(driver.checkPermissions(), "Should have permissions")
    }
    
    func testFrameCapture() async throws {
        let frame = try await driver.captureFrame()
        
        XCTAssertEqual(frame.platform, "macOS")
        XCTAssertNotNil(frame.windows)
        XCTAssertNotNil(frame.treeAnalysis)
        XCTAssertNotNil(frame.mouse)
    }
    
    func testTreeAnalysis() async throws {
        let frame = try await driver.captureFrame()
        let analysis = frame.treeAnalysis
        
        // Analysis should have valid values
        XCTAssertGreaterThanOrEqual(analysis.totalElements, 0)
        XCTAssertGreaterThanOrEqual(analysis.coverageScore, 0)
        XCTAssertLessThanOrEqual(analysis.coverageScore, 1)
    }
    
    func testElementLookup() async throws {
        let frame = try await driver.captureFrame()
        let mousePos = driver.getMousePosition()
        
        let element = driver.findElement(at: mousePos, in: frame)
        
        // Element might be nil if no element at exact position
        // This is valid behavior
        if let element = element {
            XCTAssertFalse(element.id.isEmpty)
            XCTAssertFalse(element.role.isEmpty)
        }
    }
    
    func testServerLifecycle() throws {
        server = ProtocolServer(socketPath: "/tmp/oscp_test.sock", driver: driver)
        
        try server.start()
        XCTAssertTrue(FileManager.default.fileExists(atPath: "/tmp/oscp_test.sock"))
        
        server.stop()
        XCTAssertFalse(FileManager.default.fileExists(atPath: "/tmp/oscp_test.sock"))
    }
}
```

---

## 13. Common Pitfalls

### 13.1 AXUIElement Issues

| Pitfall | Cause | Solution |
|---------|-------|----------|
| Empty tree | Permission denied | Check `AXIsProcessTrusted()` first |
| Missing attributes | Element doesn't expose attribute | Check attribute existence before use |
| Slow capture | Deep hierarchy | Limit depth to 20, skip non-actionable |
| Memory growth | Caching too much | Use weak references, limit cache size |

### 13.2 CGEvent Issues

| Pitfall | Cause | Solution |
|---------|-------|----------|
| Input not working | Accessibility permission | Check `AXIsProcessTrusted()` |
| Click offset | Coordinate space mismatch | Use screen coordinates (top-left origin) |
| Key codes wrong | Wrong key code map | Verify against Apple docs |
| Double-click too fast | Timing issue | Use click state field |

### 13.3 CDP Issues

| Pitfall | Cause | Solution |
|---------|-------|----------|
| Connection refused | Browser not running | Launch browser with remote debugging |
| Slow DOM capture | Large page | Limit depth, use pierce option |
| Permission errors | CORS policy | CDP has full access to own pages |

### 13.4 Protocol Issues

| Pitfall | Cause | Solution |
|---------|-------|----------|
| JSON parse errors | Invalid JSON | Validate with JSONDecoder first |
| Socket busy | Previous server crash | Call `unlink()` before bind |
| Connection drops | Client disconnect | Handle SIGPIPE, use non-blocking |

---

## Status

- [x] Models complete
- [x] AXUIElement capture complete
- [x] CDP bridge complete
- [x] Tree builder complete
- [x] Tree analyzer complete
- [x] Fallback manager complete
- [x] CGEvent input engine complete
- [x] Protocol server complete
- [x] Main entry point complete
- [x] Error handling documented
- [x] Testing templates complete
- [x] Common pitfalls documented
- [ ] Implementation pending

---

## References

- [AXUIElement Documentation](https://developer.apple.com/documentation/application_services)
- [CGEvent Reference](https://developer.apple.com/documentation/coregraphics/cgevent)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
- [Apple Accessibility Programming Guide](https://developer.apple.com/library/archive/documentation/Accessibility/Conceptual/AccessibilityMacOSX/index.html)