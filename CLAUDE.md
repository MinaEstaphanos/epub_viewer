# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

A **fork of [fayis672/epub_viewer](https://github.com/fayis672/epub_viewer)**
(`flutter_epub_viewer`, a Flutter EPUB reader built on
[epub.js](https://github.com/futurepress/epub.js) inside a
`flutter_inappwebview` WebView). It carries a small set of reader fixes,
mostly for iOS (WKWebView) behaviour and right-to-left books, that are meant to
be offered upstream. Consumers pin the fork as a git dependency on branch
**`main`**, so anything pushed to `main` reaches them on their next
`flutter pub upgrade flutter_epub_viewer`.

The fork tracks the **1.2.x** line (currently 1.2.8). Upstream has since
released **2.0.0**, a breaking refactor (platform abstraction layer,
`callMethod(...)` instead of `evaluateJavascript(...)`). Do not rebase onto
2.0.0 casually: it needs a migration on the consuming side and a full reader
retest.

## Layout

- `lib/src/epub_viewer.dart` — the `EpubViewer` widget. Creates the WebView,
  registers the JavaScript → Flutter handlers (`addJavaScriptHandlers`), and
  calls the page's `loadBook(...)` with a long **positional** argument list
  (`loadBook()` near the bottom of the file). Adding a parameter means changing
  both this call and the JS function signature together.
- `lib/src/epub_controller.dart` — the `EpubController` API
  (`display`, `next`/`prev`, highlights, font size, search, …). Each method is a
  one-line `evaluateJavascript` into the page.
- `lib/src/models/` — settings and value types. `EpubDisplaySettings` is
  json_serializable; regenerate `*.g.dart` with
  `dart run build_runner build --delete-conflicting-outputs` after changing it.
- `lib/assets/webpage/html/` — the WebView page: `swipe.html` (markup + the
  layout CSS that matters, see the fixes list) and **`epubView.js`** (all the
  reader logic: epub.js setup, selection handling, navigation, the WebKit
  scroll guard, swipe mapping). This is where most fork work happens.
- `lib/assets/webpage/dist/epub.js` — the bundled epub.js build. **Unmodified**
  so far; keep fixes in `epubView.js` (wrappers/monkey-patches) unless a change
  is impossible there.
- `example/` — upstream's example app.

## How the pieces talk

- Flutter → page: `webViewController.evaluateJavascript(source: 'fn(...)')`.
- Page → Flutter: `window.flutter_inappwebview.callHandler('<name>', ...args)`;
  handler names are registered in `addJavaScriptHandlers()`
  (`relocated`, `displayed`, `chapters`, `locationLoaded`, `selection`, …).
- `console.log` in the page is forwarded to Flutter as `JS_LOG: …` lines in
  **debug builds only** (`onConsoleMessage`). This is the practical way to
  debug the reader on a device or simulator; see the testing section.
- epub.js rendering: `book.renderTo('viewer', {...})` with the **continuous**
  view manager for both flows (the Dart `EpubManager` enum has no other value).
  Paginated flow = horizontal axis with epub.js's Snap helper on iOS; scrolled
  flow = vertical axis. The layout direction is always `ltr` (see the rtl fix
  below for why).

## Fork changes on top of upstream 1.2.8 (newest last)

| Commit | Change |
|--------|--------|
| `bdbbaa7` | `setFontSizePercentage(...)` — font size as a percentage, not pixels. |
| `b47392d` | First paint applies the passed font size as `%` too, so a book opens at the saved size instead of flashing huge. |
| `72fb8bc` | The custom Android swipe handler only page-turns in **paginated** flow (`flow !== 'scrolled'`); in scrolled flow a sideways drift skipped content. |
| `5bd487e` | `overflow-anchor: none` on the scrolling container in `swipe.html` — Chromium's scroll anchoring stacked with epub.js's own compensation when scrolling up into a not-yet-rendered section. |
| `91774b1`, `f6b9425` | `html`/`body`/`#viewer` at `height: 100%`, no flex centering, `overflow: hidden` — a fixed blank band appeared at the bottom when the WebView shrank (e.g. for a bottom bar in the host UI). Never set `#viewer` to a fixed pixel height. |
| `78aa56b` | **WebKit scroll-echo guard** (`installScrollEchoGuard` in `epubView.js`): iOS WKWebView echoes stale programmatic scroll positions and reverted epub.js's compensation for prepended sections, so TOC/bookmark jumps landed on the previous chapter. Full write-up: [docs/webkit-scroll-echo-guard.md](docs/webkit-scroll-echo-guard.md). |
| `4900f82` | **Swipe mapping follows the page progression**: `defaultDirection: rtl` enables the custom swipe handler on iOS too and flips left/right → next/prev, while the epub.js layout stays `ltr`. Handing `rtl` to epub.js switched its continuous manager into the rtl layout, which rendered blank pages and ignored swipes on WKWebView. |

Keep this table current when adding a fix.

## Workflow

1. Work on `main` (or a branch for anything larger); commit with an
   imperative one-line subject (matches the history) and a body that says
   *why*, especially for WebView quirks. Commit only library files (`lib/`,
   docs, `CHANGELOG.md`) — leave `example/` scratch out.
2. Push, then in a consuming app run `flutter pub upgrade
   flutter_epub_viewer` and do a **cold rebuild** (`flutter run` / `flutter
   build`). Hot reload does not pick up changes to `epubView.js` or
   `swipe.html`; the device keeps running the old JavaScript.
3. Update the table above, add a line under *Unreleased* in `CHANGELOG.md`,
   and for anything non-obvious add a note under `docs/`.
4. Keep changes self-contained and free of consumer-specific details so they
   can be sent upstream as pull requests.

## Testing

- **Both engines every time.** Android renders the page in the Chromium
  WebView, iOS in **WKWebView**. Scroll timing, async scrolling, iframe sizing
  and scroll anchoring all differ; the echo guard and the rtl layout finding are
  both WebKit-only. A change is not done until it is checked on both.
- **Read the reader's console.** Debug build + on the iOS Simulator:
  `xcrun simctl spawn <udid> log stream --style compact --predicate
  'process == "Runner"'` and grep for `JS_LOG`. On Android, `adb logcat` shows
  the same lines. The `SCROLL-ECHO-GUARD …` lines report each guard
  intervention.
- **Inspecting the page.** iOS: Safari → Develop → Simulator → the reader page.
  Android: Chrome DevTools via `adb forward tcp:9222
  localabstract:webview_devtools_remote_<pid>`.
- Injected simulator swipes reach the WKWebView; adb-injected swipes do **not**
  reach the Android WebView (check scrolling by hand there).
- Known limitation: highlight taps are not bridged to Flutter (`markClicked`
  is not forwarded), so "tap a highlight to remove it" does not fire.
