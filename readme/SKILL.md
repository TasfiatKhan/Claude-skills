---
name: readme
description: Write or update the README.md for the current project based on its actual state. Use when the user says "update the readme", "write a readme", or "update docs".
---

You are writing or updating the README.md for the current project.

## Steps

1. Check if a README.md already exists — if so, read it first
2. Read `CLAUDE.md` if it exists — it's the best source of truth for the project
3. Explore the project structure with `ls` and spot-read key files to understand:
   - What the project actually does
   - The tech stack
   - How to run it locally
   - Any important architecture decisions
4. Write or update the README

## README structure (adapt to what's relevant for this project)

```
# Project Name

One-sentence description of what it does and who it's for.

## What it does
Brief description of core features/modes — not marketing copy, just clear explanation.

## Stack
Table of technologies used.

## Quick start
Exact commands to get it running locally. Test these mentally — don't write commands that won't work.

## Project structure
Directory tree with one-line explanations of key folders/files.
Only include what's non-obvious — skip boilerplate folders everyone knows.

## Architecture / key rules (if relevant)
Any non-obvious decisions a new developer needs to know.

## Related
Links to related repos if applicable.
```

## Rules

- Write for a developer reading this cold — no assumed context
- Every command in Quick Start must actually work
- Don't pad with marketing language — clear and direct only
- If the project has a CLAUDE.md, the README should complement it, not duplicate it
- Match the existing tone if updating (don't rewrite a casual README in formal style)
- After writing, ask the user if they want it committed
