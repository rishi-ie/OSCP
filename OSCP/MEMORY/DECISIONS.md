# OSCP - Decisions

## Key Architectural Decisions

### 1. OS-Level Compositor Interception

**Decision:** Intercept at the compositor level, not per-app.

**Rationale:**
- Per-app hook only captures specific graphics APIs (DirectX, OpenGL, etc.)
- Compositor intercept captures ALL apps in one hook
- One interception point covers everything

### 2. No Screenshots, No Pixels

**Decision:** OSCP delivers geometry only. No screenshots. No VLM dependency.

**Rationale:**
- Screenshots reintroduce VLM dependency
- Pixels are slow, expensive, unreliable
- Agent provides the meaning, not the protocol
- Geometry is sufficient for reliable interaction

### 3. Three-Layer Strategy (No One Size Fits All)

**Decision:** Use best available method per platform, not a single universal approach.

**Rationale:**
- Linux: X11 is introspectable, Wayland needs compositor hooks
- macOS: Window Server has all data via CGWindowListCopyWindowInfo
- Windows: No render-op API exists, UIA + Win32 is the stable choice

### 4. Per-Platform Decisions

#### macOS: CGWindowListCopyWindowInfo

**Why:**
- Window Server is the compositor
- Layer tree accessible via official APIs
- No GPU hooks needed
- 95% coverage

#### Linux: X11 + Compositor Hook Hybrid

**Why:**
- X11 APIs give 85% coverage (X11 desktops + Xwayland)
- Compositor OpenGL hook adds Wayland-native coverage
- Unified output hides implementation details
- Total: 90-95%

#### Windows: UIA + Win32 (Not Render Ops)

**Why Not Render Ops:**
- No documented API exposes render operations from DWM
- DWM hook: undocumented, risky, 5-9 months
- Kernel driver: extreme complexity, 10-14 months, requires signing
- UIA + Win32: works today, 90% coverage, 1-2 months

**Decision:** Pragmatic. Ship with UIA + Win32. V2 explores DWM hook.

### 5. Agent Provides the Meaning

**Decision:** Protocol delivers geometry. Agent skills fill semantics gap.

**Rationale:**
- Agent can infer element types from geometry
- Position, size, z-order patterns reveal UI structure
- Agent skills provide pattern matching and learning
- Platform independence (same agent works on all OS)

### 6. Agent as Client, Driver as Server

**Decision:** Platform driver is TCP/Unix socket server. Agent harness connects.

**Rationale:**
- Clean separation of concerns
- Driver runs as background service
- Agent can be swapped without touching driver
- Multiple agents can connect to same driver

### 7. Length-Prefixed JSON Framing

**Decision:** Every message is `[4-byte big-endian length][JSON payload]`.

**Rationale:**
- Simple, no external dependencies
- Works over any transport (TCP, Unix socket, stdio)
- Self-delimiting frames
- Language-agnostic

### 8. Event-Driven Render Tree Streaming

**Decision:** Driver continuously pushes render trees to agent at target frame rate.

**Rationale:**
- Agent doesn't need to poll
- Real-time awareness of desktop state
- Window opened/closed/focused events
- Agent sees every change as it happens

## Rejected Approaches

### Per-App DXGI Hook (Windows)

**Rejected:** Only captures DirectX apps (~85%). Complex, fragile.

### DWM Hook (Windows V1)

**Rejected:** Too risky. Undocumented, breaks on updates, requires 5-9 months.

### Kernel Driver (Windows)

**Rejected:** Extreme complexity. 10-14 months, requires Microsoft signing, BSOD risk.

### Screen Recording / Screenshots

**Rejected:** Reintroduces VLM dependency, slow, expensive.

### Accessibility APIs Only (Linux/macOS)

**Rejected:** Agent provides the meaning. Accessibility tree not needed.

## Future Decisions (V2)

### Windows Render Operations

Options:
1. DWM DirectX hook (experimental, risky)
2. Kernel driver (proper but complex)

Status: Deferred from V1.

### Protocol Extensions

Possible:
- Color sampling (for error state detection)
- Audio capture API
- Hardware access API

Status: Future consideration.

## Platform-Specific Notes

### macOS
- "Screen Recording" permission name is misleading
- Actually enables Window Server layer tree access
- No GPU hooks needed

### Linux
- X11 is unified (one API works everywhere)
- Wayland is fragmented (per-compositor hooks needed)
- Xwayland bridges X11 apps to Wayland desktops

### Windows
- DWM (Desktop Window Manager) is the compositor
- No public API for render operation extraction
- UIA + Win32 is the stable, official path