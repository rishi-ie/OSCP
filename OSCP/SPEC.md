# OSCP Architecture Specification

**Version:** 0.5.0
**Status:** Specs Complete. Ready for Implementation.

---

## What OSCP Is

OSCP is an MCP server that gives CLI agents the ability to see desktop applications through native OS accessibility APIs.

**Scope:** Discovery only. Agent handles interaction via its own tools.

---

## Core Components

### 1. MCP Server

- CLI agent integration via Model Context Protocol
- Single binary, runs as `--mcp`
- No HTTP, no socket server—just MCP

### 2. Native Tree Capture

| Platform | API | Coverage |
|----------|-----|----------|
| macOS | AXUIElement | 90% |
| Linux | AT-SPI2 | 85% |
| Windows | UIAutomation | 90% |

### 3. CDP Fallback

- Safari, Chrome, Electron apps
- DOM tree with bounding boxes
- When native API can't see content

### 4. Screenshot Fallback

- Games, custom renderers, DRM
- PNG screenshot
- Agent uses VLM externally

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
  "enabled": true,
  "children": []
}
```

---

## Capture Pipeline

```
1. Native API (AXUIElement/AT-SPI2/UIA)
   └── 90% coverage

2. CDP DOM Bridge
   └── Browsers, Electron

3. Screenshot
   └── Games, custom renderers
```

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

## Status

- [x] Protocol specification complete
- [x] Platform specs complete
- [ ] Implementation pending
