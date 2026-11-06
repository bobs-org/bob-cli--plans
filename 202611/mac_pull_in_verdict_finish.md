---
tier: tale
title: Finish the Mac Pull-In card verdict UI (bob-cli-hn.9 remainder)
goal:
  Close bob-cli-hn.9. Repair the Pull-In card's refusal reopen and its stale-commit
  dismissal, fix the footer-contract test the phase broke, add the model, route, router,
  and design tests the phase requires, then pass the Mac compile and exact-tree macOS CI
  gates.
size: medium
bead: bob-cli-hn.9
proposed_by: bbugyi200.athena.bob-cli-hn.9
proposed_with: claude/opus
create_time: 2026-11-06 09:04:08
status: wip
---

- **BEAD:** bob-cli-hn.9

# Finish the Mac Pull-In card verdict UI

This plan finishes assigned phase **bob-cli-hn.9** (`mac-card`) of
`plan:202611/successor_card_verdicts.md`. The parent plan's `## Phase: mac-card` section
and its acceptance matrix are binding. The epic's reviewer decisions are final, and none
of them authorizes a memory edit in this phase. **Do not edit any memory note.**

## What already landed (do not redo)

The first tale for this phase, `plan:202611/mac_pull_in_verdict_ui.md`, landed one
bob-mac-capture commit:

- **`6f4e279`** `feat(mac-capture): stage Done and Cancel verdicts on Pull-In card`.
  - The card now uses `CapturePullInOutcomes` as its source of truth.
  - The router has exact `⌃X` / `⌃⇧X` bindings and a `pullInReasonFocused` context, and
    the controller dispatches them.
  - It added the two-step Esc, the inline reason field, the glyphs and capsules, the
    consequence strip, the gated footer hints, the context menu and VoiceOver actions,
    and verdict argv through `dispatchCapture`.
  - It updated `docs/capture-previews.md`, `docs/keyboard.md`, and `docs/protocol.md`.
- **`8ab7c4e`** (closed prerequisite bob-cli-hn.8) added the decoding, the fake-bob
  verdict hooks, and the real-Bob fixtures.

The earlier agent left the bead open for three reasons:

- The Mac was offline, so `bundle.sh` and `typecheck-tests` never ran.
- The exact-SHA macOS CI never went green.
- Its tests covered only the portable model and four card-state cases.

The phase's required `CapturePullInChoiceModelTests`,
`CaptureInboxRouteChoiceModelTests`, and `CapturePullInChoiceDesignTests` coverage was
never written. This remainder fits one coding agent, so it is a tale rather than a child
epic.

## Evidence

### macOS CI on master

- Run 42161345004 on `6f4e279` and run 42216904060 on current master `a3e533b` both fail
  the Test step. Lint and Build pass.
- Of master's 15 failing tests, 14 also fail on the pre-phase base `8ab7c4e` (run
  42157294123), and open beads already track them:
  - **bob-cli-h7:**
    - `CaptureSeparatorToggleTests` (9 cases)
    - `CapturePanelModelTests` comma partial/exact route (2 cases)
    - `CaptureEditorWordMotionTests.testMotionsLeaveTextAndUndoUntouched`
  - **bob-cli-hu:** `CaptureRelatedLaneTests` debounce.
  - **bob-cli-hv:** `CaptureRelationFlipAliasTests` equals-in-editor.
- **Phase regression:**
  `CapturePullInChoiceTests.testFooterContractPinsHintsAndAccessibilityText` was
  introduced by `6f4e279`. It fails at `CapturePullInChoiceTests.swift:1164`: the third
  assertion calls `hintItems(openAfterCapture: true)`, whose `closeSupported` parameter
  defaults to `false`, yet it expects the `⌃X` / `⌃⇧X` hints.

### Bug: a refused commit never reopens the card

- `CapturePanelModel.completeSubmit` copies `activeInboxRouteSubmitContext` into a local
  `routeContext`, then sets the property to `nil` _before_ the `switch response`.
- On `offer_close_stale`, `offer_close_refused`, `offer_verdict_conflict`, and the
  legacy `stale, unknown, or out-of-range --pull-in pick:` error, the failure branch
  calls `reopenPullInChoiceAfterOfferRefusal` / `reopenPullInChoiceAfterStalePick`.
- Both helpers read `activeInboxRouteSubmitContext?.pullInChoiceForBack`, which is now
  always `nil`. They return `false`, and the user gets a generic "Capture failed" with
  every staged choice lost.
- This contradicts binding requirement 1 ("reopen … with the outcomes reseeded, not
  fresh") and the acceptance matrix row "the Mac reopens the card with choices intact".

### Bug: a stale commit can strand the card

- `commitPullInChoice` handles a stale guard (draft, client, or generation changed) by
  calling `cancelPullInChoiceFromKeyboard()`.
- When verdicts are staged, that function _arms_ the two-step Esc warning and returns.
  It leaves a stale card on screen instead of dismissing it.

## 1. Open the repo and confirm the baseline

- Use `/sase_repo` to open **bob-mac-capture**, and read its `AGENTS.md`.
- Fast-forward to the latest `origin/master`, because other agents push there often.
- Confirm that `6f4e279` and `8ab7c4e` are ancestors of HEAD.
- Re-list the latest master macOS run's failing tests with
  `gh run view <id> --log-failed`. This is your clean-base set for the gate in step 7.

## 2. Fix the footer-contract regression

- In `testFooterContractPinsHintsAndAccessibilityText`, pass `closeSupported: true` to
  the `hintItems(openAfterCapture: true)` call.
  - Keep the production default `closeSupported: false`: old Bob must keep today's
    hints.
  - Keep the existing assertion that `closeSupported: false` drops both chords.
- Check that the `ViewThatFits` fallback rows in `CapturePullInChoiceFooter.body` still
  keep the `⌃X` / `⌃⇧X` items when space allows.

## 3. Repair the refusal reopen

- Pass the remembered card in explicitly. Give both reopen helpers a
  `remembered: CapturePullInChoiceState?` parameter, fed from
  `routeContext?.pullInChoiceForBack` in `completeSubmit`.
  - Remove their reads of `activeInboxRouteSubmitContext`.
  - Apply this to the async refresh path too: it captures `remembered` before the
    `Task`.
- Share one main-actor helper for "show the reseeded card". It should:
  - set `pullInChoice`, `editorInputLocked = true`, `completionResponse = nil`, and
    `pickerChip = nil`, and request `.pullInOfferList` focus, as `openPullInChoice`
    does;
  - set `verdictRefusalMessage` / `errorMessage` to Bob's error;
  - set `statusText`, and make exactly one `announceStatus`, including
    `· N staged choice(s) lost` when `refresh(offers:)` drops keys.

  Both the structured-offers path and the dry-run refresh path use it. This satisfies
  the binding "reopen through `openPullInChoice`" intent without discarding the
  outcomes.

- **Keep these behaviors:**
  - at most one dry-run refresh, never a write retry;
  - the narrow legacy stale-pick prefix match;
  - ignoring a refresh after the draft, client, or generation changes;
  - the empty-offers recovery state;
  - keeping the remembered card when the refresh fails.
- Bob's `why` for a still-present refused row must stay visible until the user changes
  that row. Keep the existing `refusalReason(for:)` / `verdictRefusalMessage` display;
  just verify it in the tests in step 5.

## 4. Repair the stale-commit dismissal

Factor the second-Esc discard tail of `cancelPullInChoiceFromKeyboard()` into a private
`dismissPullInChoice(status:)`. The tail is the part that sets `pullInChoice = nil`,
clears the error, runs `restoreMoveChoiceForBack()`, unlocks the editor, and focuses it.

- Call it from the `commitPullInChoice` stale guard, so a stale card never arms the
  warning.
- The keyboard path keeps its reason-edit restore and two-step arming in front of it.

## 5. Add the required tests

Put each test in the existing suite named below, reusing its harness (`makeModel`,
`fakeBobPath()`, `recordText`, `captureCallCount`, `waitUntil`,
`installPullInChoiceForPreviews`, `cardOffer`, `RenderFixtureWriter`). Every async wait
must be bounded (`waitUntil`'s timeout). Never await an untimed continuation.

**Fake-bob setup**

- Serve the dry-run preview from `Tests/Fixtures/capture-offer-close-preview.json`. It
  has two existing offers, `deps.md#^alpha` and `deps.md#^beta`, plus `pull_in.close`.
- Serve writes from `capture-offer-close-write.json`.
- Select refusals with `FAKE_BOB_OFFER_CLOSE_REFUSAL=stale|refused|conflict`. That hook
  only fires when the argv carries `--complete-offer` or `--cancel-offer`.
- Use `capture-offer-close-legacy.json` for the old-Bob case.

**`CapturePullInChoiceModelTests`** (new verdict section):

1. Staging writes nothing:
   - `⌃X`, `⌃⇧X`, the reason confirm, and restore each leave the capture call count at
     the preview's count.
2. Verdict-only commit:
   - Done on alpha, then Return, sends exactly one write.
   - That write's argv carries `--pull-in none` plus `--complete-offer` with alpha's
     opaque key.
   - There is no highlight fallback pick. The primary label is `Close 1`.
3. Mixed commit:
   - Space marks beta, then Done on alpha.
   - The single write carries the pull-in key and the verdict.
   - The label is `Pull In 1 · Close 1`.
4. Shift-Return (`chooseNoPullInOffers`) with verdicts:
   - It sends `--pull-in none` and keeps every verdict, including a staged Cancel's
     `--cancel-offer` reason.
5. Control-Return with verdicts:
   - It submits both picks and verdicts and keeps the open-notes intent.
6. Reason editing:
   - Begin Cancel on a row that was Pull In, type a reason, then restore. The exact
     prior Pull In comes back, and the Esc warning is not armed.
   - Begin again and confirm. The reason is trimmed and newlines are collapsed.
   - A second confirm (the blur) does not double-apply or advance.
   - Pressing `⌃⇧X` on a staged Cancel row clears it.
7. Two-step Esc:
   - With a verdict staged, the first `cancelPullInChoiceFromKeyboard` arms
     `Esc again to discard 1 close`, keeps the card, and writes nothing.
   - A routed or native key in between (`disarmPullInDiscardForNativeKey`) disarms it.
   - Two presses then discard and return focus to the editor without a write.
   - With pull-in marks only, a single Esc still backs out.
8. Refusal reopen, one test per code:
   - **`stale` and `conflict`:** both fixtures echo the same alpha/beta keys. The card
     reopens with alpha's verdict, beta's mark, the reason, and the highlight intact.
     Bob's error shows, and nothing is lost.
   - **`refused`:** this fixture's keys differ. The card reopens and announces
     `· 2 staged choices lost`.
   - **All three:** there is exactly one write call and no automatic retry.
9. Legacy stale-pick refresh:
   - A failure with code `nil` and the `stale, unknown, or out-of-range --pull-in pick:`
     prefix triggers exactly one dry-run and no second write.
   - The choices are reseeded from that dry-run.
   - A refresh that resolves after the draft has changed is ignored.
10. Stale guard:
    - Change `plainDraft` after staging a verdict, then commit.
    - The card is dismissed without arming the warning and without a write.
11. Old Bob:
    - With the legacy fixture, `⌃X` beeps or announces, stages nothing, and the commit
      argv has no `--complete-offer` or `--cancel-offer`.

**`CaptureInboxRouteChoiceModelTests`:**

- A verdict commit that Bob answers with an inbox-route choice keeps the verdicts,
  reasons, draft, seed, and open intent through route confirmation. The second write's
  argv still carries them.
- Route Back restores the Pull-In card with the verdicts and reason intact and no
  highlight fallback.
- Reuse the existing route fixtures and env, adding `pull_in.close` capability only
  where needed.

**`CapturePullInChoiceTests`** (router cases, if not already covered):

- Exact `⌃X` maps to `.stagePullInDone`, and `⌃⇧X` to `.beginPullInCancel`.
- Repeats map to `.consumeKey`.
- `⌘X` and other modifier sets return `nil` (native).
- With `pullInReasonFocused`, Return maps to `.confirmPullInReason` and Esc/`⌃[` to
  `.restorePullInReason`, while other keys stay native.

**`CapturePullInChoiceDesignTests`:**

- Add rendered fixtures for:
  - a Done row (with `unblocks {k}`);
  - a Cancel-with-reason row;
  - an active reason-editor row (`Why cancel? (optional)`);
  - mixed footer counts;
  - the amber Esc-armed strip and a refusal strip.
- Render at the supported widths (760 and `panelMinimumContentWidth`) in both `.aqua`
  and `.darkAqua`, following `testRenderOpenCard`.
- Give `cardOffer` an optional `close: TaskCompleteOfferClose?` argument, defaulting to
  `nil` so existing callers are unchanged.

## 6. Verify locally

- Run `just portable-check` from the Mac checkout. Swift 6 is on athena.
- **Mac scratch compile (required; the Mac answered SSH while this was planned):**
  1. Read `sase/memory/tailnet.md` through `/sase_memory_read`.
  2. `rsync` the checkout, excluding `.build` and `.git`, to a fresh temp directory on
     `mac`.
  3. Run `Scripts/bundle.sh` and `just typecheck-tests` there.
  4. Delete the copy.
  5. Never use Bryan's checkout or `just install`.
  6. If the Mac is offline, say in the final report that the macOS targets are
     unverified.
- Run `just docs-check` if any `docs/` file changes.
  - Keep `docs/protocol.md` under the mounted 6000-word reference budget (see
    bob-cli-h9).
  - The known offline-lychee anchor failure already recorded on this bead is a
    clean-base exception.
- Format only changed lines (`.swift-format`: 4-space indent, 100 columns).

## 7. Gate on the exact tree, then close

1. Run `Scripts/macos-ci-check` from the Mac checkout. It pushes a throwaway
   `ci/precommit/*` snapshot and waits for that exact SHA's run. It takes over 15
   minutes, so start it through `/sase_monitor` and wait for the monitor-start command
   to exit.
2. Compare that run's failing-test set with step 1's clean-base set. Required result:
   - Lint and Build pass.
   - `testFooterContractPinsHintsAndAccessibilityText` and every new test pass.
   - The only failures are ones that also fail on clean master (the bob-cli-h7 / hu / hv
     cluster).
   - Fix any other failure and re-run the gate on the new tree. Never substitute a prior
     SHA's result.
3. Record the residual with `sase bead note bob-cli-hn.9 'PROPOSED FOLLOW-UP: …'`,
   citing bob-cli-h7 / hu / hv and both run URLs. Do not create beads.
4. Run `sase bead epic-symbols bob-cli-hn.9`, and resolve or re-key any entries.
5. Close only this phase:
   `sase bead close bob-cli-hn.9 --note "<fixes, tests, Mac compile result, snapshot SHA + run URL, residual clean-base failures>"`.
   Never close bob-cli-hn or any ancestor.
6. Commit the bob-mac-capture changes through `/sase_final`. Do not run `git commit` by
   hand.
   - The repo obligation names the opened checkout.
   - If finalization pushes master within the turn, hand the exact-SHA master run to
     `/sase_monitor`.

## Non-goals

- Fixing the bob-cli-h7 / hu / hv failures.
- Any bob-cli Rust, plugin, or memory change.
- New verdict surfaces beyond the Pull-In card: the Move card, the `:` picker, and the
  `/` picker stay out of scope, per the parent plan's non-goals.
