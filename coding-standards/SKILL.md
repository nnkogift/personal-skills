---
name: coding-standards
description: Gift's personal coding standards, file structure conventions, and project setup rules. ALWAYS load this skill when starting a new project, scaffolding files, setting up a repo, writing new components, creating hooks, naming functions, structuring folders, composing UI, or any time code organization or architecture is involved. Also load when reviewing code for quality or consistency.
---

# Coding Standards

This skill governs how all code is written, organized, and reviewed. These are non-negotiable defaults.
For framework-specific deep dives, see the references listed at the bottom of this file.

---

## TypeScript

- Strict mode always on (`"strict": true` in tsconfig)
- No implicit `any` — ever. Use `unknown` and narrow it properly
- Prefer `type` over `interface` unless you explicitly need declaration merging
- Use Zod for all runtime validation and derive TypeScript types from schemas with `z.infer<>`
- Avoid type assertions (`as X`) unless you can justify why the compiler cannot infer it
- Enums are recommended if not possible use `const` objects with `as const` and derive union types from them
- Keep types close to where they are used; only promote to a shared `types/` folder when genuinely shared across modules

---

## Project Structure

### Monorepos

```
root/
├── apps/
│   ├── web/          # Next.js
│   └── mobile/       # Flutter
├── packages/
│   ├── ui/           # Shared component library
│   ├── config/       # Shared tsconfig, biome config
│   └── types/        # Shared cross-app types only
├── pnpm-workspace.yaml
└── biome.json
```

- Apps live in `apps/`, shared code in `packages/`
- Each package has its own `tsconfig.json` extending `packages/config/tsconfig.base.json`
- Internal packages are referenced via workspace protocol: `"@repo/ui": "workspace:*"`
- Never import across `apps/` — shared code always goes through `packages/`

### Single App (Next.js)

```
src/
├── app/              # Next.js App Router routes
├── components/       # Shared/global components
│   └── ui/           # Primitive UI components (buttons, inputs, etc.)
├── features/         # Feature-scoped modules (see Component Rules)
├── hooks/            # Global reusable hooks
├── lib/              # Third-party client setup (queryClient, axios instance, etc.)
├── utils/            # Utility functions (see Utility Rules)
├── types/            # Shared TypeScript types and Zod schemas
└── constants/        # App-wide constants
```

---

## Component Rules (React / Next.js)

These rules are strict. Follow them on every component you write or touch.

### One Component Per File

- Every component lives in its own file. No exceptions.
- File name matches the component name in `PascalCase`: `UserCard.tsx`, `InvoiceTable.tsx`
- If a component is only used by one parent, co-locate it in a `components/` subfolder beside the parent rather than at
  the global level

### Composition Over Monoliths

- If a component exceeds ~150 lines of JSX, it must be decomposed
- Extract logical sections into named sub-components: `<InvoiceHeader />`, `<InvoiceLineItems />`, `<InvoiceTotals />`
- Prefer using context for passing down data to child components unless it is not deeply nested enough to warrant
  passing it explicitly
- Use compound component patterns for UI that shares implicit state (tabs, accordions, dropdowns)
- When in doubt, split it out — small focused components are always preferable to large ones
- Always use the specified design system components unless instructed otherwise avoid updating the component styles
  directly, use component-available props, if not possible, use tailwind if available

### Feature Modules

Group everything a feature needs together:

```
features/
└── invoices/
    ├── components/
    │   ├── InvoiceTable.tsx
    │   ├── InvoiceRow.tsx
    │   └── InvoiceFilters.tsx
    ├── hooks/
    │   ├── useInvoices.ts         # TanStack Query hook
    │   └── useInvoiceForm.ts      # React Hook Form logic
    ├── utils/
    │   └── invoice.utils.ts
    ├── schemas/
    │   └── invoice.schema.ts      # Zod schemas
    └── types/
        └── invoice.types.ts
```

### Server vs Client Components (Next.js App Router)

- Server components by default — never add `"use client"` unless the component needs browser APIs, event handlers, or
  React state/effects
- Push `"use client"` as far down the tree as possible (leaf components, not layouts)
- Data fetching happens in Server Components or via TanStack Query in Client Components — never both in the same
  component
- Never fetch data in a layout unless it is truly layout-level data (user session, nav config)

---

## Utility Rules

- Utils live in `utils/` at feature or global scope depending on reuse
- Group utils by domain, not by type: `date.utils.ts`, `currency.utils.ts`, `dhis2.utils.ts`
- Utils that only serve one component live in that component's folder
- Utils that serve multiple components in a feature live in `features/<name>/utils/`
- Utils used across features live in `src/utils/`
- Multiple utils of the same domain can share a file as long as it remains cohesive and focused
- Use `lodash-es` for collection manipulation, debouncing, deep clones, and object transforms — do not reimplement what
  lodash already does well
- Every util function is pure unless it explicitly needs a side effect (must be documented if so)
- Export utils as named exports, never default exports

---

## Forms

- All forms use `react-hook-form` — no uncontrolled inputs, no manual `useState` for field values
- Schema validation always via Zod integrated with RHF using `@hookform/resolvers/zod`
- The Zod schema is the single source of truth: define it first, derive the TypeScript type from it, pass it to
  `useForm`
- Extract form logic into a custom hook: `useInvoiceForm.ts`, `useLoginForm.ts`
- Form components receive the form methods as props or via context — they do not own the form state themselves
- Field-level error messages come from Zod via RHF — never hardcoded strings in JSX
- Always use `FormProvider` to provide the form context to the entire form tree. Dont pass down control as a prop.
- If a field has properties or visibility depending on a different form value, extract it into its own component, use
  the `useWatch` hook to listen to the dependency value.

```ts
// Pattern to follow always
const schema = z.object({email: z.string().email()})
type FormValues = z.infer<typeof schema>

const useLoginForm = () => {
		return useForm<FormValues>({resolver: zodResolver(schema)})
}
```

---

## Data Fetching

- All server state is managed by TanStack Query (`@tanstack/react-query`) — no raw `useEffect` for fetching
- Query keys are defined as typed constants in a `queryKeys.ts` file per feature
- Every query lives in a dedicated hook: `useInvoices.ts`, `usePatientById.ts`
- Mutations follow the same pattern with `useMutation` and always include `onError` handling
- Never put TanStack Query logic directly inside a component — always a hook

```ts
// queryKeys.ts — pattern to always follow
export const invoiceKeys = {
		all: ['invoices'] as const,
		list: (filters: InvoiceFilters) => [...invoiceKeys.all, 'list', filters] as const,
		detail: (id: string) => [...invoiceKeys.all, 'detail', id] as const,
}
```

---


## API Design

- REST for CRUD, flat resource URLs: `/invoices`, `/invoices/:id/lines`
- Consistent error response shape across all endpoints:
  ```ts
  { error: string; code: string; details?: unknown }
  ```
- All request bodies validated with Zod before any business logic runs
- Never trust client input downstream of the validation boundary

---

## Git

- Conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`
- PR descriptions explain the *why*, not the *what* — the diff shows the what
- Branch naming: `feature/`, `fix/`, `chore/`, `refactor/`
- Commits are atomic — one logical change per commit
- Delete merged branches immediately

---

## General Rules

- No magic numbers — extract to named constants with clear names
- Functions do one thing. If you need "and" to describe it, split it into two functions
- Delete commented-out code — git history exists for a reason
- All async functions handle errors explicitly. No silent catch blocks, no swallowed rejections
- No `console.log` left in committed code — use a proper logger or remove it
- Avoid deeply nested conditionals — use early returns and guard clauses

---

## References

Read these when working in a specific context:

- `references/nextjs.md` — Next.js App Router patterns, caching, server actions
- `references/flutter.md` — Flutter project structure, state management, offline-first patterns
- `references/dhis2.md` — DHIS2 app conventions, data store patterns, DHIS2 UI library usage
