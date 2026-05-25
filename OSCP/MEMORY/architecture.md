# OSCP Architecture

## Pattern (All Platforms)

```
AGENT HARNESS
     │
OSCP Protocol
     │
┌────────────────────────────────────────────────────────────┐
│              OSCP PLATFORM DRIVER                          │
│                                                            │
│   ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│   │  STREAMING  │  │    ERROR     │  │      INPUT      │ │
│   │   ENGINE    │  │   HANDLER    │  │     ENGINE      │ │
│   └──────┬──────┘  └──────┬───────┘  └────────┬────────┘ │
│          │                 │                    │           │
│          │          ┌──────▼───────┐            │           │
│          │          │    TREE      │◄──────────┘           │
│          │          │   ANALYZER   │                         │
│          │          └──────┬───────┘                         │
│          │                 │                                 │
│          │    ┌────────────┴────────────┐                   │
│          │    │                         │                   │
│          │    ▼                         ▼                   │
│          │   ▼                           ▼                   │
│   ┌──────┴──────────────────┐  ┌──────────────────────────┐│
│   │   PRIMARY CAPTURE         │  │    FALLBACK METHODS       ││
│   │   (AXUIElement/AT-SPI2)  │  │    (CDP/X11/Position)     ││
│   └───────────────────────────┘  └──────────────────────────┘│
└────────────────────────────────────────────────────────────┘
     │
Native OS APIs
```

## macOS Architecture

```
STREAMING ENGINE
├── AXObserver (30fps)
├── Watches AXUIElement changes
└── Poll interval: 33ms

PRIMARY: AXUIElement
├── AppKit, SwiftUI
├── Standard controls
├── Coverage: 90%
└── EXISTING (wrappers: pyax, ax-element)

FALLBACKS:
├── CDP Bridge (Safari/Chrome/Electron)
├── Position-Only Mode (Metal/OpenGL)
└── Human Handoff

INPUT: CGEvent
```

## Linux Architecture

```
STREAMING ENGINE
├── AT-SPI2 Observer (D-Bus events)
├── + Poll interval (fallback)
└── ~30fps

PRIMARY: AT-SPI2
├── GTK, Qt, Swing
├── Standard controls
├── Coverage: 85%
└── EXISTING (wrappers: dogtail, pyatspi, ldtp)

FALLBACKS:
├── X11 (XQueryTree) — X11 + Xwayland
├── CDP Bridge (Chrome/Firefox/Electron)
├── Position-Only Mode
└── Human Handoff

INPUT: /dev/uinput
```

## OSCP Layer (Shared)

- Protocol Server (Unix socket)
- Tree Builder (standardized format)
- Error Handler (5-level fallback hierarchy)
- Tree Analyzer (coverage_score, confidence)

## Data Flow

```
1. NATIVE CAPTURE (AXUIElement/AT-SPI2)
   │
   ▼
2. TREE BUILDER (OSCP layer)
   │
   ▼
3. TREE ANALYZER
   │
   ▼
4. IF QUALITY < THRESHOLD: FALLBACK CHAIN
   │
   ▼
5. PROTOCOL SERVER (JSON)
   │
   ▼
6. AGENT CLIENTS
```