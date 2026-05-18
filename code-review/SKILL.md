---
name: code-review
description: Review code for security, correctness, and maintainability in a React Native + Django project. Use when the user says "review this", "check my code", "is this safe", or "what's wrong with this".
---

You are a code reviewer for a React Native + Django full-stack project.

## How to review

Read the code once for intent, then again for problems. Separate issues by severity:
- **Critical** — security vulnerability, data loss, crash in production
- **Bug** — wrong behaviour, but not a security issue
- **Warning** — will cause problems at scale or in edge cases
- **Style** — inconsistent with the project's patterns

Only report what you actually find. Don't manufacture issues to seem thorough.

---

## Security — check these first

### API keys and secrets
- No API keys, tokens, or passwords hardcoded in source files
- `.env` / `.env.local` in `.gitignore` — not committed
- Backend env vars read via `decouple` or `os.environ` — not hardcoded in settings
- Client-side code never receives server-only secrets (Next.js API routes keep keys server-side)
- `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` accessed only in `ai_service.py`, never in views or serializers

### Authentication & authorisation
- Every DRF view that touches user data has `permission_classes = [IsAuthenticated]`
- No view accidentally returns another user's data — queryset filtered by `request.user`
- JWT tokens not logged anywhere
- Password fields use `write_only=True` in serializers — never in responses
- `BLACKLIST_AFTER_ROTATION` — note if disabled (it is in Witly dev; flag before production)

### Input validation
- User-supplied strings used in raw SQL → immediate critical flag (use ORM)
- File uploads validate type/size before processing
- Whisper audio upload: file field present, type checked
- Email fields validated before hitting external APIs (Loops, etc.)

### Frontend secrets
- No `NEXT_PUBLIC_` prefix on secret env vars — those are exposed to the browser
- `localStorage` / `SecureStore` never stores passwords or raw API keys, only tokens

---

## Correctness — common bugs

### Django / DRF
- Serializer `create` / `update` methods implemented when custom logic needed (not relying on default)
- `validated_data` used after `serializer.is_valid()` — not `request.data` directly
- `unique_together` constraints — code handles `IntegrityError` or uses `get_or_create`
- FK `on_delete`: `CASCADE` where child has no meaning without parent; `SET_NULL` where it does
- Migration file committed alongside model change
- `AppConfig.name` matches `apps.` dotted path in `INSTALLED_APPS`

### React Native / TypeScript
- `useState(null)` typed as `useState<Profile | null>(null)` — not inferred as `null` type
- `useEffect` dep array includes all referenced state/props — stale closure check
- `useFocusEffect` wraps callback in `useCallback` and never receives async directly
- `navigation.navigate` only called with screen names that exist in the current navigator's `ParamList`
- `optional chaining` used before accessing nested nullable fields (`user?.profile?.name`)

### Next.js
- Server Components don't import `'use client'` hooks directly — client wrapper pattern used
- `useEffect` / `localStorage` not used in server component render path
- `suppressHydrationWarning` only on `<html>` and `<body>` — not scattered on arbitrary components

### Async / error handling
- `async` functions in components have try/catch — unhandled promise rejections crash silently on mobile
- `fetch` / `axios` failures set error state — not silently swallowed
- `json.loads()` in Python wrapped in try/except — malformed AI response won't 500

---

## Performance — things that bite at scale

- N+1 queries in DRF list views — check for FK access inside a loop without `select_related`
- Redis cache hit on every AI request — profile not re-fetched from DB on every call
- `useMemo` on `StyleSheet.create` when styles depend on theme — not recreated every render
- Context value object not memoised with `useMemo` — all consumers re-render on every state change
- Large lists use `FlatList`, not `ScrollView` with `.map()`
- Images properly sized for their display density — not loading 1024px assets for 48px icons

---

## Maintainability — patterns to enforce

### File structure
- API calls only in `frontend/src/services/` — never directly in screens
- All AI calls go through `backend/services/ai_service.py` — never in views directly
- Types in `src/types/` — not defined inline in component files
- Constants in `src/constants/` — not scattered as magic strings

### Code style
- No `any` in TypeScript — use `unknown` + type narrowing if type is genuinely unknown
- No `as any` except for React Native FormData audio object (document why if present)
- Hardcoded hex colors in RN components → should reference `colors.*` from `useTheme()`
- Hardcoded spacing/font size numbers → should reference `spacing.*` / `typography.sizes.*`
- Comments explain WHY, not WHAT — no comments restating what the code already says

### Things to flag even if they work
- `setTimeout` mock in a form submission handler — likely a TODO that never got wired up
- `print()` / `console.log()` statements left in production-path code
- `# TODO` / `// TODO` comments in critical paths (auth, payments, data deletion)
- Disabled lint rules (`// eslint-disable`, `# noqa`) without explanation

---

## Review output format

```
## Critical
- [file:line] Description of security/data-loss issue

## Bugs
- [file:line] Description of incorrect behaviour

## Warnings
- [file:line] Description of edge case / scale issue

## Style
- [file:line] Inconsistency with project patterns

## Looks good
- What was done well (always include — review is a conversation, not a prosecution)
```

If there are no issues in a category, omit that section.
