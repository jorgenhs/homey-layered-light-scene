# Designing Keyframe System for Lights

## Grammar

```
layer := <setting>(\s+<setting>)*
setting := <light name>:<pattern>
light name := [^:]+
pattern := <transition>?<setting>(<transition><setting>)*<transition>?
transition := (/<duration>/|[|]<duration>[|])
setting := (<octet>{1,3}|on|off|null)
octet := [a-f0-9]{2}
duration := [0-9]+(ms|s|m)
```

## Meaning

A list of lights is assigned a pattern. The pattern may be a constant or an animation.

If the animation starts with a transition, that is a transition from whatever previous setting the light had. If there are multiple settings separated by transitions, the light will transition between these settings. If the pattern ends with a transition, this indicates a loop, and it will transition into the first setting again. If it ends on a setting and no transition, the light will stay on that setting until a new pattern is applied.

Durations are by default in seconds, unless a unit is specified. Transitions with `/` are linear interpolations, while `|` indicates a step function.

### Settings of one octet
Simply brightness.

### Settings of two octets
Brightness and temperature.

### Settings of three octets
Indicate r g b values.

### Special values
- `off` and `on` indicate the special on and off settings.
- `null` indicates "inherit from lower priority layer or default value".

**CAVEAT:** This could be a moving target, and change from one frame of computation to another.

## Layering

The system supports assigning a layer value to one of several layer names. Layer priorities are defined as an ordered list of layer names, from high to low.

Default value is black, off.

## System Design

I want to design a system that allows assigning layers to a layer stack, at specific times, and honors the assignment time, and allows stepping through the keyframes. At each keyframe it can compute, for each light, the final current value and target value, and knows how to interpolate between them (of course target value is irrelevant for step functions - until the next keyframe comes into effect). It honors null settings and allows the layer below to "shine through". Reassigning a layer may dirty current value and target.

## Output Commands

The output commands should be light settings, depending on light capabilities:

- Classes are on/off, brightness only, brightness+temperature, and full hue+sat+brightness.
- A linear interpolation is done by setting the current value immediately, and the target value with a duration, specifying the time to take to reach the target value.

## Optimizer

These settings should pass through an optimizer step, which considers the current state of the lights, and skips redundant commands.

The current state of lights may be updated at entirely uncoordinated times, and the optimizer must do a best effort at issuing the right commands given the known light state.

## Request

Please draw up a detailed explanation and diagrams for the keyframing system, and identify ambiguities in grammar and behaviour.

Use Python or classical maths notation if pseudo code is needed.
