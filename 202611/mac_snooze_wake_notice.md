---
tier: epic
title: Presence-aware Mac snooze wake notices
goal: Complete bob-cli-q5.8 with one verified Mac wake HUD per owned batch, held during
  absence or meetings, actionable through Open and +15m, and backed by an automatically
  cleared beacon and silent Notification Center record.
phases:
- id: notice-protocol
  title: Make wake notification ownership and revalidation explicit
  size: medium
  depends_on: []
  description: 'notice-protocol: add the minimal additive bob-cli receipt and list
    fields needed to select one notification channel, negotiate compatibility, and
    verify that a receipt still refers to actionable woken tasks. Cover the channel
    matrix and stale history cases, and document the schema-1 additions.'
- id: notice-core
  title: Decode wake receipts and model notice delivery
  size: medium
  depends_on:
  - notice-protocol
  description: 'notice-core: implement portable receipt, list, and add-response models,
    bounded receipt reading, independent subprocess lanes, and a deterministic presence,
    meeting, revalidation, and expiry policy with clock-controlled tests.'
- id: notice-runtime
  title: Observe receipts and hold notices until return
  size: medium
  depends_on:
  - notice-core
  description: 'notice-runtime: implement the Darwin observer, atomic notifier claim,
    persistent wake-id deduplication, macOS presence adapter, settings storage, and
    app-lifetime delivery coordinator behind injected presentation callbacks. Claim
    only when a compatible client and live presenter are attached.'
- id: notice-hud
  title: Present and act on the snoozed-task wake HUD
  size: medium
  depends_on:
  - notice-runtime
  description: 'notice-hud: extract shared non-activating glass-panel infrastructure
    without changing existing Refs toasts, implement the accessible wake HUD and truthful
    Open/+15m actions, and wire the coordinator into AppDelegate.'
- id: notice-records
  title: Add the wake beacon, silent record, and settings
  size: medium
  depends_on:
  - notice-hud
  description: 'notice-records: complete the status-item beacon and expiry, passive
    Notification Center records and withdrawal, live Settings controls, diagnostics,
    and Mac docs. Verify their shared acknowledgement lifecycle and capture integrated
    synthetic render and behavioral evidence for the land agent.'
proposed_by: bbugyi200.athena.bob-cli-q5.8
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.8
create_time: 2026-11-26 12:54:19
status: wip
bead_id: bob-cli-q5.8.1
---

- **PROMPT:** [prompts/202611/mac_snooze_wake_notice.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_snooze_wake_notice.md)
- **PARENT:** [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:** [bob-cli-q5.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.8.1.md)

# Presence-aware Mac snooze wake notices

## Scope and ownership

This child epic implements **only `bob-cli-q5.8`**, the `mac-wake-notice` phase of
`plan:202611/snoozed_tasks.md`. Read both with the audited SASE commands before
implementation. The assigned phase is already in progress; never open it or set its
status manually. Its read on 2026-11-26 showed no earlier implementation notes or child
plans. `sase bead epic-symbols bob-cli-q5.8` found no allowances at planning time; check
again before closure.

Prerequisite `bob-cli-q5.7` is closed. Its child epic landed the Later-today band, woken
accents, and always-on scheduler in bob-mac-capture through `46c5202b` (including
`c4a30591`, `e387187f`, and `a4a7020d`). Reuse these. The current
`CaptureSnoozeWakeResponse` deliberately decodes only status and next-wake time; it is
not a notification receipt. No wake-notice observer, claim writer, presence policy, or
snooze HUD exists in that tree.

This is an epic because it crosses one missing Rust protocol boundary, portable Swift
policy, app lifecycle/process orchestration, and several AppKit surfaces. Each phase is
a bounded direct-implementation unit. The dependency chain is intentionally serial:
several phases touch the process client, AppDelegate, and notification lifecycle. No
phase should repeat the agenda implementation or absorb another phase's UI work.

Open bob-mac-capture with `/sase_repo` and use only the path it prints. Read its
`AGENTS.md`. Paths below are relative to the explicitly named repository. No memory
changes belong in this child epic. The outer close-out phase owns the authorized
glossary work; the decision record was rejected.

## Settled decisions and rules

The parent review is final: `wake_notice = hud`, `hold_in_meetings = yes`,
`uncommitted_rows = ignore`, `glossary_snoozed_task = yes` (outer phase only), and
`decision_record = no`. Do not re-ask them or implement the banner branch.

Governing records:

- `decisions:mac-capture-is-a-thin-client`: bob owns task truth and all vault mutations.
  Swift consumes JSON and owns presentation, presence, and process orchestration. Schema
  additions remain optional to older clients.
- `decisions:idle-capture-shows-ledger-agenda`: use the existing agenda store and its
  refresh path; do not parse vault Markdown or create a second agenda.
- `decisions:today-is-read-from-the-ledger`: the durable woken row alone does not prove
  that its task is still Today. Use bob's resolved live-link facts.
- `decisions:event-buffers-widen-the-span`: meeting holds use the actual
  `startsAt <= now < endsAt` meeting span, not travel/preparation buffers.
- `decisions:q-and-ctrl-bracket-mirror-esc`: any local HUD Escape handling must share
  its dispatch with q and Ctrl+[. Do not install global key capture or steal the key
  window to give a passive HUD keyboard shortcuts.
- `decisions:bob-cli-releases-ride-a-daily-train`: leave Cargo versions and CHANGELOG
  untouched; bob ships via the train. These are additive contracts.

## Result and delivery contract

A committed wake owned by Mac Capture produces one nudge per wake ID. The non-activating
glass HUD appears top-center on the display under the pointer, never focuses the app,
and is excluded from screen capture. One task reads
`Ready now · Email Dan with interview updates`, with
`woke 15:32 · in WRITE · TODAY 11/13` below. A batch reads `3 snoozed tasks are ready`,
shows up to three task titles, and names any remaining count. Over-cap text is amber.
Color is supplementary to text.

It offers body/Open, +15m, and ×, rings the alarm glyph once unless Reduce Motion is
enabled, announces through VoiceOver, and plays one soft system chime when enabled. It
dismisses after **12 seconds of unhovered display**. Reduce Transparency gets an opaque
readable fallback. Existing Refs toasts retain their current mouse pass-through,
placement, and lifetime.

Defaults are Announce wakes on, Chime on, Hold while away on, and Hold during meetings
on. Away means screen locked, displays asleep, user session inactive, or idle for **at
least 180 seconds**; ordinary app deactivation is not away. When a hold clears, wait
**1.5 seconds**, re-read bob, and check presence again before delivery. A new hold
invalidates that timer/read. A meeting hold ends only when no qualifying meeting is
running, including overlapping meetings. Meeting decisions recompute timestamps against
now, never trust an old `phase: running` label. Ignore cancelled/non-meeting agenda
rows.

Receipts must be schema 1, belong to today and the configured vault, carry a valid
timestamp, and be at most **four hours old**. Reject future-dated or invalid receipts
rather than replaying them after a clock change. Held work expires when that age limit
or day boundary is reached. Re-check the day on all async completions. A settings
change, executable/vault replacement, or shutdown invalidates obsolete callbacks.
Persist terminally handled wake IDs in a bounded day-scoped UserDefaults collection; a
held batch is not marked delivered merely because the observer read it. Distinct batches
are queued without replacing a held predecessor; duplicate IDs never replay.

The menu item carries an amber alarm and count of unacknowledged delivered tasks. Its
passive Notification Center records share the same lifecycle. Opening the capture panel,
Open from the HUD or record, 60 minutes after delivery, and day rollover clear the
associated beacon and withdraw records. The HUD's ×/timeout only ends the transient
nudge. A successful +15m removes those tasks from the notice/record state. If the
capture panel is already visible when delivery occurs, treat the beacon/record as
acknowledged; the notice itself still follows the selected announcement setting.

## Protocol gaps confirmed in the current implementation

1. `src/native/snooze/notify.rs::deliver` writes/posts the receipt **before** choosing
   `auto | desktop | off`, and always posts Darwin after a readable receipt. The receipt
   lacks channel ownership. Observing every ping would duplicate desktop notices and
   override `off`.
2. `state.rs::base_state` returns History before resolving current task status.
   `cli.rs::row_json` maps it to `state: "woken"` with no open-task proof. A completed,
   released, or missing target can still look woken.
3. Receipts use one host state path and a vault-relative `daily_file`. Revalidation must
   distinguish configured vaults, not just coincidentally equal route/block IDs.

Resolve these as supporting work inside this epic, rather than weakening the parent's
one-channel and current-task contracts. Do not change snooze row classification,
placement, wake mutation semantics, or release rules.

## Phase notice-protocol

**Repository: bob-cli.** Limit edits to snooze reporting/notification code, focused
tests, and its protocol docs.

1. Select a notification route once per committed wake, before writing its receipt:
   `mac_capture` when auto finds a live claim, `desktop` for explicit desktop or auto
   without a claim, and `off` otherwise. Store the selection as additive receipt field
   `notification_channel`; use that same selection for subsequent nudge dispatch. It is
   intended ownership, not proof of a successful OS delivery. Keep command `notified`
   truthful to actual dispatch. Receipt failure still leaves wake committed,
   `wake_id: null`, and no ping. Desktop failure still yields `notified: none`; do not
   reroute to the app.
2. Add canonical absolute `vault_root` to receipt and list responses, using bob's
   effective vault root. Keep `daily_file` vault-relative. Add `wake_notice_version: 1`
   to `bob snooze -f json` as the feature probe for these ownership/revalidation fields.
   Do not bump `schema_version` or add commands/flags. Only a client that sees this
   version may publish a claim.
3. Add row boolean `wake_notice_eligible` to the list response. It is true only for
   today's valid woken history whose identity resolves uniquely to an ordinary committed
   open task, is not effectively Blocked, and still has a live Task Link under today's
   open Pomodoros, with no replacement active snooze row for that identity. Use existing
   resolver, normalized task facts, and live-link machinery; never infer eligibility
   from a title or merely from History. Every other row reports false. Keep `state` and
   history display unchanged. The Mac additionally matches receipt identity, target, and
   actual wake time so a later snooze/wake cannot revive old copy.
4. Update `docs/snooze.md` and the relevant JSON reference. Document the ownership
   field, capability probe, canonical vault identity, and the read-only eligibility
   meaning. No generated CLI help changes are needed unless an existing documented
   output table is generated.

Tests cover auto/live, auto/dead or malformed claim, desktop with a live claim, off with
a live claim, desktop failure, receipt failure, and ordering (receipt readable before
ping). Assert no extra notification on no-op, deferred, skipped, or dry-run wakes. List
tests cover current eligible wake,
done/cancelled/released/blocked/missing/ambiguous/non-task/event targets,
unlinked/closed-ledger history, re-snoozed rows, other-day history, and vault identity.
Preserve the existing conformance fixtures' meanings. Run targeted snooze tests, then
`just check` and `just docs-check`.

## Phase notice-core

**Repository: bob-mac-capture, portable CaptureCore and its tests.**

- Add `SnoozeWakeReceipt` and a bounded reader, mirroring
  `Sources/RefsCore/RefsScanReceipt.swift` without importing RefsCore into CaptureCore.
  Resolve `XDG_STATE_HOME`, then HOME/.local/state, and read
  `bob-cli/snooze/last-wake.json` up to 4 MB. Schema mismatch is ignored;
  malformed/oversized/unreadable input reports a nonfatal diagnostic. Include
  notification ownership, vault root, tasks, plan, and RFC3339 wake instant.
- Add schema-1 list and add-response models with optional capability fields.
  Missing/unknown ownership or eligibility is not permission to announce. Preserve
  decoding with older bob; unsupported notice capability disables only this feature, not
  the already-landed wake scheduler.
- Extend `BobProcessClient` with a `snooze-list` read lane and a separately serialized,
  non-cancelling `snooze-add` mutation lane. Use argv arrays for `snooze -f json` and
  `snooze add <route:block-id> +15m -f json`. Reuse the configured environment/vault and
  timeout/error patterns. UI dismissal and another read must not cancel a started
  mutation. Preserve `snooze-wake`.
- Model pure policy with injected time, today/vault identity, receipt, current presence,
  current agenda Events, settings, and verification result. Outputs should distinguish
  hold/drop/verify/show and the next deadline. Keep a generation token across the settle
  delay and subprocess; every completion must prove it still belongs to the current
  batch and configuration.
- Build pure notice presentation and acknowledgement state: filtered task identities and
  copy, per-batch expiry, beacon count, stable record IDs, and the action target list.
  Use current list budget for TODAY, not stale receipt totals. Durable receipt fields
  supply wake-time provenance.

Tests use injected clocks, not real sleeps. Cover 179.9/180 seconds, each away signal
separately and together, independent hold toggles, immediate active delivery, 1.5-second
return settling, a second hold during verification, overlapping meetings and exact end
boundaries, actual versus buffered spans, cancelled rows, absent/old agenda fields,
day/clock/vault changes, four-hour cutoff, duplicate and distinct IDs, empty/partial
verification, failures and unknown fields, and every acknowledgement trigger. Receipt
tests cover paths, size cap, invalid schema/JSON, and fixture parity with Rust. Process
tests assert exact argv/environment and lane isolation. Run `just portable-check`. Use
bob's fresh `date`/`now` as the task-day authority, including when its configured
timezone differs from the Mac's. Clock/day signals require new verification; never
combine a stale receipt date with a new local clock. Include offset/DST and
differing-system-zone cases in the clock tests.

## Phase notice-runtime

**Repository: bob-mac-capture, app services and tests.**

- Implement `SnoozeWakeObserver` using `notify_register_dispatch` and `notify_cancel`,
  following `RefsScanReceiptObserver.swift`. Read off-main, serialize acceptance on
  MainActor, coalesce overlapping reads, and dedupe by wake ID rather than byte hash.
  Read the latest receipt once at startup after registration to recover a missed ping;
  persist only terminally handled IDs so restarting does not silently discard a held
  latest receipt. Do not poll the file or fabricate receipts. The protocol retains only
  the latest receipt: document that it is not a durable multi-batch replay queue.
- Implement the atomic claim writer at the sibling `notifier.json` path, with
  `{schema_version:1,pid,app,version,started_at}` and 0600 temp/rename. Remove only a
  claim this instance still owns (pid plus start token). Register the observer and
  attach a working presenter before claiming; also require a successful version-1 list
  capability probe. A failed registration, incompatible executable, or unconfigured
  presenter must leave desktop fallback available. Reconfigure capability and claim on
  Bob/vault changes.
- Observe distributed `com.apple.screenIsLocked`/`screenIsUnlocked`, workspace screen
  sleep/wake and session active/resign, and day/clock changes. Sample initial
  session/screen state, not merely subsequent events. Poll
  `CGEventSource.secondsSinceLastEventType` every 2 seconds only while work is held;
  stop the timer when the queue empties. Use the existing agenda store for meeting facts
  and request its normal refresh on receipt and return. Do not treat an old-day or
  failed agenda refresh as a newly verified empty calendar; retry with bounded backoff
  while retaining pending work.
- Add a `SnoozeNoticeCoordinator` (or equivalently scoped owner) with injected receipt
  registration/reader, clock, presence source, process client, defaults, and
  presentation/record callbacks. It handles queue, hold, delay, bounded revalidation
  retry, day purge, and stale callback suppression. A failed read must not show stale
  tasks or mark them delivered. Retry no more often than every 30 seconds, or on a real
  lifecycle/configuration change. Consume only `notification_channel == mac_capture`;
  other receipts are ignored for all app notice surfaces.
- Add four persisted settings, defaulting to true. Turning Announce wakes off discards
  pending notices and clears UI/records, but keeps a healthy claim while compatible
  handling is attached: otherwise auto would unexpectedly fall back to a desktop banner.
  The wake scheduler continues independently. Chime and hold toggles apply live. Later
  turning announcements back on does not replay IDs intentionally consumed while
  disabled.
- Provide callbacks for present/update/dismiss, count/record changes, and panel-open
  acknowledgement, plus start/stop/reconfigure entry points. This phase may compile
  dormant services; AppDelegate activation belongs to the next phase so no placeholder
  presenter can suppress desktop fallback.

App-target tests inject notifications, process completions, and defaults. Verify claim
ownership across overlapping instances, observer/claim ordering, startup dedupe, held
restart, late results after stop/reconfigure, multiple batches, disabled settings,
bounded retry, timer teardown, and that the app cannot claim before presentation exists.
Compile and gate as below.

## Phase notice-hud

**Repository: bob-mac-capture, HUD and app wiring.**

- Extract the reusable panel/glass host from `Refs/RefsCopyHUD.swift`, with explicit
  interaction and placement choices. Preserve Refs public entry points and tests. For
  snooze, enable mouse events and first-mouse actions, keep `canBecomeKey/canBecomeMain`
  false and `.nonactivatingPanel`, set `sharingType = .none`, and anchor within the
  pointer display's visible frame. Bound width and wrap/truncate long titles without
  pushing actions offscreen.
- Implement copy and visual states from the delivery contract, including
  single/batch/over-cap, alarm motion, countdown, hover pause, opaque fallback,
  VoiceOver labels/announcement, and chime exactly once on actual presentation. Hide
  sensitive content immediately if a new lock/sleep/meeting hold occurs during display;
  do not issue a second chime for the same already-delivered ID.
- Body/Open opens the first remaining task in receipt order through
  `ObsidianOpenURL.taskGotoURL`; make each displayed batch title separately actionable
  to its own task. Acknowledge the batch when Open is used. Use validated routes and the
  configured vault, never a shell-composed URL.
- +15m applies to all remaining tasks in the shown batch, sequentially through the
  non-cancelling add lane. Disable repeated action while in flight and revalidate tasks
  before starting. Refresh the agenda after settled writes. Confirm only bob's
  successful `added` outcome, using its returned target for `Snoozed until HH:MM`
  (relative-time rounding belongs to bob). Do not compute an exact 15-minute label
  independently. On partial failure, remove successful tasks, retain unsuccessful tasks
  with truthful retry/error copy, and never retry already-successful items.
  Near-midnight refusal is visible and does not invent next-day snoozing.
- Attach the coordinator in `AppDelegate` after process/agenda/status services exist,
  publish a claim only after the real HUD is installed, wire panel-show acknowledgement
  and process reconfiguration, and stop/remove ownership on termination. Settings UI and
  record/beacon consumers follow in the next phase; their callbacks must not be needed
  to render the HUD correctly.

Tests assert nonactivation/first mouse/sharing configuration, pointer-display geometry,
countdown pause/resume, replacement/dismiss generations, no replay chime, all action
targets and partial failures, and unchanged Refs toast behavior. Render synthetic
single/batch/over-cap/light/dark/Reduce Transparency states through the existing app
test render mechanism. Compile and gate.

## Phase notice-records

**Repository: bob-mac-capture, final integration and docs.**

- Extend `StatusItemController`, `StatusItemFrame`, and `StatusItemGlyph` with a small
  amber alarm and count without losing the unresolved-bob marker or capture-landed
  pulse. The base glyph is a monochrome template today; render the colored accessory
  deliberately so template tinting cannot erase amber. Update tooltip/accessibility text
  and clear it through the coordinator's shared acknowledgement path. No stale pulse may
  restore a cleared count.
- Extend `NotificationService` with stable record ID `snooze-woken-<wake_id>`, thread
  `snooze`, `.passive`, and no sound. Query authorization without prompting;
  denied/not-determined means skip the record, keeping the HUD functional. Add a
  snooze-specific foreground path: current `willPresent` returns banner+sound+list for
  every notification and must return list-only for these records. Other categories
  retain behavior. Default/Open action uses the same task navigation and acknowledgement
  path.
- Withdraw delivered and pending record IDs on acknowledgement/expiry/day change and
  after successful re-snooze. Handle the async add-versus-clear race so a late add
  cannot resurrect a cleared record. Reconcile existing snooze record IDs at launch; do
  not remove unrelated notifications. Retain bounded acknowledgement metadata (ID, task
  keys, delivery/expiry dates, vault) to restore an unexpired beacon and its clearing
  rules across restart without re-announcing. This also works when notification
  permission denied creation of a record. Purge expired or foreign-vault metadata and
  records.
- Add the four Snoozed tasks toggles in `SettingsView` and metadata-only diagnostics
  (claim/capability, last receipt, held reason, last verification, queue/count, channel,
  failure). Add bounded signposts around receipt read, hold/return, verification,
  presentation, action, and withdrawal; avoid task text in logs. Update
  `docs/settings.md`, `docs/diagnostics.md`, `docs/protocol.md`, and
  `docs/develop/architecture.md`. Document the old-bob fallback, defaults, actual hold
  span, expiration, and channel precedence.

Tests cover silent foreground/background content, authorization denial without request,
stable IDs, action routing, each clearing trigger, multiple batches, add/clear races,
restart/expiry, template/accessibility rendering, pulse and unresolved-bob coexistence,
settings persistence, and independent wake scheduling while announcements are disabled.
Run portable and Mac gates and `just docs-check`. Record integrated evidence on the
phase for land.

## Verification and land responsibilities

Every Swift phase runs `just portable-check`. When app sources or app tests change,
compile a scratch Mac copy without .build/.git using `Scripts/bundle.sh` and
`just typecheck-tests`, then remove it. Use `/sase_memory_read` for tailnet instructions
before remote work. Never use Bryan's checkout or `just install`. Missing/offline Mac is
recorded as unverified, never as a passing compile.

Before landing Swift, run `Scripts/macos-ci-check` through `/sase_monitor`; after any
push monitor the exact SHA's macOS CI to green. Do not end a turn with an unmanaged
long-running check. Follow `/sase_final` for host commits; do not manually invoke a
commit skill without separate authorization. All source changes must compile. Formatting
stays limited to touched lines. Run the docs gate when docs change, without repeatedly
rerunning already passing checks unless source changes or failures justify it.

The child epic's **land agent** assembles the phase evidence and performs the end-to-end
checks specific to q5.8 using a compatible bob executable and the built Mac app. It
records exact commits/check receipts and confirms:

1. One active wake produces one HUD/chime and one silent record; duplicate pings, a
   repeated wake, and app restart do not create another nudge.
2. A real locked-screen wake stays silent; after user unlock the surviving task appears
   after settling/revalidation. Also cover idle/display sleep and session inactivity
   with the deterministic adapter tests.
3. A running meeting holds until its real end, including overlap; a task
   completed/released/re-snoozed during the hold is filtered out.
4. Open and batch +15m work, partial/refused writes remain truthful, hover pauses the
   12-second countdown, and focus remains in the original app.
5. Beacon/record clearing, notification denial, Reduce Motion/Transparency, voice
   announcement, day rollover, and 60-minute expiry match the contract.
6. With the app stopped, auto uses desktop fallback. With it running, explicit desktop
   and off never also produce the HUD. Old bob remains on fallback with no claim; newer
   capability enables the Mac feature.

Use synthetic fixtures for images and a disposable vault for automated mutation checks.
A genuine lock/unlock observation is required by the parent; simulation alone is not
that evidence. Coordinate the human unlock through the SASE question workflow if
necessary, recording what was actually observed. Do not disturb the production
installation to manufacture evidence.

Phase workers close only their assigned child phase after checking its
`sase bead epic-symbols`. Record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes,
never new task beads. Identical clean-base check failures get evidence/follow-up notes
and do not strand a completed phase. Resolve any allowances or re-key them to an
actually open owning phase.

After all phases, compiled artifacts, CI, docs, and real Mac acceptance have settled,
the **child land agent** verifies `sase bead epic-symbols bob-cli-q5.8` and closes
`bob-cli-q5.8` with the evidence note using the delegated-land completion flow. Child
phase workers have no authority to close q5.8. Keep the outer `bob-cli-q5` and its
ancestor plan open for their own land agent; do not perform q5.9's glossary/guide or
broader vault acceptance work here.
