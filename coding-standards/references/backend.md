# Backend & API Standards

This reference defines backend runtime conventions, the default framework, project structure, API design, environment configuration, and worker patterns. It supplements the main `coding-standards` SKILL.md.

---

## Runtime

- **Bun** for all non-Next.js backend services, microservices, and automation scripts.
- **TypeScript everywhere** — no plain JavaScript backend code.

---

## Framework: Elysia (default)

- **[Elysia](https://elysiajs.com/)** is the backend framework of choice whenever a project or task does not explicitly name one. Do not reach for Express, Hono, Fastify, or NestJS unless the project already uses them or the user asks for them.
- **Fetch the current docs before writing code**: Elysia moves fast. Check the installed `elysia` version in `package.json` and consult https://elysiajs.com/llms.txt (Markdown index of every docs page) instead of relying on memorized APIs.
- **Official packages use the `@elysia/*` scope** (e.g. `@elysia/eden`, `@elysia/openapi`, `@elysia/cors`, `@elysia/jwt`, `@elysia/bearer`, `@elysia/static`, `@elysia/opentelemetry`). Prefer an official plugin over a hand-rolled equivalent. Scheduled work is the exception: use pg-boss (see [Background Jobs, Queues & Scheduling](#background-jobs-queues--scheduling-pg-boss)), not `@elysia/cron`.

```bash
bun create elysia my-service   # new standalone service
bun add elysia                 # add to an existing package
```

### Core rules

- **Always method-chain**: Elysia's type inference flows through the chain. Every `.use()`, `.decorate()`, `.derive()`, `.get()`, etc. must be chained on the same instance — never call them as separate statements on a stored variable, or types are lost.

```ts
// ✅ Do
export const app = new Elysia()
  .use(invoices)
  .get('/health', () => 'ok')

// ❌ Don't — the route has no knowledge of the plugin's context
const app = new Elysia()
app.use(invoices)
app.get('/health', () => 'ok')
```

- **1 Elysia instance = 1 controller**: Define routes directly on an Elysia instance. Never write controller classes whose methods take Elysia's `Context` (`static root(ctx: Context)`) — the context type depends on the plugin chain and cannot be typed correctly by hand. Destructure only what you need in the handler and pass plain values on.
- **Order matters**: Lifecycle hooks (`onRequest`, `onBeforeHandle`, `onError`, `derive`, `resolve`, …) only apply to routes registered **after** them. Register plugins and hooks before the routes that depend on them.
- **Encapsulation by default**: Hooks registered in a plugin are `local` to that instance. Use `{ as: 'scoped' }` (parent + current) or `{ as: 'global' }` only when the hook is genuinely meant to leak, and prefer `scoped` over `global`.
- **Name every reusable plugin**: `new Elysia({ name: 'Auth.Service' })` enables plugin deduplication, so a plugin `.use()`d by many modules is instantiated once. Add `seed` when the same plugin is registered with different config.
- **Decorate sparingly**: Use `decorate`/`derive`/`resolve` only for request-dependent values (session, current user, request IP). Stateless services are imported directly — overusing decorators couples business logic to Elysia and makes it harder to test.
- **Response status via `status()`**: Use the `status` helper (`status(404, 'Invoice not found')` or `status('Not Found', …)`) rather than mutating `set.status`. A **thrown** `status` goes through `onError`; a **returned** `status` bypasses it — throw from services, return from handlers when intentionally short-circuiting.

---

## API Design

- **REST for CRUD resources**: Keep URLs flat, resource-focused, and noun-based (e.g. `/invoices`, `/invoices/:id/lines`). Group a resource's routes under one Elysia instance with `prefix` (`new Elysia({ prefix: '/invoices' })`).
- **Consistent error response shape**: Maintain a unified error payload structure across all endpoints:

```ts
type ApiError = {
  error: string       // human-readable message
  code: string        // machine-readable code, e.g. "VALIDATION_ERROR"
  details?: unknown   // optional field-level errors or Zod issues
}
```

- **Strict request validation**: All request bodies, query parameters, route parameters, headers, and cookies must be validated before any business logic executes. In Elysia, declare the schema in the route's hook object (`body`, `query`, `params`, `headers`, `cookie`) — Elysia validates before the handler runs and returns `422 Unprocessable Entity` on failure, which the central error handler formats as `ApiError`.
- **Zod is the schema language**: Elysia accepts Zod directly through [Standard Schema](https://github.com/standard-schema/standard-schema), so the same Zod schemas and `z.infer` types used elsewhere stay the single source of truth. Use `z.coerce.*` for `params` and `query` values, which always arrive as strings. Elysia's built-in `t` (TypeBox) is acceptable only inside an existing project that already uses it — do not mix both within one module.
- **Declare response schemas** per status code (`response: { 200: InvoiceSchema, 404: ApiErrorSchema }`) on public endpoints. This enforces the return type at compile time and feeds OpenAPI.
- **Never trust client input**: Treat all data arriving from the client as untrusted downstream of the validation boundary.
- **Standard HTTP status codes**:
  - `200 OK` / `201 Created` for successful operations.
  - `400 Bad Request` / `422 Unprocessable Entity` for client/validation errors.
  - `401 Unauthorized` / `403 Forbidden` for auth issues.
  - `404 Not Found` for missing resources.
  - `500 Internal Server Error` for unexpected server failures.

---

## Project Structure (Elysia / Bun Services)

Feature-based modules, following Elysia's recommended layout, plus the repository layer required by `database.md`:

```
src/
  modules/
    invoices/
      index.ts         ← Elysia controller: routes, validation hooks, status codes (thin — delegates to service)
      service.ts       ← Business logic, framework-agnostic (no Elysia Context)
      repository.ts    ← Database access for this module (all Prisma calls go here)
      model.ts         ← Zod schemas for request/response + inferred types
      jobs.ts          ← pg-boss queue definitions + handlers owned by this module (only when needed)
      invoices.test.ts ← Controller tests via app.handle / Eden
  lib/
    boss.ts            ← Single PgBoss instance (see Background Jobs)
  plugins/             ← Cross-cutting named Elysia plugins: auth macro, error handler, logger, rate limiting
  utils/               ← Domain-agnostic helpers
  types/               ← Shared TypeScript type declarations
  config.ts            ← Environment variable parsing and validation
  app.ts               ← Composes plugins + modules, exports `app` and `type App`
  index.ts             ← Entry point: `app.listen(config.PORT)`
  worker.ts            ← Worker entry point: starts pg-boss, registers queues, schedules, and handlers
```

- Create modules on demand; do not pre-scaffold empty module folders.
- Keep `app.ts` separate from `index.ts` so tests and Eden can import the app without starting a server.

### Module example

```ts
// src/modules/invoices/model.ts
import { z } from 'zod'

export const InvoiceParams = z.object({ id: z.coerce.number().int().positive() })
export const CreateInvoiceBody = z.object({
  customerId: z.string().uuid(),
  amount: z.number().positive(),
})
export const Invoice = CreateInvoiceBody.extend({ id: z.number() })
export const InvoicePdfJob = z.object({ invoiceId: z.number().int().positive() }) // pg-boss payload

export type CreateInvoiceBody = z.infer<typeof CreateInvoiceBody>
export type Invoice = z.infer<typeof Invoice>
export type InvoicePdfJob = z.infer<typeof InvoicePdfJob>
```

```ts
// src/modules/invoices/service.ts
import { NotFoundError } from '../../plugins/errors'
import { InvoiceRepository } from './repository'
import type { CreateInvoiceBody } from './model'

// Stateless service: abstract class + static methods (no instance allocation)
export abstract class InvoiceService {
  static async getById(id: number) {
    const invoice = await InvoiceRepository.findById(id)
    if (!invoice) throw new NotFoundError(`Invoice ${id} not found`)
    return invoice
  }

  static create(input: CreateInvoiceBody) {
    return InvoiceRepository.create(input)
  }
}
```

```ts
// src/modules/invoices/index.ts
import { Elysia } from 'elysia'
import { authPlugin } from '../../plugins/auth'
import { CreateInvoiceBody, Invoice, InvoiceParams } from './model'
import { InvoiceService } from './service'

export const invoices = new Elysia({ name: 'Module.Invoices', prefix: '/invoices' })
  .use(authPlugin)
  .get('/:id', ({ params }) => InvoiceService.getById(params.id), {
    params: InvoiceParams,
    response: { 200: Invoice },
    auth: true,
  })
  .post('/', async ({ body, status }) => status(201, await InvoiceService.create(body)), {
    body: CreateInvoiceBody,
    auth: true,
  })
```

```ts
// src/app.ts
import { Elysia } from 'elysia'
import { errorPlugin } from './plugins/errors'
import { invoices } from './modules/invoices'

export const app = new Elysia()
  .use(errorPlugin)   // registered first so it covers every route below
  .use(invoices)

export type App = typeof app
```

---

## Plugins, Macros & Auth

- **Cross-cutting concerns are named plugins** in `src/plugins/` — never copy auth checks, logging, or error formatting into individual modules.
- **Use `macro` for opt-in route behaviour** (auth, roles, rate limits). A macro with `resolve` injects typed values into the handler context only on routes that enable it:

```ts
// src/plugins/auth.ts
import { Elysia } from 'elysia'
import { auth } from '../lib/auth' // better-auth instance

export const authPlugin = new Elysia({ name: 'Plugin.Auth' })
  .mount('/auth', auth.handler)
  .macro({
    auth: {
      async resolve({ status, request: { headers } }) {
        const session = await auth.api.getSession({ headers })
        if (!session) return status(401, { error: 'Unauthorized', code: 'UNAUTHORIZED' })
        return { user: session.user, session: session.session }
      },
    },
  })
```

- **Authentication is better-auth** (see `dev-tooling`), mounted with `.mount(prefix, auth.handler)`. Set better-auth's `basePath` to avoid redundant URLs like `/auth/api/auth`.
- **CORS** via `@elysia/cors`, configured explicitly with allowed origins — never a wildcard origin together with credentials.
- **Mounting other WinterTC apps**: Use `.mount()` for any handler exposing `fetch(Request) => Response`. Do not wrap it in custom glue code.

---

## OpenAPI & End-to-End Types

- **Every service exposes OpenAPI** via `@elysia/openapi` (`.use(openapi())`, served at `/openapi`). Because schemas are Zod, register the mapper:

```ts
import { openapi } from '@elysia/openapi'
import { z } from 'zod'

openapi({ mapJsonSchema: { zod: z.toJSONSchema } })
```

- Add `detail: { summary, tags }` to route hooks so the generated docs are meaningful. Disable or protect the docs route in production if the API is not public.
- **Eden Treaty for TypeScript consumers**: Export `type App = typeof app` and consume it with `treaty<App>(baseUrl)` from `@elysia/eden` in frontends and other services. Do not hand-write fetch wrappers or duplicate DTO types for an Elysia API. Import only the **type** across package boundaries (`import type { App }`), never runtime server code — in monorepos expose it from the API package's public entry point (see `project-structure.md`).

---

## Elysia inside Next.js

When a Next.js app needs an API surface beyond Server Actions, mount Elysia on a catch-all route instead of writing many individual route handlers:

```ts
// app/api/[[...slugs]]/route.ts
import { Elysia } from 'elysia'

export const app = new Elysia({ prefix: '/api' }) // prefix must match the route folder
  .use(invoices)

export const GET = app.fetch
export const POST = app.fetch
export const PATCH = app.fetch
export const DELETE = app.fetch
```

- The `prefix` must equal the folder path of the catch-all route (e.g. `app/user/[[...slugs]]` → `prefix: '/user'`).
- Use Eden's isomorphic pattern so Server Components call the app directly (`treaty(app)`) while the client goes through the network (`treaty<App>(url)`).

---

## Environment Variables

- **Parse and validate at startup**: Use Zod to parse and validate all environment variables when the process boots.
- **No direct `process.env` access**: Never read `process.env.VARIABLE` (or `Bun.env`) directly in business logic or handlers; import the validated `config` object from `config.ts`.
- **Maintain `.env.example`**: Keep `.env.example` in sync with all required keys. Never commit `.env` files.

```ts
// config.ts
import { z } from 'zod'

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  DHIS2_BASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
})

export const config = envSchema.parse(process.env)
```

---

## Background Jobs, Queues & Scheduling (pg-boss)

- **[pg-boss](https://pgboss.io/) is the default** for every kind of background work: queues, delayed jobs, retries, cron/RRULE schedules, dependent-job flows, pub/sub fan-out, throttling, and dead-lettering. It stores jobs in the project's PostgreSQL database, so no Redis or broker is needed. Don't use BullMQ, `node-cron`, `@elysia/cron`, or `setInterval` loops. RabbitMQ stays only for existing cross-service pipelines (see `dev-tooling`).
- **Fetch the current docs before writing code**: check the installed `pg-boss` version and consult https://pgboss.io/ (v12+: `import { PgBoss } from 'pg-boss'`).
- **One instance per process** in `src/lib/boss.ts`, built from the validated `config`, with an `error` listener wired to the logger.
- **Queues are declared, not implied**: `createQueue()` must run before `send()`/`work()`. Each module declares its queues (name, policy, retry, expiry, `deadLetter`) next to its handlers in `jobs.ts`, and they are created idempotently at worker startup. Create dead-letter queues before the queues that reference them.
- **Queue names are constants**: kebab-case, prefixed by module (`invoices-generate-pdf`). Never inline string literals at call sites.
- **Zod at both ends**: each job payload has a Zod schema in the module's `model.ts`. Enqueue helpers parse before `send()`; handlers parse `job.data` before doing work.
- **Enqueue through typed module helpers**: expose helpers such as `InvoiceJobs.enqueuePdf(input)` and call them from the service. Never call `boss.send()` directly in controllers, so queue names and payload shapes stay inside the module.
- **Atomic enqueue**: when a job must exist only if a DB write commits, pass `{ db: fromPrisma(tx) }` inside `prisma.$transaction` (Prisma v7+ with `@prisma/adapter-pg`). For worker writes that must commit together with job completion, use `work(name, { transactional: true }, (jobs, tx) => …)`, but only for handlers that finish in seconds.
- **Idempotency**: every handler must be safe to run twice. Expiry, heartbeat failures, and retries can all re-deliver a job. Use `singletonKey`, a queue `policy` (`singleton`, `stately`, `exclusive`), or `sendDebounced()`/`sendThrottled()` to stop duplicate jobs from being created.
- **Failure handling**: rely on `retryLimit`/`retryDelay`/`retryBackoff` rather than catching and swallowing errors. Throw so pg-boss records the error in `output`. Every production queue has a `deadLetter` queue. Recover with `redrive()` after fixing the cause.
- **Long jobs**: set `expireInSeconds` to the worst-case duration and `heartbeatSeconds` (≥ 10) so dead workers are detected quickly. Pass `job.signal` to `fetch`/SDK calls so aborted jobs stop early.
- **Scheduling**: `boss.schedule(queue, cron, data, { tz, key, missed })` registers a schedule, and pg-boss fires it once across all replicas. Register schedules on every worker boot (it upserts). Use 5-field cron (minute resolution), or an RRULE for calendar rules like "last Friday of the month". Set `missed: 'once'` when a job missed during downtime should still run once. The scheduled queue still needs a worker.
- **Dependencies and fan-out**: use `flow()` for jobs that must wait on other jobs (e.g. extract → load), and `subscribe()` + `publish()` when one domain event feeds several queues. Don't chain `send()` calls from inside handlers to fake either.
- **Concurrency**: tune with `localConcurrency` (per process), `batchSize`, and `groupConcurrency` + `group` on `send()` for per-tenant fairness across nodes.
- **Process isolation**: the API process only enqueues (`send`/`publish`). It still calls `await boss.start()` before `app.listen()`, but registers no handlers. Workers, schedules, and `work()` handlers run from `src/worker.ts` as a separate container using the same image with a different command. Both call `await boss.stop()` on `SIGTERM`.
- **Contextual logging**: log job id, queue, and `retryCount` on failure before rethrowing.

```ts
// src/lib/boss.ts
import { PgBoss } from 'pg-boss'
import { config } from '../config'
import { logger } from '../utils/logger'

export const boss = new PgBoss(config.DATABASE_URL)

boss.on('error', (err) => logger.error({ err }, 'pg-boss error'))
```

```ts
// src/modules/invoices/jobs.ts
import type { PgBoss } from 'pg-boss'
import { boss } from '../../lib/boss'
import { InvoicePdfJob } from './model'
import { InvoiceService } from './service'

export const INVOICE_QUEUES = {
  generatePdf: 'invoices-generate-pdf',
  generatePdfDlq: 'invoices-generate-pdf-dlq',
  overdueReminders: 'invoices-overdue-reminders',
} as const

const NIGHTLY_AT_TWO = '0 2 * * *'

export abstract class InvoiceJobs {
  static enqueuePdf(input: InvoicePdfJob) {
    return boss.send(INVOICE_QUEUES.generatePdf, InvoicePdfJob.parse(input), {
      singletonKey: String(input.invoiceId),
    })
  }

  static async register(worker: PgBoss) {
    await worker.createQueue(INVOICE_QUEUES.generatePdfDlq)
    await worker.createQueue(INVOICE_QUEUES.generatePdf, {
      retryLimit: 5,
      retryBackoff: true,
      deadLetter: INVOICE_QUEUES.generatePdfDlq,
    })
    await worker.createQueue(INVOICE_QUEUES.overdueReminders, { policy: 'singleton' })

    await worker.work(INVOICE_QUEUES.generatePdf, { localConcurrency: 5 }, async ([job]) => {
      const { invoiceId } = InvoicePdfJob.parse(job.data)
      return InvoiceService.generatePdf(invoiceId, { signal: job.signal })
    })

    await worker.schedule(INVOICE_QUEUES.overdueReminders, NIGHTLY_AT_TWO, null, {
      tz: 'Africa/Dar_es_Salaam',
      missed: 'once',
    })
    await worker.work(INVOICE_QUEUES.overdueReminders, () => InvoiceService.sendOverdueReminders())
  }
}
```

```ts
// src/worker.ts
import { boss } from './lib/boss'
import { InvoiceJobs } from './modules/invoices/jobs'

await boss.start()
await InvoiceJobs.register(boss)

process.on('SIGTERM', () => boss.stop())
```

- **Operations**: pg-boss manages its own `pgboss` schema and migrates it on `start()`, so never add its tables to `schema.prisma`. If the runtime DB role can't run DDL, run `pg-boss migrate` as a deploy step and pass `migrate: false`. Use `@pg-boss/dashboard` (protected, never public) to inspect queues, jobs, schedules, and warnings.
- **Testing**: build the test instance with `__test__enableSpies: true` and await outcomes with `boss.getSpy(queue).waitForJob(predicate, 'completed')` or `waitForJobWithId(id, 'failed')`. Call `boss.clearSpies()` between tests. Use a `TestClock` (via the `clock` option) to exercise schedules and `resolveFlow()` to unblock flows deterministically. Never use `sleep()` in tests.

---

## Error Handling

- **No swallowed errors**: Never write empty `catch` blocks (`catch (e) {}`).
- **Contextual logging**: Always log errors with descriptive context before handling or rethrowing.
- **Operational vs Programmer errors**: Distinguish between expected operational errors (invalid credentials, missing entity) and unexpected bugs (null pointers, database crash).
- **Centralized error plugin**: Handle errors in one named Elysia plugin with `.error({...})` + `.onError({ as: 'global' }, ...)`, registered before all modules. Do not format raw HTTP 500 error responses inside individual route handlers.
- **Typed operational errors**: Define error classes with a `status` property and register them with `.error({ ... })` so `onError` can narrow on `code` with full type safety. Services throw these; they never import Elysia's `Context`.

```ts
// src/plugins/errors.ts
import { Elysia } from 'elysia'
import { logger } from '../utils/logger'

export class NotFoundError extends Error {
  status = 404
}

export class ConflictError extends Error {
  status = 409
}

export const errorPlugin = new Elysia({ name: 'Plugin.Errors' })
  .error({ NotFoundError, ConflictError })
  .onError({ as: 'global' }, ({ code, error, status, path }) => {
    switch (code) {
      case 'VALIDATION':
        return status(422, { error: 'Invalid request', code: 'VALIDATION_ERROR', details: error.all })
      case 'NOT_FOUND':
        return status(404, { error: 'Route not found', code: 'NOT_FOUND' })
      case 'NotFoundError':
        return status(404, { error: error.message, code: 'NOT_FOUND' })
      case 'ConflictError':
        return status(409, { error: error.message, code: 'CONFLICT' })
      default:
        logger.error({ err: error, path }, 'Unhandled error')
        return status(500, { error: 'Internal server error', code: 'INTERNAL_ERROR' })
    }
  })
```

- **Never leak internals**: Validation details are omitted in production by default — keep it that way (don't enable `allowUnsafeValidationDetails`), and never return stack traces or raw database errors to clients.

---

## Testing

- **Test through `app.handle()`** — it runs the full lifecycle (validation, macros, error plugin) without opening a port. Requests need a **full URL** (`http://localhost/invoices/1`, not `/invoices/1`).
- **Prefer Eden Treaty in tests** (`treaty(app)`) for type-checked calls and responses; a route/schema change then breaks tests at compile time.
- Test services directly as plain functions — they are framework-agnostic by design.
- Use the project's standard test runner and co-location rules from `dev-tooling`.

```ts
import { describe, expect, it } from 'bun:test'
import { treaty } from '@elysia/eden'
import { app } from '../../app'

const api = treaty(app)

describe('GET /invoices/:id', () => {
  it('returns 422 for a non-numeric id', async () => {
    const { status } = await api.invoices({ id: 'abc' }).get()
    expect(status).toBe(422)
  })
})
```

---

## Deployment

- **Compile to a single binary** for production images — smaller, faster startup, lower memory:

```bash
bun build --compile --minify-whitespace --minify-syntax --target bun --outfile server src/index.ts
```

- Use `--minify-whitespace --minify-syntax` rather than `--minify` when OpenTelemetry is enabled (full minification mangles function names used in traces).
- Packages that cannot be bundled (e.g. native drivers) are marked `--external <pkg>` and installed in the runtime image.
- Elysia is single-threaded; scale horizontally (containers/replicas) first, and only use `node:cluster` for multi-core use of a single host when needed.
- Add a `/health` route and graceful shutdown (`app.stop()` on `SIGTERM`) to every service. Worker containers call `boss.stop()` so active jobs finish before exit.
- Instrument production services with `@elysia/opentelemetry`.
