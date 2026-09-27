# React Standards

This reference defines component architecture, hooks, state management, forms, and data fetching conventions for React applications. It supplements the main `coding-standards` SKILL.md.

For Next.js App Router-specific patterns (Server vs Client components, routing, server actions, caching), see `references/nextjs.md`.

---

## Component Architecture

### One Component Per File

- Every component lives in its own dedicated file. No multiple exported components in a single file.
- File names match the component name in `PascalCase`: `UserProfileCard.tsx`, `InvoiceTable.tsx`, `UserCard.tsx`.
- Co-locate subcomponents: if a component is only used by one parent component, place it in a `components/` subfolder beside that parent rather than at the global or module level.

### Composition Over Monoliths

- **JSX length limit**: If a component exceeds ~150 lines of JSX, decompose it immediately.
- **Logical extraction**: Extract logical UI sections into named subcomponents: `<InvoiceHeader />`, `<InvoiceLineItems />`, `<InvoiceTotals />`.
- **State passing**: Prefer using context for passing data down to child components when nesting warrants it, avoiding prop drilling deeper than 2 levels.
- **Compound components**: Use compound component patterns for UI elements that share implicit state (e.g. tabs, accordions, dropdowns).
- **Design system components**: Always use specified design system components where available. Avoid updating component styles directly; use component-provided props, and use Tailwind utilities when customization is required.

### Component File Structure

Maintain a consistent ordering inside component files:

```tsx
// 1. Imports
// 2. Types / Props
// 3. Constants local to this file
// 4. Component function
// 5. Subcomponents (if strictly private, small, and tightly coupled)
// 6. Export statement
```

### Props Conventions

- Define component props as a `type` alias named `[ComponentName]Props` (e.g. `UserProfileCardProps`).
- Destructure props directly in the function signature.
- Keep prop lists focused. If a component requires more than ~7 props, consider decomposing the component, using compound components, or providing context.

```tsx
type UserProfileCardProps = {
  userId: string
  onSelect: (id: string) => void
  isSelected?: boolean
}

export function UserProfileCard({ userId, onSelect, isSelected = false }: UserProfileCardProps) {
  // ...
}
```

---

## Feature Modules (Non-Next.js React Apps)

For Vite, DHIS2, or standard React SPAs, organize code under a `src/modules/` directory:

```
src/
├── modules/
│   └── invoices/
│       ├── components/
│       │   ├── InvoiceTable.tsx
│       │   ├── InvoiceRow.tsx
│       │   └── InvoiceFilters.tsx
│       ├── hooks/
│       │   ├── useInvoices.ts
│       │   └── useInvoiceForm.ts
│       ├── utils/
│       │   └── invoice.utils.ts
│       ├── schemas/
│       │   └── invoice.schema.ts
│       └── types/
│           └── invoice.types.ts
└── shared/           # Promoted here when used across multiple modules
    ├── components/
    │   └── ui/
    ├── hooks/
    ├── utils/
    ├── schemas/
    └── types/
```

- Rename any legacy `features/` directories to `modules/`.
- Create sub-folders (`components/`, `hooks/`, `utils/`, `schemas/`, `types/`) on demand — do not pre-scaffold empty folders.
- Avoid barrel `index.ts` files inside module subfolders — import directly from file paths.
- Promote code to `src/shared/` only when genuinely shared across multiple feature modules.

---

## Utilities

- Place utils at the closest scope covering their usage (`components/`, `modules/<name>/utils/`, or `src/shared/utils/`).
- Group utils by domain with `.utils.ts` suffix: `date.utils.ts`, `currency.utils.ts`, `dhis2.utils.ts`.
- Every utility function must be pure unless explicitly designed for a side effect (which must be documented).
- Export utils as named exports, never default exports.
- Use `lodash-es` for array/collection manipulation, debouncing, throttling, deep cloning, and transforms — do not reinvent existing standard utilities. Import individual functions:

```ts
import { groupBy, orderBy, uniqBy } from 'lodash-es'
```

---

## React Hooks

- **One hook per file**: Named in `camelCase` with `use` prefix: `useInvoices.ts`, `useInvoiceForm.ts`.
- **Memoization**: Use `useCallback` and `useMemo` for referential stability on callbacks and expensive computations.
- **State minimization**: Use `useState` and `useRef` only when necessary for local component state.
- **Avoid unnecessary `useEffect`**: Avoid `useEffect` for derived state or data synchronization. Use it only for real external side effects (e.g. event listeners, subscriptions). Always specify complete dependency arrays.

---

## Forms

- **React Hook Form + Zod**: All forms use `react-hook-form` with `@hookform/resolvers/zod`. Never manage form state manually with `useState`.
- **Zod schema as single source of truth**: Define the Zod schema first, infer the TypeScript type, and supply both to `useForm`.
- **Custom form hooks**: Encapsulate form configuration and default values in a dedicated hook (`useInvoiceForm.ts`, `useLoginForm.ts`).
- **FormProvider**: Always wrap the form tree in `FormProvider` to provide context. Avoid passing the `control` object down through props.
- **Dependent fields**: When field visibility or validation depends on other fields, isolate that section into its own component and listen with `useWatch`.
- For complete form architecture patterns, field arrays, and validation modes, consult `../forming/SKILL.md`.

```tsx
// Pattern for form hook and schema definition
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})

export type LoginFormValues = z.infer<typeof loginSchema>

export const useLoginForm = () => {
  return useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
    },
  })
}
```

---

## Data Fetching (TanStack Query)

- **Server state management**: All server state is managed via TanStack Query (`@tanstack/react-query`). Never use ad-hoc `useEffect` + `useState` fetching.
- **Typed query keys**: Define query keys as typed constants using `as const` in a `queryKeys.ts` file located in the module's `constants/` folder (or `src/shared/constants/` if shared):

```ts
// _shared/constants/queryKeys.ts or modules/invoices/constants/queryKeys.ts
export const invoiceKeys = {
  all: ['invoices'] as const,
  lists: () => [...invoiceKeys.all, 'list'] as const,
  list: (filters: InvoiceFilters) => [...invoiceKeys.lists(), filters] as const,
  details: () => [...invoiceKeys.all, 'detail'] as const,
  detail: (id: string) => [...invoiceKeys.details(), id] as const,
}
```

- **Dedicated custom query hooks**: Wrap each query and mutation in a dedicated hook file:

```ts
// useInvoices.ts
import { useQuery } from '@tanstack/react-query'
import { invoiceKeys } from '../constants/queryKeys'
import { fetchInvoices } from '../api/invoice.api'

export function useInvoices(filters: InvoiceFilters) {
  return useQuery({
    queryKey: invoiceKeys.list(filters),
    queryFn: () => fetchInvoices(filters),
  })
}
```

- **Mutations & error handling**: Wrap mutations with `useMutation` and always include `onError` handling and query invalidation on success.
- **No queries directly in JSX**: Keep all TanStack Query logic encapsulated inside custom hooks.
