# Backend & API Standards

This reference defines backend runtime conventions, project structure, API design, environment configuration, and worker patterns. It supplements the main `coding-standards` SKILL.md.

---

## Runtime

- **Bun** for all non-Next.js backend services, microservices, and automation scripts.
- **TypeScript everywhere** — no plain JavaScript backend code.

---

## API Design

- **REST for CRUD resources**: Keep URLs flat, resource-focused, and noun-based (e.g. `/invoices`, `/invoices/:id/lines`).
- **Consistent error response shape**: Maintain a unified error payload structure across all endpoints:

```ts
type ApiError = {
  error: string       // human-readable message
  code: string        // machine-readable code, e.g. "VALIDATION_ERROR"
  details?: unknown   // optional field-level errors or Zod issues
}
```

- **Strict request validation**: All request bodies, query parameters, and route parameters must be validated with Zod before any business logic executes. Return `400 Bad Request` or `422 Unprocessable Entity` formatted as `ApiError` on failure.
- **Never trust client input**: Treat all data arriving from the client as untrusted downstream of the validation boundary.
- **Standard HTTP status codes**:
  - `200 OK` / `201 Created` for successful operations.
  - `400 Bad Request` / `422 Unprocessable Entity` for client/validation errors.
  - `401 Unauthorized` / `403 Forbidden` for auth issues.
  - `404 Not Found` for missing resources.
  - `500 Internal Server Error` for unexpected server failures.

---

## Project Structure (Node/Bun Services)

```
src/
  routes/          ← Route handlers (thin controllers — validate input and delegate to services)
  services/        ← Domain and business logic
  repositories/    ← Database access layer (all Prisma or query calls go here)
  schemas/         ← Zod validation schemas and inferred TypeScript types
  proxy/           ← Middleware for auth, error handling, logging, rate limiting
  utils/           ← Domain-specific helpers and utilities
  types/           ← Shared TypeScript type declarations
  config.ts        ← Environment variable parsing and validation
```

---

## Environment Variables

- **Parse and validate at startup**: Use Zod to parse and validate all environment variables when the process boots.
- **No direct `process.env` access**: Never read `process.env.VARIABLE` directly in business logic or handlers; import the validated `config` object from `config.ts`.
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

## Queue / Workers (RabbitMQ / BullMQ)

- **Worker isolation**: One worker file per queue or job type.
- **Idempotency**: All background jobs must be designed to be idempotent where possible.
- **Contextual logging**: Log full context (job ID, payload shape, failure details) before rethrowing or acknowledging failed jobs.
- **Dead-letter queues**: Configure DLQs for all production queues to isolate failed messages.

---

## Error Handling

- **No swallowed errors**: Never write empty `catch` blocks (`catch (e) {}`).
- **Contextual logging**: Always log errors with descriptive context before handling or rethrowing.
- **Operational vs Programmer errors**: Distinguish between expected operational errors (invalid credentials, missing entity) and unexpected bugs (null pointers, database crash).
- **Centralized error middleware**: Use centralized error handling middleware in Express/Hono — do not format raw HTTP 500 error responses inside individual route handlers.
