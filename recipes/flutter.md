# Recipe: Flutter app → browser demo

Target: a Flutter mobile/desktop app whose API calls go through a central `ApiClient`
(package `http`). Produce `demo/`: a Flutter **web** project reusing the app's real screens.

## Preconditions

- The app's API client accepts an injectable `http.Client`
  (`ApiClient({http.Client? httpClient})`). If it doesn't — e.g. the app calls top-level
  `http.get/post/...` directly — either (a) add a tiny shared
  `lib/services/http_client.dart` with `http.Client httpClient = http.Client();` and route
  the top-level calls (and `MultipartRequest.send()`) through it — explicit, covers
  multipart, one assignment swaps in the fake (preferred; reference: funquila) — or
  (b) zero-touch: `http.runWithClient(() => runApp(...), () => FakeBackendClient(db))`
  in the demo entrypoint (package `http` >= 0.13.4; zone-based).
- **Dio apps** (instead of package:http): the seam is a fake `HttpClientAdapter`
  installed on the app's Dio (add an optional `HttpClientAdapter?` constructor param to
  the API client if needed — backwards-compatible). With Riverpod, override the API
  client provider in the demo's `ProviderScope`. Interceptors (bearer, 401→refresh)
  keep working against the fake. Reference: fecha.
- **Mobile-only apps** (`dart:io` imports, no web/ dir): add conditional-import shims
  (`io_shim.dart` + `io_shim_io.dart`/`io_shim_web.dart`) that re-export dart:io on
  native and provide throwing/no-op stubs on web, then repoint the imports. Same for
  `Image.file` → a `localFileImage()` helper. Reference: fecha.
- **flutter_secure_storage on web**: its WebCrypto path races master-key generation
  under parallel writes (`Future.wait` of several `write()`s) → values encrypted with
  a discarded key → `OperationError` on read. Give the storage class a demo-only
  plain-SharedPreferences mode and enable it from the demo entrypoint. Reference: fecha.
- WebSockets are NOT covered by the http seam. If the app has a WS channel (chat, live
  updates), the least-invasive pattern is a static opt-out flag on the service
  (`ChatService.disableSocket = true` set by the demo main; `connect()` early-returns so no
  reconnect loop spins) and serving the feature via its REST endpoints from the fake backend.
- The app compiles for web. Native-only plugins (firebase_messaging, push, etc.) are fine as
  transitive deps as long as the demo entrypoint never initializes them; verify none are
  initialized from widget code you reuse (guard with `kIsWeb` in the app if needed).

## Layout

```
demo/
  pubspec.yaml          # name: <app>_demo; path dep on ../app; web only
  web/                  # copied from app/web (or minimal index.html + manifest)
  lib/
    main.dart           # demo entrypoint: wires FakeBackendClient into ApiClient
    fake_backend/
      client.dart       # http.BaseClient implementing send() via the router
      router.dart       # (method, path) -> handler
      handlers/*.dart   # one file per domain (auth, users, billing, ...)
      db.dart           # JSON store persisted to window.localStorage
      seed.dart         # versioned seed dataset
  README.md
  API_COVERAGE.md
  serve.sh              # build + serve locally
```

## Key pieces

**pubspec.yaml** — depend on the app by path; never copy its source:

```yaml
name: app_demo
environment: { sdk: ^3.x.0 }
dependencies:
  flutter: { sdk: flutter }
  <app_package>: { path: ../app }
  http: ^1.2.0
  shared_preferences: ^2.3.0
```

**Fake client** — subclass `http.BaseClient`; everything else stays real:

```dart
class FakeBackendClient extends http.BaseClient {
  final FakeDb db;
  FakeBackendClient(this.db);

  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) async {
    await Future<void>.delayed(const Duration(milliseconds: 150));
    final res = await route(db, request); // -> FakeResponse(status, jsonBody)
    return http.StreamedResponse(
      Stream.value(utf8.encode(res.body)), res.status,
      headers: {'content-type': 'application/json'}, request: request,
    );
  }
}
```

**Demo main** — mirror the app's `main()` minus native services:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initializeDateFormatting('es_ES');           // same as the app
  final db = await FakeDb.open();                    // localStorage-backed, seeds itself
  final api = ApiClient(httpClient: FakeBackendClient(db));
  // Do NOT init push/firebase/analytics here.
  final state = AppState(api);
  await state.bootstrap();
  runApp(KonectaApp(state: state));                  // the app's real root widget
}
```

- The app's own `main.dart` is never imported (importing it would drag in push init);
  import the root widget + state directly.
- `shared_preferences` on web already writes to localStorage — token persistence and
  "stay logged in" work for free.
- Auth: the fake `/auth/login` (and social-login endpoints, if any) accept anything and
  return fake tokens; `/auth/refresh` always succeeds. Social login buttons that call native
  SDKs before hitting the API should be either handled by their web fallback or left
  non-functional with the regular login documented in the README.

## Gotchas

- **Drag-scrolling**: Flutter web only drag-scrolls with touch by default — in the
  desktop shell's phone frame, mouse/trackpad dragging does nothing (wheel works). Set
  `MaterialApp.scrollBehavior` to a `MaterialScrollBehavior` subclass whose `dragDevices`
  include mouse/trackpad/stylus. Patch it where the demo's MaterialApp actually lives:
  the app's root widget if the demo reuses it, or the demo's own app widget if not.
- **Cache headers on the demo host**: serve entry files (shell, index.html,
  flutter_bootstrap.js, main.dart.js, service worker) with `Cache-Control: no-cache` and
  only the heavy assets (canvaskit/, assets/, icons/) with a long max-age. Without
  headers, heuristic HTTP caching keeps serving STALE bundles after redeploys — new
  deploys silently don't reach returning visitors (and it will gaslight your own
  verification).

- **Deriving routes**: grep the app's `ApiClient` for path literals (`'/api/...'`). Every
  method there = one handler. Unknown route → 501 + `console.error` (via `dart:developer log`).
- **Renderer**: build with the default renderer. If the app uses `intl` with a locale, call
  `initializeDateFormatting` exactly like the real main does.
- **Images/uploads**: `image_picker`/`file_picker` work on web; store picked files as data
  URLs in the fake DB so avatars/photos persist.
- **URLs the UI opens** (checkout, documents): point them at in-demo placeholder pages under
  `web/` (e.g. `demo_checkout.html`) that simulate success and deep-link back — never at
  production.
- **Base URL**: the fake client ignores the host entirely (route on `request.url.path`), so
  whatever base URL the app has persisted is harmless. Nothing ever reaches the network.
- **Reset**: expose reset by seeding a "Reset demo data" entry where the product has a natural
  spot ONLY if trivial; otherwise document `localStorage.clear()` in the README. Bump the seed
  `version` constant to force re-seed on next load.

## Build & serve

```bash
cd demo
flutter pub get
flutter build web --release --no-web-resources-cdn   # -> demo/build/web (git-ignored)
python3 -m http.server 8080 -d build/web
```

`--no-web-resources-cdn` bundles CanvasKit locally instead of fetching it from
gstatic at runtime. If the demo will be served under a sub-path of the product's
domain (e.g. `example.com/demo`), build with `--base-href /demo/` and serve the
bundle at that path (Flutter's hash routing needs nothing else). For local testing
of a sub-path build, serve a parent dir with a `demo -> build/web` symlink. Two external static fetches may remain and should be listed in
the demo README as the only exceptions: the engine's fallback fonts
(`fonts.gstatic.com`) and, if the app depends on `google_sign_in`, the GSI script
its web plugin auto-injects (inert if the buttons are hidden; nothing is sent).

Verify per AGENTS.md Phase 4: walk every tab, CRUD something, reload, check the network tab
shows only local requests.
