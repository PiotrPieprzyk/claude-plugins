---
name: story
description: Write an API story spec (docs/stories/<Domain>/<story-slug>.md) describing one backend behavior as WHEN/SHOULD test cases plus the HTTP or event contract it needs. Use when the user asks for a backend/API story, wants to specify an endpoint, webhook or realtime event before implementing it, or wants existing endpoint behavior documented as a story before a refactor.
---

# Writing an API story spec

A story is a short Markdown file that pins down **what an API does, as seen by a client**:
which requests succeed, which get rejected and with what status, what state changes are
visible afterwards, and which events get broadcast. It is the source of truth for the
matching `.test.ts` (written with `/api:test`) and for the implementation. It never
mentions classes, tables, repositories or other internals.

## 1. Where it goes: grouped by domain

```
<package>/docs/stories/
  <Domain>/                     # e.g. Cart, Checkout, Payment, Catalog, Room, Game
    <story-slug>.md             # this skill
    <story-slug>.test.ts        # /api:test
    shared/                     # per-domain test helpers (see /api:test)
  testUtils/                    # cross-domain test helpers (HTTP client, event helpers)
```

- The folder is always `docs`, never `documentations` or `doc`.
- `<package>` is the package that serves the API (the one whose test runner runs these
  tests). In a full-stack package (e.g. Next.js, where route handlers and pages live
  together), put API stories one level deeper so UI stories for the same domain sit next
  to them: `docs/stories/<Domain>/api/<story-slug>.md` (and UI stories in
  `docs/stories/<Domain>/ui/`). Detect which layout the project already uses and follow it;
  only decide fresh when `docs/stories/` doesn't exist yet.
- **Domain** = the business area the behavior belongs to, named as the business names
  it (PascalCase, singular or the established name): `Cart`, `Checkout`, `Payment`,
  `Inventory`, `Catalog`. Not a technical layer (`Webhooks`, `Api`, `Lambda`) and not an
  endpoint path. A webhook that updates stock belongs to `Inventory`; a payment
  notification belongs to `Payment`. If unsure, list the existing domain folders and ask.

## 2. Naming the story

- File slug: `<actor>-can-<do-something>` or `<thing>-is-<outcome>/`, kebab-case:
  `customer-can-place-an-order.md`, `stock-is-released-when-payment-fails.md`.
- The actor is a business role (customer, host, player, admin, payment provider,
  CMS), never "the API" or "the handler".
- One story = one capability. If the WHEN list mixes two unrelated capabilities, split
  into two stories in the same domain folder.

## 3. Before writing: gather the contract

Read, in this order, and stop as soon as you have enough:

1. Existing stories in the same domain folder — reuse their wording, prerequisites and
   type names so the folder reads as one spec.
2. If the endpoint already exists (documenting before a refactor): the route handler,
   its validation schema and its error mapping, to capture **current observable
   behavior** (statuses, response shapes, side effects). Describe behavior, not code.
3. API docs if the project has them (OpenAPI under `docs/openapi/`, README sections).
4. For anything still ambiguous (limits, who's allowed, what happens on conflict), ask
   the user one compact question rather than inventing a rule.

## 4. Template

Copy exactly; keep heading levels — `/api:test` relies on them.

````markdown
## <Actor> can <do something>.

### Status:

To do

### Description

<2-6 sentences of business rules, written for a client developer: who may do it, input
limits, what changes, what is broadcast, what is explicitly NOT done. State rules once
here; the WHEN cases below exercise them.>

### Tests

Test cycle prerequisites:
- <state created once for the whole story, through the public API — e.g. "A customer
  session exists (created via POST /session)">

#### WHEN <trigger, in client terms> SHOULD <observable outcome: status + body / visible state change / event>

Prerequisites:
- <state this case needs on top of the cycle prerequisites, or "None">

#### WHEN ...

### Required API

- <METHOD> <path>
- <event channel / webhook path if any>

```typescript
// Request body
// <how the caller is identified: cookie, header, signature — not a body field unless it is one>
type <Action>Request = {
  field: string; // constraints, e.g. 3–50 characters
};

// Response body — 200
type <Action>Response = {
  ...
};

// Error body — 4xx (only if the shape is part of the contract)
type ErrorResponse = { error: string };
```
````

`Status` is one of `To do`, `In progress`, `Done`. New stories start as `To do`; a story
written for already-working behavior (before a refactor) starts as `Done`.

## 5. Writing the WHEN cases

Each `#### WHEN ... SHOULD ...` becomes one test, verbatim, so make each one precise
enough to assert on and independent of the others.

Cover, in this order, whichever apply:

1. **Happy path** — the main success, with the exact status and the response fields that
   matter.
2. **Validation boundaries** — one case per rule, at the boundary (`shorter than 3`,
   `longer than 50`, `missing`, `wrong type`). Several boundaries of the *same* rule may
   share one case ("shorter than 3 characters, longer than 50 characters").
3. **Authorization / ownership** — not logged in, wrong user, wrong role, bad signature.
4. **Not found** — the target id doesn't exist → 404.
5. **State conflicts** — the action is not allowed in the current state (already done,
   sold out, game ended, duplicate) → the status the API uses for it.
6. **Side effects** — what another request can observe afterwards (a GET returns the new
   state, a count changed, an email/event was emitted).
7. **Events** — for realtime/webhook flows: which event a subscriber receives, its type
   and key payload fields; and what a subscriber must **not** receive (scoping).
8. **Idempotency / concurrency** when it's part of the contract (replayed webhook,
   parallel checkouts over the last item).

Rules:

- Write the outcome in client terms: `SHOULD return 400`, `SHOULD return 200 with the
  updated name`, `SHOULD receive a RoomEditedEvent with ...`. Never `SHOULD call X` or
  `SHOULD save to the table`.
- For a rejection, the spec implies "and nothing changed". Write that explicitly in the
  Description once (e.g. "rejected requests change nothing") so the test checks it.
- Name concrete values when they matter for the assertion (`a name of 51 characters`,
  `quantity 0`), not when any value works.
- Prerequisites are states, not steps: "Room exists with 2 players", "Product has 1 piece
  left". The test decides how to reach that state through the API.
- Things outside the system (payment provider, CMS, email service, clock) are described
  as what they send or return: "Payment provider sends a COMPLETED notification",
  "CMS returns product P with price 100". The test fakes them at the boundary.

## 6. Required API section

- List every endpoint/event the story touches, including ones only used to set up or
  verify (mark those `(setup)` / `(verification)` if they belong to another story).
- TypeScript types describe the wire format (JSON), with constraints as comments. Use
  `string` for ids, ISO strings for dates, string literal unions or enums for statuses.
- For events: one type per event with its discriminator (`eventType: 'roomEdited'`) and
  `dto`.
- Say how the caller is identified (cookie, header, signature) in a comment.

## 7. Finish

- Re-read the WHEN list against the Description: every rule has at least one case; no
  case tests something the Description doesn't state.
- Tell the user the file path and offer to write the test with `/api:test`.
