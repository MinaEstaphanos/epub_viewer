# WebKit scroll-echo guard

**Where:** `lib/assets/webpage/html/epubView.js`, `installScrollEchoGuard()`,
installed from `loadBook` on the rendition's `attached` event.
**Affects:** iOS (WKWebView). Inert on Android (Chromium never triggered it).
**Symptom it fixes:** navigating to a chapter from the table of contents (or
to a bookmark / a restored position) landed on the *previous* chapter, on a
blank page, or many pages into a chapter. Reproduced on the iOS Simulator
(iPad Pro 13, iOS 26.1) with ordinary reflowable EPUBs.

## Mechanism

epub.js's **continuous view manager** (used for both paginated and scrolled
flow) shows a section like this:

1. clear the container and scroll it to 0;
2. add the target section's view;
3. `fill()`/`check()` **prepend the previous section(s)** so the reader can go
   backwards;
4. compensate each prepend with a *relative* `scrollBy(+width | +height)`
   (`ContinuousViewManager.counter`) so the viewport stays on the target.

Step 4 breaks on WebKit. Logging the container state around a TOC tap in
scrolled flow gave (times in ms after the tap; the target is spine section 7):

```
143  scrollBy(0,26069)   -> scrollTop 26069   prepend of section 6 compensated
204  scrollBy(0,31521)   -> scrollTop 57590   prepend of section 5 compensated; read-back OK
227  display() resolved     scrollTop 57590   = top of the target section 7   ✓
228  scroll event           scrollTop 26069   <- the browser reverted to the PREVIOUS value
230  scrollBy(0,21614)   -> scrollTop 47683   next compensation stacks on the wrong base
245  relocated: section 6                     the previous chapter is on screen
```

No script ran between 227 and 228. WKWebView's asynchronous scrolling
**echoes an older programmatic scroll position back to the page** 20–150 ms
after a newer one was set, silently undoing the newer scroll. Because the
compensation is delta based, one lost scroll leaves every later position a
section off; `update()` then destroys the target view as "not visible", which
is why simply calling `display()` again did not help and why the first chapter
(nothing to prepend) always worked.

A second form: the echo can arrive **clamped**. The jump first scrolls to 0 and
removes the old views; when the browser then echoes the *pre-jump* position
into a container that is now only one section wide, the value is clamped to the
container's current maximum. epub.js stacks its deltas on that and the chapter
opens ~20 pages in instead of at its title page.

Both forms are timing dependent, so some jumps land correctly. Upstream reports
of the same WebKit-only symptom: epub.js #355 / #1131, epub_viewer #27.

## The guard

- Wrap `manager.scrollTo` / `manager.scrollBy` — every programmatic scroll in
  epub.js goes through them. After each call remember the resulting position as
  `intended`; push the previous `intended` onto a short `stale` list (last 6).
- On every container `scroll` event, and on 60 ms / 200 ms timers after each
  programmatic scroll: if the reported position differs from `intended`, we are
  within 500 ms of the last programmatic scroll, and the position equals one of
  the `stale` values **after clamping it to the container's current scroll
  range**, it is an echo → re-apply `intended` and sync the manager's cached
  `scrollLeft`/`scrollTop` so its next `check()` is right.
- Any other mismatch is a genuine user or browser scroll → adopt it and stop
  guarding until the next programmatic scroll. The guard therefore never fights
  a finger drag or the Snap page-turn animation.

Each intervention logs `SCROLL-ECHO-GUARD <trigger>: browser reported x,y (echo
of a superseded scroll) - re-applying x,y` to the console (forwarded to Flutter
as `JS_LOG:` in debug builds).

## Verification (iOS Simulator, iPad Pro 13, font size 150 %)

| Flow | Jump (spine index) | Without guard | With guard |
|------|--------------------|---------------|------------|
| scrolled | section 6 → 7 | opened section 6 | correct, 1 echo caught |
| paginated | section 7 → 9 | — | correct (no echo that run) |
| paginated | section 9 → 6 | page ~22 of section 6 | title page, 1 clamped echo caught |
| paginated, rtl text | section 4 → 5 | previous section | correct |
| paginated | injected page-turn swipe | — | normal turn, no guard activity |

A guard-only build (no probe) passed the same jumps. Android and a physical
iPhone were not retested at the time of writing.

## Why not something else

- Re-issuing `display()` (with href, index, or a CFI inside the section) does
  not help: the target view has already been destroyed and the re-display
  re-runs the same race.
- A fixed-delay retry fires too early or re-triggers the race.
- Patching epub.js's `dist/epub.js` would work but couples us to a build we
  otherwise keep pristine; the wrapper covers every path (TOC, bookmark, CFI
  restore, next/prev, trim) from `epubView.js`.

## Related observation (not fixed)

On WebKit every section view is measured twice: the first `expand()` runs
before the injected theme/font-size CSS applies, and the ResizeObserver
re-measures ~70–90 ms later at roughly 1.8× the size (26069 → 47683 px at
150 %). epub.js compensates the scroll for the growth, but a CFI target inside a
section that then reflows can drift (epub.js #355's "1–2 views ahead").
Applying the theme before the first measurement, or re-running `moveTo(target)`
on the target view's resize, would address it.

## Also in this area: swipe mapping for right-to-left books

Many right-to-left EPUBs (e.g. Arabic) declare no
`page-progression-direction` in their spine, so a host app may pass
`EpubDisplaySettings.defaultDirection = rtl` itself. The fork uses that value
for the **swipe mapping only**: the custom swipe handler is enabled on iOS as
well for rtl books (`useCustomSwipe` in `epub_viewer.dart`) and its left/right →
next/prev mapping flips (`pageProgression` in `epubView.js`), so a left-to-right
swipe turns the page. The epub.js layout stays `ltr`: passing `rtl` to
`renderTo` switched the continuous manager into its rtl layout, which rendered a
blank page and ignored swipes on WKWebView. Trade-off: rtl books on iOS get the
instant page flip Android already has instead of the finger-following snap.
