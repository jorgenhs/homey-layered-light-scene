# Keyframe Animation System - Design Document Plan

## Context

The user has designed a grammar for a keyframe-based animation system for their Homey smart home light controller (`com.fabel.yet.layeredlight`). The existing app supports static scene layering with priorities. The new system adds time-based animations (keyframes) with interpolation.

**Task:** Produce a detailed technical design document (`KEYFRAME_DESIGN.md`) with formal grammar, mathematical model, diagrams, and ambiguity analysis. No code changes — design only.

## Design Decisions (Confirmed by user)

1. **Grammar disambiguation:** Rename inner `setting` to `value` to avoid overloading.
2. **Null interpolation:** Fade to lower layer — gradually blend from this layer's value toward the lower layer's resolved value over the transition duration.
3. **Mid-animation reassignment:** Snapshot the old animation's current computed value at reassignment time; use it as "previous value" if the new pattern starts with a transition.
4. **Type mixing in patterns:** Allowed — convert to a common representation for interpolation (e.g., expand brightness-only to full HSL with neutral hue/sat).

## Document Outline (`KEYFRAME_DESIGN.md`)

### 1. Formal Grammar
- Enhanced/corrected EBNF with `light_assignment` (top level) vs `value` (keyframe value)
- Lexer rules: how `f`, `1`, `:`, hex strings, durations, and keywords are tokenized
- Identified ambiguities with resolution rules
- Parse tree examples for representative patterns

### 2. Time Model & Keyframe Evaluation
- Mathematical definition of `eval(pattern, t_assign, t_now)` → `value | null`
- A looping timeline: walk keyframes, find active segment
- Looping timeline: modular arithmetic on cycle duration
- Leading transition: use snapshot of previous value
- Trailing transition (loop marker): wraps to first value
- Python/math pseudocode for the evaluator

### 3. Interpolation
- Type-type interpolation: HSL (with hue wrapping), temp+brightness, brightness, boolean (on/off only)
- Type promotion rules for mixed-type patterns (e.g., brightness + HSL)
- Null blending: requires resolving the lower layer's value, then interpolating toward it
- Mathematical formula: `lerp, step function, hue shortest-arc`

### 4. Layer Flattening with Animations
- At time t, each layer evaluates all its patterns → produces a scene (with possible nulls)
- Flattening iterates by priority, overlays non-null values (same as existing `Layer+Scene`)
- Null = transparent at this moment, lower layer shines through
- Diagram: layer stack evaluation at a point in time

### 5. Output Command Generation
- Map flattened values to device capability classes (`onoff`, `dim`, `dim+temp`, full HSL)
- Linear interpolation delegation: set current value immediately, set target with duration
- Step transitions: set value after duration elapses (or set immediately at next keyframe tick)

### 6. Optimizer
- `known_state` map per light: `{value, timestamp}`
- Skip commands where `known_state == target` (within epsilon)
- Staleness handling: if timestamp too old, issue command anyway
- Redundancy detection for transitions: if light is already transitioning to same target, skip
- Race condition discussion: external changes invalidate state

### 7. Ambiguities & Edge Cases Catalog
- Grammar ambiguities (hex vs keyword boundaries, etc.)
- Null interpolation mechanics (requires cross-layer evaluation)
- Empty patterns, zero-duration transitions
- Reassignment during looping animation (phase reset?)
- That-ever-assignment with leading transition (no previous value → default? black/off?)
- Evaluation cadence: how often to tick the system

## File to Create

`/home/user/homey-layered-light-scene/KEYFRAME_DESIGN.md`

## Verification

- Walk through example patterns manually against the formal grammar
- Verify interpolation formulas produce correct intermediate values
- Check that the ambiguity catalog covers all edge cases from the user's grammar
- Ensure document uses Python/maths notation as requested
