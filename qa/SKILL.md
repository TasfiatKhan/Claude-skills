---
name: qa
description: QA checklist for React Native + Django full-stack apps. Use when the user says "test this", "what could go wrong", "check before release", or "write test cases".
---

You are QA for a React Native + Django full-stack app.

## The core question for every feature

For each piece of functionality, ask:
1. What happens on the happy path?
2. What happens when the network fails mid-request?
3. What happens when the server returns an unexpected shape?
4. What happens when the user does it twice fast?
5. What happens on a slow device / slow connection?

---

## Frontend — React Native checklist

### State & data
- [ ] Loading state shown while data is fetching
- [ ] Empty state shown when list returns zero items
- [ ] Error state shown when request fails (not just silent failure)
- [ ] Data persists correctly after navigating away and returning (`useFocusEffect` reload)
- [ ] State reset when it should be (e.g. form clears after submit)
- [ ] No stale closure reading old state in `useEffect` or `useCallback`

### Navigation
- [ ] Back button works and goes to the right screen
- [ ] `replace` used instead of `navigate` for auth gates and onboarding (no back-button escape)
- [ ] Deep navigation state survives app backgrounding
- [ ] Screen doesn't flash the previous state before loading new data

### Inputs & forms
- [ ] Submit disabled while loading (no double-submit)
- [ ] Keyboard dismisses on submit
- [ ] `placeholderTextColor` set explicitly (invisible on dark backgrounds otherwise)
- [ ] `autoComplete` and `keyboardType` set correctly on email/password fields
- [ ] Error message shown inline, not just a border change
- [ ] Form resets or preserves state correctly after error

### Device / platform
- [ ] Tested on physical Android device, not just emulator
- [ ] Shadows use `elevation` on Android, `shadow*` props on iOS
- [ ] `KeyboardAvoidingView` behavior: `padding` on iOS, `height` on Android
- [ ] `includeFontPadding: false` on Android text that needs precise vertical alignment
- [ ] Audio permissions requested before first record attempt (if audio feature present)
- [ ] App icon appears correctly on device (not the default robot)

### Known React Native / Expo gotchas
- **Hermes doesn't support `ReadableStream`** — use `axios` with `responseType: 'json'`, not fetch streaming
- **`useFocusEffect` + async** — never pass async directly; use inner `async function fetch()` pattern
- **`expo prebuild` resets mipmap directories** — re-apply icon after every prebuild (Android bare workflow)
- **SecureStore has a 2048 byte limit** — store large data in AsyncStorage, reference key in SecureStore
- **`registerRootComponent` required** — without `index.js` entry point, app fails on physical device (bare workflow)

---

## Backend — Django / DRF checklist

### API endpoints
- [ ] Unauthenticated request returns 401, not 500
- [ ] Missing required fields return 400 with clear error message, not 500
- [ ] Malformed JSON body returns 400
- [ ] Extra/unknown fields in request body are ignored (not crash)
- [ ] Response shape is consistent — same fields present whether list is empty or not

### Database
- [ ] No N+1 queries — use `select_related` / `prefetch_related` on FKs in list views
- [ ] Migrations applied before testing new model fields
- [ ] `on_delete` behaviour tested for FK relationships (SET_NULL vs CASCADE)
- [ ] Unique constraints tested — duplicate submissions return 400 not 500

### Auth & permissions
- [ ] JWT-protected endpoints reject expired tokens (401)
- [ ] Users cannot access other users' data (test with two accounts)
- [ ] Onboarding / setup gate enforced before gated features
- [ ] Sensitive fields (`password`, API keys) never appear in API responses

### AI / external services
- [ ] Empty input to AI endpoint handled before hitting the API
- [ ] AI response JSON parse failure handled gracefully (try/except around `json.loads`)
- [ ] Audio transcription < threshold characters rejected before AI call (if voice feature)
- [ ] Rate limit / timeout from AI provider handled with user-facing error, not 500

---

## Integration — full-stack checklist

- [ ] Device can reach backend (ALLOWED_HOSTS includes device's local IP in dev)
- [ ] Token refresh flow works: expired access token → auto-refresh → retry original request
- [ ] Logout clears tokens from SecureStore AND blacklists on backend (if enabled)
- [ ] Redis cache invalidated when profile updated — AI doesn't serve stale data
- [ ] `docker compose up` brings all services up cleanly from scratch on a fresh machine

---

## Pre-release checklist

- [ ] No debug `print` / `console.log` statements left in production code
- [ ] No hardcoded IPs or local URLs in frontend service files
- [ ] `.env` / `.env.local` files not committed (check `.gitignore`)
- [ ] API keys not in source code or logs
- [ ] Error responses don't leak stack traces or internal model details
- [ ] Migrations committed alongside model changes

---

## Edge cases by feature type

| Feature type | Edge cases to test |
|-------------|------------------|
| Paginated list | Empty list, single item, last page has fewer items than page size |
| Item with cap | At cap — creation returns 403; below cap — succeeds |
| Toggle/undo action | Same action twice → undone; different action → replaces |
| Voice input | Empty/silent recording handled gracefully (< threshold transcription) |
| File upload | Wrong type rejected; oversized file rejected; missing file returns 400 |
| Multi-turn thread | At message cap — auto-archives; unarchive at cap fails with 403 |
| Copy tracking | Duplicate copy of same item → `get_or_create`, not duplicate row |
