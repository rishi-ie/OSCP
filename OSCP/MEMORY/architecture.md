# OSCP Architecture

## Overview

OSCP is a wrapper + on-demand layer on top of existing OS accessibility APIs.

## Components

### Protocol Layer
- Unix socket server (macOS/Linux)
- JSON over newline-delimited messages
- Request-response pattern (no streaming)

### Capture Layer
- Platform-specific accessibility API wrapper
- macOS: AXUIElement
- Linux: AT-SPI2 + X11 fallback

### Analysis Layer
- Tree builder (flatten, search, lookup)
- Tree analyzer (coverage, confidence)
- Fallback manager (5-level chain)

### Input Layer
- Platform-specific input injection
- macOS: CGEvent
- Linux: /dev/uinput + XTest

### Bridge Layer (Fallback)
- CDP bridge for browsers
- Safari, Chrome, Electron apps

## Spec Status

| Spec | Status |
|------|--------|
| Protocol | ✅ Complete |
| macOS | ✅ Complete |
| Linux | ✅ Complete |
| Windows | ⏸️ Deferred |

## Time Estimates

| Platform | Time |
|----------|------|
| macOS | 4-5 weeks |
| Linux | 6-7 weeks |
| Total | 10-12 weeks |