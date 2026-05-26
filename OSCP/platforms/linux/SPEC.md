# OSCP Linux Platform Driver Specification

**Version:** 0.4.0
**Status:** Design Updated

---

## Overview

The Linux platform driver wraps AT-SPI2 and responds to on-demand requests.

**Key Changes from v0.3:**
- No 30fps streaming
- Request-response model
- Agent controls when to request

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Linux Platform Driver                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              REQUEST HANDLER                         │  │
│  │                                                      │  │
│  │   onDemand:                                         │  │
│  │   ├── Query at-spi-bus                              │  │
│  │   ├── Handle errors                                 │  │
│  │   └── Return frame JSON                             │  │
│  └──────────────────────────────────────────────────────┘  │
│                             │                             │
  │                    TREE ANALYZER                      │
│  │   coverage_score, named_elements, confidence           │ │
│  └──────────────────────────┴───────────────────────────┘ │
│                             │                             │
│                 ┌───────────┴───────────┐                  │
│                 ▼                       ▼                  │
│  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│  │   PRIMARY: AT-SPI2        │  │   FALLBACKS:             │ │
│  │   + X11 (fallback)       │  │   • CDP Bridge           │ │
│  │   Coverage: 90%          │  │   • Heuristics            │ │
│  │                          │  │   • Human Handoff        │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                INPUT ENGINE                          │  │
│  │   click(), type() via /dev/uinput                    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Wrapped Technology

### AT-SPI2 (Primary)

```
EXISTING: AT-SPI2 (D-Bus accessibility)
WRAPPERS: dogtail, pyatspi, ldtp
COVERAGE: 85%

Provides:
├── Element role (button, menu, etc.)
├── Accessible name
├── Bounds (position, size)
├── States (enabled, visible, etc.)
└── Application enumeration
```

### X11 (Fallback)

```
EXISTING: X11 Protocol
APIS: XQueryTree, XGetWindowProperty, XGetWindowAttributes
COVERAGE: +5% (X11 desktops, Xwayland apps)
```

### Input Engine

```
PRIMARY: /dev/uinput (Linux kernel interface)
FALLBACK: XTest (X11 desktops)
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
  "platform": "linux",
  "latency_ms": 80,
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
          "source": "atspi"
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

### Fallback Chain

```
1. Capture via AT-SPI2
2. Analyze coverage_score
3. If coverage < 0.3:
   └── Try X11 (if X11/Xwayland app)
4. If still low:
   └── Try CDP bridge (Chrome, Firefox, Electron)
5. If all fails:
   └── Position-only mode
6. If truly blocked:
   └── Human handoff
```

---

## Time Estimate

| Component | Complexity | Time |
|-----------|------------|------|
| AT-SPI2 wrapping | Medium | 2 weeks |
| X11 fallback | Low | 1 week |
| Request handler | Medium | 1 week |
| CDP bridge | Medium | 1-2 weeks |
| Error handling | Low | 1 week |
| /dev/uinput input | Medium | 2 weeks |
| Testing | High | 2 weeks |
| **Total Linux** | | **6-7 weeks** |

---

## System Requirements

```
REQUIRED:
├── at-spi2-core package
├── at-spi-bus running
└── Accessibility enabled in desktop environment
```

---

## Status

- [x] Architecture updated
- [x] Request-response model defined
- [ ] Implementation pending

---

## References

- [AT-SPI2 Specification](https://www.freedesktop.org/wiki/Accessibility/)
- [dogtail](https://github.com/nick96/dogtail) - Python AT-SPI2 library
- [pyatspi](https://github.com/python-atspi/pyatspi) - PyAT-SPI2 bindings