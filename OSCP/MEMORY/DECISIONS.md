# OSCP - Decisions

## Key Architectural Decisions

### 1. OS-Level Compositor Interception (Not Per-App Hook)

**Decision:** Intercept the compositor's render tree at the OS level, not per-app DXGI hooks.

**Rationale:**
- Per-app hook only captures DirectX apps (~85%)
- Compositor intercept captures ALL apps (OpenGL, Vulkan, GDI, etc.)
- One interception point vs inject into every process
- Games with anti-cheat still captured at compositor output

### 2. No Screenshots, No Pixels, No VLM

**Decision:** OSCP delivers only raw render geometry — positions, z-order, texture IDs from compositor.

**Rationale:**
- Screenshots reintroduce VLM dependency
- Pixels are slow, expensive, unreliable
- Agent provides the meaning, not the protocol
- Compositor render tree has all the structural data needed

### 3. Intercept the Compositor (Not Per-App)

**Decision:** One interception point at the compositor covers all apps.

**Before (per-app hook):**
```
App A (DXGI) ──► Hook ──► missed
App B (OpenGL) ──► ❌ Hook ──► missed
App C (Vulkan) ──► ❌ Hook ──► missed
```

**After (compositor intercept):**
```
App A ─┐
App B ─┼─► Compositor ─► INTERCEPT ─► Agent (covers all)
App C ─┘
```

### 4. Agent as Client, Driver as Server

**Decision:** Platform driver is TCP/Unix socket server. Agent harness connects as client.

**Rationale:**
- Clean separation of concerns
- Driver runs as background service
- Agent can be swapped without touching driver
- Multiple agents can connect to same driver

### 5. Length-Prefixed JSON Framing

**Decision:** Every message is `[4-byte big-endian length][JSON payload]`.

**Rationale:**
- Simple, no external dependencies
- Works over any transport (TCP, Unix socket, stdio)
- Self-delimiting frames
- Language-agnostic

### 6. Event-Driven Render Tree Streaming

**Decision:** Driver continuously pushes render trees to agent at target frame rate.

**Rationale:**
- Agent doesn't need to poll
- Real-time awareness of desktop state
- Window opened/closed/focused events
- Agent sees every change as it happens

### 7. Raw Geometry Only from Compositor

**Decision:** Agent receives `windows` with `ops` (bounds, z, texture_id) from compositor scene tree.

**Rationale:**
- Agent figures out semantics from geometry
- No protocol-level assumptions about UI patterns
- No element types, states, or accessibility data
- Agent can be as smart or simple as needed

### 8. Render Tree Structure (Not Screenshot)

**Decision:** Frame is a hierarchical render tree, not a flat pixel buffer.

**Rationale:**
- Render tree preserves spatial structure
- Z-order, parent-child relationships preserved
- Agent can reason about layout from tree
- Tree is smaller than pixel buffer

## Rejected Approaches

### Per-App DXGI Hook
**Rejected:** Only captures DirectX apps. Misses OpenGL, Vulkan, games.

### Screen Recording / Screenshots
**Rejected:** Reintroduces VLM dependency, slow, expensive, unreliable.

### Accessibility APIs (UIA, AXUIElement, AT-SPI)
**Rejected:** Agent provides the meaning. Semantic types not needed from protocol.

### Request/Response Polling
**Rejected:** Event-driven streaming is better for real-time UI automation.

### Multiple Interception Points
**Rejected:** One point at compositor is simpler and more reliable than multiple per-app hooks.

## Platform-Specific Decisions

### Windows: DWM Compositor
- Use Graphics Capture API / DWM internals
- Intercept scene tree at compositor level
- Covers DirectX, OpenGL, Vulkan, GDI, all apps

### macOS: Window Server
- Extract layer tree from Window Server
- Intercept at compositor level
- Covers AppKit, Metal, Quartz, all apps

### Linux: X11/Wayland
- Intercept scene graph in X11 server / Wayland compositor
- Covers X11, Wayland, all apps
- Fallback: X11 if Wayland not available