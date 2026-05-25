# OSCP - Decisions

## Key Architectural Decisions

### 1. Integration, Not New Development

**Decision:** OSCP wraps existing OS accessibility APIs rather than building from scratch.

**Rationale:**
- AXUIElement (macOS), AT-SPI2 (Linux), UIAutomation (Windows) already exist
- dogtail, pywinauto, pyax already extract semantic trees
- The hard parts (tree extraction) are already done
- OSCP's contribution: unify, stream, handle errors

**Impact:** Revised time estimates down from 14-17 weeks to 12-16 weeks.

### 2. Real-Time Streaming (Not One-Shot)

**Decision:** Continuous 30fps render tree updates.

**Rationale:**
- Existing tools (dogtail, pywinauto) are one-shot queries
- Agent needs real-time awareness of screen state
- 30fps is sufficient for human-like interaction
- Streaming enables change detection and monitoring

### 3. Per-Platform Best Approach

**Decision:** Use the best available existing tools per platform, wrapped uniformly.

| Platform | Existing Tool | OSCP Role |
|----------|--------------|-----------|
| macOS | AXUIElement | Wrap + stream + error handle |
| Linux | AT-SPI2 / X11 | Wrap + stream + error handle |
| Windows | UIAutomation | Wrap + stream + error handle |

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
- AXUIElement is most stable and well-documented accessibility API
- No compositor fragmentation (unlike Linux Wayland)
- CGEvent is straightforward
- Proven wrappers exist (pyax, ax-element)

### 9. Linux AT-SPI2 First, Wayland Native Deferred

**Decision:** AT-SPI2 + X11/Xwayland for Linux. Native Wayland compositor bridges in V2.

**Rationale:**
- AT-SPI2 covers 85% of Linux apps
- X11/Xwayland covers most Wayland desktops
- Wayland compositor fragmentation is unsolvable in V1 (Mutter, KWin, Sway all different)
- Defer native Wayland to V2 when patterns emerge

### 10. No VLM in V1

**Decision:** VLM fallback is a V2+ extension, not V1 scope.

**Rationale:**
- V1 must be fast, deterministic, cheap
- VLM is slow, expensive, probabilistic
- VLM is the LAST fallback, not the foundation
- Keep V1 scope tight

### 11. Human Handoff for Edge Cases

**Decision:** Human escalation for truly unrecoverable cases.

**Rationale:**
- Agent should never be stuck
- Some things (DRM, audio, canvas) cannot be automated
- Human handoff = graceful degradation, not failure

---

## Why Existing Tools Aren't Enough

| Issue | dogtail | pywinauto | pyax | OSCP |
|-------|--------|-----------|------|------|
| Real-time streaming | ❌ One-shot | ❌ One-shot | ❌ One-shot | ✅ 30fps |
| Unified protocol | ❌ Per-platform | ❌ Per-platform | ❌ macOS only | ✅ All platforms |
| Error handling | ❌ Fail silently | ❌ Fail silently | ❌ Basic | ✅ Fallback hierarchy |
| Confidence scores | ❌ No | ❌ No | ❌ No | ✅ Per-element |
| Human handoff | ❌ Not supported | ❌ Not supported | ❌ Not supported | ✅ Escalation |
| Input engine | ❌ No | ⚠️ Basic | ❌ No | ✅ Full stack |

---

## Rejected Approaches

### Build from scratch (Rejected)
- AXUIElement, AT-SPI2, UIA are proven, stable APIs
- Years of development already invested in existing tools
- Reinventing the wheel

### Screenshot + VLM (Rejected for V1)
- Probabilistic, not deterministic
- Slow (~1-2s per frame) vs OSCP (<33ms per frame)
- Expensive ($0.01-0.10/frame) vs OSCP ($0.001/frame)
- VLM is future extension, not V1 foundation

### Pure position-based (Rejected)
- Agent loses element types, names, roles
- Too much inference needed
- Unreliable across different apps

### Per-app hooks (Rejected)
- One interception point at compositor level
- Not multiple app-specific integrations
- Simpler, more universal

---

## VLM Fallback (V2+ Extension)

**Note:** VLM is a future extension for edge cases (games, canvas apps, DRM).

V2 architecture:
```
OSCP V1 (primary, 90% of cases)
    ↓ (if coverage < 0.3)
VLM V2 (fallback, 10% of cases)
    ↓ (if VLM fails)
Human Handoff (truly edge cases)
```

---

## Future Decisions (V2+)

### Windows Render Operations
- DWM hook (experimental, risky)
- Kernel driver (proper but complex)
- Status: Deferred from V1

### Wayland Native Support
- Per-compositor IPC (Mutter, KWin, Sway)
- Wait for patterns to emerge
- Status: Deferred to V2

### Protocol Extensions
- Color sampling
- Audio capture
- Hardware access

---

## Platform-Specific Notes

### macOS
- Screen Recording + Accessibility permissions required
- AXObserver for real-time events
- CGEvent for input injection

### Linux
- at-spi2-core package required
- Accessibility must be enabled in desktop environment
- /dev/uinput + XTest for input

### Windows
- UIAutomation available on Windows 10+
- SendInput for input injection
- CDP bridge for Chrome/Edge