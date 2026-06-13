# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A personal collection of Claude Code skills — reusable instruction sets loaded via the `coding-standards` and `dev-tooling` skills. Skills live as `SKILL.md` files inside named directories. The `forming/` directory holds prompt drafts for skills in progress.

## Skill File Format

Every skill directory contains a `SKILL.md` with YAML frontmatter:

```markdown
---
name: skill-name
description: When and why to load this skill (used by the skill loader to decide relevance)
---

# Skill content...
```

The `description` field is the most important part — it drives when the skill is auto-loaded by Claude Code.

## Key Skills

- **`coding-standards/`** — TypeScript conventions, project structure, component rules, data fetching patterns, form patterns, API design, and git conventions. Load when writing or reviewing any code.
- **`dev-tooling/`** — Non-negotiable tool choices (Bun, ESLint v9 flat config, Prettier, Vitest, Playwright, Prisma, PostgreSQL, TanStack Query, React Hook Form, Zod, better_auth, Fallow). Load when setting up projects or choosing libraries.
- **`forming/`** — React form architecture using `react-hook-form` and `zod`. Covers `FormProvider`, `Controller`, `useFieldArray`, validation modes, server errors, and dynamic defaults. Load when building or reviewing any form.
- **`coding-standards/references/`** — Framework-specific deep dives: `nextjs.md`, `flutter.md`, `dhis2.md`, `backend.md`, `react.md`, `database.md`.

## Writing or Editing Skills

- Keep the `description` frontmatter precise — it determines when the skill fires
- Skills are instruction sets, not documentation: write in imperative voice, use concrete patterns and examples
- `forming/prompts/` holds WIP prompt drafts; `coding-standards/prompts/` holds auxiliary prompts for the standards skill

## Conventions That Apply Here

Since skills reference the `coding-standards` skill, those conventions apply to any code written in this repo too (TypeScript strict mode, conventional commits, `feat:`/`fix:`/`chore:` prefixes, etc.).
