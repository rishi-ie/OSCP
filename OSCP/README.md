# OSCP — Operating System Context Protocol

**Version:** 0.4.0
**Status:** Specs Complete. Ready for Implementation.

---

## Vision

> Make agents first-class citizens of operating systems designed for humans.

OSCP enables AI agents to see and interact with desktop applications using native OS accessibility APIs—no screenshots, no VLM required.

---

## What OSCP Does

OSCP wraps existing OS accessibility APIs and exposes them to AI agents through a unified protocol. When an agent needs to interact with a desktop application, OSCP:

- **Captures** the current screen state as a structured semantic tree
- **Reports** element names, roles, positions, and relationships
- **Scores** data quality so agents know when to trust it
- **Executes** mouse and keyboard actions on behalf of the agent
- **Falls back** gracefully when native APIs can't see the UI

---

## Core Capabilities

### 1. On-Demand Screen Capture

Agents request screen state when needed—not continuously streamed. Each capture returns:

| Data | Description |
|------|-------------|
| All visible windows | Titles, bounds, focus state, owning app |
| Every UI element | Buttons, text fields, menus, tabs, etc. |
| Element attributes | Names, descriptions, values, states |
| Element positions | Screen coordinates for every element |
| Confidence scores | Per-element and per-frame quality metrics |
| Mouse position | Current cursor location and hovered element |

### 2. Native OS Integration

OSCP wraps the native accessibility APIs that macOS, Linux, and Windows already expose:

| Platform | Primary API | Fallback | Coverage |
|----------|-------------|----------|----------|
| macOS | AXUIElement | CDP Bridge (Safari/Chrome/Electron) | 95% |
| Linux | AT-SPI2 | X11, CDP Bridge | 90-95% |
| Windows | UIAutomation | Win32 | 85% |

**Supported element types (all platforms):**
- Buttons, checkboxes, radio buttons
- Text fields, text areas, secure fields
- Combo boxes, dropdowns, sliders
- Menus, menu bars, menu items
- Tabs, tab groups, panels
- Lists, list items, tables, table cells
- Links, images, icons
- Windows, dialogs, alerts, sheets
- Groups, toolbars, scroll areas

### 3. Structured Element Tree

Every captured element includes:

| Field | Example |
|-------|---------|
| `id` | `"e_1234_a1b2c3d4"` |
| `role` | `"button"`, `"text_field"`, `"menu_item"` |
| `subrole` | `"push_button"`, `"search_field"` |
| `name` | `"Save"`, `"username"`, `"Cancel"` |
| `description` | `"Save the current file"` |
| `value` | `"hello@world.com"`, `"50%"` |
| `bounds` | `{"x": 1750, "y": 5, "w": 80, "h": 25}` |
| `states` | `["enabled", "visible", "actionable"]` |
| `attributes` | `{"default_button": false}` |
| `confidence` | `0.95` |
| `source` | `"axuielement"`, `"cdp"`, `"heuristic"` |

### 4. Action Execution

Agents can perform all standard desktop interactions:

#### Mouse Actions

| Action | Parameters | Description |
|--------|------------|-------------|
| `click` | x, y, button, click_type | Single, double, triple, right-click |
| `drag` | start_x, start_y, end_x, end_y, duration | Drag with configurable speed |
| `move` | x, y | Move cursor without clicking |
| `scroll` | delta_x, delta_y, scroll_type | Precise, line, or page scrolling |

#### Keyboard Actions

| Action | Parameters | Description |
|--------|------------|-------------|
| `type` | text, typing_delay_ms | Type arbitrary text |
| `key_combo` | key, modifiers | Single key or modifier combination |

**Modifier keys:** `ctrl`, `alt`, `shift`, `cmd`, `fn`

**Common combos:**
- `Ctrl+S` / `Cmd+S` — Save
- `Ctrl+Z` — Undo
- `Ctrl+Shift+T` — Reopen closed tab
- `Alt+Tab` — Switch window
- `Escape` — Cancel/close
- `Tab` / `Shift+Tab` — Navigate controls

### 5. Quality Scoring

OSCP analyzes captured data and reports quality:

#### Per-Frame Analysis

| Metric | Description |
|--------|-------------|
| `coverage_score` | Named area / window area (0-1) |
| `named_elements` | Count of elements with names |
| `unlabeled_elements` | Count without names |
| `total_elements` | Total element count |
| `avg_depth` | Average tree depth |
| `confidence` | HIGH / MEDIUM / LOW / NONE |

#### Confidence Thresholds

| Level | When | Agent Action |
|-------|------|--------------|
| **HIGH** | coverage > 80%, named ratio > 50% | Execute immediately |
| **MEDIUM** | coverage 50-80% | Execute with monitoring |
| **LOW** | coverage 30-50% | Explore candidates first |
| **NONE** | coverage < 30% | Human handoff |

#### Per-Element Confidence

Each element receives a confidence score based on:
- Has accessible name (40%)
- Has actionable role (30%)
- Has reasonable size (20%)
- Uses authoritative source (10%)

### 6. Intelligent Fallbacks

When native APIs can't see the UI, OSCP applies a 5-level fallback chain:

| Level | Method | When Used |
|-------|--------|-----------|
| 1 | Native semantic tree | Standard apps (AppKit, GTK, Qt) |
| 2 | CDP bridge | Safari, Chrome, VS Code, Electron apps |
| 3 | Structural heuristics | Partial tree, position inference |
| 4 | Position-only mode | Custom renderers, games |
| 5 | Human handoff | All methods exhausted |

Each level attempts capture; if quality is insufficient, proceeds to next level.

### 7. Error Handling

When actions fail, OSCP provides actionable error responses:

```json
{
  "type": "action_result",
  "success": false,
  "error": {
    "code": "ACTION_FAILED",
    "message": "Action did not produce expected result",
    "reasoning": "Element may have moved or changed state",
    "alternatives": [
      {"x": 1730, "y": 5, "confidence": 0.3, "element_name": "Save (alt)"},
      {"x": 1750, "y": 7, "confidence": 0.25, "element_name": "Toolbar area"}
    ]
  }
}
```

**Error codes:**
- `PERMISSION_DENIED` — Grant accessibility permission
- `ELEMENT_NOT_FOUND` — Element no longer exists
- `ELEMENT_DISABLED` — Element is disabled
- `ELEMENT_MOVED` — Element position changed
- `ACTION_FAILED` — Action didn't work, try alternatives
- `EMPTY_TREE` — No UI visible, use fallback
- `LOW_COVERAGE` — Poor data quality
- `TIMEOUT` — Operation timed out

### 8. Human Handoff

When fully stuck, OSCP supports graceful escalation:

```json
{
  "type": "handoff_request",
  "reason": "Cannot identify target element",
  "reasoning": "Custom renderer detected",
  "attempts": 3,
  "window": {"id": "win_game", "title": "Game", "bounds": {...}},
  "alternatives": [
    {"bounds": {"x": 1700, "y": 0}, "confidence": 0.3, "strategy": "Toolbar"},
    {"bounds": {"x": 960, "y": 540}, "confidence": 0.2, "strategy": "Center"}
  ],
  "options": [
    "Agent explores alternative positions",
    "Human clicks and agent learns",
    "Human completes this task",
    "Skip this task"
  ]
}
```

### 9. Human-Informed Learning

Agents can learn from human intervention:

```json
{
  "type": "handoff_response",
  "resolution": "learn_from_human",
  "human_click": {"x": 1750, "y": 5}
}
```

The agent records this position for future reference in similar contexts.

---

## Supported Platforms

### macOS

- **API:** AXUIElement (native C API)
- **Language:** Swift / Objective-C
- **Input:** CGEvent (Core Graphics)
- **Coverage:** 95% of standard macOS apps
- **Supported apps:** Finder, Safari, VS Code, Slack, Terminal, Xcode, Mail, Messages, etc.

### Linux

- **Primary API:** AT-SPI2 (D-Bus)
- **Fallback:** X11 (XQueryTree)
- **Language:** Python (pyatspi) or Rust
- **Input:** /dev/uinput (primary), XTest (fallback)
- **Coverage:** 90-95% of standard Linux apps
- **Supported apps:** GNOME apps, KDE apps, Firefox, Chrome, VS Code, Terminal

### Windows

- **Primary API:** UIAutomation
- **Fallback:** Win32 API
- **Status:** Deferred

---

## Protocol

### Connection

- **Transport:** Unix domain socket (`/tmp/oscp.sock`)
- **Format:** JSON over newline-delimited frames
- **Pattern:** Request-response (no streaming)

### Message Types

| Message | Direction | Description |
|---------|-----------|-------------|
| `hello` / `welcome` | Bidirectional | Handshake |
| `get_frame` / `frame` | Bidirectional | Screen capture |
| `action` / `action_result` | Bidirectional | Execute actions |
| `mouse_position` / `mouse_position_result` | Bidirectional | Query cursor |
| `error` | Server → Agent | Error notification |
| `handoff_request` / `handoff_response` | Bidirectional | Human escalation |
| `ping` / `pong` | Bidirectional | Keep-alive |
| `disconnect` / `goodbye` | Bidirectional | Clean disconnect |

### SDKs

**Python:**
```python
client = oscp.connect("/tmp/oscp.sock")
frame = await client.getFrame()
await client.click(x=1750, y=17)
await client.type_text("Hello, World!")
await client.key_combo("s", modifiers=["ctrl"])
```

**Swift:**
```swift
let client = OSCPClient()
let frame = try await client.getFrame()
try await client.click(x: 1750, y: 17)
try await client.typeText("Hello, World!")
try await client.keyCombo("s", modifiers: ["ctrl"])
```

---

## Agent Use Cases

OSCP enables agents to:

1. **Navigate apps** — Click buttons, select menu items, switch tabs
2. **Fill forms** — Type in text fields, check boxes, select options
3. **Read content** — Get visible text, button labels, form values
4. **Drag and drop** — Reorder items, move files
5. **Scroll content** — Page through documents, load more items
6. **Use keyboard shortcuts** — Execute app-native commands
7. **Handle dialogs** — Click OK/Cancel, close alerts
8. **Learn from humans** — Record human interventions for future use
9. **Escalate gracefully** — Request human help when stuck

---

## Implementation Status

| Component | Status | Time |
|-----------|--------|------|
| Protocol specification | ✅ Complete | Done |
| macOS detailed spec | ✅ Complete | Done |
| Linux detailed spec | ✅ Complete | Done |
| macOS implementation | ⏳ Pending | 4-5 weeks |
| Linux implementation | ⏳ Pending | 6-7 weeks |
| Windows implementation | ⏸️ Deferred | TBD |

**Total: 10-12 weeks for macOS + Linux**

---

## Directory Structure

```
OSCP/
├── README.md                    # This file
├── SPEC.md                     # Architecture overview
├── protocol/
│   └── SPEC.md                 # Protocol specification + SDKs
├── platforms/
│   ├── macos/SPEC.md           # macOS implementation spec
│   ├── linux/SPEC.md            # Linux implementation spec
│   └── windows/SPEC.md          # Deferred
├── agents/SPEC.md              # Agent integration guidelines
└── MEMORY/                     # Project context
```

---

## References

- **GitHub:** github.com/rishi-ie/OSCP
- **AXUIElement:** developer.apple.com/documentation/application_services
- **AT-SPI2:** https://atspi.readthedocs.io
- **UIAutomation:** docs.microsoft.com/en-us/windows/win32/winauto/

---

*OSCP — Desktop awareness for AI agents, without screenshots.*
