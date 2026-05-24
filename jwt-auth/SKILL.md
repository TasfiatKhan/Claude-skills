---
name: jwt-auth
description: Implement or audit JWT authentication — token lifecycle, refresh flow, blacklisting. Use when the user says "add auth", "implement JWT", "handle token refresh", or "audit the auth flow".
---

You are implementing or auditing JWT authentication.

## How JWT works

```
1. User logs in → server validates credentials → returns access + refresh tokens
2. Client stores tokens securely (SecureStore on mobile, httpOnly cookie on web)
3. Client sends access token in every request: Authorization: Bearer <token>
4. Access token expires (short-lived: 5–60 min)
5. Client uses refresh token to get a new access token (long-lived: 7–30 days)
6. Refresh token expires → user must log in again
```

## Reference implementation

### Backend (djangorestframework-simplejwt)

```python
# settings/base.py
INSTALLED_APPS = [..., 'rest_framework_simplejwt.token_blacklist']

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

from datetime import timedelta
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,        # new refresh token on every refresh
    'BLACKLIST_AFTER_ROTATION': False,    # TODO: enable before production
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

```python
# config/urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/auth/token/', TokenObtainPairView.as_view()),
    path('api/auth/token/refresh/', TokenRefreshView.as_view()),
    path('api/auth/register/', RegisterView.as_view()),
]
```

### Frontend (React Native + expo-secure-store)

```typescript
// src/services/authService.ts
import * as SecureStore from 'expo-secure-store'

const ACCESS_KEY  = '<app>_access_token'
const REFRESH_KEY = '<app>_refresh_token'

export async function login(email: string, password: string) {
  const res = await axios.post('/api/auth/token/', { email, password })
  await SecureStore.setItemAsync(ACCESS_KEY, res.data.access)
  await SecureStore.setItemAsync(REFRESH_KEY, res.data.refresh)
}

export async function logout() {
  await SecureStore.deleteItemAsync(ACCESS_KEY)
  await SecureStore.deleteItemAsync(REFRESH_KEY)
}
```

```typescript
// src/services/api.ts — Axios interceptors
api.interceptors.request.use(async config => {
  const token = await SecureStore.getItemAsync(ACCESS_KEY)
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

api.interceptors.response.use(
  res => res,
  async error => {
    if (error.response?.status === 401) {
      const refresh = await SecureStore.getItemAsync(REFRESH_KEY)
      if (refresh) {
        const res = await axios.post('/api/auth/token/refresh/', { refresh })
        await SecureStore.setItemAsync(ACCESS_KEY, res.data.access)
        // Retry original request with new token
        error.config.headers.Authorization = `Bearer ${res.data.access}`
        return api(error.config)
      }
      // Refresh failed — log out
      await logout()
    }
    return Promise.reject(error)
  }
)
```

## Token security rules

- **Access token** — short-lived (15–60 min). Safe to store in memory. If stolen, expires quickly.
- **Refresh token** — long-lived (7–30 days). Store in SecureStore (mobile) or httpOnly cookie (web). Never in localStorage.
- **Never log tokens** — not in console.log, not in Sentry, not in server logs
- **HTTPS only in production** — tokens in headers are plaintext over HTTP

## Blacklisting (enable before production)

```python
SIMPLE_JWT = {
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,  # invalidates old refresh token after rotation
}
```

Requires `rest_framework_simplejwt.token_blacklist` in `INSTALLED_APPS` and a migration.
Prevents refresh token reuse if one is stolen.

## Custom user model

Always define a custom User model at project start — adding one later requires a DB reset:

```python
# apps/users/models.py
class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []

# settings/base.py
AUTH_USER_MODEL = 'users.User'
```

## SecureStore — storage decisions and patterns

### SecureStore vs AsyncStorage

| Data | Storage |
|------|---------|
| JWT access token | `SecureStore` — encrypted |
| JWT refresh token | `SecureStore` — encrypted |
| Theme preference | `SecureStore` acceptable, or AsyncStorage |
| User ID | `SecureStore` acceptable |
| Large non-sensitive data | `AsyncStorage` — SecureStore has a 2048 byte limit |
| Cached API responses | `AsyncStorage` or in-memory |

**Rule:** anything that grants access or reveals identity → SecureStore. Preferences → either.

### Key naming convention

```typescript
// Define all keys as constants in one place — never scatter key strings across files
const ACCESS_KEY  = '<app>_access_token'
const REFRESH_KEY = '<app>_refresh_token'
const THEME_KEY   = '<app>_theme'
```

Use app-prefixed keys so multiple apps on a device don't collide.

### Reading on mount — AuthContext pattern

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

### Size limit workaround

SecureStore values are capped at 2048 bytes. For larger payloads:

```typescript
// Store a reference key in SecureStore, actual data in AsyncStorage
await AsyncStorage.setItem('<app>_profile_cache', JSON.stringify(profile))
await SecureStore.setItemAsync('<app>_cache_version', '1')

const version = await SecureStore.getItemAsync('<app>_cache_version')
const cached  = version ? await AsyncStorage.getItem('<app>_profile_cache') : null
```

### Availability check

SecureStore requires device hardware (Secure Enclave / Android Keystore). On very old Android it may fail:

```typescript
const available = await SecureStore.isAvailableAsync()
if (!available) {
  // Fallback to AsyncStorage (less secure)
}
```

---

## Common mistakes

- **Long-lived access tokens** — if stolen, attacker has extended access. Keep them short (≤60 min).
- **Storing refresh tokens in localStorage** — vulnerable to XSS. Use SecureStore or httpOnly cookies.
- **No refresh interceptor** — user gets logged out on every expired access token instead of silently refreshing.
- **Blacklisting disabled** — stolen refresh tokens remain valid until they naturally expire.
- **Not using a custom User model** — nearly impossible to change later without resetting migrations.
- **Not awaiting `getItemAsync`** — returns a Promise, not the value. Always `await` or `.then()`.
- **Forgetting `deleteItemAsync` on logout** — iOS keychain persists across reinstall; always delete on explicit logout.
- **Storing large JSON directly** — SecureStore silently fails or truncates over 2048 bytes. Large objects go in AsyncStorage.
- **Reading SecureStore synchronously** — there is no synchronous API. Use `isLoading` state to gate the UI.
