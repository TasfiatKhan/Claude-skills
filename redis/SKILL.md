---
name: redis
description: Design or audit Redis caching — caching strategy, TTL management, cache invalidation. Use when the user says "add caching", "cache this", "set up Redis", or "audit the cache layer".
---

You are designing or auditing a Redis caching layer.

## Core principle

Cache to avoid repeating expensive work. Every cache entry needs three things defined upfront: **what** is cached, **how long** it lives (TTL), and **when** it's invalidated.

## When to cache

Cache when:
- The data is read far more often than it's written (profile data, config)
- The computation is expensive (AI calls, complex DB aggregates)
- The same data is fetched on every request (user profile on every AI call)

Don't cache when:
- Data changes frequently and staleness matters (real-time state)
- The computation is fast and the data is small (no meaningful gain)
- You haven't measured the problem (premature optimization)

## Key naming convention

```python
# Pattern: <app>:<entity>:<identifier>
PROFILE_CACHE_KEY = "profile:{user_id}"       # profile:42
MOMENT_CACHE_KEY  = "moment:{moment_id}"      # moment:17
SESSION_KEY       = "session:{user_id}:{token}"
```

Rules:
- Always namespace by app — prevents key collisions across services
- Include the identifier — never cache all users under one key
- Keep keys short but readable

## Witly reference implementation

```python
# backend/apps/profiles/cache.py
import json
from services.redis_client import redis_client

PROFILE_CACHE_TTL = 3600  # 1 hour — read from settings in practice

def get_cached_profile(user_id: int) -> dict | None:
    key = f"profile:{user_id}"
    data = redis_client.get(key)
    return json.loads(data) if data else None

def set_cached_profile(user_id: int, profile_data: dict) -> None:
    key = f"profile:{user_id}"
    redis_client.setex(key, PROFILE_CACHE_TTL, json.dumps(profile_data))

def invalidate_cached_profile(user_id: int) -> None:
    key = f"profile:{user_id}"
    redis_client.delete(key)
```

**Usage pattern — Redis-first:**
```python
def get_profile(user_id):
    # 1. Try cache first
    cached = get_cached_profile(user_id)
    if cached:
        return cached

    # 2. Miss — hit DB
    profile = UserProfile.objects.get(user_id=user_id)
    data = ProfileSerializer(profile).data

    # 3. Prime cache for next time
    set_cached_profile(user_id, data)
    return data
```

**Invalidate on write:**
```python
def update_profile(user_id, data):
    profile = UserProfile.objects.get(user_id=user_id)
    # ... update fields ...
    profile.save()
    invalidate_cached_profile(user_id)  # always invalidate after write
    set_cached_profile(user_id, ProfileSerializer(profile).data)  # re-prime
```

## TTL strategy

| Data type | Suggested TTL | Reason |
|-----------|--------------|--------|
| User profile / preferences | 1 hour | Changes rarely, read constantly |
| Session / auth token | Match token expiry | Security-critical |
| AI response cache | 5–15 min | Expensive to compute, short enough to stay fresh |
| Config / feature flags | 5 min | Want changes to propagate quickly |
| Rate limit counters | Match the window | e.g. 60s for per-minute limits |

## Common mistakes

- **No TTL** — `redis_client.set(key, value)` without expiry. Data lives forever, memory grows unbounded. Always use `setex` or pass `ex=`.
- **Forgetting to invalidate** — profile updated in DB but cache still serves stale data. Always invalidate on write.
- **Caching the wrong thing** — caching a full queryset when you only need one field. Cache the minimal shape you need.
- **Key collisions** — `redis_client.set("user", data)` across two services. Always namespace.
- **Serialization mismatch** — storing a dict with `json.dumps` but reading without `json.loads`. Be consistent.
- **Cache stampede** — many requests miss the cache simultaneously and all hit the DB. Mitigate with a short lock or staggered TTLs.

## Redis client setup (Django)

```python
# services/redis_client.py
import redis
from django.conf import settings

redis_client = redis.Redis.from_url(settings.REDIS_URL, decode_responses=True)
```

```python
# settings/base.py
REDIS_URL = config('REDIS_URL', default='redis://localhost:6379/0')
```

```yaml
# docker-compose.yml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
```
