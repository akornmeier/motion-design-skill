# Vue reference (Motion for Vue)

Use **`motion-v`** — the Vue 3 port of Motion. The API mirrors Motion for React closely; the imports differ.

```bash
npm install motion-v
```

```js
import { Motion, AnimatePresence, useSpring, useReducedMotion, useMotionValue, useInView } from "motion-v";
```

## Contents

1. [The two component forms](#the-two-component-forms)
2. [`AnimatePresence` for exits](#animatepresence-for-exits)
3. [Springs and motion values](#springs-and-motion-values)
4. [`useReducedMotion`](#usereducedmotion)
5. [`useInView` for reveal-on-scroll](#useinview-for-reveal-on-scroll)
6. [Composing with headless component libraries (Reka UI, Radix Vue)](#composing-with-headless-component-libraries)
7. [Hardware-acceleration caveat](#hardware-acceleration-caveat)
8. [What is *not* the same as React](#what-is-not-the-same-as-react)

## The two component forms

`motion-v` ships both PascalCase and dot-namespaced forms. Pick whichever reads better in your project, but be consistent.

```vue
<script setup>
import { Motion, motion } from "motion-v"
</script>

<template>
  <!-- PascalCase — works for any HTML tag via the `as` prop -->
  <Motion
    as="div"
    :initial="{ opacity: 0, y: 8 }"
    :animate="{ opacity: 1, y: 0 }"
    :transition="{ duration: 0.18, ease: [0.23, 1, 0.32, 1] }"
  />

  <!-- Or the dot-namespaced form, identical to React -->
  <motion.div
    :initial="{ opacity: 0, y: 8 }"
    :animate="{ opacity: 1, y: 0 }"
  />
</template>
```

Bind props with `:` and pass objects directly. `transition` follows Motion's standard shape.

## `AnimatePresence` for exits

Wrap conditionally-rendered children (`v-if`) so exit animations run before unmount.

```vue
<script setup>
import { Motion, AnimatePresence } from "motion-v"
import { ref } from "vue"

const isOpen = ref(false)
</script>

<template>
  <AnimatePresence>
    <Motion
      v-if="isOpen"
      key="modal"
      :initial="{ opacity: 0, scale: 0.95 }"
      :animate="{ opacity: 1, scale: 1 }"
      :exit="{ opacity: 0, scale: 0.95 }"
      :transition="{ duration: 0.18, ease: [0.23, 1, 0.32, 1] }"
    />
  </AnimatePresence>
</template>
```

Each child needs a stable, unique `key` — without one, exits don't fire.

## Springs and motion values

Tying visual changes directly to mouse position feels artificial. Use `useSpring` to interpolate.

```vue
<script setup>
import { motion, useMotionValue, useSpring } from "motion-v"

const x = useMotionValue(0)
const rotation = useSpring(x, { stiffness: 100, damping: 10 })

function onMove(e) {
  x.set(e.clientX * 0.1)
}
</script>

<template>
  <motion.div :style="{ rotate: rotation }" @mousemove="onMove" />
</template>
```

`useMotionValue` updates the DOM directly without triggering Vue reactivity — that is the point. Reading `.value` from inside reactive code is fine; Motion handles the bridge.

### Spring config (Apple-style)

```js
{ type: "spring", duration: 0.5, bounce: 0.2 }
```

Easier to reason about than `mass`/`stiffness`/`damping`. Keep `bounce` subtle (0.1–0.3) and reserve it for drag-to-dismiss / playful moments.

## `useReducedMotion`

`useReducedMotion()` returns a `Ref<boolean>`. Use `.value` directly or wrap in `computed`.

```vue
<script setup>
import { Motion, useReducedMotion } from "motion-v"
import { computed } from "vue"

const props = defineProps({ isOpen: Boolean })
const shouldReduceMotion = useReducedMotion()
const closedX = computed(() => (shouldReduceMotion.value ? 0 : "-100%"))
</script>

<template>
  <Motion
    :animate="{
      opacity: isOpen ? 1 : 0,
      x: isOpen ? 0 : closedX,
    }"
  />
</template>
```

Reduced motion ≠ no motion. Keep opacity transitions; drop transforms.

## `useInView` for reveal-on-scroll

```vue
<script setup>
import { Motion, useInView } from "motion-v"
import { ref } from "vue"

const el = ref(null)
const inView = useInView(el, { once: true, margin: "-100px" })
</script>

<template>
  <Motion
    ref="el"
    :initial="{ opacity: 0, y: 24 }"
    :animate="inView ? { opacity: 1, y: 0 } : undefined"
    :transition="{ duration: 0.4, ease: [0.23, 1, 0.32, 1] }"
  >
    <slot />
  </Motion>
</template>
```

`{ once: true }` prevents replay on re-entry. `margin: "-100px"` triggers slightly before the element is on screen so the user perceives motion already in progress.

## Composing with headless component libraries

Reka UI (Radix Vue's successor) and shadcn-vue both work cleanly with `motion-v`. The pattern is to use the headless component's portal and trigger machinery, then `as-child` to forward the ref to a `motion.div`.

```vue
<Tooltip.Portal>
  <AnimatePresence>
    <Tooltip.Content as-child :side-offset="10">
      <motion.div
        :initial="{ opacity: 0, y: 8, scale: 0.95 }"
        :animate="{ opacity: 1, y: 0, scale: 1 }"
        :exit="{ opacity: 0, y: 8 }"
        :transition="{ duration: 0.18, ease: [0.23, 1, 0.32, 1] }"
      >
        <slot />
      </motion.div>
    </Tooltip.Content>
  </AnimatePresence>
</Tooltip.Portal>
```

If the user is on shadcn-vue, the same pattern applies — replace `Tooltip` with whichever Reka primitive they're using.

For pure-CSS state-driven animation, prefer Tailwind's `data-[state=...]` utilities (see `references/tailwind.md`). Use `motion-v` when you need imperative spring physics, gestures, or layout animations.

## Hardware-acceleration caveat

Same as React: Motion's shorthand props (`x`, `y`, `scale`) are not hardware-accelerated. Under main-thread load, prefer the full `transform` string or CSS animations.

```vue
<!-- NOT hardware accelerated. Drops frames when main thread is busy. -->
<motion.div :animate="{ x: 100 }" />

<!-- Hardware accelerated. -->
<motion.div :animate="{ transform: 'translateX(100px)' }" />
```

For predetermined entrances during page transitions, use Tailwind's `starting:` variant + CSS transitions instead of Motion. Reserve `motion-v` for interruptible / gesture-driven cases.

## What is *not* the same as React

- **No JSX.** Use the template syntax, prop binding (`:`), and `v-if` / `v-for`.
- **`useMotionValue` returns a Vue-friendly object** — not a React ref. Use `.set()` to write, `.value` to read.
- **`as-child` (when composing with Reka)** uses Vue's slot forwarding — not React's `Slot`/`asChild`. The mental model is the same; the wiring differs.
- **`v-for` with `AnimatePresence`** — each iterated child still needs a stable `:key`. The keying rules are identical to plain Vue lists.
