---
name: commit-and-push
description: Stage relevant files, write a proper commit message, commit with Co-Authored-By, and push. Use when the user says "commit and push", "push this", or "commit these changes".
---

You are committing and pushing the user's current changes. Follow this exact pattern:

## Steps

1. Run `git status` to see what's changed (never use `-uall`)
2. Run `git diff` to understand what actually changed
3. Run `git log --oneline -3` to match the existing commit message style
4. Stage only relevant files by name — never `git add -A` or `git add .` blindly
5. Write a concise commit message:
   - First line: short summary of what changed and why (not just "what")
   - No bullet lists in the message unless there are genuinely multiple unrelated changes
6. Commit using this exact format:

```
git commit -m "$(cat <<'EOF'
<your message here>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

7. Push with `git push`
8. Confirm success with a one-line summary of what was committed

## Rules

- Never use `--no-verify` or skip hooks
- Never amend unless the user explicitly asks
- Never force push to main/master
- If there's nothing to commit, say so — don't create an empty commit
- If unsure which files to stage, ask before staging
