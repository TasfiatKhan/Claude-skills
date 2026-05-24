---
name: theming
description: Theming for React Native and Next.js apps — design tokens, CSS custom properties, dark/light mode, ThemeContext, flash prevention. Use when the user says "add a design system", "set up theming", "implement dark mode", "add a theme toggle", "fix theme flash", "why do we have hardcoded hex colors", or "centralise styles".
---

You are implementing a theming system. The approach differs by platform — use the right section below.

---

## React Native — Design Tokens + ThemeContext

### What goes in theme.ts

```
src/theme.ts
  lightColors / darkColors   — all semantic color values
  typography                 — font sizes, line heights, font weights
  spacing                    — xs/sm/md/lg/xl/xxl scale
  radii                      — border radius scale
  shadow                     — card shadow definitions
```

### Complete theme.ts

```typescript
// src/theme.ts
export type AppColors = typeof lightColors

export const lightColors = {
  background:       '#FAFAF8',
  surface:          '#FFFFFF',
  surfaceSecondary: '#F2EDE6',
  border:           'rgba(0,0,0,0.08)',

  accent:           '#C4956A',
  accentLight:      'rgba(196,149,106,0.12)',

  text:             '#1C1610',
  textSecondary:    '#5C4E41',
  textMuted:        '#9A8070',
  placeholder:      '#C4B09A',

  safe:             '#3A7050',
  safeLight:        'rgba(58,112,80,0.12)',
  playful:          '#5B4E9A',
  playfulLight:     'rgba(91,78,154,0.12)',
  bold:             '#9A3828',
  boldLight:        'rgba(154,56,40,0.12)',

  error:            '#C0392B',
  success:          '#27AE60',

  tabBar:           '#FFFFFF',
  tabBarBorder:     'rgba(0,0,0,0.06)',
  tabBarActive:     '#C4956A',
  tabBarInactive:   '#9A8070',
}

export const darkColors: AppColors = {
  background:       '#1A1A1A',
  surface:          '#242424',
  surfaceSecondary: '#2E2E2E',
  border:           'rgba(255,255,255,0.10)',

  accent:           '#C4956A',
  accentLight:      'rgba(196,149,106,0.15)',

  text:             '#F0EBE3',
  textSecondary:    '#B8A898',
  textMuted:        '#7A6A5A',
  placeholder:      '#5A4A3A',

  safe:             '#5A8F6B',
  safeLight:        'rgba(90,143,107,0.15)',
  playful:          '#7B6FA8',
  playfulLight:     'rgba(123,111,168,0.15)',
  bold:             '#B85C4A',
  boldLight:        'rgba(184,92,74,0.15)',

  error:            '#E74C3C',
  success:          '#2ECC71',

  tabBar:           '#1E1E1E',
  tabBarBorder:     'rgba(255,255,255,0.08)',
  tabBarActive:     '#C4956A',
  tabBarInactive:   '#7A6A5A',
}

export const typography = {
  sizes: { xs: 11, sm: 13, md: 15, lg: 17, xl: 22, xxl: 28, hero: 36 },
  lineHeights: { tight: 1.2, normal: 1.5, loose: 1.7 },
  weights: {
    regular:  '400' as const,
    medium:   '500' as const,
    semibold: '600' as const,
    bold:     '700' as const,
  },
}

export const spacing = { xs: 4, sm: 8, md: 12, lg: 16, xl: 24, xxl: 40 }

export const radii = { sm: 6, md: 12, lg: 18, xl: 24, full: 999 }

export const shadow = {
  card: {
    shadowColor:   '#000000',
    shadowOffset:  { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius:  8,
    elevation:     3,
  },
}
```

### ThemeContext (React Native)

```tsx
// src/context/ThemeContext.tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState } from 'react'
import * as SecureStore from 'expo-secure-store'
import { lightColors, darkColors, AppColors } from '../theme'

const THEME_KEY = '<app>_theme'

interface ThemeContextValue {
  colors: AppColors
  isDark: boolean
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextValue>({
  colors: lightColors,
  isDark: false,
  toggleTheme: () => {},
})

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [isDark, setIsDark] = useState(false)

  useEffect(() => {
    SecureStore.getItemAsync(THEME_KEY).then(v => {
      if (v === 'dark') setIsDark(true)
    })
  }, [])

  const toggleTheme = useCallback(() => {
    setIsDark(prev => {
      const next = !prev
      SecureStore.setItemAsync(THEME_KEY, next ? 'dark' : 'light')
      return next
    })
  }, [])

  const value = useMemo(
    () => ({ colors: isDark ? darkColors : lightColors, isDark, toggleTheme }),
    [isDark, toggleTheme]
  )

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}

export const useTheme = () => useContext(ThemeContext)
```

No flash problem in React Native — no server render.

### Using tokens in a screen

```tsx
import { useMemo } from 'react'
import { StyleSheet } from 'react-native'
import { useTheme } from '../context/ThemeContext'
import { typography, spacing, radii, shadow } from '../theme'

export default function HomeScreen() {
  const { colors } = useTheme()

  // Recreate styles only when colors change (dark/light toggle)
  const styles = useMemo(() => StyleSheet.create({
    container: { flex: 1, backgroundColor: colors.background, padding: spacing.lg },
    card:      { backgroundColor: colors.surface, borderRadius: radii.lg, padding: spacing.lg, ...shadow.card },
    title:     { fontSize: typography.sizes.xl, fontWeight: typography.weights.bold, color: colors.text },
    subtitle:  { fontSize: typography.sizes.sm, color: colors.textSecondary, marginTop: spacing.xs },
  }), [colors])

  return (
    <View style={styles.container}>
      <View style={styles.card}>
        <Text style={styles.title}>Home</Text>
        <Text style={styles.subtitle}>Welcome back</Text>
      </View>
    </View>
  )
}
```

### TextInput — always explicit color

Without explicit props, inputs may be invisible in dark mode:

```tsx
<TextInput
  style={{ color: colors.text, backgroundColor: colors.surface }}
  placeholderTextColor={colors.placeholder}
/>
```

### Mapping type labels to color pairs

```typescript
const OPTION_COLORS = {
  safe:    { bg: colors.safeLight,    label: colors.safe    },
  playful: { bg: colors.playfulLight, label: colors.playful },
  bold:    { bg: colors.boldLight,    label: colors.bold    },
}
const { bg, label } = OPTION_COLORS[option.type]
```

### Audit checklist (React Native)

Search for and replace:
- `'#` (hex colors) → `colors.*`
- `fontSize:` (raw numbers) → `typography.sizes.*`
- `padding:` / `margin:` (raw numbers) → `spacing.*`
- `borderRadius:` (raw numbers) → `radii.*`

---

## Web — CSS Custom Properties + ThemeContext (Next.js)

### CSS custom properties (globals.css)

Define once, use everywhere. Override at a different selector to theme:

```css
/* src/app/globals.css */
:root {
  /* Backgrounds */
  --bg:       #121110;
  --bg2:      #181614;
  --bg3:      #1F1D19;
  --surface:  #262320;
  --surface2: #2F2C28;

  /* Borders */
  --border:   rgba(196,149,106,0.14);

  /* Brand */
  --accent:   #C4956A;
  --accent2:  #D4A574;
  --gold:     #E8C97A;

  /* Text */
  --text:     #EDE8E1;
  --text2:    #B2AAA1;
  --text3:    #7C7469;

  /* Semantic */
  --safe:     #5A8F6B;
  --playful:  #7B6FA8;
  --bold:     #B85C4A;

  /* Spacing & radius */
  --radius-sm: 10px;
  --radius:    16px;
  --radius-lg: 24px;
  --radius-xl: 32px;

  /* Fonts */
  --font-serif: 'Lora', Georgia, serif;
  --font-sans:  'DM Sans', system-ui, sans-serif;
}

/* Light mode overrides */
html[data-theme="light"] {
  --bg:       #FDFAF7;
  --bg2:      #F7F3EE;
  --bg3:      #EFE9E1;
  --surface:  #FFFFFF;
  --surface2: #F2EDE6;
  --border:   rgba(139,100,65,0.18);
  --text:     #1C1610;
  --text2:    #5C4E41;
  --text3:    #9A8070;
  --safe:     #3A7050;
  --playful:  #5B4E9A;
  --bold:     #9A3828;
}
```

Toggle by setting: `document.documentElement.setAttribute('data-theme', 'light')`

### Using vars in JSX and Tailwind

```tsx
style={{ background: 'var(--surface)', color: 'var(--text)' }}
className="bg-[var(--surface)] text-[var(--text)]"
```

### Naming conventions

- **Semantic names** (`--text`, `--text2`) not color names (`--gray-900`) — semantic names survive brand color changes
- **Intent-based** (`--accent` not `--orange`) — change the brand color in one place, not 40 usages
- **Layered backgrounds**: `--bg` → `--bg2` → `--bg3` → `--surface` → `--surface2`

### Font vars (Next.js next/font)

```tsx
// layout.tsx — load fonts, expose as CSS vars
const lora   = Lora({ variable: '--font-lora', ... })
const dmSans = DM_Sans({ variable: '--font-sans', ... })
<html className={`${lora.variable} ${dmSans.variable}`}>

// globals.css
body { font-family: var(--font-sans); }
h1   { font-family: var(--font-serif); }
```

### Keyframes — define outside @layer

```css
/* OUTSIDE @layer — works with inline style animation: */
@keyframes ticker {
  0%   { transform: translateX(0); }
  100% { transform: translateX(-50%); }
}
```

Defining `@keyframes` inside `@layer` breaks inline `animation:` style properties.

### ThemeContext (Next.js)

Solves three problems: state persistence, flash on load, hydration mismatch.

```tsx
// src/context/ThemeContext.tsx
'use client'

import { createContext, useContext, useEffect, useState } from 'react'

interface ThemeContextValue {
  isDark: boolean
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextValue>({ isDark: false, toggleTheme: () => {} })

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [isDark, setIsDark] = useState(false)  // match server default

  useEffect(() => {
    const stored = localStorage.getItem('theme')
    const dark = stored === 'dark'
    setIsDark(dark)
    document.documentElement.setAttribute('data-theme', dark ? 'dark' : 'light')
  }, [])

  const toggleTheme = () => {
    setIsDark(prev => {
      const next = !prev
      localStorage.setItem('theme', next ? 'dark' : 'light')
      document.documentElement.setAttribute('data-theme', next ? 'dark' : 'light')
      return next
    })
  }

  return <ThemeContext.Provider value={{ isDark, toggleTheme }}>{children}</ThemeContext.Provider>
}

export const useTheme = () => useContext(ThemeContext)
```

```tsx
// src/components/Providers.tsx
'use client'
import { ThemeProvider } from '@/context/ThemeContext'
export default function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider>{children}</ThemeProvider>
}
```

```tsx
// src/app/layout.tsx
import Providers from '@/components/Providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <head>
        {/* Runs synchronously before first paint — prevents flash */}
        <script dangerouslySetInnerHTML={{ __html:
          `try{var t=localStorage.getItem('theme');document.documentElement.setAttribute('data-theme',t==='dark'?'dark':'light')}catch(e){}`
        }} />
      </head>
      <body suppressHydrationWarning>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

### Why each piece matters

**Flash prevention script**: without it, the server sends HTML with no `data-theme`, browser paints (CSS default), then JS hydrates and switches — visible flash. The inline script runs before first paint.

**`suppressHydrationWarning`**: browser extensions (Grammarly, password managers) inject attributes into `<html>` and `<body>`. Our own script adds `data-theme`. React warns about the mismatch. `suppressHydrationWarning` tells React it's intentional. Only use on `<html>` and `<body>`.

**`useState(false)` default**: server renders light mode → client initial render is light mode → they match. After `useEffect` reads localStorage and potentially switches. The match prevents hydration errors.

### Theme toggle button

```tsx
'use client'
import { useTheme } from '@/context/ThemeContext'

export function ThemeToggle() {
  const { isDark, toggleTheme } = useTheme()
  return (
    <button
      onClick={toggleTheme}
      aria-label={isDark ? 'Switch to light mode' : 'Switch to dark mode'}
      style={{
        width: 36, height: 36, borderRadius: '50%',
        border: '1px solid var(--border)',
        background: 'var(--surface)',
        cursor: 'pointer',
        display: 'flex', alignItems: 'center', justifyContent: 'center',
        color: 'var(--text2)',
      }}
    >
      {isDark ? <SunIcon /> : <MoonIcon />}
    </button>
  )
}
```

### Noise texture pattern

```css
body::before {
  content: '';
  position: fixed;
  inset: 0;
  background-image: url("data:image/svg+xml,...");
  pointer-events: none;
  z-index: 999;
  opacity: 0.2;
}
html[data-theme="light"] body::before { opacity: 0.05; }
```

---

## Common mistakes

**React Native:**
- Importing `lightColors` directly in screens — always use `useTheme().colors` so dark mode works
- `StyleSheet.create` outside the component — styles become stale when theme changes
- `useMemo(() => StyleSheet.create({...}), [])` — missing `[colors]` dep means it never updates
- Hardcoding `shadowColor: '#000'` — use `shadow.card` instead

**Web:**
- Color names (`--blue-500`) instead of semantic names — breaks when brand color changes
- Forgetting to override ALL vars in the alternate theme — some components stay dark in light mode
- `@keyframes` inside `@layer` — inline `animation:` won't find them
- Hardcoded hex values in component styles — won't theme correctly
- `useState(true)` for isDark — server/client mismatch → hydration error
- Flash prevention script defaulting to dark (`t === 'light' ? 'light' : 'dark'`) — use `t === 'dark' ? 'dark' : 'light'` to default light
- `ThemeProvider` directly in layout.tsx — layout is a server component, use the Providers wrapper
- `window.localStorage` inside render — crashes on server. Always inside `useEffect`.
