# OSCP - Project Overview

## What OSCP Is

OSCP (Operating System Context Protocol) enables AI agents to see and interact with desktop applications using native OS accessibility APIs—no screenshots, no VLM required.

**Version:** 0.4.0
**Status:** Specs Complete. Ready for Implementation.

---

## Core Capabilities

### 1. On-Demand Screen Capture

- Request screen state when needed (not streaming)
- Returns all visible windows with titles, bounds, focus state
- Returns every UI element with names, roles, positions, values
- Reports per-element and per-frame confidence scores
- Reports current mouse position and hovered element

### 2. Native OS Integration

| Platform | Primary API | Coverage |
|----------|-------------|----------|
| macOS | AXUIElement | 95% |
| Linux | AT-SPI2 + X11 | 90-95% |
| Windows | UIAutomation | 85% |

### 3. Supported Element Types

All standard desktop UI elements:
- Buttons, checkboxes, radio buttons
- Text fields, text areas, secure fields
- Combo boxes, dropdowns, sliders
- Menus, menu bars, menu items
- Tabs, tab groups, panels
- Lists, list items, tables, cells
- Links, images, icons
- Windows, dialogs, alerts, sheets
- Groups, toolbars, scroll areas

### 4. Action Execution

**Mouse:** click (single/double/triple/right), drag, move, scroll

**Keyboard:** type text, key combinations (Ctrl+S, Cmd+S, etc.)

### 5. Quality Scoring

- Per-frame coverage analysis (named area / window area)
- Confidence levels: HIGH (>80%), MEDIUM (50-80%), LOW (30-50%), NONE (<30%)
- Per-element confidence based on name, role, size, source

### 6. Intelligent Fallbacks

5-level fallback chain when native APIs fail:
1. Native semantic tree (90% coverage)
2. CDP bridge (Safari/Chrome/Electron)
3. Structural heuristics (position inference)
4. Position-only mode
5. Human handoff

### 7. Error Handling

Actionable errors with alternatives:
- ELEMENT_NOT_FOUND, ELEMENT_DISABLED, ELEMENT_MOVED
- ACTION_FAILED (with alternative positions)
- EMPTY_TREE, LOW_COVERAGE
- Human handoff escalation

### 8. Human-Informed Learning

- Agents can learn from human interventions
- Record positions for future reference
- Graceful escalation when stuck

---

## Spec Status

| Spec | Status | Size |
|------|--------|------|
| Protocol | ✅ Complete | 44KB |
| macOS | ✅ Complete | 145KB |
| Linux | ✅ Complete | 31KB |
| Windows | ⏸️ Deferred | - |

---

## Implementation Stack

### macOS
- Swift/Objective-C
- AXUIElement for capture
- CGEvent for input
- Unix socket protocol server

### Linux
- Python (pyatspi) or Rust
- AT-SPI2 + X11 for capture
- /dev/uinput + XTest for input

---

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| **Total** | **10-12 weeks** |

---

## Key Principle

> "Wrap existing tools. Respond on-demand. Handle errors gracefully. Agent provides meaning."

---

## References

- GitHub: github.com/rishi-ie/OSCP