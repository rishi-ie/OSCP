# OSCP — Operating System Context Protocol

**Version:** 0.2.0
**Status:** Design Finalized

---

## Purpose

OSCP is a foundational protocol for agent-native OS interaction. It unifies existing OS accessibility APIs into a real-time, deterministic, agent-native interface.

**Key Insight:** The hard parts (semantic tree extraction) are already built. OSCP adds:
- Real-time streaming (30fps)
- Unified protocol (all platforms)
- Error handling (graceful degradation)
- Agent-native output (confidence scores)

---

## Principle

> "Wrap existing tools. Add real-time streaming. Handle errors gracefully. Agent provides meaning."

---

## What Already Exists

| Platform | Existing API | Status |
|----------|-------------|--------|
| macOS | AXUIElement | Proven, well-documented |
| Linux | AT-SPI2 | Proven, well-documented |
| Windows | UIAutomation | Proven, well-documented |

**These APIs already extract semantic trees. OSCP wraps them.**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT HARNESS (pi)                       │
│                                                                 │
│            ┌────────────────┬────────────────┐                 │
│            │                │                │                  │
│            ▼                ▼                ▼                  │
│     ┌──────────┐      ┌──────────┐      ┌──────────┐          │
│     │  macOS   │      │  Linux   │      │ Windows  │              │
│     │ Platform │      │ Platform │      │ Platform │              │
│     │  Driver  │      │  Driver  │      │  Driver  │              │
│     └──────────┘      └──────────┘      └──────┬───┘              │
│            │                │                   │                  │
│            └────────────────┴───────────────────┘                  │
│                         │                                        │
│                    Protocol Layer                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Per-Platform Approach

### macOS

**Wraps:** AXUIElement (Apple's accessibility API)

Existing tools:
- `pyax` - Python AXUIElement wrapper
- `ax-element` - Rust AXUIElement bindings
- `accessibility-service` - Native macOS

### Linux

**Wraps:** AT-SPI2 (D-Bus accessibility)

Existing tools:
- `dogtail` - Python AT-SPI2 library
- `pyatspi` - PyAT-SPI2 bindings
- `ldtp` - Linux Desktop Testing Project
- `at-spi2-core` - System accessibility daemon

### Windows

**Wraps:** UIAutomation + Win32

Existing tools:
- `pywinauto` - Python UIA/Win32 wrapper
- `UIAutomationCore` - Native COM API
- Win32 APIs (EnumWindows, GetWindowRect)

---

## What OSCP Adds

### 1. Real-Time Streaming

Existing tools are **one-shot queries**. OSCP provides **30fps continuous updates**.

```python
# Existing tools (dogtail, pywinauto):
tree = dogtail.tree.root  # One snapshot
# No updates unless you query again

# OSCP (real-time stream):
async for frame in client.stream():
    # 30fps updates
    # Agent sees every change
```

### 2. Unified Protocol

Existing tools have **per-platform APIs**. OSCP provides **same format everywhere**.

```python
# Existing tools:
if platform == 'linux':
    tree = dogtail.get_tree()
elif platform == 'windows':
    tree = pywinauto.get_tree()

# OSCP (unified):
tree = client.get_frame()  # Same on all platforms
```

### 3. Error Handling

Existing tools **fail silently**. OSCP provides **fallback hierarchy**.

```python
# Existing tools:
try:
    tree = get_tree()  # Might return empty
except:
    pass  # Silent failure

# OSCP (error handling):
if tree.coverage_score < 0.3:
    tree = try_cdp()  # Fallback
    if not tree:
        tree = position_only()  # Last resort
```

### 4. Confidence Scoring

Existing tools provide **no quality metrics**. OSCP provides **per-element confidence**.

```python
# Existing tools:
elements = tree.find_all()  # No confidence info

# OSCP:
for element in tree.elements:
    if element.confidence > 0.8:
        execute(element)  # High confidence
    else:
        explore_first(element)  # Low confidence
```

---

## Error Handling: The Empty Tree Problem

### The Challenge

Custom renderers, unlabeled icons, DRM cause empty/unhelpful trees.

### Fallback Hierarchy

```
LEVEL 1: Native Semantic Tree (90% of apps)
   └── AXUIElement / AT-SPI2 / UIA
   
LEVEL 2: CDP Bridge (Electron/Browser)
   └── Chrome DevTools Protocol
   
LEVEL 3: Structural Heuristics
   └── Position-based inference
   
LEVEL 4: Position-Only Mode
   └── Agent learns through exploration
   
LEVEL 5: Human Handoff
   └── Escalation for edge cases
```

### Tree Quality Analysis

```json
{
  "tree_analysis": {
    "coverage_score": 0.85,
    "named_elements": 150,
    "unlabeled_elements": 12,
    "confidence": "HIGH"
  }
}
```

### Confidence Thresholds

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| HIGH | > 0.8 | Execute immediately |
| MEDIUM | 0.5-0.8 | Execute with monitoring |
| LOW | 0.3-0.5 | Explore first |
| NONE | < 0.2 | Human handoff |

---

## What Agent Receives

### Standard Frame

```json
{
  "type": "render_tree",
  "frame_id": 12345,
  "platform": "linux",
  "windows": [
    {
      "id": "win_0x400001",
      "title": "VS Code",
      "bounds": {"x": 0, "y": 0, "w": 1920, "h": 1080},
      "focused": true,
      "elements": [
        {
          "id": "e_001",
          "type": "button",
          "name": "Save",
          "bounds": {"x": 1750, "y": 5, "w": 80, "h": 25},
          "confidence": 0.95,
          "source": "atspi"
        }
      ]
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.9,
    "named_elements": 150,
    "confidence": "HIGH"
  }
}
```

### Fallback Frame

```json
{
  "type": "render_tree",
  "frame_id": 12346,
  "platform": "windows",
  "windows": [
    {
      "id": "win_0x500001",
      "title": "CustomGame",
      "elements": [],
      "fallback_active": true,
      "fallback_method": "position_only",
      "fallback_reason": "custom_renderer_detected"
    }
  ],
  "tree_analysis": {
    "coverage_score": 0.05,
    "confidence": "NONE",
    "recommended_action": "human_handoff"
  }
}
```

---

## Revised Complexity

| Component | Complexity | Notes |
|-----------|------------|-------|
| **Semantic tree extraction** | Already done | 0 weeks |
| **Real-time streaming** | Medium | 2-3 weeks |
| **Unified protocol** | Low | 1 week |
| **Error handling** | Low | 1 week |
| **Input engine** | Low | 1 week |
| **Testing** | Medium | 2 weeks |

---

## Revised Time Estimates

| Platform | Original | Revised |
|----------|----------|---------|
| **macOS** | 6-8 weeks | 4-6 weeks |
| **Linux** | 10-14 weeks | 6-8 weeks |
| **Windows** | 7-9 weeks | 4-6 weeks |

**Total: 12-16 weeks**

---

## Directory Structure

```
OSCP/
├── protocol/           # Protocol specification
│   └── SPEC.md        # Core protocol
│
├── platforms/          # OS-specific drivers
│   ├── macos/         # Wraps AXUIElement
│   ├── linux/         # Wraps AT-SPI2
│   └── windows/       # Wraps UIA
│
├── agents/             # Agent integration
│   └── SPEC.md       # Agent SDK guidelines
│
└── README.md          # Project overview
```

---

## Status

🚧 **Phase 0** — Design complete. V1 implementation starting.
🚧 **Key insight:** Integration work, not new development.

---

*OSCP — First-class access for agents.*