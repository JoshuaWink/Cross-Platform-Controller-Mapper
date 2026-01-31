# Migration Tasks: GitHub Issues Blueprint

> This document serves as the handoff to the Repo Orchestrator for creating GitHub Issues.

## Milestone 1: Foundation (Week 1)

### Issue Template

```
Title: [M1-{N}] {Component Name}
Labels: migration, layer-1, codeuchain
Milestone: v2.0-foundation

## Description
{description}

## Acceptance Criteria
- [ ] Link/Chain implemented following CodeUChain patterns
- [ ] Unit tests with 90%+ coverage
- [ ] Docstrings on all public methods
- [ ] Type hints throughout

## Test Cases
{test_cases}

## Context Keys
Input: {input_keys}
Output: {output_keys}

## Blocks
{blocking_issues}
```

---

### Foundation Issues

#### M1-1: LoadConfigLink
**Description**: Create Link to load config.json into Context

**Test Cases**:
- ✅ Successfully loads valid config.json
- ✅ Sets `config` and `current_profile_index` in Context
- ✅ Handles missing file gracefully (sets ctx.error)
- ✅ Handles invalid JSON gracefully (sets ctx.error)
- ✅ Uses default path "config.json" if not specified

**Context Keys**:
- Input: `config_path` (optional)
- Output: `config`, `current_profile_index`

**Blocks**: None

---

#### M1-2: SaveConfigLink
**Description**: Create Link to save Context config to file

**Test Cases**:
- ✅ Successfully saves config to specified path
- ✅ Creates file if doesn't exist
- ✅ Overwrites existing file
- ✅ Handles write errors gracefully
- ✅ Preserves JSON formatting (indent=4)

**Context Keys**:
- Input: `config`, `config_path`
- Output: None (side effect: file write)

**Blocks**: M1-1

---

#### M1-3: LoadLayoutLink
**Description**: Create Link to load controller layout JSON

**Test Cases**:
- ✅ Successfully loads layout
- ✅ Uses default "layout__ps4.json" if not specified
- ✅ Handles missing file

**Context Keys**:
- Input: `layout_path` (optional)
- Output: `layout`

**Blocks**: None

---

#### M1-4: NormalizeJoystickLink
**Description**: Normalize joystick raw value to [-1, 1] range

**Test Cases**:
- ✅ Normalizes 0 to 0 (centered)
- ✅ Normalizes max to 1
- ✅ Normalizes min to -1
- ✅ Handles min == max (returns 0)
- ✅ Clamps values outside range

**Context Keys**:
- Input: `raw_value`, `axis_min`, `axis_max`
- Output: `normalized_value`

**Blocks**: None

---

#### M1-5: NormalizeTriggerLink
**Description**: Normalize trigger raw value to [0, 1] range

**Test Cases**:
- ✅ Normalizes min to 0 (not pressed)
- ✅ Normalizes max to 1 (fully pressed)
- ✅ Handles min == max (returns 0)

**Context Keys**:
- Input: `raw_value`, `axis_min`, `axis_max`
- Output: `normalized_value`

**Blocks**: None

---

#### M1-6: ApplyDeadzoneLink
**Description**: Apply deadzone threshold to normalized value

**Test Cases**:
- ✅ Values within deadzone return 0
- ✅ Values outside deadzone pass through
- ✅ Default deadzone is 0.1
- ✅ Custom deadzone respected

**Context Keys**:
- Input: `normalized_value`, `deadzone` (optional, default 0.1)
- Output: `action_value`

**Blocks**: M1-4, M1-5

---

## Milestone 2: Integration Layer (Week 2)

### Issues

#### M2-1: InitPygameLink
**Description**: Initialize pygame and joystick module, return controller

**Test Cases**:
- ✅ Successfully initializes pygame
- ✅ Detects connected controllers
- ✅ Sets ctx.error when no controllers
- ✅ Returns first controller (single controller case)
- ✅ Handles multiple controllers (list)

**Context Keys**:
- Input: None
- Output: `controller`, `controller_name`

**Blocks**: M1-1 (needs config for logging setup)

---

#### M2-2: GetEventsLink
**Description**: Get pygame events for current frame

**Test Cases**:
- ✅ Returns empty list when no events
- ✅ Returns list of pygame events
- ✅ Calls pg.event.pump()

**Context Keys**:
- Input: `controller`
- Output: `pygame_events`

**Blocks**: M2-1

---

#### M2-3: ParseEventLink
**Description**: Parse single pygame event into Context fields

**Test Cases**:
- ✅ Parses JOYBUTTONDOWN → button_press
- ✅ Parses JOYBUTTONUP → button_release
- ✅ Parses JOYAXISMOTION → axis_motion
- ✅ Parses JOYHATMOTION → hat_motion
- ✅ Unknown events → event_type "unknown"

**Context Keys**:
- Input: `pygame_event`
- Output: `event_type`, `button_id`/`axis_id`, `raw_value`

**Blocks**: M2-2

---

#### M2-4: NotificationLink
**Description**: Send system notification (macOS via pync)

**Test Cases**:
- ✅ Sends notification with title and message
- ✅ No-op when notification not in Context
- ✅ Handles pync errors gracefully on non-macOS

**Context Keys**:
- Input: `notification` (dict with `title`, `message`)
- Output: None (side effect: notification)

**Blocks**: None

---

## Milestone 3: Domain Logic (Week 3)

### Issues

#### M3-1: LookupMappingLink
**Description**: Find mapping for button/axis in current profile

**Test Cases**:
- ✅ Finds button mapping by button_id
- ✅ Finds axis mapping by axis_id
- ✅ Returns None mapping when not found
- ✅ Uses current_profile from Context

**Context Keys**:
- Input: `button_id`/`axis_id`, `current_profile`, `event_type`
- Output: `mapping`, `action_string`

**Blocks**: M1-1, M2-3

---

#### M3-2: KeyboardActionLink
**Description**: Press/release keyboard key via pynput

**Test Cases**:
- ✅ Presses key when action_value is truthy
- ✅ Releases key when action_value is falsy
- ✅ Handles special keys (ctrl, alt, shift, cmd)
- ✅ Handles regular keys (a, b, space, etc.)

**Context Keys**:
- Input: `key_name`, `action_value`
- Output: None (side effect: keyboard action)

**Blocks**: M3-1

---

#### M3-3: MouseButtonLink
**Description**: Press/release mouse button via pynput

**Test Cases**:
- ✅ Presses left button
- ✅ Presses right button
- ✅ Presses middle button
- ✅ Releases on action_value 0

**Context Keys**:
- Input: `button_name`, `action_value`
- Output: None (side effect: mouse action)

**Blocks**: M3-1

---

#### M3-4: MouseMoveLink
**Description**: Move mouse cursor with smoothing

**Test Cases**:
- ✅ Moves mouse by dx, dy
- ✅ Applies speed multiplier
- ✅ Smooths movement over multiple calls

**Context Keys**:
- Input: `dx`, `dy`, `speed` (optional)
- Output: None (side effect: mouse movement)

**Blocks**: M3-1

---

#### M3-5: SwapProfileLink
**Description**: Switch to next profile in list

**Test Cases**:
- ✅ Increments profile index
- ✅ Wraps around to 0 at end
- ✅ Sets notification
- ✅ Handles empty profiles list

**Context Keys**:
- Input: `config`, `current_profile_index`
- Output: `current_profile_index`, `current_profile`, `notification`

**Blocks**: M1-1, M2-4

---

#### M3-6: ComboActionLink
**Description**: Execute key combination (e.g., "cmd+shift+p")

**Test Cases**:
- ✅ Parses combo string
- ✅ Presses all keys in order
- ✅ Releases all keys in reverse order
- ✅ Handles mixed Key.X and single char

**Context Keys**:
- Input: `combo_string`
- Output: None (side effect: keyboard combo)

**Blocks**: M3-2

---

## Milestone 4: Chains (Week 4)

### Issues

#### M4-1: PreloadChain
**Description**: Chain that loads config, layout, and initializes pygame

**Test Cases**:
- ✅ Loads all config files
- ✅ Initializes controller
- ✅ Sets current_profile
- ✅ Short-circuits on any Link error

**Composition**:
```python
LoadConfigLink → LoadLayoutLink → LoadMappingsLink → InitPygameLink → GetCurrentProfileLink
```

**Blocks**: M1-1, M1-3, M2-1

---

#### M4-2: EventProcessingChain
**Description**: Chain that processes single event to action resolution

**Test Cases**:
- ✅ Parses event
- ✅ Validates event (not paused, etc.)
- ✅ Looks up mapping
- ✅ Returns resolved action

**Composition**:
```python
ParseEventLink → ValidateEventLink → LookupMappingLink
```

**Blocks**: M2-3, M3-1

---

#### M4-3: ActionExecutionChain
**Description**: Chain that executes resolved action

**Test Cases**:
- ✅ Routes to KeyboardActionLink for Key.* actions
- ✅ Routes to MouseButtonLink for Button.* actions
- ✅ Routes to ComboActionLink for combo actions
- ✅ Routes to CustomActionLink for named actions

**Composition**: Dynamic routing based on action_type

**Blocks**: M3-2, M3-3, M3-4, M3-5, M3-6

---

#### M4-4: NormalizationChain
**Description**: Chain that normalizes and applies deadzone

**Test Cases**:
- ✅ Normalizes joystick values correctly
- ✅ Normalizes trigger values correctly
- ✅ Applies deadzone

**Composition**:
```python
(NormalizeJoystickLink | NormalizeTriggerLink) → ApplyDeadzoneLink
```

**Blocks**: M1-4, M1-5, M1-6

---

#### M4-5: RunChain
**Description**: Main event loop Chain

**Test Cases**:
- ✅ Preloads configuration
- ✅ Enters event loop
- ✅ Processes events continuously
- ✅ Handles KeyboardInterrupt gracefully

**Composition**:
```python
PreloadChain → EventLoopLink (internal loop)
```

**Blocks**: M4-1, M4-2, M4-3

---

## Milestone 5: CLI & Polish (Week 5)

### Issues

#### M5-1: CLI Parser
**Description**: Refactor argparse into clean entry point

**Test Cases**:
- ✅ Parses all subcommands
- ✅ Validates arguments
- ✅ Builds initial Context

**Blocks**: M4-5

---

#### M5-2: Run Command
**Description**: Entry point for `controller-mapper run`

**Test Cases**:
- ✅ Calls RunChain
- ✅ Handles errors with user-friendly messages

**Blocks**: M5-1, M4-5

---

#### M5-3: List Command
**Description**: Entry point for `controller-mapper list`

**Test Cases**:
- ✅ Lists all mappings for current profile
- ✅ Formats output nicely

**Blocks**: M5-1

---

#### M5-4: Calibrate Command
**Description**: Entry point for `controller-mapper calibrate`

**Test Cases**:
- ✅ Runs CalibrationChain
- ✅ Saves results

**Blocks**: M5-1

---

## Summary Gantt

```
Week 1:  M1-1 M1-2 M1-3 M1-4 M1-5 M1-6
Week 2:  M2-1 M2-2 M2-3 M2-4
Week 3:  M3-1 M3-2 M3-3 M3-4 M3-5 M3-6
Week 4:  M4-1 M4-2 M4-3 M4-4 M4-5
Week 5:  M5-1 M5-2 M5-3 M5-4
```

## Dependencies Graph (for Repo Orchestrator)

```yaml
M1-1: []
M1-2: [M1-1]
M1-3: []
M1-4: []
M1-5: []
M1-6: [M1-4, M1-5]
M2-1: [M1-1]
M2-2: [M2-1]
M2-3: [M2-2]
M2-4: []
M3-1: [M1-1, M2-3]
M3-2: [M3-1]
M3-3: [M3-1]
M3-4: [M3-1]
M3-5: [M1-1, M2-4]
M3-6: [M3-2]
M4-1: [M1-1, M1-3, M2-1]
M4-2: [M2-3, M3-1]
M4-3: [M3-2, M3-3, M3-4, M3-5, M3-6]
M4-4: [M1-4, M1-5, M1-6]
M4-5: [M4-1, M4-2, M4-3]
M5-1: [M4-5]
M5-2: [M5-1, M4-5]
M5-3: [M5-1]
M5-4: [M5-1]
```

---

*"Each issue is a single, testable unit of work. Complete the tree from leaves to root."*
