---
name: tailwind-css
description: Use or audit Tailwind CSS — utility classes, responsive design, arbitrary values, custom config. Use when the user says "style this", "make it responsive", "add Tailwind", or "why isn't this class working".
---

You are styling with Tailwind CSS v3.

## Core concept

Tailwind is utility-first — you compose small single-purpose classes directly in HTML/JSX instead of writing CSS. No custom class names unless you extract a component.

```tsx
// Instead of writing a .card class in CSS:
<div className="bg-white rounded-2xl p-6 shadow-md border border-gray-100">
```

## Responsive design

Tailwind is mobile-first. Unprefixed classes apply to all sizes. Prefixes apply at breakpoints and above:

```tsx
<div className="
  grid grid-cols-1      // mobile: 1 column
  md:grid-cols-2        // ≥768px: 2 columns
  lg:grid-cols-3        // ≥1024px: 3 columns
">

<div className="
  hidden                // mobile: hidden
  md:block              // ≥768px: visible
">
```

Default breakpoints: `sm` 640px, `md` 768px, `lg` 1024px, `xl` 1280px, `2xl` 1536px.

## Arbitrary values

When you need a value not in Tailwind's scale, use square brackets:

```tsx
className="w-[280px]"              // exact width
className="top-[140px]"            // exact position
className="bg-[#C4956A]"           // exact color
className="text-[13px]"            // exact font size
className="animate-[ticker_28s_linear_infinite]"   // custom animation reference
className="grid-cols-[1fr_280px]"  // custom grid template
```

## Common utilities

```tsx
// Layout
flex items-center justify-between gap-4
grid grid-cols-3 gap-6
hidden md:flex                     // responsive show/hide

// Spacing
p-4 px-6 py-3                     // padding
m-4 mx-auto mt-8                   // margin
space-x-4 space-y-2               // gap between children

// Typography
text-sm font-medium tracking-wider uppercase
text-center text-left
leading-relaxed

// Colors
bg-white text-gray-900
bg-[var(--surface)] text-[var(--text)]   // use CSS vars inside []

// Borders
border border-gray-200 rounded-xl rounded-full
divide-y divide-gray-100

// Shadows
shadow-sm shadow-md shadow-lg

// Sizing
w-full h-full max-w-xl min-h-screen
w-[34px] h-[34px]                 // exact sizes via arbitrary values

// Position
relative absolute fixed sticky
inset-0 top-0 left-0 right-0 bottom-0 z-50

// Overflow
overflow-hidden overflow-x-auto
```

## CSS vars inside Tailwind

```tsx
// Use CSS custom properties with bracket notation
className="bg-[var(--surface)]"
className="text-[var(--accent)]"
className="border-[var(--border)]"
```

## Witly landing page patterns

```tsx
// Container utility defined in globals.css @layer utilities
className="container-witly"      // max-width 1120px, centered, padded

// Responsive show/hide for floating badges
className="hidden lg:block"       // only show on large screens

// Custom animation from tailwind.config.ts keyframes
className="animate-[ticker_28s_linear_infinite]"
className="flex w-max"            // needed alongside custom animation
```

## Custom config (tailwind.config.ts)

```ts
export default {
  theme: {
    extend: {
      colors: {
        accent: '#C4956A',
        safe: '#5A8F6B',
      },
      keyframes: {
        ticker: {
          '0%': { transform: 'translateX(0)' },
          '100%': { transform: 'translateX(-50%)' },
        },
        float: {
          '0%, 100%': { transform: 'translateY(0)' },
          '50%': { transform: 'translateY(-12px)' },
        },
      },
      animation: {
        ticker: 'ticker 28s linear infinite',
        float: 'float 5s ease-in-out infinite',
      },
    },
  },
}
```

**Important**: keyframes defined in `tailwind.config.ts` are only emitted to CSS when referenced by a Tailwind utility class. Inline `style={{ animation: 'ticker ...' }}` will NOT find them — use `className="animate-[ticker_28s_linear_infinite]"` or define the keyframe in `globals.css` instead.

## @layer utilities (globals.css)

```css
@layer utilities {
  .container-witly {
    max-width: 1120px;
    margin: 0 auto;
    padding: 0 24px;
  }

  .section-eyebrow {
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--accent);
  }
}
```

Use `@layer utilities` for reusable multi-property utilities. Use `@layer components` for component-level abstractions.

## Common mistakes

- `hidden md:flex` vs `md:hidden` — easy to confuse. `hidden md:flex` means hidden on mobile, flex on desktop. `md:hidden` means visible on mobile, hidden on desktop.
- Arbitrary values without brackets: `w-280px` doesn't work. Must be `w-[280px]`.
- Using inline `animation:` to reference config keyframes — they won't be found. Use `className="animate-[...]"`.
- Purging issues — if a class is dynamically constructed (`"bg-" + color`), Tailwind can't detect it. Use full class names or safelist them.
- Mixing Tailwind with heavy inline styles — pick one approach per component.
