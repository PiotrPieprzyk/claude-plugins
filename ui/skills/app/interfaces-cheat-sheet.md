# Project interfaces cheat sheet (what to put in `docs/frontend-interfaces.md`)

A standing, short reference so building a feature doesn't start with re-reading every
interface and context file. Create it in the project once; keep it under ~200 lines;
fix it when you notice drift. Sections:

## 1. Dependency hooks / providers

A table: hook (or prop) → what it returns → what happens with no provider (throws vs a
default no-op). No-op defaults matter: a component can silently do nothing in a story
that forgets the provider.

| Hook | Returns | No provider |
|---|---|---|
| `useCartAPI()` | `CartAPI` | throws |
| `useRouter()` | `Router` | throws |
| `useSession()` | `SessionId` | anonymous default |

## 2. API interfaces

The method signatures, request/response types, and which fields are raw strings vs value
objects.

## 3. Error-handling pattern

How failures come back (`Response | ApiError`, `{ok: false, error}`) and the standard
branch:

```ts
const res = await api.getThing(id);
if (res instanceof ApiError) { /* res.message, res.status */ } else { /* res.value */ }
```

## 4. Gotchas

Things that compile but are silently wrong, e.g.:

- Comparing a value object to a raw string (`userId === room.hostId` is always false; use
  `userId.value === room.hostId`).
- Prices in minor units vs display units.
- A provider that's a no-op outside production.

## 5. Router

The interface (`push`, `replace`), route format (hash vs path), and the list of routes.

## 6. Realtime / events

Handler types, subscribe vs scoped subscribe, the standard effect pattern with
unsubscribe, and the rule to ignore events for other ids.
