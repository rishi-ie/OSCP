# OSCP Architecture Specification

**Version:** 0.4.0
**Status:** Specs Complete. Ready for Implementation.

---

## What OSCP Is

OSCP (Operating System Context Protocol) enables AI agents to see and interact with desktop applications using native OS accessibility APIs—no screenshots, no VLM required.

---

## Core Capabilities

### 1. On-Demand Screen Capture

| Data | Description |
|------|-------------|
| Windows | Titles, bounds, focus state, owning app |
| Elements | Names, roles, positions, values |
| Quality | Confidence scores, coverage metrics |
| Mouse | Current position, hovered element |

### 2. Native OS Integration

| Platform | API | Coverage |
|----------|-----|----------|
| macOS | AXUIElement | 95% |
| Linux | AT-SPI2 + X11 | 90-95% |
| Windows | UIAutomation | 85% |

### 3. Supported Element Types

- Buttons, checkboxes, radio buttons
- Text fields, text areas, secure fields
- Combo boxes, dropdowns, sliders
- Menus, menu bars, menu items
- Tabs, tab groups, panels
- Lists, tables, cells
- Links, images, icons
- Windows, dialogs, alerts
- Groups, toolbars, scroll areas

### 4. Action Execution

**Mouse:** click (single/double/triple/right), drag, move, scroll

**Keyboard:** type text, key combos (Ctrl+S, Cmd+S, Alt+Tab, etc.)

### 5. Quality Scoring

| Confidence | Threshold | Agent Action |
|------------|-----------|--------------|
| HIGH | > 80% | Execute immediately |
| MEDIUM | 50-80% | Execute with monitoring |
| LOW | 30-50% | Explore first |
| NONE | < 30% | Human handoff |

### 6. 5-Level Fallback Chain

| Level | Method | Coverage |
|-------|--------|----------|
| 1 | Native semantic tree | 90% |
| 2 | CDP bridge | Browsers |
| 3 | Structural heuristics | Position inference |
| 4 | Position-only | Custom renderers |
| 5 | Human handoff | Graceful degradation |

### 7. Error Handling

- Actionable errors with alternatives
- Human handoff escalation
- Learning from human interventions

---

## Specs

| Spec | Status | Size |
|------|--------|------|
| Protocol | ✅ Complete | 44KB |
| macOS | ✅ Complete | 145KB |
| Linux | ✅ Complete | 31KB |
| Windows | ⏸️ Deferred | - |

---

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| **Total** | **10-12 weeks** |

---

## Status

- [x] Protocol specification complete
- [x] macOS platform detailed spec complete
- [x] Linux platform detailed spec complete
- [ ] macOS implementation pending
- [ ] Linux implementation pending
- [ ] Windows implementation deferred