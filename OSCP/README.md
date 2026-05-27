# OSCP — Operating System Context Protocol

**Version:** 0.5.0
**Status:** Specs Complete. Ready for Implementation.

---

## Vision

> Give CLI agents eyes. See any desktop app without screenshots.

OSCP captures the OS accessibility tree and exposes it to AI agents via MCP. No screenshots, no VLM—just structured JSON of every button, field, and menu.

---

## What OSCP Does

OSCP runs as an MCP server. When an agent needs to see the desktop, OSCP:

1. **Captures** the current screen as a semantic tree using native OS accessibility APIs
2. **Falls back** to CDP for web content (Safari, Chrome, Electron apps)
3. **Falls back** to screenshot for games and custom renderers
4. **Returns** a unified element tree the agent can query

That's it. The agent decides what to do with it.

---

## Architecture

```
CLI AGENT                           OSCP (MCP Server)
    │                                     │
    │ ──── list_windows ────────────────►│
    │ ◄─── [{title, pid, bounds}] ───────│
    │                                     │
    │ ──── get_tree(pid) ────────────────►│
    │                                     ├─► AXUIElement / AT-SPI2 / UIA
    │                                     ├─► CDP (if browser)
    │                                     ├─► Screenshot (if needed)
    │ ◄─── {elements: [...]} ─────────────│
    │                                     │
    │ Agent decides action                │
    │ (uses Python/macOS automation)       │
```

---

## Capture Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    OSCP CAPTURE                            │
│                                                             │
│  1. NATIVE ACCESSIBILITY                                   │
│     ├── macOS: AXUIElement                                │
│     ├── Linux: AT-SPI2                                    │
│     └── Windows: UIAutomation                            │
│                                                             │
│  2. CDP DOM BRIDGE (fallback)                             │
│     └── Safari, Chrome, VS Code, Electron                  │
│         └── DOM tree with bounding boxes                   │
│                                                             │
│  3. SCREENSHOT (last resort)                              │
│     └── Games, custom renderers, DRM                       │
│         └── PNG + VLM ready (agent uses VLM externally)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## MCP Tools

### `list_windows`

Returns all visible windows.

```json
{
  "windows": [
    {
      "pid": 1234,
      "title": "Visual Studio Code",
      "bounds": {"x": 0, "y": 0, "width": 1920, "height": 1080},
      "app": "com.microsoft.VSCode"
    }
  ]
}
```

### `get_tree`

Returns the full element tree for a window.

```json
{
  "pid": 1234,
  "source": "axuielement",
  "elements": [
    {
      "id": "e_001",
      "role": "button",
      "name": "Save",
      "bounds": {"x": 1750, "y": 5, "width": 80, "height": 25},
      "enabled": true
    },
    {
      "id": "e_002",
      "role": "text_field",
      "name": "Search",
      "value": "",
      "bounds": {"x": 300, "y": 10, "width": 300, "height": 30},
      "enabled": true
    }
  ]
}
```

### `find_elements`

Search elements by name or type.

```json
{
  "elements": [
    {
      "id": "e_001",
      "role": "button",
      "name": "Save",
      "bounds": {...}
    }
  ]
}
```

---

## Element Format

Every element includes:

| Field | Example |
|-------|---------|
| `id` | `"e_001"` |
| `role` | `"button"`, `"text_field"`, `"menu_item"`, etc. |
| `name` | `"Save"`, `"Search"`, `"File"` |
| `value` | Current value (for text fields, etc.) |
| `bounds` | `{"x": 100, "y": 50, "width": 80, "height": 25}` |
| `enabled` | `true` / `false` |
| `focused` | `true` / `false` |

### Supported Roles

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

## Capture Coverage

| Platform | Native API | Coverage |
|----------|------------|----------|
| macOS | AXUIElement | 90% |
| Linux | AT-SPI2 | 85% |
| Windows | UIAutomation | 90% |

| App Type | Coverage | Fallback |
|----------|----------|----------|
| Native apps | Excellent | — |
| Electron (VS Code, Slack) | Good | CDP |
| Browsers | Partial | CDP |
| Games, CAD | None | Screenshot |

---

## CDP Fallback

When native APIs can't see the content (browsers, Electron), OSCP connects to the browser's CDP endpoint:

```json
{
  "source": "cdp",
  "url": "https://github.com/rishi-ie/OSCP",
  "elements": [
    {
      "id": "cdp_001",
      "role": "link",
      "name": "OSCP",
      "tag": "A",
      "bounds": {...}
    }
  ]
}
```

---

## Screenshot Fallback

When everything else fails (games, custom engines):

```json
{
  "source": "screenshot",
  "width": 1920,
  "height": 1080,
  "data": "base64_encoded_png..."
}
```

Agent can then use a VLM externally to analyze.

---

## CLI Agent Integration

Add to your MCP config:

```json
{
  "mcpServers": {
    "oscp": {
      "command": "oscp",
      "args": ["--mcp"]
    }
  }
}
```

Works with Claude Code, Cursor, Windsurf, or any MCP-compatible CLI agent.

---

## Directory Structure

```
OSCP/
├── README.md                    # This file
├── SPEC.md                     # Architecture overview
├── protocol/
│   └── SPEC.md                 # MCP protocol + element format
├── platforms/
│   ├── macos/SPEC.md           # macOS implementation
│   ├── linux/SPEC.md          # Linux implementation
│   └── windows/SPEC.md        # Windows implementation
├── agents/
│   └── SPEC.md                 # Agent integration guide
└── MEMORY/                    # Project context
```

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

---

## References

- **GitHub:** github.com/rishi-ie/OSCP
- **MCP Protocol:** modelcontextprotocol.io

---

*OSCP — Desktop awareness for CLI agents.*