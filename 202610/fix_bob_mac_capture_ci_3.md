---
tier: tale
title: Fix the red bob-mac-capture CI Test step
goal:
  Make the bob-mac-capture macOS 26 SwiftPM CI job green again by fixing the five tests
  that always fail and the two recurring flakes.
size: medium
decisions:
  relation_label_guard:
    ask:
      Also add a generation guard so a stale relation-label suggestion reply can't
      overwrite a newer one?
    choices:
      guard:
        A slow blank-label reply can no longer replace the typed-label rows in the card
      test_only: The product race stays; it is rare and clears on the next keystroke
    default: guard
    why:
      A few lines that close the product half of the same race; existing tests cover it
    answer: guard
proposed_by: bbugyi200.athena.13a
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-23 12:55:46
status: wip
---

# Plan: Fix the red bob-mac-capture CI (macOS 26 SwiftPM → Test)

## Context

Every push to `bobs-org/bob-mac-capture` master has failed CI since 2026-10-19 (36
consecutive runs; the last green run was
`c81e00a fix(ci): stabilize macOS SwiftPM Test suite…`). The job always fails at **step
6: Test**; Lint, Build, and the toolchain steps pass, and the 15-minute hang watchdog
never fires. Latest run: CI #394 (`3343c5e`,
https://github.com/bobs-org/bob-mac-capture/actions/runs/39970622855): 2895 tests, 8
failures.

Nobody can run `BobMacCaptureTests` locally: athena is Linux, and the Mac has only
Command Line Tools (no XCTest). Each commit since 10-19 typechecked but added a test
that never passed on CI, so the failures piled up. CI on GitHub's `macos-26` runner is
the only test oracle.

Five tests fail on **every** run. Each first failed in the commit that introduced it.
Two more fail **intermittently** and will keep CI red once the five are fixed:

| Test (Tests/BobMacCaptureTests/…)                                                        | Since              | Kind                          |
| ---------------------------------------------------------------------------------------- | ------------------ | ----------------------------- |
| `RefsListScrollHostedTests.testHostedGrownContentAboveTopStillShowsTop`                  | 61bdd22            | wrong fixture                 |
| `RefsPanelModelTests.testTabSwitchUsesTheSwitcherPreselectionOnceOnTheNextShow`          | 5935822            | product bug + wrong wait      |
| `CaptureTemplateStartTests.testTypingThirdAtStartsTemplateAndClosesBareAtAtCompletion`   | ad85b85            | missing fake-bob branch       |
| `RefsPanelModelTests.testPanelVisibilityChangeFiresWhenShownAndHidden`                   | 785508c            | product bug (re-entrant hide) |
| `CapturePanelSwitchHostedTests.testSwitchAwayAndBackPreservesDraftCaretTemplateAndError` | b47f28b            | brittle assertion             |
| `RefsPanelModelTests.testInspectorOffsetRejectsStaleSelectionAndGeneration`              | flaky (~4/27 runs) | test race                     |
| `CaptureSeparatorToggleTests.testDependencyTailCommitPreservesSuffixByteForByte`         | flaky (2/7 runs)   | fixture lock leak + lane race |

Open the repo with the `/sase_repo` skill (`sase repo open bob-mac-capture -r …`) and
read its `AGENTS.md` first. All paths below are relative to that checkout.

## Fixes

### 1. Re-entrant Refs `hide()` fires `onVisibilityChange` twice (product)

`Tests/BobMacCaptureTests/RefsPanelModelTests.swift:27` expects 2 changes and gets 3.
`RefsPanelController.hide()` (`Sources/BobMacCapture/Refs/RefsPanel.swift`, ~line 157)
calls `panel?.orderOut(nil)`. That synchronously resigns key, and
`RefsPanel.resignKey()` runs `onResignKey`, which `makePanelIfNeeded()` (~line 355)
wires to `self.hide()`. The nested `hide()` runs `panelDidHide()` and
`onVisibilityChange()`, then the outer call runs both again. In the app, `AppDelegate`
therefore syncs `highlightsTakeover` twice per hide.

Fix: add a re-entrancy guard to `hide()`, e.g. a `private var isHiding = false` with
`guard !isHiding else { return }; isHiding = true; defer { isHiding = false }`. The
resign-key callback fired by `orderOut` then becomes a no-op. Leave
`suspendHideOnResign` behavior unchanged. No test change.

### 2. A matched-tab switch never consumes the Tabs selection (product + test)

docs/refs.md (Tabs section) says: "After Bob Refs switches to a tab, the next ordinary
or Ctrl+Tab Tabs show uses the current Highlights tab as its preselection once."
`RefsPanelModel` implements this with `tabsSelectionConsumed`: when it is set,
`restoreTabsForPresentation()` rebuilds without preserving the selection. Only
`selectOpenTab(_:)` sets it, and that path is used for **synthetic** (collision/none)
rows. A **matched** row opens through `openSelected(.highlights)` →
`openInHighlights(item, recordsOpen: false, switchedFileName:)` →
`finishHighlightsOpen`, which records the switch (`tabsSession.recordSwitch`) but never
sets `tabsSelectionConsumed`. On the next show the switched row stays selected. CI shows
this at lines 3176/3183, where `small_book` stays selected instead of the preselected
`small_opened`.

- Product fix (`Sources/BobMacCapture/Refs/RefsPanelModel.swift`, `finishHighlightsOpen`
  success branch): when `switchedFileName` is non-nil, set
  `tabsSelectionConsumed = true` next to `tabsSession.recordSwitch(fileName:)`. Set it
  only on success, mirroring `selectOpenTab`'s `.selected` case.
- Test fix (`testTabSwitchUsesTheSwitcherPreselectionOnceOnTheNextShow`, ~line 3164):
  the test waits for `reader.selectedTargets.count == 1`. Matched rows switch through
  the opener, not `HighlightsTabsReading.select`, so that wait always times out. This is
  the "model condition not met before timeout" failure reported at line 4820. Replace it
  with a wait for the switch to land:
  `await harness.waitForModel { $0.tabsSession.recency.lastActive(fileName) != nil }`,
  where `fileName = URL(fileURLWithPath: items[1].pdfPath).lastPathComponent`. The first
  snapshot only touches `items[0]`, the current tab, so this becomes true exactly when
  `finishHighlightsOpen` runs. `FakeOpener` calls its completion synchronously, but the
  finish runs in a later `Task`, so a wait on `opener.highlightsOpens` alone would race.
  Then assert `harness.opener.highlightsOpens.count == 1` and
  `reader.selectedTargets.isEmpty`.
- No other test does a matched switch followed by a re-show (checked), so nothing else
  should change.

### 3. fake-bob has no `capture-parse` branch for a bare `@@` (fixture)

`CaptureTemplateStartTests.swift:122` times out waiting for `model.completionVisible`.
The fake's default `capture-parse` reply has `needs: []` and `spans: []`, so
`CapturePanelModel.shouldRequestCompletion` returns false and `capture-complete` is
never called. (`leadingSingleAtRouteReplacementRange` excludes `@@` by design.) Real bob
answers `@@` with `needs: ["route"]` and an `interactive_placeholder` span.

- Add `Tests/Fixtures/capture-parse-template-bare.json` containing real
  `bob capture-parse --format json -- @@` output, verbatim. Captured from bob 0.3.0
  (re-capture with the current bob if available, keeping it verbatim):

  ```json
  {
    "ok": true,
    "schema_version": 1,
    "input": "@@",
    "body": "",
    "mode": "incomplete",
    "route": null,
    "section": null,
    "block_id": null,
    "needs": ["route"],
    "spans": [{ "start": 0, "end": 2, "kind": "interactive_placeholder" }],
    "diagnostics": [
      {
        "severity": "error",
        "code": "missing_capture_item",
        "message": "global destination declaration has no capture item; add a capture item to this draft",
        "range": [0, 2]
      }
    ],
    "global_destination": {
      "range": { "start": 0, "end": 2 },
      "line": 1,
      "mode": "incomplete",
      "route": null,
      "block_id": null,
      "needs": ["route"]
    }
  }
  ```

- In `Tests/Fixtures/fake-bob`, `capture-parse)` branch, add
  `elif [[ "${draft}" == "@@" ]]; then cat "${fixtures_dir}/capture-parse-template-bare.json"`
  next to the existing `@@te` branch. Add a short comment noting that this is real bob
  output, matching the neighbouring branches.
- The rest of the test should then pass on product code that already exists. The first
  half now reaches `capture-complete-template-bare.json`. The template-start rewrite
  path is already proven by the passing sibling
  `testTypingThirdAtKeepsFollowingRowsBelowFreshTemplateLine`, and
  `applyNativeSingleEditRewrite` already dismisses a visible route completion for
  `CaptureTemplateStart.rewriteRule`.
- Re-check the other `@@` tests in that file. `testNativeUndoRedo…` ends on `@@` and now
  issues one `capture-complete`; its assertion counts only `capture-rewrite`, so it is
  unaffected. `CaptureParseResponse.mode` is a `String`, and unknown keys like
  `global_destination` already decode (see `capture-parse-template-prefix.json`).

### 4. Brittle whole-struct selection equality (test)

`CapturePanelSwitchHostedTests.swift:69` runs
`XCTAssertEqual(model.editorSelection, selection)`. After the test drives the AppKit
caret (`editorBefore.setSelectedRange`), SwiftUI's `TextEditor` writes back its own
`AttributedTextSelection` for the same insertion point. That value has offset 10 but
`affinity: .upstream` and a different index payload, so the struct comparison fails. The
caret really is preserved: line 72 (`NSTextView` `selectedRange`) passes.

Fix: replace line 69 with `XCTAssertEqual(model.collapsedSelectionUTF8Offset(), 10)`
(internal API already used by other tests). Keep line 72.

### 5. Just scanned fixture marks rows already at the top (test)

`RefsListScrollHostedTests.swift:59` asserts that the first row changed after a Just
scanned section arrives. `installScannedFixture` marks `orderedIDs.prefix(3)`, and
`RefsRanking.browseSections` keeps the scan mark's own order, so the new first row is
the old first row (`hosted_0`) and the assertion can never pass. Lines 58 and 60 already
pass.

Fix: mark rows that are _not_ at the top, e.g.
`Array(harness.model.listing.orderedIDs.suffix(3))`, so the section really grows above
the old first row, as the test name and comments intend. Update the
`installScannedFixture` doc comment. `prepareForPresentation(source: .scanResults)` →
`resetToDefaults` → `rerankSelectingFirst()` selects the new first row, so line 60 still
holds. If the `<= 0.5` offset wait now fails, it has found a real reset-to-top bug in
the Refs list when content is inserted above. Fix that in the product, using
`listResetGeneration` and the scroll-reset path that
`testHostedResetReturnsSelectionAndListToTop` exercises. Do not weaken the test.

### 6. Flake: inspector-offset test selects before the listing exists (test)

`RefsPanelModelTests.swift` ~line 2892 sometimes gets `nil` instead of `80`. The test
has no suspension points after `waitForSnapshot()`, so the only failure path is
`perform(.select(id: kept))` returning false. Its result is ignored. That happens
because `waitForSnapshot` waits on `library` state, while the model rebuilds `listing`
later in `scheduleLibraryDidPublish()` (a `Task` plus `Task.yield()`).

Fix: before selecting, `await harness.waitForModel { $0.listing.contains(id: kept) }`,
then `XCTAssertTrue(harness.model.perform(.select(id: kept)))`. Change only this test;
`testLibraryPresentationRestoresQueryScopeOrderAndSelection` already asserts its select.

### 7. Flake: relation-label typed-pipe tests (fixture + test + product)

Failure (`CaptureSeparatorToggleTests.swift:740` and `:746`): the wait for a typed "see"
suggestion times out, so the submit leaves `…|` without the label. Note that
`waitUntil`'s `#line` points at the line where the call starts, so `:740` is the "see"
wait, not the record-file wait above it. Sequence:

1. The typed `|` completion opens the card (`openRelationLabelPrompt`). The card
   immediately calls `fetchRelationLabelSuggestions()`. That starts request A, a
   blank-label probe at cursor 22, on the `completion` lane.
2. Within one poll, the test calls `updateRelationLabelPromptLabel("see")`, which starts
   request B on the same lane.
   `BobProcessClient.nextGeneration(…cancelsPreviousInLane:)` SIGTERMs A.
3. `Tests/Fixtures/fake-bob` (top of file) takes its record lock with an external
   `mkdir` loop **before** it installs the TERM trap. If SIGTERM lands while A's `mkdir`
   is running, or just after it returns, bash dies with the default action and the lock
   directory leaks. B, and every later fake-bob call in the test, then spins forever in
   the lock loop, so "see" never arrives. The fixture's comment claims the trap prevents
   exactly this leak, but there is a window before the trap exists.
4. Separately, the suggestion reply handlers in `fetchRelationLabelSuggestions` check
   only `draftSnapshot` and have no generation check. A late blank reply from A can
   overwrite B's typed rows.

Fixes:

- **fake-bob lock (fixture):** install the TERM/INT trap _before_ taking the lock, and
  make the lock owner-checked so the trap can release only its own lock. For example:
  take it with `ln -s "$$" "${FAKE_BOB_RECORD_PATH}.lock"` in the retry loop, and
  release it via a helper that runs `rm -f` only when `readlink` of the lock equals
  `$$`. Bash defers a trap until the foreground child (`ln`/`sleep`) exits, so the owner
  check closes the window. Keep the existing `FAKE_BOB_TERM_PATH` behavior: write
  "terminated", exit 143, and re-install the plain trap after the record block. Update
  the comment. `CaptureCoreTests` also use fake-bob, so `just portable-check` on Linux
  exercises this change.
- **Test ordering:** in the three typed-pipe tests that call
  `updateRelationLabelPromptLabel` right after the card opens, wait for request A's
  record line before updating the label. Once A has written its record it is past the
  lock, so B's SIGTERM is harmless and the order is fixed. The exact argv lines (`"$*"`
  keeps the literal `"`):
  - `testDependencyTailCommitPreservesSuffixByteForByte`:
    `argv=capture-complete --all-tasks --cursor 22 --format json -- Fix it ~tasks:target|" &tasks:source`
  - `testLeadingRelationPipeOpensWithNarrowRangeAndCommitsSee`:
    `argv=capture-complete --all-tasks --cursor 15 --format json -- ~tasks:target|" Fix it`
  - `testRelationLabelCommitPreservesFollowingRouteText`:
    `argv=capture-complete --all-tasks --cursor 24 --format json -- New task ~tasks:target|" @work`

  Derive each from `fetchRelationLabelSuggestions`: probe = head + `|"` + tail, cursor =
  `labelRange.start + 2`. Confirm each string against the code before relying on it.
  Leave the existing cursor-21/14/23 record assertions in place.

> [!decision] relation_label_guard = guard Add a `relationLabelSuggestionGeneration`
> counter to `CapturePanelModel`. `fetchRelationLabelSuggestions()` increments it and
> captures it. Both its success and error `MainActor.run` handlers return early unless
> the captured value is still current, so only the newest probe can change `suggestions`
> or `errorMessage`.

> [!decision] relation_label_guard = test_only Skip the product counter; do the fake-bob
> lock fix and the three test waits only.

## Verification

Follow bob-mac-capture's `AGENTS.md`; never land Swift you have not compiled.

1. On athena: `just format-lint` (or the swift-format command CI uses) and
   `just portable-check`. This covers the fake-bob change through `CaptureCoreTests`.
2. Best effort on the Mac (see `sase memory read tailnet.md`): rsync a scratch copy
   without `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then
   delete the copy. Never use Bryan's checkout or `just install`. If the Mac is offline,
   say so in the final report.
3. **CI is the real test run.** Push the work to a throwaway branch (e.g.
   `ci/verify-ci-fix-<date>`) and open a **draft** PR. That is how `c81e00a` was proven
   (throwaway PR #6). Wait synchronously for the `macOS 26 SwiftPM` run, e.g.
   `gh run watch <id> --exit-status` with a generous Bash timeout. Pull failures with
   `gh run view <id> --log-failed`. Iterate until the whole job is green: Test, Bundle,
   Verify plist and signature, Launch smoke test, Install and reinstall. Then close the
   PR and delete the branch.
4. Land the fix on master through the normal SASE commit flow and confirm with
   `actstat --only-failures` (or `gh run list`) that the master run for the new commit
   is green.

## Non-goals and follow-ups

- Older one-off failures that have not recurred in recent runs are out of scope:
  `RefsPanelModelTests.testOpenRefreshesTodayWithoutRefreshingSnapshot` (10-20 only),
  `CapturePanelModelTests.testPlusRightSideOffersTasksAndCaretAuthoredIdOpensNewIDPicker`,
  and `testInvalidBareXSelectionBlocksSubmissionAndClearsStaleCard`. If any failure
  other than the seven above shows up during verification, file it as a `flake` (or
  `ci`) task bead via `/sase_new_task` with the run link, rather than widening this
  change. Fix it here only if it blocks a green run and its root cause is as clear as
  the ones above.
- CI warnings (Node 20 action deprecation, the `var calendar` never-mutated warning in
  `CaptureAgendaLayout.swift:258`) are not failures and stay out of scope.
- No docs change is needed: fixes 1 and 2 make the code match docs/refs.md.
