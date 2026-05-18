---
name: skill-manager
description: Audit, update, retire, and create skills in ~/.claude/skills/. Use when the user says "update a skill", "retire a skill", "we learned something new", "add this to the skill", or "run a skill audit".
---

You are managing the Claude Code skills library at `~/.claude/skills/`.

The goal is to keep skills sharp, accurate, and grounded in real project experience. A skill that hasn't been tested against a real project is weaker than one that has. A skill that contradicts what you just learned is actively harmful.

---

## Trigger: end-of-project skill audit

When a project wraps up or reaches a major milestone, run through this with the user:

**Ask these questions one domain at a time — don't dump the whole list at once:**

1. "Which skills did we actually use on this project?"
2. For each used skill: "Did anything we built contradict or go beyond what's in this skill?"
3. "Did we hit any bugs, gotchas, or patterns that aren't covered anywhere in the skills library?"
4. "Did we retire or replace any technology this skill is about?"
5. "Are there any new skills worth creating based on what we just built?"

Only update skills where the answer to question 2, 3, or 4 is yes. Don't update for the sake of it.

---

## Updating an existing skill

When the user says a skill needs updating:

1. Read the current skill file in full:
   ```
   ~/.claude/skills/<name>/SKILL.md
   ```

2. Identify exactly where the new content belongs — don't append everything to the bottom. Place:
   - New gotchas in the relevant checklist or "Common mistakes" section
   - New patterns alongside similar existing patterns
   - New rules in the relevant rules section
   - Corrections by editing the wrong content directly (don't leave contradictions)

3. Write the update — one short entry per new thing learned. Format to match the existing style of that skill file.

4. Commit and push:
   ```bash
   cd ~/.claude/skills
   git add <skill-name>/SKILL.md
   git commit -m "Update <skill-name>: <one line describing what changed>"
   git push origin main
   ```

**What's worth adding:**
- A bug that wasn't in the skill and cost time to debug
- A pattern that worked better than what the skill recommended
- A gotcha specific to a library version, OS, or device type
- A rule that had to be enforced more than once on the project
- A "don't do X" that came from a real mistake

**What's not worth adding:**
- Things already implied by the existing content
- Highly project-specific details that won't generalise
- Rewrites of content that's already correct

---

## Retiring a skill

A skill should be retired when:
- The technology it covers is no longer used (switched frameworks, dropped a service)
- It's been superseded by a better, more complete skill
- It contains mostly wrong or outdated information that would mislead more than help

**To retire a skill:**
1. Confirm with the user: "The `<name>` skill covers `<technology>`. You haven't used this in the last two projects — retire it?"
2. If yes, delete the skill directory and remove it from README.md
3. Commit and push:
   ```bash
   cd ~/.claude/skills
   git rm -r <skill-name>/
   git add README.md
   git commit -m "Retire <skill-name>: <reason>"
   git push origin main
   ```

Never retire a skill just because it wasn't used on the most recent project. Only retire when it's genuinely no longer relevant to the user's stack.

---

## Creating a new skill

A new skill is worth creating when:
- The same domain came up 2+ times in a project and had to be re-explained each time
- A technology or pattern was used that has no existing skill
- A new "team member" role is needed (e.g. a new language, a new service)

**Template for a new skill:**

```markdown
---
name: <kebab-case-name>
description: <One sentence — what domain this covers. Include trigger phrases like "Use when the user says X or Y".>
---

You are [role description].

## [First major section]

[Content grounded in real patterns from the project, not generic advice]

## Common mistakes

- [Real mistake from the project]
- [Another real mistake]
```

**After writing the skill:**
1. Add it to `README.md` in the appropriate category table
2. Commit and push:
   ```bash
   cd ~/.claude/skills
   git add <skill-name>/SKILL.md README.md
   git commit -m "Add <skill-name> skill: <one line description>"
   git push origin main
   ```

---

## Updating README.md

`README.md` is the index. Keep it in sync whenever a skill is added or retired.

Each skill entry format:
```markdown
| `skill-name` | One sentence — what it does, not what technology it covers |
```

The description should answer "why would I reach for this" not "what is this about."

---

## Skill quality checklist

Before finalising any new or updated skill, verify:

- [ ] Examples use real code from an actual project, not invented pseudocode
- [ ] "Common mistakes" section exists and contains real mistakes, not textbook warnings
- [ ] The `description` frontmatter contains trigger phrases — words the user would actually say
- [ ] No section contradicts another section in the same file
- [ ] The skill is specific enough to be useful and general enough to apply beyond one project

---

## Quick reference — all skill management commands

| Action | Command |
|--------|---------|
| Read a skill | `Read ~/.claude/skills/<name>/SKILL.md` |
| List all skills | `ls ~/.claude/skills/` |
| Add new skill | Write file + update README.md + commit + push |
| Update skill | Edit file + commit + push |
| Retire skill | `git rm -r <name>/` + update README.md + commit + push |
| Push all changes | `cd ~/.claude/skills && git add . && git commit -m "..." && git push origin main` |
