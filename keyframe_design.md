# Keyframe Animation System — Design Document

## 1. Context

The existing `LightLayers` system (`lightlayers.ts`) provides static scene layering: each layer maps light names to constant color/brightness values. Layers are stacked by priority, flattened, diffed against previous state, and applied.

This document extends the system with **time-based keyframe animations**. A layer's value for a light is no longer just a constant — it can be a sequence of settings separated by transitions, with durations and interpolation modes.

---

## 2. Formal Grammar

### 2.1 Full Grammar (EBNF)

```
layer       := setting ( ws+ setting )*
setting     := light_name ":" pattern
light_name  := [^:]+                        -- greedy up to ":"
pattern     := transition? ( transition setting_val )* transition?
transition  := "/" duration | "[" duration "]" | "/" | "[" "]"
              -- "/" = linear interpolation
              -- "[" "]" = step function (jump to value at end)
duration    := number unit?
number      := [0-9]+ ( "." [0-9]+ )?
unit        := "ms" | "s" | "m"             -- default: seconds
setting_val := octet{3} | octet{2} | octet{1} | "on" | "off" | "null"
octet       := [a-fA-F0-9]{2}
ws          := " " | "\t"
```

### 2.2 Tokenization

The input string is first split by `:` (same as existing code). Then each segment is tokenized into:

| Token type     | Regex                        | Examples              |
|----------------|------------------------------|-----------------------|
| `LERP`         | `/(\d+(\.\d+)?(ms|s|m)?)?`  | `/`, `/2s`, `/500ms`  |
| `STEP`         | `\[(\d+(\.\d+)?(ms|s|m)?)?\]` | `[]`, `[1s]`, `[200ms]` |
| `VALUE`        | `[a-fA-F0-9]{2,6}`          | `ff`, `00ff`, `ff8800`|
| `ON`           | `on`                         | `on`                  |
| `OFF`          | `off`                        | `off`                 |
| `NULL`         | `null`                       | `null`                |

Keywords are matched before hex values to avoid ambiguity (e.g. `off` is not hex `0ff`).

### 2.3 Examples

```
# Static (no animation, backwards compatible):
Lamp1:ff0000 Lamp2:00ff

# Fade from red to blue over 2 seconds, then hold:
Lamp1:ff0000/2s0000ff

# Pulse: red -> (1s fade) -> blue -> (1s fade) -> red -> loop
Lamp1:ff0000/1s0000ff/1s

# Step function: instantly switch to blue after 3 seconds:
Lamp1:ff0000[3s]0000ff

# Fade in from off over 5 seconds:
Lamp1:/5sff

# Mixed transitions:
Lamp1:ff0000/2s00ff00[1s]0000ff/0.5s
```

### 2.4 Parse Tree Structure

```python
@dataclass
class Keyframe:
    value: list[float] | bool | None    # parsed setting value
    transition: Transition | None        # how to arrive at this keyframe

@dataclass
class Transition:
    mode: 'lerp' | 'step'
    duration_s: float                    # duration in seconds

@dataclass
class LightPattern:
    keyframes: list[Keyframe]
    loops: bool                          # true if pattern ends with a transition
```

A parsed pattern always starts with a value or a transition-to-first-value, and alternates `[transition, value, transition, value, ...]`, optionally ending with a trailing transition (indicating a loop back to the first keyframe).

---

## 3. Time Model & Keyframe Evaluation

### 3.1 Timeline

When a layer is assigned at time `t_assign`, the pattern begins playing:

```
t_assign                                          t_now
  |----[transition 0]---->V0----[transition 1]---->V1----[transition 2]---->V2...
```

Each keyframe has an **absolute start time** computed from `t_assign` plus the cumulative durations of all preceding transitions.

### 3.2 Evaluation: `eval(pattern, t_assign, t_now)`

```python
def eval(pattern, t_assign, t_now) -> value | null:
    """Evaluate a light pattern at a given time."""
    keyframes = pattern.keyframes

    # Build timeline: list of (abs_time, value, transition_to_next)
    timeline = []
    t = t_assign
    for kf in keyframes:
        if kf.transition:
            t += kf.transition.duration_s
        timeline.append((t, kf.value, None))

    # Link transitions
    for i in range(len(timeline) - 1):
        timeline[i] = (*timeline[i][:2], keyframes[i+1].transition)

    # Total cycle duration
    total_duration = t - t_assign
    if pattern.loops and total_duration > 0:
        # Add the trailing loop transition duration
        loop_transition = pattern.trailing_transition
        cycle = total_duration + loop_transition.duration_s
        elapsed = (t_now - t_assign) % cycle
    else:
        elapsed = t_now - t_assign

    t_eval = t_assign + elapsed

    # Find active segment
    for i in range(len(timeline) - 1):
        t_start, v_start, trans = timeline[i]
        t_end, v_end, _ = timeline[i + 1]
        if t_start <= t_eval < t_end and trans:
            # In a transition
            progress = (t_eval - t_start) / trans.duration_s
            if trans.mode == 'lerp':
                return interpolate(v_start, v_end, progress)
            else:  # step
                return v_start  # hold until end
        elif t_eval >= t_start and (i == len(timeline)-1 or t_eval < timeline[i+1][0]):
            return v_start

    # If looping, handle wrap-around transition
    if pattern.loops:
        t_last = timeline[-1][0]
        v_last = timeline[-1][1]
        v_first = timeline[0][1]
        loop_trans = pattern.trailing_transition
        progress = (t_eval - t_last) / loop_trans.duration_s
        if loop_trans.mode == 'lerp':
            return interpolate(v_last, v_first, progress)
        else:
            return v_last

    # Non-looping: hold last value
    return timeline[-1][1]
```

### 3.3 Looping

A pattern **loops** if and only if it ends with a trailing transition (a `/` or `[]` after the last value). The loop returns to the first keyframe.

```
Non-looping:  V0 /2s V1         -- fades to V1, holds forever
Looping:      V0 /2s V1 /2s    -- fades V0->V1->V0->V1->... (4s cycle)
```

### 3.4 Leading Transition

If a pattern starts with a transition (no initial value), the initial value is **the previous resolved value** of the light from the layer below, or the current actual state if no lower layer exists.

```
/5s ff    -- transition FROM current state TO ff over 5 seconds
```

This is resolved at `t_assign` by snapshotting the light's previous value.

---

## 4. Interpolation

### 4.1 Color Space for Interpolation

All interpolation is performed in the **output color space**, which depends on the value type:

| Value length | Meaning                     | Interpolation space          |
|-------------|-----------------------------|-----------------------------|
| 3 floats    | `[hue, saturation, lightness]` | HSL (with hue wrapping)     |
| 2 floats    | `[temperature, brightness]` | Linear per-component        |
| 1 float     | `[brightness]`              | Linear                      |
| boolean     | `on` / `off`                | Step only (no lerp)         |
| null        | Inherit / transparent       | See §4.3                    |

### 4.2 Interpolation Formula

For linear interpolation (`/`):

```
interpolate(a, b, t) where t ∈ [0, 1]:
    result[i] = a[i] + (b[i] - a[i]) * t    for each component i
```

**Special case — hue wrapping:**
Hue is circular on `[0, 1)`. Choose the shorter arc:

```python
def lerp_hue(h0, h1, t):
    delta = h1 - h0
    if abs(delta) > 0.5:
        # wrap around
        if delta > 0:
            h0 += 1.0
        else:
            h1 += 1.0
        result = h0 + (h1 - h0) * t
        return result % 1.0
    return h0 + delta * t
```

For step functions (`[]`):

```
step(a, b, t) where t ∈ [0, 1):
    return a          -- hold previous value until transition ends
    # at t = 1.0, jump to b
```

### 4.3 Null Handling in Interpolation

`null` means "inherit from below" — it is **transparent** in the layer stack.

- **Transition TO null:** Fade out this layer's contribution. The interpolation target is the value from the layer below (resolved at the moment the transition begins).
- **Transition FROM null:** Fade in. The starting value is resolved from the layer below at `t_assign`.
- **Both null:** No-op; layer is fully transparent.

### 4.4 Type Promotion Rules

When interpolating between values of **different types**, promote to the more expressive type:

| From → To               | Rule                                          |
|-------------------------|-----------------------------------------------|
| brightness → temp+brightness | Assume temperature = 0 (warmest)          |
| brightness → HSL        | Assume H=0, S=0 (white light at given brightness) |
| temp+brightness → HSL   | Convert to neutral HSL                        |
| boolean → numeric       | `on` = brightness 1.0, `off` = brightness 0.0 |

---

## 5. Layer Flattening with Animations

### 5.1 Static Flattening (existing)

The existing system flattens layers by priority: for each light, the highest-priority non-null value wins.

### 5.2 Animated Flattening

At time `t`, each layer evaluates its pattern for each light to produce a **frame**. The flattening then proceeds as before:

```python
def flatten_animated(stack, priorities, t_now):
    """Flatten the layer stack at time t_now."""
    result = {}
    for layer_name in priorities:  # low to high priority
        layer = stack[layer_name]
        for light_name, pattern in layer.items():
            value = eval(pattern, layer.t_assign, t_now)
            if value is not None:
                result[light_name] = value
            # null = transparent, don't override
    return result
```

### 5.3 Layer Assignment Semantics

When a layer is (re)assigned:

1. The **assignment time** `t_assign` is recorded
2. The layer's patterns replace (or merge with) the existing layer content
3. The previous resolved values are **snapshotted** for any leading transitions
4. `clear=true` replaces the entire layer; `clear=false` merges (existing behavior)

**Key insight:** Reassigning a layer **dirties** the current and target values for any light that was already animating. The new pattern starts from `t_assign` with a fresh timeline.

### 5.4 Diagram: Layer Stack Over Time

```
Priority ↑
          │
  Layer C │  ████████████████████  (static: bright white)
          │
  Layer B │  ░░░░/===\░░░░░/===\  (animating: pulse red↔blue)
          │       ^
          │       t_assign_B
  Layer A │  ████████████████████  (static: warm dim)
          │
          └──────────────────────────────→ time
```

At each tick, evaluate B's animation at `t_now`, then flatten:
- Lights defined in C take C's value (highest priority)
- Lights defined only in B take B's animated value
- Lights defined only in A take A's static value
- Lights in both A and B: B wins (higher priority)

---

## 6. Output Command Generation

### 6.1 Capability Classes

The Homey API exposes different capability sets per device. The output layer must map abstract values to concrete API calls:

| Value type            | Required capabilities              | Commands                              |
|----------------------|-----------------------------------|---------------------------------------|
| `[h, s, l]` (HSL)   | `onoff`, `dim`, `light_hue`, `light_saturation` | Set all four                |
| `[temp, brightness]` | `onoff`, `dim`, `light_temperature`| Set all three                         |
| `[brightness]`       | `onoff`, `dim`                    | Set both                              |
| `on` / `off`         | `onoff`                           | Set `onoff` only                      |
| `null` (turn off)    | `onoff`                           | Set `onoff = false`                   |

### 6.2 Interpolation as Output Commands

For linear interpolation, the system sets:
1. The **current value immediately** (at `t_now`)
2. The **target value with a duration** — the device firmware handles the smooth transition

```python
def emit_transition_command(device, current_value, target_value, remaining_duration_s):
    """Tell the device to transition from current to target over remaining time."""
    # Set current value instantly (duration=0) to sync state
    set_capability(device, current_value, duration=0)
    # Set target with transition duration
    set_capability(device, target_value, duration=remaining_duration_s)
```

This leverages the Homey device's built-in transition support where available. For devices without firmware transitions, the system must emit intermediate values at a tick rate (see §6.3).

### 6.3 Tick-Based Fallback

For devices that don't support duration-based transitions:

```python
TICK_INTERVAL_MS = 500  # configurable

def tick_loop(stack, priorities, lights):
    while has_active_animations(stack):
        t_now = current_time()
        scene = flatten_animated(stack, priorities, t_now)
        changes = get_changes(previous_scene, scene)
        apply_scene(lights, changes)
        previous_scene = scene
        wait(TICK_INTERVAL_MS)
```

---

## 7. Optimizer

### 7.1 Purpose

The optimizer sits between the evaluator output and the device API. It prevents redundant commands and handles state synchronization issues.

### 7.2 Known State Tracking

```python
class LightState:
    """Tracked state for a single light."""
    known_values: dict[str, float | bool]  # capability -> last sent value
    last_update_t: float                     # when we last sent a command
    dirty: bool                              # external change suspected
```

### 7.3 Optimization Rules

```python
def optimize(light, target_value, known_state):
    """Decide whether to emit a command."""

    # 1. Skip if identical to known state (within epsilon)
    if is_equal_within_epsilon(target_value, known_state.known_values, epsilon=0.005):
        return SKIP

    # 2. Always send if state is dirty (external change detected)
    if known_state.dirty:
        return SEND

    # 3. Staleness: re-send if no command sent in > threshold
    if (t_now - known_state.last_update_t) > STALENESS_THRESHOLD:
        return SEND

    # 4. Redundancy detection for transitions:
    #    If the device is already transitioning to the correct target,
    #    don't interrupt it
    if known_state.active_transition_target == target_value:
        return SKIP

    return SEND
```

### 7.4 Race Conditions

The current state of lights may be updated by **external sources** (other Homey flows, physical switches, other apps) at any time. The optimizer must handle this:

1. **Optimistic tracking:** Assume our commands succeed. Update `known_state` when we send a command.
2. **Periodic re-sync:** The existing "HACK: Reapplying the full scene an extra time" is a form of this. For animations, a periodic full re-send (e.g. every 30s) ensures convergence.
3. **Event-based invalidation:** If Homey provides capability change events, mark the light as `dirty` when an unexpected change is observed.

---

## 8. Ambiguities & Edge Cases

### 8.1 Grammar Ambiguities

| Ambiguity | Description | Resolution |
|-----------|-------------|------------|
| **`on` as hex** | `on` could be parsed as hex `0n` — but `n` is not hex | Not ambiguous: `on` is keyword |
| **`off` as hex** | `off` is not valid hex (only 3 chars, not 2/4/6) | Not ambiguous: `off` is keyword |
| **Bare `/`** | `/` without duration — what duration? | Default duration (e.g. 1s), configurable |
| **Empty `[]`** | `[]` without duration — what duration? | Zero duration = instant jump |
| **Adjacent transitions** | `ff/2s/3s00` — two transitions with no value between | **Invalid.** Transitions must alternate with values |
| **Keyword boundary** | `offline` — is this `off` + `line`? | Keywords are only matched as whole tokens separated by transitions or whitespace |

### 8.2 Semantic Edge Cases

| Case | Behavior |
|------|----------|
| **Empty pattern** | `Lamp1:` — layer becomes `null` (transparent) for this light |
| **Zero-duration transition** | `/0s` or `/0` — instant jump (equivalent to step) |
| **Single value, no transition** | `Lamp1:ff` — static, backwards-compatible with existing system |
| **Null in animation** | `Lamp1:ff/2snull` — fade to transparent (layer below shines through) |
| **Reassignment during loop** | New pattern starts immediately; old animation is discarded |
| **Reassignment during transition** | Snapshot current interpolated value as starting point if new pattern has a leading transition |
| **Boolean interpolation** | `on/2soff` — **promote to brightness**: 1.0 → 0.0 fade over 2s |
| **Type mismatch across keyframes** | `ff0000/2s80` (HSL → brightness) — promote brightness to HSL with S=0 |
| **Layer removal** | Removing a layer mid-animation: re-flatten, diff, apply |

### 8.3 Zero-duration Patterns and Empty Transitions

```
Lamp1:ff/00    -- ff, then instant transition to 00 = same as step
Lamp1:ff[]00   -- same: step from ff to 00
Lamp1:/ff      -- transition from current to ff, default duration
Lamp1:ff/      -- ff then loop back to ff (no-op loop)
```

---

## 9. Integration with Existing System

### 9.1 Backwards Compatibility

All existing static scene strings remain valid. A pattern with no transitions is simply a single-keyframe pattern that evaluates to a constant. The `getSceneFromString()` method can be extended or wrapped:

```python
def parse_setting(raw: str) -> LightPattern:
    tokens = tokenize(raw)
    if has_transitions(tokens):
        return parse_animated_pattern(tokens)
    else:
        return LightPattern(
            keyframes=[Keyframe(value=parse_static_value(raw), transition=None)],
            loops=False
        )
```

### 9.2 Storage Changes

The scene stack currently stores JSON-serialized static scenes. With animations, each layer entry must also store:

```typescript
interface AnimatedLayerEntry {
    patterns: { [lightName: string]: LightPattern };
    t_assign: number;           // Date.now() at assignment time
    raw_scene_string: string;   // original string for debugging
}
```

The flow token (stack) must serialize this richer structure.

### 9.3 New Execution Loop

The current system is **event-driven**: apply once when a flow triggers. With animations, a **tick loop** is needed:

```typescript
class AnimationEngine {
    private intervalId: NodeJS.Timeout | null = null;
    private tickMs: number = 500;

    start() {
        this.intervalId = setInterval(() => this.tick(), this.tickMs);
    }

    stop() {
        if (this.intervalId) clearInterval(this.intervalId);
    }

    private async tick() {
        const t_now = Date.now();
        const scene = this.flattenAnimated(t_now);
        const changes = this.getChanges(this.previousScene, scene);
        if (Object.keys(changes).length > 0) {
            await this.applyScene(this.lights, changes);
            this.previousScene = scene;
        }

        // Stop ticking if no active animations remain
        if (!this.hasActiveAnimations()) {
            this.stop();
        }
    }
}
```

---

## 10. Summary

The keyframe system extends the existing layered light scene model with:

1. **A pattern grammar** that is backwards-compatible with static values
2. **Time-aware evaluation** with linear and step interpolation
3. **Looping support** via trailing transitions
4. **Layer-aware flattening** that evaluates animations per-layer before merging
5. **An optimizer** that minimizes redundant API calls given potentially stale state
6. **A tick-based execution loop** that drives animations forward

The core abstractions — layers, priority-based flattening, and diff-based application — remain unchanged. The animation layer is additive: it enriches what a "value" can be without altering the composition model.
