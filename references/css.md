# CSS reference

Pure-CSS techniques. Use these even when working in React or Vue — CSS animations run off the main thread and stay smooth when JS is busy.

## Contents

1. [`@starting-style` for entry animations](#starting-style-for-entry-animations)
2. [`clip-path` patterns](#clip-path-patterns)
3. [Transform mastery](#transform-mastery)
4. [Performance rules](#performance-rules)
5. [WAAPI for programmatic CSS animations](#waapi-for-programmatic-css-animations)

## `@starting-style` for entry animations

The modern CSS way to animate element entry without JavaScript or `useEffect`:

```css
.toast {
  opacity: 1;
  transform: translateY(0);
  transition:
    opacity 400ms ease,
    transform 400ms ease;

  @starting-style {
    opacity: 0;
    transform: translateY(100%);
  }
}
```

This replaces the React pattern of using `useEffect(() => setMounted(true), [])` to flip a `data-mounted` attribute. Use `@starting-style` when browser support allows; fall back to the attribute pattern otherwise.

Tailwind v4 exposes this as the `starting:` variant — see `references/tailwind.md`.

## `clip-path` patterns

`clip-path` is one of the most powerful animation tools in CSS, not just a shape primitive.

### The `inset` shape

`clip-path: inset(top right bottom left)` defines a rectangular clipping region. Each value "eats" into the element from that side.

```css
/* Fully hidden from right */
.hidden  { clip-path: inset(0 100% 0 0); }
/* Fully visible */
.visible { clip-path: inset(0 0 0 0); }
```

### Reveal-from-left

```css
.overlay {
  clip-path: inset(0 100% 0 0);
  transition: clip-path 200ms ease-out;
}
.button:active .overlay {
  clip-path: inset(0 0 0 0);
  transition: clip-path 2s linear;
}
```

### Tabs with perfect color transitions

Duplicate the tab list. Style the copy as "active" (different background, different text color). Clip the copy so only the active tab is visible. Animate the clip on tab change. This creates a seamless color transition that timing individual color transitions can never achieve — because there *aren't* two separate colors transitioning, there's one element revealing through a clip.

### Hold-to-delete pattern

Use `clip-path: inset(0 100% 0 0)` on a colored overlay. On `:active`, transition to `inset(0 0 0 0)` over 2s with linear timing. On release, snap back with 200ms ease-out. Add `scale(0.97)` on the button for press feedback.

```css
.delete-button {
  position: relative;
  transition: transform 160ms ease-out;
}
.delete-button:active { transform: scale(0.97); }

.delete-fill {
  position: absolute;
  inset: 0;
  background: var(--danger);
  clip-path: inset(0 100% 0 0);
  transition: clip-path 200ms ease-out; /* release: snap back */
}
.delete-button:active .delete-fill {
  clip-path: inset(0 0 0 0);
  transition: clip-path 2s linear;     /* press: deliberate fill */
}
```

### Image reveals on scroll

Start with `clip-path: inset(0 0 100% 0)` (hidden from bottom). Animate to `inset(0 0 0 0)` when the element enters the viewport. Use `IntersectionObserver` directly, or Motion's `useInView` with `{ once: true, margin: "-100px" }`.

### Comparison sliders

Overlay two images. Clip the top one with `clip-path: inset(0 50% 0 0)`. Adjust the right inset based on drag position. No extra DOM elements, fully hardware-accelerated.

## Transform mastery

### `translateY` with percentages

Percentage values in `translate()` are relative to the element's own size. Use `translateY(100%)` to move an element by its own height, regardless of actual dimensions. This is how Sonner positions toasts and Vaul hides drawers before animating in.

```css
/* Works regardless of drawer height */
.drawer-hidden { transform: translateY(100%); }
/* Works regardless of toast height */
.toast-enter  { transform: translateY(-100%); }
```

Prefer percentages over hardcoded pixels. They are less error-prone and adapt to content.

### `scale()` scales children too

Unlike `width`/`height`, `scale()` scales children. When scaling a button on press, font size, icons, and content scale proportionally. This is a feature, not a bug — it preserves visual hierarchy during the press.

### 3D transforms for depth

`rotateX()`, `rotateY()` with `transform-style: preserve-3d` create real 3D effects. Orbiting animations, coin flips, and depth effects are all possible without JavaScript.

```css
.wrapper { transform-style: preserve-3d; }

@keyframes orbit {
  from {
    transform: translate(-50%, -50%) rotateY(0deg)   translateZ(72px) rotateY(360deg);
  }
  to {
    transform: translate(-50%, -50%) rotateY(360deg) translateZ(72px) rotateY(0deg);
  }
}
```

### `transform-origin`

Every element transforms around an anchor point. The default is `center`. Match it to where the trigger lives for origin-aware interactions — popovers, tooltips, dropdowns. Modals stay centered because they have no trigger location.

## Performance rules

### Only animate `transform` and `opacity`

These skip layout and paint, running on the GPU compositor. Animating `padding`, `margin`, `height`, `width`, or any layout property triggers full re-rendering.

### CSS variables on a parent are expensive

Changing a CSS variable on a parent element recalculates styles for every descendant that reads that variable. In a drawer with many items, updating `--swipe-amount` on the container triggers expensive style recalc across all children.

```js
// Bad: triggers recalc on all children
element.style.setProperty('--swipe-amount', `${distance}px`);

// Good: only affects this element
element.style.transform = `translateY(${distance}px)`;
```

### Keep blur under 20px

Heavy `filter: blur()` is expensive, especially in Safari. Subtle blur (2–8px) to mask imperfect transitions is fine; large blurs as decoration are not.

### Use CSS transitions, not keyframes, for interruptible UI

CSS transitions retarget mid-animation. Keyframes restart from zero. For any interaction that can be triggered rapidly (toasts, toggles, hover states), transitions produce smoother results.

```css
/* Interruptible — good for UI */
.toast { transition: transform 400ms ease; }

/* Not interruptible — avoid for dynamic UI */
@keyframes slide-in {
  from { transform: translateY(100%); }
  to   { transform: translateY(0); }
}
```

### CSS animations beat JS under load

CSS animations run off the main thread. When the browser is busy loading content or running scripts, JS-driven animations using `requestAnimationFrame` (including Motion's shorthand `x`/`y` props) drop frames. CSS animations remain smooth. Use CSS for predetermined animations; JS for dynamic, interruptible ones.

## WAAPI for programmatic CSS animations

The Web Animations API gives you JavaScript control with CSS performance. Hardware-accelerated, interruptible, no library needed.

```js
element.animate(
  [
    { clipPath: 'inset(0 0 100% 0)' },
    { clipPath: 'inset(0 0 0 0)' },
  ],
  {
    duration: 1000,
    fill: 'forwards',
    easing: 'cubic-bezier(0.77, 0, 0.175, 1)',
  }
);
```

Reach for WAAPI when you need to animate something CSS transitions can't express (clip-path keyframes, transform sequences) but don't want to pull in an animation library.

## Using blur to mask imperfect transitions

When a crossfade between two states feels off despite trying different easings and durations, add subtle `filter: blur(2px)` during the transition.

**Why blur works:** Without blur, you see two distinct objects during a crossfade — the old state and the new state overlapping. Blur bridges the visual gap by blending them, tricking the eye into perceiving a single smooth transformation.

```css
.button-content {
  transition:
    filter  200ms ease,
    opacity 200ms ease;
}
.button-content[data-transitioning] {
  filter: blur(2px);
  opacity: 0.7;
}
```

Combine with `scale(0.97)` on the button for a polished state transition.
