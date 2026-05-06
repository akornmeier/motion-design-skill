# Motion Design Skill

A Claude skill that encodes Emil Kowalski's craft sensibility for UI polish, motion, and the invisible details that make software feel great.

Triggers when you ask Claude to review UI code, build a component, critique an animation, or pick easing/duration values. Defaults to **Tailwind v4 + Motion** (`motion/react` for React, `motion-v` for Vue 3) — even when the stack isn't named explicitly.

## What it covers

- **Animation decision framework.** Should this animate at all? What is the purpose? What easing? How fast?
- **Required review format.** Output is always a Before / After / Why markdown table.
- **Easing tokens.** Strong custom curves for entrances, on-screen movement, and iOS-style drawers.
- **Component patterns.** Buttons, popovers, tooltips, modals, drawers, toasts, stagger.
- **Gesture & spring physics.** Drag, swipe-to-dismiss, momentum, damping, multi-touch.
- **Performance discipline.** Only `transform` and `opacity`. Hardware-acceleration caveats for Motion. CSS animations under main-thread load.
- **Framework specifics.** `motion/react` (React), `motion-v` (Vue 3), Tailwind v4 utilities.

## File map

```
SKILL.md                  Lean entry point — philosophy, framework, top checklist
references/
├── css.md                @starting-style, clip-path, transforms, perf, WAAPI
├── tailwind.md           Tailwind v4 theme tokens, variants, recipe library
├── react.md              motion/react — hooks, AnimatePresence, hardware accel
├── vue.md                motion-v — Motion component, composables, Reka UI
├── components.md         Component-by-component patterns
└── gestures.md           Drag, swipe, springs, interruptibility
```

`SKILL.md` always loads when the skill triggers. Reference files load only when the task calls for them — that progressive-disclosure approach keeps Claude's context lean for simple questions and rich for deep ones.

## Install

Drop the repo into Claude Code's skills directory:

```bash
cp -r motion-design-skill ~/.claude/skills/
```

Or, if you're using a centralized skills location, symlink it.

## Usage

Just talk to Claude. The skill triggers on phrases like:

- "review this button component"
- "why does this dropdown feel sluggish"
- "build a drawer with swipe-to-dismiss"
- "what easing should I use for a toast"
- "migrate from framer-motion to motion"
- mentions of Motion.dev, motion/react, motion-v, Tailwind animation, etc.

## Credit

The opinions, patterns, and examples in this skill are drawn from Emil Kowalski's writing and talks on motion and design engineering, especially his work on **Sonner**, **Vaul**, and the design engineering programs at Vercel and elsewhere. This repo packages those ideas in a form Claude can reach for at the right moment.

## License

MIT.
