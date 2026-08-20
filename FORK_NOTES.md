# Fork Notes

This is a fork of [wailsapp/wails](https://github.com/wailsapp/wails) v2,
based on tag `v2.11.0`, maintained for use by
[UX-Music](https://github.com/HariBote1110/UX-Music). Upstream Wails v2 is
otherwise unmodified; the only change is the addition described below, kept
on the branch `ux-music/webview-destroy`.

## What was added

Two new runtime capabilities that let the host application fully **destroy**
the platform webview while a window is hidden, and **recreate** it on
demand:

- `runtime.WindowUnloadWebView(ctx)`
- `runtime.WindowReloadWebView(ctx)`

### Why

UX-Music (a Wails v2 macOS app) parks its SPA into a tiny placeholder page
when the window is hidden so WebKit can free memory. In practice WebKit only
returns the freed pages to the OS after roughly 4.5 minutes, and a
significant amount (~106MB) is never returned at all while the `WKWebView`
instance and its WebContent process stay alive. Actually destroying the
`WKWebView` (so the associated WebContent process exits) returns that memory
to the OS immediately. Recreating the webview on demand restores the app.

### API

Added to the `Frontend` interface (`v2/internal/frontend/frontend.go`):

```go
WindowUnloadWebView()
WindowReloadWebView()
```

And the corresponding public runtime functions
(`v2/pkg/runtime/window.go`):

```go
func WindowUnloadWebView(ctx context.Context)
func WindowReloadWebView(ctx context.Context)
```

`WindowUnloadWebView` tears down the webview; `WindowReloadWebView` rebuilds
it and reloads the application's start URL. `WindowReloadWebView` is a no-op
if the webview is already loaded (i.e. it was never unloaded, or a previous
reload already happened). Calling any webview-facing runtime function
(`WindowExecJS`, event `Notify`, `WindowPrint`, etc.) while the webview is
unloaded is a safe no-op — it does not panic or block.

### Platform support

Only implemented on **darwin**. The **windows** and **linux** frontends
implement both methods as logged no-ops (`f.logger.Warning(...)`), matching
the existing pattern used elsewhere in this codebase for
platform-unsupported features. `internal/frontend/devserver.DevWebServer`
needs no changes: it embeds `frontend.Frontend` and forwards both new
methods automatically.

## darwin implementation

Files: `v2/internal/frontend/desktop/darwin/{WailsContext.h, WailsContext.m,
Application.h, Application.m, window.go, frontend.go}`.

- `WailsContext` now retains the pieces needed to recreate a webview
  independently of the webview object itself:
  - `webviewConfiguration` (`WKWebViewConfiguration*`) — created once in
    `CreateWindow` and kept alive. Its `WKUserContentController` (added via
    `config.userContentController`) is what carries the registered
    `"external"` script message handler (the IPC bridge) and the injected
    user script (the disable-context-menu flag), so it is **not**
    re-registered on recreation — it simply comes along with the reused
    configuration.
  - `webviewIsTransparent`, `enableDragAndDrop`, `disableWebViewDragAndDrop`
    — the flags `CreateWindow` originally used to configure the webview,
    needed again by `ReloadWebView`.
  - `startURLString` — the URL the app was told to load (captured the first
    time `loadRequest:` is called, i.e. at startup via `Application.m`'s
    `Run()`), reused by `ReloadWebView` to reload the same start URL.
- The webview-construction logic that used to live inline in `CreateWindow`
  was factored into a new method, `- (void) attachWebView`, which builds a
  `WailsWebView` from the retained configuration, adds it to the content
  view with the same frame/autoresizing/transparency/delegate setup as
  before, and is now called both by `CreateWindow` (startup) and by
  `ReloadWebView` (recreation).
- `- (void) UnloadWebView` clears the navigation/UI delegates, stops any
  in-flight load, removes the webview from its superview, and sets
  `self.webview = nil` — releasing WebKit's last strong reference so the
  `WKWebView` (and its WebContent process) can be torn down. No-op if
  already unloaded.
- `- (void) ReloadWebView` is a no-op if a webview already exists (i.e. it
  wasn't unloaded); otherwise it calls `attachWebView` and reloads
  `startURLString`.
- `ExecJS:` and `loadRequest:` are both guarded against a nil `self.webview`
  and become no-ops in that case; `Application.m`'s `WindowPrint` is guarded
  the same way. Every other Objective-C call site that touches `self.webview`
  (menus, dialogs, the dev-only inspector) either doesn't touch it or relies
  on Objective-C's "message to nil is a safe no-op" semantics, which this
  codebase already assumes elsewhere.
- Both new methods are exposed as C functions (`UnloadWebView`/
  `ReloadWebView` in `Application.h`/`Application.m`), dispatched onto the
  main thread via the existing `ON_MAIN_THREAD` macro — the same mechanism
  used by every other `Window*` call (`Hide`, `Show`, `ExecJS`, ...). The Go
  side (`window.go`) exposes `(*Window).UnloadWebView()` /
  `(*Window).ReloadWebView()`, and `frontend.go` wires those into the
  `Frontend` interface methods `WindowUnloadWebView()` / `WindowReloadWebView()`.
- As a side effect of factoring out `attachWebView`, the webview is now
  correctly balanced as `alloc` + `autorelease`, owned solely by the
  `(retain)` `webview` property (previously `CreateWindow` assigned
  `[WailsWebView alloc]` straight into the retained property without an
  offsetting release/autorelease, which would have left the `WKWebView`
  retained by both the raw `alloc` and the property forever, making it
  impossible for `UnloadWebView` to actually reach a zero retain count and
  let WebKit tear the object down). This fix is required for the feature to
  actually work, not just cosmetic.

### Objective-C memory management model

This codebase does **not** use ARC (no `-fobjc-arc` in any `#cgo CFLAGS`) —
it is manual retain/release (MRC), consistent with the explicit
`retain`/`release`/`dealloc` calls already present in `WailsContext.m`. The
new `webviewConfiguration` and `startURLString` properties are declared
`(retain)` and released in `dealloc`, matching the existing style.

### IPC bridge / runtime JS injection after recreation

Verified by reading `CreateWindow`:

- The IPC bridge is the `"external"` script message handler, registered via
  `[userContentController addScriptMessageHandler:self name:@"external"]`
  on the `WKUserContentController` that is now retained via
  `self.webviewConfiguration.userContentController` (same object,
  `self.userContentController` also keeps a direct reference). It is
  registered exactly once, at `CreateWindow` time, and is never
  re-registered elsewhere in the codebase. Because `ReloadWebView` reuses
  the same configuration/controller object, a recreated webview gets the
  same message handler without any extra registration work.
- The only injected `WKUserScript` set up in `CreateWindow` is the
  "disable default context menu" flag script — also added to the same
  retained `userContentController`, so it also survives recreation.
- The actual Wails JS runtime (`window.wails`, event bridge, etc.) is
  **not** injected as a `WKUserScript` in the darwin frontend — it is served
  as part of the HTML/JS asset bundle through the custom `wails://` URL
  scheme handler (`config setURLSchemeHandler:self forURLScheme:@"wails"`,
  handled by `WailsContext`'s `WKURLSchemeHandler` implementation,
  ultimately backed by `assetserver.AssetServer`). Since a fresh webview
  reloading `startURLString` (`wails://wails/...`) re-requests and re-runs
  that bundle from scratch, the runtime boots up again exactly as it does
  on first launch — i.e. recreating the webview is equivalent to a full
  page reload/app relaunch from the SPA's point of view, which matches the
  design intent ("Reload → app relaunch semantics").
- The `WKWebViewConfiguration` itself is *copied* internally by
  `-[WKWebView initWithFrame:configuration:]` (Apple's documented behaviour:
  `WKWebViewConfiguration` conforms to `NSCopying`, and `WKWebView` makes an
  immutable copy at init time). This copy is a shallow copy — the copied
  configuration's `userContentController`, `websiteDataStore`, scheme
  handlers, etc. reference the *same* underlying objects as the original.
  This is exactly why keeping the *original* `WKWebViewConfiguration`
  (rather than only the `userContentController`) retained and reused for
  every recreation is both sufficient and necessary: every new `WKWebView`
  gets an independent copy of the configuration wrapper, but all the
  registrations inside it (message handlers, scheme handler, preferences)
  are shared, live objects.

## Build verification

Verified in this repository (macOS, Xcode toolchain):

- `cd v2 && go build ./...` — clean (only pre-existing, unrelated noise:
  `go build` errors from template-embedding stub packages under
  `pkg/templates/...` that also occur on an unmodified checkout, and one
  pre-existing macOS-15 deprecation warning for `setShowsBaselineSeparator:`
  in `WailsContext.m` at an unrelated line — both confirmed present on a
  `git stash`'d baseline of this same tree).
- `cd v2 && go build ./internal/frontend/... ./pkg/runtime/... ./internal/app/...`
  — clean, and again with `-tags dev` (exercises `internal/frontend/devserver`,
  which embeds `frontend.Frontend` and therefore must — and does — pick up
  the two new interface methods without any code change there).
- `cd v2 && GOOS=windows GOARCH=amd64 CGO_ENABLED=1 go build ./internal/frontend/desktop/windows/...`
  — clean (this fork's Windows frontend cross-compiles from macOS because it
  uses `syscall`/win32 API bindings rather than a C toolchain requiring
  platform headers).
- `cd v2 && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 go build ./internal/frontend/desktop/linux/...`
  — fails, but for reasons unrelated to this change: cross-compiling cgo for
  Linux from macOS needs a real Linux libc/toolchain, and fails already in
  Go's own `runtime/cgo` package (`setresgid`/`setresuid` undeclared against
  the macOS SDK headers) before it ever reaches this fork's code. The Linux
  frontend's `WindowUnloadWebView`/`WindowReloadWebView` no-op additions were
  instead verified via `gofmt -l` (no diff) and manual inspection against
  the `Frontend` interface — the method signatures and receiver match
  exactly what the interface requires and what the darwin/windows sibling
  files do.

## What could not be verified by compilation alone

- Runtime behaviour: that `UnloadWebView` actually causes WebKit's
  WebContent process to exit and the RSS to drop, and that
  `ReloadWebView` produces a working, IPC-connected webview again with no
  visual glitches (frame/autoresizing/transparency restored correctly).
  This needs an app-side integration test running the actual UX-Music
  binary (or a minimal Wails v2 darwin sample app) through a
  hide → unload → wait → reload → show cycle, observing both memory (e.g.
  Activity Monitor / `vmmap` on the WebContent process) and that the SPA
  and IPC bridge come back up correctly.
- Interaction with `HideWindowOnClose` / `StartHidden` / single-instance
  relaunch flows, and with window resize while unloaded (no webview to
  resize — the `NSView` autoresizing mask is only applied when the webview
  exists, so nothing should break, but this needs an on-window check).
- Multi-window support: this fork's `WailsContext` model is single-window
  (matches upstream v2's darwin frontend), so no per-window bookkeeping was
  needed; this was not otherwise exercised.

## Rebasing onto a future v2.x tag

This fork's only change is additive and touches a small, well-isolated set
of files:

```
v2/internal/frontend/frontend.go
v2/internal/frontend/desktop/darwin/WailsContext.h
v2/internal/frontend/desktop/darwin/WailsContext.m
v2/internal/frontend/desktop/darwin/Application.h
v2/internal/frontend/desktop/darwin/Application.m
v2/internal/frontend/desktop/darwin/window.go
v2/internal/frontend/desktop/darwin/frontend.go
v2/internal/frontend/desktop/windows/frontend.go
v2/internal/frontend/desktop/linux/frontend.go
v2/pkg/runtime/window.go
FORK_NOTES.md
```

To rebase onto a newer upstream `v2.x.y` tag:

1. `git checkout ux-music/webview-destroy`
2. `git rebase v2.x.y` (or merge, if the branch is shared) and resolve
   conflicts — the most likely friction points are:
   - `CreateWindow` in `WailsContext.m` if upstream refactors webview
     construction (this fork already factored the construction step into
     `attachWebView`, which should make future refactors easier to diff
     against, not harder).
   - The `Frontend` interface in `frontend.go` if upstream adds its own new
     methods in the same region.
   - `pkg/runtime/window.go` similarly, if upstream adds neighbouring
     `Window*` functions.
2. Re-run the build verification steps listed above.
3. If UX-Music's `go.mod` uses a `replace` directive pointing at this fork,
   bump the pinned commit/tag there once the rebase is pushed.
