# Project Structure Standards

This reference defines repository layout conventions, workspace setup, and modular organization. It supplements the main `coding-standards` SKILL.md.

---

## Monorepos (Bun Workspaces)

Monorepos isolate deployable applications from shared internal packages.

### Directory Layout

```
root/
├── apps/
│   ├── web/          # Next.js web application
│   └── mobile/       # Flutter mobile application
├── packages/
│   ├── ui/           # Shared UI component library
│   ├── config/       # Shared tsconfig, linter config
│   └── types/        # Shared cross-app types only
├── package.json      # Root workspaces definition
├── eslint.config.js  # Root or shared linting
└── .prettierrc
```

### Workspace Rules

- **Root workspaces declaration**: Bun workspaces are declared in the root `package.json` `workspaces` field:
  ```json
  {
    "workspaces": ["apps/*", "packages/*"]
  }
  ```
- **Package references**: Internal packages are referenced via the workspace protocol:
  ```json
  {
    "dependencies": {
      "@repo/ui": "workspace:*",
      "@repo/config": "workspace:*",
      "@repo/types": "workspace:*"
    }
  }
  ```
- **TSConfig inheritance**: Each package has its own `tsconfig.json` extending `packages/config/tsconfig.base.json`.
- **Strict boundary isolation**: Never import code directly across `apps/`. All shared code must live in `packages/` and be imported through workspace packages.

---

## Single App Architecture

### Next.js App Router

Next.js apps organize code by route hierarchy and use private folders (`_shared`) co-located with routes.

```
src/
├── app/
│   └── (invoices)/
│       ├── page.tsx
│       └── _shared/          # Private folder — excluded from routing
│           ├── components/
│           ├── hooks/
│           ├── utils/
│           ├── schemas/
│           └── types/
├── shared/           # Cross-route shared code — promoted when used across routes
│   ├── components/
│   │   └── ui/       # Primitive UI components (buttons, inputs, etc.)
│   ├── hooks/
│   ├── utils/
│   ├── schemas/
│   └── types/
├── lib/              # Third-party client setup (queryClient, axios, etc.)
└── constants/        # App-wide constants
```

- **No top-level loose folders**: No standalone `src/hooks/`, `src/utils/`, or `src/types/` folders at the root of `src/`. Everything global lives under `src/shared/`.
- For in-depth Next.js App Router routing patterns, see `references/nextjs.md`.

### Modular React / DHIS2 Web Apps

Non-Next.js React apps use a `src/modules/` folder for feature isolation.

```
src/
├── modules/
│   └── invoices/
│       ├── components/
│       ├── hooks/
│       ├── utils/
│       ├── schemas/
│       └── types/
└── shared/           # Promoted here when used across multiple modules
    ├── components/
    │   └── ui/
    ├── hooks/
    ├── utils/
    ├── schemas/
    └── types/
```

- Rename any existing `features/` directory to `modules/`.
- For in-depth React architecture patterns, see `references/react.md`.
