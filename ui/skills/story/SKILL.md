---
name: story
description: Write a UI story spec (docs/stories/<Domain>/<story-slug>.md) describing one user-visible frontend behavior — the data-testid tree, exact copy, states, navigation and WHEN/SHOULD test cases with mocked dependencies. Use when the user asks for a frontend/UI story, wants to specify a page, form or component behavior before building it, or wants existing UI behavior documented as a story before a refactor.
---

# Writing a UI story spec

A UI story is a short Markdown file that pins down **what a user sees and can do** on one
screen or component: the elements (as a `data-testid` tree), their copy, their states
(default, loading, disabled, error, empty), what happens on interaction, and where the
user ends up. It is the source of truth for the story test (`/ui:test`) and the
component (`/ui:app`). It never mentions component names' internals, hooks, state
variables or CSS classes.

## 1. Where it goes: grouped by domain

```
<package>/docs/stories/
  <Domain>/                 # business area: Cart, Checkout, Catalog, Menu, Room, Game
    <story-slug>.md         # this skill
    <story-slug>.test.ts    # /ui:test
    shared/                 # mocks, builders, render, get/ (see /ui:test)
  testUtils.ts              # design tokens shared by all domains
docs/design/                # design source (mockups, exported HTML, prompt), if any
```

- The folder is always `docs`, never `doc` or `documentations`.
- In a full-stack package (e.g. Next.js) where API stories share the same `docs/stories`,
  use `docs/stories/<Domain>/ui/<story-slug>.md` (API stories go in `<Domain>/api/`).
  Follow whatever layout exists; decide fresh only if `docs/stories/` doesn't exist.
- Domain = the business area, named as the business names it (`Cart`, not `Pages` or
  `Components`). A story about the cart badge in the header belongs to `Cart`.
- If a backend story covers the same capability, use the **same slug** in the same
  domain so the two can be read side by side, and link it: "see backend
  `<slug>` story".

## 2. Naming

- Slug: `<actor>-can-<do-something>` / `<actor>-can-see-<something>`:
  `customer-can-add-a-product-to-the-cart.md`, `customer-can-see-that-stock-is-updated.md`.
- Split interaction from destination when they're different screens: "user can click
  create room" (the button on the list) vs "user can create room" (the form).
- One story = one screen/component capability. Several stories may describe the same
  component from different angles; they then share one `data-testid` tree.

## 3. Before writing: gather

1. Existing stories in the domain — reuse the exact `data-testid` names and tree for
   components they already describe (a story never renames a testid another story uses).
2. The design source under `docs/design/` (or the screenshots/mockup the user gives):
   exact labels, button copy, error messages, states and their colors.
3. The matching API story or API contract — what the UI calls and what errors it can get.
4. If the screen already exists (documenting before a refactor): run or read it to
   capture current copy and behavior; describe behavior, not code.
5. Ask the user about anything the design doesn't settle (copy for an error, where to
   redirect) rather than inventing it.

## 4. Template

Copy exactly; keep heading levels — `/ui:test` relies on them.

````markdown
## <Actor> can <do something>.

### Status:

To do

### Description

Elements in <route, e.g. /cart or #/rooms/{roomId}>:

- data-testId="<wrapper>" < as wrapper
    - data-testId="<field>"
        - data-testId="input" — <initial value / placeholder if it matters>
        - data-testId="errorMessage"
    - data-testId="<item>" < repeated per <thing>
        - data-testId="<itemTitle>"
    - data-testId="<primaryAction>"
    - data-testId="<secondaryAction>"

<Behavior rules, one per line or short paragraph:>

<primaryAction> is not clickable when <conditions>.

When <primaryAction> is clicked should change state to loading, then redirect to <route>.

When <secondaryAction> is clicked should redirect to <route> without <side effect>.

Error messages:

- <EXACT ERROR COPY> — <when it shows>

Empty / loading states:

- <EXACT COPY> — <when>

### Tests

Test cycle prerequisites:

- Mock Router
- Render <Component/screen> at <route>

#### WHEN <user action or arriving state> SHOULD <what the user sees / where they end up>

Prerequisites:

- Mock API <METHOD> <path> — <return what | return error: message, status | do not resolve before assertions>
- Emit <event> with <data>

#### WHEN ...
````

`Status` is `To do`, `In progress` or `Done` (`Done` for stories written to document
existing behavior before a refactor).

## 5. The data-testid tree

- List elements as a nested tree, outer to inner, exactly as they nest in the DOM. Tests
  query by walking this tree (`within(wrapper).getByTestId(child)`), so the nesting is
  part of the contract.
- Each story's tree starts from a wrapper testid named after the component/region
  (`cartView`, `roomRenameForm`).
- Generic child names (`input`, `errorMessage`, `title`) are fine *inside* a named
  parent; the parent gives them meaning.
- Mark repetition (`< repeated per product`) and the wrapper (`< as wrapper`).
- Only give testids to things the user perceives or interacts with and that a test
  needs. Not to layout divs.
- When another story already defines part of the tree, reference it instead of
  repeating: `- data-testId="roomListItem" < repeated per room, see [[user-can-see-list-of-rooms]]`.

## 6. Writing the WHEN cases

Each `#### WHEN ... SHOULD ...` becomes one test, verbatim. Cover whichever apply:

1. **Initial render** — what's shown once data loads (prefilled values, list items, counts).
2. **Loading state** — what the user sees while a request is pending.
3. **Valid interaction** — typing/clicking that enables an action, the busy state while it
   runs, and the outcome (navigation, updated content).
4. **Client-side validation** — each rule at its boundary, with the exact error copy and
   the action disabled.
5. **Server errors** — each error the API can return that the UI shows differently (taken
   name, sold out, unauthorized), with exact copy, and "SHOULD NOT navigate away".
6. **Empty state** and **failed load**.
7. **Cancel / back** — navigation without the side effect.
8. **Realtime updates** — an event arrives → the matching item appears / updates in
   place / disappears; events for other items are ignored.
9. **Permissions** — elements that only some users see (host vs player, logged in vs
   anonymous).

Rules:

- Outcomes are things a user perceives: text, enabled/disabled, visible/hidden, focus,
  busy state, the route they land on, the color of a state when the design defines it.
  Never "SHOULD set state" or "SHOULD call useX".
- Quote copy exactly as it will be asserted (`ERR: ROOM_NAME_TOO_SHORT — (MIN 3 CHARS)`).
- Prerequisites name every mocked dependency the case needs and what it returns,
  including "do not resolve before assertions" for loading states. These are the only
  things a test may fake: the API, router, session, realtime socket, clock, storage.

## 7. Finish

- Check every rule in Description has a WHEN and every WHEN's assertion targets an
  element in the tree.
- Tell the user the path; offer `/ui:test` for the test and `/ui:app` for the
  component.
