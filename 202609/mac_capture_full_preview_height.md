---
tier: tale
title: Grow the Bob Mac Capture window to show the full live preview
goal: The bob-mac-capture panel grows its window to render every live-preview card
  in full (no clipped header or rows) whenever the screen has room, and only scrolls
  the auxiliary region once the window reaches the screen-height limit.
size: medium
proposed_by: bbugyi200.apollo.2z
status: done
---

# Plan: Grow the Bob Mac Capture window to show the full live preview

## Goal

When the capture panel shows a live preview (for example the Pomodoro close card for
`=x`), the panel window must grow tall enough to show the **entire** preview card and
destination line. Content may only be hidden behind scrolling when the window has
already reached the screen-height limit. Today the preview is cut off even when there is
plenty of vertical room on screen.

Reference screenshot: `~/tmp/screenshots/20260929_082619.png`. The draft is `=x`, and
the footer reads `Would close GOALS · 3 started · 0 Work Log entries`. The close card
shows:

- its `Close GOALS` header sliced in half by the card's own rounded top edge, and
- only 2 of the 3 started task rows, with the card's bottom cut off by a square edge (no
  rounded bottom corners), directly above the footer.

The window sits mid-screen with hundreds of points of free space below it.

## Where the work happens

All code changes belong in the linked `bob-mac-capture` repository (macOS 26
SwiftUI/AppKit app). `bob-cli` does not change, and neither do the capture grammar, the
`bob` subprocess arguments, or the JSON contracts. Open the repo through the `sase_repo`
skill and use only the path it prints:

```sh
sase repo open bob-mac-capture -r "Grow capture panel to show the full live preview"
# If the linked checkout is unavailable on this host, open it by provider ref instead:
sase repo open gh:bobs-org/bob-mac-capture -r "Grow capture panel to show the full live preview"
```

The paths below are relative to that checkout. The files involved are
`Sources/BobMacCapture/CapturePanelView.swift`,
`Sources/BobMacCapture/CapturePanelController.swift`,
`Sources/BobMacCapture/CapturePanelModel.swift` (one published property only), tests
under `Tests/BobMacCaptureTests/`, and `README.md`.

## Root cause (two independent clipping layers)

The autosizing pipeline works like this: SwiftUI measures the editor, the auxiliary
document, and the footer. `CapturePanelContentHeightPolicy` composes them into
`CapturePanelContentMetrics`. `CapturePanelController` applies them through
`CapturePanelWindowSizer`. Two independent defects defeat that pipeline for the preview.

### 1. `PreviewPane` is pinned to 92 pt inside a scroll view (clips the top and bottom of the card)

`PreviewPane` (near the end of `CapturePanelView.swift`) ends with:

```swift
.frame(
    maxWidth: .infinity,
    minHeight: CapturePanelLayout.previewIdealHeight,   // 92
    idealHeight: CapturePanelLayout.previewIdealHeight, // 92
    alignment: .leading
)
.padding(10)
.background(.thinMaterial)
.clipShape(RoundedRectangle(cornerRadius: 8))
```

The pane lives inside `auxiliaryScrollRegion`'s vertical `ScrollView`, which proposes an
unspecified (nil) height. A flexible frame replaces a nil proposal with its
`idealHeight`, so the frame is always exactly 92 pt no matter how tall its content is.
Taller content (a close card with a header, a destination row, three task rows, and
work-log lines; also start, toggle, link, and batch previews) overflows. The `.leading`
alignment centers it vertically, and `clipShape` then cuts the overflow off at both the
top and the bottom of the card. The auxiliary height that reaches the window sizer is
therefore always `92 + 20`, so the window never learns that the card needs more room.
This contradicts the README contract that "the outer auxiliary detail region owns
scrolling, so preview itself never nests another scroll view": the pane is supposed to
take its natural height. `previewIdealHeight` has been used only here since
`434c753 feat: autosize capture draft editor`.

### 2. The titlebar safe-area inset is missing from the height budget (clips the bottom of the auxiliary region)

`CapturePanelController.makePanel()` builds a `.titled` + `.fullSizeContentView` panel.
With a full-size content view, `frameRect(forContentRect:)` adds no chrome, so
`chromeHeight(for:)` is 0 and the frame height equals the applied content height.
`NSHostingView` still lays SwiftUI out inside the window's **safe area**, which excludes
the titlebar strip (about 32 pt on macOS 26). The view then adds its own
`CapturePanelLayout.titlebarDragInset` (28 pt) inside that region. The measured metrics
include the 28 pt drag inset but not the titlebar safe-area inset. SwiftUI therefore
always gets about 32 pt less height than the metrics asked for, and the only
compressible child, the auxiliary `ScrollView` (layout priority 0), absorbs the
shortfall.

Pixel measurements from the screenshot (2× Retina) match this exactly:

- The frame is 275 pt tall. The editor's top edge is 60 pt below the window top: a 32 pt
  safe area plus the 28 pt drag inset.
- Measured metrics: 28 inset + 42 editor + 12 spacing + auxiliary + 12 spacing + 25
  footer + 18 padding = 137 + auxiliary. A 275 pt frame therefore means the view
  reported an auxiliary height of about 138 pt. That equals the destination line (15),
  plus 12 spacing, plus the pinned card (112).
- The visible auxiliary viewport is 106–107 pt, which is 138 − 32. The card's bottom 32
  pt fall below the viewport, which is why its bottom edge is square.

When no auxiliary region is shown, the same deficit overflows the fixed editor/footer
stack instead and eats most of the 18 pt bottom padding. The earlier "footer cut off"
report that `a20055e fix(capture): keep panel actions visible while autosizing`
addressed was very likely the same missing inset.

Fixing only layer 1 would leave the bottom 32 pt of every preview hidden. Fixing only
layer 2 would still pin the card at 92 pt. Both fixes are required.

## Design

### A. Let the preview card take its natural height

In `PreviewPane`:

- Remove `idealHeight`. Keep a floor so a tiny preview or the loading spinner does not
  collapse the card. Rename `CapturePanelLayout.previewIdealHeight` to
  `previewMinimumHeight` (value unchanged: 92) so the name says it is a floor, not a
  target.
- Use `alignment: .topLeading` so that any unexpected overflow can never hide the
  header, and add `.fixedSize(horizontal: false, vertical: true)` so the pane is always
  laid out at its natural height, even if it is ever hosted outside the scroll view.
- Keep the `thinMaterial` background, padding, corner radius, and every existing
  per-item presentation. That includes deliberate horizontal `lineLimit(1)` truncation,
  the close/start card's `maxVisibleTaskRows` cap with its `+N more` line, the failure
  text's `lineLimit(3)`, and the destination summary's `lineLimit(2)`. The goal is to
  stop clipping content that is already meant to be visible, not to redesign the cards.

### B. Keep the card steady while a live preview reloads

Every keystroke sets `previewState = .loading` (see `CapturePanelModel`'s analysis path,
which debounces for 50 ms and then runs `bob`). This replaces the rendered card with a
`ProgressView`. Once the card can be taller than 92 pt, it would otherwise collapse to
the floor and regrow on every keystroke, visibly resizing the window each time. Prevent
this:

- Add a small pure helper next to the other height policies in `CapturePanelView.swift`,
  e.g. `CapturePreviewPaneHeightPolicy`. Given the preview state kind and the last
  _settled_ pane height, it returns the pane's minimum height:
  - `.ready` / `.failed`: `previewMinimumHeight`.
  - `.loading`: `max(previewMinimumHeight, settledHeight)`, with no settled height
    meaning the floor.
- Inside `PreviewPane`, keep the settled height as `@State`. Record it with
  `onGeometryChange` on the same element that carries the min-height frame, and only
  while the state is `.ready` or `.failed`. Never record it while loading, so the height
  cannot ratchet upward. When the next `.ready` arrives, the floor drops back to 92 and
  the card shrinks or grows once to the new result's natural height.
- `PreviewPane` is only in the hierarchy while `previewState != .idle`, so leaving the
  preview (empty draft, submit, discard, re-presentation) discards the retained height
  naturally. Do not publish this height through the model.
- Do not change the model's behavior of clearing `previewResults` (and therefore the
  destination summary line) during loading. That existing one-line flicker predates this
  change and is out of scope.

### C. Count the titlebar safe-area inset in the content height

The inset comes from AppKit, not a hard-coded constant. It must be the top safe-area
inset that the hosting view actually imposes on SwiftUI: the panel content view's
`safeAreaInsets.top`, or equivalently the part of the content rect above
`panel.contentLayoutRect`.

- **Controller.** Publish that inset to the model exactly the way
  `availableScreenHeight` is published today. Add a
  `@Published var titlebarSafeAreaInset: CGFloat` (name at your discretion) to
  `CapturePanelModel` and assign it only when the value actually changes, to avoid a
  metrics feedback loop. Refresh it everywhere `updateAvailableScreenHeight` runs: after
  the hosting view is installed in `makePanelIfNeeded()`, in
  `replayLatestContentMetricsForPresentation()`, in `applyContentMetrics`, and in
  `windowDidChangeScreen`. Folding both updates into one helper is fine. The value
  depends only on the window's style and screen, never on the applied height, so it
  cannot oscillate.
- **Height policy.** Add a `safeAreaTopInset: CGFloat = 0` stored property to
  `CapturePanelContentHeightPolicy` and include it in `nonEditorChromeHeight(...)`.
  Because `metrics(...)` and `CaptureEditorHeightBudget` both go through
  `nonEditorChromeHeight`, the inset then reaches the ideal height, the persistent
  minimum, and the editor's screen budget together. Sanitize it like the other heights
  (non-finite or negative becomes 0).
- **View.** In `CapturePanelView`, build the policy with the model's inset at both call
  sites (`editorHeightPolicy` and `reportContentMetrics()`), and call
  `reportContentMetrics()` from an `.onChange` of the inset, just as the view already
  does for `availableScreenHeight`.
- Leave `CapturePanelWindowSizer`, `chromeHeight(for:)`, `contentMinSize` and
  `contentMaxSize` pinning, `windowWillResize`, top-edge anchoring, and screen clamping
  unchanged. With a full-size content view, the applied "content height" is the whole
  content view, which is exactly what the corrected metrics now describe. The existing
  screen clamp (visible frame minus 2 × 24 pt margins) therefore still bounds the real
  frame, and when it binds, the auxiliary region scrolls as designed. That is the "when
  possible" limit.

Expected visible result: in every state the panel is about 32 pt taller than today, the
footer regains its full 18 pt bottom padding, and preview cards render completely. The
empty band between the traffic lights and the editor (safe area plus drag inset) is
unchanged. Tightening it by letting content extend under the titlebar would be a
separate visual-design decision and is a non-goal here.

## Implementation steps

1. `CapturePanelView.swift`: rename `previewIdealHeight` to `previewMinimumHeight`. Add
   `CapturePreviewPaneHeightPolicy`. Rework `PreviewPane`'s frame as in Design A/B, and
   make `PreviewPane` internal instead of `private` so tests can host it.
   `ActiveTaskPickerCard` is already internal for the same reason.
2. `CapturePanelView.swift`: add `safeAreaTopInset` to
   `CapturePanelContentHeightPolicy`, include it in `nonEditorChromeHeight`, and thread
   the model's inset into both policy constructions. Report metrics when it changes.
3. `CapturePanelModel.swift`: add the published inset property, defaulting to 0,
   documented next to `availableScreenHeight`.
4. `CapturePanelController.swift`: read the content view's top safe-area inset and
   publish it on change at the call sites listed in Design C.
5. `README.md`: in the popup-sizing bullet (the paragraph beginning "A fresh popup is a
   compact, Spotlight-like bar"), state that measured heights include the titlebar
   safe-area inset of the full-size-content panel. In the preview paragraph ("The outer
   auxiliary detail region owns scrolling …"), state that the preview card always takes
   its natural height, that the window grows to show it up to the screen limit before
   the auxiliary region scrolls, and that the card keeps its last rendered height while
   a live preview reloads.

## Tests

Add tests to `Tests/BobMacCaptureTests/` (e.g. `BobMacCaptureTests.swift` next to the
existing content-height tests, or a new focused file). Do not weaken any existing
autosize, sizer, stash, or active-task-picker test. They build
`CapturePanelContentHeightPolicy(displayScale:)` with the default inset of 0 and should
pass unchanged.

- **Policy includes the inset.** With `safeAreaTopInset: 32`, the empty and auxiliary
  metrics' `idealContentHeight` and `minimumVisibleContentHeight` are each exactly 32
  larger than with 0. `nonEditorChromeHeight` grows by 32. A `CaptureEditorHeightBudget`
  built with that policy has a `maximumHeight` 32 lower on a tall screen, still floored
  at the one-line editor minimum on a tiny screen. A non-finite or negative inset is
  treated as 0.
- **Preview pane takes its natural height (the core regression).** Decode
  `Tests/Fixtures/pomodoro-close-worked.json` with the existing `closeSuccessFixture`
  pattern. Set `model.previewState = .ready(close)` (plus `previewResult` and
  `previewResults`, as `testEditingDraftClearsStaleClosePreviewAndFooterAction` does).
  Host `PreviewPane(model:)` at a realistic panel content width (e.g. 724 pt) in an
  `NSHostingView`. Assert that its fitting height is strictly greater than
  `previewMinimumHeight + 20`: the card has a header, a destination row, several task
  rows, and work-log lines. This assertion fails on the old 92 pt pinned frame. Also
  assert that a fresh pane in `.loading` fits at exactly the floor plus padding (within
  0.5 pt at the display scale).
- **Loading hold policy.** Unit-test `CapturePreviewPaneHeightPolicy`:
  - loading with no settled height returns the floor;
  - loading with a settled height of 180 returns 180;
  - loading with a settled height below the floor returns the floor;
  - ready and failed always return the floor, whatever the settled height.

  If a hosted end-to-end check of the hold proves reliable in CI (ready, then loading,
  in the same hosting view, pumping the run loop briefly between steps as
  `BlockIDFieldFocusTests` does), add it too. The pure policy test is the durable
  contract.

- **Controller publishes the inset.** After `controller.makePanelIfNeeded()`, the
  model's published inset equals `panel.contentView?.safeAreaInsets.top` (within 0.5
  pt), and that value is greater than 0 for the titled full-size-content panel. If CI
  reports 0 here, the layer-2 diagnosis is wrong for that environment. Investigate and
  report it rather than deleting the assertion.

Do not assert against private SwiftUI view class names.

## Verification

- This Linux host has no Swift toolchain, so the macOS CI workflow is the build and test
  gate for this repo (`.github/workflows/ci.yml`: swift-format lint, SwiftPM build,
  tests, bundle, launch smoke test on `macos-26`). Keep new code swift-format clean and
  consistent with the surrounding style: four-space indentation, the existing
  `@available(macOS 26.0, *)` annotations, and doc comments in the style of the other
  `*HeightPolicy` types.
- After the change is committed and pushed, watch the latest `CI` run for
  `bobs-org/bob-mac-capture` (`gh run list -R bobs-org/bob-mac-capture -L 1`, then
  `gh run watch <id> -R bobs-org/bob-mac-capture`). Use `/sase_monitor` for the wait
  rather than blocking. Fix any compile, lint, or test failure in follow-up commits
  until CI is green.
- Manual check (for the user on the Mac, after `just install`): reproduce the screenshot
  with an `=x` draft against a Pomodoro that has 3 or more started tasks, and confirm
  all of the following:
  - the `Close …` header and every task row are fully visible with rounded corners at
    both ends of the card;
  - the footer keeps its bottom padding;
  - typing does not make the card collapse and regrow;
  - a very long batch preview grows the window to the screen limit and then scrolls only
    the auxiliary region, with the editor and footer staying put.
