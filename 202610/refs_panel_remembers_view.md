---
tier: tale
title: Bob Refs remembers its last view; Ctrl-U resets it
goal:
  Bob Refs reopens with its last query, filters, selection, and preview scroll, and
  Ctrl-U resets them.
size: medium
decisions:
  scroll_up_key:
    ask:
      Which key should take over the preview's half-page-up scroll, now that Ctrl-U
      resets the panel?
    choices:
      ctrl_y:
        Ctrl-Y scrolls the preview up, next to U; the search field gives up native yank
      ctrl_b:
        Ctrl-B scrolls the preview up like vim's page-back; the search field gives up
        caret-left
      ctrl_shift_u:
        Ctrl-U keeps scrolling the preview up; Ctrl-Shift-U resets the panel instead
    default: ctrl_y
    why:
      Y sits next to U and nothing in Bob Refs uses Ctrl-Y; the Ctrl-U scroll dates from
      4c22d09
    answer: ctrl_shift_u
  esc_closes:
    ask:
      Should Esc close Bob Refs at once, keeping the query, filters, selection, and
      preview scroll?
    choices:
      keep:
        After any card or banner, Esc hides the panel and the next open restores
        everything
      ladder:
        Esc still clears query, Tabs or Newest, then scope first, so an Esc close
        remembers little
    default: keep
    why:
      Capture's Esc keeps its draft; the ladder would wipe the query and selection on
      every Esc
    answer: keep
  tabs_entry:
    ask:
      Should Shift-⌘O (Bob Refs in Tabs) also restore the remembered query, scope,
      selection, and scroll?
    choices:
      restore:
        Shift-⌘O restores the remembered view with Tabs on; an open remembered row stays
        selected
      fresh:
        "Shift-⌘O keeps today's fresh Tabs view: blank query, All, previous tab
        preselected"
    default: restore
    why:
      Shift-⌘O is the usual way in from Highlights; a tab switch still restores the
      preselection
    answer: restore
proposed_by: bbugyi200.athena.12e
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 08:14:58
status: wip
---

# Bob Refs remembers its last view; Ctrl-U resets it

## Goal

Today, Bob Mac Capture's Bob Refs panel (linked repo `bob-mac-capture`) resets on every
show. The query clears, the scope returns to All, Newest and Tabs turn off, the first
row is selected, the preview jumps to the top, and revealed highlight pages collapse.
Bryan loses his place while reading the notes he left on a reference.

After this change, Bob Refs behaves like the Capture panel: it remembers its last view
in memory and reopens it exactly as he left it.

- **Remembered:** the typed query, the kind scope (⌘1–⌘5: All, Chats, Papers, Articles,
  Docs), Newest order (⌘0), the Tabs filter (⌘T), the selected reference, the revealed
  highlight pages, the preview (inspector) scroll position, and the list scroll position
  (while the selected row stays visible).
- **Ctrl-U** resets all of these to what a fresh launch shows today: blank query, All,
  library sections, Tabs off, first row selected, list and preview at the top, and
  revealed pages collapsed.
- **Lifetime:** in memory for the app's lifetime, like Capture's retained draft. Quit,
  Restart, and reinstall start fresh. Nothing is written to disk.

This reverses two earlier behaviors that each plan chose on purpose:
`plan:202610/refs_preview_scroll_keys.md` ("a fresh presentation starts at the top") and
`plan:202610/refs_inspector_notes_paging.md` (`reveal_memory = reset`). Bryan's request
replaces both. Ctrl-U is already bound in Bob Refs: it scrolls the preview up by half a
page (`RefsKeyRouter`, `docs/refs.md`). The `scroll_up_key` decision moves that scroll
to another key.

Only presentation state changes. The app still never parses or writes the vault, so this
stays within `decisions:mac-capture-is-a-thin-client`. Nothing changes in bob-cli.

## Design

### Show paths

`RefsPanelModel.prepareForPresentation(source:)` still runs on every show. It keeps its
per-show housekeeping:

- bump `presentationCount` (scale-in animation and list reveal)
- clear `selectedRowRect` and `renameStatus`
- return early while a rename or Today link is running
- close the Copy card, and handle the Rename card as today
- clear `heldTabsOpen`, `pendingClosedTab`, `preserveTabsSelectionOnNextSnapshot`,
  `banner`, and `pendingOpen`
- install a pending scan banner, and mark the scan notice seen

It stops resetting the remembered state. Each source behaves like this:

- **`.library`** (global ⌃⇧⌘O, ⌘O takeover in Highlights, menu-bar item): restore.
  - With Tabs off, re-rank from fresh library data with the remembered query, scope, and
    order, and keep the selected id while it is still listed (the
    `freshListingKeepingSelection()` path).
  - With Tabs on, set `tabsState = .loading`, rebuild the Tabs listing, and begin a tab
    re-read, both preserving the selection, unless `tabsSelectionConsumed` (see Tabs
    below) asks for the switcher preselection.
- **`.highlightsTabs`** (Shift-⌘O): see the `tabs_entry` decision.
- **`.scanResults`** (new; the hidden-scan notification click): the default view, the
  same as Ctrl-U. The notification still shows Just scanned on top with its first row
  selected. `AppDelegate`'s `NotificationService(showRefs:)` closure calls
  `panelCoordinator?.showRefs(source: .scanResults)`. That route is used only by the
  refs-scan category.

> [!decision] tabs_entry = restore Restore as for `.library`, then make sure Tabs is on:
>
> - If Tabs was already on, keep the remembered selection (the same rule as `.library`).
> - If Tabs was off, turn it on the same way ⌘T does: Newest off, query and scope kept,
>   then rebuild and re-read without preserving the selection, so the Tabs preselection
>   applies.
>
> Refactor `setTabsFilter(true)` so it can share this path without its banner side
> effect.

> [!decision] tabs_entry = fresh Shift-⌘O runs the reset helper with Tabs on: blank
> query, All, ranked, Tabs on, `rerankSelectingFirst()`, and
> `beginTabsRefresh(preservingSelection: false)`. This is today's Shift-⌘O behavior
> exactly. Whatever the panel shows when it hides becomes the remembered state.

When the remembered selection is no longer listed, selection falls back to
`RefsSelectionPolicy.initial`, and the existing `selectedID` `didSet` clears the
revealed page. A vanished row, a closed tab, or a query that no longer matches all fall
back this way. The preview place (below) must be cleared in the same `didSet`.

Remove the revealed-page clearing (`revealedPageID`/`revealedPage = nil`) from
`prepareForPresentation`. Revealed pages still reset on selection change, on a page-size
change, and on Ctrl-U. At the end of `prepareForPresentation`, call
`inspectorRequested()`. The inspector view no longer remounts on every show, so its
`.onAppear` no longer re-requests hydration. This call keeps new highlights showing up
on reopen.

Search field: the caret lands at the end of the remembered query, not with the text
selected, so typing continues the query.

- In `RefsPanelController.focusFilterField()`, after both the immediate focus claim and
  the 50 ms retry claim, collapse the field editor's selection to the end of its string.
- Do this only for the Refs filter field. Rename and Today-link focus is unchanged.

### Ctrl-U reset

- Add `RefsCommand.resetView` and a private `resetToDefaults(tabs: Bool)` helper. Ctrl-U
  (`.resetView`), the `.scanResults` show, and the `fresh` branch of `tabs_entry` all
  share the helper.
- The helper sets:
  - `query = ""`, `scope = .all`, `listOrder = .ranked`
  - Tabs off through `leaveTabsFilter()`, or on for the `fresh` Shift-⌘O
  - `banner = nil`, `pendingOpen = nil`, `heldTabsOpen = false`
  - `tabsSelectionConsumed = false`
  - clears the revealed page and the preview place
  - bumps `inspectorGeneration`
  - calls `rerankSelectingFirst()`, plus `beginTabsRefresh(preservingSelection: false)`
    when Tabs is on
- `.resetView` is always consumed, even when the view is already at its defaults. In
  that case it still returns the list and preview to the top.
- While a rename or Today-link write runs, `.resetView` is a consumed no-op. An open
  card already receives keys first: Copy consumes every key, and the Rename and
  Today-link fields pass Ctrl-U through to text editing. So Ctrl-U acts only on the bare
  panel.
- Post a VoiceOver announcement: "Bob Refs reset".
- The router maps exact Control-U to `.resetView`, except while IME-composing, when it
  passes through like Ctrl-D and Ctrl-X.

> [!decision] scroll_up_key = ctrl_y `KeyCode.y` keeps ⌘Y (Copy card). Exact Control-Y
> maps to `.scrollInspector(.up)`, except while IME-composing. Control-U maps to
> `.resetView`.

> [!decision] scroll_up_key = ctrl_b Add `KeyCode.b = 11`. Exact Control-B maps to
> `.scrollInspector(.up)`, except while IME-composing. Control-U maps to `.resetView`.

> [!decision] scroll_up_key = ctrl_shift_u Control-U stays `.scrollInspector(.up)`.
> Exactly `[.control, .shift]` + U maps to `.resetView`, except while IME-composing.
> Every "Ctrl-U" reset statement in this plan and in the docs reads Ctrl-Shift-U
> instead.

The scroll-up key keeps every Ctrl-U scroll rule from `4c22d09` and the notes-paging
work: half a viewport, consumed at the edges and when the preview is hidden or absent,
search keeps focus, and it cancels a pending reveal follow (`scrollInspector(.up)`
already calls `cancelRevealFollow()`).

### Esc

> [!decision] esc_closes = keep The `.escape` ladder becomes: Today-link card (unless
> linking), Rename card (unless renaming), Copy card, banner, then hide. Esc no longer
> clears the query, Tabs, Newest, or scope, so closing with Esc keeps the whole view,
> the way Capture's Esc keeps the draft. `.deleteBackwardOnEmpty` (Backspace on an empty
> query removes Tabs or Newest, then the scope) is unchanged. Ctrl-U is the full reset.

> [!decision] esc_closes = ladder `.escape` is unchanged. Only closing with the hotkey
> toggle, a click elsewhere, opening a reference, copying, or a tab switch keeps a
> non-default view. The docs must say so.

### Tabs

- A Tabs show keeps the remembered row while its tab is still open.
- After a successful switch from Bob Refs (`selectOpenTab` → `.selected`), set a private
  `tabsSelectionConsumed` flag. The next Tabs show then rebuilds and re-reads without
  preserving the selection, so the switcher preselection returns (the previous tab when
  Highlights is frontmost). This avoids a selected row that only "switches" to the tab
  already in front. The show that reads the flag clears it, and so does the reset
  helper.
- ⌘T, Esc, and Backspace behavior for Tabs is otherwise unchanged.

### Preview scroll memory

The inspector is keyed `.id("\(selected)#\(model.presentationCount)")` in
`Sources/BobMacCapture/Refs/RefsPanelView.swift`, which remounts it, at the top, on
every show.

- Change the key to `.id("\(selected)#\(model.inspectorGeneration)")`, where
  `inspectorGeneration` is a new `@Published` counter on the model. Only the reset
  helper bumps it.
- The `NSHostingView` persists across `orderOut`/`orderFront`, so a plain hide and show
  keeps the native scroll offset.

The preview also remounts in three other cases:

- A Copy, Rename, or Today-link card replaces it, including ⌘Y → copy → hide.
  `panelDidHide` closes the Copy card, so the inspector is rebuilt while hidden.
- A Tabs read flips `tabsContent` between its `.loading` and `.loaded` branches, which
  are different view identities.
- Any branch change in `RefsPanelView.content`.

To survive these, the model remembers a place, and each new preview instance restores it
once.

**Model**

- Add a private place, `(id: String, generation: Int, offset: CGFloat)?`, where `offset`
  is measured from the content top.
- Add `noteInspectorOffset(id:generation:offset:)`. It ignores reports whose id is not
  the selected id or whose generation is not the current one, so a fading-out old
  instance cannot overwrite the place.
- Add `inspectorRestoreOffset(for id:) -> CGFloat?`. It returns the stored offset when
  the id and generation match and the offset is above 0.5, and nil otherwise.
- Clear the place in the `selectedID` `didSet` when the id changes, and in the reset
  helper.
- Keep the place out of `@Published`, so scrolling never republishes the panel model.

**View** (`RefsInspectorView.swift`)

- `RefsInspectorView` gains `restoreOffset: CGFloat? = nil` and
  `onOffsetChange: (CGFloat) -> Void = { _ in }`.
- `RefsPanelView` passes `model.inspectorRestoreOffset(for: selected)` and a closure
  that captures the current generation and calls `model.noteInspectorOffset`.
- The view hands both, plus `contentIsFinal: inspector != nil`, to the existing
  `RefsInspectorScrollAnchor` representable.

**Anchor** (`RefsInspectorScrollAnchorView`)

The anchor already sits as the column's background inside the scroll view. Its own frame
tracks the document height, and its `enclosingScrollView` is the preview's
`NSScrollView`.

- `makeNSView` takes `restoreOffset` as a one-shot pending restore. `updateNSView`
  refreshes `contentIsFinal` and the callback, and never re-arms a restore.
- Restore attempts run from `viewDidMoveToWindow`, `setFrameSize`/`layout`, and an async
  hop after `contentIsFinal` flips. Each attempt asks a new pure helper on
  `RefsInspectorScrollGeometry`:

  ```swift
  /// The offset to restore once layout allows: the clamped remembered offset
  /// when the content is final or already tall enough, nil while waiting,
  /// and nil for invalid geometry.
  func restoreTarget(offset: CGFloat, contentIsFinal: Bool) -> CGFloat?
  ```

  On a non-nil target, scroll the clip view (flipped-aware, like `setInspectorOffset`)
  and resolve the restore.

- A live user scroll (`NSScrollView.willStartLiveScrollNotification`) resolves the
  restore without applying it. So does a keyboard scroll:
  `RefsPanelController. scrollInspector(_:)` first finds the anchor (add a finder beside
  `findInspectorScrollView`) and calls `cancelPendingRestore()`.
- **Recording:** observe the clip view's bounds-changed notifications (make sure
  `postsBoundsChangedNotifications` is on), and report the top-relative offset through
  the callback. Report only after this instance's restore has resolved, so a new
  instance's initial zero offset never overwrites the remembered place.
- Factor the flipped-aware offset math that `measureInspectorScroll()` uses into one
  shared helper, so the controller and the anchor measure identically.
- Remove the observers in `viewWillMove(toWindow: nil)` and `deinit`.
- Design fixtures (`previewMode`) have no anchor and are unaffected.

Keyboard scrolls, reveal follows, trackpad scrolls, and same-reference hydration all
change the clip bounds, so they all keep the place current. Hydration does not clear
loader content, so the offset does not collapse while revalidating.

## Implementation

Open the linked repo with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`), then
read its `AGENTS.md`.

1. **`Sources/BobMacCapture/Refs/RefsPanelModel.swift`**
   - Add `RefsPresentationSource.scanResults` and `RefsCommand.resetView`, each with a
     doc comment.
   - Add the `inspectorGeneration` counter, the place API, the `tabsSelectionConsumed`
     flag, and `resetToDefaults(tabs:)`.
   - Rewrite `prepareForPresentation(source:)` and its doc comment as described above.
   - Update the `.escape` case per `esc_closes`.
   - Set the flag in `selectOpenTab`, and clear the place in the `selectedID` `didSet`.
   - Update the class doc comment, which lists when re-ranking happens.
2. **`Sources/BobMacCapture/Refs/RefsKeyRouter.swift`**: route Ctrl-U and the scroll-up
   key per `scroll_up_key`, and update the doc comments.
3. **`Sources/BobMacCapture/Refs/RefsInspectorScroll.swift`**: add `restoreTarget` and
   the shared offset-measure helper. Update the direction doc comment ("Control-D is
   down, Control-U is up").
4. **`Sources/BobMacCapture/Refs/RefsInspectorView.swift`**: make the anchor restore and
   record, and add the new view parameters.
5. **`Sources/BobMacCapture/Refs/RefsPanelView.swift`**: change the inspector `.id`, and
   pass the restore offset and offset callback.
6. **`Sources/BobMacCapture/Refs/RefsPanel.swift`**
   - Make `focusFilterField()` leave the caret at the end of the query.
   - Make `scrollInspector(_:)` cancel a pending restore.
   - Rewrite the `show()` doc comment ("resets the model" is no longer true).
7. **`Sources/BobMacCapture/AppDelegate.swift`**: route the scan notification to
   `showRefs(source: .scanResults)`.

## Tests

Existing tests that assume a reset on every show must change. `prepareForPresentation`
appears about 120 times in `Tests/`. Audit every call site:

- Tests that use it only as setup on a fresh model still pass.
- Tests that change the view and then expect a blank view after presenting must do one
  of these:
  - call `perform(.resetView)`
  - present with `.scanResults`
  - assert the restored state instead

Known cases:

- `RefsPanelModelTests`: `testTabsAndLibraryPresentationSourcesResetTheFilter`,
  `testNewestEscapeAndBackspaceAndPresentationReset`, `testEscChainOrder`,
  `testTabsToggleKeepsQueryAndScopeAndEscapePeelsTabsBeforeScope` (`esc_closes`), and
  the revealed-page reset tests near `revealMoreInspectorNotes`.
- `RefsListScrollHostedTests`: `testHostedShowAfterScrollingDownReturnsToTop` (rewrite
  it as "show keeps the last row selected and visible", and add a Ctrl-U-returns-to-top
  test) and `testHostedGrownContentAboveTopStillShowsTop` (present with `.scanResults`).
- `RefsKeyRouterTests`: the Ctrl-U scroll expectations.

New coverage:

- **Router** (`RefsKeyRouterTests`)
  - Ctrl-U → `.resetView`, and nil while IME-composing.
  - The `scroll_up_key` key → `.scrollInspector(.up)`.
  - Neighboring modifier combinations stay unbound or keep their old meaning (⌘Y still
    opens Copy).
- **Model** (`RefsPanelModelTests`)
  - Show restores the query, scope, Newest, Tabs, and selection after fresh data
    re-ranks the list. When the row has vanished, the first row is selected and the
    revealed page and place are cleared.
  - The revealed page survives a show.
  - `.resetView` restores every default from every non-default combination, bumps
    `inspectorGeneration`, clears the banner and `pendingOpen`, is consumed at the
    defaults, and is a no-op while renaming or linking.
  - `.scanResults` selects Just scanned's first row.
  - `tabs_entry` behaves as chosen.
  - The tab-switch flag restores the preselection exactly once.
  - Esc behaves per `esc_closes`.
  - `noteInspectorOffset` ignores stale ids and generations.
- **Hosted** (`RefsInspectorScrollTests`)
  - Scroll the preview down with Ctrl-D, then `controller.hide()` and `show()`: same
    offset (±1 pt). The search field holds focus with the caret at the end of the query.
  - Reveal a second page, scroll into it, then hide and show: the page and offset
    survive.
  - Open the Copy card, then Esc: the offset is restored.
  - ⌘Y, then the panel hides through the copy path, then show: the offset is restored.
  - Ctrl-U: offset 0, the first row selected, and the query empty.
  - The new scroll-up key scrolls up.
  - Under `ctrl_y` or `ctrl_b`, Ctrl-U never scrolls.
- **Pure:** `restoreTarget` cases. Cover waiting, final-and-short (clamps), tall enough
  before final, invalid geometry, and insets.
- **Entry:** a `RefsEntryPointsTests` or `NotificationServiceTests` case that shows the
  scan route reaching `.scanResults`.

Async tests must never await an untimed continuation or gate (repo `AGENTS.md`). Use the
existing bounded `wait…` helpers.

## Docs (bob-mac-capture `docs/`, mounted into Bob's site)

- **`docs/refs.md`**
  - In "Updates", describe what `RefsPanelModel` remembers and that a show restores it.
  - In "The panel", replace "every show, and every re-rank that selects the first row,
    returns the list to its top" with the new list rule.
  - In the keyboard table, add the Ctrl-U reset row and the scroll-up row, and update
    the Esc row per `esc_closes`.
  - Update the IME paragraph that names Ctrl-U.
  - In "Opening Bob Refs", say every open restores the last view, Ctrl-U resets it, the
    scan notification opens the default view with Just scanned on top, and describe
    Shift-⌘O per `tabs_entry`.
  - In "Tabs", add the remembered-row and post-switch preselection rule.
- **`docs/refs-inspector.md`**: change "Revealed pages reset on every selection change
  and on every show" to the new rule, and add that the preview keeps its scroll position
  across shows and cards.
- **`docs/keyboard.md`**: in "Bob Refs shortcuts", add Ctrl-U and the scroll-up key.
- Grep `docs/`, `README.md`, and Swift doc comments for any remaining "every show" or
  "Ctrl-U scrolls" claims, and fix them. bob-cli's own `docs/` needs no change.

## Verification

Never land Swift that has not compiled.

- In bob-mac-capture, run `just portable-check`. `swift` is available on athena.
- Run `just docs-check`.
- `portable-check` cannot type-check `Sources/BobMacCapture*` or `Tests/BobMacCapture*`.
  Compile a scratch copy on the Mac, as best effort:
  1. Read the `tailnet.md` reference memory for SSH access.
  2. rsync the repo without `.build` or `.git`.
  3. Run `Scripts/bundle.sh` and `just typecheck-tests`.
  4. Delete the copy. Never use Bryan's checkout or `just install` there. If the Mac is
     offline, say in the final report that the macOS targets are unverified.
- After the change lands, check the macOS CI run (Lint, Build, Test, Bundle). A red
  "Lint Swift formatting" step skips the later steps, so read past it. Fix any failure
  this change caused.

## Out of scope

- Persisting the view to disk across Quit, Restart, or reinstall.
- Per-reference reading positions. Only the selected reference's place is remembered;
  selecting another reference starts it at the top, as today.
- Carrying the place across a rename.
- A footer hint for Ctrl-U.
- Any bob-cli, vault, or memory change.
