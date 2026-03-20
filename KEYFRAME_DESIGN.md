# Keyframe Animation System - Design Document

## Context

A grammar-based keyframe animation system for the Homey smart home light controller (`com.robot.yet.layeredlight`). The existing app supports static scene layering with priorities. This system adds time-based animation (keyframes) with interpolation.

---

## 1. Formal Grammar

### EBNF

```ebnf
layer         ::= Light_assignment (top level)
Light_assignment ::= light_name ":" value

value         ::= keyframe_value
```

### Lexical Rules

| Token          | Pattern                          |
| -------------- | -------------------------------- |
| `light_name`   | `[^:]+`                         |
| `octet`        | `[a-fA-F0-9]{2}`               |
| `duration`     | `[0-9]+(ms\|s\|m)`              |
| `keyword`      | `on \| off \| null`             |

### Setting Grammar

```
setting   ::= (<octet>{1,3} | on | off | null)
```

- **1 octet**: brightness only (0x00–0xFF)
- **2 octets**: brightness + color temperature
- **3 octets**: full HSL (hue, saturation, brightness) — note: interpreted as r, g, b values per the grammar description
- **`on`**: special "on" command
- **`off`**: special "off" command
- **`null`**: inherit from lower-priority layer / default value

### Pattern Grammar

```
pattern    ::= transition? <setting> (<transition> <setting>)* <transition>?
transition ::= ( "/" <duration> | "[" | "[" "]" <duration> "[" "]" )
```

Where:
- `/` prefix = linear interpolation (e.g., `/2s` means linearly interpolate over 2 seconds)
- `|` = step function (instant jump; value changes at the moment the next keyframe fires)
- Durations default to **seconds** unless a unit (`ms`, `s`, `m`) is specified

### Layer Assignment Grammar

```
layer := <setting> ("\s+" <setting>)*
setting := <light_name> ":" <pattern>
```

A layer is a whitespace-separated list of `light_name:pattern` pairs.

### Identified Ambiguities

1. **Grammar disambiguation**: The original grammar uses `\s+` as a separator between settings, but light names may contain spaces. **Resolution**: rename inner setting separator to require a comma or semicolon, OR restrict light names to exclude whitespace. **Chosen approach**: light names must not contain whitespace; use `_` or `-` instead.

2. **Null interpolation**: When transitioning to/from `null`, the system must resolve `null` to the underlying layer's current value before interpolating. This requires reading the layer stack at transition start time.

3. **Mid-animation reassignment**: If a layer is reassigned mid-animation, the old animation's current computed value becomes the "previous value" for any new transition that starts with a transition marker.

4. **Type mixing in patterns**: Transitioning between different setting types (e.g., 1-octet brightness to 3-octet HSL). **Resolution**: promote to the wider representation for interpolation (e.g., expand brightness-only to full HSL with neutral hue/saturation).

---

## 2. Parse Tree Examples

### Example 1: Simple constant brightness

```
kitchen_light:80
```

Parse tree:
```
Light_assignment
├── light_name: "kitchen_light"
└── pattern
    └── setting: 0x80 (brightness = 128)
```

### Example 2: Linear fade from off to full brightness over 3 seconds

```
hallway:00/3s FF
```

Parse tree:
```
Light_assignment
├── light_name: "hallway"
└── pattern
    ├── setting: 0x00 (brightness = 0)
    ├── transition: "/" duration=3s (linear interpolation)
    └── setting: 0xFF (brightness = 255)
```

### Example 3: Looping color cycle

```
bedroom:FF0000/2sFFA500/2sFFFF00/2s00FF00/2s0000FF/2s
```

Parse tree (trailing transition makes it loop):
```
Light_assignment
├── light_name: "bedroom"
└── pattern
    ├── setting: 0xFF0000 (red)
    ├── transition: "/" 2s
    ├── setting: 0xFFA500 (orange)
    ├── transition: "/" 2s
    ├── setting: 0xFFFF00 (yellow)
    ├── transition: "/" 2s
    ├── setting: 0x00FF00 (green)
    ├── transition: "/" 2s
    ├── setting: 0x0000FF (blue)
    └── transition: "/" 2s  ← trailing = loop back to first setting
```

### Example 4: Step function (instant change)

```
porch:|on|3s|off
```

Parse tree:
```
Light_assignment
├── light_name: "porch"
└── pattern
    ├── transition: "|" (step, start from previous)
    ├── setting: on
    ├── transition: "|" duration=3s (step)
    └── setting: off
```

### Example 5: Null (inherit from lower layer)

```
desk_lamp:null
```

Parse tree:
```
Light_assignment
├── light_name: "desk_lamp"
└── pattern
    └── setting: null (inherit from lower-priority layer)
```

---

## 3. Time Model & Keyframe Evaluation

### Mathematical Definition

Given:
- `eval(pattern, t_assign, t_now)` → value (keyframe value)
- A pattern with settings `S_0, S_1, ..., S_n` and transitions `T_1, T_2, ..., T_n` (between them)
- Each transition `T_i` has a duration `d_i` and a type (linear `/` or step `|`)

**Key-keeping timeline**: walk keyframes, find active segment.

#### Looping Timeline (loop marker: trailing transition)

The total cycle duration:

```python
D = sum(d_i for i in 1..n) + d_loop   # d_loop = trailing transition duration
```

Cycle arithmetic on elapsed time:

```python
t_elapsed = t_now - t_assign
t_cycle = t_elapsed % D     # wraps to first value
```

#### Finding the Active Segment

```python
def eval(pattern, t_assign, t_now):
    """Evaluate a pattern at a given time."""
    settings = pattern.settings      # S_0 .. S_n
    transitions = pattern.transitions # T_1 .. T_n (+ optional T_loop)
    is_loop = pattern.has_trailing_transition

    t_elapsed = t_now - t_assign

    if is_loop:
        D = sum(t.duration for t in transitions)
        t_elapsed = t_elapsed % D

    # Walk through segments to find active one
    cursor = 0.0
    for i, transition in enumerate(transitions):
        seg_end = cursor + transition.duration
        if t_elapsed < seg_end:
            # We're in this segment
            progress = (t_elapsed - cursor) / transition.duration if transition.duration > 0 else 1.0
            s_from = settings[i]
            s_to = settings[(i + 1) % len(settings)] if is_loop and i == len(transitions) - 1 else settings[i + 1]
            return interpolate(transition.type, s_from, s_to, progress)
        cursor = seg_end

    # Past all transitions (non-looping): hold last value
    return settings[-1]
```

#### Leading Transition (pattern starts with transition)

If the animation starts with a transition, `S_from` is the **previous value** — a snapshot of the old animation's computed value at `t_assign`:

```python
if pattern.starts_with_transition:
    s_from = snapshot_previous_value(light, t_assign)
    # Use s_from as the initial setting for the first segment
```

#### Transition Function (loop marker)

When pattern ends with a transition (trailing):
- wraps to first setting
- creates a cycle

---

## 4. Interpolation

### Type Hierarchy for Interpolation

Settings have different "widths":

| Type            | Octets | Represents                     |
| --------------- | ------ | ------------------------------ |
| brightness      | 1      | brightness only                |
| brightness+temp | 2      | brightness + color temperature |
| HSL / RGB       | 3      | full color (hue+sat+brightness)|

**Type promotion rules for mixed-type transitions** (e.g., brightness → HSL):

- `brightness → brightness+temp`: assume neutral temperature
- `brightness → HSL`: assume neutral hue & saturation, use brightness
- `brightness+temp → HSL`: convert to full HSL space

### Linear Interpolation (`/`)

For each channel independently:

```python
def lerp(a: float, b: float, t: float) -> float:
    """Linear interpolation: a at t=0, b at t=1."""
    return a + (b - a) * t
```

For multi-channel values:

```python
def interpolate_linear(s_from, s_to, progress):
    """Interpolate between two settings linearly."""
    # Promote both to common type
    s_from, s_to = promote_to_common_type(s_from, s_to)
    return tuple(lerp(a, b, progress) for a, b in zip(s_from, s_to))
```

### Step Function (`|`)

```python
def interpolate_step(s_from, s_to, progress):
    """Step function: hold s_from until progress reaches 1.0."""
    return s_from if progress < 1.0 else s_to
```

### Null Blending

When `null` appears in an interpolation:

```python
def resolve_null(light_name, layer_stack, current_layer_index):
    """Resolve null by looking at lower-priority layers."""
    for i in range(current_layer_index + 1, len(layer_stack)):
        val = layer_stack[i].get(light_name)
        if val is not None:
            return val
    return DEFAULT_VALUE  # black, off
```

Null blending requires resolving the lower layer value at the time of evaluation. This is critical because:
- The lower layer may itself be animating
- The resolved value is a **moving target** that changes frame to frame

---

## 5. Layer Flattening with Animations

### Evaluation at Time t

At time `t`, each layer evaluates all its patterns to produce a **scene** (a dict of light_name → value, with possible `null` entries):

```python
def evaluate_layer(layer, t_now):
    """Evaluate all light patterns in a layer at time t."""
    scene = {}
    for light_name, (pattern, t_assign) in layer.assignments.items():
        scene[light_name] = eval(pattern, t_assign, t_now)
    return scene
```

### Flattening the Layer Stack

Layers are ordered by priority (high to low). Flattening resolves `null` values:

```python
def flatten(layers, t_now):
    """
    Flatten layer stack into final light values.
    layers: list of layers, ordered highest priority first.
    """
    scenes = [evaluate_layer(layer, t_now) for layer in layers]

    # Collect all light names
    all_lights = set()
    for scene in scenes:
        all_lights.update(scene.keys())

    result = {}
    for light in all_lights:
        for scene in scenes:
            val = scene.get(light)
            if val is not None:  # null means "pass through"
                result[light] = val
                break
        else:
            result[light] = DEFAULT_VALUE  # black, off

    return result
```

### Diagram: Layer Stack Evaluation

```
Time t
┌─────────────────────────────────────┐
│  Layer 0 (highest priority)         │
│  kitchen: eval(pattern, t) → 0xA0   │  ← non-null, wins
│  bedroom: eval(pattern, t) → null   │  ← null, pass through
├─────────────────────────────────────┤
│  Layer 1                            │
│  kitchen: eval(pattern, t) → 0xFF   │  ← shadowed by Layer 0
│  bedroom: eval(pattern, t) → 0x80   │  ← first non-null, wins
│  hallway: eval(pattern, t) → 0x40   │  ← only layer, wins
├─────────────────────────────────────┤
│  Default (black, off)               │
│  All lights → off                   │
└─────────────────────────────────────┘

Result: { kitchen: 0xA0, bedroom: 0x80, hallway: 0x40 }
```

### Reassignment During Looping

When a layer pattern is reassigned (new pattern for same light on same layer):

1. Evaluate old pattern at current time → `previous_value`
2. Store `previous_value` as snapshot
3. If new pattern starts with a transition, use `previous_value` as `S_from`
4. Replace pattern with new one, set `t_assign = t_now`

**Edge case**: if the new transition's target value equals `previous_value`, the optimizer can skip the transition entirely.

---

## 6. Output Command Generation

### Light Capability Classes

Different lights support different command types:

| Capability Class | Commands Available                      |
| ---------------- | --------------------------------------- |
| `on_off`         | `on`, `off`                             |
| `dim`            | `on`, `off`, brightness                 |
| `dim_temp`       | `on`, `off`, brightness, temperature    |
| `full_color`     | `on`, `off`, hue, saturation, brightness|

### Mapped Flattened Values to Device Commands

```python
def to_commands(light_name, value, capability_class):
    """Convert a flattened value to device commands."""
    if value == OFF:
        return [Command(light_name, "off")]
    if value == ON:
        return [Command(light_name, "on")]

    match capability_class:
        case "on_off":
            return [Command(light_name, "on" if brightness(value) > 0 else "off")]
        case "dim":
            return [Command(light_name, "brightness", brightness(value))]
        case "dim_temp":
            return [
                Command(light_name, "brightness", brightness(value)),
                Command(light_name, "temperature", temperature(value)),
            ]
        case "full_color":
            return [
                Command(light_name, "hue", hue(value)),
                Command(light_name, "saturation", saturation(value)),
                Command(light_name, "brightness", brightness(value)),
            ]
```

### Linear Interpolation Delegation

For transitions using `/` (linear), set the **current value immediately** and the **target value with a duration**:

```python
def emit_interpolation(light, current_val, target_val, duration):
    """
    Emit commands for a linear interpolation.
    Sets current value immediately, then target with duration.
    """
    commands = []
    # Set current value immediately (instantaneous)
    commands.append(SetCommand(light, current_val, duration=0))
    # Set target value with transition duration
    commands.append(SetCommand(light, target_val, duration=duration))
    return commands
```

### Step Transitions

For `|` step transitions:
- Set value immediately at the moment the keyframe fires (or at next keyframe tick)
- Duration elapses, then the next value is set

---

## 7. Optimizer

The optimizer sits between the evaluator and the command output. It minimizes redundant commands by comparing desired state against known device state.

### State Tracking

```python
known_state: dict[str, LightState]  # per light: {value, last_updated}
target_state: dict[str, LightState] # per light: target within epsilon
```

### Optimization Rules

```python
def optimize(commands, known_state):
    """
    Filter out redundant commands.
    known_state: last known state of each light.
    """
    filtered = []
    for cmd in commands:
        light = cmd.light_name

        # Rule 1: Skip if known state already matches (within epsilon)
        if light in known_state:
            if is_close(known_state[light], cmd.value, epsilon=1):
                continue

        # Rule 2: Redundancy detection for transitions
        # If light is already transitioning to this target, skip
        if light in active_transitions:
            if active_transitions[light].target == cmd.value:
                continue

        # Rule 3: Staleness handling
        # If known_state is too old (> threshold), issue command anyway
        if light in known_state:
            if (t_now - known_state[light].last_updated) > STALE_THRESHOLD:
                pass  # don't skip, re-issue

        filtered.append(cmd)
        known_state[light] = LightState(cmd.value, t_now)

    return filtered
```

### Race Condition Discussion

The current state of lights may be updated at entirely uncoordinated times (e.g., physical switches, other apps, other Homey flows). The optimizer must handle this:

1. **External state changes invalidate `known_state`**: Periodically refresh known state from device queries, or subscribe to state-change events.
2. **Staleness threshold**: If `known_state` is older than a threshold (e.g., 5 seconds for actively animating lights, 30 seconds for static), re-issue commands even if they appear redundant.
3. **Best-effort approach**: The optimizer cannot guarantee perfect synchronization. It issues commands based on the best available information. Missed updates may cause a single frame of incorrect state, which self-corrects on the next evaluation cycle.

---

## 8. Ambiguities & Edge Cases Catalog

### Grammar Ambiguities

1. **Light name conflicts**: Light names with `:` would break the `name:pattern` split.
   - **Resolution**: Light names must not contain `:`.

2. **Duration parsing**: `10ms` vs `10m` followed by setting `s...` — is `10ms` ten milliseconds or ten minutes followed by a setting starting with `s`?
   - **Resolution**: Duration tokens are greedy; `ms` is matched before `m`.

3. **Octet boundaries**: `FFFF00` — is this `FF` + `FF` + `00` (3 octets) or `FF` + `FF00` (ambiguous)?
   - **Resolution**: Always parse in groups of exactly 2 hex characters. 6 hex chars = 3 octets.

### Interpolation Edge Cases

4. **Null interpolation mechanics**: When interpolating toward `null`, the resolved lower-layer value is a moving target.
   - **Approach**: Re-resolve null at each evaluation tick. The interpolation target shifts each frame.

5. **Zero-duration transitions**: `/0s` — instantaneous linear transition is equivalent to a step.
   - **Resolution**: Treat as step function.

6. **Empty patterns**: A pattern with no settings.
   - **Resolution**: Invalid; reject at parse time.

### Layer Edge Cases

7. **Reassignment during looping**: Mid-loop reassignment snapshots current animated value.
   - See Section 5 for detailed behavior.

8. **All layers null for a light**: Every layer returns `null` for a given light.
   - **Resolution**: Fall through to default (black, off).

9. **Transition from `off` to color**: `off/2sFF8040` — interpolating from "off" to a color.
   - **Resolution**: Treat `off` as brightness 0 with neutral color. Fade in from black.

10. **Transition from `on` to value**: `on/2s80` — `on` has no numeric brightness.
    - **Resolution**: Treat `on` as brightness 0xFF (full brightness) for interpolation purposes.

### Evaluation Cadence

11. **Tick rate / evaluation cadence**: How often to tick the system.
    - Not specified in grammar. Implementation decision. Recommended: 100ms–1s depending on whether any animations are active. Adaptive tick rate: fast when animating, slow when static.

---

## 9. Diagrams

### System Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Grammar    │     │    Layer     │     │   Output     │
│   Parser     │────▶│   Evaluator  │────▶│  Generator   │
│              │     │              │     │              │
│  text → AST  │     │ AST×t → val  │     │ val → cmds   │
└──────────────┘     └──────┬───────┘     └──────┬───────┘
                            │                     │
                     ┌──────▼───────┐     ┌──────▼───────┐
                     │   Layer      │     │  Optimizer   │
                     │   Flattener  │     │              │
                     │              │     │ cmds → cmds' │
                     │ layers → val │     │ (skip dups)  │
                     └──────────────┘     └──────┬───────┘
                                                  │
                                          ┌──────▼───────┐
                                          │   Homey      │
                                          │   Device API │
                                          └──────────────┘
```

### Animation Timeline Diagram

```
Pattern: 00 /3s FF /1s 00 /3s    (trailing transition = loop)

Brightness
0xFF │            ╱‾‾‾‾╲
     │          ╱        ╲
     │        ╱            ╲
     │      ╱                ╲
0x00 │____╱                    ╲____╱ ...
     └──────────────────────────────────▶ time
     t₀   t₀+3s  t₀+4s  t₀+7s  t₀+10s

     ◀─ lerp ─▶◀step▶◀─ lerp ──▶◀loop▶
```

### Layer Priority Resolution

```
Priority:   High ──────────────────────▶ Low

            Layer "alert"    Layer "mood"    Default
            ┌────────┐       ┌────────┐     ┌────────┐
kitchen:    │  null   │ ───▶  │  0xA0  │     │  off   │
            │(pass)   │       │(wins!) │     │        │
            ├────────┤       ├────────┤     ├────────┤
bedroom:    │  0xFF  │       │  0x40  │     │  off   │
            │(wins!) │       │(shadow)│     │        │
            ├────────┤       ├────────┤     ├────────┤
hallway:    │  null   │ ───▶  │  null  │ ──▶ │  off   │
            │(pass)   │       │(pass)  │     │(wins!) │
            └────────┘       └────────┘     └────────┘

Result: kitchen=0xA0, bedroom=0xFF, hallway=off
```

### Interpolation Type Promotion

```
          1 octet          2 octets          3 octets
        ┌──────────┐    ┌──────────────┐   ┌────────────────┐
        │brightness│───▶│bright + temp │──▶│ H + S + B      │
        │  (BB)    │    │  (BB TT)     │   │ (HH SS BB)     │
        └──────────┘    └──────────────┘   └────────────────┘
              │                                    ▲
              └────────────────────────────────────┘
                    (promote: assume neutral H,S)
```

---

## Verification Checklist

- [ ] Walk through example patterns manually against the formal grammar
- [ ] Verify interpolation formulas produce correct intermediate values
- [ ] Produce concrete intermediate values for representative patterns
- [ ] Check that the ambiguity catalog covers all edge cases from the user's spec
- [ ] Grammar ensures document uses Python or classical math notation as requested
