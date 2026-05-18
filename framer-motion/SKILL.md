---
name: framer-motion
description: Add or audit Framer Motion animations — entrance animations, continuous loops, scroll-triggered reveals, staggered children. Use when the user says "animate this", "add a scroll animation", "make it float", or "stagger these items".
---

You are adding or auditing Framer Motion 11 animations in a React/Next.js project.

## Core concepts

- `motion.div` (or any `motion.element`) — a DOM element with animation superpowers
- `animate` — the target state to animate to
- `initial` — the starting state
- `transition` — how to get there (duration, ease, delay, repeat)
- `whileInView` — animate when element enters the viewport
- `whileHover` — animate on hover
- `style` — static styles (not animated)

## Entrance animation

```tsx
import { motion } from 'framer-motion'

<motion.div
  initial={{ opacity: 0, y: 24 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.65, ease: [0.22, 1, 0.36, 1] }}
>
  Content
</motion.div>
```

## Scroll-triggered reveal

```tsx
<motion.div
  initial={{ opacity: 0, y: 28 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, margin: '-80px' }}   // trigger 80px before element enters view
  transition={{ duration: 0.65, ease: [0.22, 1, 0.36, 1] }}
>
  Content
</motion.div>
```

`once: true` — animates only the first time. Remove for re-trigger on scroll back.

## Continuous loop (float animation)

```tsx
<motion.div
  animate={{ y: [0, -12, 0] }}
  transition={{ duration: 5, repeat: Infinity, ease: 'easeInOut' }}
  style={{ rotate: -1 }}    // static tilt — put rotation in style, NOT animate
>
  <PhoneMockup />
</motion.div>
```

**Critical**: static transforms (rotation, scale) go in `style`, not `animate`. Putting `rotate` inside `animate` alongside `y` keyframes creates conflicting transforms. Use `style={{ rotate: -1 }}` for a static tilt.

## Staggered children

```tsx
const container = {
  hidden: {},
  show: { transition: { staggerChildren: 0.11 } },
}

const item = {
  hidden: { opacity: 0, y: 24 },
  show: { opacity: 1, y: 0, transition: { duration: 0.65, ease: [0.22, 1, 0.36, 1] } },
}

<motion.div variants={container} initial="hidden" animate="show">
  <motion.h1 variants={item}>Headline</motion.h1>
  <motion.p variants={item}>Subtext</motion.p>
  <motion.div variants={item}>CTA</motion.div>
</motion.div>
```

The parent controls timing — each child inherits from `variants` and staggers automatically.

## Floating badges (independent delays)

```tsx
{/* Phone — main float */}
<motion.div
  animate={{ y: [0, -12, 0] }}
  transition={{ duration: 5, repeat: Infinity, ease: 'easeInOut' }}
  style={{ rotate: -1 }}
>

{/* Badge 1 — slightly offset phase */}
<motion.div
  animate={{ y: [0, -8, 0] }}
  transition={{ duration: 6, repeat: Infinity, ease: 'easeInOut', delay: 1 }}
>

{/* Badge 2 — different rhythm */}
<motion.div
  animate={{ y: [0, -8, 0] }}
  transition={{ duration: 6, repeat: Infinity, ease: 'easeInOut', delay: 0 }}
>
```

Different `delay` and `duration` values make floating elements feel organic rather than in sync.

## Hover animation

```tsx
<motion.article
  whileHover={{
    y: -6,
    boxShadow: '0 24px 60px rgba(0,0,0,0.35)',
    borderColor: 'rgba(196,149,106,0.35)',
    transition: { duration: 0.2 },
  }}
>
```

## Staggered chip/tag grid

```tsx
{items.map((item, i) => (
  <motion.span
    key={item}
    initial={{ opacity: 0, y: 10 }}
    whileInView={{ opacity: 1, y: 0 }}
    viewport={{ once: true, margin: '-40px' }}
    transition={{ duration: 0.45, delay: i * 0.055, ease: [0.22, 1, 0.36, 1] }}
  >
    {item}
  </motion.span>
))}
```

## Easing reference

```tsx
ease: 'easeOut'                         // standard deceleration
ease: 'easeInOut'                       // good for loops
ease: [0.22, 1, 0.36, 1]               // custom cubic-bezier — snappy entrance
ease: [0.4, 0, 0.2, 1]                 // material design standard
```

## When NOT to use Framer Motion

- Simple CSS transitions (hover color, opacity) — use `transition:` in CSS or inline styles
- Ticker/marquee animations — use `@keyframes` in CSS + `animation:` property (Framer Motion adds unnecessary JS overhead for pure CSS loops)
- Animations that don't need JS (no scroll trigger, no state dependency) — CSS is lighter

## Common mistakes

- Rotation in `animate` alongside `y` keyframes — conflicting transforms, use `style={{ rotate }}` instead
- `whileInView` without `once: true` — re-animates on every scroll, usually annoying
- Using Framer Motion for CSS ticker animations — the keyframe runs fine in CSS, no need for JS
- Forgetting `'use client'` — Framer Motion is client-only, any component using it needs the directive
- All elements floating in sync — use different `duration` and `delay` to feel natural
