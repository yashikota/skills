---
name: frontend-builder
description: Build frontend applications with Vite+, React, TypeScript, and TanStack Router. Use this skill for new frontends, SPAs, dashboards, games, tools, screen implementation, UI changes, Vite to Vite+ migration, vp dev/build/test/lint/fmt/check workflows, TanStack Router route design, loaders, search params, navigation, and error handling.
license: MIT
---

# Frontend Builder

Use Vite+'s `vp` CLI with React and TypeScript as the standard frontend stack. Use TanStack Router for routing.

## Core Rules

- For new projects, create a React + TypeScript project with Vite+ (`vp`).
- When migrating an existing Vite project to Vite+, use `vp migrate`.
- Use `pnpm` as the package manager.
- Use `playwright-cli` when browser verification is useful.
- After every code change, run `vp check --fix`.
- Prefer Vite+ lifecycle commands whenever possible: `vp dev`, `vp build`, `vp test`, `vp lint`, `vp fmt`, and `vp check`.
- Respect existing Vite+ configuration in `vite.config.ts`, including lint/fmt/test/run/pack/staged blocks.
- When using Vite+, do not create separate config files for Oxlint, Oxfmt, Vitest, or tsdown. Consolidate tool configuration in `vite.config.ts`.
- Make the first viewport show the actual usable app. Do not start with an explanatory landing page unless the user explicitly asks for one.
- Always write dev server stdout/stderr to `web.log` in the project root.

## New Project

1. Create the project with `vp`.

```bash
vp create vite -- --template react-ts
```

2. Install dependencies.

```bash
pnpm install @tanstack/react-router @tanstack/router-devtools lucide-react
```

3. If the project has no existing CSS convention, keep styling in `src/index.css` and component CSS. Do not add extra UI frameworks unless they materially help the task.

## Vite+ Configuration

In Vite+, consolidate all tool configuration in `vite.config.ts`.

```ts
import { defineConfig } from "vite-plus";

export default defineConfig({
  server: {},
  build: {},
  preview: {},
  test: {},
  lint: {},
  fmt: {},
  run: {},
  pack: {},
  staged: {},
});
```

Enable `lint.options.typeAware` and `lint.options.typeCheck` because type-aware linting is desired.

```ts
export default defineConfig({
  lint: {
    ignorePatterns: ["dist/**"],
    options: {
      typeAware: true,
      typeCheck: true,
    },
  },
  fmt: {
    singleQuote: true,
  },
});
```

## Existing Vite Migration

For an existing Vite project, inspect the current state first, then migrate to Vite+.

```bash
vp migrate
```

After migration, review changes to `vite.config.ts`, package scripts, and the lockfile. If the project already has tests, linting, formatting, or build workflows, run `vp check` or the individual `vp build`, `vp test`, and `vp lint` commands.

## Vite+ Commands

- `vp dev`: Start the Vite dev server with HMR.
- `vp build`: Run the production build through Rolldown. This runs the Vite build, not a custom package.json build script.
- `vp preview`: Serve the production build locally.
- `vp test`: Run Vitest once. Unlike standalone Vitest, this is not watch mode.
- `vp test watch`: Run tests in watch mode.
- `vp test run --coverage`: Run tests with coverage.
- `vp lint`: Run Oxlint.
- `vp fmt`: Run Oxfmt.
- `vp check`: Run format, lint, and typecheck in one pass.
- `vp check --fix`: Auto-fix formatting and lint issues.
- `vp run build`: Run the package.json `build` script.
- `vp run build -r`: Run the script across all workspace packages in dependency order.
- `vp pack`: Build a library through the tsdown integration.

## TanStack Router Priority Rules

Implement these rules by priority.

- Critical: register the router type, use the `from` parameter for type narrowing, type the root context, and use `queryOptions` for loader type inference.
- Critical: organize the route tree around hierarchy, layouts, index routes, and pathless layouts.
- High: configure router defaults, loaders, loaderDeps, TanStack Query integration, search param validation, and not-found/error handling.
- Medium: use `Link`, active states, `useNavigate`, relative paths, code splitting, and preloading when they fit the task.
- Low: use root context, `beforeLoad`, and dependency injection when shared dependencies such as auth, API clients, or feature flags are needed.

## Router Implementation

Create `src/router.tsx` and define a path-based route tree with `createRootRouteWithContext`, `createRoute`, and `createRouter`.

```tsx
import {
  Link,
  Outlet,
  createRootRouteWithContext,
  createRoute,
  createRouter,
} from "@tanstack/react-router";
import { TanStackRouterDevtools } from "@tanstack/router-devtools";

type RouterContext = {
  queryClient?: unknown;
};

function RootLayout() {
  return (
    <>
      <nav>
        <Link to="/" activeOptions={{ exact: true }}>
          Home
        </Link>
        <Link to="/settings">Settings</Link>
      </nav>
      <Outlet />
      <TanStackRouterDevtools />
    </>
  );
}

const rootRoute = createRootRouteWithContext<RouterContext>()({
  component: RootLayout,
  notFoundComponent: NotFoundPage,
  errorComponent: ErrorPage,
});

const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/",
  component: HomePage,
});

const settingsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/settings",
  validateSearch: (search) => ({
    tab: typeof search.tab === "string" ? search.tab : "profile",
  }),
  component: SettingsPage,
});

const routeTree = rootRoute.addChildren([indexRoute, settingsRoute]);

export const router = createRouter({
  routeTree,
  defaultPreload: "intent",
  defaultPreloadStaleTime: 30_000,
  scrollRestoration: true,
  context: {},
});

declare module "@tanstack/react-router" {
  interface Register {
    router: typeof router;
  }
}
```

Use `RouterProvider` in `src/main.tsx`.

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "@tanstack/react-router";
import { router } from "./router";
import "./index.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
);
```

## Data Loading

- Prefer route `loader` functions for route data instead of component-level `useEffect`.
- Define `loaderDeps` when search params or route params affect the loader cache.
- When using TanStack Query, combine `queryOptions` with `queryClient.ensureQueryData`.
- Wait for critical data in the loader, and split non-critical data into deferred loading or component-side fetching.
- Handle loader errors with `errorComponent` and not-found states with `notFoundComponent`.
- Route loaders for parent and child routes run in parallel, so avoid unnecessary sequential dependencies.

## Search Params

- Always validate search params with `validateSearch` and treat them as typed URL state.
- Provide defaults so the page does not break when the URL omits a search param.
- Design parent routes so child routes can inherit parent search param types when useful.
- Put only shared, restorable, or bookmarkable state in the URL.
- Configure a custom serializer only when complex search param values require one.

## Navigation

- Use `Link` for normal page navigation.
- Configure active states for current-location UI.
- Use `useNavigate` for programmatic navigation after form submits or command actions.
- When using relative paths, make the source route explicit.
- Consider route masks only when modal URLs or preserved background pages are required.

## Code Splitting And Preloading

- Split large routes, settings pages, admin screens, game screens, and other non-critical initial routes into lazy routes.
- Keep critical route definitions, loaders, `validateSearch`, and error boundaries in the main route file.
- Enable auto code splitting when the environment supports it.
- Use `defaultPreload: "intent"` for primary navigation.
- Use manual preloading only for screens that are highly likely to be opened next.

## UI Implementation

- Split page components by route. Even for small apps, avoid putting everything in `src/App.tsx`; create `src/pages/` and `src/components/`.
- Implement primary, empty, error, and loading states when they are relevant.
- Use `lucide-react` for icon buttons when an icon is needed.
- Prevent text from overflowing buttons or cards by using stable dimensions, wrapping, and responsive constraints.
- For apps that need visual content, such as games or image-heavy experiences, use real assets, searched images, generated images, Canvas, Three.js, or another appropriate visual approach. Do not rely on empty decoration.

## Logs And Dev Server

Start the dev server from the project root and write logs to root-level `web.log`.

```bash
vp dev --host 0.0.0.0 > web.log 2>&1
```

After startup, inspect `web.log` for the served URL, compile errors, and runtime errors. Fix issues and restart when needed.

## Common Errors

- `vp build` does not run the expected script: `vp build` is for the Vite build. Use `vp run build` to run the package.json script.
- Imports are broken after `vp migrate`: run `vp install`, then `vp check` to find remaining issues.

## Verification

- Run `vp build` and fix type errors and build errors.
- Depending on the change, run `vp test`, `vp lint`, `vp fmt`, and `vp check`.
- Start the dev server and confirm Vite+ startup logs are written to `web.log`.
- Directly visit each TanStack Router path and verify initial render, search params, not-found behavior, and navigation.
- In the final response, briefly report changed files, the dev server URL, verification performed, and anything not verified.
