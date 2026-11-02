---
name: test
description: Write a black-box API story test (docs/stories/<Domain>/<story-slug>.test.ts) from its story spec .md — one test per WHEN heading, driving the API only through HTTP/events and faking external services at the boundary. Use when the user asks for backend/API story tests, wants tests for an endpoint, webhook or realtime event, or wants behavior-level safety tests before refactoring backend code.
---

# Writing API story tests

These tests are black-box specs of the API's behavior. They talk to the system the way a
client does — HTTP requests and event subscriptions — and assert on what a client can
observe: status, response body, and state visible through other requests or events.
They must survive any refactor that keeps the behavior, so they never import or mock
the project's internal modules.

It is expected and correct for a test to fail until the behavior exists. Do not weaken
assertions to make it pass — the test is the spec.

See `harness.md` in this skill folder for client setups per framework (Express +
supertest, Next.js route handlers, running server) and event helpers.

## 1. Layout: grouped by domain

```
<package>/docs/stories/
  testUtils/                    # cross-domain: HTTP client, event helpers, fakes of external services
    http.ts
    events.ts
    fakes/                      # in-memory replacements for external systems (payment, CMS, email, DB)
  <Domain>/
    <story-slug>.md             # the spec (/back:story)
    <story-slug>.test.ts        # this skill
    shared/                     # per-domain helpers, once a 2nd story in the domain needs them
      setup.ts                  # e.g. createCustomer(), createRoomWithPlayers(n)
      builders.ts               # request-body / fixture builders with defaults + overrides
      fixtures/                 # recorded payloads (webhook bodies, provider responses)
```

- Folder is always `docs`. In a full-stack package the domain folder may have `api/` and
  `ui/` subfolders (`docs/stories/<Domain>/api/<slug>.test.ts`); follow what exists.
- The test sits next to its `.md` with the same slug. One `.md` ↔ one `.test.ts`.
- Make sure the test runner includes `docs/stories/**/*.test.ts` and that an import alias
  for the folder exists (e.g. `@docs/*` → `docs/*`) so tests never use `../../` imports.
  Add both if missing and say so.

## 2. Read before writing

1. The story `.md` — Description (rules), `Test cycle prerequisites`, each
   `#### WHEN ... SHOULD ...` with its `Prerequisites`, and `Required API` types.
2. Other tests in the same domain folder and `shared/` — reuse their setup helpers.
3. `docs/stories/testUtils/` — the HTTP client and fakes. If it doesn't exist, create it
   from `harness.md` for this project's framework.
4. Only to find the entry point (app factory, route handler module paths, how external
   dependencies get injected): the composition root. Do not read handler internals to
   decide what to assert — assert what the `.md` says.

## 3. File structure

```ts
import {describe, it, expect, beforeAll, afterAll} from 'vitest'; // or the project's runner
import {createClient} from '@docs/stories/testUtils/http';
import {createCustomer} from '@docs/stories/Cart/shared/setup';

describe('<Story title from the ## heading, verbatim>', () => {
    // Test cycle prerequisites → beforeAll, created through the API
    beforeAll(async () => { ... });
    // Clean up what the story created → afterAll
    afterAll(async () => { ... });

    it('WHEN <heading text verbatim> SHOULD <...>', async () => {
        // Prerequisites of this case — arranged through the API
        // Act — exactly one request (or event) that the WHEN describes
        // Assert — status, body, then observable side effects
    });
});
```

- `describe` title = the story's `##` heading. `it` title = the `####` heading verbatim
  (without the `#### `). One `it` per WHEN; never split one WHEN into several tests and
  never test something that has no WHEN (add the WHEN to the `.md` first).
- If two things must hold for the same WHEN, they are multiple `expect`s in one `it`.

## 4. Arrange through the public API

- Create prerequisite state by calling the API (create the user, create the room, add
  to cart), not by writing to the database, a repository or a store directly. This keeps
  setup valid across refactors of storage.
- Exception: state owned by an **external system** the API reads from (CMS content,
  payment provider responses, stock in a store with no public write endpoint) — set it
  on the fake of that system (section 6).
- Each caller identity gets its own client with its own cookie jar (one agent per user),
  so "host" and "other player" are different clients, not a header you flip.
- Make data unique per test (unique names, fresh ids) so tests don't depend on order and
  can run in parallel with other files.

## 5. Assert behavior

Assert, in this order:

1. **Status** — `expect(res.status).toBe(400)`.
2. **Body** — the fields the WHEN names, exact values where the `.md` gives them
   (`toBe`, `toMatchObject`). Don't assert on fields the story doesn't specify, and don't
   snapshot whole bodies (generated ids/dates make them brittle and they over-specify).
3. **Observable side effects** — read back through the API:
   - success: `GET` the resource and check it changed;
   - rejection: check **nothing changed** — compare a count or the resource before/after
     (`countBefore === countAfter`, name still the old one). A 400 that silently wrote
     data is a bug these tests must catch.
4. **Effects on external systems** — what the fake received: email sent to whom, payment
   created with which amount. Assert the message content that matters to the business,
   not the SDK call shape.
5. **Events** — subscribe **before** triggering, then await the event with a timeout
   (`waitForEvent(client, 'roomEdited')`) and check type + key `dto` fields. For "SHOULD
   NOT receive", await with a short timeout and expect it to reject.

Avoid: asserting on internal function calls, `vi.mock` of project modules, spying on
repositories, checking log output, snapshot tests.

## 6. Fake external systems at the boundary, nothing else

- What may be faked: things outside the codebase — databases/key-value stores, payment
  providers, CMS, email/SMS, captcha, clock, randomness. Everything inside the project
  runs for real.
- Prefer, in this order:
  1. **Injected in-memory implementation** of the port the app already has
     (`getApp({repositories})`, a store parameter, an environment-selected adapter) — with
     the same semantics as the real thing (uniqueness, conditional writes, transactions).
  2. **Network-level interception** for HTTP services (MSW, `nock`) when the app talks to
     them over HTTP and has no injection point — the app's own client code still runs.
  3. **A local real instance** (DynamoDB Local, Postgres in Docker) when semantics matter
     more than speed; reset it between test files.
  4. `vi.mock` of a third-party SDK module — last resort, and only the SDK, never your
     own module that wraps it.
- Put fakes in `docs/stories/testUtils/fakes/` so every domain reuses them. If the app
  has no seam to inject a fake, say so and propose the smallest seam (a parameter with a
  production default) rather than mocking internals.
- Recorded external payloads (webhook bodies, provider responses) go in
  `<Domain>/shared/fixtures/` as JSON and are sent as real request bodies, with real
  signatures computed by a test helper using a test secret.

## 7. Isolation and cleanup

- `afterAll` removes what the story created through the API (or resets the fake/store).
  A test that creates something mid-test cleans it up at its end.
- Never depend on another test having run first. Each `it` arranges its own case-level
  prerequisites.
- Close servers, sockets and subscriptions in `afterAll` so the runner exits.

## 8. Shared helpers: when to extract

- First story in a domain: helpers may live in the test file.
- As soon as a second story in the same domain needs the same setup/builder, move it to
  `<Domain>/shared/` and import it from both. As soon as two domains need it, move it to
  `testUtils/`.
- Helpers only call the public API or the fakes — they're arrangement shortcuts, not a
  back door.

## 9. Run and check the failure mode

```
npx vitest run docs/stories/<Domain>/<story-slug>.test.ts
```

- Behavior not implemented yet: tests should fail on the **assertion** (e.g. 404/405 for
  a missing route, wrong status), not on a TypeScript error, import error or a crash in
  setup. Fix wiring problems; leave behavior failures.
- Behavior already implemented (safety net before a refactor): all tests must pass
  before the refactor starts. If one fails, the code and the `.md` disagree — report it
  to the user and ask which one is right; don't silently change either.
- Update the story's `### Status:` only when the user asks or the implementation is done.
