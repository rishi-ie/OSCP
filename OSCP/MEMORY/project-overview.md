# OSCP - Project Overview

## What OSCP Is

OSCP is an MCP server that gives CLI agents eyes. It captures the OS accessibility tree—no screenshots, no VLM—and exposes it via MCP.

**Version:** 0.5.0
**Status:** Specs Complete. Ready for Implementation.

---

## Vision

> Give CLI agents eyes. See any desktop app without screenshots.

---

## Scope

**Only discovery. Agent handles interaction.**

| Included | Excluded |
|----------|----------|
| MCP server | HTTP REST |
| Native tree capture | Interaction layer |
| CDP fallback | Unix socket |
| Screenshot fallback | Dashboard |
| Unified element format | Recorder |

---

## Capture Pipeline

```
1. Native API (AXUIElement/AT-SPI2/UIA) — 90% coverage
2. CDP DOM Bridge — Browsers, Electron
3. Screenshot — Games, custom renderers
```

---

## MCP Tools

| Tool | Description |
|------|-------------|
| `list_windows` | All visible windows |
| `get_tree` | Full element tree for a window |
| `find_elements` | Search by name/type |

---

## Element Format

```json
{
  "id": "e_001",
  "role": "button",
  "name": "Save",
  "bounds": {"x": 100, "y": 50, "width": 80, "height": 25},
  "enabled": true
}
```

---

## Platform Coverage

| Platform | API | Coverage |
|----------|-----|----------|
| macOS | AXUIElement | 90% |
| Linux | AT-SPI2 | 85% |
| Windows | UIAutomation | 90% |

---

## Supported Element Roles

- `window`, `dialog`, `alert`
- `button`, `check_box`, `radio_button`
- `text_field`, `text_area`, `secure_field`
- `combo_box`, `drop_down`
- `menu`, `menu_bar`, `menu_item`
- `tab`, `tab_group`
- `list`, `list_item`
- `table`, `row`, `cell`
- `tool_bar`, `group`
- `link`, `image`, `icon`
- `static_text`, `label`

---

## Spec Status

| Spec | Status |
|------|--------|
| Protocol | ✅ Complete |
| macOS | ✅ Complete |
| Linux | ✅ Complete |
| Windows | ✅ Complete |

---

## Implementation Timeline

| Platform | Time |
|----------|------|
| macOS | 3-4 weeks |
| Linux | 4-5 weeks |
| Windows | 4-5 weeks |
| **Total** | **11-14 weeks** |

---

## Directory Structure

```
OSCP/
├── README.md                    # Project overview
├── SPEC.md                     # Architecture summary
├── protocol/
│   └── SPEC.md                 # MCP protocol + element format
├── platforms/
│   ├── macos/SPEC.md           # macOS implementation
│   ├── linux/SPEC.md           # Linux implementation
│   └── windows/SPEC.md        # Windows implementation
├── agents/
│   └── SPEC.md                 # Agent integration guide
└── MEMORY/                    # Project context
```

---

## References

- GitHub: github.com/rishi-ie/OSCP
- MCP Protocol: modelcontextprotocol.io