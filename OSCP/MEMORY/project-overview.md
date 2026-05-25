# OSCP - Memory

## Project Overview

OSCP (Operating System Context Protocol) is a foundational protocol for agent-native OS interaction. It intercepts render operations at the graphics API level and delivers raw geometry to agents, enabling them to interact with desktop interfaces just like humans.

**Principle:** "Intercept at the source. Deliver the geometry. Agent provides the meaning."

## Core Concept

- **What Agent Receives:** Raw render geometry (positions, z-order, texture IDs), window metadata, mouse position
- **What Agent Provides:** Element semantics, interaction logic, state inference, visual reasoning

## Why OSCP?

| Aspect | VLM Approach | OSCP |
|--------|-------------|------|
| Speed | 2-5 seconds/frame | <50ms/frame |
| Reliability | ~70% (guessing) | ~95% (exact) |
| Precision | Fuzzy coordinates | Pixel-perfect |
| Cost | $0.01-0.05/frame | $0.0002/frame |

## Platforms

- Windows: DXGI hook (C++ DLL) + SendInput
- macOS: CGWindowList + CGEvent
- Linux: X11/Wayland + XTest

## Project Structure

```
OSCP/
├── README.md
├── SPEC.md
├── protocol/SPEC.md      # Core protocol spec
├── platforms/
│   ├── windows/SPEC.md
│   ├── macos/SPEC.md
│   └── linux/SPEC.md
└── agents/SPEC.md
```

## Status

Phase 0 complete - specs finalized. Implementation pending.