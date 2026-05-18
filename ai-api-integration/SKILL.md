---
name: ai-api-integration
description: Integrate an AI API (Claude, OpenAI, etc.) — structured JSON output, prompt assembly, error handling, service architecture. Use when the user says "add an AI call", "integrate Claude", "wire up the AI", or "audit the AI integration".
---

You are integrating an AI API into a backend application.

## Core rules

- **Never call the API from views** — all AI calls go through a dedicated service (`ai_service.py`)
- **Always return structured JSON** — not plain text. Define the schema upfront.
- **Version your prompts** — keep templates in files (`prompts/v2/`), not inline strings
- **One singleton service** — instantiate the client once at module level, reuse it

## Reference implementation

### Service structure

```python
# backend/services/ai_service.py
import anthropic   # or openai, or any other provider SDK
import json
from decouple import config

client = anthropic.Anthropic(api_key=config('AI_API_KEY'))

class AIService:
    def get_response(self, profile: dict, context: str, user_request: str) -> dict:
        system_prompt = self._build_system_prompt(profile)
        user_prompt = self._build_user_prompt(context, user_request)

        message = client.messages.create(
            model='claude-sonnet-4-6',   # pin the model version
            max_tokens=512,
            system=system_prompt,
            messages=[{'role': 'user', 'content': user_prompt}],
        )

        return json.loads(message.content[0].text)

    def _build_system_prompt(self, profile: dict) -> str:
        template = open('prompts/v2/system.txt').read()
        # Use str.replace(), not .format() — JSON braces in templates break .format()
        for key, value in profile.items():
            template = template.replace('{' + key + '}', str(value))
        return template

# Module-level singleton
ai_service = AIService()
```

### Why `str.replace()` not `.format()`

Prompt templates contain JSON examples with `{` and `}` which `.format()` interprets as format tokens and raises `KeyError`. Use `str.replace('{var_name}', value)` for each variable.

### Calling from a view

```python
# apps/<feature>/views.py
from services.ai_service import ai_service

class FeatureView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        serializer = FeatureRequestSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        profile = get_cached_profile(request.user.id)  # Redis-first
        data = serializer.validated_data

        response = ai_service.get_response(
            profile=profile,
            context=data['context'],
            user_request=data['user_request'],
        )

        # Log the response for analytics
        record = ResponseRecord.objects.create(
            user=request.user,
            mode='feature_name',
            response_json=response,
            prompt_version='v2',
        )

        return Response({**response, 'record_id': record.id})
```

## Structured JSON output

Always define the exact schema in the prompt. Claude and other models follow it reliably when told explicitly:

```
Return ONLY a JSON object with this exact structure:
{
  "options": [
    { "type": "option_a", "text": "...", "note": "..." },
    { "type": "option_b", "text": "...", "note": "..." },
    { "type": "option_c", "text": "...", "note": "..." }
  ],
  "summary": "..."
}
No markdown, no preamble, no commentary after.
```

Parse with `json.loads(message.content[0].text)`. Wrap in try/except and return a 500 if parsing fails — don't surface raw AI output to users.

## Multi-turn conversations

```python
def get_response_with_history(self, profile: dict, history: list,
                               new_input: str) -> dict:
    system_prompt = self._build_system_prompt(profile)
    user_prompt = self._build_user_prompt(new_input)

    # history is a list of {'role': 'user'/'assistant', 'content': '...'} dicts
    # Cap history to stay within context limits
    messages = history[-36:] + [{'role': 'user', 'content': user_prompt}]

    message = client.messages.create(
        model='claude-sonnet-4-6',
        max_tokens=600,
        system=system_prompt,
        messages=messages,
    )
    return json.loads(message.content[0].text)
```

## Model and token choices

| Response type | max_tokens | Reason |
|--------------|-----------|--------|
| Detailed, multi-option response | 512–1024 | Quality output |
| Short, real-time response | 300–600 | Speed matters |
| Conversational continuation | 400–600 | Short but contextual |

Always set `max_tokens` — without it, responses can be slow and expensive. Match it to the expected output length, not the maximum possible.

## Error handling

```python
try:
    response = ai_service.get_response(...)
except anthropic.APIError as e:
    return Response({'detail': 'AI service unavailable.'}, status=503)
except json.JSONDecodeError:
    return Response({'detail': 'Invalid response from AI.'}, status=500)
```

Never expose the raw exception or raw AI output in error responses.

## Common mistakes

- API calls in views — violates layered architecture, can't be tested or swapped
- Inline prompt strings — impossible to version, iterate, or diff
- `.format()` on prompts with JSON examples — breaks with KeyError
- No `max_tokens` set — runaway responses, slow and expensive
- Parsing AI output without try/except — a malformed response crashes the view
- New client instance per request — slow and wastes connections
- Hardcoding the model name in multiple places — define it as a constant or config var
