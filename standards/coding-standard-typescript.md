# TypeScript Coding Standard

Applies to `roamkit-web` (Next.js 15 App Router).

## Runtime

- Node.js LTS (match CI workflow version).
- Next.js 15 with App Router.
- TypeScript strict mode enabled.

## Layout

```
app/           # Routes and layouts (App Router)
components/    # Reusable UI components
lib/           # API client, utilities, shared types
```

- Prefer **Server Components** by default; add `'use client'` only when needed (interactivity, hooks).
- API access centralized in `lib/api.ts`.

## Style and lint

| Tool | Purpose |
|------|---------|
| **ESLint** | Lint (Next.js recommended config) |
| **Prettier** | Formatting (if configured in repo) |

Run before PR:

```bash
npm run lint
```

## Naming

| Element | Convention | Example |
|---------|------------|---------|
| Components | `PascalCase` | `PlanCard.tsx` |
| Files (components) | `PascalCase` or `kebab-case` (pick one per repo, stay consistent) |
| Functions | `camelCase` | `fetchPackages` |
| Types/interfaces | `PascalCase` | `Package`, `ApiError` |
| Env vars | `NEXT_PUBLIC_*` for browser-exposed values |

## API client

- Base URL from `process.env.NEXT_PUBLIC_API_URL`.
- Typed responses for `/api/v1/*` endpoints.
- Handle HTTP errors with user-safe messages; log details server-side only.
- No secrets in client bundles — only `NEXT_PUBLIC_*` in browser code.

## Components

- Small, focused components; extract repeated UI patterns.
- Props typed with explicit interfaces.
- Avoid prop drilling beyond two levels — consider composition or context sparingly.

## Styling

- Use the project's chosen solution (Tailwind CSS expected for greenfield).
- Responsive layouts mobile-first.
- Accessible markup: semantic HTML, labels on forms, sufficient contrast.

## Testing

- Component tests for critical flows (`/plans` listing, error states).
- Lint + `next build` must pass in CI.
- E2E optional until Faza 2+; staging smoke covers deploy health.

## Performance

- Use `next/image` for images.
- Dynamic import heavy client components.
- Avoid unnecessary client-side fetching when Server Components can fetch.

## Related

- [API versioning](./api-versioning.md)
- [Definition of Done](./definition-of-done.md)
