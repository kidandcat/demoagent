# demoagent

**Turn any product repository into a browser-only demo** — the product's real
frontend, with the entire backend emulated client-side. No servers, no accounts,
no real data: a static bundle anyone can try from a URL.

demoagent is not a library or a CLI. It is an **agent definition**: a spec that
modern coding agents (Claude Code, Codex, Cursor, Gemini CLI, …) can execute
against your repo to create — and later keep up to date — a `demo/` folder
containing the demo.

## How to use

Give your coding agent the spec and an order, from the target product repo:

```
Follow <path-to>/demoagent/AGENTS.md: create the browser demo for this product in demo/.
```

or, for an existing demo after the product changed:

```
Follow <path-to>/demoagent/AGENTS.md: update demo/ to match the current app.
```

The agent produces a self-contained `demo/` at the repo root that builds to
static files with one command, plus its own docs (`demo/README.md` and
`demo/API_COVERAGE.md`, a route-by-route coverage table).

## What the spec enforces

- **Visually identical** — the demo imports the product's real frontend
  (path dependency / workspace), never a re-implementation. It tracks the
  product automatically.
- **Zero backend** — one seam (injectable API client, network interception, or
  build-time substitution) cuts the frontend from its backend; a fake backend
  implements the API contract in the browser.
- **Browser database** — seeded, versioned state in localStorage/IndexedDB.
  Created records survive reloads; `?reset=1` reseeds.
- **No leaks** — no real user data, no secrets, no server source. Server
  behavior is re-implemented plausibly from the API contract, never copied.
- **Full feature surface** — every screen works, every CRUD completes. What
  can't be real client-side (payments, push, OAuth) is simulated, not hidden.
- **Isolation** — everything lives in `demo/`; touches outside it must be
  minimal, backwards-compatible, and documented.
- **Desktop shell** — mobile products get a no-scroll landing with a visual
  mini-guide and the app in a phone frame; small screens skip straight to the
  app.

## Repo layout

- **`AGENTS.md`** — the agent definition: mission, non-negotiables,
  architecture, phase-by-phase workflow, definition of done. The source of
  truth ([AGENTS.md convention](https://agents.md)).
- **`CLAUDE.md`** — Claude Code entrypoint (imports `AGENTS.md`).
- **`recipes/`** — per-stack implementation guides distilled from real demos,
  with the gotchas that cost real debugging time (e.g. Flutter web only
  drag-scrolls with touch by default; `flutter_secure_storage` races its
  WebCrypto master key on web; entry files must be served with
  `Cache-Control: no-cache` or redeploys never reach returning visitors):
  - `recipes/flutter.md` — Flutter app → Flutter web demo
  - `recipes/react.md` — React/Vue/JS SPA → demo build
  - `recipes/nextjs.md` — Next.js (pages router) → static export demo

If your stack has no recipe, the spec instructs the agent to follow the general
architecture and **write the missing recipe back into the repo** — that's how
the existing ones came to exist.

## License

[MIT](LICENSE)
