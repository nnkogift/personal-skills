# personal-skills

A personal collection of [Claude Code](https://claude.ai/code) skills — reusable instruction sets that load via Claude's
skill system to enforce consistent coding practices and tooling choices across projects.

## Structure

```
personal-skills/
├── coding-standards/     # TypeScript conventions, project structure, component rules,
│   │                     # data fetching patterns, API design, and git conventions
│   ├── SKILL.md
│   ├── prompts/
│   │   └── cleanup.md
│   └── references/       # Framework-specific deep dives
│       ├── nextjs.md
│       ├── react.md
│       ├── backend.md
│       ├── database.md
│       ├── flutter.md
│       └── dhis2.md
│
├── dev-tooling/          # Non-negotiable tool choices and their correct configuration
│   ├── SKILL.md
│   └── references/
│
├── forming/              # React form architecture (react-hook-form + zod)
│   ├── SKILL.md
│   └── prompts/
│       └── init.md
```

## Skills

| Skill              | When it loads                                          |
|--------------------|--------------------------------------------------------|
| `coding-standards` | Any time code is written, reviewed, or organized       |
| `dev-tooling`      | Project init, adding dependencies, configuring tooling |
| `forming`          | Building or reviewing any React form                   |

## Skill Format

Each skill is a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: skill-name
description: When and why to load this skill
---

# Skill content...
```

The `description` field drives when Claude auto-loads the skill — keep it precise.

## Tools

Key tools enforced by `dev-tooling`: Bun, ESLint v9 flat config, Prettier, Vitest, Playwright, Prisma, PostgreSQL,
TanStack Query, React Hook Form, Zod, better_auth.
