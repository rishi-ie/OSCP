# OSCP Windows Platform Driver Specification

**Version:** 0.4.0
**Status:** Design Updated

---

## Overview

The Windows platform driver wraps UIAutomation and responds to on-demand requests.

**Plan:** macOS first (simpler), then Linux, then Windows.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Windows Platform Driver                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              REQUEST HANDLER                         │  │
│  │                                                      │  │
│  │   onDemand:                                         │  │
│  │   ├── Query UIAutomation                            │  │
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
│  │   PRIMARY: UIA           │  │   FALLBACKS:             │ │
│  │   Coverage: 85%          │  │   • CDP Bridge           │ │
│  │                          │  │   • Win32 EnumWindows    │ │
│  │   Win32 fallback         │  │   • Human Handoff        │ │
│  │   Coverage: +5%          │  │                          │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                INPUT ENGINE                          │  │
│  │   click(), type() via SendInput                      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Wrapped Technology

### UIAutomation (Primary)

```
EXISTING: UIAutomationCore (Windows COM API)
WRAPPERS: pywinauto, native C# / WinRT
COVERAGE: 85%

Provides:
├── Element name and class
├── Element type (button, edit, etc.)
├── Position and size
├── Control patterns (Invoke, Value, etc.)
└── Window and control enumeration
```

### Win32 (Fallback)

```
EXISTING: Win32 API
APIS: EnumWindows, GetWindowRect, GetWindowText
COVERAGE: +5%
```

### SendInput (Input)

```
EXISTING: SendInput (user32.dll)
USAGE: Input injection
```

---

## Request-Response Model

Same as macOS/Linux:
```python
frame = await oscp.getFrame()
```

---

## Time Estimate

| Component | Complexity | Time |
|-----------|------------|------|
| UIA wrapping | Medium | 2-3 weeks |
| CDP bridge | Medium | 1-2 weeks |
| Request handler | Medium | 1 week |
| Error handling | Low | 1 week |
| SendInput | Low | 1 week |
| Testing | Medium | 1-2 weeks |
| **Total Windows** | | **6-7 weeks** |

---

## Status

- [x] Architecture updated
- [x] Request-response model defined
- [ ] Implementation pending (macOS, Linux first)

---

## Update Note

Windows implementation follows macOS and Linux. Start order: macOS → Linux → Windows.