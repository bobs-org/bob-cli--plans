---
tier: tale
title: Reset Bob Refs when the Highlights Tabs key opens it
goal: Command-T in Highlights opens Bob Refs in Tabs from fresh defaults, exactly
  as if Ctrl-Z had been pressed, with Tabs on.
size: small
decisions:
  in_panel_toggle:
    ask: Should Command-T pressed inside an already-visible Bob Refs also reset the
      view when it turns Tabs on?
    choices:
      open_only: Only opening Refs from Highlights resets; in-panel Command-T keeps
        filtering tabs by the query
      also_in_panel: Turning Tabs on inside a visible Refs also resets like Ctrl-Z;
        filtering tabs by query goes away
    default: open_only
    why: the screenshot bug is the open path; keyboard.md documents the in-panel toggle
      keeping the query
    answer: open_only
proposed_by: bbugyi200.athena.12s
proposed_with: claude/opus
decided_by: auto
status: done
---

# Plan: Reset Bob Refs when the Highlights Tabs key (Command-T) opens it

All work is in the **bob-mac-capture** linked repository. Open it with the `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<why>"`), read its `AGENTS.md`, and make
every edit in the path that command prints. Do not edit bob-cli:
`docs/reference/mac-capture/*` in bob-cli is a gitignored mount of the Mac Capture
`docs/` tree.

## Problem

With Highlights frontmost, Command-T opens Bob Refs in Tabs. The panel comes up with
whatever query, scope, and selection were remembered from earlier shows. Bryan's
screenshot shows a stale query `gas` in Tabs: "No tabs match “gas”" and "0 of 1 tabs".
Clearing it takes an extra Ctrl-Z every time.

Root cause: commit `5935822` ("feat(refs): remember panel view across shows") changed
`RefsPanelModel.prepareForPresentation(source:)`. For `.highlightsTabs` it now restores
the remembered view instead of resetting:

```swift
case .highlightsTabs:
  if tabsFilter {
    restoreTabsForPresentation()      // keeps query, scope, selection
  } else {
    tabsSelectionConsumed = false
    enterTabsFilter(preservingSelection: false)  // keeps query and scope
  }
```

Before that commit, a Tabs show cleared the query, scope, Newest order, and revealed
pages.

## Desired behavior

A show with source `.highlightsTabs` starts from the same state Ctrl-Z
(`RefsCommand.resetView`) produces, except that Tabs is on. That source comes only from
`BobPanelCoordinator.toggleRefsTabs()` when Refs is hidden. It fires for the configured
"In Highlights, show open tabs with" key (Command-T by default, or Shift-Command-O /
Command-E) whether Capture is visible or not. The show should end up with:

- query `""`, scope All, ranked order (Newest off);
- banner, pending open, held Tabs-open, and pending closed-tab dispatch cleared;
- revealed highlight pages and the inspector scroll restore cleared
  (`inspectorGeneration` bump), and the list scrolled to its top (`listResetGeneration`
  bump);
- Tabs on, with the listing rebuilt and the Tabs preselection selected:
  `RefsTabsListing.preselectedID`, the previous tab while Highlights is frontmost,
  otherwise the first tab. Then a fresh tab read starts that does not preserve the
  selection.

These stay as they are:

- A running Rename or Link to Today write still makes `prepareForPresentation` return
  early. Ctrl-Z is also a no-op during those writes.
- A failed rename card is still replayed. `resetToDefaults` does not close it, and the
  `selectedID` `didSet` keeps a failed card.
- Pending new-reference arrivals still wait for the next ordinary `.library` show. Do
  not clear `arrivals` and do not set `visibleArrivalDeadline` on this path.
- Shows from `.library`, `.panelSwitch`, and `.scanResults` behave as before. An
  ordinary library-hotkey show after a Command-T show restores the remembered view,
  which is now the reset Tabs view.
- The closed-tab reopen history (Shift-Command-T) and the Tabs recency order are not
  view state. Leave them alone.

> [!decision] in_panel_toggle = open_only Command-T inside a visible Bob Refs
> (`RefsCommand.toggleTabs`), and the coordinator's visible-library branch
> `setRefsTabsFilter(true)`, keep their documented behavior: they toggle Tabs and keep
> the query and scope.

> [!decision] in_panel_toggle = also_in_panel In `RefsPanelModel.perform(_:)`, when
> `.toggleTabs` would turn Tabs on, and for `.setTabsFilter(true)` while Tabs is off,
> first apply the `isRenaming || isLinkingToday` guard that `.resetView` uses. Then call
> `resetToDefaults(tabs: true, announcement: nil)` instead of `enterTabsFilter`. Turning
> Tabs off still keeps the query and scope. In `RefsPanel`'s key handler, run the same
> panel-level cancel helper from step 2 before dispatching a `.toggleTabs` that turns
> Tabs on. Update `testTabsToggleKeepsQueryAndScopeAndEscapeKeepsTheView` and
> `testNewestAndTabsAreExclusiveAndKeepQueryAndScope` (`RefsPanelModelTests`) to the new
> contract, and add a case where turning Tabs on with a query clears it. Update the
> in-panel Command-T rows and prose in `docs/keyboard.md` and `docs/refs.md` (the
> "Command-T | Toggle the Tabs filter; the query and kind scope stay in place" row, the
> Tabs section's "keeps the current query", and the Updates and entry-point text).

## Implementation

1. **Model: reset on Tabs shows.** In `Sources/BobMacCapture/Refs/RefsPanelModel.swift`,
   inside `prepareForPresentation(source:)`, replace the body of the `.highlightsTabs`
   case with `resetToDefaults(tabs: true, announcement: nil)`. Add a short comment
   saying that the Highlights Tabs key always starts fresh, as after Ctrl-Z, with Tabs
   on, and that pending arrivals stay deferred to the next ordinary show.
   `resetToDefaults(tabs: true)` already exists, but nothing calls it today. It sets the
   defaults, turns Tabs on with `.loading`, rebuilds the Tabs listing without preserving
   the selection, and calls `beginTabsRefresh(preservingSelection: false)`. That covers
   everything `enterTabsFilter(preservingSelection: false)` did. Check that this is
   still true, and that `restoreTabsForPresentation()` and `tabsSelectionConsumed` still
   have users (the `.library`/`.panelSwitch` restore path). Use no announcement: this is
   a fresh open, not a visible reset, so VoiceOver should not say "Bob Refs reset" on
   every Command-T. Update the doc comments that would become false:
   - `RefsPresentationSource.highlightsTabs`: it now shows the current Highlights tabs
     from fresh-launch defaults, as after Ctrl-Z.
   - `prepareForPresentation(source:)`: add the Tabs reset to its summary.

2. **Panel: match Ctrl-Z's panel-level cancels.** In
   `Sources/BobMacCapture/Refs/RefsPanel.swift`, the `.resetView` key path also runs
   `cancelRevealFollow()` and
   `Self.findInspectorScrollAnchor(in: panel?.contentView)?.cancelPendingRestore()`.
   Move those two calls into one small private helper and call it from that key path.
   Also call it from `show(source:)` when `source == .highlightsTabs`, before
   `model.prepareForPresentation(source:)`. On a hidden panel this mostly does nothing,
   because the reveal follow quits when the panel is not visible and the generation bump
   rebuilds the inspector. It keeps the Command-T show exactly in step with Ctrl-Z.

3. **Tests** (`Tests/BobMacCaptureTests/RefsPanelModelTests.swift`):
   - Rework `testTabsAndLibraryPresentationSourcesRestoreTheFilter`, and rename it to
     match. After `setQuery("keep")`, Papers scope, and Newest, a `.highlightsTabs` show
     now gives `tabsFilter == true`, query `""`, scope `.all`, and `.ranked`, with the
     permission state and `readCount == 0` unchanged. A following `.library` show
     restores Tabs on with that empty query. The final toggle-off-then-`.library` part
     keeps working.
   - Add the screenshot regression. Get to a remembered view with Tabs already on and a
     non-empty query (show `.highlightsTabs`, then `setQuery("gas")`), and a non-default
     scope. Then show `.highlightsTabs` again and assert query `""`, scope `.all`,
     `.ranked`, Tabs still on, and the selection equal to `tabsListing?.preselectedID`
     once the snapshot loads. Use `FakeTabsReader` with a trusted snapshot, as the
     nearby Tabs tests do, and wait with `harness.waitForModel`.
   - Add a test that a `.highlightsTabs` show clears the inspector restore offset and
     bumps `inspectorGeneration` and `listResetGeneration`, the way
     `testResetViewRestoresDefaultsAndAlwaysConsumes` checks Ctrl-Z.
   - Update `testHighlightsTabsShowDefersArrivalResetToNextLibraryShow`. After the Tabs
     show, assert query `""` and scope `.all` instead of `"Arrival"` and `.chats`. Keep
     the assertions that the next `.library` show leaves Tabs and selects
     `ref/chat/beta.md`, which proves the arrival is still deferred.
   - Run every other test that calls `prepareForPresentation(source: .highlightsTabs)`
     and fix any that relied on the old query or selection carry-over. Change the test,
     not the new contract.
   - Async tests must never await an untimed continuation or gate. Use the existing
     timed `waitForModel` helpers.

4. **Docs** (Mac Capture `docs/`, mounted into Bob's site):
   - `docs/refs.md`, "Updates" paragraph: say that the Highlights Tabs key (Command-T)
     always opens from fresh-launch defaults, as Ctrl-Z resets, with Tabs on, and still
     defers a new-reference reset to the next ordinary show.
   - `docs/refs.md`, entry-point bullets: replace "The configured Tabs key does the same
     when Tabs is already on; when Tabs is off, it enables Tabs while keeping the query
     and scope, turning Newest off, and selecting the current Highlights tab when
     available." with the reset contract: query, scope, Newest, selection, revealed
     pages, and list and preview positions reset, then Tabs turns on with the Tabs
     preselection selected. Also note the reset in the Command-T (Highlights) bullet.
   - `docs/refs.md`, Tabs section: limit "When Tabs remains on across a show, the
     selected row is kept…" and the one-shot preselection sentence to ordinary and
     Ctrl+Tab shows, and say that Command-T from Highlights always starts fresh.
   - `docs/keyboard.md`, the "Command-T (in Highlights)" row: add "from fresh defaults
     (as after Ctrl-Z)".
   - Keep the wording short, and do not describe the old behavior.

## Verification

- Follow the bob-mac-capture `AGENTS.md`. Never land Swift you have not compiled.
  - Run `just portable-check`, using athena if `swift` is missing locally.
  - `portable-check` cannot type-check the `BobMacCapture` source and test targets, and
    both change here. As a best effort, compile a scratch copy on the mac. Rsync the
    repo without `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`,
    then delete the copy. Never use Bryan's checkout or `just install` there. If the mac
    is offline, say in the final report that the macOS targets are unverified.
- Run `just docs-check` for the `docs/` changes.
- CI runs the XCTest suite. The Mac has no XCTest, so `--build-tests` cannot work there.
- Manual check, for Bryan after install: in Highlights, open Bob Refs with Command-T,
  type `gas`, press Escape, then press Command-T again. The panel shows every open tab
  with an empty search field, and the previous tab is selected.
