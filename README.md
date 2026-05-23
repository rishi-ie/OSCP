# OSCP — Operating System Context Protocol

## Vision

OSCP exists to make agents **first-class users of computing systems**, alongside humans.

Today's software ecosystem is designed around a single assumption:

> Humans interact with computers through graphical interfaces.

Agents currently imitate humans by observing screens, moving cursors, pressing keys, and reacting to pixels. This approach introduces fragility, hidden state, and unreliable behavior over long workflows.

OSCP aims to replace GUI imitation with a native semantic execution layer.

Instead of interacting with computers through:

- pixels
- coordinates
- screenshots
- visual guessing
- UI-specific hacks

agents interact through:

- structured state
- deterministic actions
- event streams
- semantic context

The long-term vision is:

> Everything a human can do digitally should be accessible to agents with equal or greater reliability.

OSCP does not seek to replace operating systems.

OSCP introduces a new abstraction:

> Humans and agents become equal participants in the computing environment.

---

## Objectives

### Agent-native interaction

Create an interaction model where agents no longer need to pretend to be humans.

Agents should communicate with systems through a structured protocol rather than visual simulation.

### Deterministic execution

Separate intelligence from execution.

Reasoning may remain probabilistic, but execution should be deterministic and verifiable.

The system should guarantee:

- clear outcomes
- predictable behavior
- observable state transitions
- explicit failures

### Atomic operations

Reduce all system interactions into the smallest practical semantic actions.

Actions should:

- represent a single intent
- produce a single side effect
- be independently verifiable
- remain composable into larger workflows

### Semantic system understanding

Expose machine state in structured form rather than raw visual representation.

Agents should understand:

- what exists
- what changed
- what can be acted upon
- what actions are available

without depending on screenshots or visual interpretation.

### Application-agnostic runtime

Keep the execution runtime generic.

OSCP should not embed application-specific knowledge.

Application understanding should exist separately through skills, adapters, or extensions.

### Event-driven operation

Move away from continuous polling and reactive GUI observation.

The system should expose changes as structured events so agents can operate with awareness of state transitions in real time.

### Reliability over convenience

Prioritize robustness over shortcuts.

The architecture should optimize for:

- repeatability
- fault tolerance
- recoverability
- observability
- predictable behavior

rather than minimizing implementation complexity.

---

## Goals

### Near-term goals

- Build an operating-system execution protocol for agents
- Expose deterministic semantic actions
- Provide structured state and event streams
- Support local on-device agents
- Enable reliable cross-application workflows
- Remove dependence on visual interaction wherever possible

### Mid-term goals

- Create a reusable skill ecosystem
- Standardize agent interaction patterns
- Support complex autonomous workflows
- Enable interoperability between different agent systems

### Long-term goals

- Establish agents as first-class entities within computing environments
- Create a universal execution substrate for digital tasks
- Enable agents to operate with greater consistency than humans across software systems
- Shift digital interaction from GUI-centric workflows to semantic workflows

---

## Core Principle

OSCP does not attempt to make intelligence deterministic.

OSCP attempts to make **execution deterministic**.

In simple terms:

> Humans interact through interfaces.
> Agents interact through meaning.

---

## Protocol Overview

```
┌─────────────────────────────────────────────────────┐
│                    OSCP Protocol                     │
├─────────────────────────────────────────────────────┤
│  Layer 1: Context     — Hierarchical OS state        │
│  Layer 2: Input       — Structured input injection   │
│  Layer 3: Events      — Real-time system events     │
│  Layer 4: Permissions — Capability-based access      │
└─────────────────────────────────────────────────────┘
```

---

## Status

🚧 **Early development** — protocol design in progress.

---

*OSCP — Operating System Context Protocol.*