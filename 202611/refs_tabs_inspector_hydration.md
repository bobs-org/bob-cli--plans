---
tier: tale
title: Hydrate the Bob Refs inspector for the tab Command-T selects
goal:
  Pressing Command-T in Highlights always loads the selected tab's thumbnail and
  annotations in the Bob Refs inspector, with no panel-switch round trip.
size: small
proposed_by: bbugyi200.athena.17j
proposed_with: claude/opus
create_time: 2026-11-03 12:59:34
status: wip
---

# Plan: Fix Bob Refs inspector never hydrating after Command-T from Highlights

## Symptom

When Bryan presses Command-T in Highlights to open Bob Refs in Tabs, the inspector
sometimes shows only the list-item facts: a grey thumbnail box, the `Pages`/`Notes`
counts, and no HIGHLIGHTS feed. There is no "Notes unavailable" line either, so
`RefsInspectorContent` for the selected row is still `nil`; the load never ran. A
panel-switch round trip to Capture and back fills it in. (Bryan described the key as
Ctrl+Enter. In code the panel switch is Ctrl+Tab, `RefsKeyRouter.isPanelSwitch`; either
way it is the same `BobPanelCoordinator.switchPanels` path.)

## Root cause (all paths are in the `bob-mac-capture` linked repo)

Inspector hydration is requested in exactly two places:

1. `RefsPanelModel.prepareForPresentation(source:)` ends with `inspectorRequested()`
   (`Sources/BobMacCapture/Refs/RefsPanelModel.swift`, ~line 1009).
2. A SwiftUI `.onChange(of: selected) { model.inspectorRequested() }` attached to
   `RefsInspectorView` inside `RefsPanelView.inspector(layout:)`
   (`Sources/BobMacCapture/Refs/RefsPanelView.swift:263`).

`selectedID`'s `didSet` (`RefsPanelModel.swift` ~lines 413-439) never requests
hydration. The model relies on the view's `.onChange` for every selection change made
after `prepareForPresentation`.

The Command-T flow selects the current tab after `prepareForPresentation` returns:

- `prepareForPresentation(.highlightsTabs)` → `resetToDefaults(tabs: true)` →
  `rebuildTabsListing(preservingSelection: false)` builds the listing from the
  **previous** Tabs read (`matchedTabs` survives hides) and preselects that read's
  current tab (or nothing on the first Command-T after launch). Then `beginTabsRefresh`
  starts an async AX read, and `inspectorRequested()` hydrates that stale preselection
  or cancels when nothing is selected.
- When the AX read lands, `applyTabsReadResult(.snapshot)` sets `tabsState = .loaded`
  and, through `rebuildTabsListing`, sets `selectedID` to the tab that is current now.

`RefsPanelView.tabsContent` (~lines 133-170) renders `listContent()` from separate
`switch model.tabsState` cases. `.loading` shows `listContent()` only when the listing
is non-empty and `RefsTabsDelayedSkeleton` when it is empty. `.off, .loaded` uses its
own `listContent()`. SwiftUI gives each case its own identity, so moving from `.loading`
to `.loaded` tears down the whole column subtree and builds a new one. The new
`.onChange(of: selected)` records the new selection as its initial value and never
fires. Nothing requests hydration for the row the snapshot selected.

That explains the "sometimes":

- **Works** when the current Highlights tab is the same as the one at the previous Tabs
  read. The stale preselection is the right row, and step 1 already hydrated it.
- **Fails** on the first Command-T after launch (empty listing → skeleton → list), and
  whenever Bryan has changed tabs in Highlights since the last Tabs read (stale row A is
  hydrated, then the snapshot selects B during the `.loading → .loaded` rebuild).
- **The panel-switch round trip fixes it** because
  `prepareForPresentation(.panelSwitch)` → `restoreTabsForPresentation()` keeps B
  selected and calls `inspectorRequested()` again.

The same gap also breaks the Command-T "focus the current page's notes" behavior
(`tryFireHighlightsKeyFocus` runs from the `inspectorLoader.$content` sink, and content
never arrives). The bug has been latent since Tabs rendering landed (`1b7ead4`,
2026-10-16). `461f46c` (fire-time visibility gate) did not cause it.

## Fix: the model owns selection-driven hydration

Hydration must not depend on view identity. Move the request into the model, where every
`selectedID` mutation already goes (the property is `private(set)`).

### 1. `Sources/BobMacCapture/Refs/RefsPanelModel.swift`

- In `selectedID`'s `didSet`, inside the existing `if oldValue != selectedID` block,
  call `inspectorRequested()`. Put it **before** `updateHighlightsKeyFocusArming()` /
  `tryFireHighlightsKeyFocus()`. Ordering does not affect correctness, because the focus
  also fires from the `$content` sink, but requesting first reads naturally. Add a short
  comment: the inspector subtree is rebuilt when `tabsState` changes case (and when the
  Tabs skeleton gives way to the list), so a view-level `onChange` misses selections
  that land with a Tabs read. The model therefore requests hydration on every selection
  change, and the loader's 120 ms settle plus cancel-previous behavior still debounces
  rapid movement.
- Leave the explicit `inspectorRequested()` at the end of `prepareForPresentation` in
  place. A re-show with an unchanged selection still needs it (content may have been
  evicted, or the selection settled while hidden). The duplicate request when prepare
  also changes the selection is harmless: `RefsInspectorLoader.request` cancels the
  previous scheduled load, and `ref show` is single-flight and cached.
- Safety notes the implementer should confirm rather than re-derive:
  - Hidden-panel selection changes still hydrate nothing.
    `inspectorLoader.shouldHydrate` checks `panelIsVisible()` at fire time
    (`RefsInspectorLoader.load`).
  - `installForPreviews` sets `selectedID` and then calls `installContentForPreviews` or
    `reset()`, and both cancel the scheduled load. Preview-pinned ids return early in
    `schedule`.
  - A `nil` selection calls `inspectorLoader.cancel()`, which is correct.
  - Swift does not run `didSet` for the assignment in `init`, which happens before
    `inspectorLoader` is constructed.

### 2. `Sources/BobMacCapture/Refs/RefsPanelView.swift`

Delete the now-redundant
`.onChange(of: selected) { _, _ in model.inspectorRequested() }` (line ~263). Keep
`.id(...)` and `.transition` unchanged. This makes the model the single owner of
hydration requests, so a future view restructure cannot bring the bug back.

### 3. Regression tests: `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`

Add two model tests next to `testHighlightsKeySelectsCurrentTab`. Use the existing
helpers: `makeHarness(environment:tabsReader:)`, `FakeTabsReader`,
`Self.tabsSnapshot(_:vault:frontmost:)` (index 0 is the current tab),
`showFixtureJSON(path:quote:count:)` with `FAKE_BOB_REFS_SHOW_FIXTURE`, and
`harness.waitForModel`. Model them on
`testVisibleLibraryRefreshRehydratesInspectorInPlaceAndKeepsCopyFrozen`. Write the show
fixture after `waitForSnapshot()` once the target id is known; fake bob reads it on
every call. Set `harness.model.panelIsVisible = { true }` **before**
`prepareForPresentation`. Neither test may call `harness.model.inspectorRequested()`
after the snapshot lands; that call is exactly what the bug was missing.

- `testHighlightsTabsShowHydratesSnapshotSelectedRowOnFirstShow`: first Command-T with
  no previous Tabs read. Set the reader's next result to a snapshot whose current tab is
  `items[0]`, then call `prepareForPresentation(source: .highlightsTabs)`. Assert that
  `selectedID` was `nil` right after prepare (an empty listing), which proves the
  selection came from the async read. Wait for `tabsState == .loaded` and
  `selectedID == items[0].id`, then wait for
  `inspectorContent(for: items[0].id)?.highlights.first?.quote` to equal the fixture's
  first quote.
- `testHighlightsTabsShowHydratesTabSelectedAfterStalePreselection`: the "changed tabs
  since last time" case. Run a first Command-T with `items[0]` current and wait for
  `.loaded`. Then call `panelDidHide()` and `reader.setReadDelay(...)` (around 200 ms,
  so the second read lands after prepare's synchronous rebuild). Set the next snapshot
  to `[items[1], items[0]]` so `items[1]` is current, and write the show fixture for
  `items[1].id`. Call `prepareForPresentation(source: .highlightsTabs)` and assert the
  synchronous selection is the stale `items[0].id`. Wait for `selectedID == items[1].id`
  and `.loaded`, then wait for `items[1]`'s fixture quote in `inspectorContent(for:)`.

Both tests must fail on the old code: the snapshot-selected row's content stays `nil`,
so `waitForModel` times out. They must pass after step 1. Check that existing tests stay
green, especially those that set `panelIsVisible = { true }`; the extra debounced
requests only add cached or single-flight `ref show` calls, and no test asserts an exact
`ref show` count on the model path. Also check that
`testHiddenLibraryRefreshDoesNotHydrateInspector` still passes, since hidden requests
are gated at fire time.

## Out of scope

- No change to bob-cli, `ref show`, the loader's settle/cache policy, or the Tabs
  ranking and preselection.
- No docs change. `docs/refs-inspector.md` already says hydration runs after the
  selection settles for 120 ms, cancelling on every move; the fix makes the code match
  that description.

## Verification

Follow `bob-mac-capture/AGENTS.md`. Open the repo with `/sase_repo`
(`sase repo open bob-mac-capture -r "..."`) and work only in the path it prints.

1. `just portable-check` in the bob-mac-capture checkout. Athena has `swift`. This
   parse-lints every Swift file and builds and tests RefsCore/CaptureCore, but it cannot
   type-check `Sources/BobMacCapture*`.
2. Format only the changed lines to `.swift-format` (4-space indentation, 100 columns).
   Never re-indent whole files.
3. Best effort on the mac (see the `tailnet` reference memory for access): rsync a
   scratch copy without `.build`/`.git`, run `./Scripts/bundle.sh` and
   `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
   `just install`. If the mac is offline, say in the final report that the macOS targets
   are unverified. CI runs the XCTest suite.
4. Manual repro for Bryan after install: open PDF A in Highlights, press Command-T, then
   Esc. Switch Highlights to PDF B and press Command-T again. B's thumbnail and
   HIGHLIGHTS feed should appear, scrolled to the current page's notes, without a
   panel-switch round trip. Repeat with the first Command-T after relaunching the app.

Commit the change in the bob-mac-capture repo through the normal SASE final-declaration
flow (it becomes that turn's repository obligation).
