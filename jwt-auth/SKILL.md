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

## Common mistakes

- **Long-lived access tokens** — if stolen, attacker has extended access. Keep them short (≤60 min).
- **Storing refresh tokens in localStorage** — vulnerable to XSS. Use SecureStore or httpOnly cookies.
- **No refresh interceptor** — user gets logged out on every expired access token instead of silently refreshing.
- **Blacklisting disabled** — stolen refresh tokens remain valid until they naturally expire.
- **Not using a custom User model** — nearly impossible to change later without resetting migrations.
