# OSCP - Project Overview

## What OSCP Is

OSCP is an MCP server that gives CLI agents eyes. It captures the OS accessibility tree—no screenshots, no VLM—and exposes it via MCP.

**Version:** 0.5.0
**Status:** Specs Complete. Ready for Implementation.

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

## References

- GitHub: github.com/rishi-ie/OSCP