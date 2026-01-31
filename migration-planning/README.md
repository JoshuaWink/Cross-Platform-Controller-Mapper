# Cross-Platform Controller Mapper → CodeUChain Migration

## Executive Summary

This document outlines the migration of the Cross-Platform Controller Mapper from a monolithic `main.py` (~900 lines) to a modular, testable CodeUChain architecture.

## Why CodeUChain?

The current codebase exhibits several patterns that CodeUChain naturally addresses:

| Current Pain Point | CodeUChain Solution |
|-------------------|---------------------|
| 900+ line monolithic file | Single-responsibility Links in separate files |
| Tightly coupled functions | Context-based data flow between Links |
| Difficult to test in isolation | Each Link testable independently |
| Global state (`CURRENT_PROFILE_INDEX`) | Context carries state explicitly |
| Mixed concerns (I/O + logic) | Layered architecture (Entry → Orchestration → Domain → Integration) |
| Hard to extend (adding new actions) | New Links plug into existing Chains |

## The Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           ENTRY POINTS                                  │
│   CLI (argparse) / Future: GUI, API                                     │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATION CHAINS                            │
│   RunChain, CalibrationChain, MappingChain, ProfileChain                │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           DOMAIN LINKS                                  │
│   Config, Events, Actions, Normalization, Validation                    │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         INTEGRATION LINKS                               │
│   Pygame, Pynput (Keyboard/Mouse), File I/O, Notifications             │
└─────────────────────────────────────────────────────────────────────────┘
```

## Biological Analogy

Think of this as a **nervous system**:

| Biological | Controller Mapper |
|------------|-------------------|
| **Sensory neurons** | Pygame event listeners (button presses, axis movements) |
| **Spinal cord** | Event routing chain - fast, reflexive mapping |
| **Brain** | Profile logic, combo detection, action resolution |
| **Motor neurons** | Pynput keyboard/mouse execution |
| **Hormonal feedback** | Notifications, profile state changes |
| **Homeostasis** | Deadzone/calibration keeping inputs stable |

## Key Design Decisions

### 1. Context as the Event Packet

Every controller event becomes a Context that flows through the system:

```python
ctx = Context({
    "event_type": "button_press",
    "button_id": 0,
    "profile_name": "Default",
    "raw_value": 1,
    "timestamp": time.time()
})
```

### 2. Chains as Pipelines

```
Event → [ValidateEvent] → [LookupMapping] → [ResolveAction] → [ExecuteAction] → Result
```

### 3. Profiles as Configuration, Not Code

Profiles remain in JSON. The Chain dynamically reads them instead of hardcoding behavior.

### 4. Actions as Pluggable Links

Each action type (`Key.X`, `Button.left`, `SwapProfile`, `MouseMove`) is a separate Link that can be tested and extended independently.

## Migration Strategy

### Phase 1: Foundation
- Set up CodeUChain project structure
- Create core Links for config loading and pygame initialization
- Establish Context schema

### Phase 2: Event Processing
- Migrate event handling to Links
- Create event routing Chain
- Implement action execution Links

### Phase 3: Features
- Profile management Chain
- Calibration Chain
- Human-friendly mappings Chain

### Phase 4: CLI & Polish
- Migrate argparse to entry Chain
- Add comprehensive tests
- Documentation

## Success Criteria

- [ ] Each Link has a single responsibility
- [ ] No global mutable state (all state in Context)
- [ ] 90%+ test coverage with isolated unit tests
- [ ] Each CLI command maps to a Chain
- [ ] Adding a new action type requires only a new Link file

## Next Steps

1. Review [PATTERN_ANALYSIS.md](PATTERN_ANALYSIS.md) for detailed pattern mapping
2. Review [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) for proposed directory layout
3. Review [COMPONENT_INVENTORY.md](COMPONENT_INVENTORY.md) for Link/Chain specifications
4. Use `pyproject.toml` for dependency management

---

*"The best architecture is the one that makes the right thing easy and the wrong thing hard."*
