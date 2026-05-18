---
name: css-custom-properties
description: Design or audit CSS custom properties — theming with vars, dark/light mode via data-theme. Use when the user says "set up theming", "add CSS variables", "implement dark mode", or "audit the color system".
---

You are designing or auditing a CSS custom properties theming system.

## Core concept

CSS custom properties (variables) are defined once and reused everywhere. Change the value in one place, everything updates. They cascade like any CSS property — override them at a different selector to theme.

```css
:root {
  --accent: #C4956A;
}

button {
  background: var(--accent);           /* reads the current value */
  border: 1px solid var(--accent, #000); /* fallback if var undefined */
}
```

## Witly design system (reference)

```css
/* src/app/globals.css */
:root {
  /* Backgrounds */
  --bg:        #121110;   /* page background */
  --bg2:       #181614;   /* slightly lighter */
  --bg3:       #1F1D19;   /* input backgrounds */
  --surface:   #262320;   /* card backgrounds */
  --surface2:  #2F2C28;   /* secondary surfaces */

  /* Borders */
  --border:    rgba(196,149,106,0.14);

  /* Brand */
  --accent:    #C4956A;   /* primary CTA, highlights */
  --accent2:   #D4A574;   /* hover state */
  --gold:      #E8C97A;   /* stars, decorative */

  /* Text */
  --text:      #EDE8E1;   /* primary text */
  --text2:     #B2AAA1;   /* secondary text */
  --text3:     #7C7469;   /* muted text, labels */

  /* Semantic (option types) */
  --safe:      #5A8F6B;
  --playful:   #7B6FA8;
  --bold:      #B85C4A;

  /* Spacing & radius tokens */
  --radius-sm: 10px;
  --radius:    16px;
  --radius-lg: 24px;
  --radius-xl: 32px;

  /* Fonts */
  --font-serif: 'Lora', Georgia, serif;
  --font-sans:  'DM Sans', system-ui, sans-serif;
}
```

## Dark/light mode with data-theme

Override the vars at a different selector — all components using `var(--bg)` update automatically:

```css
/* Light mode overrides */
html[data-theme="light"] {
  --bg:        #FDFAF7;
  --bg2:       #F7F3EE;
  --bg3:       #EFE9E1;
  --surface:   #FFFFFF;
  --surface2:  #F2EDE6;
  --border:    rgba(139,100,65,0.18);
  --text:      #1C1610;
  --text2:     #5C4E41;
  --text3:     #9A8070;
  --safe:      #3A7050;   /* darken for light background contrast */
  --playful:   #5B4E9A;
  --bold:      #9A3828;

  /* accent stays the same — works as button bg in both modes */
}

/* Noise texture — reduce opacity in light mode */
html[data-theme="light"] body::before {
  opacity: 0.05;
}
```

Toggle by setting `document.documentElement.setAttribute('data-theme', 'light')`.

## Using vars in JSX inline styles

```tsx
style={{ background: 'var(--surface)', color: 'var(--text)' }}
style={{ border: '1px solid var(--border)' }}
style={{ color: 'var(--accent)' }}
```

In Tailwind arbitrary values:
```tsx
className="bg-[var(--surface)] text-[var(--text)]"
```

## Naming conventions

- **Layered backgrounds**: `--bg` → `--bg2` → `--bg3` → `--surface` → `--surface2` (darkest to slightly lighter in dark mode, reversed in light mode)
- **Semantic names**: `--text`, `--text2`, `--text3` over `--gray-900`, `--gray-500` — semantic names survive a theme change, color names don't
- **Intent-based**: `--accent` not `--orange` — if you change the brand color, you change one var, not 40 usages

## Noise texture pattern

```css
body::before {
  content: '';
  position: fixed;
  inset: 0;
  background-image: url("data:image/svg+xml,...");  /* inline SVG feTurbulence */
  pointer-events: none;
  z-index: 999;
  opacity: 0.2;   /* dark mode */
}

html[data-theme="light"] body::before {
  opacity: 0.05;  /* much subtler in light mode */
}
```

## Font vars (Next.js next/font/google)

```tsx
// layout.tsx — load fonts, expose as CSS vars
const lora = Lora({ variable: '--font-lora', ... })
<html className={`${lora.variable} ${dmSans.variable}`}>

// globals.css — consume via var
body { font-family: var(--font-sans); }
h1   { font-family: var(--font-serif); }
```

## Keyframes in globals.css

Define `@keyframes` **outside** any `@layer` block — inline `animation:` style properties look in the browser's CSS scope, not Tailwind's output:

```css
/* OUTSIDE @layer — works with inline style animation: */
@keyframes ticker {
  0%   { transform: translateX(0); }
  100% { transform: translateX(-50%); }
}

@layer utilities {
  /* classes defined here */
}
```

## Common mistakes

- Color names instead of semantic names (`--blue-500`) — breaks when you change the brand color
- Forgetting to override ALL vars in the alternate theme — some components stay dark when they should go light
- Defining `@keyframes` inside `@layer` — inline `animation:` style won't find them
- Using hardcoded hex values in component styles instead of vars — the component won't theme correctly
- Not darkening semantic colors for light mode — `--safe #5A8F6B` looks fine on dark, needs to be `#3A7050` on light for contrast
