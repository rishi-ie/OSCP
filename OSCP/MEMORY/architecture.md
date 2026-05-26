# OSCP Architecture

## Pattern (All Platforms)

```
AGENT                                    OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (on-demand)                               ├─► Query OS APIs
 │                                           ├─► Analyze tree
 │  ◄────────────────────────────────────────│
 │  { windows, elements, confidence }          │
 │                                           │
 │  oscp.click(bounds) ─────────────────────►│
 │                                           ├─► Input injection
 │  ◄────────────────────────────────────────│
 │  { success: true }                        │
```

## No Streaming = On-Demand

| Aspect | Old (Streaming) | New (On-Demand) |
|--------|----------------|-----------------|
| **Pattern** | 30fps continuous | Request-response |
| **Agent control** | Passive | Active |
| **Resource usage** | Higher | Lower |
| **Complexity** | Higher | Lower |
| **Time to build** | 12-16 weeks | 10-12 weeks |

## macOS Architecture

```
REQUEST HANDLER (on-demand)
├── oscp.getFrame() call
├── AXUIElement query
└── JSON response

PRIMARY: AXUIElement
├── AppKit, SwiftUI
├── Coverage: 90%
└── EXISTING (pyax, ax-element)

FALLBACKS:
├── CDP Bridge (Safari/Chrome/Electron)
├── Position-Only Mode
└── Human Handoff

INPUT: CGEvent
```

## Linux Architecture

```
REQUEST HANDLER (on-demand)
├── oscp.getFrame() call
├── AT-SPI2 query + X11 fallback
└── JSON response

PRIMARY: AT-SPI2
├── GTK, Qt, Swing
├── Coverage: 85%
└── EXISTING (dogtail, pyatspi)

X11 FALLBACK:
├── XQueryTree
├── X11 desktops + Xwayland apps
└── Coverage: +5%

INPUT: /dev/uinput
```

## OSCP Layer (Shared)

- Request Handler (Unix socket)
- Tree Builder (standardized format)
- Error Handler (5-level fallback hierarchy)
- Tree Analyzer (coverage_score, confidence)

## Data Flow

```
1. AGENT REQUEST: oscp.getFrame()
   │
   ▼
2. REQUEST HANDLER
   │
   ▼
3. NATIVE CAPTURE (AXUIElement/AT-SPI2)
   │
   ▼
4. TREE BUILDER (OSCP layer)
   │
   ▼
5. TREE ANALYZER
   │
   ▼
6. IF QUALITY < THRESHOLD: FALLBACK CHAIN
   │
   ▼
7. PROTOCOL RESPONSE (JSON)
   │
   ▼
8. AGENT CLIENT
```