---
name: django-drf
description: Build or audit Django + DRF components — models, serializers, views, URL routing, custom permissions. Use when the user says "add a model", "create an endpoint", "write a serializer", or "add a permission".
---

You are building or auditing Django + Django REST Framework components.

## Layer responsibilities

- **Model** — data shape and DB constraints only. No business logic.
- **Serializer** — validate input, transform output. No DB queries beyond what's needed.
- **View** — receive request, call service, return response. Thin.
- **URL** — maps paths to views. Nothing else.
- **Permission** — answers "can this user do this?" Nothing else.

## Model patterns

```python
class Moment(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='moments')
    title = models.CharField(max_length=200)
    is_archived = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    last_active_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-last_active_at']
```

Rules:
- Always use `settings.AUTH_USER_MODEL`, never `User` directly
- `auto_now_add` for created, `auto_now` for updated
- `related_name` on every FK so reverse lookups are readable
- `on_delete` always explicit — never leave it unset
- `__str__` on every model

## Serializer patterns

```python
class MomentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Moment
        fields = ['id', 'title', 'relationship_context', 'is_archived', 'created_at']
        read_only_fields = ['id', 'created_at']

class MomentCreateSerializer(serializers.ModelSerializer):
    # Separate serializer for write — different fields, different validation
    class Meta:
        model = Moment
        fields = ['title', 'relationship_context']
```

Rules:
- Separate serializers for read vs write when fields differ
- Always declare `read_only_fields` explicitly
- Use `validate_<field>` for field-level validation, `validate` for cross-field
- Never put DB queries inside serializer `to_representation` — that's an N+1 waiting to happen

## View patterns

```python
class MomentListCreateView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        moments = Moment.objects.filter(user=request.user, is_archived=False)
        serializer = MomentSerializer(moments, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = MomentCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        moment = serializer.save(user=request.user)
        return Response(MomentSerializer(moment).data, status=status.HTTP_201_CREATED)
```

Rules:
- Always filter by `request.user` — never return another user's data
- `raise_exception=True` on `is_valid()` — let DRF handle 400 responses
- Pass `user=request.user` in `save()` not in serializer — view owns auth context
- Return 201 for creates, 200 for reads/updates, 204 for deletes

## URL patterns

```python
urlpatterns = [
    path('moments/', MomentListCreateView.as_view()),
    path('moments/<int:pk>/', MomentDetailView.as_view()),
    path('moments/<int:pk>/continue/', MomentContinueView.as_view()),
]
```

Rules:
- Nest related resources: `moments/<pk>/continue/` not `moments/continue/<pk>/`
- Use `<int:pk>` not `<pk>` — enforce type at the URL level
- All app URLs included in `config/urls.py` with a prefix: `path('api/', include('apps.moments.urls'))`

## Custom permission patterns

```python
class IsOnboardingComplete(BasePermission):
    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated
                    and hasattr(request.user, 'profile')
                    and request.user.profile.is_onboarding_complete)
```

Rules:
- One permission class = one question ("is this user authenticated?", "has this user completed onboarding?")
- Return `bool` — don't raise exceptions inside permission classes
- Compose permissions: `permission_classes = [IsAuthenticated, IsOnboardingComplete]`

## Django app namespace

Apps live under `apps/` and are registered as `apps.users`, `apps.moments` etc. in `INSTALLED_APPS`. `AppConfig.name` must match the dotted path exactly or migrations break.

## Common mistakes

- Business logic in views — move to `services/`
- N+1 queries — use `select_related` / `prefetch_related`
- Missing `user` filter — always scope queries to `request.user`
- Serializer used for both read and write when fields differ — use two serializers
- Hardcoded strings instead of `TextChoices` — define choices on the model
