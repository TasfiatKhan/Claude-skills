---
name: state-management
description: Manage state with React Context, hooks, useMemo, and useCallback. Use when the user says "share state between screens", "add a hook", "why is this re-rendering", or "optimise this context".
---

You are managing state in a React Native app using Context API and hooks.

## When to use what

| Need | Solution |
|------|---------|
| Local UI state (loading, form values) | `useState` in the component |
| Shared state across many screens | React Context |
| Derived/computed values | `useMemo` |
| Stable callback references | `useCallback` |
| Server state (API data) | Custom hook with `useState` + `useEffect` |
| Side effects on mount | `useEffect` |
| Side effects on screen focus | `useFocusEffect` (React Navigation) |

## Context pattern

```tsx
// src/context/AuthContext.tsx
import { createContext, useContext, useEffect, useState } from 'react'

interface AuthContextValue {
  isAuthenticated: boolean
  isLoading: boolean
  signIn: (email: string, password: string) => Promise<void>
  signOut: () => Promise<void>
}

const AuthContext = createContext<AuthContextValue>({
  isAuthenticated: false,
  isLoading: true,
  signIn: async () => {},
  signOut: async () => {},
})

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    // Check stored token on mount
    async function checkAuth() {
      const token = await SecureStore.getItemAsync(ACCESS_KEY)
      setIsAuthenticated(!!token)
      setIsLoading(false)
    }
    checkAuth()
  }, [])

  const signIn = async (email: string, password: string) => {
    const tokens = await authService.login(email, password)
    await SecureStore.setItemAsync(ACCESS_KEY, tokens.access)
    setIsAuthenticated(true)
  }

  const signOut = async () => {
    await authService.logout()
    setIsAuthenticated(false)
  }

  return (
    <AuthContext.Provider value={{ isAuthenticated, isLoading, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  )
}

export const useAuth = () => useContext(AuthContext)
```

## Custom data hook pattern

```tsx
// src/hooks/useProfile.ts
export function useProfile() {
  const [profile, setProfile] = useState<Profile | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    async function fetch() {
      try {
        const data = await profileService.getProfile()
        setProfile(data)
      } finally {
        setIsLoading(false)
      }
    }
    fetch()
  }, [])

  const update = async (data: ProfileUpdate) => {
    const updated = await profileService.updateProfile(data)
    setProfile(updated)
  }

  return { profile, isLoading, update }
}
```

## useMemo — expensive computations

```tsx
// Recalculates only when colors changes
const styles = useMemo(() => StyleSheet.create({
  container: { backgroundColor: colors.background, flex: 1 },
  title: { color: colors.text, fontSize: typography.sizes.xl },
}), [colors])

// Derived value
const activeOptions = useMemo(
  () => options.filter(o => o.isActive),
  [options]
)
```

Use when: StyleSheet creation with theme colors (every theme toggle recreates styles), filtering/sorting large lists, expensive calculations.

Don't use when: the computation is trivial — `useMemo` has overhead too.

## useCallback — stable function references

```tsx
const toggleTheme = useCallback(() => {
  setIsDark(prev => {
    const next = !prev
    SecureStore.setItemAsync(THEME_KEY, next ? 'dark' : 'light')
    return next
  })
}, [])   // no deps — function never changes

// useFocusEffect requires useCallback
useFocusEffect(
  useCallback(() => {
    async function fetch() { ... }
    fetch()
  }, [dep])
)
```

Use when: passing a callback to a child component (prevents unnecessary re-renders), passing to `useFocusEffect` or `useEffect` deps.

## Per-item state (Record pattern)

```tsx
// Track state per item ID without an array of objects
const [itemFeedback, setItemFeedback] = useState<Record<number, string | null>>({})
const [itemCopied, setItemCopied] = useState<Record<number, string | null>>({})
const [expanded, setExpanded] = useState<Record<number, boolean>>({})

// Update one item
setItemFeedback(prev => ({ ...prev, [item.id]: 'liked' }))

// Read
const feedback = itemFeedback[item.id] ?? null
```

## State machine pattern

```tsx
type State = 'idle' | 'loading' | 'success' | 'error'
const [state, setState] = useState<State>('idle')

// In UI
{state === 'loading' && <ActivityIndicator />}
{state === 'success' && <SuccessMessage />}
{state === 'error' && <ErrorMessage />}
```

Beats boolean flags (`isLoading`, `isSuccess`, `isError`) — impossible to be in two states at once.

## Context optimisation with useMemo

```tsx
// Without memoisation — value object recreated on every render → all consumers re-render
<Context.Provider value={{ isDark, toggleTheme, colors }}>

// With memoisation — only re-renders consumers when isDark changes
const value = useMemo(
  () => ({ isDark, toggleTheme, colors: isDark ? darkColors : lightColors }),
  [isDark, toggleTheme]
)
<Context.Provider value={value}>
```

## Common mistakes

- Global context for everything — local `useState` is fine for UI state that doesn't need to be shared
- Missing deps in `useEffect` — stale closures read old values. Add all referenced state/props to the dep array.
- Passing async to `useFocusEffect` — returns Promise not cleanup. Use inner async function pattern.
- Context without `useMemo` on value — every state change re-renders all consumers
- `useCallback` on everything — it has overhead. Only use when the reference stability actually matters.
