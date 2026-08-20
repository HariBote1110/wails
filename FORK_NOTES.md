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

- `WailsContext` retains only the reusable *pieces* needed to rebuild a
  webview independently of the webview object itself — deliberately **not**
  a `WKWebViewConfiguration** (see "Why `UnloadWebView` didn't actually
  terminate the process" below for why that distinction matters):
  - `userContentController` (`WKUserContentController*`, already retained
    prior to this change) — carries the registered `"external"` script
    message handler (the IPC bridge) and the injected user script (the
    disable-context-menu flag). Reused as-is on every rebuilt configuration.
  - `wailsURLSchemeHandler` (`id<WKURLSchemeHandler>`, `assign`/unretained)
    — the object registered for the `wails://` scheme, obtained once via the
    public `-[WKWebViewConfiguration urlSchemeHandlerForURLScheme:]` in
    `CreateWindow`. In practice this is always `self`; it is stored
    unretained to avoid a self-retain cycle (self already owns this
    reference's lifetime, the same pattern already used for `appdelegate`).
  - Primitive preference flags captured from `CreateWindow`'s `Preferences`
    struct and `fraudulentWebsiteWarningEnabled` argument
    (`fraudulentWebsiteWarningEnabled`, `hasTabFocusesLinks`/
    `tabFocusesLinksValue`, `hasTextInteractionEnabled`/
    `textInteractionEnabledValue`, `hasFullscreenEnabled`/
    `fullscreenEnabledValue`) — re-applied to a fresh `WKPreferences` on
    every rebuild.
  - `webviewIsTransparent`, `enableDragAndDrop`, `disableWebViewDragAndDrop`
    — the flags `CreateWindow` originally used to configure the webview,
    needed again by `ReloadWebView`.
  - `startURLString` — the URL the app was told to load (captured the first
    time `loadRequest:` is called, i.e. at startup via `Application.m`'s
    `Run()`), reused by `ReloadWebView` to reload the same start URL.
- The webview-construction logic that used to live inline in `CreateWindow`
  was factored into a method, `- (void) attachWebView`, which now builds a
  **brand new `WKWebViewConfiguration`** from the pieces above every time it
  runs (instead of reusing one retained instance), assembles a `WailsWebView`
  from it, adds it to the content view with the same
  frame/autoresizing/transparency/delegate setup as before, and is called
  both by `CreateWindow` (startup) and by `ReloadWebView` (recreation). The
  local `config` is released immediately after handing it to
  `-[WailsWebView initWithFrame:configuration:]`, which — per Apple's
  documented `WKWebViewConfiguration` copy-at-init behaviour — takes its own
  immutable copy, so releasing the local variable does not affect the live
  webview.
- `- (void) UnloadWebView` clears the navigation/UI delegates, stops any
  in-flight load, removes the webview from its superview, then — as a
  best-effort, SPI-guarded nudge — invokes the private WebKit method
  `-[WKWebView _close]` if `respondsToSelector:` confirms it exists, and
  finally sets `self.webview = nil` to drop WebKit's last strong reference.
  No-op if already unloaded.
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
`retain`/`release`/`dealloc` calls already present in `WailsContext.m`.
`startURLString` and `userContentController` are declared `(retain)` and
released in `dealloc`, matching the existing style. `wailsURLSchemeHandler`
is `(nonatomic, assign)` — deliberately unretained, because the object is
always `self`; retaining it would create a self-retain cycle that prevents
`WailsContext` (and everything it owns) from ever being deallocated. Each
call to `attachWebView` builds a local `WKWebViewConfiguration*` with
`alloc`/`new`, hands it to `-[WailsWebView initWithFrame:configuration:]`,
and releases it immediately afterwards — balanced, and safe because
`WKWebView` copies the configuration at init time rather than retaining the
instance it was given.

### Why `UnloadWebView` didn't actually terminate the WebContent process

An earlier version of this fork (see the "darwin implementation" section
above, describing the state of things immediately after the initial
addition of `UnloadWebView`/`ReloadWebView`) claimed that releasing the
`WKWebView` was sufficient for its WebContent process to exit and its
memory to be returned to the OS. **Real-world testing in the host app
(UX-Music) showed this was wrong**: `com.apple.WebKit.WebContent` survived
`UnloadWebView` indefinitely — shrunken and App Nap-suspended rather than
gone — and its memory was only returned to the OS minutes later, at a
nondeterministic time. Manually killing the WebContent process from
Activity Monitor confirmed the app's native audio playback (which runs in
the main process, not the webview) was unaffected, which is what made it
safe to pursue a more aggressive fix here.

The root cause was that `UnloadWebView` released the `WKWebView` instance
but the `WailsContext` object itself kept the **`WKWebViewConfiguration`**
retained (as `self.webviewConfiguration`) so that `ReloadWebView` could
reuse it. A `WKWebViewConfiguration` carries a `WKProcessPool`, and
WebKit's WebContent process lifecycle is tied to process pools and their
associated caches, not just to the `WKWebView` instances that were created
from a given configuration. Keeping the configuration (and therefore the
process pool) alive kept WebKit's internal bookkeeping pointed at a process
it could reuse, so instead of tearing it down WebKit suspended it — correct
behaviour from WebKit's point of view (it is optimising for a fast
subsequent reload), but the opposite of what `UnloadWebView`'s memory-relief
goal needed.

The fix (this change) is to never retain a `WKWebViewConfiguration` across
an unload/reload cycle. `attachWebView` now builds one from scratch, from
primitive/reusable pieces that don't carry a process pool reference forward
(see the "darwin implementation" section above). Additionally,
`UnloadWebView` now attempts the private `-[WKWebView _close]` method,
guarded by `respondsToSelector:`, as a best-effort synchronous nudge; this
is WebKit SPI (not public API), could disappear or change behaviour in a
future WebKit release without notice, and is not the primary fix — it is
the config-rebuild that is expected to make WebKit actually let the
WebContent process go, rather than merely suspend it, because there is no
longer a retained process pool giving WebKit a reason to keep it around.

### IPC bridge / runtime JS injection after recreation

Verified by reading `CreateWindow`:

- The IPC bridge is the `"external"` script message handler, registered via
  `[userContentController addScriptMessageHandler:self name:@"external"]`
  on the `WKUserContentController` retained as `self.userContentController`.
  It is registered exactly once, at `CreateWindow` time, and is never
  re-registered elsewhere in the codebase. Because `attachWebView` assigns
  the same retained controller object (`config.userContentController =
  self.userContentController`) onto every freshly built configuration, a
  recreated webview gets the same message handler without any extra
  registration work.
- The only injected `WKUserScript` set up in `CreateWindow` is the
  "disable default context menu" flag script — also added to the same
  retained `userContentController`, so it also survives recreation.
- The actual Wails JS runtime (`window.wails`, event bridge, etc.) is
  **not** injected as a `WKUserScript` in the darwin frontend — it is served
  as part of the HTML/JS asset bundle through the custom `wails://` URL
  scheme handler, re-registered on every fresh configuration via
  `[config setURLSchemeHandler:self.wailsURLSchemeHandler
  forURLScheme:@"wails"]` (handled by `WailsContext`'s
  `WKURLSchemeHandler` implementation, ultimately backed by
  `assetserver.AssetServer`). Since a fresh webview reloading
  `startURLString` (`wails://wails/...`) re-requests and re-runs that
  bundle from scratch, the runtime boots up again exactly as it does on
  first launch — i.e. recreating the webview is equivalent to a full page
  reload/app relaunch from the SPA's point of view, which matches the
  design intent ("Reload → app relaunch semantics").
- `WKWebViewConfiguration` is *copied* internally by
  `-[WKWebView initWithFrame:configuration:]` (Apple's documented behaviour:
  `WKWebViewConfiguration` conforms to `NSCopying`, and `WKWebView` makes an
  immutable copy at init time). This copy is a shallow copy — the copied
  configuration's `userContentController`, scheme handlers, etc. reference
  the *same* underlying objects as the original passed to
  `attachWebView`'s local `config`, which is why it is safe to build
  `config` fresh and release it immediately: the live `WKWebView` keeps its
  own copy, and that copy's registrations still point at the shared,
  long-lived `userContentController`/scheme handler objects retained on
  `self` — see "Why `UnloadWebView` didn't actually terminate the
  WebContent process" above for why the configuration *object* itself is
  deliberately *not* one of those long-lived retained things.

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

- **WebContent process exit timing**: whether `UnloadWebView`, after this
  change, causes `com.apple.WebKit.WebContent` to actually terminate (not
  just shrink/suspend), and how quickly — immediately as a result of
  dropping the last `WKWebViewConfiguration`/`WKProcessPool` reference plus
  the `_close` SPI call, or only eventually. This can only be observed by
  running the real host app (or a minimal Wails v2 darwin sample) through a
  hide → unload → wait cycle and watching the WebContent process's PID in
  Activity Monitor / `ps` — does the process disappear, and if so within
  what timeframe relative to before this change (previously: minutes,
  nondeterministic). Compilation and static review can confirm the
  configuration is no longer retained and that `_close` is invoked when
  available, but not what WebKit actually does in response on a given OS
  version.
- Runtime behaviour more broadly: that `ReloadWebView` produces a working,
  IPC-connected webview again with no visual glitches (frame/autoresizing/
  transparency restored correctly) after an unload, using the freshly
  rebuilt configuration. This needs the same app-side integration test as
  above, extended through hide → unload → wait → reload → show, observing
  memory and that the SPA and IPC bridge come back up correctly.
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
