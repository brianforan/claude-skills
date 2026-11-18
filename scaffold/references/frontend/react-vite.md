# Frontend: React + Vite + TypeScript

**Why React (over Vue) as the default:** the largest ecosystem, the most examples and component libraries, and the most help available. Vue is a good framework too, but one default keeps projects consistent. **Why Vite and not Next.js:** these projects have a separate API, so a client-side SPA is simpler. Server-side rendering is a different architecture; write a new reference if a project needs it.

## Tooling

| Concern | Choice |
|---|---|
| Scaffold | `npm create vite@latest web -- --template react-ts`, then conform to this file |
| Routing | React Router (data/router mode, `createBrowserRouter`) |
| Server state | TanStack Query: all API data goes through it (caching, loading/error states) |
| Local state | `useState` / `useReducer`; add a store only when state is shared across distant components |
| Styling | Tailwind CSS |
| Components | Plain Tailwind to start. Add shadcn/ui (copies components into `src/components/ui/`, so you own the code) once the UI needs real primitives like dialogs, menus or comboboxes |
| Forms | react-hook-form + zod (same validation library as the Node backend) |
| Lint + format | ESLint (Vite's config + `typescript-eslint`, `react-hooks`) + Prettier |
| Tests | Vitest + React Testing Library (+ `jsdom`) |

## Layout

```
web/
├── index.html
├── vite.config.ts            # dev proxy: '/api' → http://localhost:8000
├── src/
│   ├── main.tsx              # QueryClientProvider + RouterProvider
│   ├── router.tsx            # route table
│   ├── lib/
│   │   ├── api.ts            # fetch wrapper: base URL, JSON, error shape, X-Request-ID passthrough
│   │   └── query-client.ts
│   ├── components/
│   │   ├── ui/               # shadcn primitives
│   │   └── layout/           # app shell, nav
│   ├── features/             # one folder per feature: components, hooks, api calls together
│   │   └── health/
│   │       ├── use-health.ts # useQuery(['health'], ...)
│   │       └── HealthBadge.tsx
│   └── pages/                # route-level components; compose features
└── src/test/setup.ts
```

Group by **feature**, not by file type. Everything about "orders" lives in `features/orders/`.

## Rules

- **`VITE_*` env vars are public.** They're baked into the JS bundle anyone can download. Never put secrets there. The frontend calls the API, and the API holds the secrets.
- In dev, call relative `/api/...` and let the Vite proxy forward it, so there's no CORS setup. In prod, serve web and API from the same origin or set `VITE_API_BASE_URL`.
- `lib/api.ts` parses the standard error JSON and throws a typed `ApiError`, so components handle errors one way.
- Components fetch nothing directly. They use hooks from their feature folder, which use TanStack Query.
- In event handlers call `mutation.mutate(vars, { onSuccess })`, not `await mutation.mutateAsync(vars)`. `mutateAsync` rethrows, so a failed request becomes an unhandled promise rejection; `mutate` puts it in `mutation.error` for the UI.

## Conforming the Vite template

The official template changes over time. Last checked 2026-11: it ships **oxlint** and **TypeScript 7**, and sets `erasableSyntaxOnly`.
- Replace oxlint with the ESLint setup above (remove the `oxlint` dependency, `.oxlintrc.json` and the `lint` script). Decided 2026-11: ESLint stays the standard (same tool in every app). Don't re-raise this drift unless something changes, such as ESLint falling behind on TypeScript support. Pin TypeScript to the newest version typescript-eslint accepts (`npm view typescript-eslint peerDependencies`).
- Keep `erasableSyntaxOnly`. It means no constructor parameter properties (`constructor(readonly x: T)`), no `enum`s and no `namespace`s: declare fields explicitly and use union types or `as const` objects.
- Add `"strict": true` explicitly to `tsconfig.app.json`; don't rely on compiler defaults.
- Delete `web/README.md`, `web/.gitignore` (the root one covers it), `src/App.css`, `src/assets/` and demo SVGs.
- `RouterProvider` comes from `react-router/dom`; everything else comes from `react-router`.
- Vitest config goes in `vite.config.ts` via `defineConfig` from `vitest/config`.

## Logging

- No `console.log` left in committed code (ESLint `no-console: warn`).
- An error boundary at the router root catches render errors, shows a fallback, and (optional, later) POSTs a small report to `/api/client-errors`, which the API logs with ECS fields. Elastic's RUM agent is the full-featured upgrade if real-user monitoring is ever needed.

## Commands (README)

```
npm install
npm run dev      # http://localhost:5173
npm run build
npm test
npm run lint
```
