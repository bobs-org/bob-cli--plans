---
tier: tale
title: Fix the hung and failing bob-mac-capture CI
goal:
  bob-mac-capture CI finishes green in minutes, and any future test hang fails fast and
  names the stuck test.
size: medium
decisions:
  hang_watchdog:
    ask:
      Should the CI Test step run `swift test` under a 15-minute watchdog that samples
      any hung xctest process before failing?
    choices:
      watchdog:
        A future hang fails in about 15 minutes, and the log and an artifact show the
        stuck test's stack
      timeout_only:
        Only add timeout-minutes; a future hang fails in about 25 minutes with no stack
        or test name
    default: watchdog
    why:
      XCTest output is buffered in CI, so these hangs were impossible to locate from the
      log alone
    answer: watchdog
  cancel_hung_runs:
    ask:
      Before pushing, should the coder cancel in-progress bob-mac-capture CI runs that
      are already hung in the Test step?
    choices:
      cancel: Frees the macOS runners now; those superseded commits show as cancelled
      leave:
        They keep holding runners until the 6-hour cap, which delays this fix's own CI
        run
    default: cancel
    why: run 327 sat queued for 3 hours behind earlier hung jobs
    answer: cancel
proposed_by: bbugyi200.athena.11d
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 13:04:36
status: wip
---

# Plan: Fix the hung and failing bob-mac-capture CI

## Problem

CI for the `bob-mac-capture` linked repo (`.github/workflows/ci.yml`, job "macOS 26
SwiftPM") has been red since run 271 (Oct 13). The last green run was 270 (`4da688e`).
There are three layers:

1. **Hang (runs 314–329):** since `936f232` ("harden Link to Today panel flow"), the
   Test step hangs until GitHub's 6-hour default job timeout cancels it. Hung jobs also
   hold the account's macOS runners, so later runs sit queued for hours (run 327 queued
   3 h, then hung 6 h). Runs 328 and 329 are hung right now.
2. **Crash (runs 302–313):**
   `Swift/ContiguousArrayBuffer.swift:692: Fatal error: Index out of range` aborted the
   whole xctest process inside `RefsPanelModelTests`.
3. **Ordinary failures:** about 15 tests fail. All of them are test-side bugs: stale
   expectations, missing fake-bob cases, timing races, or harness setup. None is a
   product regression; each was checked against the source, the README, and commit
   intent. Because of (1) and (2), every test that sorts after
   `RefsPanelModelTests.testRenameKeeps…` has not run in CI since run 296 (`9ac536c`,
   Oct 15). A static audit of those tests found 6 more test-side bugs, listed below.

The portable targets (CaptureCoreTests, RefsCoreTests) pass on Linux: 1329 tests, 0
failures. Only the AppKit `BobMacCaptureTests` target is broken.

### Root cause of the hang

`RefsPanelModelTests.testAtoBtoAPreviewErrorsCannotReplaceLatestPreviewOrWrite` (second
test in the class alphabetically) awaits `TodayLinkResponseGate.waitForRequests(1)`.
That is an untimed `withCheckedContinuation`, and `swift test` has no per-test timeout.

The request never comes because `makeRenameHarness` builds the model while
`library.items` is still empty:

- The listing starts empty and `selectedID` is nil.
- Later library publishes go through `libraryDidPublish` → `listing.refreshingContent`,
  which never adds rows. This is deliberate: the listing is frozen while visible.
- Only `prepareForPresentation()` rebuilds the listing, and the test never calls it.
- So `openTodayLinkCard()` and `openRenameCard()` return early (`RefsPanelModel.swift`
  ~1648-1655 and ~1787-1794), and `linkToToday` is never called.

In production, the panel always presents first (`RefsPanel.swift` ~104-111). Every
rename and Today-link test built on `makeRenameHarness` has the same missing step.

### Root cause of the crash

`testRenameKeepsRunningWhenHiddenAndShowsBobErrorsInline` subscripts `failures[0]`
(~line 640) after a failed wait leaves `failures` empty. The failed wait has the same
cause: the rename card never opened.

## Implementation

First open the repo with `/sase_repo` (`sase repo open bob-mac-capture -r "<reason>"`)
and read its `AGENTS.md`. All paths below are relative to that checkout. Line numbers
are approximate as of `902f0be`; locate code by test or function name.

General rules for every edit:

- Fix the tests, not the product code. Every diagnosis below found the production
  behavior correct.
- Inside `async` tests, never poll with `RunLoop.current.run(until:)`. Poll with
  `try? await Task.sleep(nanoseconds: 20_000_000)` up to a deadline.
- After a wait that can fail, read recorded arrays with `XCTUnwrap(x.first)` or
  `.first`, never `[0]`.

### 1. CI workflow (`.github/workflows/ci.yml`)

- Add `timeout-minutes: 60` to the `macos` job. A healthy job takes about 12 minutes,
  and queue time does not count toward the limit.
- Add `timeout-minutes: 25` to the **Test** step as a hard backstop.

> [!decision] hang_watchdog = watchdog Replace the Test step's `run:` with a watchdog
> loop.
>
> - Keep `set -euo pipefail` and the render-dir `mkdir`.
> - Also create `${{ runner.temp }}/test-hang` and expose it as an env var such as
>   `BOB_MAC_CAPTURE_TEST_HANG_DIR`.
> - Start `./Scripts/xcode-swift.sh test &`, save `$!`, and poll it with `kill -0` every
>   5 s.
> - Once `SECONDS` passes 900 (a healthy Test step takes about 5 minutes):
>   - Emit `::error::swift test ran longer than 15 minutes; sampling xctest`.
>   - For each `pgrep -x xctest` pid, run
>     `sudo -n sample "$pid" 5 -file "$dir/xctest-$pid.txt"`. GitHub macOS runners have
>     passwordless sudo, and plain `sample` may be refused for Apple-signed binaries.
>     Fall back to `sample` without sudo, and use `|| true` throughout.
>   - Print the first ~200 lines of each sample. The main thread's `test…` frame names
>     the stuck test.
>   - `pkill -x xctest || true`, kill the saved pid, then `exit 1`.
> - Otherwise `wait "$pid"` so the step keeps the test's exit status.
>
> Then:
>
> - Add `NSUnbufferedIO: "YES"` to the step env. This is best-effort, to make XCTest
>   progress lines flush; don't rely on it.
> - Add an `Upload test hang samples` step right after `Upload render fixtures`, with
>   `if: failure()`. Use `actions/upload-artifact@v4`, path
>   `${{ runner.temp }}/test-hang`, `if-no-files-found: ignore`, `retention-days: 14`.
> - Comment the loop the way the existing launch smoke test is commented.
>
> If the choice is `timeout_only`, add only the two `timeout-minutes` keys.

Lint the workflow with `actionlint .github/workflows/ci.yml` (installed on athena).

### 2. `Tests/BobMacCaptureTests/RefsPanelModelTests.swift` (the hang, the crash, and the hidden failures)

1. **Present before opening a card.** Go through every test built on
   `makeRenameHarness`. If it opens a Today-link or rename card, or reads
   `selectedItem`, without presenting first, add
   `harness.model.prepareForPresentation()` right after
   `await harness.waitForSnapshot(ids:)`. Grep to find them all. Known cases:
   - **Today-link tests:**
     - `testDelayedListingFailureCannotUnlockFastPathWrite`
     - `testStaleReloadBlocksOldChoicesUntilFreshRowsArrive`
     - `testAtoBtoAPreviewErrorsCannotReplaceLatestPreviewOrWrite`
     - `testClosingAndReopeningDropsOutOfOrderListingResults`
     - `testReturnSubmitsWithoutWaitingForDebouncedPreview`
     - `testTodayLinkUsesBobDefaultStaysOpenAndOverlaysTodayPill`
     - `testBobRefusalFixturesStayRefusedWithoutSubmittingWrites`
     - `testFailedCardRowTapRecoversAndFailureRefocusesField`
     - `testUnselectableReturnAndCommandNBeepAndHiddenWriteNotifiesOnce`
     - `testTodayLinkRefreshesTaskSnapshotTodayAndDiscardsInspectorCache`
     - `testTodayCardsRemainExclusiveAcrossRenameCopyAndEscapeAndRefuseOldBob`
     - `testHiddenTodayLinkFailureNotifiesOnce`
     - `testGuardedWriteDoesNotInventTodayPillAndFailedRefreshKeepsValidOverlay`
     - `testRepeatedTodayLinksEachKeepTheirOutcomeAndPill`
   - **Rename tests:**
     - `testRenameKeepsRunningWhenHiddenAndShowsBobErrorsInline`
     - `testRenameMapsOlderBobFailureToInlineUpdateMessage`
     - `testRenameUnchangedShowsAlreadyNamedStatus`
     - `testRenameUsesBobDestinationMigratesHistoryAndNotifiesWhenHidden`
     - `testRenameTextEditingKeysPassThroughControllerAndEscapeClosesCardFirst`
     - `testRenameFieldPrefillsStemAndTakesFirstResponderWithTextSelected`
     - `testRenameDuringScanFailsWithHumanBusyMessage` (present before `perform(.scan)`)
     - the `succeedRename` helper used by the `testRenameStatusClears*` tests

   Do **not** move the call into `waitForSnapshot`, because
   `testPrepareCountsEveryPresentation` asserts `presentationCount == 0` right after it.
   A small `waitForPresentedSnapshot(ids:)` helper is fine.

2. **Bound `TodayLinkResponseGate`.**
   - Replace the continuation-based `waitForRequests` and `waitForCompletion` with
     deadline polling: 5 s default, 10 ms `Task.sleep`.
   - Both take `file: StaticString = #filePath, line: UInt = #line`, call
     `XCTFail(…, file:line:)` on expiry, and return a `@discardableResult Bool`.
   - Make `resolve(_:with:)` also wait, with the same deadline, until request `index`
     exists before resuming it. Today it silently drops a resolve that arrives before
     the request.
   - Keep `respond()` semantics. Tests depend on late responses arriving after a cancel.
3. **Crash:** in `testRenameKeepsRunningWhenHiddenAndShowsBobErrorsInline`, replace
   `failures[0]` with `let failure = try XCTUnwrap(failures.first)`. In this file, also
   convert other bare subscripts of recorded or async-result arrays that follow a wait:
   `sections[1]`, `received[0]`/`[1]`, `orderedIDs[1]`, `actionsSections()[0]`. Use
   `XCTUnwrap` or a count guard.
4. **Wrong READING ref:** in `testStaleReload…`, `testAtoBtoA…` and
   `testReturnSubmitsWithoutWaitingForDebouncedPreview`, the asserts expect
   `.entry(ref: "5:ddf7159c")`. Change them to `"4:ddf7159c"`: write selectors come from
   `refs-today-listing.json`, where READING is `4:`. The portable `RefsTodayLinkTests`
   already uses `4:`.
5. **`testStaleReloadBlocksOldChoicesUntilFreshRowsArrive`:** the fake answers
   instantly, so the transient `.reloading` phase is never observed. Drive it with
   `TodayLinkResponseGate`, in this order:
   1. Present, then open the card.
   2. Resolve request 0 with the listing, then `selectTodayLinkRow(1)` and submit.
   3. Resolve request 1 with `refs-today-stale-pomodoro.json`.
   4. `waitForRequests(3)`. Assert `.reloading`, query `""`, and empty `pomodoros`.
   5. Edit the query to `"rea"`, submit, and assert it is still `.reloading` with
      exactly 1 non-dry-run call.
   6. Resolve request 2 with the listing and wait for `.ready`. Assert query `"rea"` and
      the selected pomodoro `READING`.
   7. Submit and wait for `.running`. Assert selector `4:ddf7159c`.
   8. Resolve the write so nothing is left pending.
6. **`testCloseBlockedWhileTodayLinkWriteRunsThenWorksAfterSettlement`:** add
   `await gate.waitForRequests(2)` before `gate.resolve(1, …)`. Without it the resolve
   races the write's `respond()`.
7. **`testClosedLastHidesPanelAndShowsLastToast`:** the model calls `tabCloseDismisser`
   when the last tab closes, but the harness only wires `panelDismisser`.
   - In the shared harness builders, add
     `model.tabCloseDismisser = { [weak harness] in harness?.dismissed += 1 }` next to
     `panelDismisser`.
   - First grep for tests that assert `dismissed` after a last-tab close. If one would
     change meaning, wire it in this test only.
8. **`testFailedCloseRestoresAndSelectsRowWithFailureBanner`:** set
   `harness.model.panelIsVisible = { true }` before the close. The banner is shown only
   while the panel is visible, and the default is `{ false }`.
9. **`testRenameUsesBobDestinationMigratesHistoryAndNotifiesWhenHidden`:** use
   `let openedAt = Date().addingTimeInterval(-3_600)` instead of
   `Date(timeIntervalSince1970: 1_700_000_000)`. `RefsOpenLogStore.append` drops events
   older than 365 days; this was reproduced on Linux.
10. **`testQueuedDigitCopiesWhenTheShowLoaderReturns`:** set
    `harness.model.copyCardNoteCapacity = 9` before `openCopyCard()`. The capacity
    defaults to 0 and only the view sets it after layout, so `.note(1)` never exists.
    This test has never passed.
11. **`testRenameFieldPrefillsStemAndTakesFirstResponderWithTextSelected` and
    `testTodayLinkFailureRefocusesItsFilterField`:** replace the `RunLoop.current.run`
    polling with `Task.sleep` polling. Inside an async test's main-queue job, a nested
    run loop does not drain the field's `DispatchQueue.main.async` focus retry. In the
    Today-link test, also wait about 100 ms after `show()` so the panel's 50 ms focus
    retry has run.

    Fallback, only if CI still shows no field editor: give `RefsRenameCardView`'s field
    the claim-on-`viewDidMoveToWindow` pattern from `CapturePickerFilterField.swift`.

### 3. Other Refs tests

- **`RefsPanelDesignTests.swift`:** the fixture row for `team_fit_review.pdf` has id
  `ref/chat/today_report.md`. Change the two lookups of `"ref/chat/team_fit_review.md"`
  to that id: `testRefsPanelStatesToPNG` (~1205) and `renderTabsCloseHoverRow` (~1376).
- **`RefsInspectorTests.swift`, `testCopyActionOpensCardAndCopiesItsSelectedRow`:**
  expect `draftStem == "team_fit_review"`. The rename card prefills `item.stem` (the PDF
  stem), per `RefsRename.swift` and the README.
- **`RefsInspectorTests.swift`,
  `testIntrinsicsDiscardDropsInFlightAndAllowsFutureLoads`:** replace the fixed 20 ms
  sleep with explicit synchronisation. On CI, `discard` runs before the first load
  captures its generation: count is 1, and the test takes under 0.2 s.
  1. Build a lock-protected probe with `started`, `finished` and `isOpen`. The reader
     calls `enter()`, polls until `isOpen`, then `finish()` and returns `pageCount: 7`.
  2. Start the first load in a `Task`. Poll with a deadline until `started == 1`. That
     proves the generation was captured.
  3. `await loader.discard(id:)`, open the probe, and assert the first value is 7.
  4. Run the second and third loads and assert `finished == 2`.

  Also convert the bare `[0]` subscripts near ~333-334 and ~592 to `.first` or
  `XCTUnwrap`.

- **`RefsFooterTests.swift:15`:** the `"↵ Open note"` expectation is missing
  `"⌘T Tabs  "` before `"esc Close"`. `1b7ead4` updated only the sibling assertion.
- **`RefsInspectorScrollTests.swift`,
  `testHostedShortContentRevealsWithNoOffsetChange`:** the "short" fixture overflows
  (doc 491 pt vs clip 462 pt), so Ctrl-D correctly scrolls instead of revealing.
  1. Give `makeHostedHarness` a `notesPageSize: Int? = nil` parameter that sets
     `model.notesPageSize`, and pass `1` here.
  2. Change the expectation to `visibleCount(total: 8) == 2`.
  3. Before Ctrl-D, and again after the 0.6 s settle, assert the document height is at
     most the clip height, with the existing inspector diagnostics as the message.
     Future growth then reads as "fixture no longer fits", not as a reveal bug.

### 4. Capture template tests

- **`BobMacCaptureTests.swift`, `testTemplatePlainNewlineInsertsOnlyNewline` (~2497):**
  range `(5, 4)` covers `"plat"`, so the correct result is `"@@tem\ne:\nTask {0}"`.
  Change the expected text; the selection assert `(6, 0)` already passes.
- **`Tests/Fixtures/fake-bob`, `capture-parse)` case:** add two branches before the
  default `else`, next to the existing template branches.
  - **Draft `"@@te"`:** return real bob output with a `global_route` span
    `{start 0, end 4}`. This lets `shouldRequestCompletion` reach the existing `"@@te"`
    `capture-complete` branch and fixes
    `CapturePanelModelTests.testTemplateHeaderCandidateReachableWithPrimedCache`.
  - **Draft `"${template_one_row_draft}"`:** `cat` a new fixture,
    `Tests/Fixtures/capture-parse-template-one-row.json`. It must carry a `template`
    object with `field_count: 1` and one row. This fixes
    `CaptureTemplatePresentationTests.testTemplateOneRowPreviewUsesBobsDiagnostic`.
  - Generate both payloads verbatim from the installed bob on athena:
    `bob capture-parse --format json -- '@@te'` and
    `bob capture-parse --format json -- $'@@template:\nTask {0}\n\nvalue'`. Match the
    style of the existing `capture-parse-template-*.json` fixtures.

### 5. Capture panel and agenda tests

Since `ee08ca7` ("Control-T saved-template shelf"), setting `processClient`, showing the
panel, and each successful capture all run `bob capture-templates`. fake-bob records
those calls, so call counts and "first call" waits are off. That behavior is intended,
so fix the tests:

- **`CaptureAgendaModelTests.swift` ~486 and `CapturePanelAgendaLayoutTests.swift`
  ~432:** `contains("argv=")` is satisfied by the template-list call. Wait instead for a
  recorded call whose `argv=` block does not start with `capture-templates`:
  ```swift
  components(separatedBy: "argv=").dropFirst().contains { !$0.hasPrefix("capture-templates") }
  ```
- **`CapturePanelModelTests`,
  `testDoubleSubmitIsSuppressedWhileOneMutationIsInFlight`:** count only `argv=` blocks
  with prefix `"capture "` (trailing space).
- **`CapturePullInChoiceModelTests.swift` :22, :29, :65, :143:** count `argv=` blocks
  with prefix `"capture "` instead of matching `"argv=capture"`, which also matches
  `capture-templates`. For the seed assertions (~:70-72), take the
  `BOB_PRIORITY_ROLL_SEED=` values only from those capture blocks. Leave
  `CaptureCoreTests/CapturePullInClientTests` alone; it passes.
- **`CaptureOpenHintTests.swift`:** in
  `testHoldingControlAloneRevealsAfterDelayAndClearsOnModifierChange` and
  `testSubmitAndDismissalClearReveal`, replace each fixed sleep before
  `XCTAssertTrue(model.openHintRevealed)` with polling every 5 ms for up to 2 s. The
  main thread is busy after the design-render tests, so 60 ms isn't enough.

### 6. Repo notes

Add one short paragraph to the repo's `AGENTS.md`, matching its tone:

- A Test step that dies at the watchdog is a hang. The sampled xctest stack in the log,
  or in the `test-hang` artifact, names the stuck test.
- Async tests must never await an untimed continuation or gate.

Adapt the wording if `timeout_only` was chosen.

## Verification and landing

1. On athena, run `just portable-check` in the opened checkout. Lint, build and portable
   tests must pass. Then run `actionlint .github/workflows/ci.yml`.
2. Best effort, per `AGENTS.md`: compile on the mac. rsync a scratch copy without
   `.build`/`.git`, run `./Scripts/bundle.sh` and `just typecheck-tests`, then delete
   the copy. Never use Bryan's checkout or `just install`. If the mac is offline, say in
   the final report that the macOS targets were unverified before CI.

> [!decision] cancel_hung_runs = cancel Before pushing, list in-progress runs with
> `gh run list -R bobs-org/bob-mac-capture --status in_progress`. Cancel, with
> `gh run cancel`, every run whose Test step has been running for more than 30 minutes.
> Those are hung on the bug this plan fixes.

3. This plan explicitly authorizes committing and pushing to the `bob-mac-capture`
   linked repo mid-turn with `/sase_git_commit`, as many times as needed, because CI is
   the only place XCTest runs.
4. After each push, hand the wait to `/sase_monitor`. For example, find the run for the
   pushed SHA and run `gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`.
5. If CI shows new failures, diagnose them from the log and fix them on the same
   principles. These would be tests that haven't executed since Oct 15, or estimates
   above that missed, such as the scroll-fixture fit or focus timing.
   - Allow up to 3 extra push/CI rounds.
   - A failure that is a genuine product regression needing a behavior decision is not a
     test-side bug. Don't guess: file a task bead with `/sase_new_task`, then report it.
6. Done when the CI run for the final commit is green through every step: Test, Bundle,
   Launch smoke test, Install and reinstall. `actstat --only-failures` must no longer
   list `bobs-org/bob-mac-capture`.

## Out of scope

- Limiting the post-capture saved-template refresh (`CapturePanelModel.swift` ~6840) to
  template captures. It is a possible bob-process saving, not a CI fix.
- Deadline-hardening the portable test barriers (`RefsCallGate`, `ReadBarrier`,
  `CloseReadBarrier`). They pass today, and the CI watchdog now bounds any future hang.
