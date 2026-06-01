# OSCP — Operating System Context Protocol

**Give CLI agents eyes.**

---

## Overview

OSCP is an open-source protocol and implementation that enables AI agents to see and interact with any desktop application through native OS accessibility APIs.

Unlike screen-capture approaches, OSCP reads the actual UI element tree—the same tree that accessibility tools and screen readers use. This makes it fast, deterministic, and accurate.

---

## How It Works

OSCP runs as a local MCP server. When an agent needs to see the desktop, it sends a request and OSCP returns a structured tree of every visible element: buttons, text fields, menus, tabs, and more.

The agent interprets the tree and decides what to do. OSCP stays focused on one thing: providing an accurate, up-to-date view of the desktop.

---

## Key Features

### Native API Integration

OSCP reads directly from the OS accessibility layer on each platform. No screenshots. No pixel analysis. Just structured data from the same APIs that assistive technologies use.

| Platform | Technology | Coverage |
|----------|------------|----------|
| macOS | AXUIElement | 90% |
| Linux | AT-SPI2 | 85% |
| Windows | UIAutomation | 90% |

### Progressive Fallback

OSCP tries the most accurate source first, then falls back as needed:

1. **Native accessibility tree** — Full element hierarchy for standard applications
2. **CDP DOM bridge** — Browser and Electron applications
3. **Screenshot** — Games and custom renderers that lack accessibility APIs

### Unified Element Model

Every UI element is returned in a consistent format regardless of platform:

- Element type and name
- Screen position and dimensions
- Current state (enabled, focused, selected)
- Nested hierarchy

### CLI Agent Integration

OSCP implements the Model Context Protocol, the standard for CLI agent tool integration. It works with Claude Code, Cursor, Windsurf, and any other MCP-compatible agent.

---

## What OSCP Returns

For each visible application, OSCP provides:

- **Window information** — Title, position, dimensions, process ID
- **Element tree** — Every interactive and informational element
- **Element attributes** — Type, name, value, state, position
- **Source indicator** — Where the data came from (native API, browser DOM, screenshot)

---

## Application Compatibility

| Application Type | Coverage | Notes |
|-----------------|----------|-------|
| Native desktop apps | Excellent | Full tree, all interactions |
| Web browsers | Partial | Requires CDP bridge |
| Electron apps | Good | Variable depending on implementation |
| Custom renderers | None | Falls back to screenshot |

---

## Protocol Design

OSCP is both a specification and an implementation. The protocol defines:

- **Discovery** — List windows, get element trees, search by name or type
- **Element schema** — Unified format for all platform elements
- **Fallback behavior** — How and when to use alternative data sources

The implementation is a single binary that runs locally. No cloud services. No API keys. All processing happens on the local machine.

---

## Getting Started

Add OSCP to your MCP configuration:

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

---

## Status

OSCP is currently in the specification phase. Implementation will follow.

| Component | Status |
|-----------|--------|
| Protocol specification | Complete |
| macOS specification | Complete |
| Linux specification | Complete |
| Windows specification | Complete |
| Implementation | Pending |

---

## Project Structure

```
OSCP/
├── README.md
├── SPEC.md
├── protocol/           # Protocol specification
├── platforms/          # Platform implementations
│   ├── macos/
│   ├── linux/
│   └── windows/
└── agents/            # Agent integration guide
```

---

## Resources

- **Repository:** github.com/rishi-ie/OSCP
- **Protocol:** Model Context Protocol (modelcontextprotocol.io)

---

## License

MIT
