---
name: layered-architecture
description: Design or audit a layered architecture — keeping AI calls, business logic, and transport separate. Use when the user says "how should I structure this", "where does this code go", "audit the architecture", or "set up service layers".
---

You are designing or auditing a layered architecture for an AI-powered application.

## Core principle

Each layer has one job. No layer reaches past its neighbor. The further in you go, the less it knows about HTTP, UI, or the outside world.

## The three layers

```
Transport layer       — HTTP, WebSocket, CLI, React Native screens
    ↓
Business logic layer  — what the app does, rules, orchestration
    ↓
Service layer         — external calls: AI API, DB, cache, email, storage
```

### Transport layer (views, screens, routes)
- Receives requests, validates input shape, returns responses
- Knows about HTTP status codes, request/response format
- Does NOT contain business logic or direct API calls
- Examples: Django views, DRF serializers, React Native screens, Next.js route handlers

### Business logic layer (services, use cases)
- Orchestrates what happens: calls the right services in the right order
- Enforces rules: caps, permissions, rate limits, state transitions
- Does NOT know about HTTP or UI
- Examples: `ai_service.py`, domain service modules, profile cache logic

### Service layer (external integrations)
- Thin wrappers around external systems: AI API, email, storage, Redis
- No business rules — just "call this, return that"
- Swappable: changing from one AI provider to another only touches this layer
- Examples: AI API client, `redis_client.py`, ORM queries in managers

## Reference implementation

```
frontend/
  screens/          ← transport: renders UI, handles user input
  services/         ← all API calls — screens never call fetch() directly
  context/          ← shared state (auth, theme)

backend/
  apps/<feature>/views.py   ← transport: validates request, returns JSON
  services/ai_service.py    ← business + service: assembles prompt, calls AI
  prompts/v2/               ← configuration: prompt templates, versioned
  apps/<feature>/cache.py   ← service: Redis get/set for cached data
```

**Key rule:**
- Views never call the AI client directly — always through `ai_service`
- Screens never call `axios.get()` directly — always through `src/services/`
- This means: swapping the AI model = one file change. Swapping axios for fetch = one file change.

## When to apply this

### Adding a new feature
1. Start at the service layer — what external calls are needed?
2. Move up to business logic — what rules govern this feature?
3. Wire the transport layer last — what does the endpoint/screen look like?

### Auditing existing code
Look for these violations:
- API calls (`axios`, `fetch`, AI client) inside screens or views → move to services
- Business rules (caps, permissions, state checks) inside views → move to a service
- HTTP concepts (`request.data`, status codes) inside service files → move to views
- Duplicated logic across multiple views → extract to a shared service

### Signs the architecture is clean
- You can write a test for business logic without spinning up HTTP
- Swapping an external provider touches exactly one file
- A new developer can find where to add a feature without reading everything
- Views are thin — mostly validation + one service call + return response

## Common mistakes

- **Fat views** — views that contain all the logic. Move anything beyond input/output to a service.
- **Screens calling APIs directly** — always goes through a service module
- **Business rules in the DB layer** — model methods that enforce application logic. Keep models as data shapes, put rules in services.
- **Circular imports** — usually a sign two layers are tangled. Introduce an interface or restructure.
- **God service** — one `utils.py` or `helpers.py` that does everything. Split by domain.
