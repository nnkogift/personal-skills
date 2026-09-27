---
name: loading-skeletons
description: How to build consistent, low-layout-shift shimmer skeleton loading states in Next.js/React apps using the project's own design system. Use this skill whenever adding, editing, or reviewing a page or component that is async, suspendable, or has any loading state — a new route, a new `loading.tsx`, a new `<Suspense>` boundary, a new data-fetching hook with an `isLoading`/`isPending` flag, or a component that renders conditionally on data not yet being available. Also use it when auditing or fixing existing loading indicators (spinners, "Loading..." text, mismatched skeletons) or investigating Cumulative Layout Shift (CLS) issues tied to data loading.
---

# Loading Skeletons

Build shimmer-based loading states that mirror the real layout so nothing shifts when data arrives.

## Core principle

A loading state is a costume the real component wears temporarily — not a different component. The loading version must have the **exact same layout structure** as the loaded version: same containers, wrappers, grid/flex structure, padding, and spacing. Only the innermost content nodes (text, avatars, images, numbers, buttons) are swapped for shimmer placeholders. If the container itself appears, disappears, resizes, or restyles between loading and loaded states, that transition is itself a layout shift — the fix isn't "no containers in loading state," it's "identical containers in both states."

Get this right and Cumulative Layout Shift (CLS) during data loading approaches zero, and the app's various loading states look and feel like one designed system instead of a grab-bag of spinners.

## Before writing anything: fetch current docs

Do not rely on memorized component APIs — they drift across versions.

1. Check `package.json` for the design system in use (Mantine, Chakra, shadcn/ui, Ant Design, MUI, etc.) and its installed version.
2. Fetch that library's current docs for its shimmer/skeleton primitive at that version. For example, Mantine's `Skeleton` (`@mantine/core`) takes `height`, `width`, `radius`, `circle`, `animate`, and `visible` (the last for wrapping already-mounted content and toggling an overlay rather than swapping it out).
3. Check the installed Next.js version and fetch its current docs for `loading.js` / `<Suspense>` streaming behavior — the file-convention semantics have stayed fairly stable but details (Server vs Client Component defaults, granular Suspense guidance) are worth reconfirming per version.
4. Never introduce a new shimmer/skeleton dependency. The design system already has one — find it before building anything custom.

## Workflow

Run this whenever you add or touch anything async/suspendable. It's the same workflow whether you're building one new component or auditing a whole app.

### 1. Identify the loading mechanism

Every async component/route uses exactly one of these — pin down which:

- **`loading.tsx` / `loading.js`** — Next.js App Router convention. Automatically nests inside the route's `layout.tsx` and wraps `page.tsx` (and children) in a `<Suspense>` boundary. Server Component by default, receives no props. Good for "nothing meaningful to show until this whole segment resolves."
- **Manual `<Suspense fallback={...}>`** — for granular, per-component streaming. Wrap each independently-async piece in its own boundary so it streams in on its own schedule without blocking siblings (e.g. a sidebar shouldn't wait on a slow chart).
- **Boolean/prop-driven state** — a client-side `isLoading`/`isPending` flag (from a query hook, form state, etc.) controlling conditional rendering, unrelated to Suspense.

If you're adding new async content to an existing page, decide whether it should get its own `<Suspense>` boundary (so it streams independently) rather than being folded into a coarser boundary that would delay unrelated content.

### 2. Map the content shape

Look at the real, loaded version of the component. Note:
- Every container/wrapper and its layout props (padding, gap, grid/flex direction).
- Every leaf content node: text (and roughly how many lines / how long), images/avatars (and their dimensions), buttons, badges, numbers.
- Which parts have variable size (unknown text length, a dynamic number of list items, optional fields) — these are the highest-risk spots for a loading→loaded mismatch.

### 3. Build the skeleton from the real component, not from scratch

- Start from the real component's markup/JSX. Keep every container, wrapper, and layout prop identical.
- Replace only the leaf content nodes with the design system's shimmer primitive, sized to match what they stand in for (a 40px circular avatar gets a 40px circular skeleton, a heading gets a skeleton at roughly the heading's line-height, etc.).
- For variable-size content, pick a sensible fixed size (typical/expected length) or an explicit `min-height` — never let a skeleton collapse to zero and pop open when real content lands.
- Co-locate the skeleton next to the component it mirrors (e.g. `UserCard.tsx` + `UserCard.skeleton.tsx`), so a future layout change to the real component makes the skeleton's drift obvious.
- Reuse one skeleton across every instance of the same content shape — don't duplicate markup per usage site.

### 4. Wire it into the right mechanism

- `loading.tsx`: its contents *are* the skeleton for that route segment — compose it from the page's skeleton pieces.
- `<Suspense fallback={...}>`: the fallback prop is the skeleton for that specific boundary.
- Boolean-driven: the `isLoading` branch renders the skeleton instead of a spinner/placeholder text, with the same conditional logic otherwise unchanged.

### 5. Test the transition, not just the static skeleton

The thing that matters is the **moment of switching** from skeleton to real content — a skeleton can look perfect sitting still and still cause a jump the instant it's replaced, if its dimensions don't match.

- **Suspense/`loading.tsx`**: use React DevTools to manually toggle the Suspense boundary from fallback → resolved, watching that exact frame for any reflow of surrounding content.
- **Boolean-driven**: flip the `isLoading` flag from `true` → `false` (via DevTools state override, or a temporary hardcoded value) and watch the same transition.
- Throttle the network or add artificial delay on a couple of routes so the swap happens slowly enough to inspect frame-by-frame, and confirm nothing around the swapped element moves.
- Pay special attention to the variable-size content flagged in step 2 — it's the most likely source of a mismatch.
- Measure, don't just eyeball: use Chrome DevTools' Performance panel or a Web Vitals overlay to capture CLS specifically across the loading→loaded transition, and confirm it doesn't regress versus before your change.

## Checklist before calling it done

- [ ] Used the design system's own shimmer/skeleton primitive (no new dependency, no custom-built shimmer).
- [ ] Checked current docs for that primitive's props/version instead of assuming.
- [ ] Skeleton's container/layout structure is pixel-for-pixel identical to the loaded state's.
- [ ] Only leaf content nodes were replaced with shimmer, not whole sections wrapped in a new placeholder box.
- [ ] Variable-size content has a sensible fixed size or `min-height` rather than collapsing.
- [ ] Skeleton is co-located with (or otherwise clearly tied to) the component it mirrors.
- [ ] Correct mechanism chosen/confirmed (`loading.tsx`, `<Suspense>`, or boolean) — and boundaries are as granular as the content's independence allows.
- [ ] Tested the actual loading→loaded transition (not just the resting skeleton), ideally with throttled network, and confirmed no visible shift.
- [ ] Confirmed via DevTools Performance/Web Vitals that CLS doesn't regress on the touched route(s).