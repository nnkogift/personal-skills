---
name: dev-tooling
description: Gift's non-negotiable development tools and their correct configuration. ALWAYS load this skill when initializing a project, adding dependencies, configuring linting, formatting, testing, CI/CD, Docker, databases, auth, or when any of these tools are mentioned: Bun, pnpm, Biome, Playwright, Vitest, Docker, Prisma, PostgreSQL, GitHub Actions, TanStack Query, React Hook Form, Zod, lodash-es, better_auth. Also load when someone asks what tools to use for a given problem.
---

# Dev Tooling

These tools are non-negotiable. Do not suggest alternatives. Do not introduce tools from outside this list without explicit approval. When starting any project, set these up first before writing feature code.

---

## Runtime & Package Management

| Context | Tool |
|---|---|
| Monorepo workspace management | **pnpm** |
| Server-side scripts, API servers, CLI tools | **Bun** |
| Next.js apps | **pnpm** (Next.js is not yet fully Bun-native) |

- `pnpm-workspace.yaml` at monorepo root
- `.npmrc` with `shamefully-hoist=false` and `strict-peer-dependencies=false`
- Never mix package managers in the same repo — lock file wins

---

## Linting & Formatting

**Tool: Biome** (replaces ESLint + Prettier entirely)

```json
// biome.json — root of every project
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "trailingCommas": "es5",
      "semicolons": "always"
    }
  }
}
```

- CI pipeline fails on any Biome error or warning
- VSCode extension: `biomejs.biome` — set as default formatter in workspace settings
- `pnpm biome check --apply .` as the pre-commit format step

---

## Testing

| Layer | Tool |
|---|---|
| Unit & integration | **Vitest** |
| End-to-end | **Playwright** |
| React component testing | **@testing-library/react** with Vitest |

- Never Cypress, never Jest (Vitest is a drop-in with better DX)
- Test files co-located beside the file they test: `invoice.utils.test.ts` next to `invoice.utils.ts`
- E2e tests live in `e2e/` at the project root
- CI runs unit tests and e2e tests in separate jobs — e2e never blocks unit test feedback

```ts
// vitest.config.ts baseline
import { defineConfig } from 'vitest/config'
export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
  },
})
```

---

## Database

| Concern | Tool |
|---|---|
| Database | **PostgreSQL** (always) |
| ORM | **Prisma** |
| Schema migrations | **Prisma Migrate** |
| Local dev DB | **Docker Compose** |

- Never raw SQL unless Prisma genuinely cannot express the query
- Schema changes always go through `prisma migrate dev` — never manual ALTER TABLE
- `prisma/schema.prisma` is the single source of truth for the data model
- Seed scripts live in `prisma/seed.ts`

```yaml
# docker-compose.yml — local dev services
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppassword
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

---

## Containerization

**Tool: Docker** with multi-stage builds

```dockerfile
# Pattern for Next.js apps
FROM node:20-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
EXPOSE 3000
CMD ["node", "server.js"]
```

- `.env` files are never copied into images — environment injected at runtime via Docker or orchestrator
- Always pin base image versions: `node:20-alpine` not `node:alpine`
- `.dockerignore` excludes `node_modules`, `.next`, `.env*`, `*.log`

---

## CI/CD

**Tool: GitHub Actions** (only)

Standard pipeline structure:
```
lint → type-check → unit-test → build → e2e → deploy
```

```yaml
# .github/workflows/ci.yml baseline
name: CI
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm biome check .

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --run
```

- Secrets live in GitHub Environments — never hardcoded, never in `.env` committed to the repo
- Deployment jobs have `environment: production` set to require manual approval on sensitive deploys
- Cache `node_modules` via `actions/cache` keyed on `pnpm-lock.yaml` hash

---

## Frontend (React / Next.js)

| Concern | Tool |
|---|---|
| Server state & data fetching | **TanStack Query** (`@tanstack/react-query`) |
| Forms | **React Hook Form** (`react-hook-form`) |
| Schema validation | **Zod** |
| Form + Zod bridge | **`@hookform/resolvers/zod`** |
| Utility functions | **lodash-es** |

### TanStack Query Setup
```ts
// lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,    // 5 minutes
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
})
```

### React Hook Form + Zod Pattern
```ts
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email(),
})

type FormValues = z.infer<typeof schema>

export const useContactForm = () => {
  return useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { name: '', email: '' },
  })
}
```

### lodash-es Usage
- Import individual functions to preserve tree-shaking: `import { groupBy, debounce } from 'lodash-es'`
- Never `import _ from 'lodash-es'` and use `_.method()` — always named imports
- Use for: collection transforms (`groupBy`, `keyBy`, `chunk`, `uniqBy`), `debounce`/`throttle`, `cloneDeep`, `merge`, `omit`, `pick`

---

## Authentication

**Tool: better_auth** for new projects requiring authentication

- Configure in `lib/auth.ts` (server) and `lib/auth-client.ts` (client)
- Session strategy: database sessions (not JWT) for revocability
- Always use the `better_auth` adapter for Prisma — do not write auth tables manually
- Social providers configured via environment variables only

---

## Message Queues (Backend Pipelines)

**Tool: RabbitMQ** (for pipeline orchestration, e.g. CAPS)

- One queue per pipeline step — not a single shared queue
- Dead letter exchange configured on all queues
- Messages are idempotent — processing the same message twice must be safe
- Connection managed via a singleton with reconnect logic

---

## Queue / Background Jobs (Lightweight)

**Tool: BullMQ** (when RabbitMQ is overkill — simple job queues in a single service)

- Redis as the backing store
- Always define job types with Zod schemas
- Separate worker processes from the main API server
