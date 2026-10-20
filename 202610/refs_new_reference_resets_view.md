---
tier: tale
title: Bob Refs resets to and selects newly added references
goal: The next Bob Refs show after a reference is added resets the remembered view
  and selects it.
size: medium
decisions:
  visible_arrivals:
    ask: Should a reference landing in the first 10 s of an untouched open panel also
      reset the view and select it?
    choices:
      untouched: References the show's own vault pull brings in are selected; never
        moves after your first input
      next_show: Arrivals while open only join the list as today; the next show resets
        and selects them
    default: untouched
    why: The show-time vault pull exists so fresh athena/apollo references appear
      at once
    answer: untouched
proposed_by: bbugyi200.apollo.9a
proposed_with: claude/opus
decided_by: auto
status: done
---

# Bob Refs: a newly added reference resets the remembered view

## Context

All work is in the **`bob-mac-capture`** linked repository. Open it with the
`/sase_repo` skill (`sase repo open bob-mac-capture -r "..."`), use only the path it
prints, and read that repo's `AGENTS.md` before editing.

Commit `5935822` ("feat(refs): remember panel view across shows") made every ordinary
Bob Refs show restore the remembered query, kind scope, Newest/Tabs filters, selected
reference, revealed highlight pages, and list/preview scroll positions
(`RefsPanelModel.prepareForPresentation(source:)` → `restoreLastViewForPresentation()`).
Ctrl-Z (`RefsCommand.resetView`) and the scan-notification show
(`RefsPresentationSource.scanResults`) call `resetToDefaults(tabs: false)` instead.

Bryan wants one exception: when a new reference has been added to the library, the next
show must reset to the fresh-launch defaults **and select that new reference**, so
Return immediately opens its PDF in Highlights.

Facts the design relies on (verified while planning):

- `RefsLibrary` keeps refreshing while the panel is hidden (vault watcher, 0.5 s
  debounce; the Mac's LaunchAgent runs `bob vault-sync -q` every 15 s), and every
  publication reaches `RefsPanelModel.libraryDidPublish()` whether or not the panel is
  visible. Most arrivals therefore land while the panel is hidden.
- `RefsPanelController.show` calls `prepareForPresentation(source:)` and then
  `refreshForOpen()`, which starts the throttled `refs-pull` lane. A reference pushed
  from another host in the last ~15 s lands one or two seconds **after** the show.
- A reset that only selects the first row is not enough. A fresh arrival often has no
  `added_at` until the `-g` lane backfills it, so the NEW section (newest-first by
  `addedAt`, else the start of the `added` day) can rank it below today's timed rows.
  The new reference must be selected explicitly.
- A reference's id is its note path, so a rename or move looks like "old id vanished,
  new id appeared". bob's `added` date comes from frontmatter `captured`/`created`, and
  `-g` git birth follows renames (bob-cli `docs/ref.md`), so a renamed old reference
  keeps its old `added` day.
- The default listing (empty query, All scope, ranked) lists every library item, so a
  qualifying arrival is always present in the reset listing.

## Behavior

1. **Arrival tracking.** The model keeps a baseline of every reference id it has seen,
   plus the ids that are _pending_ (seen for the first time after the baseline was
   established), each tagged with the batch (publication) that first contained it.
   - The baseline is seeded from `library.items` at model init (the cache paint). If
     that is empty (no cache), the first non-empty publication establishes it and marks
     nothing pending, so a first-run snapshot never counts the whole library as new.
   - The baseline only grows: an id that vanishes and later returns is never pending
     again.
   - Observation happens in `libraryDidPublish()` after its existing `isRenaming` early
     return, so a rename in progress is never observed mid-move.
2. **Qualification.** A pending id qualifies only while its item is still in the library
   and its `added` day is within `RefsRankingConstants.newWindowDays` (3) local days of
   `signals.now` in `signals.calendar` — the same day test NEW's arrival rule uses. This
   excludes renames and moves of older references. Rows with no `added` date do not
   qualify. Reading state, Today links, and opens do not matter.
3. **Show-time reset.** On a `.library` show that gets past the existing
   `isRenaming || isLinkingToday` early return:
   - If any pending id qualifies and no failed rename card is being re-presented, run
     the arrival reset (below) instead of restoring the remembered view.
   - Otherwise restore exactly as today.
   - Either way, clear all pending ids, except when a failed rename card was
     re-presented. In that case keep them for the next show.
   - `.scanResults` shows keep today's reset and first-row selection (the Just scanned
     section), and also clear pending ids.
   - `.highlightsTabs` shows (Shift-Command-O) never reset for arrivals and keep pending
     ids. The new reference is not an open tab, and resetting would defeat the Tabs
     request. The next ordinary show resets.
   - Shows that return early, and `represent()` (open-error re-show), never reset and
     keep pending ids.
4. **Arrival reset.** Reuse `resetToDefaults(tabs: false)` (query "", All scope, ranked
   order, Tabs off, revealed pages and preview place cleared, `inspectorGeneration` and
   `listResetGeneration` bumped). Then select the target:
   - Among qualifying pending ids present in the fresh listing, choose the most recent
     batch.
   - Within that batch, choose the id that appears first in `listing.orderedIDs`.
   - If none is listed, keep the first-row selection.

   Post the VoiceOver announcement `New reference: <display title>` instead of "Bob Refs
   reset". Keep any banner that is already up. At show time the banner was already
   cleared and the pending scan banner installs afterward, as today. Then clear all
   pending ids: every one of them is now visible in the default view.

5. **Consumption without a show.** While the panel is visible:
   - Selecting a pending id by any path (arrow, click, open, re-rank) marks it seen, so
     the next show does not reset for a reference the user already landed on. Do this in
     `selectedID`'s `didSet`, guarded by `panelIsVisible()`. Hidden re-ranks must not
     consume.
   - A visible ⌘S scan that presents its results (`buildFreshListingForScan`) marks
     `outcome.created` paths seen.
   - A successful rename (`runRename`, visible or hidden) marks the destination id seen
     before it clears `renameCard`.
   - Ctrl-Z does not consume anything by itself. It consumes only the row it ends up
     selecting.
6. **Visible arrivals.** Arrivals that land while the panel is open stay pending (they
   still join the open listing exactly as `arrivalsCanJoin` allows today). The next
   `.library` show resets for them unless the user selected them meanwhile.

> [!decision] visible_arrivals = untouched Additionally, a `.library` show that gets
> past the early return arms a grace deadline of `now() + 10 s` (constant
> `RefsArrivalTracker.visibleGraceSeconds`, or similar). These clear the deadline: any
> `perform(_:)` call, `setQuery`/`queryDidChange`, and `panelDidHide()`. Other show
> sources leave it unset. In `libraryDidPublish()`, run the arrival reset instead of the
> join/re-rank branch when all of the following hold:
>
> - `panelIsVisible()` is true and the deadline is set and not passed.
> - Tabs is off.
> - No Copy, Rename, or Link to Today card is open.
> - No rename or Today link is running, and `pendingOpen` is nil.
> - A qualifying target exists.
>
> The rest of `libraryDidPublish()` (Today-link overlay pruning, content refresh, and
> unavailable flags) still runs first. To make the clock testable, add an injected
> `now: @escaping () -> Date = { Date() }` as the **last** `RefsPanelModel.init`
> parameter, mirroring `RefsLibrary`'s `now`, so existing call sites compile unchanged.

> [!decision] visible_arrivals = next_show Skip the grace deadline, the `now` injection,
> and the visible reset branch. Visible arrivals only join (unchanged) and stay pending
> for the next `.library` show.

## Implementation

### RefsCore (portable, testable on Linux)

- `Sources/RefsCore/RefsRanking.swift`: extract the `added`-day test from
  `RefsRanker.arrivedNew` into
  `public static func addedWithinNewWindow(_ item: RefItem, signals: RefsSignals) -> Bool`.
  Have `arrivedNew` call it, with no behavior change.
- New `Sources/RefsCore/RefsArrivals.swift`:
  `public struct RefsArrivalTracker: Equatable, Sendable` holding an optional baseline
  `Set<String>`, pending ids → batch number, and a batch counter. Suggested API:
  - `init(knownIDs: some Sequence<String>)`: seeds the baseline when it is non-empty.
  - `mutating func observe(_ ids: some Sequence<String>)`: implements Behavior 1. One
    batch number per call that records anything.
  - `mutating func markSeen(_ ids: some Sequence<String>)`: removes ids from pending and
    adds them to an established baseline. It never establishes the baseline itself;
    otherwise the next observe would flag the whole library.
  - `mutating func clearPending()` and `var hasPending: Bool`.
  - `func hasQualifying(item: (String) -> RefItem?, signals: RefsSignals) -> Bool`.
  - `func target(orderedIDs: [String], item: (String) -> RefItem?, signals: RefsSignals) -> String?`:
    implements Behavior 2 and the target rule from Behavior 4.
  - For the `untouched` branch, the grace constant
    (`public static let visibleGraceSeconds: TimeInterval = 10`).

  Doc-comment the type in the house style. Explain the baseline-only-grows rule and why
  `markSeen` never seeds the baseline.

### RefsPanelModel (`Sources/BobMacCapture/Refs/RefsPanelModel.swift`)

- Add `private var arrivals: RefsArrivalTracker`, seeded in `init` from
  `library.items.map(\.id)`.
- `libraryDidPublish()`: call `arrivals.observe(library.items.map(\.id))` right after
  the `isRenaming` guard. Under `untouched`, add the visible-reset branch ahead of the
  existing `pendingRefreshRerank` / `arrivalsCanJoin` branches. The reset re-ranks, so
  it also satisfies a pending ⌘R re-rank.
- `prepareForPresentation(source:)`: implement Behavior 3. Add a private
  `resetForArrival()` that implements Behavior 4. Give `resetToDefaults` an announcement
  parameter (default "Bob Refs reset") so the arrival path posts only its own
  announcement.
- `selectedID` `didSet`, `buildFreshListingForScan`, and `runRename`: consumption per
  Behavior 5.
- `installForPreviews(...)`: replace the tracker with one seeded from the installed
  items, so design fixtures and previews never see arrivals.
- Update the doc comments that now overstate restoration:
  - `prepareForPresentation` in `RefsPanelModel.swift`.
  - `RefsPanelController.show` in `RefsPanel.swift`.
  - `RefsListView` in `RefsListView.swift` ("A show keeps the selected row when it
    remains listed").

### Docs (`docs/` is mounted into Bob's site)

- `docs/refs.md` § Updates, first paragraph: add the exception. A reference added since
  the panel was last shown — one whose `added` day is within the 3-day NEW window, so
  renames and moves don't count — makes the next ordinary show reset to fresh-launch
  defaults and select it (the newest batch first). Under `untouched`, also document the
  10-second untouched grace for arrivals brought in by the show's own pull. Tabs shows
  defer the reset to the next ordinary show. Selecting the reference first cancels it.
- `docs/refs.md` § The panel ("a show keeps the selected row when it remains listed")
  and § Opening Bob Refs (the "Ordinary shows restore…" bullet): mention the exception.
- `docs/refs-inspector.md` ("They survive a show."): say a new-reference reset clears
  them like Ctrl-Z.
- Run `just docs-check`.

### Tests

- New `Tests/RefsCoreTests/RefsArrivalsTests.swift` (portable). Use a fixed
  `RefsSignals(now:calendar:)` with a UTC Gregorian calendar and build `RefItem`s
  directly. Cover:
  - Seeded baseline; new ids become pending; a repeated observe is a no-op.
  - A vanished-then-returned id is never pending.
  - An empty seed means the first non-empty observe establishes the baseline without
    pending ids.
  - `markSeen` drops pending ids and never seeds the baseline. `clearPending` works.
  - Qualification inside and outside the 3-day window, and with nil `added`.
  - The latest batch wins over an earlier one; the earliest listing position wins within
    a batch; unlisted ids are ignored.
- Existing `RefsNewTests`/`RefsRankingTests` must stay green (proves the `arrivedNew`
  extraction is behavior-neutral).
- `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`. Use `makeRenameHarness` (extend
  it with an optional `now` closure for the `untouched` tests), `FakeRefsFetcher`, and
  `renameRecord(path:stem:added:)` with `todayDateString()`. Cover:
  1. Hidden arrival: remember a non-default view (query, Chats scope, Newest, a selected
     row). With `panelIsVisible = { false }`, publish a new record added today. A
     `.library` show then yields defaults, `selectedID` equal to the new id, and a
     bumped `inspectorGeneration`. Change the view again; the next show restores it
     (pending consumed).
  2. A new id with an old `added` (rename/move-like) or nil `added`: the show restores
     the remembered view.
  3. A `.highlightsTabs` show does not reset. A following `.library` show does.
  4. Two hidden batches: the later batch's id is selected.
  5. A successful in-panel rename does not trigger a reset on the next show.
  6. A visible arrival after the user selects it (touched) joins without moving the
     selection; after hide, the next show restores the view without a reset.
  7. Under `untouched`: an untouched visible arrival inside the grace window resets and
     selects it. One after a `setQuery`/`perform`, or after the injected clock passes 10
     s, only joins; the next show resets for it. Under `next_show`: a visible arrival
     only joins, and the next show resets.
  8. A `.scanResults` show keeps selecting Just scanned's first row and clears pending
     ids (the next `.library` show restores).
- Run the whole Refs suite in CI. Existing tests that add a recently `added` reference
  before a `.library` show and assert restoration may now see the intended reset. Update
  them only where the new rule is intentionally exercised, and say so in the commit.

## Verification

- This host may lack Swift. Read the `tailnet.md` reference memory
  (`sase memory read tailnet.md -r "..."`) for host access. Run `just portable-check`
  against a scratch rsync copy of the repo on athena, excluding `.build`/`.git`, then
  delete the copy. It builds and tests RefsCore, including the new tests.
- Best effort on the Mac, per `AGENTS.md`: rsync a scratch copy without `.build`/`.git`,
  run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
  Bryan's checkout or `just install`. If the Mac is offline, say in the final report
  that the macOS targets are unverified.
- `just docs-check` for the docs changes.
- Never land Swift that has not compiled. CI runs the BobMacCapture XCTest suite. After
  a red **Lint Swift formatting** step, check the later CI steps too.
- Commit the `bob-mac-capture` changes as one `feat(refs): …` commit (for example
  "feat(refs): reset to and select newly added references on show").

## Non-goals

- No bob-cli, bob-plugins, or vault changes. NEW marks, the open log, and reading state
  are untouched.
- No new setting or keybinding. Ctrl-Z, Escape, Tabs, and the scan notification keep
  their current behavior apart from clearing pending ids as described.
- Arrival state lives in memory only; quitting starts fresh, like the remembered view.
