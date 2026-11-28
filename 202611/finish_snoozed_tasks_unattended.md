---
tier: epic
title: Finish Snoozed Tasks without asking Bryan to verify anything
goal: 'bob-cli-q5.9 closes, leaving bob-cli-q5 ready for its waiting land agent. The
  stalled human-observation flow is retired, and the four q5-owned discovered issues
  are fixed. The Mac wake notice is proven on the real Mac from in-app machine evidence
  and a natural lock and unlock. The q5.8 subtree is landed. The glossary strand,
  guide, and live end-to-end acceptance ship. No question, prompt, or observation
  is asked of Bryan.

  '
parent_bead: bob-cli-q5.9
decisions:
  glossary_snoozed_task:
    ask: Add the Snoozed Task glossary strand that the bob-cli-q5 review already approved?
    memory:
    - glossary:snoozed-task
    default: true
    requested: To be clear I want you to completely take over the work for this epic.
    answer: true
phases:
- id: takeover
  title: Retire the stalled question flow and take ownership
  size: small
  depends_on: []
  description: 'takeover: stop the stale waiting agents and the pending nine-question
    round, record Bryan''s no-verification directive on the affected beads, then harvest
    and disarm the Mac lock driver.'
- id: bob-fixes
  title: Fix the q5-owned bob-cli contract and doc drift
  size: small
  depends_on: []
  description: 'bob-fixes: give deferred snooze reports their documented envelope,
    describe relative-time rounding as implemented, and correct the stale capture
    grammar revision claims.'
- id: plugin-resnooze
  title: Make the Obsidian planner replace struck history on re-snooze
  size: small
  depends_on: []
  description: 'plugin-resnooze: align the bob-navigation-hotkeys snooze planner with
    bob''s canonical re-snooze rule, flip the two tests that pinned the split, verify
    in real Obsidian, and deploy.'
- id: mac-probe
  title: Add an acceptance-only in-app probe and evidence-grade signposts
  size: medium
  depends_on: []
  description: 'mac-probe: build a probe that only the acceptance bundle can enable
    and that reads and drives the live wake HUD, beacon, and records. Add wake-ID
    and reason metadata to snooze signposts, the missing HUD tests, and harness subcommands,
    including the passive lock watcher.'
- id: mac-accept
  title: Accept checks 1 and 3 to 6 on the harness from machine evidence
  size: medium
  depends_on:
  - takeover
  - mac-probe
  description: 'mac-accept: rebuild the harness in place at the probe SHA. Run delivery,
    routing, meeting holds, HUD actions, records, denial, accessibility, and the 60-minute
    expiry. Use probe snapshots, signposts, and deterministic tests instead of Bryan''s
    eyes.'
- id: lock-accept
  title: Accept check 2 from a natural lock and Bryan's own unlock
  size: small
  depends_on:
  - mac-accept
  description: 'lock-accept: arm the lock watcher and wait passively, with an idle-gated
    display-sleep fallback, for a real lock and unlock. Prove the locked wake stayed
    silent and surfaced after unlock.'
- id: notice-land
  title: Tear down the harness and land the q5.8 subtree
  size: small
  depends_on:
  - lock-accept
  description: 'notice-land: tear down the harness and confirm production is intact.
    Write the per-check evidence and close the open q5.8 beads bottom-up, carrying
    their untriaged follow-ups forward.'
- id: closeout-docs
  title: Glossary strand, guide, and concepts paragraph
  size: medium
  depends_on:
  - bob-fixes
  description: 'closeout-docs: add the approved Snoozed Task glossary strand and regenerate
    the glossary page. Write the snooze how-to guide and the Later today concepts
    paragraph with nav entries, and pass the docs gate.'
- id: e2e
  title: Live end-to-end acceptance on bob and Obsidian
  size: medium
  depends_on:
  - bob-fixes
  - plugin-resnooze
  description: 'e2e: run every bob and Obsidian acceptance case against the current
    masters, on a snapshot of the real vault and in real Obsidian, and record commands,
    JSON excerpts, and screenshots.'
proposed_by: bbugyi200.athena.1hl
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 20:55:01
status: wip
bead_id: bob-cli-q5.9.1
---

- **PROMPT:** [prompts/202611/finish_snoozed_tasks_unattended.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/finish_snoozed_tasks_unattended.md)
- **BEAD:** [bob-cli-q5.9.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.9.1.md)

# Finish Snoozed Tasks without asking Bryan to verify anything

## Why this child epic exists

Bryan, 2026-11-27, about the snooze epic:

> Can you help me finish the bob-cli-q5 epic bead without requiring me to verify
> anything else? I feel like I've done enough work at this point. The last agent just
> asked me nine questions, which depended on me closely watching my MacBook screen for
> ten minutes. I have other things to do. To be clear I want you to completely take over
> the work for this epic. Don't half-ass it but don't ask me any more questions.

The directive overrides every earlier plan line that asks Bryan to act or observe. That
includes "a genuine lock/unlock observation is required" in
`plan:202611/mac_snooze_wake_notice.md`, the "Batch the human" rule in
`plan:202611/mac_wake_notice_acceptance.md`, and the screenshot and human glances in
`plan:202611/snoozed_tasks.md` close-out. Each requirement is still met. Only the
evidence changes, as described in [Evidence standard](#evidence-standard).

The work is attached to `bob-cli-q5.9`, the outer close-out phase, so that its land
closes q5.9 and the waiting `bob-cli-q5.land` lands q5 through the normal flow. Read
with `sase bead read`: `bob-cli-q5`, `bob-cli-q5.8.1.6.5`, `bob-cli-q5.8.1.6.5.1`,
`bob-cli-q5.8.1.6.5.1.1`, and `bob-cli-q5.8.1.6.5.1.2`. Read the four plans named above
and `plan:202611/mac_snooze_wake_notice_repair.md` with `sase artifact read`.

State at planning time (2026-11-27 ~20:15 EST):

- **Done.** q5.1–q5.7 are closed. All five q5.8.1 build phases and all four repair
  phases (q5.8.1.6.1–.4) are closed. Harness phase `bob-cli-q5.8.1.6.5.1.1` is closed.
  `Scripts/snooze-accept.sh` in bob-mac-capture is committed. The scratch app
  `org.bobs.bob-mac-capture.accept` runs from
  `~/Library/Application Support/bob-wake-accept/`, and Bryan granted it notifications.
- **Machine evidence already standing** (on `bob-cli-q5.8.1.6.5.1.2` note 7, harness
  `bbdb79e6`, CI `46067478956` green):
  - Check 1: one present for wake `18ec0e9d853c4840-0-60450`. Two pings, a nothing-due
    wake, a repeated explicit wake refused as `not_snoozed`, and a restart produced no
    second present.
  - Check 6: the full routing matrix (stopped, desktop, off, old bob, claim back).
- **Human evidence already standing:** Bryan saw one HUD, chime, and record for the
  06:33 wake `18ebe3a676f46ca8-0-38001`. The 17:12 smoke wake `18ec068e9413da48-0-65215`
  was presented and acknowledged.
- **Stalled.** The round-2 question (notification
  `fbf1167e-2f12-494f-971a-123148bc07bb`, request
  `d66607cd-82c3-48c3-ad38-3cf618c2e27f`) asks nine observation questions. Driver
  `/tmp/bob-wake-accept/delivery-c2.sh` (PID 82289) is armed on the Mac until about
  23:47 to wake a task on Bryan's next lock. Checks 2–5 have no evidence.
- **Waiting agents that would resume the old flow:** `bob-cli-q5.8.1.6.5.1.3`, `.4`,
  `bob-cli-q5.8.1.6.5.1.land`, `bob-cli-q5.8.1.6.land`, and `bob-cli-q5.9`. Each starts
  when a predecessor bead closes and follows a plan that asks Bryan to observe.
  `bob-cli-q5.land` also waits. It stays, because it lands q5 after q5.9 closes.
- **q5-owned discovered issues** (q5 notes 1–6):
  - relative-time rounding wording;
  - grammar revision doc drift;
  - the slim deferred wake report;
  - the JS planner keeping a struck history row on re-snooze, which leaves bob-plugins
    `npm test` red on a clean base.

## Evidence standard

Bryan's eyes are replaced by evidence the real app produces on the real Mac:

- **Probe snapshots.** A JSON dump of the live HUD, beacon, records, coordinator, and
  focus state, read inside the scratch app. The HUD has `sharingType = .none`, so
  screenshots cannot show it, and the probe is the only faithful view. The probe also
  renders the live HUD view to PNG.
- **Signposts.** These now carry wake ID, reason, and count metadata. Collect them with
  `/usr/bin/log show --signpost` per scratch PID. `log show` is persisted, while the
  harness stream misses launch-time events.
- **bob's own reports.** Receipts, `bob snooze -f json`, the agenda JSON, and argv logs
  from pass-through shims.
- **Deterministic tests,** named by class and method.
- **CI render fixtures.** Download the exact-SHA `render-fixtures` artifact and inspect
  the PNGs.

A lock counts as genuine when Bryan's own behaviour causes it, or when an idle-gated
display sleep does while he is away. The unlock must be his own. Simulated
`com.apple.screenIsLocked` posts never count. Earlier human observations remain
supporting evidence.

## Settled decisions and governing records

The bob-cli-q5 reviewer decisions stay final:

- `uncommitted_rows = ignore`
- `wake_notice = hud`
- `hold_in_meetings = yes`
- `glossary_snoozed_task = yes` (restated above for the closeout-docs worker)
- `decision_record = no`: never write `decisions:snoozed-tasks-wake-through-bob`

Child-epic decision `harness_home = commit` also stands.

Rulings this plan makes (Bryan must not be asked):

- **Re-snooze is canonical in bob.** `plan:202611/snoozed_tasks.md` rule "Re-snoozing a
  woken task replaces its struck row with a new active row." It agrees with bob-cli
  `docs/snooze.md` ("a matching history row is replaced"), the Rust `upsert_active_row`,
  and vector `snooze-add-resnooze-replaces-history`. The JS narrowing in bob-plugins
  `99458bd9` came from the agent-written `snooze_card_landing.md` line "leave history
  unchanged". That line concerns closed ledger history, not struck Snoozed rows.
- **Relative rounding is delta rounding.** The parser rounds the offset up to a multiple
  of five minutes and adds it to the current minute: 12:03 `+45m` → 12:48, `+2m` →
  12:08. The endorsed plan's own example agrees: a 15:32 wake plus `+15m` reads
  `Snoozed until 15:47`. Only the wording changes.
- **The deferred report gets the full envelope.** Make the output match the docs. Do not
  shrink the docs.
- **The tested bob is the master build.** The release decision says Bryan's hosts track
  master and releases may lag (v0.6.0 is from 2026-11-22), so e2e tests the exact master
  SHA. It records the installed `bob --version` and does not wait on crates.io.

Governing records:

- `decisions:mac-capture-is-a-thin-client`: the probe never parses the vault, computes
  previews, or writes it. Every vault change in a check goes through `bob` with the
  scratch environment. Only fixture setup writes the disposable vault directly.
- `decisions:idle-capture-shows-ledger-agenda` and
  `decisions:today-is-read-from-the-ledger`: meeting and Today facts come from bob's
  agenda and live-link facts.
- `decisions:event-buffers-widen-the-span`: a meeting hold ends at the real end, not the
  buffered end.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the HUD is never key and has no local key
  handling. Record that as consistent with this decision.
- `decisions:consumption-keeps-notes-and-recorded-time` and
  `decisions:task-lanes-are-sticky`: snooze writes keep notes and recorded time, and
  raise Ready to Next.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit bob-cli `Cargo.toml`
  versions or `CHANGELOG.md`. The fixes here are additive, so no `!` is needed.

## Rules for every phase

- **Never involve Bryan.**
  - No `/sase_questions`, and no gate that asks him to act or look.
  - No system permission prompt or modal on his screen, and no message telling him to
    watch, click, lock, or toggle anything.
  - If the Mac is unreachable, wait with `/sase_monitor` (an SSH probe loop with a
    generous timeout and `--next`), then resume. Never ask him to bring it online.
  - When something truly cannot be proven by machine, record a residual: the strongest
    machine evidence plus the deterministic tests that cover it. Then keep going.
- **Repositories.**
  - Open bob-mac-capture and bob-plugins with `/sase_repo` and read each `AGENTS.md`.
  - Read `tailnet.md` with `/sase_memory_read` before SSH. The Mac is `ssh mac` (user
    `bbugyi`) with a zsh login shell, so call `/usr/bin/log`.
  - bob-plugins changes deploy with `bob plugins sync` after landing.
- **Production is off limits.** Never write to, kill, or relaunch any of these:
  - `~/Applications/Bob Mac Capture.app` or its process;
  - the `org.bobs.bob-mac-capture` defaults domain (reading it is fine);
  - `~/bob` on any host, except the plugin deploy done by `bob plugins sync`;
  - the Mac's `~/.local/state/bob-cli`;
  - Bryan's checkouts.

  Never run `just install` or `just install-all`. Kill scratch processes only by PIDs
  the harness recorded, after confirming their command lines. Run `guard` before and
  after every check. Bryan's own `just install-all` restarts change the production PID.
  Re-adopt a new PID only under the harness rule: its claim names it, the production
  domain has no `bobDirectory`, and the adopt-grep finds 0 files.

- **Keep Bryan's notification grant.**
  - Keep bundle ID `org.bobs.bob-mac-capture.accept` and its app path.
  - Rebuild in place with the `rebuild` subcommand added in mac-probe. Never run `down`
    before notice-land.
  - If a snapshot shows authorization lost, do not re-request it. Record the residual
    and use the earlier record evidence plus tests.
- **Be considerate of Bryan's screen.** Test task titles end with "(test, no action
  needed)". Before each noisy batch (HUDs, chimes, desktop banners), wait up to 2 h for
  HID idle ≥ 300 s. If he stays active, proceed anyway. Checks other than lock-accept
  run with `snoozeWakeHoldWhileAway` off in the `.accept` domain, so presence cannot
  confound them. State that in the evidence.
- **Evidence outlives the agent.** After each check, write a note on your own phase bead
  with:
  - the app SHA under test and its exact-SHA CI run;
  - the Mac bob version;
  - wake IDs and times;
  - the signpost lines with metadata;
  - probe snapshot excerpts, with full JSON and PNGs attached per `sase_artifacts.md`
    (read it with `/sase_memory_read`);
  - the tests relied on.
- **Swift changes.**
  - Run `just portable-check`.
  - Compile a scratch Mac copy: rsync without `.build`/`.git`, run `Scripts/bundle.sh`
    and `just typecheck-tests`, then delete the copy.
  - Run `Scripts/macos-ci-check` through `/sase_monitor` before landing. After each
    push, watch the exact-SHA macOS CI to green with `/sase_monitor`.
  - A red run in a file the change did not touch gets one rerun. If it is green, record
    the flake as a `PROPOSED FOLLOW-UP:`.
- **bob-cli changes.** Run the targeted `cargo test` and `just check`, plus
  `just docs-check` when docs change. Identical clean-base failures (for example a
  pre-existing word budget or lint) get an evidence note and never strand a phase.
- **Closing.** Phase workers never create beads; they record `PROPOSED FOLLOW-UP:` notes
  on their own phase bead. Each worker closes its own phase after
  `sase bead epic-symbols`. Only takeover and notice-land touch beads outside this epic,
  and only as described below.

## Phase takeover

1. **Stop the old flow.** Run `sase agent list -a -j`. For each of
   `bob-cli-q5.8.1.6.5.1.3`, `bob-cli-q5.8.1.6.5.1.4`, `bob-cli-q5.8.1.6.5.1.land`,
   `bob-cli-q5.8.1.6.land`, and `bob-cli-q5.9`:
   - If it is still WAITING, `sase agent kill -n <name>`.
   - If one has started, read its live state first. Kill it unless it is mid-commit; in
     that case let the commit finish, then kill it.
   - Leave `bob-cli-q5.land` waiting, and confirm it still is.
2. **Retire the round-2 question.**
   - If response
     `~/.sase/interaction_requests/question/d66607cd-82c3-48c3-ad38-3cf618c2e27f/response.json`
     exists, Bryan answered. Copy his answers verbatim as a supporting-evidence note on
     `bob-cli-q5.8.1.6.5.1.2`.
   - Run `sase notify apply-state fbf1167e-2f12-494f-971a-123148bc07bb dismiss`. Confirm
     no successor for agent session `bob-cli-q5.8.1.6.5.1.2` was launched, and kill it
     if one was.
   - Dismiss any other undismissed question notification from a q5 agent the same way.
3. **Record ownership.** Add a note quoting Bryan's directive, naming this epic as the
   owner, and stating "no agent may ask Bryan to observe, verify, or act for q5" on:
   - `bob-cli-q5` (so `bob-cli-q5.land` sees it) and `bob-cli-q5.8`;
   - `bob-cli-q5.8.1.6.5.1.2`, `.3`, and `.4`;
   - `bob-cli-q5.8.1.6.5.1`, `bob-cli-q5.8.1.6`, and `bob-cli-q5.9`.
4. **Mac state** (wait for SSH if it is offline).
   - Run `bash /tmp/bob-wake-accept/src/Scripts/snooze-accept.sh status` and `guard`.
   - Read `/tmp/bob-wake-accept/evidence/delivery-c2.log`. If it recorded a lock, a
     wake, and an unlock, quote it with `log show --signpost` lines for its scratch PID
     as supporting check-2 evidence at `bbdb79e6`.
   - If PID 82289 is alive and `ps -o command= -p 82289` names `delivery-c2.sh`, kill
     it.
   - Leave the scratch app running; mac-accept rebuilds it.
   - Record PIDs, the claim, and `defaults read org.bobs.bob-mac-capture.accept`.

## Phase bob-fixes

bob-cli only.

1. **Deferred envelope** (q5 note 3).
   - `deferred_action` in `src/native/snooze/cli.rs` (about line 1356) returns a slim
     object for add, drop, and wake. A deferred wake must carry every key
     `docs/snooze.md` promises for wake reports: `status`, `woken`, `dropped`,
     `ignored`, `pending`, `next_wake_at`, `plan`, `daily_file`, `notified`, and
     `wake_id`.
   - Use empty arrays and nulls, and the values already read before deferring. Never
     invent a plan block that needs the write lock; emit null and document it.
   - Deferred add and drop carry their own verbs' documented keys and no wake-only keys.
     Split the helper per verb if needed.
   - Add tests for each verb's deferred shape. Feed the new deferred JSON to
     bob-plugins' `snoozeWakeDecodeReport` (read-only node check) and confirm the Mac's
     `CaptureSnoozeWakeResponse` keys still decode.
   - Update `docs/snooze.md` and `docs/reference/json-outputs.md`.
2. **Rounding wording** (q5 note 1).
   - Reword every description of relative rounding to the delta rule with the 12:03
     examples. Cover `docs/snooze.md` (about line 71), `docs/capture.md` (about line
     685), the `src/native/snooze/when.rs` module doc, and
     `docs/reference/cli/snooze.md` if it mentions rounding.
   - Grep bob-plugins and bob-mac-capture docs for the same claim and fix them in this
     phase, committing those repos too.
3. **Grammar drift** (q5 note 2; `GRAMMAR_REVISION` is now 34).
   - In `docs/reference/compatibility.md` (about line 71), make the current-revision
     claim 34. Add Grammar 33 (terminal `z:<when>` snooze marker) and Grammar 34
     sentences to its history.
   - Add the missing Grammar 33 sentence to the `docs/capture.md` history paragraph
     (about line 7668).
4. Note each fixed q5 note number and its commit on this phase bead, for the land.

## Phase plugin-resnooze

bob-plugins, with bob-cli as the reference.

1. In `plugins/bob-navigation-hotkeys/src/738-snooze-mutation-planner.js`,
   `snoozeMutationUpsertRow` (about lines 312–360), mirror bob-cli `upsert_active_row`
   (`src/native/snooze/plan.rs`, about lines 881–941):
   - When no active row matches the identity, remove the last matching `history` row and
     its subtree. Do not carry its children as notes.
   - When active rows exist, retime them and keep history.
   - Replace the comment at about lines 323–324.
2. Flip both tests that pinned the split:
   - `scripts/test-navigation-snooze-mutation-planner.cjs` (about line 151);
   - check `history-is-preserved-on-resnooze-and-closed-history-snooze` in
     `scripts/obsidian-snooze-card-e2e.cjs` (about line 767). Snoozing from a closed
     ledger history link must still leave ledger history untouched; only the struck
     Snoozed row is replaced.
3. Align any bob-plugins doc that says struck rows survive re-snooze, for example
   `docs/task-card.md` and `docs/bob-navigation-hotkeys.md`.
4. Verify:
   - `npm run build`;
   - `node --test scripts/test-snooze-model-vectors.cjs` (21/21) and the focused snooze
     tests;
   - full `npm test` (record known flakes and ENOSPC baselines);
   - a real-Obsidian run of `scripts/obsidian-snooze-card-e2e.cjs` under `xvfb-run -a`
     on athena, inspecting its screenshots and attaching the report.
5. Land, then run `bob plugins sync`. Note q5 notes 4–6 as fixed, with the commit.

## Phase mac-probe

bob-mac-capture. No product behaviour changes for the production bundle.

1. **Gate.** Add `SnoozeWakeAcceptanceProbe`. It is enabled only when
   `Bundle.main.bundleIdentifier` starts with `org.bobs.bob-mac-capture.accept` and that
   domain has `snoozeAcceptanceProbe = true`. Unit-test that the production identity can
   never enable it. A runtime gate keeps the code compiled and tested in CI; a
   compile-flag build would rot.
2. **Transport.**
   - The harness atomically writes
     `$XDG_STATE_HOME/bob-mac-capture-accept/probe/request.json` (`seq`, `command`,
     `args`), then posts Darwin `org.bobs.bob-mac-capture.accept.probe`.
   - The app runs the request on the main actor and atomically writes
     `response-<seq>.json` (`seq`, `ok`, `error`, `result`). Bound the file sizes.
3. **Commands.**
   - `snapshot [png]` reports:
     - **HUD:** visible, on-active-space, occlusion, frame and the screen under the
       pointer, level, nonactivating style, `canBecomeKey`, isKey, `sharingType`,
       headline, row texts, button titles and AX labels from the live AX tree, countdown
       remaining, hovered, reduce-motion and opaque-fallback state, and the Open URL
       each control would use.
     - **Beacon:** count, accessory visible, tooltip, and AX label.
     - **Records:** authorization status, the delivered `snooze-woken-*` identifiers
       with their default-action URLs from `getDeliveredNotifications`, and the
       `requestAuthorization` call count.
     - **Coordinator:** state, hold reasons, queue, and handled IDs for today.
     - **Last events:** the last announcement posted (text and time) and the last chime
       (sound and `play()` result).
     - **Focus:** `NSApp.isActive` and the frontmost app.
     - **Agenda:** the Later-today band rows from the app's agenda store.

     `png` renders the live HUD content view through the `RenderFixtureWriter` path.

   - `press <open|snooze15|dismiss|body|row:N>` finds the control by a new
     `accessibilityIdentifier` and calls `accessibilityPerformPress()`, so the real
     SwiftUI binding runs. Fall back to the controller handler only if that fails, and
     report which path ran.
   - `hover <enter|exit>` calls the same handler `.onHover` uses.
   - `set <snoozeWakeAnnounce|snoozeWakeChime|snoozeWakeHoldWhileAway|snoozeWakeHoldDuringMeetings|bobDirectory> <value>`
     goes through the `AppSettings` setters, so the real reconfigure path runs.
   - `override <reduce-motion|reduce-transparency> <on|off|system>` injects the HUD's
     providers. Make Reduce Transparency injectable; it is read directly today.
   - `open-sink <on|off>` defaults on in probe mode. Open then records the URL instead
     of launching Obsidian on a vault it does not know.
   - `panel <show|hide>` uses the real capture-panel paths.

4. **Signposts.** Give every `snooze-*` event `%{public}s` metadata: wake ID, hold or
   drop reason, task count, and channel, with no task titles. Give the record post its
   own `snooze-record` event, so one presentation is one `snooze-present`. Update the
   metadata promise in `Signposts.swift` and `docs/diagnostics.md`.
5. **Missing tests** (`Tests/BobMacCaptureTests`), with injected clocks:
   - HUD hover pauses the 12 s countdown and resumes without restarting;
   - partial and refused +15m feedback keeps the refused task with bob's words and never
     retries a success;
   - one VoiceOver announcement per new or ordered-in wake, none on a silent restore;
   - opaque fallback under injected Reduce Transparency;
   - the diagnostics row under `.denied` and `.notDetermined`;
   - probe gating and response shape.
6. **Harness** (`Scripts/snooze-accept.sh`, shellcheck clean, inside
   `just check-scripts`):
   - `up` seeds `snoozeAcceptanceProbe`.
   - `rebuild --sha S` stops the scratch app by PID and rebuilds and re-signs the bundle
     at the same path and bundle ID. It keeps the domain and LaunchServices
     registration, relaunches, and waits for the claim.
   - `probe …` prints the response JSON.
   - `idle` reports HID idle seconds, `CGSSessionScreenIsLocked`, display sleep, and
     display-sleep-preventing `pmset` assertions.
   - `evidence --since T` saves `log show --signpost` for the scratch PID.
   - `shim <passthrough|oldbob|off>` sets an argv-logging `bobExecutableOverride`. The
     old-bob mode strips `wake_notice_version` from `snooze -f json`.
   - `variant noauth` brings up `org.bobs.bob-mac-capture.accept-noauth`, never
     authorized, as the only scratch instance.
   - `lockwatch` implements the lock-accept driver.
   - `wake` ensures today's disposable daily note exists first, so midnight cannot break
     it.
7. Document the probe, harness-only, in `docs/diagnostics.md`. Run the docs gate, land,
   and get exact-SHA CI green.

## Phase mac-accept

Run `status` and `guard`. Stop any leftover driver by recorded PID. Run `rebuild --sha`
at the green mac-probe SHA, then `probe snapshot` to confirm `ok` and that authorization
is still granted. Set hold-while-away off and the other three settings on. Run the old
`/tmp` driver logic as committed `check <n>` subcommands or a sibling script; commit it
at the end, which needs no app rebuild.

1. **Check 1, delivery.**
   - `wake` one task. Within 3 s, snapshot. Expect:
     - HUD visible, top-center on the pointer's screen, nonactivating, not key;
     - row `Ready now · <title>` and the `woke HH:MM · in <TYPE> · TODAY n/m` line,
       matching bob's link budget;
     - chime `play()` true, one announcement, one record `snooze-woken-<wake_id>`, and
       beacon 1;
     - frontmost app unchanged (`lsappinfo front` before and after).
   - Then run two pings, a nothing-due wake, a repeated explicit wake, and a plain
     `restart` (no rebuild). Snapshots and signposts must show no second present, chime,
     record, or announcement, and the beacon restored to 1.
   - Snapshot the PNG.
2. **Check 6, routing.**
   - Rerun the stopped, desktop, off, old-bob, and claim-back matrix at this SHA with
     probe snapshots.
   - For the desktop routes, prove bob's fallback ran through a PATH shim that logs and
     forwards `osascript` argv.
   - For off, prove nothing ran. Cite the earlier matrix as corroboration.
3. **Check 3, meetings.**
   - Write Event Tasks per bob-cli `docs/reference/vault-format.md` and
     `glossary:event-task`. Event A runs from now to now+4 min with a buffer. Event B
     overlaps it and ends at now+7 min.
   - Confirm the agenda JSON gives both real `startsAt`/`endsAt`.
   - Wake three tasks during A. Snapshot every 15 s: the HUD is hidden, the hold reason
     is meeting, and the hold survives A's real and buffered ends.
   - During the hold, through `bob`: complete one task, `bob snooze drop` one, and
     re-snooze one. After B's real end plus the settle, nothing appears and the batch
     drops with that reason.
   - Repeat with one survivor, which alone appears.
4. **Check 4, actions.**
   - **Open:** `press open`. The sink URL equals `ObsidianOpenURL.taskGotoURL` with the
     block ID and equals the record's default-action URL. The beacon clears and the
     record is withdrawn.
   - **+15m on three tasks:** `press snooze15` (shim passthrough). bob's list shows each
     task at bob's target, and the HUD headline states bob's returned target before
     closing.
   - **Partial:** complete one of two tasks through `bob`, then press +15m. The HUD
     keeps that task with bob's refusal words, and the shim log shows no retry of the
     success. Cite the existing near-midnight `invalid_time` tests.
   - **Hover:** `hover enter` at 2 s. After 20 s it is still visible. On `hover exit`,
     the countdown resumes from the paused value and the HUD closes about 10 s later.
   - **Focus and Escape:** frontmost and `NSApp.isActive` never change, the HUD is never
     key, and no key handling exists (see the Escape record above).
5. **Check 5, records and accessibility.**
   - **Clearing:** `×` and timeout end only the nudge, so the beacon and record stay.
     Then clear them through:
     - Open;
     - `panel show` then `panel hide`, only while HID idle ≥ 300 s, waiting up to 6 h
       before falling back to naming the coordinator's panel-visible acknowledgment
       test;
     - `set bobDirectory` to a second disposable vault (reconfigure).

     Each clears the beacon and withdraws the records. Restore the first vault.

   - **Denial:** with `variant noauth` as the only scratch instance, a wake shows the
     HUD and beacon with no record and zero `requestAuthorization` calls. The
     diagnostics row reports it. Restore the main scratch afterwards and confirm its
     authorization.
   - **Reduce Motion and Transparency:** with the overrides on, the alarm does not
     animate, the opaque fallback is active, and the PNG is legible. Then restore
     `system`. Inspect the exact-SHA `render-fixtures` PNGs:
     - `snooze-wake-hud-*-{light,dark}.png`, including `opaque-fallback`;
     - `status-item-glyph-ready-beacon-2-*`, for light and dark menu bars.
   - **VoiceOver:** the snapshot shows exactly one announcement with the expected text.
     Cite the new announcement test. Never turn VoiceOver on.
   - **Day rollover:** cite `testDayRolloverClearsDeliveredBeacon`,
     `testDayPurgeClearsPending`, and `testMidnightDuringWriteDiscardsStaleOutcome`.
   - **60-minute expiry:** wake one task and leave it, sleep 61 min with
     `/sase_monitor`, then snapshot. The beacon is 0, the record is withdrawn, and
     `snoozeWakeBeaconMetadata` is cleared.

6. **Mac band.** For outer acceptance case 1, show a snoozed task in the snapshot's
   Later-today rows before its wake and absent after it.

## Phase lock-accept

Hold-while-away and hold-during-meetings are on. Run `status` and `guard`, then start
`lockwatch` under `nohup` with a pid file. Watch its result lines with `/sase_monitor`
(re-arm with `--next` until it passes).

`lockwatch` loop:

1. Wait for `CGSSessionScreenIsLocked` to stay true for 10 s.
2. Run `guard`, then `wake "Lock check N (test, no action needed)"` with channel auto.
3. While locked, snapshot every 60 s. Expect: HUD hidden, hold reason locked, no chime,
   no new record, no announcement.
4. When the batch nears the 4 h receipt age limit or midnight, let it expire or drop and
   record that as bonus expiry or rollover evidence. Then wake a fresh one, so exactly
   one live held batch exists.
5. On unlock, snapshot every 0.5 s for 30 s. Pass when, after the 1.5 s settle and a
   fresh verify read, exactly one present, chime, announcement, and record appear for
   the live wake ID.
6. Retry when the unlock comes within 10 s of the lock.

**Accelerator.** When HID idle ≥ 30 min, the screen is unlocked, the display is awake,
and no `PreventUserIdleDisplaySleep`/`NoDisplaySleep` assertion is held, run
`pmset displaysleepnow` at most once per 6 h. Count the run only if the lock then
engages. If only the display sleeps, record it as display-sleep hold evidence. Never
lock any other way.

On pass, quote the lock and unlock timestamps from `ioreg` polling, the signposts with
metadata, and the snapshots. Name the adapter tests:

- `SnoozeWakePresenceSourceTests`: `testIdleThresholdBoundary`,
  `testLaunchingWithDisplayAsleepHolds`, `testLaunchingWithInactiveSessionHolds`,
  `testScreenLockHoldsUntilUnlock`,
  `testDisplaySleepAndSessionSignalsHoldIndependently`,
  `testIdlePollingRunsOnlyWhileHeld`;
- `SnoozeWakeNoticePolicyTests.testEachAwaySignalHoldsIndependently`.

Then stop `lockwatch` by its PID.

## Phase notice-land

1. **Teardown.**
   - Run `down` for every scratch identity, then the final production check: production
     runs from `~/Applications`, its domain has no `bobDirectory`, its claim is live,
     and `~/bob` has no harness slugs.
   - Record the leftovers `down` prints, such as Notification Center entries for the
     Accept apps.
2. **Close the subtree.** Post one evidence note on each of `bob-cli-q5.8.1.6.5.1.2`,
   `.3`, and `.4`. Each note covers its checks (`.2`: checks 1, 2, and 6; `.3`: checks 3
   and 4; `.4`: check 5), states that Bryan's directive replaced observation, and cites
   this epic's phase beads.
   - Then close them in order.
   - Next, close `bob-cli-q5.8.1.6.5.1`, then `bob-cli-q5.8.1.6.5`. Its consolidated
     note covers checks 1–6 in the format of `plan:202611/mac_wake_notice_acceptance.md`
     "Land agent", with probe evidence replacing observations.
   - Then close `bob-cli-q5.8.1.6`, `bob-cli-q5.8.1`, and `bob-cli-q5.8`, the last with
     q5.8's land evidence: commits, CI, docs, and real-Mac acceptance.
   - Before each close, run `sase bead epic-symbols <bead>` and resolve or re-key its
     entries. Confirm every child is closed.
3. **Carry follow-ups.** Collect every untriaged `PROPOSED FOLLOW-UP:` from that subtree
   onto this phase bead, each with its source bead and note number, for the land agent.

## Phase closeout-docs

bob-cli plus memory, in line with the `glossary_snoozed_task` decision.

1. **Glossary strand.** Use `/sase_memory_write` to add
   `sase/memory/glossary/snoozed-task.md` with the shape and claims from the close-out
   phase of `plan:202611/snoozed_tasks.md`. Polish it to what shipped:
   - re-snooze replaces the struck row;
   - the notice is the Mac HUD, else bob's desktop fallback, else the Obsidian Notice;
   - wakes are held while away or in meetings.

   Use the `uncommitted_rows = ignore` wording. Then run `sase memory init` and
   `just docs-gen` to regenerate `docs/reference/glossary.md`. Edit no other memory.

2. **Guide.** Write `docs/guides/snooze-a-task-until-later.md` covering:
   - the goal, Alt+Z and Task Card `z`, capture `z:<when>`, and `bob snooze add`, with
     the delta-rounding examples;
   - what you see asleep, due, and woken, and the notice;
   - when a wake seems late: the Mac asleep or lid shut, `bob tick` cadence, a deferred
     report, an unset timezone skipping automatic sweeps (`7213dd89`), a hold while away
     or in a meeting, and checking with `bob snooze` and `bob snooze wake`.
3. **Concepts.** Add one "Later today" paragraph to
   `docs/concepts/today-and-the-pomodoro-ledger.md`.
4. **Nav.** Add entries to `mkdocs.yml` and `docs/guides/index.md`, then run
   `just docs-check`.

## Phase e2e

Use bob built from the exact bob-cli master SHA that holds bob-fixes, in a scratch
target dir; record athena's installed `bob --version` as well. Use bob-plugins master
after plugin-resnooze.

- **Vault snapshot.** Snapshot the real vault:
  `rsync -a --exclude .git --exclude '.obsidian/workspace*' ~/bob/ <scratch>/vault/`.
  Run with scratch `BOB_DIR`, `XDG_STATE_HOME`, and `XDG_CONFIG_HOME`, and
  `snooze.notify: off`, so athena shows no banners and `~/bob` is never written.
- **bob cases.** Run outer acceptance cases 1–7 live on the snapshot, with targets a few
  minutes ahead of now.
  - Case 1: absent from TODAY, the `=`/`x` lineups, `]s`/`]w`, `bob walks`, the dash
    sections, and `links.count`; present in `bob plan`, `bob snooze`, and the agenda
    JSON.
  - Case 2: one move, the struck row, a no-op second run, and a late run catching an
    early row.
  - Case 3: running-entry placement.
  - Case 4: every leaving gesture, the dependency park and recovery, and over-cap.
  - Case 5: Event refusals, with byte-identical gcal, MEET, Gmail, successor, close, and
    mention outputs.
  - Case 6: rows that do not wake, and cache and preference deletion.

  Where real time cannot reach a case (DST and midnight in case 7), cite the Rust tests
  by name.

- **Obsidian.** Run `scripts/obsidian-snooze-marks-e2e.cjs` and
  `scripts/obsidian-snooze-card-e2e.cjs` for real under `xvfb-run -a` on athena. Inspect
  every screenshot: ⏰ marks, the woken pill, row paint, the plan chip, and the card.
  For case 8 on Linux, prove the ledger-tools waker spawns `bob snooze wake` on a due
  row and shows its Notice when `notified` is `none`. If no runner covers that, add the
  check to the marks runner and its offline self-test.
- **Mac.** The Mac half of cases 1 and 8 comes from mac-accept and lock-accept; cite
  them.
- **Defects.** Fix small q5 defects in place, re-running the affected checks. Record
  anything larger, or not caused by q5, as `PROPOSED FOLLOW-UP:`. Attach the reports and
  screenshots.

## Land agent

After every phase closes, the land agent:

1. confirms `bob-cli-q5.8` is closed, and that production on the Mac is intact as
   notice-land recorded;
2. triages every `PROPOSED FOLLOW-UP:` from this epic's phases and notice-land's carried
   list with `/sase_new_task`, filing, corroborating, or declining each with a reason;
3. notes on `bob-cli-q5` how each DISCOVERED ISSUE note 1–6 was resolved (phase,
   commit), so `bob-cli-q5.land` need not re-triage;
4. runs `sase bead epic-symbols` for this epic and `bob-cli-q5.9`, closes this epic,
   then closes `bob-cli-q5.9` with its evidence (strand, guide, concepts, e2e);
5. leaves `bob-cli-q5` to `bob-cli-q5.land`, confirming that agent still waits. If it is
   gone, rerun `sase bead work bob-cli-q5`; every other q5 bead is closed by then.

Never ask Bryan anything. The final summary may tell him, as information only, which
harmless leftovers he can remove whenever he likes, such as the Accept apps' entries in
System Settings › Notifications.
