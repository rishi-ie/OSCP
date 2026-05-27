# OSCP - Key Decisions

## Scope: Discovery Only

OSCP captures the tree. Agent handles interaction.

| Included | Excluded |
|----------|----------|
| MCP server | HTTP REST |
| Native tree capture | Interaction layer |
| CDP fallback | Unix socket |
| Screenshot fallback | Dashboard |

## Transport: MCP Only

CLI agents use MCP. Single binary, `--mcp` flag.

## No Interaction Layer

Agent uses Python/macOS automation packages for clicks/keyboard.

## No Quality Scoring

Agent provides meaning. That's the point.

## No Human Handoff

Agent handles edge cases via its own logic.

## CDP Bridge

For browsers and Electron apps when native API fails.

## Screenshot Fallback

For games and custom renderers when everything else fails.

## Platform Order

macOS first (AXUIElement most stable), then Linux, then Windows.

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 3-4 weeks |
| Linux | 4-5 weeks |
| Windows | 4-5 weeks |
| **Total** | **11-14 weeks** |

## Rejected Approaches

### Screenshot + VLM as Primary
- Slow (~1-2s/frame)
- Expensive
- Probabilistic

### Interaction Layer
- Agent handles via Python packages
- Keep OSCP simple
