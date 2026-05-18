---
name: update-claude-md
description: Update the CLAUDE.md progress log to reflect what was built or changed in this session. Use when the user says "update CLAUDE.md", "update the docs", or "log what we did".
---

You are updating the CLAUDE.md file in the current project directory.

## Steps

1. Read the existing `CLAUDE.md` fully to understand its structure and tone
2. Look at what changed this session — use `git log --oneline -10` and `git diff HEAD~5..HEAD` to reconstruct what was done if needed
3. Update the following sections as needed:

### Progress Log
Add a new dated entry (use today's date) at the bottom of the progress log. Format:
```
- **YYYY-MM-DD** — <what was built/changed, why it matters, key technical details>
```

Each entry should:
- Be specific — mention file names, component names, decisions made
- Include the "why" not just the "what"
- Be a single bullet (multiple sentences are fine, use em-dashes to chain them)
- Match the tone and detail level of existing entries

### Other sections to update if relevant:
- **Current Status** — update the date and status summary
- **Key Files** — add any new files that are architecturally significant
- **Page Sections / Feature Scope** — update if new sections or features were added
- **Upcoming** — remove anything that's now complete, add anything newly planned

## Rules

- Never delete existing log entries — the log is append-only
- Match the existing writing style exactly — don't change tone or formatting
- Only update sections that actually changed
- If CLAUDE.md doesn't exist, ask the user before creating one
- After updating, commit the file using the commit-and-push pattern
