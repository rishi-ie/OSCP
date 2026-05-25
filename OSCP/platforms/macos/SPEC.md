# OSCP macOS Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The macOS platform driver wraps AXUIElement (Apple's accessibility API) into a real-time, streaming, error-resilient interface.

**Key Insight:** AXUIElement already extracts the semantic tree. OSCP adds streaming, fallbacks, and agent-friendly output.

---

## Wrapped Technology

### AXUIElement (Apple's Accessibility API)

```swift
// Native macOS accessibility:
// AXUIElement is Apple's standard accessibility API
// Used by: VoiceOver, Switch Control, accessibility tools

// Existing wrappers:
// - pyax (Python)
// - ax-element (Rust)
// - accessibility-service (native)
```

### What AXUIElement Provides

```swift
struct AccessibilityElement {
    var role: String           // kAXButtonRole, kAXMenuRole, etc.
    var title: String           // kAXTitleAttribute
    var value: String           // kAXValueAttribute
    var position: CGPoint       // kAXPositionAttribute
    var size: CGSize           // kAXSizeAttribute
    var children: [AXUIElement] // kAXChildrenAttribute
    var states: [String]       // kAXEnabledAttribute, etc.
}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  AXUIElement     │  │   CDP Bridge    │  │   Input      │  │
│  │  (wrapped)       │  │  (Safari/Chrome)│  │   Engine     │  │
│  │  EXISTING        │  │  EXISTING       │  │   (CGEvent)  │  │
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬──────┘  │
│           │                     │                   │          │
│           └──────────┬──────────┘                   │          │
│                      │                              │          │
│               ┌──────▼──────┐                       │          │
│               │   Stream    │◄──────────────────────┘          │
│               │   + Error   │                              │
│               │   Handler   │                              │
│               └──────┬──────┘                              │
│                      │                                     │
│               ┌──────▼──────┐                              │
│               │  Protocol   │                              │
│               │  Server     │                              │
│               │  (Unix)     │                              │
│               └─────────────┘                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Fallback Hierarchy

### Level 1: AXUIElement (Primary)

```swift
// Wrapped from existing tools
func captureAXUI() -> ElementTree {
    let app = AXUIElementCreateApplication(pid, kAXToolbarButtonRole)
    return extractElementTree(app)
}
```

**Coverage: ~90%**

### Level 2: CDP Bridge

```swift
// Safari WebInspector / Chrome DevTools
func captureCDP() -> ElementTree {
    // Connect to CDP port
    // Extract DOM with bounding boxes
}
```

**Coverage: +4% (Safari, Chrome, Electron)**

### Level 3: Position-Only Mode

```swift
// For Metal/OpenGL games
func positionOnly(window: Window) -> EmptyTree {
    return WindowBounds(window)
}
```

### Level 4: Human Handoff

```swift
func humanHandoff(reason: String) -> HandoffRequest {
    return HandoffRequest(...)
}
```

---

## Error Detection

### Tree Quality Analysis

```swift
struct TreeAnalysis {
    coverageScore: Float        // 0.0-1.0
    namedElements: Int
    unlabeledElements: Int
    avgDepth: Float
    confidence: Confidence      // HIGH, MEDIUM, LOW, NONE
}
```

### Detection Rules

```swift
if coverageScore < 0.3 { triggerFallback() }
if namedRatio < 0.5 { triggerLowConfidence() }
if rootHasNoChildren { triggerCustomRenderer() }
```

---

## Revised Complexity

| Component | Complexity | Notes |
|-----------|------------|-------|
| **AXUIElement wrapping** | Low | Already exists |
| **Real-time streaming** | Medium | 2-3 weeks |
| **Error handling** | Low | 1 week |
| **Input engine** | Low | 1 week |
| **Testing** | Low | 1-2 weeks |

**Time Estimate: 4-6 weeks**

---

## Installation

```bash
brew install oscp
```

### Permissions

- **Screen Recording** — Enables AXUIElement access (misleading name)
- **Accessibility** — Enables input injection (CGEvent)

---

## Status

🚧 **V1 Target:** Wrapped AXUIElement + streaming + fallbacks
⏱️ **Time:** 4-6 weeks

---

## References

- [AXUIElement Documentation](https://developer.apple.com/documentation/application-services)
- [pyax](https://github.com/pyax/pyax) - Python AXUIElement wrapper
- [ax-element](https://github.com/nalexand/ax-element) - Rust AXUIElement bindings