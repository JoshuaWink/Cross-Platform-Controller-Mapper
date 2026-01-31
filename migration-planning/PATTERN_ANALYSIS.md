# Pattern Analysis: Cross-Platform Controller Mapper

## Current Code Analysis

### Identified Responsibilities (from `main.py`)

| Function Group | Responsibility | Lines | Coupling |
|----------------|---------------|-------|----------|
| `load_json`, `save_json`, `save_config` | File I/O | ~30 | Low |
| `normalize_*`, `apply_deadzone`, `calculate_axis_value` | Value Transformation | ~60 | Low |
| `initialize_pygame`, `initialize_controller` | Hardware Init | ~40 | Medium |
| `handle_controller_events`, `listen_for_controller_input` | Event Processing | ~80 | High |
| `execute_action`, action_map lambdas | Action Execution | ~150 | High |
| `execute_profile_actions`, `get_button_action`, `get_axis_action` | Profile Resolution | ~100 | High |
| `swap_to_next_profile`, `apply_new_mappings` | Profile Management | ~80 | High |
| `calibrate_axes`, `set_deadzone` | Calibration | ~60 | Medium |
| CLI functions (`run`, `list`, `add`, `remove`, `map`) | Entry Points | ~200 | Medium |

### Code Smells Detected

1. **God Object**: `main.py` does everything
2. **Global State**: `CURRENT_PROFILE_INDEX`, `controller`, `is_inputs_paused`
3. **Nested Function Definitions**: `parse_and_execute_combination` inside `execute_action`
4. **Mixed Abstraction Levels**: Low-level pygame calls next to business logic
5. **Copy-Paste Code**: Multiple normalize functions with slight variations
6. **TODO Comments**: Unfinished combo logic scattered throughout

---

## Pattern Mapping

### 1. Pipeline Pattern (Primary)

**Where It Applies**: The entire event processing flow

```
Current:
  listen_for_controller_input() → handle_controller_events() → execute_profile_actions() → execute_action()

CodeUChain:
  [ListenLink] → [ParseEventLink] → [LookupMappingLink] → [ResolveActionLink] → [ExecuteActionLink]
```

**Benefit**: Each step becomes independently testable

---

### 2. Strategy Pattern → Interchangeable Links

**Where It Applies**: Action execution

```python
# Current: Giant if/elif/else chain in execute_action
if action.startswith("Key."):
    # keyboard logic
elif action.startswith("Button."):
    # mouse logic
elif action == "SwapProfile":
    # profile logic
# ... etc

# CodeUChain: Action resolver returns the right Link
action_links = {
    "keyboard": KeyboardActionLink(),
    "mouse": MouseActionLink(),
    "profile": ProfileActionLink(),
    "scroll": ScrollActionLink(),
    "combo": ComboActionLink(),
}
```

**Benefit**: Adding new action types = adding new Links without modifying existing code

---

### 3. Chain of Responsibility → Sequential Links with Short-Circuit

**Where It Applies**: Event validation and routing

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ ValidateLink│───▶│ RouteLink   │───▶│ ProcessLink │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │
       ▼                  ▼
   (if invalid)      (if unknown)
   [ErrorLink]       [LogUnknownLink]
```

**Benefit**: Invalid events rejected early, unknown events logged for debugging

---

### 4. Observer Pattern → Context Event Emission

**Where It Applies**: Profile switches, notifications

```python
# Current: Direct call to Notifier
Notifier.notify(f"Switched to profile: {profile_name}")

# CodeUChain: Context carries notification intent
ctx.set("notification", {
    "title": "Profile Switch",
    "message": f"Switched to profile: {profile_name}"
})
# NotificationLink picks it up later in the Chain
```

**Benefit**: Decouples business logic from notification implementation

---

### 5. State Machine → Profile Context

**Where It Applies**: Profile and input state management

```
States:
  - IDLE (waiting for input)
  - PROCESSING (handling event)
  - PAUSED (inputs disabled)
  - CALIBRATING (calibration mode)

Transitions:
  IDLE + button_press → PROCESSING
  PROCESSING + action_complete → IDLE
  IDLE/PROCESSING + pause_toggle → PAUSED
  PAUSED + pause_toggle → IDLE
```

**CodeUChain Expression**: State lives in Context, Chains check state before processing

---

### 6. Adapter Pattern → Integration Links

**Where It Applies**: External library interfaces

```
┌──────────────────┐
│ PygameEventLink  │  ← Adapts pygame.event.get() to Context
└──────────────────┘

┌──────────────────┐
│ PynputActionLink │  ← Adapts pynput keyboard/mouse to Context
└──────────────────┘

┌──────────────────┐
│ NotificationLink │  ← Adapts pync.Notifier to Context
└──────────────────┘
```

**Benefit**: External dependencies isolated, mockable in tests

---

### 7. Factory Pattern → Link Constructors

**Where It Applies**: Creating action-specific handlers

```python
def create_action_link(action_type: str) -> Link:
    """Factory for action Links based on action string."""
    if action_type.startswith("Key."):
        return KeyboardActionLink(key=action_type.split(".")[1])
    elif action_type.startswith("Button."):
        return MouseButtonLink(button=action_type.split(".")[1])
    elif "+" in action_type:
        return ComboActionLink(combo_string=action_type)
    else:
        return CustomActionLink(action_name=action_type)
```

---

### 8. Template Method → Abstract Link Base

**Where It Applies**: All action Links share common structure

```python
class ActionLink(Link):
    """Base class for all action Links."""
    
    def execute(self, ctx: Context) -> Context:
        if not self.validate(ctx):
            return ctx
        
        value = ctx.get("action_value")
        if value:  # Press/activate
            self.activate(ctx)
        else:  # Release/deactivate
            self.deactivate(ctx)
        
        return ctx
    
    def validate(self, ctx: Context) -> bool:
        """Override in subclasses."""
        return True
    
    def activate(self, ctx: Context):
        """Override in subclasses."""
        raise NotImplementedError
    
    def deactivate(self, ctx: Context):
        """Override in subclasses."""
        raise NotImplementedError
```

---

## Data Flow Analysis

### Current Data Flow

```
Global: config, layout, mappings, controller, CURRENT_PROFILE_INDEX

preload() → config, layout, mappings, controller
    │
    ▼
run() → check_controller()
    │
    ▼
listen_for_controller_input() ← pg.event.get()
    │
    ├─ data['events']
    ├─ data['pressed_buttons']
    ├─ data['released_buttons']
    ├─ data['axis_values']
    └─ data['held_buttons']
    │
    ▼
execute_profile_actions() ← profile, data
    │
    ▼
execute_action() ← action, value, config
```

### Proposed Context Flow

```python
# Initial Context (from CLI)
ctx = Context({
    "command": "run",
    "config_path": "config.json",
    "layout_path": "layout__ps4.json"
})

# After LoadConfigLink
ctx = Context({
    ...previous,
    "config": { ... },
    "layout": { ... },
    "current_profile_index": 0
})

# After InitPygameLink
ctx = Context({
    ...previous,
    "controller": <pygame.joystick.Joystick>
})

# Event Loop Context (per-event)
event_ctx = Context({
    "event_type": "JOYBUTTONDOWN",
    "button": 0,
    "value": 1,
    "profile": "Default",
    "mapping": {
        "name": "X",
        "action": "Key.alt"
    },
    "resolved_action": {
        "type": "keyboard",
        "key": "alt",
        "press": True
    }
})
```

---

## Transformation Rules

| Current | CodeUChain |
|---------|------------|
| `load_json(path)` | `LoadJsonLink(path_key="config_path")` |
| `save_json(data, path)` | `SaveJsonLink(data_key="config", path_key="config_path")` |
| `initialize_pygame()` | `InitPygameLink()` |
| `normalize_joystick_value(v, min, max)` | `NormalizeAxisLink(normalization_type="joystick")` |
| `execute_action(action, value, config)` | `ActionChain` with strategy Links |
| `swap_to_next_profile()` | `SwapProfileLink()` |
| `handle_controller_events(data, config)` | `EventProcessingChain` |

---

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Performance overhead | Profile hot paths, optimize critical Links if needed |
| Learning curve | Comprehensive documentation, examples |
| Migration time | Incremental migration, maintain working state |
| Feature parity | Test-driven migration ensures nothing lost |

---

## Recommendations

1. **Start with Config Links** - Low risk, high value, establishes pattern
2. **Extract Action Links Next** - Most tangled code, biggest improvement  
3. **Profile Chain Third** - Enables parallel development
4. **Event Loop Last** - Most complex, needs foundation in place
5. **Write Tests First** - TDD ensures parity with current behavior

---

*"Every function in main.py has a clear destiny as either a Link or a Chain."*
