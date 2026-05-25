# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It intercepts the compositor's render pipeline at the OS level and delivers raw render operations to agents — enabling them to see and interact with the full desktop just like humans.

**Principle:** "Intercept the compositor. Decode the render tree. Agent provides the meaning."

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 FIRST-CLASS AGENT                           │
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
- Texture IDs: what's being rendered
- Mouse position: cursor and hovered element
- Real-time stream: 30fps render trees

### What Agent Skills Provide (filled by agent)
- Element types: button vs label vs input
- Text content: inferred from patterns
- Element states: enabled, disabled, focused
- Layout understanding: toolbar, sidebar, etc.
- Interaction logic: what's clickable

### What Intelligence Provides
- Reasoning about the geometry
- Planning multi-step tasks
- Decision making

## Platforms (V1)

| Platform | Compositor | Intercept Method |
|----------|-----------|------------------|
| **macOS** | Window Server | Layer tree extraction |
| **Linux** | X11/Wayland | Scene graph |
| **Windows** | Deferred | V2 (UIA + Win32) |

## Coverage

- macOS: ~95%
- Linux: ~90%
- Windows: Deferred to V2

## Key Decisions

1. **OS-level compositor intercept** — Not per-app hooks. One point, all apps.
2. **No screenshots** — Raw geometry only. No pixels. No VLM dependency.
3. **Agent provides meaning** — Agent skills fill the gap between geometry and understanding.
4. **macOS + Linux only for V1** — Windows lacks documented render op APIs.

## Status

Phase 0 complete. Implementation starting for macOS and Linux.