---
name: analytics-design
description: Design or audit an analytics and feedback system — what to log, how to model it, what surfaces to build. Use when the user says "add analytics", "track this", "what should we log", or "design a feedback system".
---

You are designing or auditing an analytics and feedback system.

## Core principle

Log events that answer real product questions — not everything, not nothing. Every model or event you add should map to a question you actually want answered.

## Questions first, schema second

Before writing any code, ask:
- What do we want to know about how users behave?
- What does a "successful" interaction look like?
- What would tell us something is broken or unpopular?
- Who needs to see this data — developer only, or user-facing too?

## Witly reference implementation

### Models

**AIResponseRecord** — every AI response generated
```python
class AIResponseRecord(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    mode = models.CharField(max_length=20)                # texting / live / moments
    relationship_context = models.CharField(max_length=50, blank=True)
    situation_summary = models.TextField(blank=True)
    response_json = models.JSONField()                    # full structured response
    prompt_version = models.CharField(max_length=10)      # 'v2', 'v3' — compare quality
    feedback_counts = models.JSONField(default=dict)      # running tally
    created_at = models.DateTimeField(auto_now_add=True)
```

**ResponseFeedback** — per-user reaction to a response
```python
class ResponseFeedback(models.Model):
    FEEDBACK_TYPES = ['natural', 'loved', 'cringe', 'risky']

    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    response_record = models.ForeignKey(AIResponseRecord, on_delete=models.CASCADE)
    feedback_type = models.CharField(max_length=20)

    class Meta:
        unique_together = ('user', 'response_record')  # one feedback per user per response
```

Undoable: same type → delete (undo). Different type → delete old, create new (replace).

**SavedResponse** — user explicitly saved an option
```python
class SavedResponse(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    response_record = models.ForeignKey(AIResponseRecord, on_delete=models.CASCADE)
    option_type = models.CharField(max_length=20)   # safe / playful / bold
    option_text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

**CopiedResponse** — user copied an option to clipboard
```python
class CopiedResponse(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    response_record = models.ForeignKey(AIResponseRecord, on_delete=models.CASCADE)
    option_type = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'response_record', 'option_type')  # deduplicated
```

Use `get_or_create` when logging — deduped, silent if already tracked.

### What this enables

| Question | How to answer |
|----------|--------------|
| Which option type gets used most? | Count CopiedResponse + SavedResponse by option_type |
| Which mode has the most cringe responses? | ResponseFeedback filtered by mode via response_record |
| Does prompt v3 outperform v2? | Compare feedback distribution grouped by prompt_version |
| What relationship contexts are most common? | Count AIResponseRecord by relationship_context |
| Which users are most engaged? | Count AIResponseRecord + CopiedResponse by user |
| Are Moments threads being completed? | Count MomentMessage per Moment, look at archive rate |

## General patterns

### Always worth logging
- Every AI response — with enough context to reproduce it (mode, relationship, situation)
- Explicit user feedback (labels, ratings) — high-signal, user chose to give it
- Save / bookmark — high-intent signal (they want to reference it later)
- Copy to clipboard — intent to use signal (they're about to send it)
- Error states — what failed, with what input context

### Rarely worth logging at first
- Every screen open or page view — noise before you have questions for it
- Passive time-on-screen — misleading without behavioural context
- Hover events, scroll depth — premature for a v1 product

## Schema principles

- Always store `user` FK — you'll want per-user analysis later
- Always store `created_at` — timestamps are free and always useful
- Store enough raw context to replay or understand the event (don't just store counts)
- `unique_together` to prevent duplicate events where it matters
- `JSONField` for flexible blobs (response content, metadata) — don't over-normalize early
- Log `prompt_version` on every AI response — essential for comparing prompt iterations

## Logging pattern in views

```python
# After every successful AI response
record = AIResponseRecord.objects.create(
    user=request.user,
    mode='texting',
    relationship_context=data.get('relationship_context', ''),
    situation_summary=data.get('user_request', '')[:500],
    response_json=response,
    prompt_version='v2',
)

return Response({**response, 'record_id': record.id})
```

Always return `record_id` to the frontend — it's needed to attach feedback, saves, and copies to the right record.

## Frontend tracking pattern

```typescript
// Fire-and-forget — never block the user on an analytics call
const trackCopy = async (recordId: number, optionType: string) => {
  try {
    await responsesService.trackCopy(recordId, optionType)
  } catch {
    // silent — analytics failure should never affect UX
  }
}
```

## Building analytics surfaces

1. **Start with raw DB queries** — before building a dashboard, verify the data is actually being captured correctly by querying the DB directly
2. **Admin view first** — Django admin with `list_display`, `list_filter`, `search_fields` gives you a usable interface immediately
3. **Aggregate on read, not write** — don't maintain running totals in the model (except `feedback_counts` for fast display). Query and aggregate when you need the number.
4. **User-facing stats last** — only build a "your stats" screen once you've validated users actually want it

## Common mistakes

- Logging everything from day one — creates noise and maintenance burden before you know what questions matter
- Not returning `record_id` to the frontend — feedback and copy tracking can't be attached to the right record
- Forgetting `unique_together` on feedback/copy — duplicate rows corrupt your counts
- Storing only aggregate counts, not raw events — you can always aggregate later, you can't un-aggregate
- Analytics failures affecting UX — always fire-and-forget, always silent on error
- No `prompt_version` field — impossible to know if a prompt change improved quality
