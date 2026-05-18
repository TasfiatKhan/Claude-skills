---
name: secure-storage
description: Persist tokens, preferences, and sensitive data with expo-secure-store. Use when the user says "save a token", "persist auth state", "store user preferences", or "why is the user logged out after restart".
---

You are managing secure persistent storage in a React Native + Expo bare workflow app.

## When to use expo-secure-store vs AsyncStorage

| Data | Storage |
|------|---------|
| JWT access token | `SecureStore` — encrypted |
| JWT refresh token | `SecureStore` — encrypted |
| Theme preference | `SecureStore` — acceptable, or AsyncStorage |
| User ID | `SecureStore` — acceptable |
| Large non-sensitive data | `AsyncStorage` — SecureStore has a 2048 byte limit |
| Cached API responses | `AsyncStorage` or in-memory |

**Rule:** anything that grants access or reveals identity → SecureStore. Preferences → either.

## Key naming convention

```typescript
// src/services/apiClient.ts  (or src/constants/storage.ts)
const ACCESS_KEY  = '<app>_access_token'
const REFRESH_KEY = '<app>_refresh_token'
const THEME_KEY   = '<app>_theme'
```

Use app-prefixed keys so multiple apps on a device don't collide. Define all keys as constants in one place — never scatter key strings across files.

## Auth token storage pattern

```typescript
import * as SecureStore from 'expo-secure-store'

// Write — after login
await SecureStore.setItemAsync(ACCESS_KEY, tokens.access)
await SecureStore.setItemAsync(REFRESH_KEY, tokens.refresh)

// Read — on app start or before an API call
const token = await SecureStore.getItemAsync(ACCESS_KEY)
if (!token) {
  // not authenticated — redirect to login
}

// Delete — on logout
await SecureStore.deleteItemAsync(ACCESS_KEY)
await SecureStore.deleteItemAsync(REFRESH_KEY)
```

## Reading on mount — AuthContext pattern

```tsx
// src/context/AuthContext.tsx
useEffect(() => {
  async function checkAuth() {
    const token = await SecureStore.getItemAsync(ACCESS_KEY)
    setIsAuthenticated(!!token)
    setIsLoading(false)
  }
  checkAuth()
}, [])
```

`isLoading` stays `true` until the async read completes — prevents a flash to the login screen before the token is found.

## Axios interceptor — attach token to every request

```typescript
// src/services/apiClient.ts
api.interceptors.request.use(async config => {
  const token = await SecureStore.getItemAsync(ACCESS_KEY)
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

## Token refresh interceptor

```typescript
api.interceptors.response.use(
  res => res,
  async error => {
    const original = error.config
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true
      try {
        const refresh = await SecureStore.getItemAsync(REFRESH_KEY)
        const { data } = await axios.post(`${BASE_URL}/api/auth/token/refresh/`, { refresh })
        await SecureStore.setItemAsync(ACCESS_KEY, data.access)
        original.headers.Authorization = `Bearer ${data.access}`
        return api(original)
      } catch {
        // Refresh failed — force logout
        await SecureStore.deleteItemAsync(ACCESS_KEY)
        await SecureStore.deleteItemAsync(REFRESH_KEY)
        // AuthContext will detect missing token and show Login
      }
    }
    return Promise.reject(error)
  }
)
```

## Theme preference

```typescript
// ThemeContext.tsx (React Native version)
const THEME_KEY = '<app>_theme'

useEffect(() => {
  async function load() {
    const stored = await SecureStore.getItemAsync(THEME_KEY)
    if (stored === 'dark') setIsDark(true)
    setIsLoading(false)
  }
  load()
}, [])

const toggleTheme = useCallback(() => {
  setIsDark(prev => {
    const next = !prev
    SecureStore.setItemAsync(THEME_KEY, next ? 'dark' : 'light')
    return next
  })
}, [])
```

Note: `SecureStore.setItemAsync` is intentionally not awaited in toggle — fire and forget is fine for preferences.

## Size limit workaround

SecureStore values are capped at 2048 bytes. For larger payloads:

```typescript
// Store a reference key in SecureStore, actual data in AsyncStorage
import AsyncStorage from '@react-native-async-storage/async-storage'

// Write
await AsyncStorage.setItem('<app>_profile_cache', JSON.stringify(profile))
await SecureStore.setItemAsync('<app>_cache_version', '1')

// Read
const version = await SecureStore.getItemAsync('<app>_cache_version')
const cached = version ? await AsyncStorage.getItem('<app>_profile_cache') : null
```

## Availability check

SecureStore requires device hardware (Secure Enclave / Android Keystore). On simulators it still works. On very old Android it may fail:

```typescript
const available = await SecureStore.isAvailableAsync()
if (!available) {
  // Fallback to AsyncStorage (less secure)
}
```

## Common mistakes

- Not awaiting `getItemAsync` — returns a Promise, not the value. Always `await` or `.then()`.
- Forgetting `deleteItemAsync` on logout — user uninstalls and reinstalls but token is still in keychain (iOS). Always delete on explicit logout.
- Storing large JSON directly — SecureStore will silently fail or truncate. Store small primitives; large objects go in AsyncStorage with a SecureStore key.
- Reading SecureStore synchronously — there is no synchronous API. Use `isLoading` state to gate the UI while reading.
- Hardcoded key strings scattered across files — define all keys as constants in one place.
