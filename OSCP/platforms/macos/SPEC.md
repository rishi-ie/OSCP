# OSCP macOS Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The macOS platform driver provides OSCP protocol implementation for macOS 12+ (Monterey and later). It uses AXUIElement with a comprehensive fallback hierarchy to handle problematic apps.

**Principle:** Deterministic. Error-resilient. Graceful degradation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   AXUIElement │  │ Window Server │  │   Input Engine   │   │
│  │  Accessibility │  │  Layer Tree   │  │   (CGEvent)      │   │
│  └───────┬──────┘  └───────┬──────┘  └────────┬─────────┘   │
│          │                 │                  │               │
│          └──────────┬──────┘                  │               │
│                     │                         │               │
│              ┌──────▼──────┐                   │               │
│              │    Tree     │◄──────────────────┘               │
│              │   Builder   │                                   │
│              │   + Error   │                                   │
│              │   Handler   │                                   │
│              └──────┬──────┘                                   │
│                     │                                          │
│              ┌──────▼──────┐                                   │
│              │  Protocol   │                                   │
│              │  Server     │                                   │
│              │  (Unix)     │                                   │
│              └─────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Primary Capture: AXUIElement

### What It Captures

```swift
// AXUIElement APIs used:
struct AccessibilityElement {
    var role: String           // kAXButtonRole, kAXMenuRole, etc.
    var title: String           // kAXTitleAttribute
    var value: String          // kAXValueAttribute
    var position: CGPoint       // kAXPositionAttribute
    var size: CGSize           // kAXSizeAttribute
    var children: [AXUIElement] // kAXChildrenAttribute
    var states: [String]       // kAXEnabledAttribute, etc.
}
```

### Coverage

| App Type | Works? | Notes |
|----------|--------|-------|
| AppKit apps | ✅ | Full AXUIElement support |
| SwiftUI apps | ✅ | Full AXUIElement support |
| Carbon apps | ⚠️ | Basic support |
| Metal apps | ⚠️ | Container only |
| OpenGL apps | ❌ | No AXUIElement |

**Primary Coverage: ~90%**

---

## Fallback Hierarchy

### Level 1: AXUIElement (Primary)

```swift
func captureAXUI(window: AXUIElement) -> ElementTree {
    // Standard accessibility extraction
    // Returns full element tree with roles, titles, bounds
}
```

### Level 2: CDP Bridge (Browser/Electron)

```swift
func captureCDP(app: AppInfo) async -> ElementTree {
    // Connect to Safari WebInspector or Electron CDP
    // Extract DOM tree with bounding boxes
    // Covers: Safari, Chrome, Electron apps
}
```

### Level 3: Window Server Layer Tree

```swift
func captureWindowServer() -> [WindowInfo] {
    // CGWindowListCopyWindowInfo
    // Layer tree from Window Server
    // No semantic info, just positions
}
```

### Level 4: Position-Only Mode

```swift
func positionOnlyMode(window: Window) -> EmptyTree {
    // Return window bounds only
    // Agent must learn through exploration
    // Triggered when coverage_score < 0.3
}
```

### Level 5: Human Handoff

```swift
func humanHandoff(reason: String, attempts: Int) -> HandoffRequest {
    // Report failure
    // Ask for guidance
    // Return options to agent
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
    confidence: Confidence
}

enum Confidence {
    case high    // > 0.8
    case medium   // 0.5-0.8
    case low      // 0.3-0.5
    case none     // < 0.2
}
```

### Detection Triggers

```swift
func analyzeTree(_ tree: ElementTree) -> TreeAnalysis {
    let namedRatio = Float(tree.namedCount) / Float(tree.totalCount)
    let coverage = tree.coveredArea / tree.windowArea
    
    if coverage < 0.3 {
        return TreeAnalysis(confidence: .none, fallback: "position_only")
    }
    
    if namedRatio < 0.5 {
        return TreeAnalysis(confidence: .low, fallback: "heuristics")
    }
    
    // ... etc
}
```

---

## Output Format

### Standard Frame

```json
{
  "type": "render_tree",
  "platform": "macos",
  "windows": [
    {
      "id": "win_12345",
      "title": "Visual Studio Code",
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
          "source": "axui"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH"
  }
}
```

### Fallback Frame

```json
{
  "type": "render_tree",
  "platform": "macos",
  "windows": [
    {
      "id": "win_99999",
      "title": "CustomGame",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "elements": [],
      "fallback_active": true,
      "fallback_method": "position_only",
      "fallback_reason": "metal_renderer_detected"
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.05,
    "named_elements": 0,
    "confidence": "NONE",
    "recommended_action": "human_handoff"
  }
}
```

---

## Action Result with Confidence

```json
{
  "type": "action_result",
  "action_id": "act_001",
  "success": true,
  "confidence": 0.95,
  "source": "axui",
  "error": null
}
```

```json
{
  "type": "action_result",
  "action_id": "act_002",
  "success": false,
  "confidence": 0.2,
  "source": "position",
  "error": {
    "code": "EMPTY_TREE",
    "message": "Metal renderer detected, no accessibility",
    "alternatives": [
      {"bounds": {"x": 960, "y": 540}, "confidence": 0.3}
    ],
    "recommended_action": "explore_and_confirm"
  }
}
```

---

## Protocol Implementation

### Transport

Default: `unix:///tmp/oscp.sock`

### Capabilities

```json
{
  "platform": "macos",
  "driver": "oscp-macos-v0.2",
  "semantic_apis": ["axui", "window_server"],
  "fallback_methods": ["cdp", "position_only", "human_handoff"],
  "capabilities": ["render_tree", "actions", "events", "error_handling"],
  "features": {
    "element_tree": true,
    "cdp_bridge": true,
    "position_only": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Permission Model

### Required Permissions

**"Screen Recording"** (misleading name) — Enables:
- AXUIElement for all apps
- CGWindowListCopyWindowInfo
- Window Server layer tree access

**Accessibility** — For input injection (CGEvent).

### Permission Flow

```swift
// Check permissions
let hasAX = AXIsProcessTrusted()
let hasScreenRecording = CGPreflightScreenCaptureAccess()

// If missing, request
if !hasAX {
    AXIsProcessTrustedWithOptions(nil)
}
if !hasScreenRecording {
    CGRequestScreenCaptureAccess()
}
```

---

## Installation

```bash
brew install oscp
```

---

## Status

🚧 **V1 Target:** AXUIElement + fallback hierarchy

---

## References

- [AXUIElement](https://developer.apple.com/documentation/application-services)
- [CGWindowListCopyWindowInfo](https://developer.apple.com/documentation/application-services/1503048-cgwindowlistcopywindowinfo)
- [CGEvent](https://developer.apple.com/documentation/coregraphics/cgevent)