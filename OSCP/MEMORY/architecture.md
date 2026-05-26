# OSCP Architecture

## Pattern (All Platforms)

```
AGENT                                    OSCP SERVICE
 │                                           │
 │  oscp.getFrame() ─────────────────────────►│
 │  (on-demand)                               ├─► Query OS APIs
 │                                           ├─► Fallback chain
 │                                           ├─► Tree analysis
 │  ◄────────────────────────────────────────│
 │  { windows, elements, confidence }          │
 │                                           │
 │  oscp.click(bounds) ─────────────────────►│
 │                                           ├─► Input injection
 │  ◄────────────────────────────────────────│
 │  { success: true }                        │
```

## On-Demand vs Streaming

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
├── CDP bridge (fallback)
└── JSON response

PRIMARY: AXUIElement
├── AppKit, SwiftUI
├── Coverage: 90%
└── IMPLEMENTED: Swift/Obj-C + C bridging

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
├── AT-SPI2 query
├── X11 fallback
├── CDP bridge (fallback)
└── JSON response

PRIMARY: AT-SPI2
├── GTK, Qt, Swing
├── Coverage: 85%
└── IMPLEMENTED: Python (pyatspi) or Rust

X11 FALLBACK:
├── XQueryTree
├── X11 desktops + Xwayland apps
└── Coverage: +5%

INPUT: /dev/uinput (primary) / XTest (fallback)
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
4. FALLBACK CHAIN (if needed)
   │
   ▼
5. TREE BUILDER + ANALYZER
   │
   ▼
6. PROTOCOL RESPONSE (JSON)
   │
   ▼
7. AGENT CLIENT
```

## Spec Status

| Spec | Status |
|------|--------|
| Protocol | ✅ Complete |
| macOS | ✅ Complete |
| Linux | ✅ Complete |
| Windows | ⏸️ Deferred |

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| Total | 10-12 weeks |