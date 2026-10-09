---
tier: epic
title:
  "Bob Refs landing fixes: make search, error recovery, refresh, ranking, and the
  inspector match the bob_refs_panel spec"
goal: "Finish epic bob-cli-5s. Close every gap its land audit found between the shipped
  Bob Refs panel and plan:202610/bob_refs_panel.md: typed queries reach the model, an
  open error re-shows the panel with its state and working buttons, unavailable rows
  stay in place dimmed, refresh fires on wake and every open, ranking follows the spec
  tables in local days, the inspector tells the truth, the ⌘K menu anchors at the row,
  and bob-cli drops stray Swift build files and finishes the blocked-field contract.

  "
phases:
  - id: cli-blocked-polish
    title: bob-cli blocked-field polish and stray .build cleanup
    depends_on: []
    size: small
    description:
      "cli-blocked-polish: untrack the stray `.build/` files and ignore `/.build/`,
      serialize `blocked` directly after `reading_state_source`, add a
      several-trackers-with-one-[?] fixture test, and finish the docs/ref.md contract
      (client note, additive under schema_version 1)."
  - id: refs-core-fixes
    title: RefsCore ranking, captions, dates, and refs-rank fixes
    depends_on: []
    size: medium
    description:
      "refs-core-fixes: in bob-mac-capture RefsCore, keep vanished ids in
      refreshingContent as unavailable, let 2-character word prefixes reach T1, compute
      days in signals.calendar instead of UTC, sort Ready by added desc, mark git dates
      approximate, fix the weekday range, wire frecencyHalfLifeDays, cache prepared
      items per snapshot, and make refs-rank --now accept the documented form; with
      tests."
  - id: refs-model-fixes
    title:
      Search binding, open-error re-show, unavailable rows, refresh triggers, and live
      settings
    depends_on:
      - refs-core-fixes
    size: medium
    description:
      "refs-model-fixes: in bob-mac-capture, send typed text to the model, re-show the
      panel after an open error without resetting it and through BobPanelCoordinator,
      render unavailable rows per spec §5.5, refresh on wake, Today on every open, git
      dates in their own lane, ⌘R and Recheck Bob refreshes that re-rank, and live
      settings re-registration that reads the new value; with tests."
  - id: refs-ui-fixes
    title: Panel visuals, inspector honesty, ⌘K anchor, cleanup, README, and final CI
    depends_on:
      - refs-model-fixes
      - cli-blocked-polish
    size: medium
    description:
      "refs-ui-fixes: in bob-mac-capture, add the content well, Reduce Transparency
      base, and per-show scale-in; fix inspector honesty rules, the 3 s intrinsics
      timeout, and the ⌘K anchor; render stem/secondary caption ranges; fix row
      VoiceOver labels; remove dead symbols and debug renders; make the README's Bob
      Refs section coherent; confirm final CI and render fixtures."
proposed_by: bbugyi200.apollo.bob-cli-5s.land
parent_bead: bob-cli-5s
create_time: 2026-10-09 08:04:25
status: wip
---

- **PROMPT:**
  [prompts/202610/bob_refs_land_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_refs_land_fixes.md)
- **PARENT:**
  [202610/bob_refs_panel.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_refs_panel.md)

# Bob Refs landing fixes

## Why

Epic `bob-cli-5s` built the Bob Refs panel in Bob Mac Capture from
`plan:202610/bob_refs_panel.md` (the **parent plan**). All nine phases closed and macOS
CI is green at bob-mac-capture `2016864` (run 37919892089). The epic's land audit read
the parent plan against the code and found defects CI cannot see, because they live in
AppKit wiring the tests do not exercise or in tests that pin the wrong behavior. The
worst: **typing in the search field never reaches the model**, so search does not work.

This plan fixes only those gaps. The parent plan's design specification (its §1–§15)
stays the contract. Every section reference below (§N) means the parent plan. Read the
sections a phase cites before changing code. The parent plan's DECISIONS still hold:
`highlights_open_key = cmd_o`, and no `decisions` memory edits
(`refs_decision_memory = no`). Do not edit any memory note.

## Ground rules (every phase)

- **Repositories.**
  - `cli-blocked-polish` works in **bob-cli** (this project).
  - The other phases work in the linked repo **bob-mac-capture**. Open it with
    `sase repo open bob-mac-capture -r "<reason>"`. If that fails because the linked
    repo's primary checkout is missing on this host, use
    `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`. Work only in the path
    the command prints, and read that repo's `AGENTS.md` if it names one.
- **Same rules as the parent plan.** Its "Ground rules" section applies unchanged: thin
  client, opening never mutates, the JSON contract, Swift 5 mode and conservative APIs,
  `@MainActor` on AppKit types, swift-format (4-space indent, ≤ 100 columns), privacy,
  neighbor-matching tests, synthetic fixture titles only.
- **Commit and CI loop.** Each bob-mac-capture phase commits to `master` with
  `/sase_git_commit` from that checkout, using conventional headers (`fix(refs): …`). It
  then finds the CI run
  (`gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`),
  watches it through `/sase_monitor`
  (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`), and fixes forward
  until the whole job is green. On failure, read `gh run view <id> --log-failed` and
  grep for ` error:`. A phase is not done until its CI run is green. Its bead note
  records the run URL and SHA.
- **Known flake.** Fake-bob live-preview `waitUntil` timeouts in
  `CapturePanelModelTests` (tracked by task `bob-cli-4k`) can fail CI on untouched code.
  If one fails, rerun the failed job once (`gh run rerun <id> --failed`). If the same
  test fails twice in a row, note it on the phase bead and keep going. Do not change
  those tests here.
- **Look at the pixels.** Every phase that changes Refs visuals downloads the CI
  artifact `render-fixtures`
  (`gh run download <id> -R bobs-org/bob-mac-capture -n render-fixtures -D <tmpdir>`),
  opens its `refs-*` PNGs with the Read tool in light and dark, and checks them against
  §7–§10 before declaring done. Keep the ImageRenderer workarounds from `bob-cli-5s.6`:
  live glass through `NSGlassEffectView` (SwiftUI `.glassEffect` blanks snapshots),
  static preview stacks instead of `ScrollView` content, and plain button styles.
- **Tests are part of every fix.** Each fix below names the test that proves it. When a
  test currently pins the wrong behavior, change the test to pin the spec and say so in
  the commit message.
- **README.** Keep the README's `## Bob Refs` section true after each phase. Any claim a
  phase makes true or false gets updated in that phase.

## Phase `cli-blocked-polish`: bob-cli blocked-field polish and stray `.build` cleanup

**Repo:** bob-cli.

1. **Stray Swift build files.** Commit `a4c69ff`
   (`chore(build): record Swift build cache from RefsCore verification`, from
   bob-cli-5s.4) committed `.build/.buildSystem_debug` and `.build/CACHEDIR.TAG` to
   bob-cli. bob-cli has no Swift code. Remove both from the index (`git rm --cached`
   plus deleting the untracked directory), and add `/.build/` to `.gitignore` next to
   `/target/`.
2. **Key order.** The parent plan's `cli-blocked` phase puts `blocked` on `RefRow`
   (`src/native/ref_library/row.rs`) **directly after `reading_state_source`**. It
   currently sits after `legacy_status`. Move the field so `serde` emits it right after
   `reading_state_source`. Check that `ref show` and `ref find` still emit it (`ShowRow`
   flattens `RefRow`).
3. **Several trackers, one `[?]`.** `decide_status` (`status.rs`) sets `blocked` only
   when there is exactly one `^ref` tracker and its mark is `?`. No fixture covers
   several trackers where one is `[?]` (`two_trackers.md` has `[ ]` and `[x]`). Add a
   synthetic fixture under the ref-library test vault with two `^ref` trackers, one of
   them `[?]`. Assert `blocked == false` in `src/native/ref_library/tests.rs` and in
   `tests/cli/ref_library/list.rs`. Update any hard-coded fixture counts the new file
   shifts.
4. **`docs/ref.md`.** Next to the existing `blocked` paragraph, add:
   - that clients such as Bob Mac Capture's Refs panel read Blocked from this field and
     never parse `reading_state_source`;
   - in the JSON envelope's row field list, that `blocked` is additive under
     `schema_version` 1.

**Done when:** `just check` passes except the 9 known
`native::highlights_ref::return_links` failures tracked by task `bob-cli-5t` (Pandoc
3.1.3 link form; leave them alone), `git ls-files .build` prints nothing, and
`cargo run -q -- ref list -R all -A -f json` shows `"blocked"` immediately after
`"reading_state_source"` in each row.

## Phase `refs-core-fixes`: RefsCore ranking, captions, dates, and `refs-rank`

**Repo:** bob-mac-capture, `Sources/RefsCore`, `Sources/refs-rank`,
`Tests/RefsCoreTests`. All of it is Foundation-only. If a Swift toolchain exists on the
host, run `swift build` and `swift test --filter RefsCoreTests` on a scratch copy before
pushing.

1. **Unavailable rows stay in place (§5.5).** `RefsListing.refreshingContent`
   (`RefsRanking.swift`, about line 264) drops ids that vanished from the new snapshot.
   The spec says a frozen id that is gone stays at its index, marked unavailable.
   - Keep vanished ids in `orderedIDs` and their sections. Expose them, for example
     `unavailableIDs: Set<String>` on `RefsListing`, together with the last known
     `RefItem` content so the row can still draw its title.
   - A fresh listing (query or scope edit, ⌘R, next open) drops them.
   - Tests in `RefsRankingTests`: a vanished id keeps its index and is reported
     unavailable; the next fresh listing omits it.
2. **2-character word prefixes reach T1 (§4).** `RefsRanking.swift` (about line 1348)
   returns `.fuzzy` for every 2-character token. The spec's Word rule has no length
   limit: a token that is a prefix of a title or stem word is a Word match (T1). Only
   when it is not a word prefix does the 2-character contiguous-substring rule apply
   (T2). One-character tokens stay word-prefix only. Change `testShortQueryPolicy` to
   pin this: `om` is T1 for a title word starting "Om…", and T2 for a mid-word "…om…"
   substring. Update the README's ranking paragraph.
3. **Local days, not UTC (§3, §4).** The ranker's day ordinal (`RefsRanking.swift` about
   lines 434–437 and 537) uses UTC days and ignores `signals.calendar`. That shifts Just
   added, Recently opened, the R recency term, and Library activity by a day in US
   evenings, while captions use the local calendar. Compute every day count with
   `signals.calendar` (start of day in that calendar's time zone). Add a test with a
   `America/New_York` calendar at 21:00 local time, where UTC is already the next day: a
   row added "today" local stays in Just added and its recency matches day 0.
4. **Ready sorts by added desc (§3).** Ready uses `browseLane`, which sorts by last
   opened first. The spec's Ready order is added desc, with blocked rows last and the id
   tie-break. Next and Reading keep last-opened-then-added. Pin it in the browse golden
   test.
5. **Git dates are approximate (§2, §10).** `RefsModel.swift` (about lines 349–352) sets
   `addedIsApproximate = false` when `added_source == "git"`. Any git-sourced added date
   is approximate, so its caption and inspector row carry "≈". The golden
   `git_added_note` fixture row stores a git date in the plain list, which live `bob`
   never produces. Move that date into the `-g` merge path (`gitAddedDates`) so the
   golden test exercises the real flow, and assert "≈" in the caption.
6. **Weekday range (§8 captions).** `RefsCaption.swift` (about line 171) uses `2...7`,
   so a date exactly 7 days old shows today's weekday name. Use `2...6`, and add a
   caption test for 7 days old.
7. **Wire the frecency half-life.** `RefsRankingConstants.frecencyHalfLifeDays` is never
   referenced. `RefsStores.swift` (about line 320) hard-codes 14. Use the constant.
8. **Cache prepared items per snapshot.** `RefsPreparedItem`s are rebuilt on every
   `listing` call (about line 743), although the doc comment says the caller caches
   them. Add a small cache keyed by snapshot identity, plus any inputs that change the
   prepared fields (outline headings from the inspector). Rebuild only when the snapshot
   or those inputs change. Keep the existing perf guard test, and add one that asserts
   two listings over the same snapshot reuse the prepared items.
9. **`refs-rank --now`.** `Sources/refs-rank/main.swift` (about line 72) parses `--now`
   with `.withInternetDateTime`, which rejects the documented `2026-10-08T09:00:00`. It
   also hard-codes a UTC calendar. Accept both a full ISO 8601 time with an offset and a
   local date-time without one, interpreted in a `--tz <identifier>` time zone (default:
   the current time zone). Use that calendar for ranking. Align output columns as the
   README describes, or fix the README to match.

**Done when:** CI is green; the new and changed tests pass; the README's Sorting and
Tuning paragraphs match the code. Record `INTERFACE CHANGE:` on this phase's bead for
any public RefsCore name the next phase must use (for example the unavailable-ids
accessor).

## Phase `refs-model-fixes`: search binding, error re-show, unavailable rows, refresh, settings

**Repo:** bob-mac-capture, `Sources/BobMacCapture` (`Refs/`, `AppDelegate.swift`,
`HighlightsTakeover.swift`, `BobPanelCoordinator.swift`) and `Tests/BobMacCaptureTests`.
Read this phase's predecessor bead notes for `INTERFACE CHANGE:` entries first.

1. **Typed text reaches the model (critical).** In `RefsSearchBar.swift` (about lines
   83–90) the `onTextChange` closure drops the typed string and calls
   `queryDidChange()`. The coordinator never calls the binding setter, so `model.query`
   stays `""`, and the next `updateNSView` wipes the field. Set the query from the
   closure argument before re-ranking, as `CapturePickerView.swift` does. A
   `model.setQuery(_:)` method that sets the query and calls `queryDidChange()` keeps it
   in one place.
   - Test: build a `RefsFilterField`, make its coordinator, and send
     `controlTextDidChange` with an `NSTextField` whose `stringValue` is `"omni"`.
     Assert that `model.query == "omni"` and that the listing is in search mode.
2. **An open error re-shows the panel intact (§11, §13.4).** `finishHighlightsOpen` and
   `finishDefaultAppOpen` (`RefsPanelModel.swift`, about lines 710–740) set
   `pendingOpen` and call `panelPresenter`, which is `RefsPanelController.show()`.
   `show()` calls `prepareForPresentation()`, which clears query, scope, selection, and
   `pendingOpen`. So Try Again and Open in Default App do nothing, and the query is
   lost. Re-showing also bypasses `BobPanelCoordinator`, so Capture can be visible at
   the same time.
   - Add a re-present path that orders the panel front without
     `prepareForPresentation()`. It keeps the query, scope, frozen listing, selection,
     and `pendingOpen`, and sets the banner. Route it through `BobPanelCoordinator`, so
     Capture closes (retaining its draft) exactly as when Refs opens normally.
   - Fix `testHighlightsErrorRepresentsWithBannerAndKeepsQuery`. Its fake presenter must
     exercise the real re-present semantics, and the test asserts that query, selection,
     and `pendingOpen` survive and that Try Again re-dispatches. Add the same test for
     the default-app error.
3. **Unavailable rows render and refuse to open (§5.5, §11).** Use the listing's
   unavailable ids from `refs-core-fixes`.
   - `rowContent` (`RefsPanelModel.swift` about line 378) returns content for an
     unavailable id from the last known item. The row draws at 0.45 opacity with the
     caption "No longer in your library".
   - ↵, Tab, double-click, ⌘↵, and ⌥↵ on an unavailable row show that message (banner or
     footer), and never open anything. Navigation keeps the selection on that id, and ↓
     moves to the next row, not the top.
   - The inspector shows the title and the message instead of "Select a reference".
   - Replace the assertion in `RefsPanelModelTests` (about lines 68–71) that pins the
     drop, and add tests for the Return refusal and for navigation.
4. **Wake refresh.** `RefsLibrary.swift` (about line 467) observes
   `NSWorkspace.didWakeNotification` on `NotificationCenter.default`. It is posted only
   on `NSWorkspace.shared.notificationCenter`. Observe there, behind an injectable
   `NotificationCenter`, and add a test that posts the wake notification and sees one
   `.wake` refresh.
5. **Today on every open (§1).** `RefsPanelController.show()` only calls
   `refreshIfStale`. The spec refreshes Today (`bob plan -f json`, lane `refs-plan`) in
   the background on every panel open. Add that call, keeping the snapshot's 60 s
   staleness rule. Test that two opens 1 s apart run two Today refreshes and one
   snapshot refresh.
6. **Git dates in their own lane (§1).** `RefsLibrary.swift` (about lines 365–375) runs
   the `-g` pass inside the `refs-list` pass. A `-g` failure throws away a good snapshot
   and records `.failed`, and the first paint waits for `-g`. Publish the snapshot
   first. Then, only if some row has no `added` and its path is not yet in
   `gitAddedDates`, run the `refs-git` lane. Merge only values whose
   `added_source == "git"` into rows whose `added` is null, then apply them as a content
   refresh. A `-g` failure keeps the snapshot and the refresh state as they are; log it
   only. Tests: a failing `-g` leaves the published snapshot and a non-failed state; the
   lane does not run when every row has a date.
7. **⌘R and Recheck Bob refresh, then re-rank (§1, §5.2, §12).** ⌘R
   (`RefsPanelModel.swift` about lines 321–324) re-ranks at once with the old data, so
   the refreshed data never reorders the list. Make ⌘R refresh library, Today, and git
   dates, then build a fresh listing (keeping the selected id when it still exists) when
   that refresh completes. Recheck Bob (`AppDelegate.swift` about lines 598–606) must
   also refresh the snapshot with reason `.recheck`. Capture success uses
   `.captureSuccess` for its Today refresh, and the 10-minute timer uses `.timer`.
   Delete any `RefsRefreshReason` case still unused after that. Tests cover ⌘R
   reordering after new data, and Recheck triggering a snapshot refresh.
8. **Live settings read the new value.** `AppDelegate.observeRefsSettings()` (about
   lines 515–524) sinks `@Published` publishers, which emit before the property changes,
   and then re-reads `settings.*`. It always sees the old value: turning ⌃⇧⌘R off leaves
   it registered, and turning it on unregisters it. Pass the value the sink receives
   into `registerRefsHotKey(enabled:)` and `highlightsTakeover?.sync(key:appPath:)`, or
   deliver on the next main-queue turn. Make the README's "re-registers immediately"
   claim true.
   - Missing tests from the parent plan's `refs-entry-points` phase, in
     `RefsEntryPointsTests.swift`:
     - toggling `refsHotkeyEnabled` off and on registers and unregisters ⌃⇧⌘R live;
     - changing `refsHighlightsOpenKey` on a running takeover swaps ⌘O for ⌃O, and Off
       unregisters it;
     - `NSWorkspace.didTerminateApplicationNotification` for Highlights unregisters the
       takeover;
     - capture success refreshes Today and marks the snapshot stale.
9. **Copy Diagnostic on the load-failed card (§11).** It copies `lastDiagnostic`, which
   only open errors set. Copy the refresh failure's bounded message, through the
   injected pasteboard (not `NSPasteboard.general` directly).

**Done when:** CI is green and every listed test exists and passes. The README's Updates
and Opening paragraphs match the new behavior (wake, Today on open, git lane, ⌘R,
unavailable rows, error re-show).

## Phase `refs-ui-fixes`: visuals, inspector, ⌘K anchor, cleanup, README, final CI

**Repo:** bob-mac-capture. Read the predecessor bead notes first.

1. **Content well (§7).** `RefsPanelView.swift` (about lines 203–236) has no content
   well. Wrap the list and inspector in an r=12 `.regularMaterial` well with a 0.5 pt
   `primary.opacity(0.08)` stroke, inset 8 pt from the glass. Use
   `RefsVisualTokens.wellRadius`. Compute the list width and the 760 pt inspector cutoff
   from the **panel** width (§6: list `round(width × 0.52)`, inspector when width ≥
   760), not the width left after padding.
2. **Reduce Transparency and motion (§6, §15).** Under Reduce Transparency, paint an
   opaque `Color(nsColor: .windowBackgroundColor)` rounded rect (r=20) beneath the
   content. The SwiftUI content scales from 0.98 to 1.0 with `.easeOut(duration: 0.12)`
   on **every** show, not once per process: drive it from a presentation counter the
   model bumps in `prepareForPresentation()`. Under Reduce Motion, fade only. The
   divider honors Increase Contrast. Add a design render under Reduce Transparency.
3. **Inspector honesty (§10).**
   - Omit a fact row whose value is unknown. Never print "Unknown date" (about line 293
     of `RefsInspectorView.swift`).
   - The Added value must not repeat its label ("Added Added Oct 8", about line 299).
   - Omit the Tasks row when there are no open tasks.
   - The missing-PDF callout reads "↵ opens the note instead", with the ↵ glyph.
   - Show the ABSTRACT block for papers only (`RefsInspectorLoader.swift` about lines
     451–459). Update the README sentence that says otherwise.
4. **Intrinsics timeout (refs-inspector phase).** `loadWithTimeout`
   (`RefsInspectorLoader.swift` about lines 143–154) never times out: `withTaskGroup`
   waits for every child, and the PDFKit read runs synchronously on the actor. Run the
   PDFKit read in a detached task. Return "Preview unavailable" when 3 s pass, without
   waiting for the read, and let the abandoned read finish off the actor. Do not cache a
   timeout permanently: a later selection may retry. Also:
   - write `intrinsics.json` and thumbnails with mode 0600 in a 0700 directory, as the
     other Refs stores do;
   - delete a thumbnail file when its entry is evicted;
   - bound the in-memory `content` and `thumbnails` dictionaries with the same LRU
     capacity as the show cache.
   - Test with an injected slow reader: the result arrives in about 3 s, and the next
     load is not queued behind it.
5. **⌘K anchors at the selected row (§10 actions menu).** The spec: "an `NSMenu` popped
   up below the selected row; when the row frame is unknown, at the list's center."
   `RefsPanelController.actionsAnchor` (`RefsPanel.swift` about lines 155–171) uses the
   mouse position instead. Track the selected row's frame in the panel's coordinate
   space (for example a SwiftUI anchor preference published to the model). Pop the menu
   below it, and fall back to the list center. Drop the mouse heuristic. The first three
   menu items show their shortcut hints (↵, ⌘↵, ⌥↵) like Refresh shows ⌘R.
6. **Search captions show stem and secondary matches (§4).** `RefsRowView.swift` (about
   line 170) ignores the stem and secondary highlight ranges. When a token matched only
   the stem, the caption shows the stem with its ranges in the accent color. On a T3
   row, it shows the matched author, area, or heading with its ranges. Do not repeat a
   matched author that the caption already shows. Add a design fixture with a stem-only
   match.
7. **Row accessibility (§15).** The row's VoiceOver label (`RefsRowView.swift` about
   lines 219–236) says the kind once, says "pages" (not "pp"), and includes the Today
   Pomodoro name.
8. **Dead symbols and debug leftovers.** Wire up or delete each of these so nothing
   public goes unused outside tests:
   - `RefsPanelController.toggle()`. Wire it to the "global hotkey or takeover key while
     visible: close" rule (§12) if that path doesn't already close the panel; otherwise
     delete it.
   - `RefsKeyContext.bannerVisible`. Use it in the Esc ladder, or delete it.
   - `RefsBanner.Action.retry`, if production never shows it.
   - `RefsVisualTokens.heroTileRadius` and `kindTileRadius`. Use them in the tiles or
     delete them (`wellRadius` gets used by item 1).
   - `RefsLibrary.isSafePDFPath`. Use the RefsCore helper instead.
   - `RefsHighlightsOpenKey.displayName`. Use it for the Settings picker labels instead
     of the hard-coded strings.
   - `RefsPanelModel.presented`, which nothing reads.
   - Delete the `refs-piece-*` debug renders in `RefsPanelDesignTests.swift` (about
     lines 365–421, added by `f14e098` to localize blank views), including the "Boom."
     banner fixture.
9. **README coherence (closeout).** Make the README's `## Bob Refs` section one coherent
   user-facing contract:
   - delete the stale "Later phases add…" lines;
   - drop the Sorting sentence that repeats Updates;
   - keep a one-sentence statement of the ranking-constant change (dropped prior −0.20
     instead of −0.10) and its reason, as the parent plan requires, without
     implementation trivia;
   - add a short Privacy note listing the Refs files (open log, snapshot cache,
     intrinsics cache, thumbnails) with their locations and modes, the References
     `UserDefaults` keys, and that signposts carry names only;
   - add a short Troubleshooting note for "Bob couldn't load your references",
     "Highlights not found", and how to reset open history;
   - check every claim against the code as it stands after all three phases.
10. **Final verification.** CI green on the final SHA. Download `render-fixtures` and
    review every `refs-*` PNG in light and dark: well, sections, rows, unavailable row,
    stem-match caption, inspector (chat, paper, encrypted, missing PDF), banner, and
    Reduce Transparency. Fix misalignment, clipping, contrast, or truncation. Then
    append a short note to this phase's bead updating Bryan's Mac checklist from
    `bob-cli-5s.9` note #3 with what changed: typing filters live; an open error keeps
    the query; a deleted reference stays dimmed; and toggling ⌃⇧⌘R in Settings applies
    at once.

**Done when:** CI is green on the final SHA, the fixtures are reviewed, no epic-added
public symbol is unused outside tests, and the README is coherent.

## Out of scope

- The 9 `return_links` Pandoc failures (task `bob-cli-5t`), the fake-bob live-preview
  flake (task `bob-cli-4k`), and the Bob Refs decisions record (task `bob-cli-5u`).
- Any new feature beyond the parent plan's specification.
- Closing `bob-cli-5s` itself. This epic's `parent_bead` link (stamped by SASE) hands
  the landing back to `bob-cli-5s`'s land agent after this epic lands.
