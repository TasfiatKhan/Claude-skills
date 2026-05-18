---
name: api-design
description: Design or audit RESTful API endpoints — URL structure, request validation, structured error responses. Use when the user says "design an API", "add an endpoint", "audit the API structure", or "how should this endpoint work".
---

You are designing or auditing a RESTful API.

## Core principles

- **Resources, not actions** — URLs identify things, HTTP verbs identify what you do to them
- **Consistent structure** — every endpoint follows the same shape for requests and responses
- **Fail loudly with useful errors** — a 200 with `{error: true}` in the body is a design smell
- **Validate at the boundary** — serializers validate input so views don't have to

## URL design

```
GET    /api/moments/              → list user's moments
POST   /api/moments/              → create a moment
GET    /api/moments/{id}/         → get one moment
PATCH  /api/moments/{id}/         → update a moment
DELETE /api/moments/{id}/         → delete a moment

POST   /api/moments/{id}/continue/   → sub-action on a resource
PATCH  /api/moments/{id}/archive/    → sub-action (toggle archive)
```

Rules:
- Plural nouns for collections: `/moments/` not `/moment/`
- Nest sub-actions under the resource: `/moments/{id}/continue/` not `/continue-moment/{id}/`
- Never use verbs in URLs: not `/getMoments/`, not `/createMoment/`
- Always trailing slash in Django

## HTTP verbs

| Verb | Use | Idempotent |
|------|-----|-----------|
| `GET` | Read, never mutate | Yes |
| `POST` | Create, or non-idempotent action | No |
| `PATCH` | Partial update | Yes |
| `PUT` | Full replace | Yes |
| `DELETE` | Remove | Yes |

## Status codes

```python
# Creates
return Response(serializer.data, status=status.HTTP_201_CREATED)

# Reads / updates
return Response(serializer.data, status=status.HTTP_200_OK)

# Deletes
return Response(status=status.HTTP_204_NO_CONTENT)

# Validation failure (DRF handles automatically with raise_exception=True)
return Response(errors, status=status.HTTP_400_BAD_REQUEST)

# Auth required
# DRF handles via IsAuthenticated permission class → 401

# Forbidden (authenticated but not allowed)
return Response({'detail': 'You have reached the active Moments cap.'}, status=status.HTTP_403_FORBIDDEN)

# Not found
return Response({'detail': 'Not found.'}, status=status.HTTP_404_NOT_FOUND)
```

## Request validation

```python
class MomentContinueSerializer(serializers.Serializer):
    new_input = serializers.CharField(allow_blank=True, default='')
    audio = serializers.FileField(required=False)
    environment = serializers.CharField(required=False, allow_blank=True, default='')

    def validate(self, attrs):
        # Cross-field validation
        if not attrs.get('new_input') and not attrs.get('audio'):
            raise serializers.ValidationError('Provide either text input or audio.')
        return attrs
```

Rules:
- Field-level: `validate_<field>` method
- Cross-field: `validate` method
- Always `raise_exception=True` on `is_valid()` — let DRF return 400 automatically
- Never validate in views — that's the serializer's job

## Structured error responses

DRF default error format:
```json
{ "field_name": ["This field is required."] }
{ "detail": "Authentication credentials were not provided." }
{ "non_field_errors": ["Provide either text or audio."] }
```

For business logic errors (not validation):
```json
{ "detail": "You have reached the 5 active Moments cap." }
```

Consistency rule: always `detail` for non-field errors. Never `error`, `message`, `msg`, or `status`.

## Response shape

```python
# List
[{ "id": 1, "title": "...", "created_at": "..." }, ...]

# Single resource
{ "id": 1, "title": "...", "is_archived": false, "created_at": "..." }

# Create (return the created resource, not just an ID)
{ "id": 5, "title": "New moment", "created_at": "..." }

# Action response (toggle archive)
{ "is_archived": true }

# AI response
{ "record_id": 42, "options": [...], "delivery": "..." }
```

## Common mistakes

- Verbs in URLs: `/api/getProfile/` → should be `GET /api/profiles/me/`
- Wrong status codes: returning 200 for a create, 200 for a delete
- Validation in views instead of serializers — duplicated and inconsistent
- Inconsistent error keys: sometimes `error`, sometimes `detail`, sometimes `message`
- Exposing internal errors to the client: never return raw Python exceptions or stack traces
- Missing auth scoping: forgetting to filter by `request.user` so users see each other's data
