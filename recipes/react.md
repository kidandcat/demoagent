# Recipe: React / Vue / JS SPA → browser demo

Target: a JS/TS single-page app (React, Vue, Svelte...) talking to a REST/GraphQL backend.
Produce `demo/`: a build of the **same source** with a mock network layer and a browser DB.

## Seam options (pick the first that applies)

1. **Central API module** (`src/api/*.ts`, an axios instance, a fetch wrapper): create
   `demo/src/fakeApi.ts` implementing the same exported surface and alias it in at build time
   (Vite `resolve.alias` / webpack alias / tsconfig paths). Type-safe, no interception.
2. **MSW (Mock Service Worker)**: intercept at the network level with handlers for every
   route. Works with any codebase, survives scattered `fetch` calls, and the network tab shows
   the mocked requests — great for demo credibility. Prefer this when calls are not
   centralized.
3. **GraphQL**: mock the client's link layer (Apollo `SchemaLink` with mock resolvers against
   the real schema) — never spin a server.

## Layout

```
demo/
  package.json          # private pkg; deps: the real frontend (workspace/file:), msw or none
  vite.config.ts        # root points at the real frontend source; alias/env for demo mode
  src/
    main.tsx            # demo entry: start MSW / inject fakeApi, then mount the real <App/>
    fake_backend/
      handlers.ts       # route handlers (msw) or fakeApi implementation
      db.ts             # Dexie (IndexedDB) or localStorage JSON store
      seed.ts           # versioned seed dataset
  README.md
  API_COVERAGE.md
```

Depend on the real app via `file:../app` (or a workspace) and import its root component —
do not copy `src/` into `demo/`.

## Key pieces

**Browser DB** — Dexie for anything non-trivial:

```ts
import Dexie from 'dexie';
export const db = new Dexie('product_demo');
db.version(SEED_VERSION).stores({ users: 'id', invoices: 'id, userId', ... });
// on boot: if (await db.users.count() === 0 || storedSeedVersion !== SEED_VERSION) await reseed();
```

**MSW boot** (option 2):

```ts
import { setupWorker } from 'msw/browser';
import { handlers } from './fake_backend/handlers';
await setupWorker(...handlers).start({ onUnhandledRequest: 'error' }); // loud on drift
```

`onUnhandledRequest: 'error'` is the drift alarm — any route the frontend adds later fails
visibly instead of hitting production.

**Auth**: handlers for login/refresh return fake JWTs (unsigned, correct shape); accept any
credentials; seed the demo user profile.

## Gotchas

- **Env/config**: point the app's `API_BASE_URL` env at same-origin (`/api`) so MSW can
  intercept; never leave a production URL that could be hit if a handler is missing.
- **WebSockets/SSE**: stub the client (alias) or use MSW's ws support; emit seeded events on
  a timer if the UI has live elements.
- **Third-party scripts** (analytics, Stripe.js, maps): exclude via demo build flags or load
  no-op shims from `demo/src/shims/`. External payment flows → fake in-demo checkout page
  that redirects back with success.
- **Latency**: `await delay(150)` in handlers so loading states render as designed.

## Build & serve

```bash
cd demo && npm i && npm run build   # -> demo/dist (git-ignored)
npx serve dist
```

Verify per AGENTS.md Phase 4 (every screen, CRUD, reload persistence, zero real requests).
