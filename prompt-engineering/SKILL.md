---
name: prompt-engineering
description: Write or audit AI prompts — multi-layer architecture, tone calibration, structured output, quality testing. Use when the user says "write a prompt", "improve this prompt", "audit the prompts", or "the AI output isn't good enough".
---

You are writing or auditing AI prompts for an application.

## Multi-layer prompt architecture

Never put everything in one prompt. Split into independent layers:

```
Layer 1 — System personality (who the AI is, how it behaves)
Layer 2 — Situational context (what's happening right now)
Layer 3 — Request (what the user specifically needs)
```

This lets you improve each layer independently and reuse the personality layer across modes.

## Reference: multi-mode prompt file structure

```
prompts/v2/
  system.txt           ← Layer 1: loaded as system prompt, applies to all modes
  mode_a.txt           ← Layer 2+3: context and request for mode A
  mode_b.txt           ← Layer 2+3: context and request for mode B
  mode_c.txt           ← Layer 2+3: ongoing/conversational mode
```

One system prompt shared across all modes. Mode-specific prompts handle situational framing and output format.

## VAR substitution pattern

Use `{var_name}` placeholders replaced with `str.replace()` — not `.format()`:

```python
template = open('prompts/v2/system.txt').read()
for key, value in profile.items():
    template = template.replace('{' + key + '}', str(value))
```

**Why not `.format()`**: prompt templates contain JSON examples with `{}` which `.format()` treats as format tokens → `KeyError`. `str.replace()` is explicit and safe.

## What makes a good system prompt

```
You are [role description — told through analogy, not job title].

You are NOT here to [anti-identity — what the AI explicitly should not do].
You ARE here to [positive identity — what success looks like].
```

Key elements:
- **Identity** — who is the AI, told through analogy not job title
- **Anti-identity** — what the AI explicitly is NOT (equally important)
- **Calibration inputs** — user profile vars that shape every response
- **Quality test** — an internal check the AI runs before outputting
- **Output schema** — exact JSON structure, field by field
- **Rules** — hard constraints listed explicitly

## Quality test (embed in every prompt)

```
Before writing each response, ask internally:
"Would a real person actually say this naturally in this exact moment,
without it sounding scripted or AI-generated?"
If no, rewrite it.
```

This single instruction eliminates most AI-sounding output.

## Tone calibration

Profile vars in the system prompt let the same prompt serve very different users:

```
- Communication style: {communication_style}   → formal, casual, dry, warm
- Confidence level: {confidence_level}         → low, moderate, high
- Expertise level: {expertise_level}           → beginner, intermediate, expert
- Context preference: {context_preference}     → brief, detailed
```

The values come from the user's profile — the same prompt serves every user without branching.

## Structured JSON output

Always specify the exact schema in the prompt:

```
Return ONLY a JSON object using exactly this structure:
{
  "options": [
    { "type": "safe", "text": "...", "note": "..." },
    { "type": "moderate", "text": "...", "note": "..." },
    { "type": "direct", "text": "...", "note": "..." }
  ],
  "guidance": "..."
}
No markdown fences, no preamble, no commentary after.
```

- `text` — the usable response, no placeholders
- `note` — one sentence of context or framing (suggestion form, not command)
- `guidance` — 1-2 sentences of situational coaching

## Anti-patterns (enforce in prompt rules)

```
- No overly clever or workshopped phrasing
- No AI-sounding phrases: "certainly", "of course", "great question", "absolutely"
- No scripted or screenwriter energy
- No forced humor
- No emotional over-analysis
- Nothing manipulative, passive-aggressive, or inappropriate
```

## Length constraints

For real-time or conversational modes, enforce brevity explicitly in the prompt:

```
Every "text" must be 1-2 short sentences — something a person could say naturally in under 3 seconds.
Note: one sentence only.
Guidance: 1-2 sentences max.
```

Enforce with `max_tokens` in the API call as a hard ceiling.

## Prompt versioning

- Keep prompts in `prompts/v{n}/` directories
- Log `prompt_version` on every response record
- This lets you compare output quality across versions using real feedback data
- Never edit a deployed prompt file — create a new version

## Iterating on prompts

1. Look at feedback data — which responses get negative reactions?
2. Read the actual response content for those records
3. Identify the pattern: is it a specific option type? A specific context?
4. Write a targeted rule in the prompt that addresses the pattern
5. Deploy as `v3`, run both versions, compare feedback distribution
6. Kill the worse one

## Common mistakes

- One giant system prompt — hard to iterate, hard to understand
- Inline prompt strings in code — can't version, diff, or reuse
- No quality test — AI optimizes for impressiveness, not naturalness
- Vague tone instructions ("be casual") — concrete examples work better
- No anti-patterns list — the AI will do the thing you didn't say not to do
- Forgetting `max_tokens` — prompts that ask for short output still need a ceiling
