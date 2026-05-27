# OSCP Architecture

## Overview

OSCP = MCP Server + Tree Capture + CDP Bridge + Screenshot Fallback

## Components

### MCP Server
- CLI agent integration
- Runs as `--mcp` binary
- No HTTP, no socket

### Discovery Layer
- Native accessibility (AXUIElement/AT-SPI2/UIA)
- CDP DOM bridge (browsers)
- Screenshot fallback (games)

### Element Registry
- UUID-based element IDs
- Unified element format

## Capture Pipeline

```
1. Native API (90%)
2. CDP DOM Bridge (browsers)
3. Screenshot (games)
```

## Spec Status

| Spec | Status |
|------|--------|
| Protocol | ✅ Complete |
| macOS | ✅ Complete |
| Linux | ✅ Complete |
| Windows | ✅ Complete |

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 3-4 weeks |
| Linux | 4-5 weeks |
| Windows | 4-5 weeks |
| Total | 11-14 weeks |