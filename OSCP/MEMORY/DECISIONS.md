# OSCP - Key Decisions

## Scope: Discovery Only

OSCP captures the tree. Agent handles interaction.

| Included | Excluded |
|----------|----------|
| MCP server | HTTP REST |
| Native tree capture | Interaction layer |
| CDP fallback | Unix socket |
| Screenshot fallback | Dashboard |
| Unified element format | Confidence scoring |
| | Human handoff |
| | Recorder |

**Rationale:** Keep OSCP simple. Agent uses Python packages for interaction. Agent provides meaning.

---

## Transport: MCP Only

- CLI agents use MCP
- Single binary, runs as `--mcp`
- No HTTP server
- No Unix socket

**Rationale:** MCP is the standard for CLI agent integration.

---

## No Interaction Layer

OSCP does not:
- Click buttons
- Type text
- Scroll
- Drag and drop

Agent handles all of this via its own tools (Python automation, pyautogui, etc.)

**Rationale:** OSCP stays focused on discovery. Interaction is a separate concern handled by the agent.

---

## No Quality Scoring

OSCP does not:
- Calculate confidence scores
- Recommend actions
- Suggest fallbacks

Agent interprets the tree and decides what to do.

**Rationale:** Agent provides meaning. OSCP just provides the data.

---

## CDP Bridge as Fallback

When native APIs can't see browser content (Safari, Chrome, Electron), OSCP connects to CDP endpoint.

**Covers:** Web content, DOM elements, browser apps

**Rationale:** Native APIs don't expose browser DOM. CDP bridges this gap.

---

## Screenshot as Last Resort

When everything else fails (games, custom renderers, DRM), OSCP returns a screenshot.

Agent can then use VLM externally to analyze.

**Rationale:** Some apps have zero accessibility API. Screenshot is the universal fallback.

---

## Platform Order

1. **macOS** — 3-4 weeks (AXUIElement most stable)
2. **Linux** — 4-5 weeks (AT-SPI2 + X11 fallback)
3. **Windows** — 4-5 weeks (UIAutomation)

**Rationale:** macOS has the most stable, well-documented accessibility API.

---

## Implementation Stack

| Component | Technology |
|-----------|------------|
| MCP Server | Rust or Swift |
| macOS | AXUIElement (C API) |
| Linux | AT-SPI2 (D-Bus) |
| Windows | UIAutomation (COM) |
| CDP Bridge | WebSocket + CDP protocol |

---

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 3-4 weeks |
| Linux | 4-5 weeks |
| Windows | 4-5 weeks |
| **Total** | **11-14 weeks** |

---

## Why Not OculOS?

OculOS is a similar project (also Rust, also accessibility APIs). OSCP differs in:

| Aspect | OculOS | OSCP |
|--------|--------|------|
| Transport | HTTP + MCP | MCP only |
| Scope | Discovery + Interaction | Discovery only |
| Confidence | No | No |
| CDP Bridge | No | Yes |
| Screenshot | Yes | Yes |

OSCP is simpler, focused, and adds CDP bridge for better browser coverage.

---

## Rejected Approaches

### Screenshot + VLM as Primary
- Slow (~1-2s/frame)
- Expensive (API costs)
- Probabilistic (VLM can be wrong)
- Unnecessary for 90% of apps

### HTTP REST as Primary
- Extra dependency for CLI agents
- MCP is the standard for CLI agents
- Keep it simple

### Interaction Layer
- Agent handles via Python packages
- Keeps OSCP focused
- Separation of concerns