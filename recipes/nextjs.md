# Recipe: Next.js (pages router) → static browser demo

Target: a Next.js app (pages router) that talks to its backend via gRPC-web/Connect and/or REST,
possibly with SSR (`getServerSideProps` / `getInitialProps`) and `pages/api` routes.
Produce `demo/`: a **sibling Next app** that re-exports the product's real pages and builds with
`output: 'export'` to plain static files, with the whole backend faked in-process.

Distilled from a real demo of a large production portal (Next 16, 19 Connect services, ~174 RPCs,
93 auth-wrapped pages) — every gotcha below cost real debugging time.

## Architecture

Don't try to statically export the product app itself: its `next.config` (rewrites/redirects/
headers), `pages/api` and SSR pages will fight you. Instead:

```
demo/
  README.md, API_COVERAGE.md
  app/                        # a second Next app; node_modules resolve upward from the repo root
    next.config.ts            # output:'export', images:{unoptimized:true}, experimental.externalDir,
                              # transpilePackages copied from the product, + the ALIAS SEAM (below)
    postcss.config.mjs        # copy of the product's, with ABSOLUTE paths (see gotchas)
    tsconfig.json             # baseUrl -> product root so product-internal absolute imports resolve
    pages/                    # thin re-exports: `export { default } from '../../../pages/carriers'`
    public/<content> -> symlink into the product's public/ (logos, favicon)
    demoModules/              # alias targets (fake auth HOC, neutralized script loaders, SDK stubs)
    fakeBackend/              # transports, fetch interceptor, db, seed, handlers
```

`experimental.externalDir: true` lets the demo app's pages import product source outside its dir.
Only the pages you re-export get built — add pages module by module, driven by the loud-error alarm.

## The seam: build-time module replacement

Product code imports its transports/auth as module-level singletons via **relative** paths, so a
plain `resolve.alias` (request-string match) is not enough. Use webpack's
`NormalModuleReplacementPlugin` and build with `next build --webpack`:

```ts
new webpack.NormalModuleReplacementPlugin(/services\/connectClients\/portalTransport$/, res => {
  res.request = path.join(DEMO, 'fakeBackend/transports/portalTransport.ts');
});
```

Replace: every transport module, the auth HOC (see below), and any module that injects external
`<script>` tags (analytics/chat/billing SDK loaders) — a fetch interceptor cannot stop script-tag
loads, so neutralize `loadSnippet`-style helpers at the module level. For a loader imported both
bare and as relative `./shared`, match on the importer *context* too.

## Fake backend for Connect/gRPC-web

`createRouterTransport` from `@connectrpc/connect` runs real service handlers in-process — the
generated clients stay type-safe and never touch the network:

```ts
export const fakeTransport = createRouterTransport(({ service }) => {
  service(CarrierService, carrierHandlers);  // register EVERY service, even partially
}, { interceptors: [latency(220), loudUnimplemented] });
```

- Register **all** services; unimplemented methods must throw `Code.Unimplemented` plus a
  `console.error('[demo] unimplemented RPC: Svc/Method')`. This is the drift alarm AND the fastest
  way to discover what a screen actually calls (code reading gets it wrong; the alarm doesn't).
- **Server-streaming with no server**: implement the handler as an `async *method(req)` generator
  yielding protobuf messages with `await delay()` between chunks — the transport drives the real
  client's `for await` loop. Scripted AI-chat/progressive-results streams demo beautifully.
- Build responses with the generated schemas (`create(FooSchema, {...})`); verify field/enum values
  against the generated `*_pb.ts`, never from memory.

REST calls (login, session, flags, log sinks, presigned uploads): patch `window.fetch` at demo boot.
Same-origin static assets pass through; known API paths get fake responses; anything else external is
rejected loudly — zero real network is the hard requirement.

## SSR, auth and dynamic routes under `output: 'export'`

- `getServerSideProps` pages cannot be exported — replace each with a small client-side equivalent
  (`router.replace`, simulated token flows). `getInitialProps` pages *can* export, but the function
  runs at build time in Node — usually better to alias the auth HOC to a demo variant that does the
  session check client-side (localStorage session; redirect to the real sign-in page when absent).
  Keep the product's real login screen and accept any credentials.
- Dynamic routes (`[param]`) need `getStaticPaths` (`fallback: false`) in the demo re-export page,
  with paths from the seed. Non-seeded ids 404 on direct load (fine; document it). For records the
  user creates at runtime, pre-render a small slot pool of ids and hand those out on create.
- `images: { unoptimized: true }` (no image optimizer server). A static host with clean-URL support
  is required (`npx serve`, not `python http.server`) so `/home` maps to `home.html`.

## Gotchas

- **Monorepo/workspace**: the demo app needs no workspace registration — Node resolves
  `node_modules` upward. Build the workspace packages first if the product does (codegen!).
- **PostCSS/Tailwind/Panda**: the demo app's cwd changes the config base — copy the product's
  postcss config with **absolute** paths (Panda `configPath`, Tailwind `base` at the product root),
  or the build silently purges every product class.
- **`ignoreBuildErrors` is common in real apps** → a green build proves nothing; browser-verify
  every page. Blank screen + console throw is almost always a strict domain parser (zod) rejecting
  fake data — use real UUIDs (`crypto.randomUUID()`) and non-empty required fields in seeds.
- RPCs behind an error boundary degrade gracefully when unimplemented; RPCs under a bare
  `<Suspense>` crash the page — implement those first.
- Product `beforeunload` guards (dirty forms) block automated navigation with a dialog during
  verification — handle the dialog; it's product behavior, not a bug.
