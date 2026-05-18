# Claude Skills

A library of Claude Code skill files for full-stack development.

Each skill is a markdown file that Claude reads before working in that domain — grounding its output in real, battle-tested patterns rather than generic advice.

## What are skills?

Skills live in `~/.claude/skills/<name>/SKILL.md`. When a task matches a skill's description, Claude reads it before writing code. They act as persistent, project-agnostic tribal knowledge.

## How to use

Clone into `~/.claude/skills/`:

```bash
git clone https://github.com/TasfiatKhan/Claude-skills ~/.claude/skills
```

Claude Code will pick them up automatically on any project.

## Skills

### Architecture & planning
| Skill | Description |
|-------|-------------|
| `architect` | Start a new project — stack selection, folder structure, architecture rules, build order |
| `layered-architecture` | Service/transport/domain layer separation |
| `api-design` | RESTful URLs, HTTP verbs, error response shape |

### Backend
| Skill | Description |
|-------|-------------|
| `django-drf` | Models, serializers, views, URL routing, permissions |
| `postgresql` | Schema design, migrations, on_delete choices, indexes |
| `redis` | Redis-first caching pattern, TTL strategy, invalidation on write |
| `jwt-auth` | Token lifecycle, Axios interceptors, refresh flow, SecureStore |
| `docker-compose` | Multi-service setup, service networking, named volumes |
| `env-config` | .env/.env.example separation, secret scanning |

### AI & prompts
| Skill | Description |
|-------|-------------|
| `ai-api-integration` | Claude API service pattern, structured JSON responses |
| `prompt-engineering` | Multi-layer prompt architecture, VAR substitution, versioning |
| `analytics-design` | Response tracking, feedback models, copy tracking patterns |

### Frontend — Web
| Skill | Description |
|-------|-------------|
| `nextjs-app-router` | Server vs client boundary, metadata, fonts, Providers pattern |
| `tailwind-css` | Utilities, responsive, arbitrary values, @layer, keyframe gotcha |
| `framer-motion` | Entrance, scroll-triggered, continuous loops, staggered children |
| `css-custom-properties` | Theming with vars, data-theme overrides |
| `theme-context` | ThemeContext implementation, flash prevention, suppressHydrationWarning |

### Frontend — Mobile
| Skill | Description |
|-------|-------------|
| `react-native-expo` | Bare workflow, native modules, platform differences, audio recording |
| `typescript` | Typing API responses, props, navigation params, hooks |
| `navigation` | Stack navigators, auth flow gating, useFocusEffect pattern |
| `state-management` | Context API, useMemo, useCallback, per-item Record pattern |
| `secure-storage` | expo-secure-store patterns for tokens and preferences |
| `design-tokens` | theme.ts, centralised colors/typography/spacing |

### Quality & process
| Skill | Description |
|-------|-------------|
| `qa` | Full QA checklist — frontend, backend, integration, device gotchas |
| `code-review` | Security, correctness, performance, maintainability review guide |
| `commit-and-push` | Commit pattern with Co-Authored-By, staged by name |
| `update-claude-md` | Append-only progress log, matches existing tone |
| `readme` | Developer-focused README from CLAUDE.md |
