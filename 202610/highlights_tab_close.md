---
tier: epic
title: Bob Refs ⌃D — close a Highlights tab from Tabs
goal: 'In Bob Mac Capture''s Bob Refs panel, ⌃D on a Tabs row closes that Highlights
  tab without leaving the panel. The row folds away, the selection moves to the row
  that took its place, and a glass toast confirms the close, so ⌃D ⌃D ⌃D clears several
  tabs in a row. Closing the last open document also quits Highlights. Every close
  is verified against a fresh Accessibility read, and a failed close puts the row
  back with a banner.

  '
decisions:
  library_rows:
    ask: Should ⌃D also close tabs from the library view, on rows marked as open in
      Highlights?
    choices:
      tabs_only: Close only in Tabs; library ⌃D keeps scrolling (⌘T then ⌃D closes
        a library row's tab)
      library_too: Library rows open in Highlights get a glyph and close on ⌃D; costs
        a quiet read per show
    default: tabs_only
    why: The library never shows which rows are open, so ⌃D there would close tabs
      Bryan can't see
    answer: tabs_only
  reopen:
    ask: Should ⇧⌘T reopen the most recently closed tab, like a browser?
    choices:
      reopen: ⇧⌘T reopens the newest of up to 20 closed tabs kept in memory; close
        toasts say so
      no_reopen: No undo; recover an accidental close by searching the library and
        pressing ↵
    default: reopen
    why: ⌃D sits beside ⌃J/⌃K/⌃N/⌃P, so a slip should cost one chord; ⇧⌘T is browser
      habit
    answer: reopen
phases:
- id: close-core
  title: RefsCore close contracts, session close, and copy
  depends_on: []
  size: medium
  description: 'close-core: pure Foundation additions to RefsCore. They add the close
    result and outcome types, two new reader requirements with safe protocol-extension
    defaults (so every existing conformer keeps compiling), and `HighlightsTabsSession.close`,
    which decides the last-tab quit. They also add the closed-tab stack (reopen decision),
    the after-removal selection rule, and all close copy (toasts, announcement, failure
    banner). Golden tests run on Linux with `just portable-check`.'
- id: close-reader
  title: Accessibility close ladder and graceful quit
  depends_on:
  - close-core
  size: medium
  description: 'close-reader: `AXHighlightsTabsReader` implements `close` and `quitIfNoDocuments`.
    A close first tries the tab''s own close button, then the window''s close button
    when the tab is visible, and only then selects the tab, closes it, and restores
    the previously visible tab. Every close is verified by bounded re-reads, and an
    unsaved-changes sheet is detected and handed to Bryan. When no documents remain,
    Highlights quits through `NSRunningApplication.terminate()`. The phase also runs
    a read-only Mac probe and adds a CI test.'
- id: close-model
  title: Panel close commands, queue, outcomes, keys, and toasts
  depends_on:
  - close-core
  size: medium
  description: 'close-model: RefsPanelModel gains ⌃D close commands. A close removes
    the row optimistically and moves the selection to its neighbour; closes run one
    at a time in press order; each outcome maps to a toast, banner, state card, or
    hide. The router sends ⌃D to close only while a tab row is selected (scroll otherwise)
    and maps ⇧⌘T to reopen. The controller shows the glass toasts. ⌘K gains Close
    Tab. Model, router, and actions-menu tests run against a fake reader.'
- id: close-views
  title: Row close button, footer, inspector hint, motion, design tests, and README
  depends_on:
  - close-model
  size: medium
  description: 'close-views: Tabs rows show a quiet close button on hover and selection
    in a reserved trailing column, and the row exposes a VoiceOver "Close Tab" action.
    Removals fold out with a short animation (fade only under Reduce Motion). The
    footer reads "↵ Switch  ⌃D Close Tab", and the inspector card hint and the closed-tab
    toast style are added, with design fixtures for each. The README covers ⌃D, the
    last-tab quit, reopen, Privacy, and Troubleshooting.'
proposed_by: bbugyi200.apollo.85
decided_by: auto
create_time: 2026-10-16 14:11:29
status: done
bead_id: bob-cli-8l
---

- **PROMPT:** [prompts/202610/highlights_tab_close.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/highlights_tab_close.md)
- **BEAD:** [bob-cli-8l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8l/README.md)

# Plan: Bob Refs ⌃D — close a Highlights tab from Tabs

## Context

Epic bob-cli-84 (`plan:202610/highlights_tabs_switcher.md`) gave Bob Refs a **Tabs**
filter. ⇧⌘O while Highlights is frontmost, or ⌘T in the panel, lists every document open
in Highlights, with ↵ switching to it. That epic listed closing as a non-goal (A11) and
deferred "⌘W to close a tab, behind a separate review". This epic is that review. Bryan
asked for **⌃D**, active only while the selected row is a PDF open in Highlights, which
closes that tab and quits Highlights when the last tab closes.

What the code and research establish (phase workers should re-read the cited files
rather than trust this summary):

- Highlights is a non-scriptable `NSDocument` app with native window tabs. Each tab is
  its own `NSWindow`, but `AXWindows` returns only the visible window per tab group.
  That window's `AXTabGroup` → `AXTabs` lists every tab as an `AXRadioButton` with
  subrole `AXTabButton`. Background tabs expose only their title (research E1, E4–E7,
  `research:202610/highlights_tab_switcher/highlights_tab_switcher.md`, explicit ref
  `file:explicit:9ba3692a69678bff4d21e72b`).
- `AXHighlightsTabsReader` already reads under a 1 s budget on one serial queue. It
  locates targets through `HighlightsTabLocator`, never by stale index, and verifies its
  `select` with a re-read. Close reuses all of that.
- ⌃D is already bound in Bob Refs to `.scrollInspector(.down)`, the half-page preview
  scroll paired with ⌃U (`RefsKeyRouter`, README "The panel" key table). This plan
  overloads ⌃D: it closes while a tab row is selected and keeps scrolling otherwise. ⌃U
  is unchanged everywhere.
- The Copy and Link to Today flows already share a glass toast
  (`RefsGlassToastController` in `RefsCopyHUD.swift`). It can rise above the footer
  while the panel stays up, or centre where the panel was after a hide. Close
  confirmations reuse it.

Whether a document-based app quits after its last window closes is the app's choice, and
Highlights' choice is unverified. The design handles both cases: close first, then quit
only if Highlights is still running with no documents.

## The experience (design)

This is the design every phase implements. The words in quotes are the exact copy.

**The gesture.** In Tabs (⇧⌘O or ⌘T), ⌃D closes the selected row's Highlights tab. This
covers matched, collision, and unmatched rows alike, because ⌃D acts on the _tab_, not
the note. That is unlike ⌘↵, ⌘Y, and ⌘K, which still warn on synthetic rows. On every
close:

1. The row folds out of the list at once (≈0.2 s fade-and-collapse, with the rows below
   gliding up; a plain fade under Reduce Motion).
2. The selection lands on the row that took its place: the next row, or the previous one
   when the closed row was last. The inspector follows.
3. The count ticks down ("9 tabs" → "8 tabs"), and VoiceOver hears "Closed Attention Is
   All You Need. 8 tabs."
4. Once Highlights confirms the close, a glass toast rises above the footer:
   `xmark.square.fill` · **"Closed tab"** · _title_ · "⇧⌘T reopens" (the last line only
   under the reopen decision, and only when the tab can be reopened). Each new toast
   replaces the previous one, so they never stack.

The panel stays up and Highlights stays frontmost, so ⌃D ⌃D ⌃D clears several tabs in a
row. Each press targets the newly selected row. Closing a background tab never changes
the tab Highlights is showing. Closing the visible tab lets Highlights pick its
neighbour, and the "Current" pill moves to it.

**The last tab.** When a close leaves Highlights with no documents, Bob Refs quits
Highlights gracefully (`terminate()`, never force). The panel then hides, since nothing
is left to manage and Highlights' exit would take key status anyway. A toast centres
where the panel was: **"Closed the last tab"** · _title_ · "Highlights quit". The
"Highlights quit" line appears only once the exit is verified. Closing the document
before quitting matters: if Highlights quit with the tab still open, macOS window
restoration could bring that tab back on the next launch.

**Mouse and discoverability.**

- Tabs rows show a quiet close button (`xmark.circle.fill`, secondary, primary on hover)
  in a reserved trailing column. It appears only on the hovered and selected rows, so
  nothing shifts. Its tooltip reads "Close Tab (⌃D)".
- ⌘K on a matched Tabs row lists **Close Tab ⌃D** after Open in Highlights.
- The Tabs footer leads with "↵ Switch ⌃D Close Tab". It says "Close Tab", not "Close",
  because "esc Close" already means hide the panel.
- The collision and unmatched inspector card's hint becomes "↵ switches to this tab · ⌃D
  closes it".

**Where ⌃D does what.**

- With a tab row selected (Tabs filter on, row present in `tabsListing.tabs`), ⌃D closes
  the tab.
- Everywhere else ⌃D keeps today's behaviour: it scrolls the preview half a page, and at
  the very bottom it reveals the next notes page.
- While IME-composing, ⌃D passes through, as today.
- Cards (Copy, Rename, Link to Today) capture keys before the router, as today.

In Tabs the preview still scrolls with ⌃U, the trackpad, and its disclosure row.

**Failures stay calm and truthful.**

- **Already gone.** If the tab was closed in Highlights meanwhile, the row stays gone, a
  fresh snapshot runs, and no error appears.
- **Unsaved changes.** If Highlights shows a save sheet instead of closing, the reader
  brings Highlights forward with that window raised and the panel hides, so the sheet is
  in front of Bryan.
- **Not running / no permission.** These show the existing "not running" card and
  permission card.
- **Anything else.** The row returns to its place, selected, with the error banner
  "Couldn't close “title” in Highlights." Pressing ⌃D again retries.

> [!decision] reopen = reopen ⇧⌘T anywhere in Bob Refs (Tabs or library) reopens the
> most recently closed tab. The stack holds up to 20 entries, in memory only, for the
> app's lifetime. Reopening goes through the existing Highlights open path, which hides
> the panel and shows the PDF in Highlights, and it never records an open (A10). A
> matched row's entry reopens its library PDF. An unmatched or collision row gets an
> entry only when its full path was known, which means it was a window's visible tab. An
> empty stack makes ⇧⌘T a consumed no-op. ⇧⌘T while IME-composing passes through.

> [!decision] reopen = no_reopen Do not add ⇧⌘T, the closed-tab stack, or the toast's
> "⇧⌘T reopens" line. Toasts carry no detail line except the last tab's "Highlights
> quit".

> [!decision] library_rows = tabs_only Library mode is untouched: no extra Accessibility
> read on show, no glyph, and ⌃D always scrolls the preview there.

> [!decision] library_rows = library_too Each show in either source also starts a
> background tab read through the session, but only when `reader.isTrusted()`. It never
> prompts and never shows the permission card in library mode. Re-read on ⌘R as well.
> Match the snapshot against `library.items` and keep an `openTabsByItemID` map. Library
> rows with an open tab show a small open-tab glyph (`square.on.square`, caption size,
> secondary) before the state glyph, and VoiceOver appends ", open in Highlights". ⌃D on
> such a row closes its tab with the same queue, toasts, and last-tab quit. The row
> itself stays (it is still a reference) and loses its glyph. If the read is untrusted,
> not running, or failed, there are no glyphs and ⌃D scrolls everywhere in the library.

## Where the work lives

All code lives in the **`bob-mac-capture` linked repo**. Phase workers open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`), read its `AGENTS.md`, and
commit there. bob-cli does not change. The thin-client rule
(`decisions:mac-capture-is-a-thin-client`) is untouched: closing a Highlights tab is
macOS UI state and never touches the vault, `bob`, or the open log.

## Rules for every phase

- **Compile everything you change** (bob-mac-capture `AGENTS.md`). Run
  `just portable-check` on a host with Swift; this host has none, so use `athena`. Any
  phase that touches `Sources/BobMacCapture*` or `Tests/BobMacCapture*` must also
  compile a scratch copy on the Mac (`ssh mac`, best effort):
  1. rsync the repo without `.build` or `.git` into a temp directory;
  2. run `Scripts/bundle.sh` (ad-hoc default) and `just typecheck-tests`;
  3. delete the copy.

  Never use Bryan's checkout on the Mac, and never run `just install` there. If the Mac
  is offline, say in the final report that the macOS targets are unverified. CI runs the
  XCTest suites.

- **Follow the lint order.** Run `swift-format lint` first. A parse error in it skips
  the later CI steps, so check them after a red lint step.
- **Never change Bryan's live Highlights session.**
  - No agent presses, closes, opens, activates, or quits anything in Highlights. The
    only exception is the read-only probe in `close-reader`.
  - Close behaviour is verified by verification logic in code and by Bryan's smoke test
    below.
- **Keep AX use contained.**
  - Every `AXUIElement*` call stays in `AXHighlightsTabsReader.swift`.
  - The allowed actions grow to exactly these, all against Highlights' process only:
    - read window, tab, and close-button attributes, plus sheet presence;
    - `AXPress` on a tab button, a tab's own close button, or a window's close button;
    - `AXRaise` on a window;
    - `NSRunningApplication.activate` (select and the unsaved-changes hand-off only);
    - `NSRunningApplication.terminate()`, only when no documents remain.
  - Never `forceTerminate`, synthesize keystrokes, open context menus, or use Apple
    Events, Screen Recording, or private API.
- **Keep tab names private.**
  - Titles, file names, and paths stay in memory, including the closed-tab stack. They
    never go into logs, signposts (event names only), diagnostics, `UserDefaults`,
    files, commits, bead notes, or test fixtures copied from Bryan's live session.
  - Toasts and VoiceOver may show a title, as Copy and Link to Today already do.
- **Match the house style.** Every type and public member gets a doc comment. Inject
  protocols and use fakes in tests. Pure logic lives in `RefsCore`. Keep existing
  library-mode and switching behaviour unchanged.

## Shared contracts (owned by `close-core`)

`close-reader` and `close-model` run in parallel against these RefsCore names. The names
are binding. `close-core` may add members or helpers, but must not rename these or
change their meaning.

- **`HighlightsTabsSnapshot.documentCount: Int`** is the total number of tabs across all
  windows.
- **`HighlightsTabCloseResult`** (`Equatable`, `Sendable`) is the reader's answer:
  - `.closed(HighlightsTabsSnapshot)`: the close was verified. The payload is the
    post-close snapshot, which is empty when Highlights exited.
  - `.notFound`: no tab with that title remains.
  - `.notRunning`.
  - `.notTrusted`.
  - `.needsAttention`: Highlights showed a sheet on that window instead of closing it.
    The reader has already brought Highlights forward.
  - `.failed(String)`: a bounded, name-free reason.
- **`HighlightsTabsReading`** gains two requirements, both declared in the protocol and
  both given defaults in a protocol extension, so `AXHighlightsTabsReader`,
  `UnavailableHighlightsTabsReader`, and every test fake keep compiling unchanged:
  - `close(_ target: HighlightsTabTarget) async -> HighlightsTabCloseResult` defaults to
    `.failed("Closing Highlights tabs isn't available in this build")`.
  - `quitIfNoDocuments() async -> Bool` defaults to `false`. True means Highlights is
    not running afterwards. Implementations must re-check that no documents are open
    before quitting.
- **`HighlightsTabCloseOutcome`** (`Equatable`, `Sendable`) is the session's answer:
  - `.closed(remaining: HighlightsTabsSnapshot)`;
  - `.closedLast(highlightsQuit: Bool)`;
  - `.alreadyClosed`;
  - `.notRunning`, `.notTrusted`, `.needsAttention`, `.failed(String)`.
- **`HighlightsTabsSession.close(_ tab: RefsOpenTab, reopen: RefsClosedTab?) async -> HighlightsTabCloseOutcome`**
  - Bump the refresh generation first, so a read that started earlier can never
    overwrite the close's result.
  - Forward `tab.target` to `reader.close`, then map the result:
    - On `.closed(snapshot)`: set `lastSnapshot`, touch the current tab's recency, and
      push `reopen` when it is non-nil. Then, if `snapshot.documentCount == 0`, await
      `reader.quitIfNoDocuments()` and return `.closedLast(highlightsQuit:)`. Otherwise
      return `.closed(remaining: snapshot)`.
    - `.notFound` becomes `.alreadyClosed`, and nothing is pushed.
    - The other results map one-to-one.
- **`RefsClosedTab`** (`Equatable`, `Sendable`): `title`, `fileName`, `url: URL`,
  `itemID: String?`.
- **`RefsClosedTabs`** is a value type: `push(_:)` (newest last, deduped by standardized
  URL so a re-close moves to the top, capped at 20 by dropping the oldest),
  `popLast() -> RefsClosedTab?`, `isEmpty`, `count`. The session exposes
  `closedTabs: RefsClosedTabs` (read-only) and `popClosedTab() -> RefsClosedTab?`.
- **`RefsSelectionPolicy.afterRemoving(_ id: String, in listing: RefsListing) -> String?`**
  returns the next id in `orderedIDs`, else the previous one, else nil. It never wraps.
  An id not in the listing returns the listing's first id.
- **`RefsTabClosePresentation`** is the single home for close copy (pure, tested):
  - `toast(title:canReopen:) -> RefsTabCloseToast`: "Closed tab" / title / "⇧⌘T reopens"
    or nil.
  - `lastToast(title:highlightsQuit:) -> RefsTabCloseToast`: "Closed the last tab" /
    title / "Highlights quit" or nil.
  - `announcement(title:countText:) -> String`: "Closed T. 8 tabs." When `countText` is
    nil it reads "Closed T."
  - `lastAnnouncement(title:highlightsQuit:) -> String`: "Closed T. Highlights quit." or
    "Closed T."
  - `failureMessage(title:) -> String`: "Couldn't close “T” in Highlights."
  - `toastHoldDuration`: 2.2 s, matching Link to Today.
- **`RefsTabCloseToast`** (`Equatable`, `Sendable`): `title`, `value`,
  `detail: String?`, `isLast: Bool`.

Under `reopen = no_reopen`, omit `RefsClosedTab`, `RefsClosedTabs`, `closedTabs`,
`popClosedTab()`, the `reopen:` parameter, and the `canReopen` toast argument. The
signature becomes `close(_ tab: RefsOpenTab)`.

## Phase `close-core`: RefsCore close contracts, session close, and copy

Implement every name in [Shared contracts](#shared-contracts-owned-by-close-core) in
`Sources/RefsCore`, using only Foundation and CaptureCore:

- extend `RefsHighlightsTabs.swift` for the result, outcome, protocol, and session
  additions;
- extend `RefsSelection.swift` for `afterRemoving`;
- add a new `RefsTabClose.swift` for the stack and presentation, or another name in the
  house style.

Keep `UnavailableHighlightsTabsReader` on the protocol defaults. Do not touch
`Sources/BobMacCapture*`. The protocol defaults are what keep the Mac targets compiling
until `close-reader` and `close-model` land.

Golden tests in `Tests/RefsCoreTests/` (for example `RefsTabCloseTests.swift`), with the
existing `FakeTabsReader` extended to script `close` and `quitIfNoDocuments` results and
to count calls. They run with `just portable-check` on athena.

- **Session results.**
  - A verified close with tabs remaining returns `.closed(remaining:)`, updates
    `lastSnapshot`, and never calls quit.
  - Closing the last document calls `quitIfNoDocuments` once and returns
    `.closedLast(highlightsQuit: true)`; a false quit gives `false`.
  - `.notFound` returns `.alreadyClosed`, pushes nothing, and never quits.
  - `needsAttention`, `notRunning`, `notTrusted`, and `failed` pass through and never
    quit.
- **Generations.** A `refresh()` started before a close returns nil (superseded).
- **Stack** (reopen decision). It covers push and pop order, dedupe by standardized URL,
  the cap of 20, a nil `reopen` pushing nothing, and `popClosedTab` draining
  newest-first.
- **Selection.** `afterRemoving` covers middle, last, only, and unknown ids.
- **Copy.** Pin every string, including the nil-detail variants.
- **Defaults.** A conformer that does not override `close` or `quitIfNoDocuments` gets
  the documented defaults.
- **Snapshot.** Test `documentCount` for zero windows, a window without tabs, and two
  windows.

## Phase `close-reader`: Accessibility close ladder and graceful quit

Implement `close(_:)` and `quitIfNoDocuments()` in
`Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`. Run both on the reader's
existing serial queue with its messaging timeout. Give `close` an overall budget of
about 2 s, which covers the verification polls.

**`close(_ target:)`**, in this order:

1. **Guards.** Use the same trust/running guard order as `read()` and `select`.
2. **Locate.** Take a fresh read and run `HighlightsTabLocator.locate`.
   - When it returns nil and no tab titled `target.fileName` exists, return `.notFound`.
   - When it returns nil because of ambiguity, return
     `.failed("That tab's name is ambiguous")`.
   - Re-check the button's title, as `select` does.
3. **Close, using the first rung that applies.**
   1. **The tab's own close button.** Search the tab button's `AXChildren` (depth ≤ 2)
      for an `AXButton` whose subrole is `AXCloseButton`, or that is advertised through
      `kAXCloseButtonAttribute`. `AXPress` it. This closes a background tab without
      switching to it.
   2. **The window's close button.** Use this when the target is its window's selected
      tab, or its window has no tab bar. `AXPress` that window's
      `kAXCloseButtonAttribute`. In a tabbed window this closes only the visible tab.
   3. **Select, close, restore.** Use this only for a background tab with no close
      button of its own.
      1. Remember the window's selected tab title.
      2. `AXPress` the target tab and re-read, bounded, until it is selected. Because
         each tab is its own `NSWindow`, take the newly visible window element from a
         fresh `AXWindows` read.
      3. Press that window's close button.
      4. `AXPress` the remembered tab again if it still exists, so Highlights shows what
         it showed before.

   No rung activates Highlights or raises a window. This is the one sanctioned tab
   switch, and it never cycles tabs to learn paths.

4. **Verify.** Re-read every ~50 ms within the budget until the target is gone, meaning
   the count of tabs titled `target.fileName` dropped below its pre-close count. Then
   return `.closed(snapshot)` with that fresh snapshot. If Highlights exited meanwhile,
   return `.closed` with an empty snapshot.
5. **Not closed.**
   - If the target's window now has an `AXSheet` child, activate Highlights, `AXRaise`
     that window, and return `.needsAttention`.
   - Otherwise return `.failed("Highlights didn't close that tab")`.
   - Never retry in a loop.
6. **Signposts.** Emit event names only: `refs-tabs-close-tab-button`,
   `refs-tabs-close-window-button`, `refs-tabs-close-select-fallback`,
   `refs-tabs-close-unverified`, and `refs-tabs-close-needs-attention`. They let Bryan's
   smoke test show which rung Highlights needs.

**`quitIfNoDocuments()`:**

1. If Highlights is not running, return true.
2. If it is untrusted, return false.
3. Take a fresh read. If any tab remains, return false. This is the reader's own guard:
   it never quits with a document open.
4. Call `terminate()` (never `forceTerminate`), then poll `isTerminated` for up to about
   2 s, and return whether it exited.

**Read-only probe (best effort, Mac scratch copy).** Before choosing rung 1's exact
lookup, run a throwaway script against Bryan's running Highlights that only reads.
Record, as counts and shapes only:

- whether tab buttons have children;
- their roles, subroles, and advertised actions;
- whether a close button is exposed when the tab is not hovered;
- whether windows expose `kAXCloseButtonAttribute`.

Never press, close, open, or activate anything. Delete the script with the copy, and
never record names. If the probe cannot run, keep all three rungs and say so in the
final report.

**Tests.** Extend `AXHighlightsTabsReaderTests`. A reader built with an unknown bundle
id returns the same not-running/not-trusted result from `close` as from `read()`. That
proves the protocol default was overridden: it is never the default `.failed`. That
reader's `quitIfNoDocuments()` returns true and launches nothing.

## Phase `close-model`: Panel close commands, queue, outcomes, keys, and toasts

Change `RefsPanelModel`, `RefsPanelController` (`RefsPanel.swift`), `RefsKeyRouter`,
`RefsActionsMenu`, and the toast style in `RefsCopyHUD.swift`. Library mode, ↵
switching, and every other key keep their behaviour.

- **Commands.** Add these to `RefsCommand`:
  - `.closeSelectedTab` (⌃D);
  - `.closeTab(id: String)` (the row button and ⌘K);
  - `.reopenClosedTab` (⇧⌘T, reopen decision).
- **Router.**
  - Add `selectionIsOpenTab: Bool` (default false) to `RefsKeyContext`.
  - ⌃D (key code 2): pass through while IME-composing, as today. Otherwise return
    `.closeSelectedTab` when `selectionIsOpenTab`, else `.scrollInspector(.down)`.
  - ⇧⌘T (key code 17, flags minus Caps Lock == ⌘⇧) returns `.reopenClosedTab` unless
    IME-composing. Plain ⌘T stays `.toggleTabs`.
  - The controller fills the new context field from `model.canCloseSelectedTab`.
  - Update the router's README-keyboard doc comment.
- **Model API.**
  - `canCloseSelectedTab` / `canCloseTab(id:)` are true when the Tabs filter is on and
    the id is in `tabsListing.tabs`.
  - `perform(.closeSelectedTab)` re-checks. When nothing is closable it falls back to
    the inspector scroll, so the router race can never swallow a scroll.
  - Rename, Link to Today, and Copy guards match `.showActions`.
  - Synthetic rows close. They never get the note-action warning.
- **Optimistic removal and the queue.**
  - On a close, compute the neighbour with `RefsSelectionPolicy.afterRemoving` before
    mutating anything.
  - Add the row id to a private `closingRowIDs` set. `rebuildTabsListing` filters
    `matchedTabs` by that set before ranking.
  - Rebuild, then select the neighbour. Wrap the listing change in `withAnimation` (≈0.2
    s snappy). Skip the animation when
    `NSWorkspace.shared.accessibilityDisplayShouldReduceMotion` is set.
  - Post the VoiceOver announcement from `RefsTabClosePresentation.announcement`, with
    the new `RefsTabsCount` text.
  - Do not bump `tabsRefreshGeneration` here. The `closingRowIDs` filter already stops a
    read in flight from resurrecting the row. Because the session's close supersedes any
    earlier refresh, every close outcome must leave `tabsState` settled, never stuck at
    `.loading`.
  - Closes run **one at a time in press order** through a single worker task draining a
    FIFO of `(tab, title, reopen entry)`. Rapid ⌃D presses therefore reach the reader in
    order.
  - Presentation and hiding never cancel queued closes. A close stays hidden until its
    own outcome lands.
- **Reopen entry** (reopen decision).
  - For a matched row:
    `RefsClosedTab(title: item title, fileName:, url: library.pdfURL(for: item), itemID:)`.
  - For a synthetic row: an entry only when `tab.documentPath` is known.
- **Outcomes.** Each outcome first removes its id from `closingRowIDs`.
  - **`.closed(remaining:)`**
    - Apply it through the existing snapshot path: re-match, and preserve the selection.
    - Call `tabCloseCompletion` with `RefsTabClosePresentation.toast`. `canReopen` is
      true when an entry was pushed.
  - **`.closedLast(highlightsQuit:)`**
    - Clear `matchedTabs` and the closing set.
    - Hide through the coordinator path that Copy uses, so a retained Capture draft
      comes back as it does after a copy.
    - Call `tabCloseCompletion` with `lastToast`, and announce `lastAnnouncement`.
  - **`.alreadyClosed`**: start a selection-preserving refresh. No banner, no toast.
  - **`.needsAttention`**: hide the panel. Highlights is already frontmost with its
    sheet.
  - **`.notRunning` / `.notTrusted`**: use the existing `tabsState` transitions.
  - **`.failed`**
    - Rebuild so the row returns, and re-select it.
    - Show the error banner `RefsTabClosePresentation.failureMessage`.
    - Start a selection-preserving refresh, so the list stays truthful and a superseded
      ⌘R cannot leave the state loading.
  - If the panel is hidden when an outcome lands, apply state silently. Show no banner
    and no above-footer toast, except the last-tab toast, which always shows.
- **Reopen** (reopen decision). `perform(.reopenClosedTab)` pops the newest entry.
  - An entry whose `itemID` still resolves goes through
    `openInHighlights(item, recordsOpen: false, switchedFileName: fileName)`.
  - Any other entry calls `opener.openInHighlights(url, app:)`, with the same hide,
    error re-present, and banner handling as a failed Highlights open, and no
    `recordOpen`.
  - A missing Highlights app shows the existing "isn't installed" banner.
  - An empty stack consumes the key and does nothing. Reopen works in Tabs and library
    modes alike.
- **Controller toasts.**
  - Add `RefsGlassToastStyle.closedTab` (`xmark.square.fill`, secondary tint). Add a
    `show` overload taking that style plus an optional detail.
  - Wire `model.tabCloseCompletion`:
    - with the panel visible and the toast not last: show it above the footer, rising,
      like Link to Today;
    - for the last-tab toast: hide first, then centre the toast on the former frame,
      like Copy.
- **⌘K.** Add `RefsAction.closeTab`, titled "Close Tab" with key equivalent `d` +
  `.control`. `actionsSections()` appends it to the opens section, after Open in
  Highlights, for matched rows while the Tabs filter is on. `performAction(.closeTab)`
  runs `.closeTab(id:)` for the selected row.
- **Preview hook.** Extend `installTabsForPreviews` (or add a sibling) so `close-views`
  can render a row mid-close and a selected closable row without a reader.

> [!decision] library_rows = library_too This phase also owns the background read and
> the `openTabsByItemID` map from the decision callout above. `canCloseSelectedTab`
> covers those library rows, and a library close keeps the row (it only drops it from
> the map). Add tests for these cases: the read runs only when trusted, a library ⌃D on
> an open row closes it, a library ⌃D on any other row scrolls, and the row survives the
> close.

**Tests** go in `RefsPanelModelTests.swift`, or a new `RefsPanelTabCloseTests.swift`.
The fake reader overrides `close` and `quitIfNoDocuments`, scripts their results, and
records targets. Cover:

- the optimistic removal, with the neighbour selected (middle row, last row, and only
  row);
- two rapid closes reaching the reader in press order, with the second targeting the
  neighbour;
- a superseded refresh never resurrecting a closing row;
- `.closed` applying the snapshot and keeping the selection;
- `.closedLast`: panel hidden, completion called with the last toast, quit requested
  once;
- `.failed`: row restored and selected, banner text pinned;
- `.alreadyClosed` refreshing without a banner;
- `.needsAttention` hiding the panel;
- `.notTrusted` showing the permission state;
- a close never touching `refs-open-log.json` or `signals.opens`;
- ⌃D with the Tabs filter off scrolling the inspector;
- synthetic rows closing instead of warning;
- reopen: popping the newest entry, opening with `recordsOpen: false`, and a no-op on an
  empty stack.

Router tests cover: ⌃D with and without `selectionIsOpenTab`, ⌃D while IME-composing, ⌃U
unchanged, ⇧⌘T versus ⌘T, and ⇧⌘T while IME-composing. Actions-menu tests check that
Close Tab appears only for matched Tabs rows.

## Phase `close-views`: Row close button, footer, inspector hint, motion, design tests, and README

SwiftUI changes under `Sources/BobMacCapture/Refs/`. They render only what `close-model`
publishes.

- **Row.**
  - `RefsRowView` gains `onClose: (() -> Void)?`, which is non-nil when
    `model.canCloseTab(id:)`.
  - The row reserves an 18 pt trailing column after the state glyph whenever `onClose`
    is set, so nothing shifts.
  - That column shows `xmark.circle.fill` (13–15 pt, `.secondary`, `.primary` while the
    pointer is over it) on the hovered or selected row. It fades in and out over 0.12 s;
    there is no fade under Reduce Motion.
  - It is a borderless `Button`. Clicking it runs `.closeTab(id:)` and never selects or
    activates the row.
  - Its tooltip is `.help("Close Tab (⌃D)")`.
  - The row adds `.accessibilityAction(named: "Close Tab")`.
- **Motion.** In `RefsListView`, give rows a removal transition: opacity plus a slight
  collapse, so the rows below glide up. Under Reduce Motion use opacity only. Query
  re-ranks stay unanimated: only the model's `withAnimation` close path animates.
- **Footer.**
  - Tabs variants become "↵ Switch ⌃D Close Tab ⌘↵ Note …".
  - Keep "⌃D Close Tab" in every Tabs variant. If the narrowest variant no longer fits
    at 760 pt, drop "⌥↵ Reveal" from the Tabs core first.
  - Library variants are unchanged.
  - Update `RefsFooterTests`.
- **Inspector.** The collision and unmatched card hint reads "↵ switches to this tab ·
  ⌃D closes it".
- **Toast view.** Make sure `RefsGlassToastView` renders the `.closedTab` style and a
  nil detail cleanly.
- **Design tests** (`RefsPanelDesignTests` / `RefsTabsViewsTests` style, via the preview
  hooks and `RenderFixtureWriter`):
  - a 9-row Tabs list with the selected row's close button;
  - a hovered, unselected row, if the preview can force the hover state;
  - the Tabs footer;
  - the "Closed tab" toast with and without "⇧⌘T reopens";
  - the "Closed the last tab" toast, with and without "Highlights quit".
- **README** (`bob-mac-capture/README.md`).
  - **Tabs subsection**: ⌃D closes the selected tab, synthetic rows included; the panel
    stays up; the last close quits Highlights; the failure behaviours, including the
    unsaved-changes hand-off; and ⇧⌘T reopen under the reopen decision.
  - **"The panel" key table**: Ctrl-D reads "In Tabs, close the selected tab in
    Highlights; elsewhere scroll the preview…". Add Shift-Command-T (reopen decision).
  - **Bob Refs shortcuts table**: add Ctrl-D and Shift-Command-T.
  - **Privacy**: Accessibility also presses a tab's or window's close button. Bob Refs
    quits Highlights only after closing its last document. Closed-tab entries stay in
    memory.
  - **Troubleshooting**: add "Couldn't close …" and the save-sheet hand-off.
  - **⌘K list**: add Close Tab.

> [!decision] library_rows = library_too This phase also renders the library open-tab
> glyph and its VoiceOver suffix, adds a design fixture with two open library rows, and
> documents the quiet per-show read in the README Tabs and Privacy sections.

## Manual verification (Bryan)

These steps need the GUI Mac and cannot be done by agents. After all phases land and
`just install`, open a few throwaway PDFs in Highlights:

1. ⇧⌘O, then ⌃D on a background row. The row folds away, the next row is selected, the
   toast shows, and Highlights' visible tab does not change.
2. ⌃D on the current row. Highlights shows a neighbour, and the Current pill moves.
3. ⌃D ⌃D ⌃D quickly. Each press closes the next row, in order.
4. Click the row close button, then ⌘K › Close Tab.
5. Close an unmatched (`~/Downloads`) tab.
6. With two tabs left, close both. Highlights quits, and the "Closed the last tab ·
   Highlights quit" toast appears. Relaunch Highlights and check it does not restore the
   closed tabs.
7. ⇧⌘T reopens the last closed tab (reopen decision).
8. In the library, ⌃D still scrolls the preview (tabs_only).
9. Run Console with the `refs-tabs-close-*` signposts to see which rung Highlights uses.
   Report it, so a follow-up can drop dead rungs.
10. Edit an annotation, then close that tab quickly. If a save sheet appears, Highlights
    comes forward with it.

## Follow-ups and non-goals

- **Non-goals:**
  - Close Other Tabs and Close All;
  - closing every window's copy of the same document (Highlights reuses an open
    `NSDocument`, so a row is one tab in practice);
  - a confirmation dialog (closing is cheap, and reopen is the safety net);
  - persisting closed tabs;
  - any bob-cli change.
- **Pending memory task bob-cli-8c** proposes a decisions strand claiming that Bob Mac
  Capture's Accessibility use is read and select only. That claim becomes stale with
  this epic. The `close-reader` phase records a `PROPOSED FOLLOW-UP:` note on its own
  bead saying that bob-cli-8c's claim must add the close-button presses and the
  no-documents `terminate()`. The land agent carries that note to bob-cli-8c. No memory
  file changes in this epic.
