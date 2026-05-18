---
name: design-tokens
description: Centralise colors, typography, spacing, and radii in a theme.ts file. Use when the user says "add a design system", "centralise styles", "apply consistent spacing", or "why do we have hardcoded hex colors everywhere".
---

You are building a design token system for a React Native app.

## What goes in theme.ts

```
src/theme.ts
  colors       — all semantic color values (light + dark variants)
  typography   — font sizes, line heights, font weights
  spacing      — xs/sm/md/lg/xl/xxl scale
  radii        — border radius scale
  shadow       — card shadow definitions
```

## Complete theme.ts (Witly reference)

```typescript
// src/theme.ts

export type AppColors = typeof lightColors

export const lightColors = {
  background:      '#FAFAF8',
  surface:         '#FFFFFF',
  surfaceSecondary:'#F2EDE6',
  border:          'rgba(139,100,65,0.15)',

  accent:          '#C4956A',
  accentLight:     'rgba(196,149,106,0.12)',

  text:            '#1C1610',
  textSecondary:   '#5C4E41',
  textMuted:       '#9A8070',
  placeholder:     '#C4B09A',

  safe:            '#3A7050',
  safeLight:       'rgba(58,112,80,0.12)',
  playful:         '#5B4E9A',
  playfulLight:    'rgba(91,78,154,0.12)',
  bold:            '#9A3828',
  boldLight:       'rgba(154,56,40,0.12)',

  error:           '#C0392B',
  success:         '#27AE60',
  copied:          '#27AE60',

  tabBar:          '#FFFFFF',
  tabBarBorder:    'rgba(139,100,65,0.12)',
  tabBarActive:    '#C4956A',
  tabBarInactive:  '#9A8070',
}

export const darkColors: AppColors = {
  background:      '#1A1A1A',
  surface:         '#242424',
  surfaceSecondary:'#2E2E2E',
  border:          'rgba(255,255,255,0.10)',

  accent:          '#C4956A',
  accentLight:     'rgba(196,149,106,0.15)',

  text:            '#F0EBE3',
  textSecondary:   '#B8A898',
  textMuted:       '#7A6A5A',
  placeholder:     '#5A4A3A',

  safe:            '#5A8F6B',
  safeLight:       'rgba(90,143,107,0.15)',
  playful:         '#7B6FA8',
  playfulLight:    'rgba(123,111,168,0.15)',
  bold:            '#B85C4A',
  boldLight:       'rgba(184,92,74,0.15)',

  error:           '#E74C3C',
  success:         '#2ECC71',
  copied:          '#2ECC71',

  tabBar:          '#1E1E1E',
  tabBarBorder:    'rgba(255,255,255,0.08)',
  tabBarActive:    '#C4956A',
  tabBarInactive:  '#7A6A5A',
}

export const typography = {
  sizes: {
    xs:   11,
    sm:   13,
    md:   15,
    lg:   17,
    xl:   22,
    xxl:  28,
    hero: 36,
  },
  lineHeights: {
    tight:  1.2,
    normal: 1.5,
    loose:  1.7,
  },
  weights: {
    regular: '400' as const,
    medium:  '500' as const,
    semibold:'600' as const,
    bold:    '700' as const,
  },
}

export const spacing = {
  xs:   4,
  sm:   8,
  md:   12,
  lg:   16,
  xl:   24,
  xxl:  40,
}

export const radii = {
  sm:   6,
  md:   12,
  lg:   18,
  xl:   24,
  full: 999,
}

export const shadow = {
  card: {
    shadowColor:   '#8B6441',
    shadowOffset:  { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius:  8,
    elevation:     3,
  },
}
```

## ThemeContext — switching at runtime

```tsx
// src/context/ThemeContext.tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState } from 'react'
import * as SecureStore from 'expo-secure-store'
import { lightColors, darkColors, AppColors } from '../theme'

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
    SecureStore.getItemAsync('witly_theme').then(v => {
      if (v === 'dark') setIsDark(true)
    })
  }, [])

  const toggleTheme = useCallback(() => {
    setIsDark(prev => {
      const next = !prev
      SecureStore.setItemAsync('witly_theme', next ? 'dark' : 'light')
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

## Using tokens in a screen

```tsx
import { useMemo } from 'react'
import { StyleSheet } from 'react-native'
import { useTheme } from '../context/ThemeContext'
import { typography, spacing, radii, shadow } from '../theme'

export default function HomeScreen() {
  const { colors } = useTheme()

  // Recreate styles only when colors change (dark/light toggle)
  const styles = useMemo(() => StyleSheet.create({
    container: {
      flex: 1,
      backgroundColor: colors.background,
      padding: spacing.lg,
    },
    card: {
      backgroundColor: colors.surface,
      borderRadius: radii.lg,
      padding: spacing.lg,
      ...shadow.card,
    },
    title: {
      fontSize: typography.sizes.xl,
      fontWeight: typography.weights.bold,
      color: colors.text,
    },
    subtitle: {
      fontSize: typography.sizes.sm,
      color: colors.textSecondary,
      marginTop: spacing.xs,
    },
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

## Option colors — mapping type to token

```typescript
// In any screen that shows AI options
const OPTION_COLORS = {
  safe:    { bg: colors.safeLight,    label: colors.safe    },
  playful: { bg: colors.playfulLight, label: colors.playful },
  bold:    { bg: colors.boldLight,    label: colors.bold    },
}

// Usage
const { bg, label } = OPTION_COLORS[option.type]
<View style={{ backgroundColor: bg, borderRadius: radii.md, padding: spacing.md }}>
  <Text style={{ color: label }}>{option.text}</Text>
</View>
```

## TextInput — always explicit color + placeholderTextColor

Without explicit props, inputs may be invisible in dark mode:

```tsx
<TextInput
  style={{ color: colors.text, backgroundColor: colors.surface }}
  placeholderTextColor={colors.placeholder}
  placeholder="What's the situation?"
/>
```

## Audit checklist — removing hardcoded values

Search for:
- `'#` (hex colors)
- `fontSize:` (font sizes as numbers)
- `padding:` / `margin:` (spacing as raw numbers)
- `borderRadius:` (raw radius numbers)

Replace with token references. Example:
```
fontSize: 13      → fontSize: typography.sizes.sm
padding: 16       → padding: spacing.lg
borderRadius: 12  → borderRadius: radii.md
color: '#9A8070'  → color: colors.textMuted
```

## Common mistakes

- Importing `lightColors` directly in screens — always use `useTheme().colors` so dark mode works
- `StyleSheet.create` outside the component — styles become stale when theme changes. Move inside component + wrap in `useMemo`
- Missing `useMemo` deps — `useMemo(() => StyleSheet.create({...}), [])` never updates. Add `[colors]`
- Hardcoding `shadowColor: '#000'` — use `shadow.card` or define in tokens
- Defining new raw colors inline when a token already exists — check `theme.ts` first
