# Tailwind reference

Tailwind v4 is the assumed default. Most patterns from this skill collapse into a couple of utility classes once your theme is set up — that is the whole point of doing this work in CSS.

## Contents

1. [Theme setup: easing and duration tokens](#theme-setup-easing-and-duration-tokens)
2. [Variants you actually use for motion](#variants-you-actually-use-for-motion)
3. [Pattern recipes](#pattern-recipes)
4. [When to drop to arbitrary values](#when-to-drop-to-arbitrary-values)
5. [Tailwind 3 backports](#tailwind-3-backports)

## Theme setup: easing and duration tokens

In `app.css` (or wherever you import Tailwind), expose the foundational tokens once:

```css
@import "tailwindcss";

@theme {
  /* Easing — Tailwind maps --ease-* to the ease-* utility name */
  --ease-out-quint:    cubic-bezier(0.23, 1, 0.32, 1);
  --ease-in-out-quint: cubic-bezier(0.77, 0, 0.175, 1);
  --ease-drawer:       cubic-bezier(0.32, 0.72, 0, 1);

  /* Duration — Tailwind maps --duration-* to the duration-* utility name */
  --duration-press:  120ms;
  --duration-pop:    180ms;
  --duration-drawer: 320ms;
}
```

You can now write `ease-out-quint`, `duration-press`, etc. anywhere Tailwind accepts those utilities. **Do this once and stop hand-authoring `cubic-bezier` values inline.** The point of tokens is consistency; one-off curves defeat it.

## Variants you actually use for motion

| Variant                     | When                                                                                 |
| --------------------------- | ------------------------------------------------------------------------------------ |
| `hover:`                    | Already gated on `@media (hover: hover) and (pointer: fine)` in Tailwind v4          |
| `active:`                   | Press feedback — `active:scale-[0.97]`                                               |
| `motion-safe:`              | Wrap any movement-based animation. The default if `prefers-reduced-motion` is unset. |
| `motion-reduce:`            | Provide a gentler alternative — opacity-only, no transform                           |
| `starting:`                 | Tailwind v4 maps to CSS `@starting-style`. Use for entry animations.                 |
| `data-[state=open]:`        | Radix / Base UI / shadcn state attributes                                            |
| `aria-expanded:`            | When toggling on ARIA state instead of a data attribute                              |

The `hover:` gating is important: it means `hover:scale-105` will not falsely trigger on touch devices that synthesize hover on tap. You don't need to wrap it manually.

## Pattern recipes

### Pressable button

```html
<button
  class="
    transition-transform duration-press ease-out-quint
    active:scale-[0.97]
    motion-reduce:transition-none motion-reduce:active:scale-100
  "
>
  Save
</button>
```

### Popover / dropdown enter (Radix)

Radix exposes `data-[state=open]` and `data-[state=closed]` plus `--radix-popover-content-transform-origin`. Combine with `starting:` for first-paint:

```jsx
<div
  class="
    origin-[var(--radix-popover-content-transform-origin)]
    transition duration-pop ease-out-quint
    data-[state=open]:opacity-100  data-[state=open]:scale-100
    data-[state=closed]:opacity-0  data-[state=closed]:scale-95
  "
/>
```

Modals do **not** get a trigger-relative origin — they stay `origin-center`.

### Tooltip with skip-on-subsequent

The first tooltip should delay and animate. Once one is open, hovering over adjacent triggers should open instantly. Drive this from a `data-instant` attribute on a wrapping provider:

```jsx
<div
  class="
    origin-[var(--radix-tooltip-content-transform-origin)]
    transition duration-pop ease-out-quint
    data-[state=closed]:opacity-0  data-[state=closed]:scale-95
    data-[instant=true]:duration-0
  "
/>
```

### Toast enter (with `@starting-style`)

```jsx
<div
  class="
    translate-y-0 opacity-100
    transition duration-drawer ease-drawer
    starting:translate-y-full starting:opacity-0
  "
/>
```

### Drawer slide-up

```jsx
<div
  class="
    translate-y-0
    transition-transform duration-drawer ease-drawer
    data-[state=closed]:translate-y-full
  "
/>
```

### Stagger entrance (lists)

CSS keyframes with `nth-child` is the most maintainable approach for short lists. For long or dynamic lists, drive from JS — see `references/react.md` and `references/vue.md`.

```html
<ul>
  <!-- repeat -->
  <li class="animate-fade-in [animation-delay:0ms]   [animation-fill-mode:both]">…</li>
  <li class="animate-fade-in [animation-delay:50ms]  [animation-fill-mode:both]">…</li>
  <li class="animate-fade-in [animation-delay:100ms] [animation-fill-mode:both]">…</li>
</ul>
```

```css
@theme {
  --animate-fade-in: fade-in 300ms var(--ease-out-quint);
}
@keyframes fade-in {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

Keep stagger delays short (30–80ms between items). Stagger is decorative — never block interaction while it plays.

### Hold-to-delete

```jsx
<button class="group relative active:scale-[0.97] transition-transform duration-press ease-out-quint">
  <span class="
    absolute inset-0 bg-red-500
    [clip-path:inset(0_100%_0_0)]
    transition-[clip-path] duration-200 ease-out-quint
    group-active:[clip-path:inset(0_0_0_0)]
    group-active:duration-[2000ms] group-active:ease-linear
  " />
  <span class="relative">Delete</span>
</button>
```

### Reduced-motion alternative

Always provide a fallback that conveys the same state change without movement:

```jsx
<div class="
  transition-transform duration-pop ease-out-quint
  motion-safe:data-[state=closed]:translate-x-full
  motion-reduce:transition-opacity
  motion-reduce:data-[state=closed]:opacity-0
"/>
```

Reduced motion does **not** mean *no* motion — it means *no movement*. Opacity transitions are still fine and aid comprehension.

## When to drop to arbitrary values

Use the theme. When you can't:

- **Custom `clip-path`** — `[clip-path:inset(0_50%_0_0)]` (note: underscores read as spaces inside `[]`)
- **Specific transform-origin** — `origin-[var(--radix-popover-content-transform-origin)]`
- **`will-change`** for hot-path drag handles — `[will-change:transform]` (use sparingly; promotion has cost)

If you find yourself writing the same arbitrary value three times, lift it into `@theme`.

## Tailwind 3 backports

If the user is on v3:

- Use `tailwind.config.js` with `theme.extend.transitionTimingFunction` and `theme.extend.transitionDuration` for the same tokens.
- `starting:` does not exist — use a `data-mounted` attribute toggled in `useEffect`/`onMounted` instead.
- `motion-safe:` / `motion-reduce:` work the same way.
- `hover:` does not gate on `(hover: hover)` automatically. Wrap with custom variant or accept the false positive on touch.

Recommend upgrading to v4 if the user has the option — the motion story is materially better.
