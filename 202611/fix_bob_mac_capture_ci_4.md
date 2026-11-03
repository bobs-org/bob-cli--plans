---
tier: tale
title: Restore green macOS CI for bob-mac-capture
goal:
  bob-mac-capture's macOS 26 SwiftPM job passes on master again, with every one of the
  23 failing tests fixed at its root.
size: medium
decisions:
  fixture_source:
    ask:
      Regenerate the real-bob directive fixtures from current bob instead of
      hand-editing their grammar?
    choices:
      regenerate:
        Rerun the generator on bob-cli master; 66 one-line diffs, fixtures stay truthful
      hand_bump:
        Hand-set grammar 5 in three clip parse fixtures; smallest diff, but no longer
        real output
    default: regenerate
    why:
      A trial regeneration changed only grammar, a null drawer, and one template preview
    answer: regenerate
  fake_bob_scope:
    ask: Move every inline fake-bob body that hard-codes grammar 3 to grammar 5?
    choices:
      all: Bump all 67 literal grammar-3 bodies to match the injected default 5
      failing_only:
        Bump only the ^sase:deep-fix and @file:ready bodies the failing tests hit
    default: all
    why:
      fake-bob simulates current bob; gate tests use injected grammar, never a literal 3
    answer: all
  pull_in_order:
    ask:
      Restore legacy Pull-In section order when Bob lists only some groups and sends no
      copy?
    choices:
      product:
        Fix CapturePullInChoice so keep-moving ranks last again for copy-less partial
        lists
      test: Keep always-Bob-order and rewrite the test to expect keep-moving first
    default: product
    why: Current bob lists every group with copy, so only older payloads change
    answer: product
  line_case_undo:
    ask: Keep AppKit's native caret placement after undoing Ctrl-G U?
    choices:
      native:
        Drop the post-undo caret assertion, as docs/keyboard.md promises native undo
      restore:
        Register selection restoration so undo returns the caret to its pre-toggle spot
    default: native
    why: The documented contract is native undo, and AppKit leaves the caret at {5,0}
    answer: native
proposed_by: bbugyi200.athena.17r
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-03 17:08:53
status: wip
---

# Fix bob-mac-capture's red macOS CI

## Context

`bobs-org/bob-mac-capture` has had no green `CI` run on `master` since at least
2026-10-30. Its only job, **macOS 26 SwiftPM**, fails at the **Test** step. Every recent
run fails the same 23 tests (52 assertion failures, out of 3799 tests). Runs checked:
41714104187 (b9896cc, the current HEAD), 41692235919, 41676545644, 41570850719, and
precommit snapshot 41709503302.

All work happens in the linked repo `bob-mac-capture`. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "..."`), use the printed path, and read its
`AGENTS.md` first. It requires that Swift is compiled, that `.swift-format` style is
used only on changed lines, and that a `test-hang` sample is checked if the Test step
times out.

### Prior attempt (reference only)

Earlier today, another agent pushed two `Scripts/macos-ci-check` snapshots but never
landed them: `5ac5db653d641b9d12fbea554646df4e35694c4c` (run 41659863277) and
`aceea6ae64161bfd56a83a6cade11b37fc4d8b39` (run 41666050673), both based on 17bd529.
`aceea6ae` brought CI down to **one** failure. You can fetch it for reference with
`git fetch origin aceea6ae64161bfd56a83a6cade11b37fc4d8b39`, then
`git diff 17bd529 aceea6ae…`. The diff still applies cleanly to HEAD. Do not copy it
blindly, for two reasons:

- It hand-edits the grammar number in real-bob fixtures (see `fixture_source`).
- Its rewrite of `testTwoClipboardChangesSupersedeOlderPreviewsForTheSameDraft` reads
  fake-bob's counter file with `Int("1\n")`. That is always `nil` in Swift, so the wait
  never succeeds. This bug is the snapshot's one remaining failure.

## Root causes and fixes

### 1. The grammar gate outran the fixtures (13 tests)

Commit 72daffb raised `CaptureGrammarGate.minimumRevision` from 3 to 5. It also changed
fake-bob's _injected_ default grammar to 5. Two things were left behind:

- The real-bob fixtures `directive-parse-clip.json` and
  `directive-parse-clip-count.json` still report grammar 3.
  `directive-parse-batch-three-items.json` reports grammar 2.
- About 67 inline fake-bob bodies still hard-code `"grammar": 3`.

A parse below the gate is not "compatible". `updateClipboardPolling`,
`isCurrentLivePreviewRequest`, and submit therefore refuse to read the clipboard, arm
polling, or submit. Affected tests:

- **`CaptureClipPreviewHostedTests` (11 tests):**
  `testActivationRefreshesThePreviewAndArmsPolling`,
  `testClipboardCounterChangeRepreviewsAnUnchangedDraft`,
  `testClipboardPollingTaskDoesNotRetainTheModelAfterTeardown`,
  `testClipDraftPreviewOmitsNoClip`, `testCountedClipCarriesHistoryEntries`,
  `testFocusLossAndHideInvalidateInflightClipboardPreviews`,
  `testPollingObservationsStayAtOrBelowOneHertz`,
  `testPollingStopsOnHideFocusLossAndDraftRemoval`,
  `testRemovedDirectiveDuringInflightPreviewInvalidatesTheRead`,
  `testRemovedDirectiveRestoresNoClip`, and
  `testTwoClipboardChangesSupersedeOlderPreviewsForTheSameDraft`. The last one also has
  the race in §2.
- **`CapturePanelModelTests` (2 tests):** `testCommandAcceptInsertsAndSubmits` and
  `testBlockIDLinkCommandAcceptSubmits`. Their parses come from the inline
  `^sase:deep-fix` and `@file:ready` bodies.

The installed bob (`bob 0.4.0`, bob-cli master) reports grammar 7. Fixes:

> [!decision] fixture_source = regenerate

1. **Regenerate the directive fixtures** (default `regenerate`). Build bob from a clean
   bob-cli checkout at `master` (`cargo build`, giving `target/debug/bob`; `target/` is
   ignored, so the tree stays clean). Then, from bob-mac-capture, run:

   ```bash
   BOB_BIN=<bob-cli>/target/debug/bob BOB_CLI_DIR=<bob-cli> Scripts/generate-directive-fixtures.sh
   ```

   A trial run against bob-cli 0b4cebdf changed 66 files:
   - `grammar` goes from 3 (or 2) to 7.
   - `"drawer":null` is added to capture results.
   - `capture-template-example.json` loses `plan_budget.themes`, and its first
     `pomodoro_blocks` line becomes `changed`/`before`.
   - `open_notes[].ref` hashes change.
   - The README records the new revision.

   If you pick `hand_bump`, edit only the three parse fixtures named above to grammar 5,
   and add a README note saying they were bumped by hand.

2. **Make the generator enforce the gate,** so the next gate bump cannot leave fixtures
   behind. In `Scripts/generate-directive-fixtures.sh`, replace the hard-coded `3` floor
   in the probe check (~line 151) and in the per-fixture assertion (~line 477) with the
   Mac's gate. Read it from `Sources/CaptureCore/CaptureGrammarGate.swift`
   (`minimumRevision = N`), and fail loudly if it cannot be parsed. Also:
   - Write the grammar the probe actually reported into the README template, instead of
     the literal "(grammar revision 3, …)" (~line 635).
   - Update the header comment that says "grammar-3 … fixtures".

   Keep the `e660c8d8` ancestry check. Skip this step under `hand_bump`.

   > [!decision] fake_bob_scope = all

3. **fake-bob inline bodies** (default `all`, decision `fake_bob_scope`): replace every
   literal `"grammar": 3` and `"grammar":3` in `Tests/Fixtures/fake-bob` with 5. This
   includes the `printf` fallbacks at ~278, ~406, ~559, ~567, ~581, ~865, ~878, and
   ~890. Do not touch `inject_parse_grammar`; `FAKE_BOB_PARSE_GRAMMAR` gate tests depend
   on it. Afterwards, `grep -cE '"grammar": ?3' Tests/Fixtures/fake-bob` must print 0.
   Under `failing_only`, change only the two bodies (~1430 and ~1534).
4. **Guard against recurrence** in
   `Tests/CaptureCoreTests/CaptureDirectiveWireTests.swift`.
   `testRealBobFixturesReportGrammarThree` asserts `== 3` and will fail after
   regeneration. Replace it with `testRealBobDirectiveFixturesMeetGrammarGate`:
   - Enumerate every `directive-parse-*.json` and `directive-complete-*.json` in the
     fixtures directory (same `#filePath`-based directory as `fixtureURL`).
   - For each, decode it and assert `schemaVersion == 1` and `grammar` ≥
     `CaptureGrammarGate.minimumRevision`, using the file name as the message.
   - Assert that at least one file of each family was found.

   Also fix the doc comment above the class, which says "grammar-3 wire". Under
   `hand_bump`, cover only the parse fixtures that were bumped.

5. **`docs/protocol.md` ~line 82** says "the gate stays at grammar 3". Change it to say
   the gate requires grammar 5.

### 2. The supersede test races fake-bob's sequence counter (1 test)

`testTwoClipboardChangesSupersedeOlderPreviewsForTheSameDraft` serves three dry-run
fixtures in order. fake-bob's `capture --dry-run` sequence block (~line 4006) picks the
fixture from a `${FAKE_BOB_RECORD_PATH}.dry-run-count` file, guarded by a symlink lock.
It writes the argv record line at the very top of the script, before it parses the large
`case` statement.

The test waits only for that record line. It then fires the next poll, which cancels and
`terminate()`s the in-flight process. If that process dies before it increments the
counter, the next preview is served the wrong fixture and `"\t- newest"` never arrives.
If it dies while holding the lock, every later dry run spins forever.

Rewrite the test body to wait on settled sequence steps instead of record lines:

- Define `sequenceURL = recordURL.path + ".dry-run-count"` and
  `lockPath = sequenceURL + ".lock"`.
- `settledDryRuns()`: if
  `(try? FileManager.default.destinationOfSymbolicLink(atPath: lockPath)) != nil`,
  return 0. The lock is a **dangling symlink**, so `fileExists(atPath:)` would report it
  absent. Otherwise return
  `Int(text.trimmingCharacters(in: .whitespacesAndNewlines)) ?? 0`.
- Stage 1: `prepareForPresentation()`, then
  `guard await waitUntil { model.clipboardPollingForTests && settledDryRuns() >= 1 }`.
  Then assert the record contains `argv=capture --dry-run --format json -- jot idea %c3`
  (no `--no-clip`).
- Stage 2: set `counter = 101`, `fireClipboardPollForTests()`, then
  `guard await waitUntil { settledDryRuns() >= 2 }`.
- Stage 3: set `counter = 102`, fire, then guard-wait for `previewCallCount >= 3` and
  first entry lines `["\t- newest"]`.
- Keep the 2.2 s settle and the final three assertions.

Use `guard … else { return }` on each wait, so one timeout does not cascade.

### 3. Self-contradictory retype fixture (1 test)

`CapturePanelModelTests.testTaskTypeRetypeMoveKeepsDraftAndUsesEnsureNextAction` gets
"Ensured Next and created" and "Moved LATER → FIX (created FIX)". It expects "…moved"
and "Moved LATER → FIX".

The fake-bob `capture` body for `@file+goog-exit#fix` (~line 4650–4685, added in
9d023bb) says `"pomodoro_link_action": "moved"` to the existing FIX entry on line 3, but
also `"creates_pomodoro": true`. Those two fields contradict each other. Set
`creates_pomodoro` to `false`; no other test uses this body.

### 4. Pull-In ordering and caption (3 tests)

- **`testKeepMovingChoiceHasDistinctContextAndRanksLast`**
  (`CapturePullInChoiceModelTests`). Since 17bd529, `CapturePullInChoiceState` sections
  emit Bob's listed groups first whenever `pullIn.groups` is non-empty. A copy-less
  payload that lists only `keep_moving` (for its folded counts) therefore puts
  keep-moving ahead of the hand-off and project sections.

  > [!decision] pull_in_order = product

  Under `product` (decision `pull_in_order`), in
  `Sources/BobMacCapture/CapturePullInChoice.swift`:
  - Add `private var carriesBobCopy: Bool`
    (`authoredCopy != nil || pullIn.groups.contains { $0.copy != nil }`) and reuse it in
    `contextText`.
  - In the sections builder, use the Bob-order-plus-appended-extras path only when
    `carriesBobCopy`.
  - Otherwise, use Bob order only when every row identity is listed (the pre-17bd529
    rule). Fall back to `legacySectionOrder` in all other cases.
  - Update the comment.

  Current bob (`src/native/capture/pull_in_copy.rs` in bob-cli) lists every offered
  group with copy, so real payloads are unaffected. Under `test`, change the assertions
  instead.

- **`testRealKeepMovingFixtureOpensChoiceAndSubmitsItsOpaqueKey`** (same class). The
  test and the fixture were written in the same commit (90bd9ba), so this test has never
  passed. The real fixture `task-complete-keep-moving.json` has three keep-moving offers
  (`^a` n=3, `^b` n=2, `^p` n=1), not one. Rewrite the test to:
  - Derive the keys sorted by offer `n`.
  - Assert they are `["sase_badges.md#^p", "sase_badges.md#^b", "sase_badges.md#^a"]`.
  - Assert `selectableKeys` equals them and `effectiveSelectionKeys == [first]` (the
    highlighted first row).
  - Assert the confirm call contains `--pull-in <first>`.
- **`testMixedBobGroupOrderKeepsHandOffBeforeRelation`** (`CapturePullInChoiceTests`).
  The real mixed fixture carries Bob group copy. 17bd529 documented that supplied copy
  suppresses the legacy "Unblocked by …" caption. Change the last assertion to
  `XCTAssertNil(state.contextText)` and add a one-line comment saying why.

### 5. Stale AppKit and test expectations (6 tests)

- **`CaptureEditorLeaderTests.testToggleLineCaseUndoRestoresOriginalLine`**. AppKit
  native undo leaves the caret at {5,0}; the test expects {2,0}. Under `native`
  (decision `line_case_undo`), delete the caret assertion and add a comment pointing to
  docs/keyboard.md: undo behaves as for any native edit. Under `restore`, change
  `toggleLineCaseInEditableTextView` (`CapturePanelController.swift` ~940) to register
  undo that restores the prior selection, and keep the assertion.
- **`CaptureSeparatorToggleTests.testPlusAfterCaretIDSwitchesToSubBulletInOneUndoStep`**
  and **`testPlusToggleUndoRestoresLiteralPostKeyDraftOnce`**. A window-less
  `NSTextView` has no `undoManager`, so `canUndo` is false.
  - Add a `makeHostedTextView(_:) -> (NSTextView, NSWindow)` helper: a borderless or
    titled `NSWindow`, `isReleasedWhenClosed = false`, text view added to `contentView`,
    `makeFirstResponder`, caret at the end, and `XCTAssertNotNil(textView.undoManager)`.
  - Use it in both tests, with `defer { window.close() }`.
- **`CaptureSeparatorToggleTests.testTypedPlusAsWholeItemOpensActiveTaskPickerThroughKeyIntent`**.
  Since e9b5f81 (Oct 3), a lone `+` opens the **parent-task** picker (fixture
  `parent-task-parse-plus.json`). Rename the test to
  `…OpensParentTaskPickerThroughKeyIntent`, and assert `model.pickerSourceIsParentTask`
  and
  `model.picker?.source.operatorContinuationHint == "Type a number or + to adjust; Esc to extend +5m"`.
  This matches `ParentTaskPickerPresentationTests`.
- **`CaptureTaskTypeHostedTests.testBareLeadingTrailingAndChildMarkersOpenTaskTypeMenu`**.
  Real bob gives no `task_type` context with the cursor at the end of `#fix Fix it`
  (`task-type-complete-leading-full.json`, cursor 11). It does at cursor 4
  (`task-type-complete-leading-prefix.json`).
  - Make the loop take explicit cursors: `("#f","f",2)`, `("#fix Fix it","fix",4)`,
    `("Fix the parser #f","f",17)`.
  - In fake-bob's task-type completion `case` (~line 2027), serve
    `task-type-complete-leading-prefix.json` for `#fix Fix it` when `${cursor}` is `4`,
    and `…-leading-full.json` otherwise.
- **`CaptureTaskTypeHostedTests.testTypedRowDoesNotWriteVisibility`**. Other fake-bob
  calls create the record file, so "file does not exist" is the wrong check. Read the
  record (empty if absent) and assert it has no `argv=themes hide` and no
  `argv=themes unhide` lines.

## Verification

This host has Linux Swift but no AppKit or XCTest for the `BobMacCapture` targets. Real
CI is the only thing that runs them.

1. **fake-bob sanity (Linux).**
   - `bash -n Tests/Fixtures/fake-bob`.
   - The grep from §1.3 prints 0.
   - Spot-check the edited branches:
     - `FAKE_BOB_TASK_TYPE_CHOOSER=1 Tests/Fixtures/fake-bob capture-complete --cursor 4 --format json -- '#fix Fix it'`
       shows `"context":"task_type"`.
     - The dry-run `capture` for `@file+goog-exit#fix` shows
       `"creates_pomodoro": false`.
     - `capture-parse -- '^sase:deep-fix'` shows grammar 5.
2. **`just portable-check`.** Runs swift-format lint, then builds and tests CaptureCore
   and RefsCore, including the new `CaptureDirectiveWireTests` guard. It must pass.
3. **Mac typecheck (best effort, per AGENTS.md).** Use `/sase_memory_read` on
   `tailnet.md` for host details. rsync a scratch copy to the Mac without `.build` and
   `.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy.
   Never use Bryan's checkout or `just install`. If the Mac is offline, say so in the
   final report.
4. **`just docs-check`** (for the `docs/protocol.md` edit).
5. **Real CI.** Run `Scripts/macos-ci-check` from the bob-mac-capture checkout. It
   pushes a throwaway `ci/precommit/*` snapshot, waits for the macOS workflow, and
   deletes the branch. Allow about 15–25 minutes; run it synchronously with a large
   timeout or through `/sase_monitor`. It must exit 0, with all 3799+ tests passing. If
   new failures appear, fix each at its root and rerun:
   - A regenerated fixture's new content breaks a test: update the test to the truthful
     output. Never hand-edit a regenerated fixture.
   - A test passed only because a stale grammar-3 body blocked submit: fix that test's
     expectation.
6. **Land it.** Commit in bob-mac-capture with a Conventional Commit, for example
   `fix(capture): restore green macOS CI for the grammar-5 gate`. After it reaches
   `master`, run `actstat` and check that `bobs-org/bob-mac-capture` goes green, if the
   turn allows. Otherwise report the passing `macos-ci-check` run as the evidence.

## Out of scope

- Do not regenerate the drawer, task-type, or other fixture families that still report
  grammar 3. No failing test depends on them passing the gate.
- Do not change `CaptureGrammarGate.minimumRevision`.
- Do not touch the bob-cli, bob-plugins, or vault repos.
