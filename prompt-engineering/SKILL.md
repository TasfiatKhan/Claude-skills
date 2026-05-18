---
name: prompt-engineering
description: Write or audit AI prompts — multi-layer architecture, tone calibration, structured output, quality testing. Use when the user says "write a prompt", "improve this prompt", "audit the prompts", or "the AI output isn't good enough".
---

You are writing or auditing AI prompts for a social intelligence application.

## Multi-layer prompt architecture

Never put everything in one prompt. Split into independent layers:

```
Layer 1 — System personality (who the AI is, how it behaves)
Layer 2 — Situational context (what's happening right now)
Layer 3 — Request (what the user specifically needs)
```

This lets you improve each layer independently and reuse the personality layer across modes.

## Witly reference: three prompt files

```
prompts/v2/
  system_personality.txt   ← Layer 1: loaded as system prompt, applies to all modes
  texting_mode.txt         ← Layer 2+3: texting-specific context and request
  live_mode.txt            ← Layer 2+3: live mode context and constraints
  moments_mode.txt         ← Layer 2+3: ongoing situation context and request
```

## VAR substitution pattern

Use `{var_name}` placeholders replaced with `str.replace()` — not `.format()`:

```python
template = open('prompts/v2/system_personality.txt').read()
for key, value in profile.items():
    template = template.replace('{' + key + '}', str(value))
```

**Why not `.format()`**: prompt templates contain JSON examples with `{}` which `.format()` treats as format tokens → `KeyError`. `str.replace()` is explicit and safe.

## What makes a good system prompt

```
You are a socially intelligent coach — like a friend who's naturally good
with people, reads any room without trying, and gives advice that works
because it sounds like a human said it.

You are not here to write impressive lines. You are here to offer options
that help people sound like a more comfortable, natural version of themselves.
```

Key elements:
- **Identity** — who is the AI, told through analogy not job title
- **Anti-identity** — what the AI explicitly is NOT (equally important)
- **Calibration inputs** — the user's profile vars that shape every response
- **Quality test** — an internal check the AI runs before outputting
- **Output schema** — exact JSON structure, field by field
- **Rules** — hard constraints listed explicitly

## Quality test (embed in every prompt)

```
Before writing each option, ask internally:
"Would a naturally smooth, socially aware person actually say this out loud
in this exact moment, without having pre-planned it?"
If no, rewrite it.
```

This single instruction eliminates most AI-sounding output.

## Tone calibration

The profile vars in `system_personality.txt` let the same prompt serve very different users:

```
- Natural style: {humor_style}         → dry, warm, sarcastic, observational
- Social style: {persona_type}         → The Charmer, The Observer, etc.
- Confidence level: {confidence_level} → shy, moderate, confident, fearless
- Social anxiety: {social_anxiety_level} → none, mild, moderate, high
```

High anxiety → safe options are genuinely low-pressure, bold is toned down
No anxiety → can push further, bold is more daring

## Structured JSON output

Always specify the exact schema in the prompt:

```
Return ONLY a JSON object using exactly this structure:
{
  "options": [
    { "type": "safe", "text": "...", "note": "..." },
    { "type": "playful", "text": "...", "note": "..." },
    { "type": "bold", "text": "...", "note": "..." }
  ],
  "delivery": "..."
}
No markdown fences, no preamble, no commentary after.
```

- `text` — the usable response, word-for-word, no placeholders
- `note` — one sentence, framed as suggestion ("if the vibe feels right...")
- `delivery` — 1-2 sentences of situational coaching

## Anti-patterns (enforce in prompt rules)

```
- No workshopped punchlines
- No screenwriter energy
- No internet meme humor
- No overly clever phrasing
- No forced humor
- No emotional over-analysis in notes
- No AI-sounding words: "certainly", "of course", "great question", "absolutely"
- Nothing manipulative, passive-aggressive, or creepy
```

## Length constraints (Live and Moments)

```
Every "text" must be 1-2 short lines — something they can say out loud in under 2 seconds.
Note: one sentence only.
Delivery: 1-2 sentences max.
```

Enforce with `max_tokens` in the API call as a hard ceiling.

## Prompt versioning

- Keep prompts in `prompts/v{n}/` directories
- Log `prompt_version` on every `AIResponseRecord`
- This lets you compare output quality across versions using real feedback data
- Never edit a deployed prompt file — create a new version

## Iterating on prompts

1. Look at `ResponseFeedback` — which responses get "cringe" or "risky" tags?
2. Read the actual `response_json` for those records
3. Identify the pattern: is it the `bold` type? A specific relationship context?
4. Write a targeted rule in the prompt that addresses the pattern
5. Deploy as `v3`, run both versions, compare feedback distribution
6. Kill the worse one

## Common mistakes

- One giant system prompt — hard to iterate, hard to understand
- Inline prompt strings in Python — can't version, diff, or reuse
- No quality test — AI optimizes for impressiveness, not naturalness
- Vague tone instructions ("be casual") — concrete examples work better
- No anti-patterns list — the AI will do the thing you didn't say not to do
- Forgetting `max_tokens` — prompts that ask for short output still need a ceiling
