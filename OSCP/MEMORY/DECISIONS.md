# OSCP - Key Decisions

## Architecture Decisions

### 1. Integration, Not New Development
- OSCP wraps existing OS APIs (AXUIElement, AT-SPI2, UIA)
- These APIs are proven and well-documented
- OSCP's contribution: unify, on-demand, error handling

### 2. On-Demand, Not Streaming
- Changed from 30fps streaming to request-response
- Agent controls when to see screen
- Simpler architecture, lower resource usage

### 3. No VLM in V1
- Keep scope tight
- VLM is V2+ extension for edge cases
- V1: fast, deterministic, deterministic

### 4. Error-Resilient Design
- 5-level fallback hierarchy
- Agent must NEVER get stuck
- Always have next step

### 5. Agent Provides Meaning
- OSCP delivers coordinates and types
- Agent supplies semantics
- Position patterns reveal UI structure

## Platform Decisions

### macOS First
- AXUIElement most stable
- No compositor fragmentation
- 4-5 weeks

### Linux Second
- AT-SPI2 + X11/Xwayland
- Wayland native deferred to V2
- 6-7 weeks

### Windows Deferred
- Added after macOS + Linux work

## Protocol Decisions

### JSON over Socket
- Simple parsing, human-readable
- Unix socket for local fast communication
- Request-response (no streaming)

### Confidence Scoring
- Every element has confidence score
- Every frame has tree analysis
- Enables smart agent decisions

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| Total | 10-12 weeks |

## Rejected Approaches

### Screenshot + VLM (Rejected for V1)
- Probabilistic, not deterministic
- Slow (~1-2s/frame vs ~50-100ms)
- Expensive ($0.01-0.10/frame vs ~free)

### Continuous Streaming (Rejected in v0.4)
- Higher complexity
- Agent doesn't need always-on awareness
- On-demand sufficient