# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It intercepts the compositor's render pipeline at the OS level — one point covering all apps — and delivers raw render operations to agents.

**Principle:** "Intercept the compositor. Decode the render tree. Agent provides the meaning."

**Key difference from V1:** OS-level compositor interception, not per-app hooks.

## Core Concept

- **What Agent Receives:** Raw render geometry from compositor (positions, z-order, texture IDs), window metadata, mouse position
- **What Agent Provides:** Element semantics, interaction logic, state inference, visual reasoning
- **No:** Screenshots, pixels, VLM dependency

## Why OSCP vs VLM?

| Aspect | VLM | OSCP |
|--------|-----|------|
| Speed | 2-5 sec | <50ms |
| Reliability | ~70% | ~95% |
| Precision | Fuzzy | Pixel-perfect |
| Cost | High (tokens) | Low |
| Coverage | All (pixels) | ~95% (ops) |

## Why Compositor vs Per-App?

| Aspect | Per-App Hook | Compositor Interception |
|--------|--------------|-------------------------|
| Coverage | 85% | 100% |
| OpenGL | ❌ | ✅ |
| Vulkan | ❌ | ✅ |
| Games | ❌ | ✅ |
| Complexity | High | Lower |

## Platforms

- Windows: DWM compositor (Graphics Capture API / DWM internals)
- macOS: Window Server (Layer tree extraction)
- Linux: X11/Wayland compositor (Scene graph)

## Project Structure

```
OSCP/
├── README.md
├── SPEC.md
├── protocol/SPEC.md      # Core protocol
├── platforms/
│   ├── windows/SPEC.md  # DWM compositor intercept
│   ├── macos/SPEC.md    # Window Server intercept
│   └── linux/SPEC.md   # X11/Wayland intercept
├── agents/SPEC.md
└── MEMORY/
    ├── project-overview.md
    └── DECISIONS.md
```

## Status

Phase 0 complete - compositor interception spec finalized. Implementation pending.