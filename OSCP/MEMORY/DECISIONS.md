# OSCP - Decisions

## Key Architectural Decisions

### 1. Integration, Not New Development

**Decision:** OSCP wraps existing OS accessibility APIs rather than building from scratch.

**Rationale:**
- AXUIElement (macOS), AT-SPI2 (Linux), UIAutomation (Windows) already exist
- dogtail, pywinauto, pyax already extract semantic trees
- OSCP's contribution: unify, on-demand request, handle errors

### 2. On-Demand, Not Streaming

**Decision:** Request-response model instead of continuous 30fps streaming.

**Rationale:**
- Agent controls when to see screen (not passive stream)
- Simpler architecture (no stream management)
- Lower resource usage (idle when not called)
- Faster to build (less complexity)
- ~50-100ms latency per frame is acceptable for task-oriented agents

**Change from v0.3:** Removed 30fps streaming. Agent calls `oscp.getFrame()` when needed.

### 3. Per-Platform Best Approach

**Decision:** Use the best available existing tools per platform, wrapped uniformly.

| Platform | Existing Tool | OSCP Role |
|----------|--------------|-----------|
| macOS | AXUIElement | Wrap + respond on-demand |
| Linux | AT-SPI2 / X11 | Wrap + respond on-demand |
| Windows | UIAutomation | Wrap + respond on-demand |

### 4. Error-Resilient Design

**Decision:** 5-level fallback hierarchy for empty/unhelpful trees.

**Hierarchy:**
1. Native semantic tree (90% coverage)
2. CDP bridge (Electron/Browser)
3. Structural heuristics
4. Position-only mode
5. Human handoff

**Rationale:** Agent must NEVER get stuck. Always have next step.

### 5. Confidence Scoring

**Decision:** Every element and action includes confidence score.

**Rationale:**
- Agent needs to know data reliability
- Tree quality analysis enables smart decisions
- Enables "explore first" vs "execute immediately" decisions

### 6. Agent Provides the Meaning

**Decision:** Protocol delivers coordinates and types. Agent supplies semantics.

**Rationale:**
- Position, size, z-order patterns reveal UI structure
- Agent skills provide pattern matching (not VLM)
- Platform independence
- Deterministic (not probabilistic like VLMs)

### 7. Hardware-Level Actuation

**Decision:** Use OS-native input APIs (CGEvent, /dev/uinput, SendInput).

**Rationale:**
- OS cannot distinguish agent from human
- Bypasses bot detection
- Stable, well-documented

### 8. macOS First

**Decision:** Implement macOS before Linux and Windows.

**Rationale:**
- AXUIElement is most stable and well-documented
- No compositor fragmentation
- CGEvent is straightforward
- Proven wrappers exist

### 9. Linux AT-SPI2 First, Wayland Native Deferred

**Decision:** AT-SPI2 + X11/Xwayland for Linux. Native Wayland compositor bridges in V2.

**Rationale:**
- AT-SPI2 covers 85% of Linux apps
- X11/Xwayland covers most Wayland desktops
- Wayland compositor fragmentation is unsolvable in V1

### 10. No VLM in V1

**Decision:** VLM is a V2+ extension, not V1 scope.

**Rationale:**
- V1 must be fast, deterministic, cheap
- VLM is slow, expensive, probabilistic
- Keep scope tight for V1

### 11. Human Handoff for Edge Cases

**Decision:** Human escalation for truly unrecoverable.

**What human needs:**
- Games with custom renderers (50% recovery)
- DRM video playback (blocked)
- Canvas apps (Figma, CAD) (blocked)
- Remote desktop (physical limitation)

**What agent handles:**
- Standard desktop work (95%)
- Web (90% via CDP)
- Custom apps (80% with heuristics)
- Most daily agent tasks autonomously

---

## Why Existing Tools Aren't Enough

| Issue | dogtail | pywinauto | pyax | OSCP |
|-------|--------|-----------|------|------|
| Request-response model | ❌ One-shot | ❌ One-shot | ❌ One-shot | ✅ On-demand |
| Unified protocol | ❌ Per-platform | ❌ Per-platform | ❌ macOS only | ✅ All platforms |
| Error handling | ❌ Fail silently | ❌ Fail silently | ❌ Basic | ✅ Fallback hierarchy |
| Confidence scores | ❌ No | ❌ No | ❌ No | ✅ Per-element |
| Human handoff | ❌ Not supported | ❌ Not supported | ❌ Not supported | ✅ Escalation |
| Input engine | ❌ No | ⚠️ Basic | ❌ No | ✅ Full stack |

---

## Rejected Approaches

### Build from scratch (Rejected)
- AXUIElement, AT-SPI2, UIA are proven
- Years of development invested in existing tools

### Screenshot + VLM (Rejected for V1)
- Probabilistic, not deterministic
- Slow (~1-2s per frame) vs OSCP (~50-100ms)
- Expensive ($0.01-0.10/frame) vs OSCP ($0.001/frame)
- VLM is future extension, not V1 foundation

### Continuous Streaming (Rejected in v0.4)
- Higher complexity
- Passive agent (not active)
- Higher resource usage
- Agent doesn't need always-on awareness

### Pure position-based (Rejected)
- Agent loses element types, names
- Too much inference needed

---

## Revised Complexity

| Component | Complexity | Time |
|-----------|------------|------|
| Request handler | Low | 1 week |
| API wrapping | Low-Medium | 2-3 weeks |
| Fallback handling | Low | 1 week |
| Input engine | Low-Medium | 1-2 weeks |
| Testing | Medium-High | 2-3 weeks |

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| Windows | 6-7 weeks |

**Total: 10-12 weeks (down from 12-16 weeks)**

---

## VLM Fallback (V2+ Extension)

```
V1: OSCP on-demand (90% of cases)
    ↓ (if coverage < 0.3)
V2: VLM fallback (10% of cases)
    ↓ (if VLM fails)
V2: Human Handoff
```

---

## Platform-Specific Notes

### macOS
- Screen Recording + Accessibility permissions
- AXUIElement for capture
- CGEvent for input

### Linux
- at-spi2-core package + Accessibility enabled
- AT-SPI2 + X11 fallback
- /dev/uinput + XTest for input

### Windows
- UIAutomation on Windows 10+
- SendInput for input injection
- CDP bridge for Chrome/Edge