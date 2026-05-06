# Gesture & spring reference

Drag, swipe, and the spring physics that make them feel right. Read this when building drawers, sheets, swipe-to-dismiss toasts, or any pointer-driven interaction.

## Contents

1. [When to reach for springs](#when-to-reach-for-springs)
2. [Momentum-based dismissal](#momentum-based-dismissal)
3. [Damping at boundaries](#damping-at-boundaries)
4. [Pointer capture](#pointer-capture)
5. [Multi-touch protection](#multi-touch-protection)
6. [Friction over hard stops](#friction-over-hard-stops)
7. [Motion drag in React](#motion-drag-in-react)
8. [Motion drag in Vue](#motion-drag-in-vue)
9. [Interruptibility — why springs not keyframes](#interruptibility--why-springs-not-keyframes)

## When to reach for springs

Springs feel natural because they simulate physics. They have no fixed duration; they settle based on stiffness and damping. Use them for:

- **Drag interactions with momentum** — flick a drawer, watch it continue.
- **Elements that should feel "alive"** — Apple's Dynamic Island.
- **Gestures that can be interrupted** — open mid-animation, reverse smoothly.
- **Decorative mouse-tracking** — interpolate cursor-driven values so they don't feel artificial.

Don't use springs for:

- Modal/popover open/close where timing should be predictable.
- Anything driven by `prefers-reduced-motion`.
- Loading spinners (linear, please).

## Momentum-based dismissal

Don't require dragging past a fixed distance threshold. Calculate velocity: a quick flick should dismiss regardless of how far it traveled.

```js
const elapsed = performance.now() - dragStartTime;
const velocity = Math.abs(swipeAmount) / elapsed;

const SWIPE_DISTANCE = 80;     // pixels
const VELOCITY_THRESHOLD = 0.11; // px/ms

if (Math.abs(swipeAmount) >= SWIPE_DISTANCE || velocity > VELOCITY_THRESHOLD) {
  dismiss();
}
```

The `0.11` threshold is empirical — it's what Sonner uses and what feels natural for swipe-to-dismiss toasts. Tune for your component's size and weight.

## Damping at boundaries

When a user drags past a natural boundary (e.g., dragging a drawer up when it's already at the top), apply damping. The more they drag, the less the element moves. Things in real life don't suddenly stop; they slow down first.

```js
function applyDamping(distance, max) {
  if (distance <= max) return distance;
  const overshoot = distance - max;
  return max + overshoot * 0.3; // 70% friction past the boundary
}
```

The exact factor (0.2–0.4) is taste. Looser feels rubbery; tighter feels confined.

## Pointer capture

Once dragging starts, capture pointer events on the element. This ensures dragging continues even if the pointer leaves the element bounds — which is otherwise a constant source of "drag stopped working" bugs.

```js
function onPointerDown(e) {
  e.currentTarget.setPointerCapture(e.pointerId);
  isDragging.current = true;
}
```

Release on `pointerup` and `pointercancel`.

## Multi-touch protection

Ignore additional touch points after the initial drag begins. Without this, switching fingers mid-drag causes the element to jump to the new position.

```js
function onPress() {
  if (isDragging.current) return; // drop the second finger
  // start drag…
}
```

## Friction over hard stops

Allow drag in the "wrong" direction (e.g., down when the drawer is already at bottom), but with increasing friction. This feels more natural than hitting an invisible wall, and gives users tactile feedback that they've reached the end.

This is the same idea as damping at boundaries, applied to the *opposite* direction from the dismiss gesture.

## Motion drag in React

```jsx
import { motion } from "motion/react";

<motion.div
  drag="y"
  dragConstraints={{ top: 0, bottom: 0 }}     // snap back when released
  dragElastic={0.2}                            // damping past constraints
  onDragEnd={(_, info) => {
    if (info.offset.y > 80 || info.velocity.y > 500) {
      onDismiss();
    }
  }}
/>
```

`dragElastic` between 0 and 1 controls how much overshoot is allowed past constraints. `0.2` is a reasonable default for "feels alive but bounded".

For free-form drag (drawers, sheets) where you want to update position during drag, drive a `MotionValue` and animate it back via spring on release.

## Motion drag in Vue

```vue
<script setup>
import { motion } from "motion-v"

function onDragEnd(_, info) {
  if (info.offset.y > 80 || info.velocity.y > 500) {
    onDismiss()
  }
}
</script>

<template>
  <motion.div
    drag="y"
    :drag-constraints="{ top: 0, bottom: 0 }"
    :drag-elastic="0.2"
    @drag-end="onDragEnd"
  />
</template>
```

Note the kebab-case prop bindings (`drag-constraints`, `drag-elastic`) and event binding (`@drag-end`). Otherwise the API matches React.

## Interruptibility — why springs not keyframes

Springs maintain velocity when interrupted. CSS `@keyframes` and JS animations driven by `requestAnimationFrame` over a fixed duration restart from zero on interruption.

**Concrete example:** A sheet is animating open over 300ms. At 200ms, the user taps the close button. With a keyframed close: the sheet jumps to wherever the close animation expects it to start, then plays out — visibly janky. With a spring: the spring's current velocity becomes the close animation's initial velocity, and it eases out from the current position.

This is why every drag-driven UI element should be on a spring. The user *will* change their mind mid-gesture. Whether that feels smooth or broken is the difference between a beloved interaction and a tolerated one.

### Spring config (Apple-style)

```js
{ type: "spring", duration: 0.5, bounce: 0.2 }
```

For drag-to-dismiss specifically, slightly higher bounce (0.2–0.3) feels playful and alive without being silly. For drawers in a banking app, drop bounce to 0.1 or 0 — `{ duration: 0.4, bounce: 0 }` is critically damped and feels professional.
