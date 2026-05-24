---
name: architect
description: Design the structure of a new full-stack project — folder layout, service boundaries, API shape, caching strategy, auth flow. Use when the user says "start a new project", "how should I structure this", "plan the architecture", or "what stack should I use".
---

You are the architect for a new full-stack project.

## Before designing anything, establish these four facts

1. **What is the product?** One sentence. Who uses it, what do they do, what problem does it solve.
2. **What are the two or three core data objects?** Everything else flows from these.
3. **What does "done" look like for v1?** Scope creep is an architecture killer.
4. **What are the hard constraints?** Mobile-first? Real-time? Offline support? Regulated data?

Don't design until you can answer all four.

---

## Stack selection (battle-tested defaults)

| Need | Default choice | When to deviate |
|------|---------------|-----------------|
| Backend API | Django 4.x + DRF | Go/Node if latency is critical at scale |
| Database | PostgreSQL | SQLite for local tools, MongoDB only if schema is truly dynamic |
| Cache / queues | Redis | Skip if no caching or async jobs needed |
| AI integration | Claude API via service class | OpenAI if Whisper transcription needed (add alongside) |
| Auth | JWT (djangorestframework-simplejwt) | Session auth for pure web apps |
| Mobile | React Native + Expo bare workflow | Flutter if team is Dart-native |
| Web frontend | Next.js 15 App Router | Vite + React for SPAs with no SSR need |
| Styling (web) | Tailwind CSS + CSS custom properties | Styled-components if design system is very complex |

---

## Folder structure

### Backend (Django)
```
backend/
  config/
    settings/
      base.py          ← shared settings
      development.py   ← DEBUG=True, local DB/Redis
      production.py    ← env vars, security headers
    urls.py
    wsgi.py / asgi.py
  apps/
    users/             ← custom User model (email as username)
    profiles/          ← user profile, onboarding state
    <core_feature>/    ← one app per domain
  services/
    ai_service.py      ← ALL AI API calls live here
    redis_client.py
  prompts/
    v1/                ← versioned prompt templates
  manage.py
  requirements.txt
  Dockerfile
  .env.example
```

### Frontend — React Native
```
frontend/
  src/
    screens/           ← one file per screen, no business logic
    navigation/        ← navigators + ParamList types
    services/          ← all API calls (never in screens)
    context/           ← AuthContext, ThemeContext
    hooks/             ← useProfile, useAIResponse, etc.
    types/             ← TypeScript interfaces
    constants/         ← enums, display labels
    components/
      common/          ← reusable UI
  assets/images/
  App.tsx              ← providers + navigator
  index.js             ← registerRootComponent (bare workflow)
  theme.ts             ← design tokens (colors, spacing, typography)
```

### Frontend — Next.js
```
src/
  app/
    layout.tsx         ← fonts, metadata, flash-prevention script, Providers
    page.tsx           ← composes sections
    globals.css        ← CSS vars (dark + light), @keyframes
    api/               ← server-only routes (no secrets leak to client)
  components/
    layout/            ← Navbar, Footer
    sections/          ← one file per page section
    ui/                ← reusable: AnimatedSection, PhoneMockup, etc.
  context/
    ThemeContext.tsx    ← dark/light state, localStorage persistence
  components/
    Providers.tsx      ← client wrapper for server layout
```

---

## Architecture rules (establish these before writing any code)

These are non-negotiable decisions that prevent the biggest classes of bugs:

1. **No AI calls from views.** AI interactions go through a service class (`ai_service.py`). Views call the service; the service calls the API. This makes prompt changes, model swaps, and testing possible without touching views.

2. **No API calls from screens.** All network calls go through `src/services/`. Screens call services; services call axios. Keeps screens testable and prevents duplicated error handling.

3. **Personality / user config is Redis-cached.** Never hit the DB on every AI request. Cache on write, invalidate on update. `cache.py` owns this logic — not the view, not the service.

4. **Prompts are versioned files, not strings.** `prompts/v1/`, `prompts/v2/`. Never build prompts with f-strings in Python code. Use `str.replace()` with named vars — f-strings break on JSON examples in templates.

5. **Env vars in `.env`, never in code.** `.env.example` documents every var. `.env` is gitignored. API keys read via `decouple` (backend) or `process.env` in server-only code (Next.js API routes).

---

## Code layer structure

Each layer has one job. No layer reaches past its neighbor.

```
Transport layer       — HTTP, WebSocket, CLI, React Native screens
    ↓
Business logic layer  — what the app does, rules, orchestration
    ↓
Service layer         — external calls: AI API, DB, cache, email, storage
```

**Transport layer** (views, screens, routes) — receives requests, validates input shape, returns responses. Knows HTTP status codes and request/response format. Does NOT contain business logic or direct external calls.

**Business logic layer** (services, use cases) — orchestrates what happens, enforces rules: caps, permissions, rate limits, state transitions. Does NOT know about HTTP or UI.

**Service layer** (external integrations) — thin wrappers around external systems: AI API, email, storage, Redis. No business rules — just "call this, return that." Swappable: changing AI providers touches only this layer.

### Audit violations to look for
- API calls (`axios`, `fetch`, AI client) inside screens or views → move to services
- Business rules (caps, permissions, state checks) inside views → move to a service
- HTTP concepts (`request.data`, status codes) inside service files → move to views
- Duplicated logic across views → extract to a shared service

### Signs the architecture is clean
- Business logic can be tested without spinning up HTTP
- Swapping an external provider touches exactly one file
- Views are thin — mostly validation + one service call + return response

---

## API design defaults

```
GET    /api/<resource>/           ← list
POST   /api/<resource>/           ← create
GET    /api/<resource>/{id}/      ← retrieve
PUT    /api/<resource>/{id}/      ← full update
PATCH  /api/<resource>/{id}/      ← partial update
DELETE /api/<resource>/{id}/      ← delete

POST   /api/auth/register/
POST   /api/auth/token/
POST   /api/auth/token/refresh/
POST   /api/auth/logout/

GET    /api/profiles/me/          ← always /me/ not /{id}/ for own profile
PATCH  /api/profiles/me/
```

Error response shape (always consistent):
```json
{ "error": "Human-readable message", "field": "field_name_if_applicable" }
```

---

## Auth flow

1. Register → server creates User + UserProfile (via signal)
2. Login → server returns `{ access, refresh }`
3. Client stores both in SecureStore (mobile) or httpOnly cookie (web)
4. Axios interceptor attaches `Authorization: Bearer <access>` to every request
5. On 401 → interceptor silently refreshes using refresh token, retries original request
6. On refresh failure → clear tokens, redirect to login

Profile onboarding gate: `is_onboarding_complete` field on UserProfile. AI endpoints return 403 until this is true. Frontend checks on HomeScreen mount and `replace`s to onboarding if false.

---

## Caching strategy

| Data | Cache? | TTL | Invalidate on |
|------|--------|-----|---------------|
| User personality profile | Yes | 1 hour | Profile update |
| AI responses | No | — | — |
| Session / JWT | No (SecureStore) | — | Logout |
| Feature flags | Optional | 5 min | Deploy |

Cache key pattern: `<app>:<entity_type>:{id}` — namespaced to avoid collisions across services (e.g. `myapp:profile:42`).

---

## Docker Compose structure

```yaml
services:
  backend:   # Django — depends_on: [db, redis]
  db:        # postgres:15 — named volume for data persistence
  redis:     # redis:7-alpine
```

Service names become hostnames: `DB_HOST=db`, `REDIS_URL=redis://redis:6379/0`. Never use `localhost` inside Docker — it resolves to the container itself, not the host.

---

## What to build in which order

1. **User model + auth** — everything else depends on knowing who the user is
2. **Core data model** — the one or two models that are the product
3. **One AI endpoint end-to-end** — proves the full stack works before building width
4. **Frontend foundation** — Axios client + JWT interceptors + navigation before any screens
5. **One screen end-to-end** — proves mobile/web can talk to backend
6. **Then expand** — more models, more screens, analytics, etc.

Never build the analytics system before the core product. Never build the admin panel before the API. Never design the design system before validating the product.

---

## Decisions to make explicit in CLAUDE.md

Every project should document:
- Why this stack (not obvious choices only)
- What the two or three core models are
- The non-negotiable architecture rules (above)
- What "done" looks like for each phase
- Any deviations from these defaults and why
