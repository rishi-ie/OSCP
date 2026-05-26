# OSCP macOS Platform Driver Specification

**Version:** 0.4.0
**Status:** Design Updated

---

## Overview

The macOS platform driver wraps AXUIElement and responds to on-demand requests.

**Key Changes from v0.3:**
- No 30fps streaming
- Request-response model
- Agent controls when to request

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 macOS Platform Driver                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              REQUEST HANDLER                         │  │
│  │                                                      │  │
│  │   onDemand:                                         │  │
│  │   ├── C API call to AXUIElement                     │  │
│  │   ├── Handle errors                                 │  │
│  │   └── Return frame JSON                             │  │
│  └──────────────────────────────────────────────────────┘  │
│                             │                             │
│  ┌──────────────────────────┼───────────────────────────┐ │
│  │                    TREE ANALYZER                      │ │
│  │   coverage_score, named_elements, confidence           │ │
│  └──────────────────────────┴───────────────────────────┘ │
│                             │                             │
│                 ┌───────────┴───────────┐                  │
│                 ▼                       ▼                  │
│  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│  │   PRIMARY: AXUIElement  │  │   FALLBACKS:             │ │
│  │   Coverage: 90%          │  │   • CDP Bridge           │ │
│  │   AppKit, SwiftUI        │  │   • Position-Only        │ │
│  │                          │  │   • Human Handoff        │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                INPUT ENGINE                          │ │
│  │   click(), type(), key_combo() via CGEvent            │ │
│  └──────────────────────────────────────────────────────┘ │
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

## Request-Response Model

### Agent Request

```python
# When agent needs to see screen
frame = await oscp.getFrame()
```

### OSCP Response

```json
{
  "type": "frame",
  "request_id": "req_001",
  "frame_id": 12345,
  "platform": "macOS",
  "latency_ms": 45,
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "confidence": 0.95,
          "source": "axuielement"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "confidence": "HIGH"
  }
}
```

---

## Error Handling

### Tree Quality Check

```
1. Capture via AXUIElement
2. Analyze coverage_score
3. If coverage < 0.3:
   └── Try CDP bridge (Safari, Chrome, Electron)
4. If still low:
   └── Position-only mode
5. If all fails:
   └── Human handoff
```

---

## Time Estimate

| Component | Complexity | Time |
|-----------|------------|------|
| AXUIElement wrapping | Low | 1-2 weeks |
| Request handler | Low | 1 week |
| CDP bridge | Medium | 1-2 weeks |
| Error handling | Low | 1 week |
| CGEvent input | Low | 1 week |
| Testing | Medium | 1-2 weeks |
| **Total macOS** | | **4-5 weeks** |

---

## Permissions

- **Screen Recording** — Enables AXUIElement access
- **Accessibility** — Enables CGEvent input injection

---

## Status

- [x] Architecture updated
- [x] Request-response model defined
- [ ] Implementation pending

---

## References

- [AXUIElement Documentation](https://developer.apple.com/documentation/application-services)
- [pyax](https://github.com/pyax/pyax) - Python AXUIElement wrapper
- [ax-element](https://github.com/nalexand/ax-element) - Rust AXUIElement bindings