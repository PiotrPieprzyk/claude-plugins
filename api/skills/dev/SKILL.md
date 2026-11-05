---
name: dev
description: Write or refactor backend TypeScript code in the domain / application / infra style — entities with explicit readonly value-object fields, value objects with a private constructor and static create(), collections as value objects, repositories typed by entities and ID value objects, and persistence mappers in infra. Use when implementing a backend story (docs/stories, /api:story, /api:test), adding an entity, value object, repository/store or persistence adapter, or refactoring backend code that passes primitives, plain snapshots or declaration-merged classes around.
---

# Writing backend code

Backend code is written so the **domain** says what the business means, in types that
can't hold an invalid value, and everything else (HTTP, storage, providers) converts to
and from it at the edges. The story tests (`/api:test`) define the behavior; this skill
defines the shape of the code that makes them pass.

Before writing, read one existing entity, one value object, its repository interface and
its persistence mapper in the project, and follow their naming, file layout and error
classes. This skill sets the pattern; the project sets the spelling.

## 1. Layers and the dependency rule

```
<module or aggregate>/
  domain/          entities, value objects, repository (store) interfaces, domain errors
  application/     use cases / routers: parse input → call domain → persist → build output
  infra/           repository implementations, persistence mappers, provider adapters
```

- `domain/` imports only other `domain/` code (and pure shared code). Never application,
  infra, frameworks, SDKs or `*Persistence` types.
- `application/` imports domain. It gets infra through injected interfaces (constructor
  or `deps` parameter), wired in the composition root.
- `infra/` imports domain and application.
- If the project has a lint rule for this (e.g. `import/no-restricted-paths`), keep it
  passing; if it doesn't, don't add one unasked.
- Shared domain building blocks (the `ValueObject` base) go in a shared `domain/` folder,
  not in one module.

## 2. Value objects

A value object wraps one value, validates it once in `create`, and is immutable. Anything
that has a rule or a meaning beyond its primitive type becomes one: IDs, money, quantities,
emails, phone numbers, names with length limits, positions, codes with a pattern.

```ts
// shared/domain/value-object.ts
export abstract class ValueObject<T> {
  readonly value: T;

  protected constructor(value: T) {
    this.value = value;
  }

  exact(other: ValueObject<T> | undefined): boolean {
    return other !== undefined && this.value === other.value;
  }
}
```

```ts
export class Money extends ValueObject<number> {
  private constructor(value: number) {
    super(value);
  }

  static create(grosze: number): Money {
    if (!Number.isInteger(grosze) || grosze < 0) {
      throw new InvariantError(`${grosze} isn't an amount in grosze (an integer >= 0)`);
    }
    return new Money(grosze);
  }

  static zero(): Money {
    return new Money(0);
  }

  add(other: Money): Money {
    return new Money(this.value + other.value);
  }

  times(quantity: Quantity): Money {
    return new Money(this.value * quantity.value);
  }
}
```

Rules:

- **Private constructor, `static create(raw)`** that validates and is the only way in.
  Internal operations whose result is valid by construction (`add`, `zero`) may call the
  constructor directly.
- **Every ID is a value object** (`OrderId`, `ProductId`, `PaymentId`, `UserId`), even
  when it only checks "not empty". Validate the real format only when the format is a
  rule (a GUID the system generates, a pattern the provider requires), not when tests or
  external systems use other shapes.
- A pattern the request validation also needs lives on the value object
  (`static readonly PATTERN = /…/`) and the request schema reuses it, so the two can't
  drift.
- Behavior that belongs to the value goes on it (`add`, `times`, `isGreaterThan`,
  `isZero`), instead of `.value` arithmetic spread around the callers.
- Compare with `exact`, never `===` on two instances.
- Error messages name the field and the broken rule. Leave personal data (email,
  phone, name) **out** of the message: it ends up in logs.

## 3. Collections

A list with rules (non-empty, max size, unique members, totals) is its own class holding
`readonly values`, not a bare array on the entity.

```ts
export class OrderLines {
  readonly values: readonly OrderLine[];

  private constructor(values: readonly OrderLine[]) {
    this.values = values;
  }

  static create(raw: OrderLineRaw[]): OrderLines {
    return OrderLines.of(raw.map((line) => OrderLine.create(line)));
  }

  /** From lines that are already value objects (e.g. after a domain operation changed them). */
  static of(lines: readonly OrderLine[]): OrderLines {
    if (lines.length === 0) throw new InvariantError("no lines");
    if (new Set(lines.map((l) => l.productId.value)).size !== lines.length) throw new InvariantError("duplicate products");
    return new OrderLines(lines);
  }

  total(): Money {
    return this.values.reduce((sum, line) => sum.add(line.total()), Money.zero());
  }
}
```

- A collection member with several fields (`OrderLine`) is a value object class too: its
  fields are value objects, its constructor checks the rules between them (e.g. oversold
  ≤ qty), and it has `create(raw)` plus `with…` methods that return a new instance.
- Changing a collection returns a new one (`add`, `remove`), validated again.

## 4. Entities

```ts
type OrderProps = { orderId: OrderId; status: OrderStatus; lines: OrderLines; amount: Money; buyerEmail?: Email; … };

export class Order {
  readonly orderId: OrderId;
  readonly status: OrderStatus;
  readonly lines: OrderLines;
  /** Always Σ unitPrice × qty + shipping fee. */
  readonly amount: Money;
  readonly buyerEmail: Email | undefined;
  …

  private constructor(props: OrderProps) {
    // Only the rules that span several fields; single-field rules live in the value objects.
    const expected = props.lines.total().add(props.shipping?.fee ?? Money.zero());
    if (!props.amount.exact(expected)) throw new InvariantError(`Order ${props.orderId.value}: amount …`);
    this.orderId = props.orderId;
    this.status = props.status;
    …
    Object.freeze(this);
  }

  /** Raw data (request, storage) → entity. Throws when the data breaks a rule. */
  static create(raw: OrderPropsRaw): Order {
    return new Order({
      orderId: OrderId.create(raw.orderId),
      lines: OrderLines.create(raw.lines),
      amount: Money.create(raw.amount),
      buyerEmail: raw.buyerEmail ? Email.create(raw.buyerEmail) : undefined,
      …
    });
  }

  hostRenames(hostId: UserId, newName: RoomName): Room { … }       // a business operation
  with(changes: OrderChanges): Order { return new Order({ ...this, ...changes }); }
}
```

- **Fields are declared explicitly** on the class, `readonly`, typed by value objects,
  optional ones as `T | undefined`, and assigned one by one in the constructor so the
  compiler checks every field is set.
- **Private constructor** taking `<Entity>Props` (value-object types, not exported);
  **`static create(raw: <Entity>PropsRaw)`** builds the value objects. Add named
  factories for business starts (`Order.reserve(...)`, `Room.createForHost(...)`) that
  compute derived fields and end in `create`.
- **Immutable**: every operation returns a new instance (`new Entity({ ...this, field })`),
  or `undefined` to mean "the aggregate should be deleted". `Object.freeze(this)` when the
  project already does.
- **Methods take value objects**, not primitives, for any parameter that has one
  (`playerMarksCell(userId: UserId, position: Position)`). Only the raw factories take
  primitives.
- **Pure**: no repository fields, no I/O, no clock reads (pass `now` in). Rules that need
  other aggregates (does the user exist, does the host already have a room) are looked up
  by the application layer and passed in as arguments (`createForHost(raw, hostAlreadyHasRoom)`).
- Business words for methods (`userJoins`, `hostRemovesPlayer`, `stepFor(target)`), not
  setters.

## 5. Raw types

- `<Entity>PropsRaw` (domain) is the plain-data input of `create`: IDs as strings, money
  as integers, nested raws (`OrderLineRaw[]`). Derive input types from it with
  `Pick`/`Partial` (`NewOrder = Pick<OrderPropsRaw, "orderId" | "lines"> & { now: number }`).
- Change sets that the domain applies (`OrderChanges`) are typed by **props** (value
  objects), since they come out of domain operations.
- The entity has **no** `toSnapshot()` / `toJSON()` / `toPersistence()`: turning it into
  storage or API shapes is the mapper's job (sections 7, 8).

## 6. Repository / store interface

```ts
// domain/<aggregate>-repository.ts (or the project's store.ts)
export interface RoomRepository {
  save(room: Room): Promise<void>;
  find(roomId: RoomId): Promise<Room | undefined>;
  findRoomByHostId(hostId: UserId): Promise<Room | undefined>;
  delete(roomId: RoomId): Promise<void>;
}
```

- Lives in `domain/`. Takes and returns **entities** and **ID value objects**, never
  `*Persistence` types, rows or bare string IDs.
- Implementations (database, in-memory for tests) live in `infra/` (or the test folder)
  and both go through the same persistence mapper, so the in-memory one stores exactly
  what the real one would.
- Conditional / transactional writes (optimistic locking, "only if status is X") stay
  expressible: a write type carries the entity or a typed change set plus the condition.

## 7. Persistence mapper (infra)

```ts
// infra/<entity>-persistence-map.ts
export type OrderPersistence = OrderPropsRaw; // or its own type when the stored shape differs (rows, column names)

export class OrderPersistenceMap {
  static toPersistence(order: Order): OrderPersistence {
    return {
      orderId: order.orderId.value,
      lines: order.lines.values.map((l) => ({ productId: l.productId.value, qty: l.qty.value, … })),
      amount: order.amount.value,
      buyerEmail: order.buyerEmail?.value,
      …
    };
  }

  /** Re-validates stored data: a corrupt record fails loudly instead of spreading. */
  static toDomain(stored: OrderPersistence): Order {
    return Order.create(stored);
  }

  static changesToPersistence(changes: OrderChanges): Partial<OrderPersistence> { … }

  // Only when the table differs from the persistence shape (snake_case, join tables):
  static fromRow(row: OrderRow): OrderPersistence { … }
  static toRow(stored: OrderPersistence): OrderRowInsert { … }
}
```

- Write every field out by hand. No `Object.assign(this, data)`, no `{ ...entity }` into
  storage, no `structuredClone` of entities: it drops class prototypes (value objects
  come back as plain objects) or writes instances where primitives belong.
- Legacy-data fixes (renamed fields, missing defaults) happen here or in the store's read
  function, before `toDomain`, never in the entity.
- API responses get their own mapper in `application/` (`RoomMap.toDTO(room, …)`), so
  storage and API shapes can change independently.

## 8. Application layer

- The application layer (router, use case, handler) is the **boundary**: it receives raw
  input (HTTP body, webhook, provider response), validates its *shape* with the request
  schema (zod etc.), then creates value objects (`OrderId.create(body.orderId)`) and
  calls the domain.
- Order of a handler: parse → look up what cross-aggregate rules need → call the domain
  → persist through the repository → map to the response DTO. No business `if`s inline;
  if you're writing a rule check, it belongs in an entity or value object.
- Functions it exports may keep taking primitives (they're called by routes and other
  modules); they convert at the top. Values going out to providers, logs and responses
  use `.value`.
- Dependencies (repositories, gateways, `now`, `log`) are injected, typed by interfaces.

### Page loaders (full-stack frameworks: Next.js App Router, Remix, SvelteKit…)

A server-rendered page is an entry point like a route handler, so its logic belongs in the
module, not in the page file. For each page, the module exports one **page loader** that
returns exactly the props of the client component that renders the page:

```ts
// src/server/modules/catalog/application/pages.ts, wired in the module's index.server.ts
loadShopPage(query): Promise<ShopPageProps>
loadProductPage(slug): Promise<ProductPageProps | null> // null = not found
```

- The props type lives in the shared contracts file, next to the HTTP request/response
  types (e.g. `src/kernel/contracts.ts`), and the client component types its props with
  it. The loader and the component then can't drift apart without a compile error.
- The loader does everything between the module's use cases and the screen: mapping to
  plain props (prices in minor units, IDs, `null` for "missing"), limits (at most 3
  images), fallbacks (a missing title), and **turning expected failures into values**
  (`{status: "error"}`; a streamed promise resolves to `{ok: false}` instead of
  rejecting). No display formatting (`"179,00 zł"`, labels, copy): that's the UI's job.
- The page file is composition only, a few lines, with nothing worth testing:

  ```tsx
  export default async function ProductRoute({ params }) {
    const page = await loadProductPage((await params).slug);
    if (!page) notFound();
    return <ProductPage {...page} />;
  }
  ```

  A `try`/`catch`, a `.map`, a fallback or an `if` on data in a page file is logic the
  tests can't reach: move it into the loader. Only framework calls (`connection()`,
  `notFound()`, `redirect()`) stay in the page.
- Use cases that only the pages used stop being exported from the module; the loaders
  are the pages' entry points.

## 9. Errors

- Domain and value objects throw the project's domain error classes (`InvariantError`,
  `ValidationError`, `ForbiddenError`, …), never HTTP errors and never a bare `Error`.
- Only the application/HTTP edge maps them to status codes, in one place.
- Request-shape problems (missing field, wrong type) are rejected by the request schema
  before the domain is called, with user-facing messages. Value objects still validate:
  they also guard stored data and internal callers.

## 10. What may stay primitive

Keep a field primitive when it has no rule and no behavior worth naming: a status union
handled by the entity's own state machine, ISO timestamps that are only stored and shown,
free display text (titles, addresses), provider names. Don't wrap for the sake of it; do
wrap as soon as a second place checks or computes with the value.

## 11. Tests

- Value objects: one small test file per folder (`value-objects.test.ts`): `create`
  accepts the valid edge, rejects each broken rule, operations compute right, `exact`.
- Entities: build from a raw fixture with `Entity.create(RAW)`; assert on fields via
  `.value` (`expect(order.amount.value).toBe(3299)`), not `toMatchObject` against
  primitives. Cover cross-field rules and that operations return new instances.
- Store/mapper tests: compare entities with `toEqual(Entity.create(raw))`, and stored data
  with the raw/persistence object.
- Story tests (`/api:test`) don't change: they see only HTTP/events, page loader props and
  the stored plain data, which the mapper keeps the same.
- Keep rules out of places the story tests can't reach or replace with a fake: page files
  (→ page loaders, section 8) and queries sent to an external system. A filter in a
  GROQ/SQL query is invisible to a story test whose fake answers that query. When a rule
  has to live in a query, cover it with an adapter test against the real system (local
  instance) or a recorded response, and say in the story that the query decides it.

## 12. Refactoring existing code to this style

1. Run the tests, type check and lint first; note pre-existing failures.
2. Add the `ValueObject` base, then the value objects, collections first if the entity has
   lists.
3. Rewrite the entity: explicit fields, `Props`/`PropsRaw`, `create`, operations. Delete
   declaration merging (`interface X extends Readonly<Snapshot>`), `Object.assign(this, …)`,
   `toSnapshot()`.
4. Add the persistence mapper; switch the real and the in-memory store to it; change the
   repository interface to entities and ID value objects.
5. Convert at the application boundary; replace primitive arithmetic on entity fields with
   value-object methods.
6. Let the type checker list every remaining call site; fix them, then the tests (raw
   fixtures + `create`, `.value` assertions).
7. Same checks again. Report behavior that became stricter (e.g. stored data now
   validated on read) so the user can judge it.
