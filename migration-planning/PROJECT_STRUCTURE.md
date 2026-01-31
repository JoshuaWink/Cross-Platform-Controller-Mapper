# Proposed Project Structure

## Directory Layout

```
cross_platform_controller_mapper/
├── pyproject.toml                    # Package config & dependencies
├── README.md                         # User documentation
├── config.json                       # Runtime configuration (unchanged)
├── layout__ps4.json                  # Controller layout (unchanged)
├── mappings.json                     # Human-friendly export (unchanged)
│
├── src/
│   └── controller_mapper/
│       ├── __init__.py
│       │
│       ├── links/                    # ═══ DOMAIN LOGIC LAYER ═══
│       │   ├── __init__.py
│       │   │
│       │   ├── config/               # Configuration Links
│       │   │   ├── __init__.py
│       │   │   ├── load_config.py        # LoadConfigLink
│       │   │   ├── save_config.py        # SaveConfigLink
│       │   │   ├── load_layout.py        # LoadLayoutLink
│       │   │   └── load_mappings.py      # LoadMappingsLink
│       │   │
│       │   ├── normalization/        # Value Transformation Links
│       │   │   ├── __init__.py
│       │   │   ├── normalize_joystick.py # NormalizeJoystickLink
│       │   │   ├── normalize_trigger.py  # NormalizeTriggerLink
│       │   │   ├── apply_deadzone.py     # ApplyDeadzoneLink
│       │   │   └── restrict_value.py     # RestrictValueLink
│       │   │
│       │   ├── events/               # Event Processing Links
│       │   │   ├── __init__.py
│       │   │   ├── parse_event.py        # ParseEventLink
│       │   │   ├── route_event.py        # RouteEventLink
│       │   │   ├── lookup_mapping.py     # LookupMappingLink
│       │   │   └── validate_event.py     # ValidateEventLink
│       │   │
│       │   ├── actions/              # Action Execution Links
│       │   │   ├── __init__.py
│       │   │   ├── base_action.py        # ActionLink (abstract base)
│       │   │   ├── keyboard.py           # KeyboardActionLink
│       │   │   ├── mouse_button.py       # MouseButtonLink
│       │   │   ├── mouse_move.py         # MouseMoveLink
│       │   │   ├── mouse_scroll.py       # MouseScrollLink
│       │   │   ├── combo.py              # ComboActionLink
│       │   │   ├── arrow_keys.py         # ArrowKeysLink
│       │   │   └── custom.py             # CustomActionLink (ExtendScript, etc.)
│       │   │
│       │   ├── profile/              # Profile Management Links
│       │   │   ├── __init__.py
│       │   │   ├── get_current.py        # GetCurrentProfileLink
│       │   │   ├── swap_profile.py       # SwapProfileLink
│       │   │   ├── set_mapping.py        # SetMappingLink
│       │   │   └── export_human.py       # ExportHumanFriendlyLink
│       │   │
│       │   └── calibration/          # Calibration Links
│       │       ├── __init__.py
│       │       ├── calibrate_axis.py     # CalibrateAxisLink
│       │       ├── set_deadzone.py       # SetDeadzoneLink
│       │       └── get_axis_value.py     # GetAxisValueLink
│       │
│       ├── chains/                   # ═══ ORCHESTRATION LAYER ═══
│       │   ├── __init__.py
│       │   ├── preload.py                # PreloadChain (config + pygame init)
│       │   ├── run.py                    # RunChain (main event loop)
│       │   ├── event_processing.py       # EventProcessingChain
│       │   ├── action_execution.py       # ActionExecutionChain
│       │   ├── profile_management.py     # ProfileManagementChain
│       │   ├── calibration.py            # CalibrationChain
│       │   ├── mapping.py                # MappingChain (interactive)
│       │   └── export.py                 # ExportChain (human-friendly)
│       │
│       ├── integration/              # ═══ INTEGRATION LAYER ═══
│       │   ├── __init__.py
│       │   ├── pygame_adapter.py         # PygameEventLink, InitPygameLink
│       │   ├── pynput_adapter.py         # PynputKeyboardLink, PynputMouseLink
│       │   ├── notification_adapter.py   # NotificationLink (pync)
│       │   ├── file_adapter.py           # FileReadLink, FileWriteLink
│       │   └── script_runner.py          # ScriptExecutionLink
│       │
│       ├── middleware/               # ═══ CROSS-CUTTING CONCERNS ═══
│       │   ├── __init__.py
│       │   ├── logging_middleware.py     # LoggingMiddleware
│       │   ├── timing_middleware.py      # TimingMiddleware
│       │   ├── error_handler.py          # ErrorHandlerMiddleware
│       │   └── pause_check.py            # PauseCheckMiddleware
│       │
│       └── entry/                    # ═══ ENTRY POINTS LAYER ═══
│           ├── __init__.py
│           ├── cli.py                    # CLI argument parsing
│           └── commands/
│               ├── __init__.py
│               ├── run.py                # `run` command
│               ├── list.py               # `list` command
│               ├── add.py                # `add` command
│               ├── remove.py             # `remove` command
│               ├── map.py                # `map` command (interactive)
│               ├── calibrate.py          # `calibrate` command
│               ├── listen.py             # `listen` command
│               ├── log.py                # `log` command
│               └── human.py              # `human` command
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                   # Shared fixtures
│   │
│   ├── unit/                         # ═══ UNIT TESTS (Isolated Links) ═══
│   │   ├── __init__.py
│   │   ├── links/
│   │   │   ├── test_config_links.py
│   │   │   ├── test_normalization_links.py
│   │   │   ├── test_event_links.py
│   │   │   ├── test_action_links.py
│   │   │   ├── test_profile_links.py
│   │   │   └── test_calibration_links.py
│   │   └── middleware/
│   │       └── test_middleware.py
│   │
│   ├── integration/                  # ═══ INTEGRATION TESTS (Chains) ═══
│   │   ├── __init__.py
│   │   ├── test_preload_chain.py
│   │   ├── test_event_processing_chain.py
│   │   ├── test_action_execution_chain.py
│   │   └── test_profile_chain.py
│   │
│   └── e2e/                          # ═══ END-TO-END TESTS ═══
│       ├── __init__.py
│       ├── test_cli_commands.py
│       └── test_full_workflow.py
│
└── docs/
    ├── architecture.md               # Architecture decision records
    ├── extending.md                  # How to add new actions
    └── troubleshooting.md            # Common issues
```

## Layer Responsibilities

### Entry Points Layer (`entry/`)

**Purpose**: Accept user input, parse arguments, call appropriate Chains

**Contract**:
- Parse CLI arguments → build initial Context
- Call Chain.execute(context)
- Handle Chain result (success/error) → user feedback

**Example**:
```python
# entry/commands/run.py
def run_command(args):
    ctx = Context({
        "command": "run",
        "config_path": args.config or "config.json",
        "verbose": args.verbose
    })
    result = RunChain().execute(ctx)
    if result.error:
        print(f"Error: {result.error}")
        sys.exit(1)
```

---

### Orchestration Layer (`chains/`)

**Purpose**: Compose Links into workflows, define execution order

**Contract**:
- Chains are composed of Links
- Chains don't contain business logic—they delegate
- Chains handle Link results and control flow

**Example**:
```python
# chains/preload.py
class PreloadChain(Chain):
    def __init__(self):
        super().__init__()
        self.add_link(LoadConfigLink())
        self.add_link(LoadLayoutLink())
        self.add_link(LoadMappingsLink())
        self.add_link(InitPygameLink())
        self.add_link(ValidateSetupLink())
```

---

### Domain Logic Layer (`links/`)

**Purpose**: Pure business logic, single responsibility per Link

**Contract**:
- Each Link does ONE thing
- Links read from Context, write to Context
- Links are stateless—all state in Context
- Links testable in complete isolation

**Example**:
```python
# links/normalization/normalize_joystick.py
class NormalizeJoystickLink(Link):
    def __init__(self):
        super().__init__("normalize_joystick")
    
    def execute(self, ctx: Context) -> Context:
        raw = ctx.get("raw_value")
        min_val = ctx.get("axis_min")
        max_val = ctx.get("axis_max")
        
        if min_val == max_val:
            ctx.set("normalized_value", 0.0)
            return ctx
        
        normalized = 2 * (raw - min_val) / (max_val - min_val) - 1
        normalized = max(-1.0, min(1.0, normalized))
        ctx.set("normalized_value", normalized)
        return ctx
```

---

### Integration Layer (`integration/`)

**Purpose**: Wrap external libraries, provide mockable interfaces

**Contract**:
- Adapts external APIs to Context-based interface
- Isolates dependencies so they can be mocked
- Handles library-specific errors gracefully

**Example**:
```python
# integration/pygame_adapter.py
class InitPygameLink(Link):
    def __init__(self):
        super().__init__("init_pygame")
    
    def execute(self, ctx: Context) -> Context:
        try:
            pg.init()
            pg.joystick.init()
            count = pg.joystick.get_count()
            
            if count == 0:
                ctx.error = Exception("No controllers detected")
                return ctx
            
            controller = pg.joystick.Joystick(0)
            controller.init()
            ctx.set("controller", controller)
            ctx.set("controller_name", controller.get_name())
        except Exception as e:
            ctx.error = e
        
        return ctx
```

---

### Middleware Layer (`middleware/`)

**Purpose**: Cross-cutting concerns that wrap Links

**Contract**:
- Wraps Link execution
- Can modify Context before/after Link
- Handles logging, timing, error catching

**Example**:
```python
# middleware/logging_middleware.py
class LoggingMiddleware:
    def __init__(self, logger=None):
        self.logger = logger or logging.getLogger(__name__)
    
    def wrap(self, link: Link) -> Link:
        original_execute = link.execute
        
        def logged_execute(ctx: Context) -> Context:
            self.logger.debug(f"Entering {link.name}")
            result = original_execute(ctx)
            if result.error:
                self.logger.error(f"{link.name} failed: {result.error}")
            else:
                self.logger.debug(f"Exiting {link.name}")
            return result
        
        link.execute = logged_execute
        return link
```

---

## Context Schema

```python
# Common Context keys and their types

# Configuration
"config_path": str          # Path to config.json
"layout_path": str          # Path to layout file
"config": dict              # Loaded config data
"layout": dict              # Loaded layout data
"mappings": dict            # Loaded mappings data

# Profile
"current_profile_index": int
"current_profile": dict     # Active profile object
"profile_name": str

# Controller
"controller": pygame.joystick.Joystick
"controller_name": str

# Event Processing
"event_type": str           # "button_press", "button_release", "axis_motion"
"button_id": int
"axis_id": int
"raw_value": float
"normalized_value": float
"action_value": float       # After deadzone

# Action Resolution
"mapping": dict             # Looked up mapping for input
"action_string": str        # e.g., "Key.alt", "SwapProfile"
"action_type": str          # "keyboard", "mouse", "custom", "combo"
"resolved_action": dict     # Fully resolved action details

# State
"is_paused": bool
"held_buttons": set
"skip_axes": dict

# Output
"notification": dict        # {"title": str, "message": str}
"error": Exception          # Set when Link fails
```

---

## File Naming Conventions

| Pattern | Example | Description |
|---------|---------|-------------|
| `{action}.py` | `keyboard.py` | Action execution Links |
| `{verb}_{noun}.py` | `load_config.py` | Operation Links |
| `{noun}_adapter.py` | `pygame_adapter.py` | Integration Links |
| `{noun}_middleware.py` | `logging_middleware.py` | Middleware |
| `{noun}.py` | `preload.py` | Chains |
| `test_{module}.py` | `test_config_links.py` | Test files |

---

## Import Structure

```python
# Clean imports from package root
from controller_mapper.links.config import LoadConfigLink, SaveConfigLink
from controller_mapper.links.actions import KeyboardActionLink, MouseButtonLink
from controller_mapper.chains import PreloadChain, RunChain
from controller_mapper.middleware import LoggingMiddleware
```

---

*"The structure reveals the architecture. If you understand the folders, you understand the system."*
