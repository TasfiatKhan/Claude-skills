---
name: theme-context
description: Implement dark/light theme switching with React context — localStorage persistence, flash prevention, suppressHydrationWarning. Use when the user says "add a theme toggle", "persist the theme", "fix theme flash", or "fix hydration mismatch".
---

You are implementing a dark/light theme toggle with React context in a Next.js app.

## The three problems to solve

1. **State** — which theme is active, toggle function, persisted across sessions
2. **Flash** — page loads dark, then flickers to light (or vice versa) after JS hydrates
3. **Hydration mismatch** — server renders one theme, browser extensions inject attributes, React sees a difference and warns

## Complete implementation

### ThemeContext.tsx

```tsx
'use client'

import { createContext, useContext, useEffect, useState } from 'react'

interface ThemeContextValue {
  isDark: boolean
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextValue>({
  isDark: false,           // default: light mode
  toggleTheme: () => {},
})

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [isDark, setIsDark] = useState(false)   // default: light

  useEffect(() => {
    const stored = localStorage.getItem('witly-theme')
    const dark = stored ? stored === 'dark' : false  // fallback: light
    setIsDark(dark)
    document.documentElement.setAttribute('data-theme', dark ? 'dark' : 'light')
  }, [])

  const toggleTheme = () => {
    setIsDark(prev => {
      const next = !prev
      localStorage.setItem('witly-theme', next ? 'dark' : 'light')
      document.documentElement.setAttribute('data-theme', next ? 'dark' : 'light')
      return next
    })
  }

  return (
    <ThemeContext.Provider value={{ isDark, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export const useTheme = () => useContext(ThemeContext)
```

### Providers.tsx (client wrapper for server layout)

```tsx
'use client'

import { ThemeProvider } from '@/context/ThemeContext'

export default function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider>{children}</ThemeProvider>
}
```

### layout.tsx

```tsx
import Providers from '@/components/Providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <head>
        {/* Flash prevention: runs synchronously before React hydrates */}
        <script dangerouslySetInnerHTML={{ __html:
          `try{var t=localStorage.getItem('witly-theme');document.documentElement.setAttribute('data-theme',t==='dark'?'dark':'light')}catch(e){}`
        }} />
      </head>
      <body suppressHydrationWarning>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

## Why each piece matters

### The flash prevention script

Without it:
1. Server sends HTML with no `data-theme`
2. Browser paints (light mode — CSS default)
3. JS loads, `useEffect` runs, reads localStorage, sets dark mode
4. Page re-renders → visible flash

With the inline script:
- Runs synchronously in `<head>` before the browser paints anything
- Sets `data-theme` before first paint
- No flash

The script is minified intentionally — it runs on every page load, size matters.

### suppressHydrationWarning

Browser extensions (Grammarly, password managers) inject attributes like `cz-shortcut-listen="true"` into `<html>` and `<body>`. Our own inline script adds `data-theme`. React sees these attributes didn't exist in the server render and warns about hydration mismatches.

`suppressHydrationWarning` on `<html>` and `<body>` tells React: "I know these elements may have attribute differences — it's intentional, ignore them."

Only use it on `<html>` and `<body>` — not on arbitrary components where a real mismatch might need to be caught.

### Why useState defaults match

Server renders: `isDark = false` → light mode markup
Client initial render: `isDark = false` → light mode markup ✓ (match)
After `useEffect`: reads localStorage, potentially switches to dark

The match between server and client initial render is what prevents hydration errors in the component tree. The `useEffect` switch happens after hydration, so React doesn't care.

## Toggle button (Navbar)

```tsx
'use client'
import { useTheme } from '@/context/ThemeContext'

function SunIcon() {
  return (
    <svg width="16" height="16" viewBox="0 0 24 24" fill="none"
      stroke="currentColor" strokeWidth="2" strokeLinecap="round" aria-hidden="true">
      <circle cx="12" cy="12" r="4" />
      <path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41" />
    </svg>
  )
}

function MoonIcon() {
  return (
    <svg width="16" height="16" viewBox="0 0 24 24" fill="none"
      stroke="currentColor" strokeWidth="2" strokeLinecap="round" aria-hidden="true">
      <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z" />
    </svg>
  )
}

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

## React Native equivalent

In React Native, replace `localStorage` with `expo-secure-store` and `document.documentElement.setAttribute` with a state-driven style system:

```tsx
// context reads from SecureStore on mount
useEffect(() => {
  SecureStore.getItemAsync('theme_preference').then(value => {
    if (value === 'dark') setIsDark(true)
  })
}, [])

// Components consume colors from context
const { colors } = useTheme()
<View style={{ backgroundColor: colors.background }}>
```

No flash problem in React Native — no server render.

## Common mistakes

- `useState(true)` for isDark default when you want light mode first — use `false`
- `stored ? stored === 'dark' : true` — this defaults to dark. Use `: false` to default to light.
- Flash prevention script defaulting to dark: `t === 'light' ? 'light' : 'dark'` — if no stored value it falls through to dark. Use `t === 'dark' ? 'dark' : 'light'` to default light.
- Missing `suppressHydrationWarning` — hydration warnings from extensions
- Putting ThemeProvider directly in layout.tsx — layout is a server component, context requires `'use client'`. Use the Providers wrapper pattern.
- Using `window.localStorage` inside render — crashes on server. Always inside `useEffect`.
