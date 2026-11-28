---
tier: tale
title: Fix the three q5.8 Mac wake-notice defects and land bob-cli-q5
goal:
  Bob Mac Capture claims the snooze notifier even when it starts with a held wake
  receipt, samples the real on-console session key at launch, and presents a wake HUD
  without stalling the main thread for seconds; then epic bob-cli-q5 closes and its plan
  is marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-q5.land
proposed_with: claude/opus
bead: bob-cli-q5
status: done
---

- **PARENT:**
  [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:**
  [bob-cli-q5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-q5.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-q5.land.md)
- **COMMITS:**
  - [b897b88](https://github.com/bobs-org/bob-mac-capture/commit/b897b88d4b4df5de7ce2bf6f61ad392cc8deae72)
    — fix(snooze-notice): hold-safe capability probe, real on-console key, HUD wake-path
    instrumentation

# Fix the three q5.8 Mac wake-notice defects and land bob-cli-q5

This tale is the remainder of the `bob-cli-q5` landing. It has no land agent of its own,
and nothing resumes the landing after it. Its final step closes epic `bob-cli-q5` in the
same turn as the last code change. No closeout step waits for this work's own commit,
push, or exact-SHA CI run. The pre-commit gate `Scripts/macos-ci-check` stands in for
that run.

Governing decision: `decisions:mac-capture-is-a-thin-client`. Every fix here stays
inside the existing thin-client boundary: Mac Capture still runs `bob snooze -f json`
for facts and never reads or writes the vault. The epic's reviewer decisions still
apply: `wake_notice = hud` and `hold_in_meetings = yes`. No memory note is edited
(`decision_record = no`, and the only authorized memory edit, `glossary:snoozed-task`,
has already landed).

Bryan's directive, recorded in q5 note #7, also applies: do not ask Bryan to verify,
observe, act, or answer questions. Use machine evidence: deterministic tests, signposts,
CI, and compiled builds. Record anything that cannot be proven as a residual in the
close note.

## Audited baseline

- bob-cli `0aa33a49` (= `origin/master`). All nine q5 phases and all 74 descendant beads
  are CLOSED. `sase bead epic-symbols bob-cli-q5` reports no entries. The Justfile has
  no `symvision` recipe.
- bob-mac-capture `5bda4cc3` (= `origin/master`). bob-plugins `1d5595b7`.
- q5 notes #1–#6 are resolved (note #11). Notes #8, #9, and #10 are DISCOVERED ISSUEs
  caused by the q5.8 wake-notice code. They are the only remaining epic work.
- Integration review of the commits that landed after the epic started found nothing to
  adopt or deconflict:
  - bob-cli: 90 non-q5 commits. `7213dd89` (skip automatic wake sweeps when timezone is
    unset) is a snooze fix that already composes with `684811a9`. `c261c6a3` (notes on
    existing-task links) keeps the snooze-link refusals.
  - bob-plugins: through `9c4e7775`, with no snooze-path diff.
  - bob-mac-capture: through `5bda4cc3`.

Open bob-mac-capture with `/sase_repo` (`sase repo open bob-mac-capture -r "..."`) and
read its `AGENTS.md` before editing. All paths below are relative to that checkout.

## Step 1: let the capability probe finish while a receipt is held (q5 note #8)

Root cause, confirmed in source at `5bda4cc3`, in
`Sources/BobMacCapture/SnoozeNoticeCoordinator.swift`:

1. `evaluate(reason:)` calls `probeCapabilityIfNeeded()` (~line 708) while
   `isCapable == nil`. That calls `fetchVerification()`, which bumps `fetchID` and
   starts `fetchTask`.
2. In the same pass, the `.hold` branch (~lines 802–826) handles every hold reason
   except `.agendaUnavailable`. It sets `verification = .unverified`, cancels
   `fetchTask`, clears `fetchInFlight`, and bumps `fetchID`.
3. `finishFetch` drops the probe result at its `expectedFetch == fetchID` guard (~line
   1170), so `isCapable` stays `nil`.
4. `updateClaim()` (~line 1233) requires `isCapable == true`, so `notifier.json` is
   never published. bob's `auto` channel then routes every later wake to the desktop
   fallback.

The next 2 s cycle repeats the same sequence. Any start, restart, or reconfigure with a
held receipt pending (meeting, away, locked, or session inactive) therefore never
claims. The same cancel also starves the `lastListDate == nil` day-refresh read (~line
680).

Fix: a hold must not cancel a read that the coordinator needs for capability or bob's
day. One acceptable shape:

- Cancel and bump only when `isCapable == true` and `lastListDate != nil`, or
- Run the capability probe on its own task and generation, separate from the
  verification fetch.

Pick whichever keeps the existing invariants. The required behavior is the same either
way:

- A probe that completes during a hold sets `isCapable` and the `lastList*` facts and
  publishes the claim through `updateClaim()`.
- A held receipt is still not presented while the hold lasts.
- After the hold clears, delivery still waits for a fresh post-hold verification read
  (`wasHeld && !deliveryHeld` settle and re-verify). A probe that completed during the
  hold never counts as that post-hold verification. Keep the
  `verificationFetchID > entry.verificationAfterFetchID` gating intact, and make sure
  the hold branch still leaves `verification` unverified for held entries.
- The `.agendaUnavailable` path is unchanged.

Tests, in `Tests/BobMacCaptureTests/SnoozeNoticeCoordinatorTests.swift`, reusing
`makeHarness`, `goodClient()`, `waitUntil`, and the existing presence and meeting fakes:

- `testStartWithHeldReceiptWhileAwayStillClaims`: away or locked presence at start, with
  a pending receipt delivered before or at start. Assert that `isCapableForTests`
  becomes `true`, that the claim file exists, and that nothing presented while held.
  Then return presence and assert exactly one present after a fresh verification.
- `testStartWithHeldReceiptDuringMeetingStillClaims`: the same with a running same-day
  meeting hold, delivering after the real meeting end.
- `testClaimDoesNotDependOnNotificationAuthorization`: start and attach with the
  harness's notification-authorization surface reporting not determined or denied, if
  the harness exposes one. Otherwise assert at the coordinator level that the claim
  publishes with no authorization input at all, and add a one-line comment naming why.
  Note #8 suspected the acceptance `noauth` variant failed to claim for this same
  starvation. The coordinator has no authorization dependency (`updateClaim` gates only
  on started, observer, presenter, capability, and process client). Live re-proof on the
  harness is not required. Record it as a residual.
- Keep `testMeetingHoldDelaysUntilRealEnd`, `testOverlappingMeetingsHoldPastFirstEnd`,
  `testHoldWhileAwayThenDeliverOnReturn`, `testNoUsableAgendaHoldsWithRefreshBackoff`,
  and the claim tests green and unmodified in intent.

Every async test must stay timed: use `waitUntil` deadlines and no untimed
continuations.

## Step 2: read the real on-console key at launch (q5 note #9)

`SnoozeWakePresenceSource.liveInitialState()`
(`Sources/BobMacCapture/SnoozeWakePresenceSource.swift`, ~line 91) looks up
`"kCGSessionOnConsoleKey"`, `"CGSessionOnConsole"`, and `"onConsole"`. The real session
dictionary key is the constant's value, `"kCGSSessionOnConsoleKey"` (seen in
IOConsoleUsers on the Mac). The lookup misses, so `sessionInactive` always starts
`false`.

Fix:

1. Factor the dictionary interpretation into a pure static function, for example
   `static func initialState(fromSession dict: [String: Any]?, displaysAsleep: Bool)`,
   that `liveInitialState()` calls.
2. Include `kCGSessionOnConsoleKey` (the CoreGraphics constant, bridged to `String`) and
   the literal `"kCGSSessionOnConsoleKey"` first in the on-console key list. Keep the
   old spellings only as harmless fallbacks.
3. Keep fail-open: with no dictionary or no key, report present.

Tests, in `Tests/BobMacCaptureTests/SnoozeWakePresenceSourceTests.swift`:

- An off-console sample (`["kCGSSessionOnConsoleKey": false]` and the `NSNumber(0)`
  form) yields `sessionInactive == true`.
- An on-console sample yields `false`.
- A missing key or nil dictionary yields `false`.
- The existing lock-key cases still parse.

## Step 3: take slow work off the synchronous wake-present path (q5 note #10)

Evidence: two production-path wakes showed a 2.2 s and a 2.8 s main-thread gap between
the coordinator `snooze-present` signpost and the `snooze-record` signpost. In
`AppDelegate.swift` (~lines 639–642), `coordinator.onPresent` runs `hud.show` and then
`notificationService.postSnoozeWakeRecord`. The record post does only a signpost plus a
`Task` before any await, so the gap is in `present()` (`beaconStore.save`) and
`SnoozeWakeHUD.show` (`Sources/BobMacCapture/SnoozeWakeHUD.swift`, lines 113–210). That
show path synchronously:

- builds the panel and SwiftUI hosting view on first use
- calls `layoutSubtreeIfNeeded` and `fittingSize`
- runs `GlassPanelHost.install`, `setFrame(display: true)`, and `orderFrontRegardless()`
- plays `NSSound(named: "Pop")`. A cold first play can initialize CoreAudio.
- posts the `NSAccessibility` announcement

The root cause is not proven. Do all three of the following:

1. **Attribute.** Add metadata-only interval signposts around the show stages, with no
   task text: panel/hosting build, layout plus install, order-front, chime, and
   announce. Also add one around `beaconStore.save` in `present()`. Use the existing
   `CaptureSignpost` helpers and naming style (`snooze-hud-*`). Update the signpost list
   in `docs/` wherever the existing snooze signposts are documented.
2. **Restructure** so the HUD reaches the screen before non-visual work:
   - Put the panel on screen first.
   - Run the chime and the accessibility announcement on the next main-queue turn,
     guarded by the same `generation`/`wakeID` check, so a superseded or dismissed wake
     does not chime.
   - Preload the chime sound once at HUD init or `attachPresenter`, not per show.
   - Optionally prewarm the hidden panel and hosting view once after launch, off the
     wake path.

   Behavior must not change:
   - one chime per new wake ID
   - the announcement still fires for a new wake or order-in
   - `restore` stays silent
   - the HUD's 12 s countdown starts when the panel is on screen

   Existing HUD tests that assert `lastChimePlayed`, `lastChimeSound`, or the
   announcement sink synchronously may need to wait for the next main-queue turn. Update
   them to wait with a timed deadline, not to assert less.

3. **Measure, best effort, without Bryan.** If the Mac is reachable and the scratch
   build in Step 4 runs, you may launch the scratch copy only under its own scratch
   state and bundle conventions. Never touch Bryan's production app, `~/bob`, or
   `just install`. Then read the new interval signposts for one synthetic wake.

   If no safe measurement is possible, record the unmeasured residual in the close note:
   the stall is restructured and instrumented but not re-timed. The attribution
   signposts are the durable deliverable either way.

## Step 4: verify bob-mac-capture

Follow its `AGENTS.md`:

1. Format only the lines you changed with the repo's `.swift-format` (4-space indent,
   100 columns). Never re-indent whole files.
2. Run `just portable-check` on athena. It cannot type-check `Sources/BobMacCapture*` or
   `Tests/BobMacCapture*`, which is where all three fixes live, so it is a floor, not
   proof.
3. Best effort on the Mac (`ssh` per the tailnet memory):
   - rsync a scratch copy without `.build`/`.git` to a temp directory
   - run `bundle.sh` and `just typecheck-tests` there
   - delete the copy

   Never use Bryan's checkout or `just install`. If the Mac is offline, say so in the
   close note: macOS targets were unverified locally and CI is the only proof.

4. If `docs/` changed, run `just docs-check`.
5. Run `Scripts/macos-ci-check` from the bob-mac-capture checkout on the final working
   tree, through `/sase_monitor` if it will outlast a foreground call, and wait for it.
   It must report every macOS step green, including Test, which runs the new
   coordinator, presence, and HUD tests.
   - Fix and rerun on any red caused by this work.
   - A red that `bob-cli-qk` (MeetingNotes ordering) or `bob-cli-rw` (Go To panel)
     already tracks is a known flake. Rerun once, and record the +1 through
     `/sase_new_task` if it recurs.
   - A red **Lint Swift formatting** skips the later steps, so read the whole log.

bob-cli itself needs no code change. Do not run `just check-full`.

## Step 5: close epic bob-cli-q5 (final step)

Do these in order, in this same turn:

1. Run `sase bead epic-symbols bob-cli-q5`. At audit time it reported no entries. If any
   have appeared, resolve each symbol (wire it up, privatize it, add a non-test pragma,
   or delete it per the Symvision epic-whitelist policy). Re-key a Justfile
   `--epic-symbol` line only to a still-open later bead that still needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-q5 --note "<verification>"
   ```

   The note must cover:
   - all 9 phases and 74 descendants closed
   - q5 notes #1–#6 resolved per note #11
   - notes #8–#10 fixed, with the bob-mac-capture change summary, the new test names,
     and the `macos-ci-check` run ID and verdict
   - the Mac scratch typecheck result, or that the Mac was offline
   - the residuals: noauth variant not re-run live, and HUD stall re-timed or not
   - the integration review result above
   - the follow-up triage summary recorded on the epic by its lander

   Never use `--force` to make the close succeed. If the close is refused for leftover
   epic symbols, clean them up and close again.

3. Run `just symvision` if the recipe exists. At audit time it does not, so record "no
   symvision recipe" in the close note instead of claiming a pass.
4. In the bob-cli checkout, set `status: done` in the frontmatter of
   `sase/repos/plans/202611/snoozed_tasks.md`. It currently reads `status: wip`. Change
   nothing else in that file.
5. `bob-cli-q5` has no `parent_bead`, so the landing chain ends here.

Finish through `/sase_final`, with commit decisions for the bob-mac-capture checkout and
the bob-cli plans sidecar change.
