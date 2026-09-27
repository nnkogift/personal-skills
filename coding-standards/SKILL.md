---
name: coding-standards
description: Gift's personal coding standards, file structure conventions, and project setup rules. ALWAYS load this skill when starting a new project, scaffolding files, setting up a repo, writing new components, creating hooks, naming functions, structuring folders, composing UI, or any time code organization or architecture is involved. Also load when reviewing code for quality or consistency.
---

# Coding Standards

This skill governs how all code is written, organized, and reviewed across all projects. It defines high-level non-negotiable principles and acts as a router to specialized reference guides.

---

## Reference Router

Always consult the specific reference guide for detailed patterns, conventions, and examples:

| Domain / Context | Reference File | Key Topics Covered |
| :--- | :--- | :--- |
| **TypeScript** | `references/typescript.md` | Strict typing, `type` vs `interface`, Zod schemas, `z.infer`, enums vs `const as const`, type locality |
| **Project Structure & Monorepos** | `references/project-structure.md` | Bun workspaces, apps vs packages, dependency boundaries, single-app directory layouts |
| **React & Frontend** | `references/react.md` | Component composition, file naming, props, feature modules, custom hooks, utilities, forms, TanStack Query |
| **Next.js (App Router)** | `references/nextjs.md` | Route groups, `_shared` private folders, Server vs Client components, caching (`'use cache'`), Server Actions |
| **Backend & APIs** | `references/backend.md` | Bun runtime, **Elysia as the default framework**, feature modules, plugins/macros, REST design, unified `ApiError` shape, Zod request validation, OpenAPI + Eden, error plugin, testing, deployment, **pg-boss background jobs, queues, cron scheduling, flows & pub/sub** |
| **Database & ORM** | `references/database.md` | PostgreSQL, Prisma schema design, relations, migrations, indexing, query performance |
| **Flutter & Mobile** | `references/flutter.md` | Feature-first architecture, Riverpod state management, offline-first sync, widgets, GoRouter, forms |
| **DHIS2 Web Apps & Integration** | `references/dhis2.md` | App platform conventions, versioned API paths, data store schemas, `@dhis2/ui`, climate integration |
| **Form Architecture** | `../forming/SKILL.md` | React Hook Form + Zod architecture, `FormProvider`, field arrays, server validation errors, dynamic defaults |

---

## General Instructions

These core standards apply across every language, framework, and project in this repository.

### Code Quality & Clean Code

- **Single responsibility**: Functions and components must do one thing well. If describing what it does requires "and", split it into smaller units.
- **Composition over monoliths**: Decompose large files and functions into smaller, focused, and cohesive units.
- **No magic numbers or strings**: Extract all literals and constants to clearly named identifiers.
- **No commented-out code**: Remove unused and commented-out code immediately; rely on Git history.
- **Early returns & guard clauses**: Use guard clauses and early returns to eliminate deeply nested conditionals.
- **No `console.log` in production**: Use a dedicated logger or remove logging statements before committing code.
- **Pure functions**: Keep utility functions pure and deterministic unless side effects are explicitly required and documented.

### Modularity & Co-location

- **Proximity principle**: Place files, utilities, types, and subcomponents as close as possible to where they are consumed.
- **On-demand creation**: Create folders and sub-modules on demand; do not pre-scaffold empty directory trees.
- **Promote deliberately**: Keep code local to a component or route first; only promote to shared directories when actively consumed across multiple modules.
- **Strict boundary isolation**: Respect architectural boundaries (e.g. no cross-app imports in monorepos, no importing private route modules across routes).

### Type Safety & Validation

- **Strict type checking**: Enable strict compiler options across all projects. Avoid `any` — use `unknown` and narrow types safely.
- **Validate at boundaries**: Treat all external inputs (HTTP requests, user forms, environment variables, data stores) as untrusted. Validate with runtime schemas (such as Zod) before data reaches core application logic.
- **Derive types from schemas**: Use schema definitions as the single source of truth for runtime validation and static types.

### Error Handling

- **No swallowed errors**: Explicitly handle errors in all async and fallible operations. Never write empty `catch` blocks.
- **Contextual errors**: Log failures with sufficient diagnostic context before handling, recovering, or rethrowing.
- **Consistent error responses**: Standardize error payload structures across APIs and user-facing surfaces.

### Git Conventions

- **Conventional commits**: Use standard commit prefixes: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`.
- **Atomic commits**: Keep each commit scoped to a single logical change.
- **Context-driven PRs**: Focus pull request descriptions on the *why* behind the change rather than repeating the diff.
- **Branch naming**: Use standard branch prefixes: `feature/`, `fix/`, `chore/`, `refactor/`.
- **Branch lifecycle**: Delete merged branches immediately.
