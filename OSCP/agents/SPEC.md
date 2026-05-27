# OSCP Agent Integration

**Version:** 0.5.0

---

## Overview

OSCP runs as an MCP server. CLI agents connect via MCP and receive the desktop accessibility tree.

---

## MCP Configuration

### Claude Code / Cursor / Windsurf

Add to MCP config:

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

### Custom Agent

```python
import subprocess
import json

# Run OSCP as subprocess
proc = subprocess.Popen(
    ["oscp", "--mcp"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)

# Send MCP requests
def send_request(method, params=None):
    request = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": method,
        "params": params or {}
    }
    proc.stdin.write(json.dumps(request).encode() + b"\n")
    proc.stdin.flush()
    response = proc.stdout.readline()
    return json.loads(response)
```

---

## Agent Workflow

```
1. Agent → list_windows
   └─► Get all visible windows

2. Agent → get_tree(pid)
   └─► Get element tree for target window

3. Agent → find_elements(pid, query="Save")
   └─► Find specific elements

4. Agent decides action
   └─► Uses its own tools (Python automation, etc.)
```

---

## Example: Claude Code

```
User: Click the Save button in VS Code

Claude:
1. Calls list_windows → finds VS Code (pid: 1234)
2. Calls get_tree(1234) → gets element tree
3. Finds element: {role: "button", name: "Save"}
4. Uses automation tool → clicks at element.bounds
```

---

## Element Reference

```json
{
  "id": "e_001",
  "role": "button",
  "name": "Save",
  "bounds": {"x": 1750, "y": 5, "width": 80, "height": 25},
  "enabled": true
}
```

Agent uses `bounds` to determine where to click/type.

---

## Screenshot Fallback

```json
{
  "source": "screenshot",
  "data": "base64_png..."
}
```

When `source` is `"screenshot"`, agent uses VLM externally to analyze.

---

## Status

- [x] MCP integration defined
- [ ] Implementation pending
