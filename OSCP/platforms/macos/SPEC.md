# OSCP macOS Platform Driver Specification

**Version:** 0.3.0
**Status:** Architecture Finalized

---

## Overview

The macOS platform driver wraps AXUIElement with a streaming layer.

**Key Insight:** AXUIElement already extracts the semantic tree. OSCP adds streaming, unified protocol, and error handling.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Platform Driver                             │
│                                                             │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────┐ │
│  │  STREAMING      │  │  ERROR HANDLER   │  │   INPUT   │ │
│  │   ENGINE        │  │  + FALLBACKS     │  │  ENGINE   │ │
│  │  (30fps)        │  │                  │  │           │ │
│  └────────┬────────┘  └────────┬─────────┘  └─────┬─────┘ │
│           │                    │                  │         │
│           │             ┌──────▼──────┐          │         │
│           │             │   TREE      │◄─────────┘         │
│           │             │  ANALYZER   │                    │
│           │             └──────┬──────┘                    │
│           │                    │                          │
│           │    ┌───────────────┴────────────┐            │
│           │    │                            │            │
│           │    ▼                            ▼            │
│           │   ▼                              ▼           │
│  ┌────────┴──────────────┐  ┌──────────────────────────┐ │
│  │   PRIMARY CAPTURE      │  │    FALLBACK METHODS      │ │
│  │                        │  │                          │ │
│  │   AXUIElement          │  │    CDP Bridge             │ │
│  │   ────────────         │  │    (Safari/Chrome/Electron)│ │
│  │   Coverage: 90%        │  │                          │ │
│  │                        │  │    Position-Only Mode     │ │
│  │   AppKit, SwiftUI     │  │    (Metal/OpenGL games)   │ │
│  │   Standard controls  │  │                          │ │
│  │                        │  │    Human Handoff          │ │
│  └────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 INPUT ENGINE                         │  │
│  │  CGEventPost — click, type, key combos               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Wrapped Technology

### AXUIElement (Primary)

```
EXISTING: AXUIElement (Apple's accessibility API)
WRAPPERS: pyax, ax-element
COVERAGE: 90%

Provides:
├── Element role (button, menu, etc.)
├── Title, description, value
├── Position and size
├── Child hierarchy
├── States (enabled, visible, etc.)
└── Application enumeration
```

### AXObserver (Streaming)

```
EXISTING: AXObserver (Apple's observation API)
USAGE: Real-time change notifications

Provides:
├── Element creation events
├── Element destruction events
├── Property change events
└── Window focus changes
```

### CGEvent (Input)

```
EXISTING: CGEvent (Core Graphics)
USAGE: Input injection

Provides:
├── CGEventCreateMouseEvent (click, move, drag)
├── CGEventCreateKeyboardEvent (type, key combos)
└── CGEventPost (inject into system)
```

---

## Streaming Engine

```
IMPLEMENTATION: AXObserver + Poll Interval

AXObserver (preferred):
├── 30fps notification loop
├── Watch for AXUIElement changes
├── System events trigger capture
└── Efficient (event-driven)

Poll Interval (fallback):
├── 33ms interval timer
├── Query focused app hierarchy
└── Detect structural changes

Both channels feed the same tree builder.
```

---

## Fallback Hierarchy

```
LEVEL 1: AXUIElement (Primary)
├── AppKit controls
├── SwiftUI controls
├── Standard macOS apps
└── Coverage: 90%

LEVEL 2: CDP Bridge
├── Safari (WebKit)
├── Chrome
├── Electron apps
└── Coverage: +4%

LEVEL 3: Position-Only Mode
├── Metal apps
├── OpenGL apps
├── Games
└── Works when tree is empty

LEVEL 4: Human Handoff
├── Non-app windows
├── Screen sharing content
└── Unrecoverable cases
```

---

## Tree Analyzer

```swift
func analyzeTree(_ tree: AXUIElementHierarchy) -> TreeAnalysis {
    let coverage = tree.coveredBounds / tree.windowBounds
    let namedRatio = tree.namedCount / tree.totalCount
    
    let confidence: Confidence
    if coverage < 0.3 {
        confidence = .NONE
    } else if coverage < 0.5 {
        confidence = .LOW
    } else if coverage < 0.8 {
        confidence = .MEDIUM
    } else {
        confidence = .HIGH
    }
    
    return TreeAnalysis(
        coverageScore: coverage,
        namedElements: tree.namedCount,
        unlabeledElements: tree.unlabeledCount,
        confidence: confidence)
}
```

---

## Output Format

```json
{
  "type": "render_tree",
  "platform": "macOS",
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "pid": 1234,
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "states": ["enabled", "visible"],
          "confidence": 0.95,
          "source": "axuielement"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH",
    "recommended_action": "execute"
  }
}
```

---

## Implementation Stack

| Layer | Technology |
|-------|------------|
| **Driver** | Swift, Objective-C, or Rust |
| **AXUIElement Wrapper** | pyax, native C, or Rust bindings |
| **Protocol Server** | Unix socket + JSON |
| **Input** | CGEvent (Core Graphics) |

---

## System Requirements

```
REQUIRED:
├── macOS 12.0+
├── Screen Recording permission (for full access)
└── Accessibility permission (for input and tree)

PERMISSIONS:
├── Screen Recording — enables AXUIElement access
├── Accessibility — enables input injection
└── Grant via System Preferences > Privacy & Security
```

---

## Installation

```bash
brew install oscp
```

After installation, grand permissions:
1. System Preferences > Privacy & Security > Screen Recording
2. Check: OSCP
3. System Preferences > Accessibility
4. Check: OSCP

---

## Time Estimate

| Component | Complexity | Time |
|-----------|-----------|------|
| AXUIElement wrapping | Low | 1-2 weeks |
| AXObserver streaming | Low | 1 week |
| CDP bridge | Medium | 1-2 weeks |
| Error handling | Low | 1 week |
| CGEvent input | Low | 1 week |
| Testing | Medium | 1-2 weeks |
| **Total macOS** | | **4-6 weeks** |

---

## Status

✅ Architecture documented
✅ Existing tools identified
⏳ Implementation pending