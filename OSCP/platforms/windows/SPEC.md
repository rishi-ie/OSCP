# OSCP Windows Platform Driver Specification

**Version:** 0.2.0
**Status:** Draft

---

## Overview

The Windows platform driver provides OSCP protocol implementation for Windows 10/11. It uses UI Automation (UIA) and Win32 APIs with a comprehensive fallback hierarchy to handle problematic apps.

**Principle:** Deterministic. Error-resilient. Graceful degradation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Driver                             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │     UIA      │  │    Win32     │  │   Input Engine   │   │
│  │  Element Tree │  │  Window Enum │  │     (Rust)       │   │
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
│              │  (TCP)      │                                   │
│              └─────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Primary Capture: UIAutomation

### What It Captures

```rust
// UIA APIs used:
struct UIAElement {
    pub element_type: ElementType,      // Button, Menu, etc.
    pub name: String,                   // Display text
    pub bounds: Rect,                   // Position and size
    pub state: Vec<ElementState>,       // Enabled, Focused, etc.
    pub children: Vec<UIAElement>,      // Child elements
    pub automation_id: String,         // Internal ID
}
```

### Coverage

| App Type | Works? | Notes |
|----------|--------|-------|
| Win32 apps | ✅ | Full UIA support |
| WPF apps | ✅ | Full UIA support |
| UWP apps | ✅ | Full UIA support |
| Electron (Chromium) | ✅ | UIA + CDP fallback |
| WinForms | ⚠️ | Basic UIA |
| Legacy Win32 | ⚠️ | Limited |
| DirectX/Vulkan | ❌ | No UIA |

**Primary Coverage: ~85%**

---

## Fallback Hierarchy

### Level 1: UIA (Primary)

```rust
fn capture_uia(hwnd: HWND) -> Result<ElementTree> {
    // Standard UIA extraction
    // Returns full element tree with names, types, bounds
}
```

### Level 2: CDP Bridge (Electron/Browser)

```rust
async fn capture_cdp(app_name: &str) -> Result<ElementTree> {
    // Connect to CDP port (localhost:9222)
    // Extract DOM tree with bounding boxes
    // Same data as UIA, different source
    
    // Covers: Chrome, Edge, Firefox, VS Code, Slack, Discord
}
```

### Level 3: Win32 Enumeration (Fallback)

```rust
fn capture_win32() -> PartialTree {
    // EnumWindows for window list
    // GetWindowRect for positions
    // GetWindowText for titles
    // No element tree, just window metadata
}
```

### Level 4: Position-Only Mode

```rust
fn position_only_mode(window: Window) -> EmptyTree {
    // Return window bounds only
    // Agent must learn through exploration
    // Triggered when coverage_score < 0.3
}
```

### Level 5: Human Handoff

```rust
fn human_handoff(reason: &str, attempts: u32) -> HandoffRequest {
    // Report failure
    // Ask for guidance
    // Return options to agent
}
```

---

## Error Detection

### Tree Quality Analysis

```rust
struct TreeAnalysis {
    coverage_score: f32,      // 0.0-1.0
    named_elements: u32,
    unlabeled_elements: u32,
    avg_depth: f32,
    confidence: Confidence,
}

enum Confidence {
    HIGH,    // > 0.8
    MEDIUM,  // 0.5-0.8
    LOW,     // 0.3-0.5
    NONE,    // < 0.2
}
```

### Detection Triggers

```rust
fn analyze_tree(tree: &ElementTree) -> TreeAnalysis {
    let named_ratio = tree.named_count / tree.total_count;
    let coverage = tree.covered_area / tree.window_area;
    
    if coverage < 0.3 {
        return TreeAnalysis {
            confidence: Confidence::NONE,
            recommended_fallback: "position_only"
        };
    }
    
    if named_ratio < 0.5 {
        return TreeAnalysis {
            confidence: Confidence::LOW,
            recommended_fallback: "heuristics"
        };
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
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x12345",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "state": ["enabled", "visible"],
          "confidence": 0.95,
          "source": "uia"
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
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x500001",
      "title": "CustomGame",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
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
    "message": "Semantic tree empty",
    "reasoning": "Custom renderer detected",
    "alternatives": [
      {"bounds": {"x": 1700, "y": 5}, "confidence": 0.3},
      {"bounds": {"x": 1750, "y": 5}, "confidence": 0.2}
    ],
    "recommended_action": "explore_and_confirm"
  }
}
```

---

## Error Codes

| Code | Description |
|------|-------------|
| `EMPTY_TREE` | Semantic tree empty or useless |
| `ELEMENT_NOT_FOUND` | Target element missing |
| `CUSTOM_RENDERER` | OpenGL/Vulkan/Metal detected |
| `PERMISSION_DENIED` | UIA access blocked |
| `DRM_BLOCKED` | Protected content |

---

## Protocol Implementation

### Transport

Default: `tcp://localhost:9876`

### Capabilities

```json
{
  "platform": "windows",
  "driver": "oscp-windows-v0.2",
  "semantic_apis": ["uia", "win32"],
  "fallback_methods": ["cdp", "win32", "position_only", "human_handoff"],
  "capabilities": ["render_tree", "actions", "events", "error_handling"],
  "features": {
    "element_tree": true,
    "cdp_bridge": true,
    "heuristics": true,
    "multi_window": true,
    "multi_monitor": true
  }
}
```

---

## Installation

```powershell
winget install OSCP.Windows
```

---

## Status

🚧 **V1 Target:** UIA + Win32 + fallback hierarchy

---

## References

- [UI Automation](https://learn.microsoft.com/en-us/windows/win32/winauto/ui-automation)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
- [Win32 Window Enumeration](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumwindows)