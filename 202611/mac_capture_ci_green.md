---
tier: tale
title: Restore green macOS CI for bob-mac-capture
goal:
  bob-mac-capture's macOS CI workflow passes every step on master again, proven by
  Scripts/macos-ci-check before landing.
size: medium
decisions:
  mixed_context_caption:
    ask:
      Should copy-bearing mixed Pull-In cards keep 17bd529's no-synthesized-caption
      rule?
    choices:
      keep_rule:
        Update the stale test to expect no context line; Bob's group copy stays the only
        copy
      restore_caption:
        Drop the copy gate for mixed cards so they show "Unblocked by …" again (product
        change)
    default: keep_rule
    why:
      The approved mac_relation_landing_repairs plan required it; Bob's copy feeds every
      client
    answer: keep_rule
  grammar_sweep:
    ask:
      Should all hand-written fake-bob capture-parse bodies move from grammar 3 to 5,
      not just failing ones?
    choices:
      targeted:
        Bump only what the failing tests use; file one task bead for the remaining
        grammar drift
      sweep:
        Bump every inline grammar-3 parse body as 9a1876d did; may surface more CI
        failures
    default: targeted
    why:
      Keeps this repair to proven failures so each 15-minute CI round trip has a known
      outcome
    answer: targeted
  line_case_undo_caret:
    ask: Should Command-Z after Ctrl-G U also put the caret back where it was?
    choices:
      native:
        No product change; the test stops asserting AppKit's post-undo caret position
      restore:
        Register the original selection in the undo group and document it in
        docs/keyboard.md
    default: native
    why:
      docs/keyboard.md promises only that undo restores the line, like any native edit
    answer: native
proposed_by: bbugyi200.athena.17a
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 09:40:31
status: wip
---

# Restore green macOS CI for bob-mac-capture

## Goal

`bobs-org/bob-mac-capture` CI (`.github/workflows/ci.yml`, job **macOS 26 SwiftPM**) has
been red on `master` since 2026-10-30. The last green master commit is `6376d21`. Make
the whole workflow pass again — Lint, Build, Test, Bundle, Verify plist and signature,
Launch smoke test, Install and reinstall — without weakening coverage. Prove it with
`Scripts/macos-ci-check` before finishing.

All edits belong in the linked repo `bob-mac-capture`. Open it with your `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<reason>"`), use only the path it prints, and
read its `AGENTS.md` first. The main points: never land uncompiled Swift; format only
the lines you change (4-space indentation, 100 columns); async tests must never await an
untimed continuation.

## Diagnosis (already done; re-verify, don't re-derive)

`actstat` reports the last three master runs as failing. Run 41565546665 (`5e95b8f`) is
a **Build** failure: the compiler gave up type-checking `CapturePanelView.swift:2198`.
`17bd529` fixed it by moving the relation-slot color `switch` into
`relationSlotColor(for:)`, so no action is needed. Run 41570850719 (`17bd529`, current
master) builds but fails **Test**: `Executed 3779 tests, with 52 failures`, in 23 tests.
Every step after Test was skipped, so Bundle, Launch smoke, and Install haven't run
since Oct 30. Fetch the log with
`gh run view 41570850719 -R bobs-org/bob-mac-capture --log-failed`.

The 23 failures come from five root causes.

### RC1: the grammar gate rose to 5 but the fixtures still report grammar 3 (13 tests)

`72daffb` ("present Pomodoro split and merge previews") changed
`Sources/CaptureCore/CaptureGrammarGate.swift` `minimumRevision` from 3 to 5. It also
changed `inject_parse_grammar`'s default in `Tests/Fixtures/fake-bob` to 5. That helper
fills in `grammar` only when it is missing, so fixtures and inline heredoc bodies that
hard-code `"grammar": 3` still report 3. The app then treats those drafts as coming from
an older Bob. Real Bob reports `GRAMMAR_REVISION = 6` (bob-cli
`src/native/capture_language/mod.rs`), and `docs/protocol.md` documents the ≥ 5 gate, so
the product is right and the fixtures are stale. These failures were hidden at first:
the Build stayed broken from `c4d8cf0` through `0f2f46f`, and the tests first ran under
the new gate at `2884ea2`.

- **All 11 failing `CaptureClipPreviewHostedTests`**
  (testActivationRefreshesThePreviewAndArmsPolling,
  testClipboardCounterChangeRepreviewsAnUnchangedDraft,
  testClipboardPollingTaskDoesNotRetainTheModelAfterTeardown,
  testClipDraftPreviewOmitsNoClip, testCountedClipCarriesHistoryEntries,
  testFocusLossAndHideInvalidateInflightClipboardPreviews,
  testPollingObservationsStayAtOrBelowOneHertz,
  testPollingStopsOnHideFocusLossAndDraftRemoval,
  testRemovedDirectiveDuringInflightPreviewInvalidatesTheRead,
  testRemovedDirectiveRestoresNoClip,
  testTwoClipboardChangesSupersedeOlderPreviewsForTheSameDraft):
  - `Tests/Fixtures/directive-parse-clip.json` and `directive-parse-clip-count.json`
    contain `"grammar":3`; `directive-parse-batch-three-items.json` contains
    `"grammar":2`.
  - Every clip decision in `Sources/BobMacCapture/CapturePanelModel.swift` checks
    `CaptureGrammarGate.isCompatible`: clipboard-read authorization near :12675,
    `isCurrentLivePreviewRequest` near :13127, `updateClipboardPolling` near :13153, and
    `refreshPreviewForClipboardChange` near :13235.
  - So no `%c` draft ever reads the clipboard or arms polling. The tests that pass in
    that file are exactly the ones that expect `--no-clip` or never check it.
- **`CapturePanelModelTests.testCommandAcceptInsertsAndSubmits`** (draft
  `^sase:deep-fix`) and **`testBlockIDLinkCommandAcceptSubmits`** (draft `@file:ready`):
  - Their inline `capture-parse` heredoc bodies in `Tests/Fixtures/fake-bob` (near lines
    1425 and 1529) hard-code `"grammar": 3`.
  - `resumeExplicitSubmit` stops at its `CaptureGrammarGate.isCompatible` guard (near
    `CapturePanelModel.swift:2483`). The submit never happens: the draft isn't cleared
    and `lastSuccess` stays nil.
  - The sibling tests that pass end on drafts served by the default inline body, which
    goes through `inject_parse_grammar`.

### RC2: 17bd529 regressed Pull-In section order for older payloads without copy (1 test)

`CapturePullInChoiceModelTests.testKeepMovingChoiceHasDistinctContextAndRanksLast` sends
hand_off, project, and keep_moving offers, with a `groups` list that names only
`keep_moving` and carries no `copy`.

- `17bd529` replaced the "Bob order only when every row identity is listed" rule in
  `CapturePullInChoiceState.sections`
  (`Sources/BobMacCapture/CapturePullInChoice.swift`, near :779–815) with "listed groups
  in Bob order, then the rest in legacy order".
- The sections are now `[keep_moving, hand_off, project]`, so `sections.last` is the
  project section, and every failing assertion at lines 128–137 follows from that.
- `docs/protocol.md` (the `pull_in.copy` row) says: "Missing copy uses the legacy
  per-group Swift wording and ordering".
- The 17bd529 plan wanted Bob order with appended extras for payloads that carry Bob's
  copy. Current Bob always sends copy, and every real-Bob fixture lists all of its
  groups.

### RC3: windowless text views have no undo manager (2 tests)

In `CaptureSeparatorToggleTests`,
**testPlusAfterCaretIDSwitchesToSubBulletInOneUndoStep** (lines 399/401/405) and
**testPlusToggleUndoRestoresLiteralPostKeyDraftOnce** (lines 497/499/503) build their
editor with `makeTextView` (near line 29), which is a bare `NSTextView()` with no
window.

- `textView.undoManager` is nil, so `canUndo` is false and `undo()` does nothing. The
  "Do work @file+id" in the failure messages is the text after the toggle, not after
  undo.
- The `+` toggle itself works.
- `testNativeUndoAndRedoDoNotRetriggerAssist` in the same file already hosts its text
  view in an `NSWindow`, with a comment explaining why, and passes on the same
  `applySeparatorToggleInEditableTextView` path. `f4ed5bb` fixed `CaptureListPanelTests`
  the same way.

### RC4: 17bd529 suppressed the mixed-card caption, and a test still expects it (1 test)

`CapturePullInChoiceTests.testMixedBobGroupOrderKeepsHandOffBeforeRelation` (line 455)
expects `contextText == "Unblocked by Ship schema change"` for the real-Bob fixture
`relation-successor-mixed-x-star-dependency.json`, whose groups carry Bob copy.

- `17bd529` added a `hasSuppliedCopy` early return to `contextText` (near
  `CapturePullInChoice.swift:633–649`). With Bob copy present, it returns nil unless the
  card is keep-moving only.
- That was deliberate: plan `202611/mac_relation_landing_repairs.md` says "Respect
  supplied copy in the context line too; an absent `why` in a supplied copy object means
  no invented legacy caption. Keep old strings only for absent-copy payloads".
- The test assertion dates from `52a222a` and was never updated. See decision
  `mixed_context_caption`.

### RC5: stale test or fake-bob expectations (6 tests)

- **`CapturePullInChoiceModelTests.testRealKeepMovingFixtureOpensChoiceAndSubmitsItsOpaqueKey`**
  (lines 79/80/88):
  - `task-complete-keep-moving.json` has three keep-moving offers in note order: `^a`
    (n=3), `^b` (n=2), `^p` (n=1).
  - The product sorts `selectableKeys` by `n` and highlights the lowest `n`, so the
    result is `[^p, ^b, ^a]` and the submit is `--pull-in sase_badges.md#^p`. Bob
    numbers keep-moving offers by lane, Pending → Next → Ready.
  - The test wrongly takes `keepMoving.first` and expects a single key. It has been
    wrong since it was added in `90bd9ba`.
- **`CaptureTaskTypeHostedTests.testBareLeadingTrailingAndChildMarkersOpenTaskTypeMenu`**
  (lines 17/22/23, case `"#fix Fix it"`):
  - The loop puts the cursor at `draft.utf8.count` (11), which is outside the `#fix`
    token.
  - Real Bob's `task-type-complete-leading-full.json` correctly returns no context at
    cursor 11. The real-Bob fixture for a cursor inside the token,
    `task-type-complete-leading-prefix.json` (generated by
    `Scripts/generate-task-type-fixtures.sh` as `complete_at '#fix Fix it' 4`:
    `task_type`, query `fix`, cursor 4), is never served.
  - The fake-bob `case` maps `"#fix Fix it"` to leading-full whatever the cursor is
    (near line 2027).
- **`CaptureTaskTypeHostedTests.testTypedRowDoesNotWriteVisibility`** (line 327):
  - It asserts that the fake-bob record file doesn't exist.
  - But assigning `processClient` triggers `savedTemplateShelf.refresh`, which runs
    `capture-templates` and creates the file.
  - The real assertion is "no `themes` visibility write".
- **`CapturePanelModelTests.testTaskTypeRetypeMoveKeepsDraftAndUsesEnsureNextAction`**
  (lines 4720/4721):
  - The hand-written fake-bob payload for this move (near line 4677) says
    `"pomodoro_link_action": "moved"` but `"creates_pomodoro": true`, a leftover from
    `9d023bb`'s rewrite.
  - Swift correctly renders "created". The real-Bob move fixture
    `task-type-capture-move-to.json` has `creates_pomodoro: false`.
- **`CaptureSeparatorToggleTests.testTypedPlusAsWholeItemOpensActiveTaskPickerThroughKeyIntent`**
  (line 720):
  - A lone whole-item `+` parses as `pomodoro_adjust` (`parent-task-parse-plus.json`)
    and opens the **parent-task** picker.
  - `docs/task-pickers.md` says a lone `+` stays a +5m adjustment and the picker footer
    hint reads "Type a number or + to adjust; Esc to extend +5m".
  - The expected status "Pick an active task — press Tab to browse" was copied from the
    `^` sibling test. The picker opens (line 717 passes), so only the expectation is
    wrong.
- **`CaptureEditorLeaderTests.testToggleLineCaseUndoRestoresOriginalLine`** (line 49):
  - Undo restores "Hello" (line 48 passes), but AppKit leaves the caret at {5,0}, the
    end of the replaced range, not {2,0}.
  - Nothing in `toggleLineCaseInEditableTextView`
    (`Sources/BobMacCapture/CapturePanelController.swift`) registers the selection for
    undo. See decision `line_case_undo_caret`.

## Implementation

Make each change, then re-read the test it targets to confirm the change satisfies every
assertion in that test, not just the one that failed.

### 1. Grammar fixtures (RC1)

1. Set `"grammar"` to `5` (the gate minimum and fake-bob's injected default) in
   `Tests/Fixtures/directive-parse-clip.json`, `directive-parse-clip-count.json`, and
   `directive-parse-batch-three-items.json`. Change only that field and keep each file's
   existing formatting.
2. In `Tests/Fixtures/fake-bob`, change `"grammar": 3` to `"grammar": 5` in the
   `^sase:deep-fix` and `@file:ready` `capture-parse` heredoc bodies.
3. Update `Tests/CaptureCoreTests/CaptureDirectiveWireTests.swift`
   `testRealBobFixturesReportGrammarThree`:
   - assert that the `directive-parse-clip.json` parse grammar is
     `>= CaptureGrammarGate.minimumRevision`;
   - keep the completion-fixture assertion unchanged, because completion grammar is not
     gated;
   - rename the test to say what it now checks.

   Grep `Tests/CaptureCoreTests` and `Tests/BobMacCaptureTests` for every other use of
   the three edited JSON fixtures and adjust any assertion that pins their grammar.

4. Fix the stale doc sentence at `docs/protocol.md` ("grammar 6; the gate stays at
   grammar 3") so it states that the gate requires grammar 5.
5. Harden the flaky `testTwoClipboardChangesSupersedeOlderPreviewsForTheSameDraft`:
   - It failed intermittently before 72daffb (dfdd52e, b48d1b4, d91ff5b, 01c753b).
   - fake-bob claims its `.dry-run-count` slot only after it writes the argv record
     line. So the wait for `previewCallCount >= 2` can finish, and the third poll can
     cancel the second process, before that process claims its slot.
   - Fix: after that wait, also wait (with a bounded `waitUntil`) until
     `recordURL.path + ".dry-run-count"` reads at least `2`, and only then set
     `counter = 102`.

> [!decision] grammar_sweep = targeted Do only steps 1–5. Then use your `/sase_new_task`
> skill to file one task bead for the remaining drift:
>
> - about 50 other inline `"grammar": 3` capture-parse bodies in
>   `Tests/Fixtures/fake-bob`;
> - about 110 JSON fixtures in `Tests/Fixtures` that hard-code grammar ≤ 4;
> - generator floors (for example `Scripts/generate-directive-fixtures.sh` still
>   requires only `grammar >= 3`).
>
> Name the latent hazard in the bead: any future submit test that uses one of these
> silently hits the grammar gate.

> [!decision] grammar_sweep = sweep Do steps 1–5, and also change every hand-written
> `"grammar": 3` (and lower) inside fake-bob's `capture-parse` case to `5`. Leave bodies
> that tests deliberately keep below the gate, such as the gate-test paths that use
> `FAKE_BOB_PARSE_GRAMMAR`. Before editing, grep the tests for any that rely on a
> specific body being gated. Real-Bob JSON fixtures stay out of scope; file the task
> bead for those only.

### 2. Pull-In ordering for older payloads without copy (RC2)

In `Sources/BobMacCapture/CapturePullInChoice.swift`:

- Factor the predicate
  `authoredCopy != nil || pullIn.groups.contains { $0.copy != nil }` into one private
  computed property, for example `carriesBobCopy`. Have both `contextText` and
  `sections` use it.
- In `sections`:
  - When `carriesBobCopy` is true, keep 17bd529's path: listed groups in Bob order, then
    unmatched groups in `legacySectionOrder`.
  - When it is false, restore the pre-17bd529 rule: Bob order only when
    `!pullIn.groups.isEmpty` and every row identity is listed; otherwise
    `legacySectionOrder` for all sections.
- Keep 17bd529's `PullInSectionIdentity.init(row:)` change: relation `after`/`label`
  only on `relation` rows. Keep its `detailText`/`metaDetails` changes.
- Update the comment above the ordering code to state both rules.

These tests pass today and must stay green:
`testHandOffWithSecondReasonJoinsBobGroupAndKeepsMeta`,
`testRelationAfterSuffixAndSecondReasonReachMetaAndAccessibility`,
`testDistinctAfterKeysAndDistinctLabelsStaySeparate`,
`testGroupMetadataDoesNotCreateEmptyPerItemSections`, and
`testAbsentWhyDoesNotSynthesizeLegacyCaption`. Trace each through the new code.

### 3. Mixed-card context line (RC4)

> [!decision] mixed_context_caption = keep_rule No product change. In
> `CapturePullInChoiceTests.testMixedBobGroupOrderKeepsHandOffBeforeRelation`, replace
> the `"Unblocked by Ship schema change"` assertion with
> `XCTAssertNil(state.contextText)`. Add a short comment: Bob-authored group copy
> suppresses synthesized legacy captions.

> [!decision] mixed_context_caption = restore_caption
>
> - In `contextText`, remove the `hasSuppliedCopy` early return. Keep-moving-only cards
>   with Bob copy must still return `pullInNonempty(section.copy?.why)`, and
>   relation-only cards must still return nil (`isOnlyRelation`).
> - Mixed and hand-off cards then fall through to the legacy "Unblocked by …" / "New
>   project · …" lines.
> - Rewrite the doc comment to match.
> - Keep line 455's expectation.
> - Confirm that `testAbsentWhyDoesNotSynthesizeLegacyCaption` and
>   `testHandOffWithSecondReasonJoinsBobGroupAndKeepsMeta` (relation-only, so nil) still
>   hold.

### 4. Host the undo tests in a window (RC3)

In `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`:

1. Add a private helper that returns `(NSTextView, NSWindow)`, modelled on the hosting
   code already in `testNativeUndoAndRedoDoNotRetriggerAssist`:
   - an `NSWindow` with `isReleasedWhenClosed = false`;
   - `contentView?.addSubview(textView)`;
   - `makeFirstResponder(textView)`;
   - the selection reset to the end of the text;
   - `XCTAssertNotNil(textView.undoManager)`.
2. Use the helper in `testPlusAfterCaretIDSwitchesToSubBulletInOneUndoStep` and
   `testPlusToggleUndoRestoresLiteralPostKeyDraftOnce`, with `defer { window.close() }`.
   Optionally switch the existing hosted test to the helper too.
3. Leave the windowless `makeTextView` for tests that don't undo.

### 5. Stale expectations (RC5)

- **`testRealKeepMovingFixtureOpensChoiceAndSubmitsItsOpaqueKey`:**
  - Derive `keys` from `success.taskComplete?.keepMoving`: take `offer`, sort by `n`,
    map to `key`.
  - Assert `keys == ["sase_badges.md#^p", "sase_badges.md#^b", "sase_badges.md#^a"]` and
    `choice.selectableKeys == keys`.
  - Assert `effectiveSelectionKeys` and the `--pull-in` argv against `keys.first`. That
    is the lowest-`n` default highlight; confirm that against `CapturePanelModel`'s
    default-selection code before asserting.
- **`testBareLeadingTrailingAndChildMarkersOpenTaskTypeMenu`:**
  - Iterate `(draft, query, cursor)`: `("#f","f",2)`, `("#fix Fix it","fix",4)`,
    `("Fix the parser #f","f",17)`.
  - Call `editorTextDidChange(cursorUTF8Offset: cursor)` and wait for
    `completionResponse?.cursor == cursor`.
  - In `Tests/Fixtures/fake-bob`'s capture-complete task-type routing, serve
    `task-type-complete-leading-prefix.json` for `"#fix Fix it"` when the requested
    cursor is `4`, and keep `task-type-complete-leading-full.json` for every other
    cursor. Read how that branch obtains the cursor before editing.
- **`testTypedRowDoesNotWriteVisibility`:** replace the file-exists assertion. Read the
  record if it exists (empty string otherwise) and assert that it doesn't contain
  `"argv=themes "`. Use the record line format the other theme-visibility tests in that
  file already match.
- **`testTaskTypeRetypeMoveKeepsDraftAndUsesEnsureNextAction`:** in fake-bob's
  hand-written payload for this scenario (near line 4677), change
  `"creates_pomodoro": true` to `false`. Confirm no other test uses that payload.
- **`testTypedPlusAsWholeItemOpensActiveTaskPickerThroughKeyIntent`:**
  - Rename the test to say it opens the parent-task picker.
  - Replace the status assertion with `XCTAssertTrue(model.pickerSourceIsParentTask)`
    and
    `XCTAssertEqual(model.picker?.source.operatorContinuationHint, "Type a number or + to adjust; Esc to extend +5m")`.
    Adjust the accessor path to whatever the model actually exposes.
  - Don't assert `"Ready"`.

> [!decision] line_case_undo_caret = native In
> `CaptureEditorLeaderTests.testToggleLineCaseUndoRestoresOriginalLine`, delete the
> caret assertion at line 49 and add a one-line comment: post-undo caret placement is
> AppKit's native behavior. Keep the text, `canUndo`, and leader-state assertions.

> [!decision] line_case_undo_caret = restore In
> `CapturePanelController.toggleLineCaseInEditableTextView`, wrap the edit in one undo
> group:
>
> - capture `textView.selectedRange()` before the edit;
> - register an undo action that restores that selection, and whose handler re-registers
>   the inverse;
> - perform `insertText` and `setSelectedRange(edit.resultingSelection)`;
> - register the redo-side selection mark.
>
> Undo must leave the caret at the original position, and redo at `resultingSelection`.
> Extend the test to cover redo too. Add one sentence to the Ctrl-G U row in
> `docs/keyboard.md`.

## Verification

Run the checks in this order, inside the opened bob-mac-capture checkout.

1. **Offline:** run `bash -n Tests/Fixtures/fake-bob`, then `just check-scripts`, then
   `just portable-check`. `portable-check` lints formatting for every Swift file and
   builds and tests CaptureCore/RefsCore, including the updated
   `CaptureDirectiveWireTests`.
2. **Docs:** run `just docs-check`, because `docs/protocol.md` changed (and
   `docs/keyboard.md` too under `restore`).
3. **Mac compile (best effort):**
   - Read `tailnet.md` with your `/sase_memory_read` skill to get the `mac` SSH alias.
   - With a bounded connect timeout, rsync a scratch copy (no `.build`/`.git`) to a
     unique directory on the Mac.
   - Run `Scripts/bundle.sh` and `just typecheck-tests` there, then delete only that
     scratch copy.
   - Never use Bryan's checkout or `just install`.
   - If the Mac is unreachable, say so in the final report.
4. **Live macOS CI gate (required):**
   - Run `Scripts/macos-ci-check` through your `/sase_monitor` skill, with profile
     `verify`, `--timeout 95m`, and a `--next` continuation that inspects the result.
     Follow `docs/develop/ci.md` § Precommit macOS check.
   - Success is exit `0`, with every required step green: Lint, Build, Test, Bundle,
     Verify plist and signature, Launch smoke test, Install and reinstall.
   - If it fails, fetch that run's failed log. Fix any of the 23 tests above that is
     still red, and any new failure these changes caused. Then run the gate again.
   - The post-Test steps haven't run since Oct 30. If one of them fails, diagnose it and
     fix it in scope.
   - If a failure needs a product decision this plan doesn't cover, stop. Report the
     failing step, test, and assertion with evidence, and file a task bead with your
     `/sase_new_task` skill.
5. Before finishing, check whether `origin/master` of bob-mac-capture has moved since
   you started. If it has, integrate the new commits and rerun step 4 on the result.

The final report must list:

- each of the 23 tests and its fix;
- the `macos-ci-check` run URL and verdict;
- the Mac compile outcome;
- any bead you filed.

## Out of scope

- The Swift 6 `Sendable` warnings and the `var calendar` warning in the Build log. They
  are warnings, not failures.
- The `CapturePanelModelTests` file-existence assertion near line 7352. It has the same
  latent template-refresh race as `testTypedRowDoesNotWriteVisibility` but passes today.
  Mention it in the report, and fold it into the grammar-drift bead only if one is
  filed.
- bob-cli source. No wire contract changes are needed.
