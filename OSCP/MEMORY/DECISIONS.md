# OSCP - Decisions

## Key Architectural Decisions

### 1. Integration, Not New Development

**Decision:** OSCP wraps existing OS accessibility APIs rather than building from scratch.

**Rationale:**
- AXUIElement (macOS), AT-SPI2 (Linux), UIA (Windows) already exist
- dogtail, pywinauto, pyax already extract semantic trees
- The hard parts (tree extraction) are already done
- OSCP's contribution: unify, stream, handle errors

### 2. Real-Time Streaming (Not One-Shot)

**Decision:** Continuous 30fps render tree updates.

**Rationale:**
- Existing tools (dogtail, pywinauto) are one-shot queries
- Agent needs real-time awareness
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

**Decision:** Comprehensive fallback hierarchy for empty/unhelpful trees.

**Rationale:**
- Custom renderers, unlabeled icons, DRM cause empty trees
- Agent must NEVER get stuck
- Existing tools fail silently; OSCP provides alternatives

### 5. Confidence Scoring

**Decision:** Every element and action includes confidence score.

**Rationale:**
- Agent needs to know data reliability
- Tree quality analysis enables smart decisions
- Existing tools provide no quality metrics

### 6. Agent Provides the Meaning

**Decision:** Protocol delivers coordinates and types. Agent supplies semantics.

**Rationale:**
- Position, size, z-order patterns reveal UI structure
- Agent skills provide pattern matching
- Platform independence

### 7. Hardware-Level Actuation

**Decision:** Use OS-native input APIs (CGEvent, SendInput, /dev/uinput).

**Rationale:**
- OS cannot distinguish agent from human
- Bypasses bot detection
- Stable, well-documented

## What OSCP Adds That's New

| Feature | Existing Tools | OSCP |
|---------|----------------|------|
| Real-time streaming | ❌ One-shot only | ✅ 30fps |
| Unified protocol | ❌ Per-platform APIs | ✅ All OS |
| Error handling | ❌ Fail silently | ✅ Fallbacks |
| Confidence scores | ❌ No quality metrics | ✅ Per-element |
| Human handoff | ❌ Not supported | ✅ Escalation |
| Agent-native output | ❌ Human-native | ✅ JSON + confidence |

## Why Existing Tools Aren't Enough

### dogtail (Linux)
- One-shot queries, not streaming
- No confidence scores
- No error handling
- Human-centric API

### pywinauto (Windows)
- One-shot queries, not streaming
- No fallback for empty trees
- No tree quality analysis
- Windows only, not cross-platform

### pyax (macOS)
- Basic AXUIElement wrapper
- No streaming
- No error handling
- macOS only

## Rejected Approaches

### Build from scratch (No)
- AXUIElement, AT-SPI2, UIA are proven
- Years of development already invested
- Reinventing wheel

### Screenshot + VLM (No)
- Probabilistic, not deterministic
- Slow, expensive
- No semantic tree

### Pure position-based (No)
- Agent loses element types/names
- Too much inference needed
- Unreliable

## Time Estimates (Revised)

| Platform | Original | Revised |
|----------|----------|---------|
| macOS | 6-8 weeks | 4-6 weeks |
| Linux | 10-14 weeks | 6-8 weeks |
| Windows | 7-9 weeks | 4-6 weeks |

**Key insight:** The hard parts are already built. OSCP is integration work.

## Future Decisions (V2+)

### Windows Render Operations
- DWM hook (experimental, risky)
- Kernel driver (proper but complex)
- Status: Deferred from V1

### Protocol Extensions
- Color sampling
- Audio capture
- Hardware access

## Platform-Specific Notes

### macOS
- "Screen Recording" permission enables AXUIElement access
- Existing tools: `pyax`, `ax-element`, `accessibility-service`

### Linux
- AT-SPI2 via D-Bus (standard)
- Existing tools: `dogtail`, `pyatspi`, `ldtp`
- X11 as fallback

### Windows
- UIAutomation is primary
- Existing tools: `pywinauto`, `UIAutomationCore`
- Win32 as fallback