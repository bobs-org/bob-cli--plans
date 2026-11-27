---
tier: epic
title: Finish the Mac wake notice acceptance on an isolated harness
goal: Close bob-cli-q5.8.1.6.5 by running the six end-to-end wake-notice checks on
  the real Mac against a scratch app that has its own bundle ID, defaults domain,
  hotkey, state, and vault, so production Bob Mac Capture is never disturbed again,
  and by recording durable per-check evidence.
parent_bead: bob-cli-q5.8.1.6.5
decisions:
  harness_home:
    ask: Should the acceptance harness be committed to bob-mac-capture as a dev script?
    choices:
      commit: One reviewed script in Scripts/ that every phase reuses; costs one push
        and CI run
      scratch: Lives only in Mac /tmp and bead notes; no CI, but a reboot loses it
    default: commit
    why: four agents share the harness across turns and possible reboots
    answer: commit
phases:
- id: harness
  title: Build the isolated acceptance harness and settle the CI precondition
  size: medium
  depends_on: []
  description: 'harness: settle master exact-SHA CI for ba213181, then build a scratch
    Bob Mac Capture with its own bundle ID, defaults domain, disabled hotkeys, XDG
    state, disposable vault, signpost evidence log, production guard, and teardown,
    prove one wake end to end through signposts, and get its notification permission
    granted.'
- id: delivery
  title: Accept delivery, lock and unlock, and channel routing
  size: medium
  depends_on:
  - harness
  description: 'delivery: on the harness, run checks 1, 2, and 6 of plan:202611/mac_snooze_wake_notice.md:
    one active wake gives one HUD, chime, and record with no second nudge from pings,
    repeats, or restart; a genuine locked wake stays silent and appears after unlock;
    and auto, desktop, off, and old-bob routing behave.'
- id: meetings-actions
  title: Accept meeting holds and HUD actions
  size: medium
  depends_on:
  - delivery
  description: 'meetings-actions: on the harness, run checks 3 and 4: running and
    overlapping meetings hold until their real end and filter tasks changed during
    the hold; Open and batch +15m work, partial and refused writes stay truthful,
    hover pauses the countdown, and focus stays in the original app.'
- id: records-a11y
  title: Accept beacon, records, accessibility, and expiry
  size: medium
  depends_on:
  - meetings-actions
  description: 'records-a11y: on the harness, run check 5: beacon and record clearing,
    notification denial, Reduce Motion and Reduce Transparency, the VoiceOver announcement,
    day rollover, and the 60-minute expiry, then tear the harness down.'
proposed_by: bbugyi200.athena.bob-cli-q5.8.1.6.5
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-27 06:58:53
status: wip
bead_id: bob-cli-q5.8.1.6.5.1
---

- **PROMPT:** [prompts/202611/mac_wake_notice_acceptance.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_wake_notice_acceptance.md)
- **BEAD:** [bob-cli-q5.8.1.6.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.8.1.6.5.1.md)

# Finish the Mac wake notice acceptance on an isolated harness

## Why this child epic exists

Phase `bob-cli-q5.8.1.6.5` (`notice-acceptance` of
`plan:202611/mac_snooze_wake_notice_repair.md`) must run the six end-to-end checks of
`plan:202611/mac_snooze_wake_notice.md` ("Verification and land responsibilities") on
the real Mac. Six agent runs worked it on 2026-11-27 and left it open. Read every note
on that bead with `sase bead read bob-cli-q5.8.1.6.5` before starting. In short:

- **Isolation failed.** The earlier scratch bundle kept bundle ID
  `org.bobs.bob-mac-capture`. It therefore shared the production defaults domain, the
  LaunchServices identity, the Carbon capture hotkey, and the Notification Center app
  with Bryan's installed app. `Sources/BobMacCapture/AppRelauncher.swift` notes that
  LaunchServices dedupes by bundle identifier. Its `bobDirectory` override leaked into
  production, and the scratch app took over Bryan's capture hotkey. Bryan's
  `just install-all` then relaunched production against the disposable vault and killed
  the scratch app. A "keeper" that restarted the scratch app was never identified.
- **Production is restored.** At 06:51 EST the `bobDirectory` key was deleted (its
  recorded original state was absent), and production was relaunched from
  `~/Applications` against `~/bob` with a live claim. All earlier Mac debris under
  `/tmp` was removed.
- **Evidence was lost.** The 06:39 locked-screen retest (`18ebe4001dc96750-0-76504`) has
  no evidence. The agent died before asking what Bryan saw, the app's signposts had aged
  out of the unified log, and Notification Center held no record. The earlier locked
  attempt is also inconclusive. Bryan saw "the banner" on the lock screen, but it was
  never established whether that was the app's record or bob's desktop fallback from a
  no-claim wake. No post-unlock HUD appeared.
- **What stands:**
  - Check 1's single active wake `18ebe3a676f46ca8-0-38001` at 06:33 gave one HUD, one
    chime, and one record (Bryan's answer).
  - bob's CLI routing matrix in note 3 stands: auto with no claim → desktop, auto with a
    live claim → `mac_capture`, desktop → desktop, off → off.
  - Mac bob 0.6.0 reports `wake_notice_version: 1`.
- **CI precondition.** bob-mac-capture master `ba213181` CI run `45960329324` was red
  only in `CaptureTaskGotoPanelTests.testLateFailureAfterChipReopenSuccessHasNoEffect`.
  That file is untouched by `ba213181`. A failed-jobs rerun was requested at about 06:55
  EST.

This epic adds no product scope. It builds a harness that cannot touch production, then
reruns the checks with evidence that outlives any one agent.

## Settled decisions and governing records

The `bob-cli-q5.8` decisions stay final: `wake_notice = hud`, `hold_in_meetings = yes`,
`uncommitted_rows = ignore`, `decision_record = no`. The glossary work belongs to
`bob-cli-q5.9`, not here. **No memory changes belong in this epic.**

Governing records:

- `decisions:mac-capture-is-a-thin-client`: every vault change in a check goes through
  the `bob` CLI with the scratch environment. Only the harness's fixture setup writes
  the disposable vault's files directly.
- `decisions:idle-capture-shows-ledger-agenda` and
  `decisions:today-is-read-from-the-ledger`: meeting and Today facts come from bob's
  agenda and live-link facts.
- `decisions:event-buffers-widen-the-span`: a meeting hold ends at the event's real end,
  not its buffered end.
- `decisions:q-and-ctrl-bracket-mirror-esc`: any HUD Escape observation must match q and
  Ctrl+[.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit bob-cli `Cargo.toml`
  versions or `CHANGELOG.md`.

## Rules for every phase

- **Repositories.** Mac code and the harness live in bob-mac-capture. Open it with
  `/sase_repo` and read its `AGENTS.md`. Read `tailnet.md` with `/sase_memory_read`
  before any SSH. The Mac is `ssh mac` (user `bbugyi`). Its login shell is zsh, so call
  `/usr/bin/log`, never bare `log`.
- **Production is off limits.** Never write to, kill, or relaunch any of these:
  - `~/Applications/Bob Mac Capture.app` or its process;
  - the `org.bobs.bob-mac-capture` defaults domain (reading it is fine);
  - `~/bob`;
  - `~/.local/state/bob-cli`;
  - Bryan's checkout.

  Never run `just install` or `just install-all`. Kill scratch processes only by the PID
  the harness recorded, never by name or bundle ID. Run `guard` before and after every
  check. If it fails, stop, restore production only to its recorded state, note what
  happened on the phase bead, and ask Bryan before anything else.

- **Evidence outlives the agent.**
  - After every `/sase_questions` answer, first write Bryan's verbatim observation as a
    note on your phase bead, with wake IDs and times.
  - Keep the harness's signpost stream running for the whole session. Quote its
    `snooze-*` lines with timestamps and the scratch PID in your notes.
  - The HUD is excluded from screen capture (`sharingType = .none`), so screenshots
    cannot prove it. Use signposts plus Bryan's observation.
- **Batch the human.** Each question round ends your turn. Group steps so that each
  round asks Bryan for one action and one observation set. When a check needs Bryan at
  the Mac, ask first whether he is at it now. The successor turn begins by running
  `status` and `guard`.
- **Fixes.**
  - Fix small defects found by a check in bob-mac-capture.
  - For any Swift change: run `just portable-check`, then compile a scratch Mac copy
    (rsync without `.build`/`.git`, `Scripts/bundle.sh`, `just typecheck-tests`, delete
    the copy).
  - Run `Scripts/macos-ci-check` through `/sase_monitor` before landing. After the push,
    watch the exact-SHA macOS CI to green with `/sase_monitor`.
  - Rebuild the harness from the new SHA and rerun the failing check.
  - A defect too large for the phase gets a `PROPOSED FOLLOW-UP:` note naming the
    failing check, and the phase stays open.
  - Anything not caused by this feature is a `PROPOSED FOLLOW-UP:`.
- **Docs.** Run `just docs-check` if any doc changes.
- **Closing.** Workers record follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and never create beads. Each worker closes only its own phase, after
  `sase bead epic-symbols` for it. No phase closes `bob-cli-q5.8.1.6.5` or any ancestor.

## Phase harness

1. **CI precondition.**
   - Read the rerun of run `45960329324` (`gh run view 45960329324 --attempt 2`). If it
     is still running, watch it with `/sase_monitor`.
   - If it is green, the precondition holds. Record the flaky test as a
     `PROPOSED FOLLOW-UP:`, citing any open task bead that already tracks
     `CaptureTaskGotoPanelTests` flakes.
   - If it fails the same way again, decide whether `ba213181` caused it. Rerun it, or
     compare against green `586d9ccb`. Fix it only if this feature caused it; otherwise
     record a follow-up.
   - Any newer master SHA must also be green before acceptance runs on it.
2. **`Scripts/snooze-accept.sh`** (`decisions` `harness_home`: committed by default). It
   is a bash script run on the Mac. Its root is `/tmp/bob-wake-accept`. Every subcommand
   prints a JSON line.
   - **`up`:**
     - Rsync the checkout at the exact master SHA, without `.build` and `.git`, to
       `<root>/src`. Run `Scripts/bundle.sh --identity -` there.
     - Copy the bundle to `<root>/Bob Mac Capture Accept.app`. With `PlistBuddy`, set
       `CFBundleIdentifier` to `org.bobs.bob-mac-capture.accept` and the bundle name and
       display name to `Bob Mac Capture Accept`. Re-sign ad hoc.
     - bob's `parse_live_claim` (bob-cli `src/native/snooze/notify.rs`) accepts any
       non-empty `app`, so the claim stays valid.
     - Seed only the `org.bobs.bob-mac-capture.accept` domain:
       - `bobDirectory=<root>/vault`;
       - `useProductionHotkey`, `captureAliasHotkeyEnabled`, `refsHotkeyEnabled`, and
         `refsAliasHotkeyEnabled` all false, so no production hotkey is ever taken.
     - Create a minimal disposable vault at `<root>/vault` with `bob.yml`, today's daily
       note, and `jobs.md`. Confirm `bob snooze -f json` against it reports
       `wake_notice_version: 1` and its own `vault_root`.
     - Record the production PID and its claim. Launch the scratch app with
       `XDG_STATE_HOME=<root>/state` and `BOB_DIR=<root>/vault`, for example with
       `open -n --env`, or by executing the binary under `nohup env` if `open --env` is
       unavailable. Record the scratch PID.
     - Start `/usr/bin/log stream --signpost --style compact` filtered to subsystem
       `org.bobs.bob-mac-capture` and the scratch PID. Write it to
       `<root>/evidence/signposts.log` under `nohup`, with a pid file.
   - **`guard`** asserts and reports:
     - The production PID is unchanged, and production's `notifier.json` still names it.
     - The production domain has no `bobDirectory`.
     - Exactly one scratch process is running.
     - The scratch `notifier.json` names the scratch PID.
     - The signpost stream is alive.
   - **`wake <slug> <title> [--channel auto|desktop|off]`:** add a task to the
     disposable vault, then wake it now through `bob snooze` with the scratch
     environment. Print `wake_id`, `notification_channel`, and `notified`. Read
     `bob snooze --help` for the exact verbs; earlier runs used an explicit
     `bob snooze wake <route:block>`. Selecting the channel per wake goes through the
     vault config, as the earlier run's `bob-desktop.yml`/`bob-off.yml` did.
   - **`ping`:** post the `org.bobs.bob.snooze.woken` Darwin notification
     (`notifyutil -p`).
   - **`restart`:** stop the scratch app by its PID, then relaunch it the way `up` did.
   - **`status`:** print PIDs, the claim, the latest receipt, and the last `snooze-*`
     signposts.
   - **`down`:**
     - Kill the scratch app and the stream by PID. Delete the `.accept` defaults domain.
     - Unregister the bundle (`lsregister -u`), then remove `<root>`.
     - Print anything left over, such as the Notification Center entry for
       `Bob Mac Capture Accept`, which Bryan may remove in System Settings.
   - Keep the script within the repo's shell and docs gates. Commit it, then push and
     watch CI as above.
3. **Smoke test and permission.**
   - Run `up`, then `guard`.
   - Ask Bryan, through `/sase_questions`, to:
     - grant notifications to Bob Mac Capture Accept, from its Settings permission
       button or System Settings;
     - confirm that his production capture hotkey still opens his production panel.
   - Then run one `wake` while he is active. Show from the signpost log that the scratch
     PID read the receipt, verified it, and presented it.
   - Record the harness SHA, the scratch and production PIDs, and the signpost lines on
     the phase bead. Leave the harness running for `delivery`. If the Mac must reboot,
     the next phase reruns `up`.

## Phase delivery

Use the harness. Run `status` and `guard` first, and rerun `up` if the scratch app is
gone.

1. **Check 1, active delivery.**
   - With Bryan active and unlocked, `wake` one task. Bryan observes exactly one glass
     HUD, one chime, one silent record attributed to Bob Mac Capture Accept, and menu
     beacon 1.
   - Then, without a new due task:
     - run `ping` twice;
     - repeat the same `bob snooze wake` for nothing due;
     - run `restart`.
   - Each must give no second HUD, chime, record, or announcement. Prove that with
     signposts (no second `snooze-present` for that wake ID) and with Bryan's
     observation.
   - The 06:33 observation is supporting evidence only. Redo the check on the isolated
     harness.
2. **Check 2, genuine lock and unlock.**
   - Ask Bryan to lock the screen (Ctrl+Cmd+Q) and answer once it is locked. Before
     triggering, confirm the lock on the Mac: `ioreg -n Root -d1 -a` shows
     `CGSSessionScreenIsLocked` under `IOConsoleUsers`.
   - Run `guard`, then `wake` one task while locked. Capture the hold signposts.
   - In the next round, ask Bryan:
     - what he saw or heard while locked;
     - which app any lock-screen banner named, since the distinct bundle name now tells
       an app record apart from bob's desktop fallback;
     - what appeared after he unlocked.
   - The contract: nothing while locked. After unlock, the surviving task appears after
     the 1.5 s settle and a fresh revalidation read: one HUD, one chime, one record.
   - If the HUD does not return after unlock, as the earlier inconclusive run suggested,
     debug the lock-return path in `SnoozeWakePresenceSource` and the coordinator from
     the signposts. Fix it in place if it is small.
   - Idle, display sleep, and session inactivity rely on the existing deterministic
     adapter tests. Name them in the evidence.
3. **Check 6, routing.**
   - **Stopped app.** Stop the scratch app with `restart`'s stop half. Confirm the
     scratch claim is gone or dead, then `wake` with `auto`. bob must route to
     `desktop`, and Bryan sees bob's desktop banner.
   - **Running app.** Run `up` or `restart`, then `wake` with explicit `desktop` and
     with `off`. The receipts carry those channels, and the scratch PID shows no
     `snooze-present`. For `desktop`, Bryan sees only bob's banner; for `off`, nothing.
   - **Old bob.**
     - Point the `.accept` domain's `bobExecutableOverride` at a shim script in the
       harness root. The shim forwards to the real bob but strips `wake_notice_version`
       from `bob snooze -f json`.
     - After `restart`, the scratch app publishes no claim, and an `auto` wake goes to
       desktop.
     - Clear the override and `restart` again. The claim returns and `auto` routes to
       `mac_capture`.
   - Cite note 3's CLI routing matrix for bob's side.

## Phase meetings-actions

Use the harness, with `status` and `guard` first.

1. **Check 3, meeting holds.**
   - In the disposable vault's daily note, write Event Tasks in bob's documented format
     (bob-cli `docs/reference/vault-format.md`; `sase memory read glossary:event-task`):
     - event A spans now to now+4 minutes and has a buffer;
     - event B overlaps it and ends at now+7 minutes.
   - Before any wake, confirm the agenda (`bob capture-pomodoros --tasks -f json` with
     the scratch environment) reports both events with their real `startsAt`/`endsAt`.
   - `wake` a batch of three tasks during event A. Nothing may appear while either
     meeting runs, and the hold must not end at A's end or at any buffered end.
   - During the hold, change one task each through the `bob` CLI: complete one, release
     one with `bob snooze drop`, and re-snooze one. After B's real end, only surviving
     tasks may appear. With all three changed, nothing appears and the batch is dropped.
     Run a second batch so that one task survives and appears.
   - Prove the hold window, the release time, and the filtering from signposts plus
     Bryan's observation.
2. **Check 4, HUD actions.** Ask Bryan before each step what he will click, then what he
   saw.
   - **Open.** Clicking Open, a batch title, or the HUD body must use the same validated
     route and block-ID URL as the record's default action
     (`ObsidianOpenURL.taskGotoURL`). Do not register the disposable vault in Bryan's
     Obsidian. Prove the URL from signposts or app diagnostics, and from the URL tests
     added in `hud-repair`. If Obsidian is already open on another vault, say what Bryan
     saw.
   - **+15m on a batch of two or three.** bob's list then shows each task re-snoozed to
     its new target, and the HUD headline states bob's returned target (for example
     `Snoozed until 15:47`).
   - **Partial or refused.** Before clicking +15m, complete one task of a batch through
     the CLI. The HUD keeps that task and shows bob's refusal words, and it never
     retries a task that already succeeded. Near-midnight `invalid_time` relies on the
     `hud-repair` tests; name them.
   - **Hover.** Bryan hovers over the HUD past 12 seconds and it stays. When he leaves,
     the countdown resumes, and it never restarts under the pointer.
   - **Focus.** Run `lsappinfo front` before the wake, after it, and after each click
     that should not activate the app. The frontmost app stays Bryan's original app, and
     the HUD never takes the key window. Escape handling follows
     `decisions:q-and-ctrl-bracket-mirror-esc`.

## Phase records-a11y

Use the harness, with `status` and `guard` first.

1. **Beacon and record clearing.**
   - After a wake, the menu beacon shows an amber alarm accessory with its count beside
     the template-tinted Bob glyph.
   - Opening or acknowledging the batch clears the beacon and withdraws the record.
     Prove the withdrawal from signposts and the Notification Center state.
   - Changing the `.accept` domain's `bobDirectory` to a second disposable vault, which
     reconfigures the app, clears that vault's beacon and records.
   - Bryan confirms what he saw, in light and dark menu bars if he will switch.
2. **Notification denial.**
   - Bryan turns notifications off for Bob Mac Capture Accept only.
   - A wake still shows the HUD and beacon, posts no record, and raises no permission
     prompt. The Wake notices diagnostics row reports the denial without overwriting the
     app-wide status.
   - Bryan turns notifications back on afterwards.
3. **Reduce Motion, Reduce Transparency, and VoiceOver.**
   - Ask Bryan whether he consents to toggling each one himself, briefly.
   - With Reduce Motion on, the alarm glyph does not ring. With Reduce Transparency on,
     the HUD stays legible.
   - With VoiceOver on, it speaks the wake announcement once, even though the panel is
     non-key and non-activating.
   - He restores each setting afterwards.
   - If he declines, cite the `hud-repair` tests and CI render fixtures: the run ID and
     artifact names on bead `bob-cli-q5.8.1.6.3`.
4. **Day rollover and 60-minute expiry.**
   - Day rollover relies on the deterministic tests from `records-repair` (name them).
   - For the expiry, `wake` one task and leave it unacknowledged. Use a `/sase_monitor`
     sleep of 61 minutes. Then show that the beacon and record cleared, using the
     `.accept` domain's persisted beacon state, the Notification Center, and a glance
     from Bryan.
5. **Teardown.**
   - Run `down`, then a final production check: the production PID or a later
     Bryan-started instance is running from `~/Applications`, the production domain has
     no `bobDirectory`, the production claim is live, and `~/bob` contains no test
     slugs.
   - Record what `down` reported as left over.

## Land agent

After every phase is closed, the land agent:

- confirms the harness is down and production is intact, as in the teardown above;
- writes one evidence note on `bob-cli-q5.8.1.6.5` listing, for each of checks 1 to 6:
  the bob-mac-capture SHA under test, its exact-SHA CI run ID, the wake IDs, the
  signpost lines, Bryan's recorded observations, and the deterministic tests relied on;
- carries over each phase's `PROPOSED FOLLOW-UP:` notes and the earlier ones on
  `bob-cli-q5.8.1.6.5`, for that bead's own land flow to triage;
- runs `sase bead epic-symbols bob-cli-q5.8.1.6.5` and resolves or re-keys any entries;
- closes `bob-cli-q5.8.1.6.5`.

It must not close `bob-cli-q5.8.1.6`, `bob-cli-q5.8.1`, `bob-cli-q5.8`, or `bob-cli-q5`.
Their own land agents resume from this evidence.
