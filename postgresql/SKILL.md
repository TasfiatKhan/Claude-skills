---
name: postgresql
description: Design or audit PostgreSQL schema, migrations, and relational modeling in Django. Use when the user says "design the schema", "add a migration", "model this relationship", or "audit the DB structure".
---

You are designing or auditing a PostgreSQL schema in a Django project.

## Schema design principles

- **Every table needs a purpose** — if you can't say in one sentence what it stores, split or rethink it
- **Normalize to third normal form** — no repeated groups, no partial dependencies, no transitive dependencies
- **Denormalize only with evidence** — when a join is proven slow, not as a premature optimization
- **Timestamps on everything** — `created_at` (auto_now_add) and `updated_at` (auto_now) on every model
- **Soft delete over hard delete** — `is_archived` / `is_deleted` flag preserves history

## Relationship modeling

```python
# One-to-many (most common)
class MomentMessage(models.Model):
    moment = models.ForeignKey(Moment, on_delete=models.CASCADE, related_name='messages')

# One-to-one (profile pattern)
class UserProfile(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='profile')

# Many-to-many (when needed)
class Response(models.Model):
    tags = models.ManyToManyField(Tag, blank=True)
```

**`on_delete` choices:**
- `CASCADE` — delete children when parent is deleted (messages when moment deleted)
- `SET_NULL` — null out the FK, keep the child (response_record on MomentMessage — keep message even if record deleted)
- `PROTECT` — prevent parent deletion if children exist (use when deletion would be catastrophic)
- Never leave `on_delete` unset

## Migrations

```bash
# After changing a model:
python manage.py makemigrations
python manage.py migrate

# Check what would run without running it:
python manage.py migrate --plan

# In Docker:
docker compose exec backend python manage.py makemigrations
docker compose exec backend python manage.py migrate
```

**Migration rules:**
- Never edit a migration that's already been applied — make a new one
- Name migrations meaningfully: `0003_userprofile_social_anxiety_level`
- Squash migrations before production if you have many small ones
- Data migrations (backfilling) go in a separate migration from schema changes
- Always commit migrations alongside the model change — never leave them uncommitted

## Indexing

Django adds indexes automatically for:
- Primary keys
- ForeignKey fields
- Fields with `unique=True`

Add indexes manually for:
```python
class Meta:
    indexes = [
        models.Index(fields=['user', 'is_archived']),  # common filter combo
        models.Index(fields=['-created_at']),           # sort by newest
    ]
```

Add an index when:
- You filter by this field on every request
- You sort by this field frequently
- The table will have >10k rows

## Common query patterns

```python
# Always filter by user
Moment.objects.filter(user=request.user, is_archived=False)

# Select related to avoid N+1
MomentMessage.objects.select_related('moment', 'response_record')

# Prefetch for reverse FK
Moment.objects.prefetch_related('messages')

# Existence check (faster than count)
Moment.objects.filter(user=request.user).exists()

# Count active moments
Moment.objects.filter(user=request.user, is_archived=False).count()
```

## JSONField (PostgreSQL-native)

```python
feedback_counts = models.JSONField(default=dict)
```

Use for:
- Flexible metadata that doesn't need querying
- Counts or aggregates that change frequently
- Response blobs from external APIs

Don't use for:
- Data you need to filter or sort by — use a proper column
- Relationships — use FK

## Common mistakes

- Missing `related_name` — reverse lookups become unreadable (`moment_set` instead of `messages`)
- Wrong `on_delete` — `CASCADE` when you meant `SET_NULL` loses data silently
- No index on a heavily filtered field — slow queries at scale
- Forgetting to run `makemigrations` after model change — model and DB out of sync
- Direct SQL strings instead of ORM — loses safety and portability
