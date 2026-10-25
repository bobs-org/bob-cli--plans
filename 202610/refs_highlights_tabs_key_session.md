---
tier: tale
title: "Highlights Tabs key: ephemeral session, focused PDF, current-page preview"
goal:
  The Highlights Tabs key opens Bob Refs on the focused PDF with its preview at the
  current page's notes, and closing that panel never changes what the next ordinary open
  restores.
size: medium
decisions:
  ctrl_tab_round_trip:
    ask:
      Should a Ctrl+Tab round trip out of a Highlights-key session come back to that
      session's Tabs view?
    choices:
      discard:
        Ctrl+Tab away ends the session; switching back shows the view from before the
        Highlights key
      keep:
        Ctrl+Tab back returns to the session's Tabs view; only hiding Refs ends the
        session
    default: discard
    why:
      Leaving Refs never saves a Highlights-key session; Ctrl+Tab restores what the
      hotkeys restore
    answer: discard
  focused_preselect_scope:
    ask:
      Should every Tabs entry, not only the Highlights key, preselect the focused PDF?
    choices:
      takeover_only:
        Only the Highlights-key show selects it; in-panel Command-T keeps the
        previous-tab preselection
      everywhere:
        Every Tabs preselection selects the focused PDF; the previous-tab rule is
        removed
    default: takeover_only
    why:
      The request names the Highlights keymap; the in-panel switcher rule is documented
      behavior
    answer: everywhere
  decision_record:
    ask:
      Should we add a decisions record that a Highlights-key Tabs session is ephemeral
      and opens on the focused page?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.14b
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 14:54:54
status: wip
---

# Highlights Tabs key: ephemeral session, focused PDF, current-page preview

## Problem

The Highlights Tabs key (Command-T by default; Settings › References can change it; it
reaches the model as `RefsPresentationSource.highlightsTabs`) opens Bob Refs filtered to
the PDFs open in Highlights. It has three bugs:

1. **The session leaks into the next open.** The model's live state is the remembered
   view. `RefsPanelModel.prepareForPresentation(source: .highlightsTabs)` calls
   `resetToDefaults(tabs: true)`, which overwrites that state, so the next ordinary show
   (global hotkey, Command-O, menu) restores the Command-T Tabs view instead of what the
   user had before. The test `testHighlightsTabsShowResetsAndLibraryRestoresTheFilter`
   in `Tests/BobMacCaptureTests/RefsPanelModelTests.swift` asserts this old behavior.
2. **The focused PDF is not selected.** `RefsTabsRanker.listing` in
   `Sources/RefsCore/RefsTabsRanking.swift` sets `preselectedID = ids[1]` when
   `highlightsIsFrontmost && ids.count > 1`. That is the app-switcher rule behind the
   docs sentence "When Highlights is frontmost, the previous tab is preselected". The
   takeover key only registers while Highlights is frontmost, so the Highlights key
   always selects the previous tab. Browse order already lists the `isCurrent` row (the
   selected tab of Highlights' main window) first.
3. **The preview does not go to the current page.** The inspector opens at its top with
   only the first notes page (5 by default) revealed. Bob Refs already reads the page
   from the Highlights window title (`name.pdf – Page 5 of 25`,
   `HighlightsWindowTitle.pageCaption` in `Sources/RefsCore/RefsHighlightsTabs.swift`),
   but only as the "Current · p. 5/25" caption string.

## Scope

All code and docs changes go in the **bob-mac-capture** linked repo. Open it with your
`/sase_repo` skill and read its `AGENTS.md` before editing. bob-cli needs no change:
`bob ref show` already returns the `page_label` strings the preview needs. Follow the
thin-client rule (`decisions:mac-capture-is-a-thin-client`): this plan only reads labels
for scroll placement and never writes the vault.

## Design

### 1. A Highlights-key session is never remembered

In `Sources/BobMacCapture/Refs/RefsPanelModel.swift`:

- Add a private `RefsRememberedView` value with these fields: `query`, `scope`,
  `listOrder`, `tabsFilter`, `selectedID`, `revealedPageID` + `revealedPage`, the
  preview offset for the selected id (the `inspectorPlace` offset when it is valid for
  the current `inspectorGeneration`), and the list scroll offset (see below).
- Add a private session value, for example
  `highlightsKeySession: (stash: RefsRememberedView, …)?`.
- **Begin.** In the `.highlightsTabs` case of `prepareForPresentation`, before
  `resetToDefaults(tabs: true, …)`, capture the stash, but only when no session is
  active. Pressing Command-T, Escape, then Command-T again keeps the original
  pre-session stash. The existing early return for a running rename or Today link stays
  first, so a show that keeps a running card never starts a session.
- **End, lazily, on the next presentation.** At the start of every non-`.highlightsTabs`
  `prepareForPresentation`, after the existing early return, restore the stash and clear
  the session. Two exceptions:
  - a show that is replaying a failed rename card keeps the session and stash for a
    later show;
  - `.panelSwitch`, depending on the decision below.

  Then let the existing source logic run unchanged:
  - `.library` re-ranks with fresh data while keeping the restored selection, or resets
    and selects a qualifying arrival as it does today.
  - `.panelSwitch` restores.
  - `.scanResults` resets to defaults anyway, which discards the stash.

  Do **not** restore on hide. `hide()` followed by `represent()` (the open-error re-show
  after `selectOpenTab` fails with "That tab was closed in Highlights.") must re-show
  the session's own view with its banner. Because `represent()` never calls
  `prepareForPresentation`, the lazy restore keeps that working.

- **Restore order matters**, because the `selectedID` `didSet` clears revealed pages and
  `inspectorPlace`:
  1. Assign `query`, `scope`, `listOrder`.
  2. Turn Tabs on or off. Use `leaveTabsFilter()` when the stash has it off. When the
     stash has it on, set it on and let `restoreLastViewForPresentation` re-read Tabs.
  3. Assign `selectedID`.
  4. Assign `revealedPageID` / `revealedPage`.
  5. Bump `inspectorGeneration` so the inspector remounts and applies its one-shot
     restore.
  6. Set `inspectorPlace` to `(selectedID, newGeneration, offset)`.
  7. Request the list offset restore.
- **List offset.** The list keeps its scroll position in view state (`RefsListView`'s
  `ScrollPosition`), so the model needs these additions:
  - `RefsListView`'s `onScrollGeometryChange` action reports the offset to a new
    non-published `model.noteListOffset(_:)`, the same pattern as `noteInspectorOffset`.
  - On restore, the model publishes a list-restore request (generation + offset).
  - `RefsListView` applies the request once the content can reach that offset, or clamps
    the offset when the content is final, the same way `RefsInspectorScrollAnchorView`
    handles a pending restore. Then it updates `viewportBox.offset` and calls
    `revealSelection()`, so the restored selection is always visible whatever order the
    `onChange` handlers fire in.
- **Not stashed** (these are reactions to the world, not view state):
  - `tabsSelectionConsumed`, tab recency, the closed-tab stack, arrivals, the scan
    notice, banners, and cards;
  - real effects made during a session: renames, Today links, NEW toggles, tab closes,
    and tab switches.

> [!decision] ctrl_tab_round_trip = discard `.panelSwitch` restores the stash like any
> other ordinary show. Ctrl+Tab from a Highlights-key session to Capture and back shows
> the view from before the Highlights key.

> [!decision] ctrl_tab_round_trip = keep `.panelSwitch` skips the restore and keeps the
> session and stash, so Ctrl+Tab back shows the session's Tabs view. The next `.library`
> or `.scanResults` show restores or discards the stash.

### 2. The Highlights key selects the focused PDF

- `RefsTabsListing` gets `currentID: String?`: the listed row whose tab `isCurrent`, or
  nil when that row is filtered out. `RefsTabsRanker.listing` computes it in both browse
  and search modes. Add it as an init parameter defaulting to nil, so existing call
  sites compile.

> [!decision] focused_preselect_scope = takeover_only Keep `preselectedID` (the switcher
> rule) for every other Tabs entry.
>
> In the model, add a `preferCurrentTab` flag:
>
> - **Set** by the `.highlightsTabs` show.
> - **Read** by `rebuildTabsListing`: while the flag is set, a non-preserving rebuild
>   selects `currentID ?? listing.orderedIDs.first` instead of `preselectedID`. That
>   covers both the stale-data rebuild inside `resetToDefaults` and the session's first
>   fresh snapshot.
> - **Cleared** by the session's first applied read result (snapshot, failure, not
>   running, or needs permission), or by a user query, scope, Newest, or Tabs-toggle
>   change before that, so user intent wins.
>
> After that the session behaves as today: Command-R preserves the selection, and an
> in-panel Command-T toggle uses the switcher rule.

> [!decision] focused_preselect_scope = everywhere Make `preselectedID` equal
> `currentID ?? ids.first` in browse mode, and drop the `highlightsIsFrontmost` index-1
> rule. Search mode keeps the first match. No model flag is needed. Update the ranker
> test that expects `items[1]` and
> `testTabSwitchUsesTheSwitcherPreselectionOnceOnTheNextShow` to the new rule.

### 3. The preview opens at the current page's notes

**RefsCore** (portable; tests run on Linux):

- `HighlightsWindowTitle.pageNumber(fromTitle:) -> Int?` reuses the existing page-suffix
  regex.
- `HighlightsTabsSnapshot.Window.pageNumber: Int?` is a new init parameter defaulting to
  nil. `AXHighlightsTabsReader.readSnapshot` fills it from the window title next to
  `pageCaption`.
- `RefsOpenTab.pageNumber: Int?` is a new init parameter defaulting to nil.
  `HighlightsTabMatcher.match` sets it only for each window's **selected** tab, because
  the title describes only the visible document.
- Add a page-label reader, for example `RefsPageLabel.pages(in:) -> [ClosedRange<Int>]`,
  for the labels bob writes:
  - `Page 5`, `Page: 5`, `Pages 11–12`, `pp. 3–5, 9`, `p. 7`, `p 7`, and a bare `12`;
  - en dash or hyphen as the range separator, case-insensitive prefixes;
  - anything else (for example roman numerals) returns `[]`.
- `RefsInspectorFeed.focusRange(page:) -> ClosedRange<Int>?` returns entry indices:
  - Each entry's pages are the union of its own label and, for tucked groups, its
    pieces' labels.
  - The target page is the labelled page closest to `page`. On a tie, use the lower
    page: the notes the reader has just passed.
  - The result runs from the first through the last entry index labelled with that
    target page.
  - Return nil when the feed is empty or no entry has a numeric page.
- `RefsNotesPage` gets a method like
  `mutating func reveal(through index: Int, total: Int) -> Bool`. It raises `shown` to
  the smallest whole number of pages that covers `index` (capped at `total`) and bumps
  `generation`. Set `batchStart` so the fade-in staggers at most one page batch; do not
  stagger dozens of rows.

**Model** (`RefsPanelModel`):

- **Arm** a one-shot focus while a Highlights-key session is active and the selection
  lands on the current tab's row: the initial preselection, or a later re-selection in
  the same session.
- **Fire** when all of these hold:
  - that row is still selected;
  - the session's fresh snapshot has been applied (`tabsState == .loaded`);
  - the row's `RefsOpenTab` has `isCurrent` and a `pageNumber`;
  - `inspectorLoader.content(for:)` is hydrated with a feed whose `focusRange(page:)` is
    non-nil.

  Check these on selection changes, in `applyTabsReadResult`, and in the existing
  `inspectorLoader.$content` sink.

- **On fire:**
  1. Reveal through the range's last index and store `revealedPageID` / `revealedPage`.
  2. Publish an `inspectorFocus` value (`id`, first `index`, `generation`) and send it
     on a new non-published `inspectorFocusRequests` subject, the same pattern as
     `inspectorNoteReveals`.
  3. Disarm.
- **Never re-arm** on a library-refresh rehydration or Command-R.
- **Disarm or cancel** on Ctrl-D / Ctrl-U (`.scrollInspector`), a selection change away,
  a reset, or a hide.
- When the page is unknown, the row is unmatched or a collision, or no note has a
  numeric page, the preview stays at its top.

**View and controller:**

- `RefsInspectorView` takes the focus value. It draws a hit-test-transparent
  `RefsInspectorFocusAnchor(generation:)` background on the entry at `focus.index`,
  mirroring `RefsInspectorRevealAnchor`. `RefsPanelView` passes the focus value only for
  the selected id.
- `RefsPanelController` subscribes to the new subject, the same way it does for
  `inspectorNoteReveals`. Its `followFocus` mirrors `followReveal`:
  - Lay out and poll for the anchor until a deadline of about 1 s.
  - Convert the anchor's frame to document coordinates.
  - Scroll instantly to a target from a new pure
    `RefsInspectorScrollPlanner.focusTarget(geometry:entryTop:entryBottom:context:)` in
    `Sources/BobMacCapture/Refs/RefsInspectorScroll.swift`. It returns nil when the
    entry's top is already inside the viewport, so an early-page note leaves the hero in
    view. Otherwise it returns
    `clamped(entryTop − RefsVisualTokens.inspectorRevealContext)`.
  - Stop when the panel hides, the selection changes, the generation is stale, or the
    user scrolls: `scrollInspector`, `cancelPendingViewRestores`, or a trackpad offset
    change during the follow.

## Docs (bob-mac-capture `docs/`)

`docs/refs.md` is close to the 6000-word reference cap. Replace sentences rather than
adding them, and put the preview detail in `docs/refs-inspector.md`.

- `docs/refs.md` § Updates: rewrite the "Highlights Tabs key (Command-T)" sentence. The
  key opens from fresh defaults with Tabs on and the focused PDF selected. Nothing done
  in that session is remembered: the next ordinary show returns to the view from before
  it (and so does Ctrl+Tab back, depending on the decision). It still defers a
  new-reference reset.
- `docs/refs.md` hotkeys bullets ("While Highlights is frontmost…" and "Ordinary shows
  restore…"), and § Tabs: "When Highlights is frontmost, the previous tab is
  preselected." and "Command-T from Highlights always starts fresh." Reword both per
  `focused_preselect_scope`.
- `docs/refs-inspector.md`, in the notes-paging bullet: a Highlights-key open reveals
  and scrolls the focused PDF's preview to the notes on Highlights' current page. If
  that page has no notes, it goes to the nearest labelled page, and ties go to the
  earlier page. It stays at the top when the page or labels are unknown.
- `docs/keyboard.md`, the "Command-T (in Highlights)" row.
- Run `just docs-check` (required by that repo's `AGENTS.md`).

## Tests

**RefsCore** (`Tests/RefsCoreTests`, runs under `just portable-check`):

- `pageNumber(fromTitle:)`: en dash, hyphen, and no suffix.
- The matcher sets `pageNumber` only on each window's selected tab.
- `currentID` in browse and search mode, and nil when the current row is out of scope.
- The page-label reader covers every format above, plus a non-numeric label returning
  `[]`.
- `focusRange`: exact page, several notes on one page, nearest page, a tie going to the
  lower page, group page ranges, no numeric labels, and an empty feed.
- `RefsNotesPage.reveal(through:total:)`.

**BobMacCapture** (`Tests/BobMacCaptureTests`; the Mac type-checks them and CI runs
them):

- **Rewrite** `testHighlightsTabsShowResetsAndLibraryRestoresTheFilter`:
  - after a Highlights-key show, a `.library` show restores the pre-session query,
    scope, Newest, Tabs-off state, and selection;
  - the stash survives `represent()`;
  - a second Highlights-key show before an ordinary show keeps the original stash;
  - `.panelSwitch` follows the decision;
  - `.scanResults` discards the stash;
  - a qualifying arrival still resets on the next `.library` show;
  - revealed pages and the preview offset come back for the restored selection.
- **Selection:** with `frontmost: true` and two or more tabs, the Highlights-key show
  selects the current tab's row. The in-panel toggle follows `focused_preselect_scope`.
  Extend the `Self.tabsSnapshot` helper with an optional page number.
- **Focus:** with a `RefsShowAnnotation` fixture whose `pageLabel` values span pages
  (the existing `(1...13)` helper is a model):
  - the model publishes a focus at the expected index, and the revealed page covers it;
  - no focus outside a session, with an unknown page, or with non-numeric labels;
  - re-selection re-arms;
  - Ctrl-D before hydration cancels.
- **`RefsInspectorScrollTests`:** `focusTarget`.
- **Hosted:** in `RefsListScrollHostedTests` the list returns to its pre-session offset
  after a Highlights-key session, and a hosted inspector test lands the focused entry
  inside the viewport.

## Verification

1. `just portable-check` in the bob-mac-capture checkout on athena (Swift is on PATH).
   It lints every Swift file and builds and tests CaptureCore, RefsCore, and refs-rank.
2. Compile the macOS targets on the Mac as `AGENTS.md` says:
   1. rsync a scratch copy without `.build` / `.git`;
   2. run `Scripts/bundle.sh`, then `just typecheck-tests`;
   3. delete the copy.

   Never use Bryan's checkout or `just install`. The tailnet reference memory covers Mac
   access. If the Mac is offline, say in the final report that the macOS targets are
   unverified.

3. `just docs-check` for the docs edits.
4. Do not edit `Package.swift` versions or release files. Commit the bob-mac-capture
   changes with conventional `fix(refs)` / `feat(refs)` messages.

## Memory

> [!decision] decision_record In this bob-cli project, use your `/sase_memory_write`
> skill to add `sase/memory/decisions/highlights-tabs-key-session-is-ephemeral.md`
> (`metadata.status: accepted`, `decided:` the implementation date), then run
> `sase memory init`. Follow the record shape: Applies to, Claim, Why with rejected
> alternatives, Cost, and Reopens when.
>
> - **Claim:** a Highlights-key Tabs session opens from fresh defaults on the focused
>   PDF's row, scrolls its preview to the current page's notes, and is never remembered.
> - **Rejected alternatives:** the previous-tab switcher preselection for this key,
>   restoring on hide (breaks the open-error re-show), and remembering the session.
> - **Evidence:** this plan and the bob-mac-capture `docs/refs.md` § Tabs.
