---
name: test
description: Write a behavior-focused UI story integration test (docs/stories/<Domain>/<story-slug>.test.ts) from its story spec .md — renders the real component with mocked dependencies, queries by walking the data-testid tree and asserts what the user perceives. Use when the user asks for frontend/UI story tests, component integration tests, or tests for a component that isn't implemented yet or is about to be refactored.
---

# Writing UI story tests

These are integration tests of a screen/component **as the user experiences it**: the
real component tree renders with only its outside dependencies replaced (API, router,
session, realtime socket, clock, storage). Assertions target what a user perceives —
text, enabled/disabled, focus, busy state, color of a state, where they navigate.

They may be written before the component exists; failing then is expected and correct.
Never adjust an assertion to make it pass — the test is the spec.

See `examples.md` in this skill folder for full examples of every shared file.

## 1. Layout: grouped by domain

```
<package>/docs/stories/
  testUtils.ts                       # design tokens (DESIGN_COLORS, DESIGN_FONTS) for all domains
  <Domain>/
    <story-slug>.md                  # spec (/ui:story)
    <story-slug>.test.ts             # this skill
    shared/
      mocks.ts                       # createMock<Dependency>(overrides) per injected dependency
      builders.ts                    # build<Thing>(overrides) per API response shape + DEFAULT_* ids
      render.ts                      # render<Component>(deps, route?) — providers + component
      get/
        <component>.ts               # wrapper getter + one getter per testid in its tree
```

- Folder is always `docs`. In a full-stack package the domain may have `ui/` and `api/`
  subfolders (`docs/stories/<Domain>/ui/...`); follow what exists.
- Ensure the test runner includes `docs/stories/**/*.test.ts` and there's an alias for
  it (e.g. `@docs` → `./docs`); use aliases, never `../` imports. Add them if missing.
- If the stories assert computed colors/fonts (`toHaveStyle`) and the styles come from a
  CSS pipeline (Tailwind, CSS variables, `@layer`), jsdom can't resolve them — run the
  stories project in real-browser mode (Vitest browser mode + Playwright) as a separate
  test project. Otherwise jsdom is fine.

## 2. Read before writing

1. The story `.md` — `data-testid` tree, exact copy, rules, each `#### WHEN` with its
   `Prerequisites`.
2. Other stories in the domain and their `shared/` — reuse mocks, builders, render
   helpers and getters; never redefine one locally.
3. The design source (`docs/design/`) for exact copy and colors not spelled out in the
   `.md`.
4. To wire `render.ts`: the app's composition root (where real implementations are
   provided — `App.tsx`, root `layout.tsx`, providers file) and the dependency interfaces
   (API client, router, session, socket). Only interfaces — not the component's internals.

## 3. Design tokens in `testUtils.ts`

Never inline raw colors or fonts in a test. Put them once in
`docs/stories/testUtils.ts` as `as const` objects named by meaning (`errorRed`,
`accentGreen`, `mutedText`) and import them. Write values the way the browser reports
them back: hex for opaque colors, `rgba(r, g, b, a)` with spaces for translucent ones.

## 4. Query by walking the data-testid tree, outer to inner

Never `screen.getByTestId('input')` directly. Resolve the wrapper, then each level with
`within`, mirroring the `.md` tree exactly:

```ts
export function cartView() { return within(screen.getByTestId('cartView')); }
export function getCartLines() { return cartView().getAllByTestId('cartLine'); }
export function getCartLine(index: number) { return within(getCartLines()[index]); }
export function getCartLineTitle(index: number) { return getCartLine(index).getByTestId('title'); }
export function getCheckout() { return cartView().getByTestId('checkout'); }
```

- One getter per node, in `shared/get/<component>.ts`, outer-to-inner order — including
  nodes only one story uses today.
- Prefix getters with their parent's name when the testid is generic (`getRoomNameInput`,
  not `getInput`) so getters from different components never collide.
- Call getters fresh in every assertion / `waitFor` — don't hold an element across an
  interaction that can re-render it.
- Absence: use `queryBy...` in the getter variant (`queryCheckout()`) and assert
  `.not.toBeInTheDocument()`.

## 5. Shared helpers rules

- `mocks.ts`: one `createMock<Dependency>(overrides = {})` per injected dependency,
  `vi.fn()` per method, `...overrides` last. For event sources, return the mock plus a
  way to emit (`{socket, getHandlers}` / `emit(event)`), and let the mock signal
  "connected" the way the real one does.
- `builders.ts`: one `build<Thing>(overrides = {})` per API response shape with
  realistic defaults and `...overrides` last; export `DEFAULT_*` ids that routes and
  assertions reuse. Ids must satisfy whatever validation the app applies (e.g. real
  GUIDs if the app parses them).
- `render.ts`: one `render<Component>(deps..., route = default)` per component under
  test. It mounts the providers **in the same order as the composition root**, plus the
  router at the right path so route params resolve, and imports the component by the
  exact public path other code uses (the folder's root component / `index` re-export).
  Leave a provider out on purpose only when the story says the component must work
  without it.
- First story in a domain may keep helpers local; the moment a second story needs one,
  move it to `shared/`. A helper needed by two domains moves to `docs/stories/`
  top-level `testUtils`.

## 6. Test file structure

```ts
import {describe, expect, it, vi} from 'vitest';
import {waitFor} from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import {DESIGN_COLORS} from '@docs/stories/testUtils';
import {createMockCartAPI, createMockRouter} from '@docs/stories/Cart/shared/mocks';
import {buildCartProduct} from '@docs/stories/Cart/shared/builders';
import {renderCartView} from '@docs/stories/Cart/shared/render';
import {getCheckout, getCartLineTitle} from '@docs/stories/Cart/shared/get/cartView';

describe('<Story title from the ## heading>', () => {
    it('WHEN <#### heading verbatim>', async () => {
        const user = userEvent.setup();
        // Arrange: mocks per this case's Prerequisites
        // Render
        // Wait for the initial state the WHEN starts from
        // Act like a user: user.type / user.click / emit an event
        // Assert what the user perceives
    });
});
```

- `describe` = the `##` title; one `it` per `#### WHEN`, title verbatim.
- Several things true for the same WHEN = several `expect`s in one `it`; don't fragment.
- Interact with `userEvent` (typing, clicking, keyboard), not `fireEvent`, unless the
  interaction can't be expressed otherwise.

## 7. What to assert — behavior, not implementation

- **Text**: exact copy from the `.md` via `toHaveTextContent`; a regex only when the
  design splits text across elements (animated dots).
- **State visuals**: `toHaveStyle({color: DESIGN_COLORS.errorRed})` for every state the
  design distinguishes (default, error, disabled, loading).
- **Enabled/disabled**: `toBeEnabled()` / `toBeDisabled()`.
- **Focus**: `toHaveFocus()` when the spec requires it.
- **Loading**: make the mock return a promise you resolve manually, assert the busy
  signal the spec defines (`aria-busy="true"`, disabled, label change), then resolve and
  assert the outcome.
- **Navigation**: `expect(router.push).toHaveBeenCalledWith('/cart')` inside `waitFor`
  when it follows async work; `not.toHaveBeenCalled()` for "SHOULD NOT navigate away".
- **Calls to dependencies**: only whether the API was / wasn't called and with which
  business arguments (`not.toHaveBeenCalled()` on cancel). Not how many renders, not
  call order between internals.
- **Realtime**: emit through the mock socket inside `act`/after render, then assert the
  item appeared / updated in place / disappeared; and that events for other ids changed
  nothing.

Avoid: snapshots, querying by class/tag/text-as-locator for structure, asserting props
or internal state, mocking child components or the project's own hooks/modules.

## 8. Framework notes

- **Next.js App Router**: async Server Components can't be rendered by Testing Library.
  Test the client component that receives their data (render it with built props and
  mocked client-side dependencies), and keep server-side data fetching covered by API
  stories (`/api:test`) or a few end-to-end tests. Mock `next/navigation`'s
  `useRouter`/`usePathname` through `createMockRouter` + `vi.mock('next/navigation')`
  in `render.ts` only (framework boundary), or better, inject a router interface via
  context if the app has one.
- **Browser storage / clock**: treat as dependencies — clear `localStorage` in
  `beforeEach`, use fake timers only where the spec involves time.

## 9. Run and check the failure mode

```
npx vitest run docs/stories/<Domain>/<story-slug>.test.ts
```

- Component not built yet: every test should fail on a missing testid
  (`Unable to find an element by: [data-testid="cartView"]`) or the missing import —
  that means wiring is right and it's waiting for the implementation.
- Component already exists (safety net before a refactor): all tests must pass before
  refactoring. A failure means the code and the `.md` disagree — report it and ask which
  is right; don't silently change either.
