# OSCP - Decisions

## Key Architectural Decisions

### 1. Deterministic Semantic Architecture (Not Visual)

**Decision:** Use OS semantic trees (UIA, AXUIElement, AT-SPI2) instead of visual parsing (screenshots, VLMs).

**Rationale:**
- Semantic trees are mathematical facts, not guesses
- Probabilistic vs deterministic: 100% vs 70% reliability
- No pixel analysis, no hallucination risk
- Agent receives exact coordinates with element names/types

### 2. Hardware-Level Actuation

**Decision:** Use OS-native input APIs (CGEvent, SendInput, /dev/uinput) for actions.

**Rationale:**
- OS cannot distinguish agent from human
- Bypasses application-level bot detection
- Kernel-level interrupts = undetectable
- Stealth and reliability

### 3. Error-Resilient Design

**Decision:** Comprehensive fallback hierarchy for empty/unhelpful trees.

**Rationale:**
- Custom renderers, unlabeled icons, DRM all cause empty trees
- Agent must NEVER get stuck
- Fallback chain: Native → CDP → Heuristics → Position-only → Human

### 4. Per-Platform Best Approach

**Decision:** Use the best available method per platform, not a single universal approach.

| Platform | Primary | Fallback |
|----------|---------|----------|
| macOS | AXUIElement | CDP, Position-only |
| Linux | AT-SPI2 | X11, CDP, Heuristics |
| Windows | UIA | CDP, Win32, Position-only |

**Rationale:**
- Each OS has different accessibility APIs
- Unified output hides implementation details
- No one-size-fits-all exists

### 5. Confidence Scoring

**Decision:** Every element and action includes confidence score.

**Rationale:**
- Agent needs to know how reliable the data is
- High confidence: execute immediately
- Low confidence: explore first or handoff
- Transparency enables better agent decisions

### 6. Tree Quality Analysis

**Decision:** Continuously measure tree quality to detect problematic apps.

**Metrics:**
- `coverage_score` — percentage of UI covered
- `named_elements` — elements with text labels
- `unlabeled_elements` — elements without text
- `avg_depth` — tree depth (shallow = custom renderer)

**Trigger rules:**
- `coverage_score < 0.3` → Low confidence
- `named_ratio < 0.5` → Many unlabeled
- `avg_depth < 2` → Possible canvas/custom renderer

### 7. CDP Bridge for Browser/Electron Apps

**Decision:** Chrome DevTools Protocol for apps where native accessibility fails.

**Rationale:**
- Electron apps (VS Code, Slack, Discord) expose DOM via CDP
- Same data as semantic tree, different source
- High reliability for browser-based UIs
- Covers 95% of modern productivity apps

### 8. Agent Provides the Meaning

**Decision:** Protocol delivers coordinates and types. Agent supplies semantics.

**Rationale:**
- Position, size, z-order patterns reveal UI structure
- Agent skills provide pattern matching and learning
- Platform independence
- Same agent works on all OSes

### 9. Graceful Degradation

**Decision:** Never fail silently. Always provide a path forward.

```
If semantic tree works → Use it
If semantic tree fails → Try CDP
If CDP fails → Try heuristics
If heuristics fail → Position-only mode
If position-only fails → Human handoff
```

**Rationale:**
- Edge cases are handled systematically
- Agent always has next step
- Human escalation as safety net

## Rejected Approaches

### Per-App Hooks (DXGI, OpenGL, Vulkan)
**Rejected:** Complex, fragile, per-app, only covers specific graphics APIs.

### Screenshots + VLM
**Rejected:** Probabilistic, slow, expensive, not deterministic.

### Pure Position-Based (No Semantic)
**Rejected:** Agent loses element types and names. Too much inference needed.

### No Fallback (Single Source Only)
**Rejected:** Agent gets stuck on custom renderers, unlabeled icons, DRM.

### Cloud-Based Processing
**Rejected:** Latency, privacy, dependency on external services.

## Future Decisions (V2+)

### Windows Render Operations (DWM Hook)
- Option: DWM DirectX hook (experimental, risky)
- Option: Kernel driver (proper but complex)
- Status: Deferred from V1

### Protocol Extensions
- Color sampling (for error state detection)
- Audio capture API
- Hardware access API

### CDP Enhancement
- Real-time DOM mutation tracking
- CSS computed styles for better inference
- Shadow DOM support

## Platform-Specific Notes

### macOS
- "Screen Recording" permission name is misleading
- Actually enables Window Server layer tree access
- AXUIElement provides full semantic tree

### Linux
- AT-SPI2 via D-Bus (standard accessibility)
- X11 as fallback (XQueryTree, XGetWindowProperty)
- Wayland: per-compositor support needed

### Windows
- UIAutomation is the primary semantic API
- Win32 as fallback (EnumWindows, GetWindowRect)
- WMI for system state
- DWM (Desktop Window Manager) render ops: no documented API