# Test harness per framework

`docs/stories/testUtils/http.ts` exposes one way to send requests as a given caller, so
story tests read the same whatever the framework. Pick the variant that matches the
project, then keep the exported names stable.

Every variant offers:

- `createClient()` — a client with its own cookie jar (one per caller identity).
- `client.get/post/put/patch/delete(path, body?, headers?)` → `{status, body, headers}`.

## A. Express (or any Node `http` handler) — supertest

```ts
// docs/stories/testUtils/http.ts
import supertest from 'supertest';
import {getApp} from '@/app';

const app = getApp(); // inject fakes here if the factory accepts them: getApp({repositories})

export function createClient() {
    return supertest.agent(app); // agent keeps cookies between requests
}
```

When events need a real port (WebSocket), start a server once per file:

```ts
const server = http.createServer(app);
new WebsocketServer(server);
await new Promise<void>((r) => server.listen(0, r));
const port = (server.address() as {port: number}).port;
const client = supertest.agent(server);
// afterAll: await new Promise<void>((r) => server.close(() => r()));
```

## B. Next.js App Router route handlers — call the handler

Route handlers take a Web `Request` and return a `Response`, so no server is needed.
Map paths to route modules explicitly; this file is the only place that knows where
handlers live, so moving a route means one edit here, not in every test.

```ts
// docs/stories/testUtils/http.ts
type Handler = (req: Request, ctx: {params: Promise<Record<string, string>>}) => Promise<Response>;
type RouteModule = Partial<Record<'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE', Handler>>;

// '[id]' segments become params. Order: most specific first.
const routes: [pattern: string, load: () => Promise<RouteModule>][] = [
    ['/api/cart', () => import('@/app/api/cart/route')],
    ['/api/checkout', () => import('@/app/api/checkout/route')],
    ['/api/orders/[id]', () => import('@/app/api/orders/[id]/route')],
];

function match(path: string) {
    const parts = path.split('?')[0].split('/');
    for (const [pattern, load] of routes) {
        const pp = pattern.split('/');
        if (pp.length !== parts.length) continue;
        const params: Record<string, string> = {};
        if (pp.every((p, i) => (p.startsWith('[') ? ((params[p.slice(1, -1)] = parts[i]), true) : p === parts[i]))) {
            return {load, params};
        }
    }
    return undefined;
}

export function createClient() {
    const jar = new Map<string, string>();
    async function send(method: string, path: string, body?: unknown, headers: Record<string, string> = {}) {
        const route = match(path);
        if (!route) return {status: 404, body: undefined, headers: new Headers()};
        const mod = await route.load();
        const handler = mod[method as keyof RouteModule];
        if (!handler) return {status: 405, body: undefined, headers: new Headers()};
        const cookie = [...jar].map(([k, v]) => `${k}=${v}`).join('; ');
        const req = new Request(`http://test.local${path}`, {
            method,
            headers: {
                ...(body !== undefined && typeof body !== 'string' ? {'content-type': 'application/json'} : {}),
                ...(cookie ? {cookie} : {}),
                ...headers,
            },
            body: body === undefined ? undefined : typeof body === 'string' ? body : JSON.stringify(body),
        });
        const res = await handler(req, {params: Promise.resolve(route.params)});
        for (const c of res.headers.getSetCookie()) {
            const [pair] = c.split(';');
            const [k, ...v] = pair.split('=');
            jar.set(k.trim(), v.join('='));
        }
        const text = await res.text();
        let parsed: unknown = text;
        try { parsed = text ? JSON.parse(text) : undefined; } catch { /* keep text */ }
        return {status: res.status, body: parsed as any, headers: res.headers};
    }
    return {
        get: (p: string, h?: Record<string, string>) => send('GET', p, undefined, h),
        post: (p: string, b?: unknown, h?: Record<string, string>) => send('POST', p, b, h),
        put: (p: string, b?: unknown, h?: Record<string, string>) => send('PUT', p, b, h),
        patch: (p: string, b?: unknown, h?: Record<string, string>) => send('PATCH', p, b, h),
        delete: (p: string, h?: Record<string, string>) => send('DELETE', p, undefined, h),
    };
}
```

Limits of calling handlers directly — switch to C (or `next-test-api-route-handler`)
for the stories that hit them:

- Handlers using `cookies()`/`headers()` from `next/headers`, `after()`,
  `revalidateTag()`/`revalidatePath()` need Next's request context and throw here.
  Either test them via C, or stub only those framework functions
  (`vi.mock('next/cache', ...)`) and assert the revalidation via the stub — that's a
  framework boundary, not your own code.
- Middleware/`proxy.ts`, rewrites, redirects and headers from `next.config` don't run.
- `server-only` throws outside an RSC build: alias it to an empty module in the test
  config.

## C. Any framework — a running server

```ts
const BASE = process.env.STORY_BASE_URL ?? 'http://localhost:3000';
export function createClient() {
    // same send()/cookie-jar as B, but: const res = await fetch(BASE + path, {method, headers, body, redirect: 'manual'})
}
```

Start the app once (`next build && next start`, `node dist/server.js`) in a global
setup, configured to use local fakes (env vars pointing to DynamoDB Local, an MSW/mock
server for providers). Slowest, but covers routing, middleware and config. Use it for a
few stories that need them, not as the default.

## Events / realtime

```ts
// docs/stories/testUtils/events.ts
import WebSocket from 'ws';

export function waitForEvent(ws: WebSocket, eventType: string, timeoutMs = 200) {
    return new Promise<{eventType: string; dto: any}>((resolve, reject) => {
        const timer = setTimeout(() => reject(new Error(`Timeout waiting for event: ${eventType}`)), timeoutMs);
        ws.on('message', (data) => {
            const msg = JSON.parse(data.toString());
            if (msg.eventType === eventType) { clearTimeout(timer); resolve(msg); }
        });
    });
}

export async function connectAndSubscribe(port: number, message: object) {
    const ws = new WebSocket(`ws://localhost:${port}/ws`);
    await new Promise<void>((resolve, reject) => { ws.on('open', resolve); ws.on('error', reject); });
    ws.send(JSON.stringify(message));
    return ws;
}
```

Usage:

```ts
const event = waitForEvent(ws, 'roomEdited');   // subscribe first
await host.put(`/rooms/${id}`, {name: 'Renamed'}); // then trigger
expect((await event).dto).toMatchObject({id, name: 'Renamed'});

// SHOULD NOT receive
const none = waitForEvent(ws, 'roomAdded', 150);
await other.post('/rooms', {name: 'Elsewhere'});
await expect(none).rejects.toThrow();
```

For outbound webhooks/emails instead of sockets, the fake of that system records what it
received; assert on its recorded messages the same way.

## Signed inbound webhooks

Compute the real signature in a helper with a test secret configured for the app, and
send the fixture body byte-for-byte as a string (re-serializing JSON can change the
signature):

```ts
const raw = await readFile(fixturePath, 'utf8');
await client.post('/api/payu/notify', raw, {'openpayu-signature': sign(raw, TEST_SECRET)});
```
