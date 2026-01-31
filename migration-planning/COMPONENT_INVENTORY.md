# Component Inventory: Links & Chains

## Links Inventory

### Configuration Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `LoadConfigLink` | `links/config/load_config.py` | Load config.json into Context | `config_path` | `config` |
| `SaveConfigLink` | `links/config/save_config.py` | Save Context config to file | `config`, `config_path` | (file write) |
| `LoadLayoutLink` | `links/config/load_layout.py` | Load controller layout JSON | `layout_path` | `layout` |
| `LoadMappingsLink` | `links/config/load_mappings.py` | Load human-friendly mappings | `mappings_path` | `mappings` |

**Example Implementation**:
```python
# links/config/load_config.py
from codeuchain import Link, Context

class LoadConfigLink(Link):
    def __init__(self):
        super().__init__("load_config")
    
    def execute(self, ctx: Context) -> Context:
        import json
        path = ctx.get("config_path", "config.json")
        
        try:
            with open(path, 'r') as f:
                config = json.load(f)
            ctx.set("config", config)
            ctx.set("current_profile_index", config.get("current_profile_index", 0))
        except FileNotFoundError:
            ctx.error = Exception(f"Config file not found: {path}")
        except json.JSONDecodeError as e:
            ctx.error = Exception(f"Invalid JSON in config: {e}")
        
        return ctx
```

---

### Normalization Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `NormalizeJoystickLink` | `links/normalization/normalize_joystick.py` | Normalize joystick to [-1, 1] | `raw_value`, `axis_min`, `axis_max` | `normalized_value` |
| `NormalizeTriggerLink` | `links/normalization/normalize_trigger.py` | Normalize trigger to [0, 1] | `raw_value`, `axis_min`, `axis_max` | `normalized_value` |
| `ApplyDeadzoneLink` | `links/normalization/apply_deadzone.py` | Apply deadzone threshold | `normalized_value`, `deadzone` | `action_value` |
| `RestrictValueLink` | `links/normalization/restrict_value.py` | Clamp value to valid range | `value`, `min`, `max` | `restricted_value` |

**Example Implementation**:
```python
# links/normalization/apply_deadzone.py
from codeuchain import Link, Context

class ApplyDeadzoneLink(Link):
    def __init__(self):
        super().__init__("apply_deadzone")
    
    def execute(self, ctx: Context) -> Context:
        value = ctx.get("normalized_value", 0.0)
        deadzone = ctx.get("deadzone", 0.1)
        
        if abs(value) <= deadzone:
            ctx.set("action_value", 0.0)
        else:
            ctx.set("action_value", value)
        
        return ctx
```

---

### Event Processing Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `ParseEventLink` | `links/events/parse_event.py` | Parse pygame event to Context | `pygame_event` | `event_type`, `button_id`/`axis_id`, `raw_value` |
| `RouteEventLink` | `links/events/route_event.py` | Determine which chain to call | `event_type` | `route` |
| `LookupMappingLink` | `links/events/lookup_mapping.py` | Find mapping for input | `button_id`/`axis_id`, `current_profile` | `mapping`, `action_string` |
| `ValidateEventLink` | `links/events/validate_event.py` | Validate event is processable | `event_type`, `is_paused` | (pass/fail) |

**Example Implementation**:
```python
# links/events/parse_event.py
from codeuchain import Link, Context
import pygame as pg

class ParseEventLink(Link):
    def __init__(self):
        super().__init__("parse_event")
    
    def execute(self, ctx: Context) -> Context:
        event = ctx.get("pygame_event")
        
        if event.type == pg.JOYBUTTONDOWN:
            ctx.set("event_type", "button_press")
            ctx.set("button_id", event.button)
            ctx.set("raw_value", 1)
        
        elif event.type == pg.JOYBUTTONUP:
            ctx.set("event_type", "button_release")
            ctx.set("button_id", event.button)
            ctx.set("raw_value", 0)
        
        elif event.type == pg.JOYAXISMOTION:
            ctx.set("event_type", "axis_motion")
            ctx.set("axis_id", event.axis)
            ctx.set("raw_value", event.value)
        
        elif event.type == pg.JOYHATMOTION:
            ctx.set("event_type", "hat_motion")
            ctx.set("raw_value", event.value)
        
        else:
            ctx.set("event_type", "unknown")
        
        return ctx
```

---

### Action Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `KeyboardActionLink` | `links/actions/keyboard.py` | Press/release keyboard key | `key_name`, `action_value` | (keyboard action) |
| `MouseButtonLink` | `links/actions/mouse_button.py` | Press/release mouse button | `button_name`, `action_value` | (mouse action) |
| `MouseMoveLink` | `links/actions/mouse_move.py` | Move mouse cursor | `dx`, `dy`, `speed` | (mouse action) |
| `MouseScrollLink` | `links/actions/mouse_scroll.py` | Scroll mouse wheel | `scroll_direction`, `action_value` | (mouse action) |
| `ComboActionLink` | `links/actions/combo.py` | Execute key combination | `combo_string` | (keyboard action) |
| `ArrowKeysLink` | `links/actions/arrow_keys.py` | Handle arrow key axis mapping | `axis_value`, `direction` | (keyboard action) |
| `CustomActionLink` | `links/actions/custom.py` | Execute named custom actions | `action_name` | varies |

**Example Implementation**:
```python
# links/actions/keyboard.py
from codeuchain import Link, Context
from pynput.keyboard import Key, Controller

class KeyboardActionLink(Link):
    def __init__(self):
        super().__init__("keyboard_action")
        self.keyboard = Controller()
    
    def execute(self, ctx: Context) -> Context:
        key_name = ctx.get("key_name")
        action_value = ctx.get("action_value")
        
        try:
            # Handle special keys (Key.alt, Key.ctrl, etc.)
            key = getattr(Key, key_name, key_name)
            
            if action_value:  # Press
                self.keyboard.press(key)
            else:  # Release
                self.keyboard.release(key)
        
        except Exception as e:
            ctx.error = Exception(f"Keyboard action failed: {e}")
        
        return ctx
```

---

### Profile Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `GetCurrentProfileLink` | `links/profile/get_current.py` | Get active profile | `config`, `current_profile_index` | `current_profile` |
| `SwapProfileLink` | `links/profile/swap_profile.py` | Switch to next profile | `config`, `current_profile_index` | `current_profile_index`, `notification` |
| `SetMappingLink` | `links/profile/set_mapping.py` | Update a mapping | `current_profile`, `input_type`, `input_id`, `action` | `current_profile` |
| `ExportHumanFriendlyLink` | `links/profile/export_human.py` | Export readable mappings | `config`, `layout` | `human_mappings` |

**Example Implementation**:
```python
# links/profile/swap_profile.py
from codeuchain import Link, Context

class SwapProfileLink(Link):
    def __init__(self):
        super().__init__("swap_profile")
    
    def execute(self, ctx: Context) -> Context:
        config = ctx.get("config")
        profiles = config.get("profiles", [])
        current_index = ctx.get("current_profile_index", 0)
        
        if not profiles:
            ctx.error = Exception("No profiles available")
            return ctx
        
        next_index = (current_index + 1) % len(profiles)
        ctx.set("current_profile_index", next_index)
        ctx.set("current_profile", profiles[next_index])
        
        # Queue notification
        ctx.set("notification", {
            "title": "Profile Switch",
            "message": f"Switched to: {profiles[next_index]['name']}"
        })
        
        return ctx
```

---

### Calibration Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `CalibrateAxisLink` | `links/calibration/calibrate_axis.py` | Record min/max for axis | `axis_id`, `controller` | `axis_min`, `axis_max` |
| `SetDeadzoneLink` | `links/calibration/set_deadzone.py` | Set deadzone value | `axis_id`, `deadzone_value` | (config update) |
| `GetAxisValueLink` | `links/calibration/get_axis_value.py` | Read current axis value | `axis_id`, `controller` | `raw_value` |

---

### Integration Links

| Link | File | Purpose | Input Keys | Output Keys |
|------|------|---------|------------|-------------|
| `InitPygameLink` | `integration/pygame_adapter.py` | Initialize pygame/joystick | - | `controller`, `controller_name` |
| `GetEventsLink` | `integration/pygame_adapter.py` | Get pygame events | `controller` | `pygame_events` |
| `NotificationLink` | `integration/notification_adapter.py` | Send system notification | `notification` | (notification sent) |
| `FileReadLink` | `integration/file_adapter.py` | Read file contents | `file_path` | `file_content` |
| `FileWriteLink` | `integration/file_adapter.py` | Write file contents | `file_path`, `content` | (file written) |
| `ScriptExecutionLink` | `integration/script_runner.py` | Execute external script | `script_path`, `args` | `script_output` |

---

## Chains Inventory

### PreloadChain

**File**: `chains/preload.py`

**Purpose**: Load all configuration and initialize hardware

**Composition**:
```python
class PreloadChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(LoadConfigLink())
        self.add_link(LoadLayoutLink())
        self.add_link(LoadMappingsLink())
        self.add_link(InitPygameLink())
        self.add_link(GetCurrentProfileLink())
```

**Input Context**:
```python
{
    "config_path": "config.json",
    "layout_path": "layout__ps4.json",
    "mappings_path": "mappings.json"
}
```

**Output Context**:
```python
{
    ...input,
    "config": {...},
    "layout": {...},
    "mappings": {...},
    "controller": <Joystick>,
    "current_profile_index": 0,
    "current_profile": {...}
}
```

---

### EventProcessingChain

**File**: `chains/event_processing.py`

**Purpose**: Process a single pygame event through to action resolution

**Composition**:
```python
class EventProcessingChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(ParseEventLink())
        self.add_link(ValidateEventLink())
        self.add_link(LookupMappingLink())
```

**Flow**:
```
pygame_event → [Parse] → [Validate] → [Lookup] → mapping + action_string
```

---

### ActionExecutionChain

**File**: `chains/action_execution.py`

**Purpose**: Execute resolved action

**Composition**:
```python
class ActionExecutionChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(ResolveActionTypeLink())
        self.add_link(ActionRouterLink())  # Routes to appropriate action Link
```

**Note**: Uses conditional routing based on `action_type`

---

### NormalizationChain

**File**: `chains/normalization.py`

**Purpose**: Normalize axis values through calibration pipeline

**Composition**:
```python
class NormalizationChain(Chain):
    def __init__(self, axis_type="joystick"):
        super().__init__()
        if axis_type == "trigger":
            self.add_link(NormalizeTriggerLink())
        else:
            self.add_link(NormalizeJoystickLink())
        self.add_link(ApplyDeadzoneLink())
```

---

### RunChain

**File**: `chains/run.py`

**Purpose**: Main event loop orchestration

**Composition**:
```python
class RunChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(PreloadChainLink())  # Run PreloadChain as a Link
        self.add_link(EventLoopLink())     # Contains the while True loop
```

**Notes**: 
- `EventLoopLink` internally calls `EventProcessingChain` and `ActionExecutionChain` per event
- Handles KeyboardInterrupt gracefully

---

### ProfileManagementChain

**File**: `chains/profile_management.py`

**Purpose**: Profile CRUD operations

**Composition**:
```python
class ProfileManagementChain(Chain):
    def __init__(self, operation="swap"):
        super().__init__()
        if operation == "swap":
            self.add_link(SwapProfileLink())
            self.add_link(SaveConfigLink())
            self.add_link(NotificationLink())
        elif operation == "set_mapping":
            self.add_link(SetMappingLink())
            self.add_link(SaveConfigLink())
```

---

### CalibrationChain

**File**: `chains/calibration.py`

**Purpose**: Interactive axis calibration

**Composition**:
```python
class CalibrationChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(LoadConfigLink())
        self.add_link(InitPygameLink())
        self.add_link(CalibrateAllAxesLink())  # Loops through axes
        self.add_link(SaveConfigLink())
```

---

### MappingChain

**File**: `chains/mapping.py`

**Purpose**: Interactive button/axis mapping

**Composition**:
```python
class MappingChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(LoadConfigLink())
        self.add_link(InitPygameLink())
        self.add_link(InteractiveMappingLink())  # User interaction loop
        self.add_link(SaveConfigLink())
```

---

### ExportChain

**File**: `chains/export.py`

**Purpose**: Export human-friendly mappings

**Composition**:
```python
class ExportChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(LoadConfigLink())
        self.add_link(LoadLayoutLink())
        self.add_link(ExportHumanFriendlyLink())
        self.add_link(SaveMappingsLink())
```

---

## Task Breakdown for Implementation

### Layer 1: Foundation (No Dependencies)

| Task ID | Component | Description | Test File |
|---------|-----------|-------------|-----------|
| L1-1 | `LoadConfigLink` | Load JSON config | `test_config_links.py` |
| L1-2 | `SaveConfigLink` | Save JSON config | `test_config_links.py` |
| L1-3 | `LoadLayoutLink` | Load layout JSON | `test_config_links.py` |
| L1-4 | `NormalizeJoystickLink` | Normalize [-1, 1] | `test_normalization_links.py` |
| L1-5 | `NormalizeTriggerLink` | Normalize [0, 1] | `test_normalization_links.py` |
| L1-6 | `ApplyDeadzoneLink` | Apply deadzone | `test_normalization_links.py` |

### Layer 2: Integration (Blocks on L1)

| Task ID | Component | Description | Test File |
|---------|-----------|-------------|-----------|
| L2-1 | `InitPygameLink` | Init pygame/joystick | `test_integration.py` |
| L2-2 | `GetEventsLink` | Get pygame events | `test_integration.py` |
| L2-3 | `NotificationLink` | Send notifications | `test_integration.py` |
| L2-4 | `ParseEventLink` | Parse pygame event | `test_event_links.py` |

### Layer 3: Domain Logic (Blocks on L2)

| Task ID | Component | Description | Test File |
|---------|-----------|-------------|-----------|
| L3-1 | `LookupMappingLink` | Find mapping | `test_event_links.py` |
| L3-2 | `KeyboardActionLink` | Keyboard actions | `test_action_links.py` |
| L3-3 | `MouseButtonLink` | Mouse button actions | `test_action_links.py` |
| L3-4 | `MouseMoveLink` | Mouse movement | `test_action_links.py` |
| L3-5 | `SwapProfileLink` | Profile switching | `test_profile_links.py` |
| L3-6 | `ComboActionLink` | Key combinations | `test_action_links.py` |

### Layer 4: Chains (Blocks on L3)

| Task ID | Component | Description | Test File |
|---------|-----------|-------------|-----------|
| L4-1 | `PreloadChain` | Config + init | `test_preload_chain.py` |
| L4-2 | `EventProcessingChain` | Event pipeline | `test_event_processing_chain.py` |
| L4-3 | `ActionExecutionChain` | Action routing | `test_action_execution_chain.py` |
| L4-4 | `RunChain` | Main loop | `test_run_chain.py` |

### Layer 5: Entry Points (Blocks on L4)

| Task ID | Component | Description | Test File |
|---------|-----------|-------------|-----------|
| L5-1 | `cli.py` | Argument parsing | `test_cli.py` |
| L5-2 | `run` command | Start mapper | `test_cli_commands.py` |
| L5-3 | `list` command | List mappings | `test_cli_commands.py` |
| L5-4 | `calibrate` command | Calibration | `test_cli_commands.py` |

---

## Dependency Graph

```
┌─────────────────────────────────────────────────────────────┐
│                       Layer 5: CLI                          │
│         run.py, list.py, add.py, calibrate.py               │
└─────────────────────────────┬───────────────────────────────┘
                              │ uses
┌─────────────────────────────▼───────────────────────────────┐
│                     Layer 4: Chains                         │
│    PreloadChain, RunChain, EventProcessingChain             │
└─────────────────────────────┬───────────────────────────────┘
                              │ composes
┌─────────────────────────────▼───────────────────────────────┐
│                   Layer 3: Domain Links                     │
│   LookupMappingLink, ActionLinks, ProfileLinks              │
└─────────────────────────────┬───────────────────────────────┘
                              │ uses
┌─────────────────────────────▼───────────────────────────────┐
│                 Layer 2: Integration Links                  │
│        InitPygameLink, GetEventsLink, NotificationLink      │
└─────────────────────────────┬───────────────────────────────┘
                              │ uses
┌─────────────────────────────▼───────────────────────────────┐
│                   Layer 1: Foundation Links                 │
│    LoadConfigLink, NormalizeLinks, DeadzoneLink             │
└─────────────────────────────────────────────────────────────┘
```

---

*"Each Link is a Lego brick. The Chain is the instruction booklet. The Context is the baseplate that holds everything together."*
