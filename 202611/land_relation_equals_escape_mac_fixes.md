---
tier: tale
size: medium
title:
  Finish bob-cli-hz — repair the Mac fast == escape and its tests, then close the epic
goal:
  In Bob Mac Capture, a fast `==` after Bob's `=` opener reply types one literal `=` and
  dismisses Links To. The equals window rejects out-of-order timestamps. Every
  bob-cli-hz macOS test suite is green in CI, apart from the bob-cli-h7 baseline. Epic
  bob-cli-hz is closed and its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-hz.land
proposed_with: claude/opus
bead: bob-cli-hz
status: done
---

- **PARENT:**
  [202611/relation_equals_alias_and_direction_flips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/relation_equals_alias_and_direction_flips.md)
- **BEAD:**
  [bob-cli-hz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hz/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-hz.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-hz.land.md)
- **COMMITS:**
  - [1e0a092](https://github.com/bobs-org/bob-mac-capture/commit/1e0a092441d0a2bc0a50fc5c42ec09fd7463ce89)
    — fix(mac-capture): repair fast == escape and record race (bob-cli-hz)

# Finish epic bob-cli-hz: repair the Mac fast `==` escape, then close the epic

## Context

Epic `bob-cli-hz` ("Relation = alias, direction flips, and the == escape"; plan
`plan:202611/relation_equals_alias_and_direction_flips.md`) has five closed phases. Its
lander verified the bob-cli side. `bob capture-rewrite --cursor` matches every Bob row
of the behavior contract (`=` opener, literal `-`, `~=`/`~~=` flips,
`toggle_relation_direction` both ways, labelled `~` flips, labelled `=` stays literal,
`x=2-`). `cargo test --lib capture_language::tests::rewrite` (66 tests) and
`cargo test --test cli capture::rewrite` (34 tests) pass. The decision record
`decisions:relation-equals-alias-and-flips`, its supersede-in-part of
`decisions:relation-card-tab-accepts-ctrl-d-hides`, and `docs/task-relations.md` are
landed. Later docs commits (`e8b8ce73`, `2e3ccc90`) already carry the new grammar into
`docs/reference/capture-syntax.md`. The lander already triaged every
`PROPOSED FOLLOW-UP:` (see the epic's notes), so do not file anything for them.

**Remaining epic work is in the linked repo `bob-mac-capture`.** Phase `equals-escape`
(`bob-cli-hz.4`, commit `4fbbe84`) never got macOS test verification, because the Mac
has no XCTest. The exact-SHA master macOS CI run **42323414005** (`4fbbe84`) fails four
epic tests on top of the twelve-test baseline that `bob-cli-h7` tracks:

1. `CaptureRelationEqualsAliasTests.testEqualsGestureGuardsRepeatMarkedTextAndZeroTimestamps`
   (line 630). `CaptureEqualsGesture.isFastSecond`
   (`Sources/CaptureCore/CaptureEqualsGesture.swift`) accepts a second key **earlier**
   than the first: `0.1 - 1.0 <= window` is true. Root cause known: it is missing a
   `second >= first` guard.
2. `CaptureRelationEqualsAliasTests.testFastSecondEqualsUndoesOpenerAfterReply` (lines
   463–470) and
3. `CaptureRelationEqualsAliasTests.testTripleEqualsGivesDoubleEquals` (lines 566–580).
   These two are a **real user-facing bug**: in the editor, `Fix it ` + `=` opens Links
   To (`Fix it ~`). If the second fast `=` arrives _after_ Bob's reply, while the picker
   is open, `controller.routeKeyDown` returns the event unconsumed. The draft stays
   `Fix it ~`, the picker stays open, and `model.equalsGestureArm` is **still set**
   (surface `.editor`, `isOpener: true`, `preDraft "Fix it "`, `postDraft "Fix it ~"`,
   `firstTimestamp 1.0`). The CI dump shows `picker.draftSnapshot == "Fix it ~"` and an
   empty filter. The contract row "Editor, `D ` + fast `==` → `D =`, picker dismissed"
   is therefore broken in its most common timing. The before-reply path
   (`testFastDoubleEqualsBeforeReplyCollapsesToOne`) and the picker-surface undo both
   pass.
   - Lander's reading of the code: the slow test (`testSlowSecondEqualsFlipsAfterReply`,
     same state, 351 ms) is consumed as a picker flip. So in the fast case the router
     (`CaptureKeyCommandRouter.pickerCommand`) must have returned
     `.undoEqualsGestureAndTypeLiteral`.
     `CapturePanelModel.undoEqualsGestureAndTypeLiteral()` then returned `false` from a
     guard **before** `equalsGestureArm = nil`, and the controller fell through to
     `return event`.
   - Prime suspects are the guards that `equalsGestureSecondEqualsSurface` does not
     share: `!isSubmitting`, `!editorInputLocked`, the `insertRange` lookup, and
     `openPicker.draftSnapshot == arm.postDraft` in `undoEditorEqualsGesture`.
   - This is unconfirmed; confirm the real cause.
4. `CaptureRelationFlipAliasTests.testLabeledRelationTildeFlipsAndEqualsStaysLiteral`
   (line 343). This is an async race. The literal `=` path returns `changed:false`, so
   `waitUntil { literalView.string == draft + "=" }` succeeds before fake-bob writes the
   record file, and `String(contentsOf: literalRecord)` throws "no such file".
   `bob-cli-hz.3--4` fixed it (verified green in CI snapshot run 42308430792), but the
   fix **never reached master**. Re-apply it.

The other twelve failures on `4fbbe84` are the pre-existing `bob-cli-h7` baseline. They
also fail on clean bases `11dcba7` (run 42268256300) and `cbb50dc` (run 42278025697):

- `CaptureEditorWordMotionTests.testMotionsLeaveTextAndUndoUntouched`
- `CapturePanelModelTests.testCommaCommitsPartialRouteThenBobOpensParentTaskPicker`
- `CapturePanelModelTests.testCommaOnExactRouteUsesBobRewriteLane`
- these `CaptureSeparatorToggleTests`:
  - `testBackslashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses`
  - `testCommaAfterCompleteTargetRunOpensNextParentPicker`
  - `testDelayedAcceptedAliasSupersedesGatedCompletion`
  - `testNativeUndoRestoresLiteralSigilAliasWithoutRetriggering`
  - `testPeriodAliasAfterRouteOpensNewIDBlockPicker`
  - `testPeriodAliasOnMoveBulletOpensMovePicker`
  - `testSemicolonAliasOnOfferMoveBulletOpensMoveAndLinkPicker`
  - `testSlashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses`
  - `testUndoAfterMoveAliasRestoresLiteralWithoutRetriggering`

They are **out of scope**. Do not fix, skip, or weaken them.

Governing records: `decisions:mac-capture-is-a-thin-client` (the Mac never lexes; Bob
decides syntax) and `decisions:relation-equals-alias-and-flips` (the fast `==` contract,
a 350 ms key-to-key window measured from `NSEvent.timestamp`). Read both with
`sase memory read decisions:<keyword> -r "<why>"` before changing behavior.

## Steps

1. **Open the repo.** Run `sase repo open bob-mac-capture -r "<why>"` and read its
   `AGENTS.md`. Every path below is relative to that checkout. Work on `master`.

2. **Fix the negative-delta window (failure 1).**
   - In `Sources/CaptureCore/CaptureEqualsGesture.swift`, make `isFastSecond` require
     `second >= first` as well as the existing guards.
   - Add a portable case to `Tests/CaptureCoreTests/CaptureEqualsGestureTests.swift`:
     `second` earlier than `first` never qualifies, and `second == first` is allowed.
   - Check other `isFastSecond` callers, such as `equalsProbeReplyPending` and the
     label-card path, so that none relied on the old behavior.

3. **Re-apply the dropped race fix (failure 4).** In
   `Tests/BobMacCaptureTests/CaptureRelationFlipAliasTests.swift`,
   `testLabeledRelationTildeFlipsAndEqualsStaysLiteral`, replace
   `await waitUntil { literalView.string == draft + "=" }` with a wait for both
   conditions:

   ```swift
   await waitUntil {
       literalView.string == draft + "="
           && (try? String(contentsOf: literalRecord))?.contains(
               "argv=capture-rewrite --cursor 20 --format json -- Fix it ~foo:bar|see="
           ) == true
   }
   ```

   Keep the existing assertions after it. Apply the same wait to the first half of that
   test (the `~` flip record at line 326) if it has the same shape of race.

4. **Root-cause and fix the editor opener undo after Bob's reply (failures 2–3).**
   - Find which condition makes `undoEqualsGestureAndTypeLiteral()` (or the routing in
     `CapturePanelController.routeKeyDown` around the `equalsGestureFastSecond` /
     `.undoEqualsGestureAndTypeLiteral` handling) refuse while the alias-opened Links To
     picker is open.
   - The Mac cannot run XCTest, so diagnose through `Scripts/macos-ci-check` snapshot
     runs. Temporary richer assertion messages in the two failing tests are fine (for
     example, dump `isSubmitting`, `editorInputLocked`, `plainDraft`,
     `picker?.draftSnapshot`, `picker?.replacementRange`, and which guard refused).
     Remove all temporary diagnostics before committing.
   - Fix the **product code** so the contract holds:
     - `Fix it ` + `=` (reply lands, Links To open, empty filter) + fast `=` gives
       `Fix it =`. The picker is dismissed, the record is cleared, the key is consumed,
       and one Undo gives back `Fix it ~`.
     - `===` gives `==`.
     - Slow `= =` still flips to `Fix it ~~` (Linked From).
     - Space/Escape restores on the alias-opened empty Links To still work at any speed.
   - Make routing and undo agree. Share one precondition check between
     `equalsGestureSecondEqualsSurface` and `undoEqualsGestureAndTypeLiteral`, so the
     router never claims `.undoEqualsGestureAndTypeLiteral` when the undo will refuse. A
     refused claim must also never leave the key half-handled.
   - Do not change the tests' expected behavior; they encode the accepted contract.
   - Also confirm that the same after-reply-with-picker-open timing works for the `~=`
     flip. `testTildeFlipThenFastEqualsRestoresTildeEquals` only covers the case before
     the Linked From picker opens. Add a sibling test that waits for
     `model.picker?.source == .linkedFrom` and then sends the fast `=` (expected text
     `New task ~=`, record cleared). Add the same kind of sibling for a post-token
     `toggle_relation_direction` flip armed by `=`, if the existing tests do not already
     cover it.
   - Use fake-bob replies and injected `NSEvent` timestamps; never wall-clock sleeps for
     synchronization. Async tests must never await an untimed continuation.
   - Format only the lines you change (`.swift-format`).

5. **Verify** (all required):
   - Run `just portable-check` (on athena; it must be green, including
     `CaptureEqualsGestureTests`).
   - Mac scratch compile, best effort: rsync without `.build`/`.git`, then `bundle.sh`
     and `just typecheck-tests`, then delete the copy (`tailnet.md` reference memory for
     access). If the Mac is offline, say so in the close note.
   - Run `Scripts/macos-ci-check` on the final tree. It must show Lint and Build green.
     Its Test step may fail **only** on the twelve `bob-cli-h7` baseline tests listed
     above. Every test in `CaptureRelationEqualsAliasTests`,
     `CaptureRelationFlipAliasTests`, `CaptureRelationPanelTests`, and
     `CaptureEqualsGestureTests` must pass.
   - If an epic test fails once and passes on an unchanged rerun, treat it as a flake:
     compare with `bob-cli-hv`, rerun once, and file it with `/sase_new_task` only if it
     is new. If any non-baseline test outside those suites newly fails, compare it with
     clean base `4fbbe84` (run 42323414005). Fix it if this work caused it.
   - If you touched docs (not expected), also run `just docs-check`.

6. **Commit.** Commit the bob-mac-capture changes through `/sase_final`'s repository
   obligation, as `fix(mac-capture): …` naming `bob-cli-hz`. After the push, hand the
   exact-SHA macOS CI watch to `/sase_monitor`, as AGENTS.md requires. Do not wait for
   it before the closeout below; `Scripts/macos-ci-check` is the landing gate.

7. **Close out epic `bob-cli-hz`** (final step, same turn as the code):
   - Run `sase bead epic-symbols bob-cli-hz`. At landing time it printed "No
     --epic-symbol entries". If any now appear, resolve each one: wire it up, privatize
     it, add a non-test pragma, or delete it per the Symvision epic-whitelist policy. Do
     not re-key it unless a still-open later bead needs it.
   - Close the epic: `sase bead close bob-cli-hz --note "<verification>"`. The note
     should state:
     - the bob-cli contract rows checked live and the rewrite test counts above;
     - the four Mac fixes and their root causes;
     - the `portable-check` result;
     - the `macos-ci-check` snapshot SHA/run, with every hz suite green and only the
       `bob-cli-h7` baseline failing;
     - the Mac scratch typecheck status;
     - the follow-up outcomes already recorded on the epic: `bob-cli-h7` +1,
       `bob-cli-i9` filed, and the CLT-only-mac proposal declined.

     Never use `--force` merely to make the close succeed.

   - Run `just symvision` if that recipe exists (bob-cli has none at landing time; say
     so if it is still absent).
   - In the bob-cli checkout, set `status: done` in the frontmatter of
     `sase/repos/plans/202611/relation_equals_alias_and_direction_flips.md` (the PLAN
     path that `sase bead read bob-cli-hz` shows).
   - `bob-cli-hz` has no `parent_bead`, so nothing further needs closing.
