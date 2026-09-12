## Unreleased (fork)
- Dragging a selection handle no longer clears the selection on iOS. The
  swipe-clears-selection guard now leaves a gesture alone when the selection is
  changing under the finger (WKWebView forwards handle drags to the page as
  touchmove events; Android's native selection layer does not)
- Fixed `EpubTheme.customCss` being ignored on the initial load: `loadBook`
  re-applied the theme without the custom rules after registering them, and
  epub.js replaces a theme's rules on re-register, so sections rendered with
  only the body colour
- While text is selected, a swipe (and a tap) now clears the selection. Upstream
  blocks page-turn swipes whenever a selection is active — via the JS touch
  handlers, which `preventDefault` horizontal moves (the `touch-action: pan-y`
  style is invalid and a no-op) — and relies on the selection clearing to lift
  the block. On Android the native selection layer swallows a tap, so the
  selection never cleared and the reader froze on swipe and scroll with no way
  out. The touch handlers now call `clearSelection()` when a horizontal swipe is
  detected during a selection (the same touchmove that blocks the swipe, so it
  reliably fires), and the tap branch clears too; both reset the
  `isSelecting`/`lastCfiRange` flags and notify the host to lift the block.
- Fixed TOC/bookmark/position-restore jumps landing on the previous chapter or
  the wrong page on iOS: WKWebView echoes stale programmatic scroll positions
  and reverted epub.js's compensation for prepended sections. A scroll-echo
  guard in `epubView.js` re-applies the intended position
  (see `docs/webkit-scroll-echo-guard.md`)
- `defaultDirection: rtl` now maps swipes to the page progression (a
  left-to-right swipe turns the page) on iOS and Android; the epub.js layout
  stays ltr because its rtl layout renders blank pages on WKWebView
- Fixed a blank band at the bottom of the reader after the WebView shrinks
  (`html`/`body`/`#viewer` at `height: 100%`, `overflow: hidden`)
- Disabled native scroll anchoring on the scrolling container in scrolled
  flow (it stacked with epub.js's own compensation on Chromium)
- The custom Android swipe handler no longer page-turns in scrolled flow
- Added `setFontSizePercentage`; the initial font size is applied as a
  percentage too

## 1.2.8
- Fixed `getCurrentLocation` function

## 1.2.7
- Added `customCss` support in `EpubTheme` for granular styling

## 1.2.6
- Fixed dispose issue ([#71](https://github.com/fayis672/epub_viewer/issues/71))

## 1.2.5
- Added onTouchUp and onTouchDown ([#93](https://github.com/fayis672/epub_viewer/pull/93))
- Add selectAnnotationRange parameter to control annotation selection behavior ([#96](https://github.com/fayis672/epub_viewer/pull/96))
- Block navigation while text is selected ([#92](https://github.com/fayis672/epub_viewer/pull/92))
- Add XPath/XPointer support for EPUB navigation and search ([#89](https://github.com/fayis672/epub_viewer/pull/89))

## 1.2.4
- Initial progress fixes
- Relocation on font change fixes
- background decoration fixes
- Added onSelection callback with WebView-relative coordinates for custom selection UI
- Added onSelectionChanging callback for detecting selection handle dragging
- Added onDeselection callback for selection cleared events
- Added clearSelectionOnPageChange property to control selection behavior on navigation
- Fixed suppressNativeContextMenu to properly hide native context menu
- Fixed text selection coordinate mapping for accurate positioning

## 1.2.3
- iOS chapter parsing fixes

## 1.2.2
- Added book metadata

## 1.2.1
- Fixed book loading issues
- Fixed font size adjust issues
- Added change theme function
- Added navigation to first and last pages

## 1.2.0
- Added Epub Theme with background and foreground color

## 1.1.6
- Remove Highlight fix

## 1.1.5
- LTR -RTL fixes
- Sub chapter parsing fixes
- Fixed `onRelocated` callback on Android
- Changed Default display settings

## 1.1.4

- Size fit fixes

## 1.1.3

- Added reading progress

## 1.1.2

- Added Annotation click handler

## 1.1.1

- Fixed book reloading issues

## 1.1.0

- Added Local file and asset support
- Added underline annotation
- Added text content extraction

## 1.0.2

- Document changes

## 1.0.1

- Fixed blank screen

## 1.0.0

- Highlight text
- Search in Epub
- List chapters
- Text selection
- Highly customizable UI
- Resume reading using cfi
- Custom context menus for selection
