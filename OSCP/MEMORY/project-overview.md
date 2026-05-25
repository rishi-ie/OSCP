# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It intercepts the compositor's render pipeline at the OS level and delivers raw render operations to agents — enabling them to see and interact with the full desktop just like humans.

**Principle:** "Intercept the compositor. Decode the render tree. Agent provides the meaning."

## Core Concept

OSCP makes agents first-class citizens of operating systems designed for humans.

```
┌─────────────────────────────────────────────────────────────┐
│                 FIRST-CLASS AGENT = OSCP + SKILLS          │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  OSCP           │ +  │  Agent Skills   │  = First-Class  │
│  │  (Geometry)     │    │  (Inference)    │    Citizen      │
│  └─────────────────┘    └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### What OSCP Provides
- Exact geometry: positions, bounds, z-order
- Window structure: titles, sizes, focus state
- Render operations (macOS/Linux) or element tree (Windows)
- Mouse position: cursor and hovered element
- Real-time stream: 30fps render trees

### What Agent Skills Provide
- Element types: button vs label vs input
- Text content: inferred from patterns
- State detection: enabled, disabled, focused
- Layout reasoning: toolbar, sidebar, etc.
- Interaction logic: what's clickable

## Platforms (V1)

| Platform | Method | Coverage | Time | Risk |
|----------|--------|----------|------|------|
| **macOS** | CGWindowList (Window Server) | 95% | 2-3 mo | Low |
| **Linux** | X11 + Compositor Hook | 90-95% | 3-4 mo | Medium |
| **Windows** | UIA + Win32 | 90% | 1-2 mo | Very Low |

## Per-Platform Approach

### macOS
- Window Server is the compositor
- CGWindowListCopyWindowInfo reads layer tree
- No GPU hooks needed
- Official, stable API

### Linux
- **Tier 1:** X11 APIs (XQueryTree, XGetWindowProperty)
  - Covers X11 desktops + Xwayland apps (~85%)
- **Tier 2:** Compositor OpenGL Hook
  - Hooks: Mutter, KWin, Sway EGL/OpenGL
  - Captures native Wayland windows (~5-10% more)

### Windows
- **Tier 1:** Win32 APIs (EnumWindows, GetWindowRect)
- **Tier 2:** UI Automation (element types, names, states)
- Note: No documented API for Windows render ops. UIA + Win32 chosen for stability.

## Coverage

| Platform | Coverage | Gaps |
|----------|----------|------|
| **macOS** | 95% | Screen sharing, DRM, sandboxed |
| **Linux** | 90-95% | Some Wayland compositors, TTY, KMS |
| **Windows** | 90% | Non-UIA apps, legacy Win32, protected |

## Gaps After V1

| Gap | Fillable? |
|-----|-----------|
| Element semantics (macOS/Linux) | ✅ Agent skills |
| WebGL/Canvas content | ⚠️ Partially |
| Color semantics | ✅ Protocol extension |
| Audio | ✅ Separate API later |
| Protected content | ❌ OS restriction |

## Status

Phase 0 complete. V1 implementation starting for all three platforms.

## References

- GitHub: github.com/rishi-ie/OSCP
- Protocol: OSCP/protocol/SPEC.md
- Platforms: OSCP/platforms/{macos,linux,windows}/SPEC.md