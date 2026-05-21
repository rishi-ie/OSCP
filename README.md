# Conduit

> An agent-facing layer on top of Linux — giving AI systems full, privileged GUI control.

<p align="center">
  <img src="https://img.shields.io/badge/Status-Building-FF6B35?style=flat-square" />
  <img src="https://img.shields.io/badge/License-MIT-42B883?style=flat-square" />
  <img src="https://img.shields.io/badge/Platform-Linux- FCC624?style=flat-square&logo=linux&logoColor=black" />
</p>

---

## The Problem

Operating systems — Linux included — were designed for human operators. AI agents need a different interface entirely.

Current approach: agents pseudo-lives in a world built for fingers and eyeballs. They see pixels, not semantics. They send keystrokes through fragile OCR pipelines. They navigate desktops like blind users with a keyboard.

This is not sustainable.

---

## The Vision

**Conduit** adds an agent-native layer between the Linux kernel and the desktop environment. Not a wrapper. Not a hack. A first-class interface for autonomous systems to perceive and interact with the GUI — the way the OS intended humans to.

```
┌─────────────────────────────────────────┐
│         AI Agents (pi, OpenClaw, etc)   │
├─────────────────────────────────────────┤
│              CONDUIT                     │
│   ┌─────────┬──────────┬──────────┐    │
│   │ Visual  │   Input  │  Window  │    │
│   │ Context │  Injection│  State   │    │
│   └─────────┴──────────┴──────────┘    │
├─────────────────────────────────────────┤
│       Linux Kernel / X11 / Wayland       │
├─────────────────────────────────────────┤
│              Hardware                   │
└─────────────────────────────────────────┘
```

Agents get:
- **Canonical screen context** — structured UI hierarchy, not raw pixels
- **Privileged input** — keyboard/mouse at the kernel level, not through OCR
- **Window semantics** — focus, ownership, lifecycle, z-order
- **Permission model** — explicit grant/revoke for desktop access

---

## Architecture

```
conduit/
├── kernel-module/       # Linux kernel module (evdev, uinput)
├── daemon/              # Userspace daemon (IPC, agent communication)
├── agent-sdk/           # SDK for agent integration
├── examples/            # Example implementations
└── docs/                # Design docs & specs
```

### Core Components

| Component | Responsibility |
|-----------|---------------|
| **kernel-module** | Input injection, screen capture primitives |
| **daemon** | Agent IPC, permission enforcement, state management |
| **agent-sdk** | Client library for agent frameworks |

---

## Status

🚧 **Early development** — architecture being defined, first components in progress.

This is a foundational infrastructure project. The goal is to establish Conduit as the standard way AI agents interact with Linux desktops.

---

## Why Linux First

Linux is the natural starting point — open, modular, and already the substrate for most server-side AI infrastructure. Once Conduit is stable on Linux, the patterns extend.

> Mac and Windows weren't designed with the vision that someday agents would use them. We are adding that layer.

---

## Contributing

This project is in active development. Architecture discussions, PRs, and ideas welcome.

---

## Philosophy

The future of computing is not humans clicking buttons. It's autonomous systems perceiving and acting with the same depth humans have — but at machine speed and scale.

**Conduit is the bridge.**

---

*Conduit. An agent-facing layer on top of Linux.*