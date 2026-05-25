# OSCP - Decisions

## Key Architectural Decisions

### 1. No screenshots, no semantics
**Decision:** OSCP delivers only raw render geometry — positions, z-order, texture IDs. No screenshots. No element types. No accessibility data.

**Rationale:**
- Screenshots reintroduce VLM dependency
- Agent provides the meaning, not the protocol
- Simpler architecture, deterministic output

### 2. Intercept at render source
**Decision:** Intercept render operations at the graphics API layer (DXGI, CGWindowList, X11) rather than observing pixels.

**Rationale:**
- Draw calls already ARE the element list
- Pixel-perfect precision vs fuzzy VLM guessing
- <50ms latency vs 2-5 seconds for vision

### 3. Tiered platform approach
**Decision:** Each OS uses its native graphics API for capture.

| Platform | Capture Method |
|----------|---------------|
| Windows | DXGI hook |
| macOS | CGWindowList |
| Linux | X11/Wayland |

**Rationale:** Native APIs provide the raw render data. Each platform is different, no single abstraction works.

### 4. Agent as client, driver as server
**Decision:** Platform driver is TCP/Unix socket server. Agent harness connects as client.

**Rationale:**
- Clean separation of concerns
- Driver can be a background service
- Agent can be swapped without touching driver

### 5. Length-prefixed JSON framing
**Decision:** Every message is `[4-byte big-endian length][JSON payload]`.

**Rationale:**
- Simple, no external dependencies
- Works over any transport (TCP, Unix socket, stdio)
- Self-delimiting frames

### 6. Event-driven frame streaming
**Decision:** Driver continuously pushes frames to agent at target frame rate.

**Rationale:**
- Agent doesn't need to poll
- Real-time awareness of UI state
- Window opened/closed/focused events

### 7. Raw geometry only
**Decision:** Agent receives `render_ops` array with bounds, z, texture_id only.

**Rationale:**
- Agent figures out semantics from geometry
- No protocol-level assumptions about UI patterns
- Agent can be as smart or simple as needed

## Rejected Approaches

### Screenshots as fallback
**Rejected:** We initially considered screenshots as a fallback for apps that don't render via DirectX.

**Reason:** Reintroduces VLM dependency. OSCP is pure render interception.

### Accessibility API integration
**Rejected:** Using UIA/AXUIElement/AT-SPI for semantic element data.

**Reason:** Agent provides the meaning. Semantic types are not needed from protocol.

### Screenshot quality levels
**Rejected:** Multiple screenshot quality options (low, medium, high).

**Reason:** No screenshots in final design.

### Request/response model
**Rejected:** Agent polls for frames on demand.

**Reason:** Event-driven is better for real-time UI automation.