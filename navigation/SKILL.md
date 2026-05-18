---
name: navigation
description: Set up or audit React Navigation — stack navigators, auth flow gating, tab navigators. Use when the user says "add a screen", "set up navigation", "gate a screen behind auth", or "navigate to X".
---

You are setting up or auditing React Navigation in a React Native app.

## Stack structure

```
AppNavigator
  ├── AuthNavigator (if not authenticated)
  │     ├── Login
  │     └── Register
  └── MainNavigator (if authenticated)
        ├── Home
        ├── Dashboard
        ├── DetailView
        ├── Settings
        └── [feature screens...]
```

## Type definitions

```typescript
// src/navigation/types.ts
export type AuthStackParamList = {
  Login: undefined
  Register: undefined
}

export type MainStackParamList = {
  Home: undefined
  Dashboard: undefined
  DetailView: { itemId: number | null }
  Settings: undefined
}
```

## Navigator setup

```tsx
// src/navigation/AppNavigator.tsx
import { NavigationContainer } from '@react-navigation/native'
import { createNativeStackNavigator } from '@react-navigation/native-stack'
import { useAuth } from '../context/AuthContext'

const Stack = createNativeStackNavigator()

export default function AppNavigator() {
  const { isAuthenticated, isLoading } = useAuth()

  if (isLoading) return <LoadingScreen />

  return (
    <NavigationContainer>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <Stack.Screen name="Main" component={MainNavigator} />
        ) : (
          <Stack.Screen name="Auth" component={AuthNavigator} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

## Auth flow gating

The navigator conditionally renders Auth vs Main based on `isAuthenticated`. When auth state changes (login/logout), React Navigation automatically switches — no `navigate()` call needed.

```tsx
// AuthContext controls the switch
const { isAuthenticated } = useAuth()

// After login — just update state, navigator handles the rest
await SecureStore.setItemAsync(ACCESS_KEY, tokens.access)
setIsAuthenticated(true)   // → AppNavigator re-renders → Main stack shown

// After logout
await SecureStore.deleteItemAsync(ACCESS_KEY)
setIsAuthenticated(false)  // → AppNavigator re-renders → Auth stack shown
```

## Onboarding gate (within Main)

```tsx
// HomeScreen — redirect to setup if not onboarded
const { profile, isLoading } = useProfile()

useEffect(() => {
  if (!isLoading && profile && !profile.is_setup_complete) {
    navigation.replace('Setup')  // replace — not push, so back button doesn't return here
  }
}, [profile, isLoading])
```

## Navigating between screens

```tsx
// Type-safe navigation via Props
type Props = NativeStackScreenProps<MainStackParamList, 'Home'>

export default function HomeScreen({ navigation }: Props) {
  // Navigate to screen with no params
  navigation.navigate('Dashboard')

  // Navigate with params
  navigation.navigate('DetailView', { itemId: 42 })

  // Replace current screen (no back button)
  navigation.replace('Setup')

  // Go back
  navigation.goBack()
}
```

## Accessing route params

```tsx
type Props = NativeStackScreenProps<MainStackParamList, 'DetailView'>

export default function DetailScreen({ navigation, route }: Props) {
  const { itemId } = route.params   // typed — itemId: number | null
}
```

## Back button pattern

```tsx
// Custom back button in header
<TouchableOpacity onPress={() => navigation.goBack()} style={{ padding: 8 }}>
  <Text style={{ fontSize: 22, color: colors.text }}>←</Text>
</TouchableOpacity>
```

## useFocusEffect — reload data on screen focus

```tsx
import { useFocusEffect } from '@react-navigation/native'

useFocusEffect(
  useCallback(() => {
    // Runs every time this screen comes into focus
    async function fetchData() {
      const data = await itemService.listItems()
      setItems(data)
    }
    fetchData()
  }, [dep])   // re-runs when dependency changes
)
```

**Important**: never pass an async function directly to `useFocusEffect` — it returns a Promise, not a cleanup function. Define `async function` inside and call it.

## Common mistakes

- `navigation.navigate` in the wrong navigator — each navigator only knows its own screens
- Passing async to `useFocusEffect` directly — use the inner async function pattern
- Using `navigate` instead of `replace` for onboarding redirects — user can press back and return to the gate
- Missing `headerShown: false` — double header appears when nesting navigators
- Forgetting to add new screens to `ParamList` type — TypeScript won't catch invalid screen names
