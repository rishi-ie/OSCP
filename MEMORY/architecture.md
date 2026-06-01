# OSCP Architecture

## Overview

OSCP = MCP Server + Tree Capture + CDP Bridge + Screenshot Fallback

**Purpose:** Give CLI agents eyes. See any desktop app without screenshots.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI AGENT                                │
│              (Claude Code, Cursor, etc.)                    │
└─────────────────────┬───────────────────────────────────────┘
                      │ MCP
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                     OSCP (MCP Server)                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  MCP Handler                         │    │
│  │   - list_windows                                    │    │
│  │   - get_tree                                        │    │
│  │   - find_elements                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               Discovery Layer                         │    │
│  │                                                     │    │
│  │   1. Native API ────────► Element Tree             │    │
│  │      (AXUIElement/AT-SPI2/UIA)                     │    │
│  │                                                     │    │
│  │   2. CDP Bridge ─────────► DOM Tree               │    │
│  │      (if browser detected)                         │    │
│  │                                                     │    │
│  │   3. Screenshot ─────────► PNG + Base64          │    │
│  │      (if all else fails)                          │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Components

### MCP Server
- Protocol: MCP (modelcontextprotocol.io)
- Transport: stdin/stdout (JSON-RPC 2.0)
- Mode: `--mcp` flag

### Native Capture
| Platform | API | Coverage |
|----------|-----|----------|
| macOS | AXUIElement | 90% |
| Linux | AT-SPI2 | 85% |
| Windows | UIAutomation | 90% |

### CDP Bridge
- Connects to browser CDP endpoint
- Returns DOM tree with bounding boxes
- For Safari, Chrome, Electron apps

### Screenshot Fallback
- Returns PNG as base64
- Used when native + CDP fail
- Agent uses VLM externally

---

## MCP Tools

| Tool | Input | Output |
|------|-------|--------|
| `list_windows` | None | `[{pid, title, bounds, app}]` |
| `get_tree` | `pid` | `{pid, source, elements: [...]}` |
| `find_elements` | `pid, query?, type?` | `{pid, source, elements: [...]}` |

---

## Element Format

```json
{
  "id": "e_001",
  "role": "button",
  "name": "Save",
  "value": null,
  "bounds": {"x": 100, "y": 50, "width": 80, "height": 25},
  "enabled": true,
  "focused": false,
  "children": []
}
```

---

## Capture Pipeline

```
┌─────────────────────────────────────────┐
│         Agent requests tree             │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│     Try Native API (AXUIElement/etc.)   │
│     Coverage: 90%                       │
└─────────────────┬───────────────────────┘
                  │
        ┌─────────┴─────────┐
        │ coverage < 30%?   │
        └─────────┬─────────┘
                  │ Yes
                  ▼
┌─────────────────────────────────────────┐
│        Try CDP Bridge (browser)          │
│     Coverage: +5-10%                     │
└─────────────────┬───────────────────────┘
                  │
        ┌─────────┴─────────┐
        │ still insufficient?│
        └─────────┬─────────┘
                  │ Yes
                  ▼
┌─────────────────────────────────────────┐
│        Screenshot fallback               │
│     Returns PNG base64                   │
│     Agent uses VLM externally           │
└─────────────────────────────────────────┘
```

---

## File Structure

```
src/
├── main.rs              # Entry point, CLI args
├── mcp.rs               # MCP server (JSON-RPC)
├── platform/
│   ├── mod.rs           # UiBackend trait
│   ├── macos.rs         # AXUIElement implementation
│   ├── linux.rs         # AT-SPI2 implementation
│   └── windows.rs       # UIA implementation
├── capture/
│   ├── native.rs        # Native API capture
│   ├── cdp.rs           # CDP bridge
│   └── screenshot.rs    # Screenshot fallback
└── types.rs             # Element, Window, etc.
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

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 3-4 weeks |
| Linux | 4-5 weeks |
| Windows | 4-5 weeks |
| **Total** | **11-14 weeks** |