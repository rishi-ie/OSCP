# OSCP Windows Platform Driver Specification

**Version:** 0.3.0
**Status:** Architecture Finalized

---

## Overview

The Windows platform driver wraps UIAutomation with a streaming layer.

**Plan:** macOS first (simpler), then Linux, then Windows.

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
│  │   UIAutomation         │  │    CDP Bridge             │ │
│  │   ─────────────────    │  │    (Chrome/Edge/Electron) │ │
│  │   Coverage: 85%        │  │                          │ │
│  │                        │  │    Win32 EnumWindows      │ │
│  │   Win32                │  │    (window list only)     │ │
│  │   ─────────────        │  │                          │ │
│  │   Coverage: +5%        │  │    Position-Only Mode      │ │
│  │                        │  │                          │ │
│  │                        │  │    Human Handoff          │ │
│  └────────────────────────┘  └──────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 INPUT ENGINE                         │  │
│  │  SendInput — click, type, key combos                 │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Wrapped Technology

### UIAutomation (Primary)

```
EXISTING: UIAutomationCore (Windows COM API)
WRAPPERS: pywinauto, native C#
COVERAGE: 85%

Provides:
├── Element name and class
├── Element type (button, edit, etc.)
├── Position and size
├── Control patterns (Invoke, Value, etc.)
├── Window and control enumeration
└──HUIAutomation or IUIAutomation
```

### Win32 (Fallback)

```
EXISTING: Win32 API
APIS: EnumWindows, GetWindowRect, GetWindowText
COVERAGE: +5% (non-UIA apps)

Provides:
├── Window list
├── Window positions and sizes
└── Window titles
```

### SendInput (Input)

```
EXISTING: SendInput (user32.dll)
USAGE: Input injection

Provides:
├── Mouse click/move/drag
├── Keyboard input
└── Key combinations
```

---

## Fallback Hierarchy

```
LEVEL 1: UIAutomation (Primary)
├── Win32 apps with UIA support
├── WPF apps
├── UWP apps
└── Coverage: 85%

LEVEL 2: CDP Bridge
├── Chrome
├── Edge
├── Electron apps (VS Code, Slack)
└── Coverage: +5%

LEVEL 3: Win32 EnumWindows
├── Legacy Win32 apps (no UIA)
└── Coverage: +5%

LEVEL 4: Position-Only Mode
├── OpenGL apps
├── Vulkan apps
└── Works when tree is empty

LEVEL 5: Human Handoff
├── Protected content
└── Unrecoverable cases
```

---

## System Requirements

```
REQUIRED:
├── Windows 10 or Windows 11
├── UIAutomation available
└── Application running

OPTIONAL:
├── Chrome/Electron (for CDP bridge)
└── Admin rights (for certain input operations)
```

---

## Installation

```powershell
winget install oscp
# or
choco install oscp
```

---

## Time Estimate

| Component | Complexity | Time |
|-----------|-----------|------|
| UIA wrapping | Medium | 2-3 weeks |
| CDP bridge | Medium | 1-2 weeks |
| Streaming engine | Medium | 2 weeks |
| Error handling | Low | 1 week |
| SendInput | Low | 1 week |
| Testing | Medium | 1-2 weeks |
| **Total Windows Not Started** | | **6-8 weeks** |

---

## Update Note

This spec is documented for completeness. Windows implementation follows macOS and Linux.

---

## Status

✅ Architecture documented
⏸️ Implementation pending (macOS, Linux first)