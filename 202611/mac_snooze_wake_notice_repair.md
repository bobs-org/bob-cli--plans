---
tier: epic
title: Repair and accept the Mac snooze wake notice
goal: Finish bob-cli-q5.8.1 by making the landed Mac wake notice compile, verify every
  wake against a fresh bob read, honor bob's day, vault, presence, and meeting facts,
  meet the HUD, beacon, and record contract, and prove it end to end on the real Mac.
parent_bead: bob-cli-q5.8.1
phases:
- id: runtime-verify
  title: Restore the build and verify each wake against a fresh bob read
  size: medium
  depends_on: []
  description: 'runtime-verify: fix the StatusItemGlyph build break, make every accepted
    receipt verify against a bob snooze read started after it, deliver immediately
    when nothing held, gate and atomically publish the notifier claim, and make the
    snooze process lanes serialize writes, surface bob refusals, and never cancel
    each other.'
- id: runtime-facts
  title: Take day, vault, presence, and meeting facts from their real owners
  size: medium
  depends_on:
  - runtime-verify
  description: 'runtime-facts: use bob''s date/now and vault_root as the authorities,
    hold instead of delivering when the agenda has no same-day facts, sample presence
    at launch, and re-evaluate when a meeting starts while a notice is on screen.'
- id: hud-repair
  title: Bring the wake HUD and its actions up to the delivery contract
  size: medium
  depends_on:
  - runtime-facts
  description: 'hud-repair: exclude the HUD from capture, render the TODAY link budget
    and woke time from bob, share task navigation with the record, end the nudge on
    x/timeout without replays, key HUD state by wake ID, keep hover across updates,
    and make +15m copy truthful.'
- id: records-repair
  title: Fix the beacon glyph, record lifecycle, diagnostics, and docs
  size: medium
  depends_on:
  - hud-repair
  description: 'records-repair: keep the base status glyph template-tinted beside
    an amber alarm beacon, clear beacon and records on vault change, stop restores
    from re-posting records, isolate snooze diagnostics, finish the protocol and settings
    docs, and close the records test gaps.'
- id: notice-acceptance
  title: Accept the wake notice end to end on the real Mac
  size: medium
  depends_on:
  - records-repair
  description: 'notice-acceptance: with a compatible bob and the built app, run the
    six end-to-end checks of plan:202611/mac_snooze_wake_notice.md, including a genuine
    human lock/unlock, fix small defects found, and record exact evidence for the
    land agent.'
proposed_by: bbugyi200.athena.bob-cli-q5.8.1.land
proposed_with: claude/opus
create_time: 2026-11-26 20:05:48
status: wip
bead_id: bob-cli-q5.8.1.6
---

- **PROMPT:** [prompts/202611/mac_snooze_wake_notice_repair.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_snooze_wake_notice_repair.md)
- **PARENT:** [202611/mac_snooze_wake_notice.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/mac_snooze_wake_notice.md)
- **BEAD:** [bob-cli-q5.8.1.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.8.1.6.md)

# Repair and accept the Mac snooze wake notice

## Why this child epic exists

`bob-cli-q5.8.1` (plan `plan:202611/mac_snooze_wake_notice.md`) landed all five phases,
but its land agent found the result incomplete:

- **It does not compile.** macOS CI run `45917911039` on bob-mac-capture `d2e9ca4a`
  failed in **Build**:
  `StatusItemGlyph.swift:232:36: error: immutable value 'self.accessibilityLabel' may only be initialized once`.
  The test target never built, so nothing from the final phase (`d2e9ca4a`) has a green
  compile behind it. The Mac was offline during that phase.
- **The main delivery path drops real wakes.** The coordinator's `verification` state is
  set once by the launch capability probe and never reset when a new receipt arrives. A
  wake that occurs while the user is active is judged against a pre-wake
  `bob snooze -f json` read. It is dropped as `emptyVerification` or `noEligibleTasks`
  and persisted as handled, and bob has already routed it to `mac_capture`, so no
  desktop banner appears either.
- About twenty further contract gaps are listed per phase below. Each was confirmed by
  reading `git diff 46c5202b d2e9ca4a` in bob-mac-capture.

The bob-cli half (`3659edc0`, `59037b23`: `notification_channel`, `vault_root`,
`wake_notice_version`, `wake_notice_eligible`) was reviewed and is sound. The Swift
models match its field names and shapes. **No bob-cli change is planned.** If a phase
finds that one is needed, it must stay additive and leave Cargo versions and
`CHANGELOG.md` alone (`decisions:bob-cli-releases-ride-a-daily-train`).

This epic only repairs and accepts work the parent epic planned. Every requirement below
restates a requirement of `plan:202611/mac_snooze_wake_notice.md`, or of its parent
`plan:202611/snoozed_tasks.md` phase `mac-wake-notice` (bead `bob-cli-q5.8`). Read both
plans with `sase artifact read` before starting. Add no new product scope.

## Settled decisions and governing records

The `bob-cli-q5.8` reviewer decisions are final:

- `wake_notice = hud`
- `hold_in_meetings = yes`
- `uncommitted_rows = ignore`
- `glossary_snoozed_task = yes`, owned only by the outer close-out phase `bob-cli-q5.9`,
  not this epic
- `decision_record = no`

Do not re-ask them, and do not implement the banner branch. **No memory changes belong
in this epic.**

Governing decisions:

- `decisions:mac-capture-is-a-thin-client`: bob owns task truth, the task day, the
  canonical vault identity, and every vault mutation. Swift decodes bob's JSON
  additively, rendering and orchestrating only.
- `decisions:idle-capture-shows-ledger-agenda`: meeting facts come from the existing
  `CaptureAgendaStore` and its refresh path. There is no second agenda and no vault
  Markdown parsing. (`decisions:idle-capture-mirrors-walk-attention` supersedes it only
  for walk marks.)
- `decisions:today-is-read-from-the-ledger`: "still Today" means bob's live-link facts
  (`wake_notice_eligible`), never the woken row alone.
- `decisions:event-buffers-widen-the-span`: meeting holds use the actual
  `startsAt <= now < endsAt` span, not the buffered span.
- `decisions:q-and-ctrl-bracket-mirror-esc`: any local HUD Escape handling shares
  dispatch with q and Ctrl+[. No global key capture, and no stealing the key window.
- `decisions:bob-cli-releases-ride-a-daily-train`: see above.

## Repository, gates, and rules for every phase

- **Repository.** All code is in bob-mac-capture. Open it with `/sase_repo` and use only
  the path it prints. Read its `AGENTS.md` first. Paths below are relative to that
  checkout.
- **Formatting.** `.swift-format` pins the house style. Format only the lines you
  change.
- **Portable gate.** Every phase runs `just portable-check`. If Swift is missing
  locally, run it on athena.
- **Mac compile.** When `Sources/BobMacCapture*` or `Tests/BobMacCapture*` changes,
  compile a scratch Mac copy:
  - rsync it without `.build` and `.git`;
  - run `Scripts/bundle.sh` and `just typecheck-tests`;
  - delete the copy.

  Read `tailnet.md` with `/sase_memory_read` before remote work. Never use Bryan's
  checkout or `just install`. If the Mac is offline, record "Mac targets unverified" and
  rely on exact-SHA CI. Never treat an offline Mac as a passing compile.

- **CI.** Before landing Swift, run `Scripts/macos-ci-check` through `/sase_monitor`.
  After every push, watch the exact-SHA macOS CI run to green with `/sase_monitor`. A
  phase is not complete while its exact-SHA CI is red. CI is the only gate that runs the
  app-target tests.
- **Docs.** Run `just docs-check` when docs change. The bob-cli lint failure
  `docs/reference/vault-format.md` (6030/6000 words) is pre-existing and tracked by bead
  `bob-cli-oc`. Record it as baseline; do not fix it here.
- **Tests.**
  - Inject clocks, presence, defaults, and process completions. Async tests must never
    await an untimed continuation or gate.
  - Extend `Tests/Fixtures/fake-bob` so a test can change the snooze list response
    between calls. Several fixes below are only observable when the post-wake read
    differs from the launch probe.
- **Follow-ups and closing.** Phase workers record discovered follow-ups as
  `PROPOSED FOLLOW-UP:` notes on their own phase bead, never as new beads. A phase
  closes only itself, after checking `sase bead epic-symbols` for its own bead. No phase
  closes `bob-cli-q5.8.1` or `bob-cli-q5.8`. When this child epic lands, its land agent
  resumes the interrupted `bob-cli-q5.8.1` landing through `parent_bead`.

## Phase runtime-verify

Make the delivery path correct before anything else depends on it.

1. **Build first.** In `Sources/BobMacCapture/StatusItemGlyph.swift` (around lines
   228-233), assign `accessibilityLabel` exactly once on every branch, keeping the
   tooltip and label text unchanged. Get a green compile (Mac scratch or exact-SHA CI)
   before the other items, so every later CI run tests code that built.
2. **Fresh verification per batch.** In
   `Sources/BobMacCapture/SnoozeNoticeCoordinator.swift`:
   - Verify a pending batch only against a `bob snooze -f json` read that **started
     after that batch was accepted**. Neither the launch capability probe nor a read
     made for an earlier batch counts.
   - Implement this by tagging verification results with the fetch that produced them,
     or by resetting to `.unverified` on receipt acceptance, after `present`, after
     `handleDrop`, and on `reconfigure`. Choose whichever keeps the generation and
     fetch-ID guards simple.
   - Never persist a wake ID as handled because of a stale or foreign verification.
   - Tests:
     - The probe returns no eligible rows, then a matching receipt arrives with a
       post-wake list that contains its row. The HUD shows.
     - A second distinct wake after a shown batch also shows.
     - A receipt whose post-wake list truly lacks its rows is dropped.
3. **Immediate active delivery.**
   - When the batch was never held, start its verification read immediately.
   - The 1.5-second settle applies only after a hold clears, and a new hold still
     invalidates the timer and read. Today every delivery waits 1.5 s because the policy
     returns `verify(recheckAt: now + 1.5)` for any unverified batch.
   - Update `SnoozeWakeNoticePolicy` and its tests to match. Remove the policy's unused
     internal generation token, or have the coordinator actually use it; do not leave a
     dormant token that changes on each `decide` call.
4. **Stale completions.**
   - In `finishFetch`, clear `fetchInFlight`/`fetchTask` only when the completion passes
     the generation and fetch-ID guard. A superseded read must not mark the current read
     finished.
   - A real lifecycle or configuration change (`reconfigure`, day/clock change,
     unlock/return) clears `fetchRetryAfter`. Ordinary failures still retry no more
     often than every 30 seconds.
   - Test late results after stop/reconfigure, and test the retry spacing.
5. **Claim gating and atomicity.** In `SnoozeWakeObserver.swift` and
   `SnoozeWakeNotifierClaim.swift`:
   - The observer exposes whether `notify_register_dispatch` succeeded. `updateClaim`
     requires that, besides the attached presenter and the version-1 probe. A failed
     registration publishes no claim, so bob's `auto` falls back to desktop.
   - Publish the claim by creating the temp file with 0600 permissions from the start,
     then making one atomic `rename(2)` over `notifier.json`. There must be no
     remove-then-move window in which bob sees no claim.
   - Overlapping instances need deterministic single ownership with no
     republish-every-evaluate flip-flop. An instance removes only a claim it still owns
     (pid plus start token). A displaced instance re-claims only when the claim is
     missing or its owner is dead, which bob's `parse_live_claim` decides (bob-cli
     `src/native/snooze/notify.rs`).
   - Test ordering (observer registered and presenter attached before claim), failed
     registration, and two overlapping instances.
6. **Process lanes.** In `Sources/CaptureCore/BobProcessClient.swift`:
   - `snooze-add` must actually serialize: a second add starts only after the first
     process exits. It is never cancelled by UI dismissal or a read. Make the doc
     comment true.
   - On refusal, bob prints `{"ok":false,"error":{"kind","message"}}` to stdout and
     exits 2 (bob-cli `src/native/snooze/cli.rs`, the JSON error path). Decode that into
     a typed refusal carrying `kind` and `message`, so the HUD can show bob's words
     (`hud-repair` renders them).
   - Coordinator verification reads and the HUD's +15m revalidation must not cancel each
     other. Either provide a coordinator-owned verification entry point that the HUD
     will call, or give them separate non-cancelling read lanes. Keep `snooze-wake`
     unchanged.
   - Tests: exact argv, add serialization, a decoded refusal, and read-lane isolation.
7. **Receipt strictness.** In `SnoozeWakeReceipt.swift`:
   - A receipt without `schema_version` is not schema 1.
   - A well-formed receipt with an unknown future schema is ignored silently, not
     reported as malformed.
   - Add a fixture whose `woken_at` has fractional seconds and an offset, to match
     chrono's `to_rfc3339()`.

## Phase runtime-facts

Every fact must come from the owner the parent plan named.

1. **bob's day.**
   - Use bob's `date`/`now` from the latest list read made for the batch as the task day
     and wall-time authority. Never use `CaptureAgendaStore.localToday()` or the Mac
     clock: not for the receipt-day check, the handled-ID day, purge, meeting-event
     filtering, or wall-time agenda parsing.
   - Today `SnoozeNoticeCoordinator` defaults `todayProvider` to `localToday()` (around
     line 128), and `AppDelegate.swift` around lines 561-580 never injects bob's date.
   - Order the checks so that a receipt is checked against bob's date only after a fresh
     read. Day/clock-change signals require a new read; never pair a stale receipt date
     with a new local clock.
   - Tests:
     - bob's zone differs from the Mac zone near midnight: bob in New York, Mac in Los
       Angeles, 21:00-24:00 PT.
     - A DST transition, with differing instants rather than equal ones.
     - bob's real agenda wall-time format (`YYYY-MM-DDTHH:MM`, no offset), interpreted
       with bob's offset and not UTC.
2. **Vault identity from bob.**
   - Compare the receipt's `vault_root` with the `vault_root` reported by the current
     `bob snooze -f json` read. That read runs with the app's configured environment,
     and bob canonicalizes both values.
   - If you also compare against Settings, resolve symlinks on both sides. A symlinked
     `~/bob` or a `/tmp` (`/private/tmp`) vault must not drop every receipt as
     `vaultMismatch`.
   - Missing `vault_root` on either side is not permission to announce.
   - Test a symlinked vault.
3. **No agenda is not "no meetings".**
   - When Hold during meetings is on and the agenda has no usable snapshot for bob's
     day, hold the batch with a visible held reason and request a refresh with bounded
     backoff, retaining it until expiry. This covers loading, failed with no same-day
     data, and other-day.
   - A same-day snapshot whose last refresh failed may still supply its last good
     events.
   - Today `runningMeetingEnd(events: nil)` returns no hold (policy around line 362),
     and `AppDelegate`'s provider returns nil for `.stale`, `.failed`, `.loading`, and
     other-day snapshots.
   - Replace `testMeetingHoldDelaysUntilRealEnd`'s "nil events releases" assumption. Add
     a test where a refresh fails during a running meeting, and a test where overlapping
     meetings hold past the first meeting's end.
4. **Launch presence.**
   - `SnoozeWakePresenceSource` must sample the initial lock and session state
     (`CGSessionCopyCurrentDictionary`: screen-locked and on-console keys) and the
     display-sleep state at start, instead of assuming present. Inject the sampler for
     tests.
   - Replace `testInitialPresenceIsPresent` with tests for launching locked and
     launching with the session inactive.
5. **Meeting start during display.** While a notice is on screen, schedule a
   re-evaluation at the next qualifying meeting start from the agenda. A meeting that
   begins mid-display then hides the HUD immediately. Lock and display sleep are already
   event-driven.

## Phase hud-repair

Bring `Sources/BobMacCapture/SnoozeWakeHUD.swift` and the coordinator's HUD callbacks up
to the delivery contract.

1. **Capture exclusion.** Set `sharingType = .none` on the snooze HUD panel, either
   through an explicit `GlassPanelHost` interaction/sharing choice or in `makePanel`.
   Leave Refs toast behavior byte-equivalent. Test the panel configuration.
2. **Subtitle facts from bob.**
   - The second line is `woke 15:32 · in WRITE · TODAY 11/13`. `11/13` is the current
     list read's `plan.count`/`plan.cap` (bob's live-link budget; `plan.over` marks over
     cap). It is **not a calendar date**. Today the HUD and the record body print
     `TODAY <month>/<day>` from `batch.date`.
   - When over cap, the budget text itself (for example `14/13`) is amber, so the signal
     is both text and color.
   - `woke HH:MM` uses bob's `woke_at`, not Mac-zone formatting of `wokenAt`.
   - Apply the same copy to the record body built in `NotificationService`.
   - Test the exact strings for single, batch, and over-cap. Change the render fixture
     so its date cannot coincide with its budget.
3. **One navigation path.**
   - HUD Open, each separately clickable batch title, and the clickable body all use the
     same validated route + block-ID navigation as the record's default action:
     `ObsidianOpenURL.taskGotoURL` with the task's block reference.
   - Today `validatedTaskURL` pairs the task file with the list row's `line`. That is
     the line of the Snoozed row in the daily note, and no block or ref is passed, so
     Open lands on the wrong line.
   - Open acknowledges the batch through the shared acknowledgement path. Test the URLs.
4. **× and timeout end the nudge.**
   - The coordinator must learn, keyed by wake ID, that the transient HUD ended.
   - A batch whose nudge ended is never re-presented, re-chimed, re-announced, or
     re-posted after a later hold. Today `restoreSuppressedDeliveries` calls `onPresent`
     for every delivered batch, which re-shows the HUD and re-posts the record.
   - After a hold clears, restore only a HUD that the hold hid while it was still on
     screen. Restore it without a chime, without a second announcement, and without
     re-posting the record.
   - The beacon and record keep their own lifecycle.
   - Test: timeout, then lock, then unlock gives no replay; display, then lock, then
     unlock restores silently.
5. **Wake-ID-keyed HUD state.**
   - `onTransientDismiss`/`onDismiss` in `AppDelegate.swift` hide only the HUD showing
     that wake ID.
   - A new batch arriving while +15m is in flight must not inherit the in-flight
     feedback. Queue it until the action settles, or keep per-batch state.
6. **Hover across updates.** Re-showing or updating the HUD (after a partial +15m or a
   stale-row removal) preserves the hover state. The 12-second countdown never restarts
   or runs under the pointer.
7. **Truthful +15m.**
   - Revalidate through the non-interfering read from `runtime-verify`.
   - After full success, the headline states the outcome: for example
     `Snoozed until 15:47`, using bob's returned target. Today it falls back to the
     zero-task "Snoozed tasks are ready".
   - Partial failure keeps the failed tasks, with bob's refusal message.
   - A near-midnight `invalid_time` refusal shows bob's words, not a generic error.
     Never retry a task that already succeeded.
   - Tests: all-success, partial with a multi-row batch, and refusal.
8. **Motion and voice.**
   - The alarm glyph rings once per actual presentation, including a reused hosting view
     (not only on the first `onAppear`), and never under Reduce Motion.
   - The VoiceOver announcement uses a request that is spoken for a non-key,
     non-activating panel: post `.announcementRequested` with high priority against an
     element VoiceOver tracks.
   - The chime plays exactly once per wake ID.
9. **Renders.**
   - Through the existing app render mechanism and the real `GlassPanelHost` path,
     render: single, batch of three or fewer, batch over cap, light, dark, and Reduce
     Transparency.
   - CI uploads render fixtures. Record the run ID and artifact names on the phase bead.

## Phase records-repair

1. **Beacon glyph.**
   - Keep the base Bob glyph template-tinted so it stays legible on light, dark, and
     highlighted menu bars. Today `StatusItemGlyph` draws the base in `NSColor.black`
     and turns the whole image non-template when the count is above 0.
   - Render the beacon as a small amber alarm-clock accessory with its count, so
     template tinting cannot erase the amber. For example, use a separate non-template
     accessory layer or view, or draw the base in the button's effective appearance
     color.
   - Preserve the unresolved-bob marker and the capture-landed pulse. A stale pulse
     never restores a cleared count.
   - Update the tooltip and accessibility text.
   - Add glyph tests and a beacon render, light and dark.
2. **Vault change clears.**
   - The docs promise that beacon and records clear on vault change, so implement it.
   - A reconfigure to a different vault or bob executable withdraws that vault's
     delivered and pending records and clears its delivered and restored beacons. Today
     foreign-vault purge happens only at launch.
   - Restores after a hold never re-post a record (see `hud-repair` item 4).
3. **Diagnostics isolation.**
   - Snooze diagnostics feed their own metadata-only Settings row and must not overwrite
     the app-wide `diagnosticStatus`. Today they replace "Ready" or the bob-resolution
     error.
   - Keep task text out of logs and signposts.
4. **Docs.**
   - Make `docs/settings.md` and `docs/diagnostics.md` match the code: where the Wake
     notices diagnostics row lives (today the Snoozed Tasks section of
     `SettingsView.swift`), and the clearing triggers.
   - In `docs/protocol.md`, document:
     - `bob snooze -f json` and `bob snooze add <route:block-id> +15m -f json`;
     - the `last-wake.json` receipt as a latest-only, not-a-replay-queue record;
     - the `org.bobs.bob.snooze.woken` Darwin notification;
     - the `notifier.json` claim and channel precedence;
     - the old-bob fallback.

     Stay within the mounted reference word budget. Related open bead `bob-cli-h9`
     tracks `protocol.md` budget pressure; move detail to a shorter page rather than
     overrunning it.

   - Add the user-facing description of the HUD where Mac Capture documents its
     surfaces.
   - Run `just docs-check`.

5. **Test gaps.**
   - Fix `testSixtyMinuteBeaconExpiryClears`: it currently exercises day rollover. Make
     it test the 60-minute expiry, and keep a separate day-rollover test.
   - Fix `testRemoveDeliveredTasksPartialKeepsRemainder` to use a multi-row batch.
   - Add tests for:
     - authorization denied or not determined, without a prompt;
     - the add-versus-clear race;
     - launch reconciliation that leaves unrelated notifications alone;
     - restored-beacon clearing by each trigger;
     - foreign-vault purge;
     - the beacon restored across restart when record permission is denied.

## Phase notice-acceptance

Perform the end-to-end acceptance the parent plan assigned to its land agent, now that
the code is complete. Record exact commits and check receipts on this phase bead.

1. **Preconditions.**
   - bob-mac-capture master's exact-SHA macOS CI is green.
   - The Mac is reachable (`tailnet.md`).
   - A compatible bob is available: `bob snooze -f json` reports
     `wake_notice_version: 1`. Use a scratch bob build from bob-cli master if the Mac's
     installed bob predates `3659edc0`.
   - If the Mac is offline, ask Bryan through `/sase_questions` to bring it online. Do
     not record an unverified result as passing.
2. **Isolation.**
   - Do not disturb the production installation or Bryan's vault to manufacture
     evidence. Prefer a scratch bundle built from the landed SHA, running against a
     disposable vault, with its own `XDG_STATE_HOME`, so its claim, receipt, and
     handled-ID state do not touch production, plus its own defaults domain if the
     bundle can be given one.
   - If a check can only be observed with Bryan's installed app, ask Bryan through
     `/sase_questions` first, and record exactly what he consented to and what was
     observed.
3. **Checks.** Run the six checks from `plan:202611/mac_snooze_wake_notice.md`
   "Verification and land responsibilities", recording each outcome:
   1. One active wake gives one HUD, one chime, and one silent record. Duplicate pings,
      a repeated wake, and an app restart give no second nudge.
   2. A real locked-screen wake stays silent, and the surviving task appears after
      unlock, settling, and revalidation. This needs a **genuine human lock/unlock**:
      coordinate it through `/sase_questions`, and record what was actually observed.
      Idle, display sleep, and session inactivity rely on the deterministic adapter
      tests.
   3. A running meeting, including an overlapping one, holds the notice until its real
      end. A task completed, released, or re-snoozed during the hold is filtered out.
   4. Open and batch +15m work, partial or refused writes stay truthful, hover pauses
      the 12-second countdown, and focus stays in the original app.
   5. Beacon and record clearing, notification denial, Reduce Motion and Reduce
      Transparency, the VoiceOver announcement, day rollover, and the 60-minute expiry
      all match the contract.
   6. With the app stopped, `auto` uses the desktop fallback. With it running, explicit
      `desktop` and `off` never also produce the HUD. An old bob leaves no claim, and
      the newer capability enables the feature.
4. **Fixes.**
   - Fix small defects found here in place, with the compile and CI gates above.
   - Record anything not caused by this feature as `PROPOSED FOLLOW-UP:`.
   - A defect too large for this phase gets a `PROPOSED FOLLOW-UP:` note naming the
     failing check, and the phase stays open.
5. **Evidence.** Close this phase with a note listing, per check, the SHA, the CI run
   ID, and the observation or test that proves it.
