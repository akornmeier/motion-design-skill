# Component reference

Component-by-component patterns. Tailwind-first. React/Vue specifics live in `references/react.md` / `references/vue.md`.

## Contents

1. [Buttons](#buttons)
2. [Popovers, dropdowns, selects](#popovers-dropdowns-selects)
3. [Tooltips (with skip-on-subsequent)](#tooltips-with-skip-on-subsequent)
4. [Modals vs drawers](#modals-vs-drawers)
5. [Toasts](#toasts)
6. [Stagger](#stagger)
7. [The Sonner principles (building loved components)](#the-sonner-principles-building-loved-components)
8. [Asymmetric enter/exit](#asymmetric-enterexit)
9. [Debugging animations](#debugging-animations)

## Buttons

Pressable elements need responsive feedback. Without it, the UI feels dead.

```html
<button class="
  transition-transform duration-press ease-out-quint
  active:scale-[0.97]
  motion-reduce:transition-none motion-reduce:active:scale-100
">
  Save
</button>
```

Subtle scale (0.95–0.98). Apply to *any* pressable element — not just `<button>`. Cards, list items, and tab triggers benefit too.

For loading states, **don't** swap the button content abruptly. Crossfade with a brief blur to mask the state change (see `references/css.md` → "Using blur to mask imperfect transitions").

## Popovers, dropdowns, selects

The default `transform-origin: center` is wrong. Popovers should scale from their trigger.

```jsx
<div class="
  origin-[var(--radix-popover-content-transform-origin)]
  transition duration-pop ease-out-quint
  data-[state=open]:opacity-100  data-[state=open]:scale-100
  data-[state=closed]:opacity-0  data-[state=closed]:scale-95
"/>
```

Whether the user notices the difference individually doesn't matter. In aggregate, unseen details become visible. They compound.

If the user is on Base UI, the variable is `--transform-origin`. shadcn already wires this up; if it's missing in their code, that's a bug worth flagging.

## Tooltips (with skip-on-subsequent)

Tooltips should delay before the *first* one appears (prevents accidental activation). But once one is open, hovering adjacent tooltips should open them instantly with no animation. This is what makes a toolbar feel snappy.

Drive the difference from a `data-instant` flag set by the tooltip provider:

```jsx
<div class="
  origin-[var(--radix-tooltip-content-transform-origin)]
  transition duration-pop ease-out-quint
  data-[state=closed]:opacity-0  data-[state=closed]:scale-95
  data-[instant=true]:duration-0
"/>
```

If the headless library doesn't expose this, build it yourself: track "any tooltip is currently open" in a context/provider, and propagate it as the `data-instant` attribute on each tooltip content.

## Modals vs drawers

Modals and drawers are different beasts. Don't reuse the same animation.

| Property          | Modal                                                           | Drawer                                          |
| ----------------- | --------------------------------------------------------------- | ----------------------------------------------- |
| `transform-origin` | `center` (no anchor)                                           | n/a — uses translate, not scale               |
| Enter             | scale 0.95 → 1, opacity 0 → 1                                   | translateY(100%) → 0                            |
| Easing            | `ease-out-quint`                                                | `ease-drawer` (iOS-style)                       |
| Duration          | 200–250ms                                                       | 280–360ms                                       |
| Exit              | snap back faster (~150ms)                                       | snap back faster (~200ms), or follow drag       |

Modal:

```jsx
<div class="
  origin-center
  transition duration-pop ease-out-quint
  data-[state=open]:opacity-100   data-[state=open]:scale-100
  data-[state=closed]:opacity-0   data-[state=closed]:scale-95
"/>
```

Drawer:

```jsx
<div class="
  translate-y-0
  transition-transform duration-drawer ease-drawer
  data-[state=closed]:translate-y-full
"/>
```

Drawers benefit from gesture-aware dismissal — see `references/gestures.md`.

## Toasts

Stack from a corner, animate in from the same edge they exit toward. Sonner's enter/exit symmetry is what makes swipe-to-dismiss feel intuitive — the toast is *already* moving in the dismiss direction every time you see it appear.

```jsx
<div class="
  translate-y-0 opacity-100
  transition duration-drawer ease-drawer
  starting:translate-y-full starting:opacity-0
  data-[state=closed]:translate-y-full data-[state=closed]:opacity-0
"/>
```

**Pause timers when the tab is hidden.** `document.visibilitychange` → pause; visible → resume. Users hate returning to a tab to find toasts dismissed by a timer they couldn't see.

**Fill stack gaps with pseudo-elements** so hover state holds across the entire stack region, not just the visible cards. This is invisible until you remove it and notice the stack flickers when the cursor crosses gaps.

## Stagger

When multiple elements enter together, stagger them. 30–80ms between items. Stagger is decorative — never block interaction while it plays.

CSS-only for short, static lists (see `references/tailwind.md` → "Stagger entrance").

For dynamic lists in React, use Motion's stagger:

```jsx
<motion.ul
  initial="hidden"
  animate="visible"
  variants={{ visible: { transition: { staggerChildren: 0.05 } } }}
>
  {items.map((item) => (
    <motion.li
      key={item.id}
      variants={{ hidden: { opacity: 0, y: 8 }, visible: { opacity: 1, y: 0 } }}
    />
  ))}
</motion.ul>
```

In Vue with `motion-v`, the same pattern works using `:initial="'hidden'"` / `:animate="'visible'"` on `Motion`.

### The opacity + height combination

When items enter and exit a list, opacity must work well with the height animation. There is no formula — adjust until it feels right. Fast opacity (~150ms) bridges the height transition (~250ms) so you don't see a half-faded element shrinking.

## The Sonner principles (building loved components)

These come from building Sonner (now millions of weekly downloads) and apply to any component:

1. **Developer experience is key.** No hooks, no context, no complex setup. Insert one component once, call from anywhere. The less friction, the more people adopt.

2. **Good defaults matter more than options.** Ship beautiful out of the box. Most users never customize. Default easing, timing, and visual design should be excellent.

3. **Naming creates identity.** "Sonner" (French for "to ring") feels more elegant than "react-toast". Sacrifice discoverability for memorability when appropriate.

4. **Handle edge cases invisibly.** Pause toast timers when tab hidden. Fill stack gaps. Capture pointer events during drag. Users never notice these — and that is exactly right.

5. **Use transitions, not keyframes, for dynamic UI.** Toasts add and remove rapidly. Keyframes restart from zero on interruption; transitions retarget smoothly.

6. **Build a great documentation site.** Let people touch the product, play with it, understand it before they use it. Interactive examples lower the adoption barrier more than any API design choice.

### Cohesion matters

Sonner's animation feels satisfying partly because the whole experience is cohesive — the easing and duration fit the vibe of the library. It's slightly slower than typical UI animation and uses `ease` rather than `ease-out` to feel more elegant. Animation style matches the toast design, the page design, the name — everything in harmony.

Match the motion to the mood. A playful component can be bouncier. A professional dashboard should be crisp and fast.

## Asymmetric enter/exit

Slow when the user is deciding. Fast when the system is responding.

| Situation             | Speed                                |
| --------------------- | ------------------------------------ |
| Hold-to-delete press  | 2s linear (deliberate)               |
| Hold-to-delete release | 200ms ease-out (snappy)             |
| Modal enter           | 200ms                                |
| Modal exit            | 120–150ms (faster than enter)        |
| Toast enter           | 320ms                                |
| Toast exit            | 200ms                                |

This applies broadly: anywhere the user expresses intent, give weight; anywhere the system confirms, get out of the way.

## Debugging animations

### Slow motion testing

Play animations at reduced speed to spot issues invisible at full speed. Temporarily multiply `duration` by 5×, or use Chrome DevTools' Animations panel to slow playback.

Watch for:

- Crossfade ghosts — do you see two distinct states overlapping? Use blur to bridge (see `references/css.md`).
- Easing feel — does it start/stop abruptly? Try a stronger custom curve.
- Wrong `transform-origin` — does the element scale from the wrong point?
- Out-of-sync properties — opacity finishes before transform, or vice versa.

### Frame-by-frame inspection

Step through animations frame by frame in Chrome DevTools' Animations panel. This reveals timing issues between coordinated properties that you can't see at full speed.

### Test on real devices

For touch interactions (drawers, swipe gestures), test on physical devices. Connect via USB, visit your local dev server by IP, and use Safari's remote devtools. The Xcode Simulator is acceptable for layout, but real hardware is required for gesture feel.

### Review the next day

You notice imperfections with fresh eyes that you missed during development. The day-after review is the cheapest taste-improving habit you can adopt.
