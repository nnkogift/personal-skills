# Next.js App Router Reference

Load this file when working on Next.js projects. It supplements the main coding-standards SKILL.md with App Router-specific patterns.

---

## Routing & Layout Conventions

- Use route groups `(group)` to share layouts without affecting the URL
- Parallel routes (`@slot`) for complex dashboard layouts with independent loading states
- Intercepting routes (`(.)path`) for modal patterns (e.g. photo lightboxes, detail drawers)
- `loading.tsx` at the route segment level for streaming skeletons — not a single global spinner
- `error.tsx` per segment so errors are contained and don't blow up the whole page
- `not-found.tsx` for 404 handling at the appropriate segment level

---

## Data Fetching Patterns

### Server Components
```ts
// Fetch directly in the component — no useEffect, no SWR, no query client
export default async function InvoicePage({ params }: { params: { id: string } }) {
  const invoice = await getInvoice(params.id) // direct DB/API call
  return <InvoiceDetail invoice={invoice} />
}
```

- Always `async` server components when fetching
- Use `Promise.all` for parallel fetches to avoid waterfalls
- Never pass entire DB objects to client components — project only the fields needed

### Client Components with TanStack Query
```ts
// Hydrate on the server, take over on the client
export default async function InvoicePageWrapper({ params }) {
  const queryClient = new QueryClient()
  await queryClient.prefetchQuery({
    queryKey: invoiceKeys.detail(params.id),
    queryFn: () => getInvoice(params.id),
  })
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <InvoiceDetail id={params.id} />
    </HydrationBoundary>
  )
}
```

---

## Caching Strategy

- Use `cacheTag` and `cacheLife` with the `use cache` directive for fine-grained invalidation
- Group cache profiles by data volatility:
  - `cacheLife('hours')` — reference data, org units, metadata
  - `cacheLife('minutes')` — aggregate data, dashboard values
  - `cacheLife('seconds')` — near-realtime indicators
- On-demand revalidation via `revalidateTag()` in Server Actions after mutations
- Use `React.cache()` to deduplicate repeated calls within a single render pass

---

## Server Actions

- Server Actions handle form mutations — no separate API route needed for simple cases
- Always validate action inputs with Zod before touching the DB
- Return a typed result object, never throw from a Server Action that touches the UI:
  ```ts
  type ActionResult<T> = { success: true; data: T } | { success: false; error: string }
  ```
- Use `useActionState` (React 19) or `useFormState` (React 18) to wire action state to the form

---

## Environment Variables

- `NEXT_PUBLIC_` prefix only for values that must be on the client — keep the list minimal
- All env vars validated at startup with Zod in `src/lib/env.ts`:
  ```ts
  const envSchema = z.object({
    DATABASE_URL: z.string().url(),
    NEXT_PUBLIC_API_BASE: z.string().url(),
  })
  export const env = envSchema.parse(process.env)
  ```
- Import from `@/lib/env` everywhere — never `process.env` directly in components

---

## Performance

- Use `next/image` for all images — never raw `<img>` tags
- Dynamic imports (`next/dynamic`) for heavy client components not needed on initial paint
- `generateStaticParams` for static generation of known dynamic routes
- Avoid layout shift — always provide `width` and `height` for images or use `fill` with a sized container
