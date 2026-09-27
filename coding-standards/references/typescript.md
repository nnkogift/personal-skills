# TypeScript Standards

This reference defines TypeScript conventions and type-safety rules. It supplements the main `coding-standards` SKILL.md.

---

## Strict Type Safety

- **Strict mode always enabled**: Ensure `"strict": true` in `tsconfig.json`.
- **No implicit `any` — ever**: When type is unknown, use `unknown` and narrow it properly with type guards or schemas.
- **Avoid type assertions (`as X`)**: Treat type assertions as code smells unless you can justify why the compiler cannot infer the type (e.g. external interop, specific DOM APIs).
- **No non-null assertions (`!`)**: Use optional chaining (`?.`) or explicit null checks / guard clauses instead.

---

## Type Definitions

- **Prefer `type` over `interface`**: Use `type` aliases by default. Only use `interface` when you explicitly require declaration merging or extending third-party library interfaces.
- **Enums vs Const Objects**:
  - TypeScript enums are recommended when appropriate.
  - If enums are not used, use `const` objects with `as const` and derive union types from them:

```ts
export const OrderStatus = {
  PENDING: 'PENDING',
  PROCESSING: 'PROCESSING',
  COMPLETED: 'COMPLETED',
  CANCELLED: 'CANCELLED',
} as const

export type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus]
```

---

## Runtime Validation & Type Inference

- **Zod as single source of truth**: Use Zod for all runtime validation (API payloads, environment variables, data stores, forms).
- **Derive TypeScript types from Zod schemas**: Never define redundant manual types alongside Zod schemas. Use `z.infer<typeof schema>`:

```ts
import { z } from 'zod'

export const userSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(['ADMIN', 'USER', 'GUEST']),
})

export type User = z.infer<typeof userSchema>
```

---

## Type Organization & Locality

- **Keep types close to usage**: Define types in the file or module where they are used.
- **Promotion hierarchy**:
  1. Component/function-local types → stay in that file.
  2. Feature/route types → place in `_shared/types/` (Next.js) or `modules/<name>/types/` (React).
  3. App-wide types → place in `src/shared/types/` only when genuinely shared across multiple routes/modules.
  4. Monorepo shared types → place in `packages/types/` only when shared across distinct apps.
- **No giant type dumping grounds**: Avoid monolithic `types.ts` files containing unrelated domain types.
