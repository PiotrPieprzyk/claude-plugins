---
name: app
description: Build or split frontend app components (pages/features) so they satisfy their UI story specs with high cohesion and low coupling — one folder per feature, one root component, split by concern, dependencies injected via interfaces. Use when creating a new page/feature component from a story in docs/stories/**, when a component is growing past ~150-200 lines or mixing concerns (fetching, pagination, realtime, rendering), or when refactoring existing UI to a story-driven structure.
---

# Building app components

Components are built to make their story tests (`docs/stories/<Domain>/*.test.ts`)
pass. The story spec defines the `data-testid` tree, copy and behavior; the component is
free in everything else. The goal is **high cohesion, low coupling**: each file does one
job, can be read on its own, and doesn't know how its siblings work.

## 0. Before you start: read in this order

It's a fixed list, not a search:

1. **All story specs + tests for the feature** — `docs/stories/<Domain>/*.md` and
   `.test.ts`. Skim every story touching the component first; they share one testid
   tree and describe different facets of the same component.
2. **`docs/stories/<Domain>/shared/render.ts`** — the exact provider nesting the
   component is mounted under (and any provider deliberately missing), the route it
   renders at, and the **exact import path** the test uses (e.g.
   `@/app/Room/RoomPage` as a bare folder import needs an `index` re-export).
3. **`shared/mocks.ts`, `shared/builders.ts`, `shared/get/*.ts`** — the mocks give the
   exact dependency interfaces you code against; the builders give the data shapes; the
   getters give the testid nesting to render, outer to inner.
4. **The matching part of the design** under `docs/design/` — structure, copy, tone per
   state. Grep large mockups for a testid or the feature name.
5. **The closest existing sibling feature**, to copy its conventions (a list page → the
   existing list feature; a single-submit form → the existing form). Read its root and
   one or two children in full.
6. **Shared presentational components** (`components/`, `comp/`, `ui/`) — list them and
   skim their props. Missing primitives the design needs get built there first, following
   the existing pattern (variant maps, `className`/`...rest` passthrough, theme tokens
   only).
7. **The project's interfaces cheat sheet** (`docs/frontend-interfaces.md`, see
   `interfaces-cheat-sheet.md` in this skill folder). If it doesn't exist, create it
   once from the dependency interfaces and keep it short; afterwards re-read the source
   only if the sheet looks stale.

## 1. One folder per feature, one root component

```
app/<Domain>/<Feature>/          # or the framework's routing folder + a feature folder
  index.ts                       # export {Feature} from './Feature' — if tests import the folder
  Feature.tsx                    # root: outer shell + wrapper data-testid, composes the parts
  FeatureHeader.tsx              # one region
  FeatureList.tsx                # another region
  PaginationAdapter.tsx          # glue between a shared primitive and feature state
  useOwnThing.ts                 # reusable/independently testable logic
```

- The root component (named like the folder) is the only thing imported from outside.
  It owns the layout shell and the wrapper `data-testid` from the story.
- Group features by the same domains as `docs/stories/<Domain>/`, so a domain's specs,
  tests and components line up.

## 2. Split by concern, not by size alone

Split out a new file when a component takes a second responsibility, even if short:

- Two visually/behaviorally distinct regions (a header with status + actions; a list with
  loading/empty/error states) → one component per region.
- Fetching/derived-state logic that's reusable or worth reasoning about alone → its own
  `useXxx.ts` hook.
- Glue that adapts a shared primitive (`Pagination`, `Dialog`) to feature state → its own
  `XxxAdapter.tsx`.

~150-200 lines is a signal to split by one of these rules — never cut arbitrarily.

## 3. Keep files independent

- One export per file, named after the file.
- A component/hook may depend on **dependency hooks/interfaces** (API client, router,
  session, socket — via context or props) and **shared presentational components**. It
  must not import a sibling's internals or reach into a sibling's state.
- Two siblings needing the same data each call the same hook; don't lift state into the
  parent and prop-drill it just to dedupe a fetch. Solve caching with a caching layer
  (TanStack Query, framework cache) when one exists.
- For a child to notify a sibling (list reports an error the header shows), use a small
  feature-scoped event bus or store, not a callback relayed through the root.
- A file must be understandable by reading only it plus the interfaces it calls.

## 4. Dependencies come from one composition root

- Components never import concrete infrastructure (fetch wrappers, SDK clients, real
  socket, `window.location`) directly. They use interfaces provided via context/props.
- Concrete implementations are created once (memoized) in the composition root
  (`App.tsx`, root `layout.tsx`/providers file) and handed down through providers. Never
  module-level singletons. This is what lets story tests swap them for mocks.
- Expected API failures come back as values (`Result | Error` / discriminated union),
  and the component branches on them; exceptions are for bugs.

## 5. Render exactly what the story specifies

- Every `data-testid` in the story tree, with the same nesting. Don't add testids the
  stories don't use.
- Copy exactly as in the `.md` (error messages, labels, empty states).
- State signals the tests read: `disabled`, `aria-busy="true"` while pending, focus,
  colors via theme tokens (never raw hex in components).
- Navigation through the injected router, to the route the story names.
- Realtime: subscribe in an effect and return the unsubscribe; prefer a scoped
  subscription (one id) over subscribe-all + filter when the component only needs one
  item, but still ignore events whose id doesn't match.

## 6. Framework notes

- **Next.js App Router**: keep server components thin — fetch data and pass plain props
  to a client feature component that owns interaction and is what the story renders.
  Interfaces (API client, router) reach client components via a providers component in
  the root layout. `next/navigation` is wrapped behind the router interface, not used
  directly in feature components.
- Respect the project's lint import rules (layer boundaries, alias-only imports).

## 7. Done when

```
npx vitest run docs/stories/<Domain>
```

All story tests for the feature pass without editing them. If a test seems wrong, stop
and ask — the test is the spec; change the `.md` + test first (`/ui:story`,
`/ui:test`) only with the user's agreement. Then run lint and type-check.
