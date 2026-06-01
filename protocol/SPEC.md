# OSCP Protocol Specification

**Version:** 0.5.0
**Status:** Ready for Implementation

---

## Overview

OSCP uses the Model Context Protocol (MCP) for agent integration. OSCP runs as an MCP server; CLI agents connect via MCP.

---

## MCP Transport

- **Mode:** `--mcp` flag
- **Protocol:** JSON-RPC 2.0 over stdin/stdout
- **Spec:** modelcontextprotocol.io

---

## MCP Server Setup

```rust
// Entry point
if args.mcp {
    mcp::run_mcp(backend);
}
```

---

## MCP Tools

### `list_windows`

Returns all visible windows.

**Parameters:** None

**Response:**
```json
{
  "windows": [
    {
      "pid": 1234,
      "title": "Visual Studio Code",
      "bounds": {
        "x": 0,
        "y": 0,
        "width": 1920,
        "height": 1080
      },
      "app": "com.microsoft.VSCode"
    }
  ]
}
```

---

### `get_tree`

Returns the full element tree for a window.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| `pid` | number | Process ID of the window |

**Response:**
```json
{
  "pid": 1234,
  "source": "axuielement",
  "elements": [
    {
      "id": "e_001",
      "role": "button",
      "name": "Save",
      "value": null,
      "bounds": {
        "x": 1750,
        "y": 5,
        "width": 80,
        "height": 25
      },
      "enabled": true,
      "focused": false,
      "children": []
    },
    {
      "id": "e_002",
      "role": "text_field",
      "name": "Search Files",
      "value": "",
      "bounds": {
        "x": 300,
        "y": 10,
        "width": 300,
        "height": 30
      },
      "enabled": true,
      "focused": true,
      "children": []
    }
  ]
}
```

---

### `find_elements`

Search elements within a window.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| `pid` | number | Process ID |
| `query` | string? | Name to search for |
| `type` | string? | Element role (e.g., "button") |
| `interactive_only` | boolean? | Only actionable elements |

**Response:**
```json
{
  "pid": 1234,
  "source": "axuielement",
  "elements": [
    {
      "id": "e_001",
      "role": "button",
      "name": "Save",
      "bounds": {
        "x": 1750,
        "y": 5,
        "width": 80,
        "height": 25
      },
      "enabled": true
    }
  ]
}
```

---

## Element Schema

```typescript
interface Window {
  pid: number;
  title: string;
  bounds: Rect;
  app: string;
}

interface Element {
  id: string;
  role: string;
  name: string | null;
  value: string | null;
  bounds: Rect;
  enabled: boolean;
  focused: boolean;
  children: Element[];
}

interface Rect {
  x: number;
  y: number;
  width: number;
  height: number;
}
```

---

## Element Roles

| Role | Description |
|------|-------------|
| `window` | Top-level window |
| `dialog` | Dialog window |
| `alert` | Alert/popup |
| `button` | Push button |
| `check_box` | Checkbox |
| `radio_button` | Radio button |
| `text_field` | Text input |
| `text_area` | Multi-line text |
| `secure_field` | Password field |
| `combo_box` | Dropdown |
| `menu` | Menu |
| `menu_bar` | Menu bar |
| `menu_item` | Menu item |
| `tab` | Tab |
| `tab_group` | Tab container |
| `list` | List |
| `list_item` | List item |
| `table` | Table |
| `row` | Table row |
| `cell` | Table cell |
| `tool_bar` | Toolbar |
| `group` | Group container |
| `link` | Hyperlink |
| `image` | Image |
| `static_text` | Static text |
| `label` | Label |
| `unknown` | Unknown type |

---

## Source Types

| Source | Description |
|--------|-------------|
| `axuielement` | macOS AXUIElement |
| `atspi` | Linux AT-SPI2 |
| `uia` | Windows UIAutomation |
| `cdp` | Chrome DevTools Protocol (DOM) |
| `screenshot` | Screenshot fallback |
| `unknown` | Unknown source |

---

## CDP Fallback

When native APIs can't see browser content:

```json
{
  "pid": 5678,
  "source": "cdp",
  "url": "https://github.com/rishi-ie/OSCP",
  "elements": [
    {
      "id": "cdp_001",
      "role": "link",
      "name": "OSCP",
      "tag": "A",
      "href": "https://...",
      "bounds": {
        "x": 100,
        "y": 50,
        "width": 50,
        "height": 20
      }
    }
  ]
}
```

---

## Screenshot Fallback

When everything else fails:

```json
{
  "pid": 9999,
  "source": "screenshot",
  "width": 1920,
  "height": 1080,
  "data": "base64_encoded_png..."
}
```

Agent can use VLM externally to analyze.

---

## MCP Protocol Implementation

```rust
// JSON-RPC 2.0 types
#[derive(Deserialize)]
struct RpcRequest {
    jsonrpc: String,
    id: Value,
    method: String,
    params: Value,
}

#[derive(Serialize)]
struct RpcResponse {
    jsonrpc: &'static str,
    id: Value,
    result: Option<Value>,
    error: Option<RpcError>,
}

// Method dispatch
fn dispatch(backend: &Backend, req: RpcRequest) -> Result<RpcResponse> {
    match req.method.as_str() {
        "initialize" => Ok(initialize()),
        "tools/list" => Ok(list_tools()),
        "tools/call" => call_tool(backend, req.params),
        _ => Err(unknown_method(req.method)),
    }
}
```

---

## Tool Schema

```json
{
  "name": "list_windows",
  "description": "List all visible windows",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}

{
  "name": "get_tree",
  "description": "Get the full element tree for a window",
  "inputSchema": {
    "type": "object",
    "properties": {
      "pid": {
        "type": "number",
        "description": "Process ID of the window"
      }
    },
    "required": ["pid"]
  }
}

{
  "name": "find_elements",
  "description": "Find elements by name or type",
  "inputSchema": {
    "type": "object",
    "properties": {
      "pid": {
        "type": "number"
      },
      "query": {
        "type": "string"
      },
      "type": {
        "type": "string"
      },
      "interactive_only": {
        "type": "boolean"
      }
    },
    "required": ["pid"]
  }
}
```

---

## Error Handling

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32603,
    "message": "Window not found",
    "data": {
      "pid": 1234
    }
  }
}
```

---

## Status

- [x] MCP tools defined
- [x] Element schema defined
- [x] Source types defined
- [x] CDP fallback defined
- [x] Screenshot fallback defined
- [ ] Implementation pending
