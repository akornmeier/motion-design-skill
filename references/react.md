# React reference (Motion for React)

Use `motion/react`. **Do not use `framer-motion`** — that package is the legacy name. The API is the same; the import path changed. If the user's code imports from `framer-motion`, flag it as a migration item.

```bash
npm install motion
```

```js
import { motion, AnimatePresence, useSpring, useReducedMotion, useInView } from "motion/react";
```

## Contents

1. [Migration: `framer-motion` → `motion/react`](#migration-framer-motion--motionreact)
2. [Hardware-acceleration caveat (this matters)](#hardware-acceleration-caveat-this-matters)
3. [Springs for mouse/decorative motion](#springs-for-mousedecorative-motion)
4. [`useReducedMotion`](#usereducedmotion)
5. [`useInView` for reveal-on-scroll](#useinview-for-reveal-on-scroll)
6. [Mount animation patterns](#mount-animation-patterns)
7. [`AnimatePresence` for exits](#animatepresence-for-exits)

## Migration: `framer-motion` → `motion/react`

Symptoms in legacy code: imports from `"framer-motion"`. Fix:

```diff
- import { motion, useSpring } from "framer-motion";
+ import { motion, useSpring } from "motion/react";
```

Package change in `package.json`:

```diff
- "framer-motion": "^11.x"
+ "motion": "^12.x"
```

The component/hook APIs are unchanged. If a project pins to a very old `framer-motion`, run the upgrade against Motion's migration notes — the breaking changes between Framer Motion v11 and Motion v12 are minimal but real (mostly around `LayoutGroup` defaults).

## Hardware-acceleration caveat (this matters)

Motion's shorthand props (`x`, `y`, `scale`) are **not** hardware-accelerated. They run on the main thread via `requestAnimationFrame`. For hardware acceleration, use the full `transform` string.

```jsx
// NOT hardware accelerated. Convenient. Drops frames when main thread is busy.
<motion.div animate={{ x: 100 }} />

// Hardware accelerated. Stays smooth even during page loads.
<motion.div animate={{ transform: "translateX(100px)" }} />
```

This matters when the browser is simultaneously loading content, running scripts, or painting. The Vercel dashboard tab animation used Shared Layout Animations and dropped frames during page loads — switching to CSS animations (off main thread) fixed it.

**Heuristic:** for predetermined animations the user will see during a transitional moment (route change, modal open while page is still loading), prefer CSS or `transform` strings. Reserve `x`/`y` for interruptible, gesture-driven animations where the main thread is the right place.

## Springs for mouse/decorative motion

Tying visual changes directly to mouse position feels artificial because it lacks momentum. Use `useSpring` to interpolate value changes with spring-like behavior.

```jsx
import { useSpring, useMotionValue } from "motion/react";

const mouseX = useMotionValue(0);
const rotation = useSpring(mouseX, { stiffness: 100, damping: 10 });

return <motion.div style={{ rotate: rotation }} onMouseMove={(e) => mouseX.set(e.clientX * 0.1)} />;
```

This works because the animation is **decorative** — it doesn't serve a function. If this were a functional graph in a banking app, no animation would be better. Know when decoration helps and when it hinders.

### Apple-style spring config (recommended)

```jsx
{ type: "spring", duration: 0.5, bounce: 0.2 }
```

Easier to reason about than `mass`/`stiffness`/`damping`. Keep `bounce` subtle (0.1–0.3). Avoid bounce in most UI contexts; reserve it for drag-to-dismiss and playful interactions.

### Why springs over duration-based for gestures

Springs maintain velocity when interrupted. CSS animations and keyframes restart from zero. When a user clicks an expanded item and quickly presses Escape, a spring smoothly reverses from its current position. A keyframed animation jerks back to the start.

## `useReducedMotion`

```jsx
import { useReducedMotion, motion } from "motion/react";

function Drawer({ isOpen }) {
  const reduce = useReducedMotion();
  const closedX = reduce ? 0 : "-100%";

  return (
    <motion.div
      animate={{ x: isOpen ? 0 : closedX, opacity: isOpen ? 1 : 0 }}
      transition={{ type: "spring", duration: 0.4, bounce: 0.1 }}
    />
  );
}
```

Reduced motion ≠ no motion. Keep opacity transitions — they aid comprehension. Drop translation, rotation, and scale animations.

## `useInView` for reveal-on-scroll

```jsx
import { useInView, motion } from "motion/react";
import { useRef } from "react";

function RevealOnScroll({ children }) {
  const ref = useRef(null);
  const inView = useInView(ref, { once: true, margin: "-100px" });

  return (
    <motion.div
      ref={ref}
      initial={{ opacity: 0, y: 24 }}
      animate={inView ? { opacity: 1, y: 0 } : undefined}
      transition={{ duration: 0.4, ease: [0.23, 1, 0.32, 1] }}
    >
      {children}
    </motion.div>
  );
}
```

`{ once: true }` so the animation doesn't replay on re-entry. `margin: "-100px"` so it triggers slightly before the element is on screen — the user perceives the animation as already in progress when they look at it.

## Mount animation patterns

Three options, in order of preference:

### 1. CSS `@starting-style` (preferred, modern)

No React state needed. See `references/css.md`.

### 2. Motion's `initial` + `animate`

```jsx
<motion.div
  initial={{ opacity: 0, scale: 0.95 }}
  animate={{ opacity: 1, scale: 1 }}
  transition={{ duration: 0.18, ease: [0.23, 1, 0.32, 1] }}
/>
```

### 3. `data-mounted` legacy pattern

Use only when supporting browsers without `@starting-style` and you don't want Motion as a dependency.

```jsx
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);

return <div data-mounted={mounted} className="..." />;
```

```css
[data-mounted="false"] { opacity: 0; transform: scale(0.95); }
[data-mounted="true"]  { opacity: 1; transform: scale(1); transition: ...; }
```

## `AnimatePresence` for exits

Wrap conditionally-rendered children. The exit animation runs before unmount.

```jsx
<AnimatePresence>
  {isOpen && (
    <motion.div
      key="modal"
      initial={{ opacity: 0, scale: 0.95 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.95 }}
      transition={{ duration: 0.18, ease: [0.23, 1, 0.32, 1] }}
    />
  )}
</AnimatePresence>
```

**Asymmetric timing** — make exits faster than entrances. `transition` accepts per-property overrides:

```jsx
transition={{
  duration: 0.18,
  exit: { duration: 0.12 },
}}
```

Each `motion.*` child of `AnimatePresence` needs a stable, unique `key`. Without it, exits won't fire.
