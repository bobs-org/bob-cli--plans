---
tier: epic
title: Mac snoozed agenda and always-on wake scheduler
goal: 'Complete bob-cli-q5.7: show snoozed tasks separately in the Mac agenda, preserve
  their summary under folding, mark fresh wakes, and run Bob''s wake writer at its
  advertised instant while the panel is hidden.'
phases:
- id: snooze-models
  title: Decode snoozed agenda facts and style the capture token
  size: small
  depends_on: []
  description: 'snooze-models: add backward-compatible agenda and wake-provenance
    models, import Bob contract fixtures, and map the snooze semantic span to the
    editor''s time-marker palette. Update protocol documentation and verify the Swift.'
- id: later-today-band
  title: Render and fold Later today with woken accents
  size: medium
  depends_on:
  - snooze-models
  description: 'later-today-band: implement the separate presentation, persistent
    summary and first-to-fold behavior, measured Mac views, separate budget count,
    and sixty-minute woken accents. Cover empty-ledger, compact, and legacy cases
    and document the shipped idle agenda behavior.'
- id: snooze-waker
  title: Wake snoozed tasks while the capture panel is hidden
  size: medium
  depends_on:
  - later-today-band
  description: 'snooze-waker: add the schema-checked non-cancelling Bob wake lane
    and app-owned SnoozeWakeScheduler, with bounded retries and lifecycle invalidation.
    Include snooze instants in visible boundary refresh, verify integration with fake
    time and fake Bob, and document the command and lane contracts.'
proposed_by: bbugyi200.athena.bob-cli-q5.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.7
create_time: 2026-11-26 07:49:17
status: wip
bead_id: bob-cli-q5.7.1
---

- **PROMPT:** [prompts/202611/mac_snoozed_agenda.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_snoozed_agenda.md)
- **PARENT:** [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:** [bob-cli-q5.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.7.1.md)

# Mac snoozed agenda and always-on wake scheduler

Complete the implementation scope of **bob-cli-q5.7**, the `mac-agenda` phase of
`plan:202611/snoozed_tasks.md`. An empty capture panel will show work reserved for later
today separately from work available now. At Bob's advertised target instant plus five
seconds, the resident app will ask Bob to wake due work even while the panel is hidden.
Bob remains the sole vault writer and source of task facts.

## Evidence and size

The assigned bead is in progress, has no prior-work notes, and depends on closed phases
bob-cli-q5.2 and bob-cli-q5.3. Do not reopen it or assign statuses by hand. The existing
implementation was checked before choosing this scope:

- bob-cli commit `f32e774c` already publishes the agenda's snoozed rows and wake
  provenance. `docs/capture.md`, `docs/snooze.md`,
  `tests/cli/capture/pomodoros_agenda.rs`, and the `agenda-snoozed*.json` fixtures
  define and test the wire format. Capture parsing already publishes `snooze` spans;
  this plan does not change capture grammar.
- The opened bob-mac-capture checkout was clean at `5949d253`. Searches of Sources,
  Tests, and docs found no snooze implementation. Its agenda models lack the new fields,
  and its timer only refreshes Calendar boundaries while visible.
- `sase bead epic-symbols bob-cli-q5.7` currently reports no entries. Recheck at
  completion rather than assuming that remains true.

Use an epic because presentation crosses the measured-row/folding pipeline, while the
writer scheduler introduces an independent app-lifetime state machine. The small model
phase provides a stable contract first. Two medium coding phases then handle the
separate behavior changes. Dependencies are deliberately sequential: presentation and
scheduler integration both touch `CapturePanelModel.swift` and the protocol docs. Every
phase is direct implementation work for one coding agent.

## Rules and ownership

Read the source epic using `sase artifact read`, and open bob-mac-capture with
`/sase_repo` before reading or writing it. Read that repository's `AGENTS.md`. All
product changes in this plan belong to bob-mac-capture. Repository paths below are
relative to that checkout unless explicitly prefixed with `bob-cli:`.

Governing decisions:

- `decisions:mac-capture-is-a-thin-client`: decode Bob's facts, present them, and invoke
  Bob through argv arrays. Do not parse Markdown, implement capture grammar, resolve
  task identity, choose a wake destination, or write the vault in Swift. New JSON fields
  are optional; schema mismatches remain errors.
- `decisions:idle-capture-shows-ledger-agenda`: preserve the cached same-day snapshot,
  fixed editor eye line, numbered ledger rows, measured layout, and farthest-first
  folding. The approved source epic adds Later today as the farthest band and makes its
  collapse the first fold step.
- `decisions:today-is-read-from-the-ledger`: snoozed rows stay outside ledger groups,
  Today, close-list numbering, and the existing budget calculation.

The source epic's reviewer choices are final: uncommitted rows are ignored; the later
notification phase uses a HUD and holds during meetings. Only `glossary:snoozed-task`
was authorized for memory editing, and its authoring belongs to the outer close-out
phase. This child epic edits **no memory files** and adds no decisions. Do not edit
versions or CHANGELOG.md.

The notification claim, receipt observer, HUD, presence policy, chime, settings, beacon,
and re-snooze actions belong to **bob-cli-q5.8**. This work merely runs Bob's wake
command and refreshes the agenda after it settles. It must not emit a second notice or
create notifier claims/receipts of its own.

## Wire contract to consume

`bob capture-pomodoros --tasks --format json` remains schema 1. The optional top-level
`snoozed` object contains `rows` and nullable `next_wake_at`. Each row has `title`,
resolved vault-relative `route`, `block_id`, four-digit `target`, nullable RFC 3339
`wake_at`, `state` (`asleep`, `due`, or `blocked`), and `notes` using the existing
agenda display-line shape. Preserve Bob's row ordering and identity. The `next_wake_at`
instant includes its UTC offset and excludes blocked rows; it can be in the past for due
work. A hand-authored DST-gap row can have no `wake_at`. Do not synthesize an instant
from `target` for scheduling.

Optional Pomodoro item fields `woke_at` (`HH:MM`) and `snoozed_target` (`HHMM`) describe
a matching same-day wake. These are wall-clock strings, not RFC 3339 timestamps. The
agenda date supplies their day; Bob does not expire the fields. Use the existing agenda
calendar convention for display aging, and do not apply the naive-local date parser to
the offset-bearing wake instants.

The wake command is exactly `bob snooze wake -q -f json`, with no task argument. Its
schema-1 envelope includes `status` (`woke`, `nothing_due`, `deferred`, or `skipped`),
`date`, `now`, `woken`, `pending`, `next_wake_at`, and notification metadata. The CLI
owns writer-host pinning and write idempotence. `deferred` and `skipped` are valid
zero-exit results; a skipped host must not be treated as a successful wake or trigger an
explicit-task bypass.

## Phase snooze-models

1. Extend `Sources/CaptureCore/CaptureAgendaModels.swift` with optional
   `CaptureAgendaSnapshot.snoozed` and item wake provenance. Add focused snooze
   row/block types with defaulted initializers and `decodeIfPresent` fields.
   Missing/null blocks mean no feature, empty rows stay empty, nullable times stay nil,
   and an unknown row state degrades neutrally using the existing unknown-enum
   convention. Keep `schema_version` validation in the process client.
2. Reuse `CaptureAgendaLine` for row notes. Keep the raw identity/time facts intact. Do
   not add snoozed rows to `pomodoros`, task counts, or budgets.
3. Copy the following existing Bob JSON fixtures into `Tests/Fixtures` through the
   opened checkout paths: `bob-cli:tests/fixtures/capture_pomodoros/`
   `agenda-snoozed.json`, `agenda-snoozed-only.json`, `agenda-snoozed-history.json`, and
   `agenda-snoozed-past-day.json`. Record their Bob provenance in the fixture/test
   convention. Add focused decoder assertions for these and missing/null/unknown
   optional fields; retain old fixtures as backward-compatibility coverage.
4. Map the `snooze` span in `Sources/CaptureCore/CompletionRowContent.swift` to the
   existing time-marker semantic family (`.schedule`), so the capture editor styles
   Bob's `z:` range. Add the mapping assertion beside existing semantic-category tests.
   This is display-only: no regex/token recognition or Swift-side time interpretation.
5. Update the optional-field/span descriptions in `docs/protocol.md`. Run the common
   Swift and documentation gates below. This phase is independently useful and compiles
   before the new fields have consumers.

## Phase later-today-band

1. Build a separate snoozed presentation in
   `Sources/CaptureCore/CaptureAgendaPresentation.swift` (a focused sibling file is
   appropriate). Add explicit band row kinds and unit identities rather than disguising
   rows as Pomodoros. Thread the data through row equality and measurement keys so
   title/state/countdown/accent changes invalidate sizes. A same-day snoozed-only
   snapshot must show the band even when there is no `plan_budget`, no Pomodoros, or no
   Calendar band. Yesterday's payload stays hidden under the existing date guard.
2. Full rendering is a dim **Later today** header and compact rows such as
   `alarm 19:00  Fix the sink · in 3h 28m`. Use SF Symbol `alarm`, tabular times,
   subdued indigo for asleep, amber plus `due · waking` for due, and muted
   `waits on prerequisites` for blocked. Text and accessibility labels carry state
   without relying on color. Nullable/invalid times and unknown states remain neutral.
   Display Bob's clean titles and states; do not infer blocking. Keep the full row's
   note data available in the decoded model without adding a new notes interaction
   absent from the approved design.
3. Extend `CaptureAgendaFitPlanner.swift` with a full-to-summary snooze fold before
   every existing group/Calendar fold. Render the band after the Later name strip, or
   after the other content if there is no strip. Its summary is one row, e.g.
   `alarm 2 later · next 19:00`, and never disappears while rows remain. With only
   blocked rows, report the count/blocked state without a fabricated next wake. Reuse
   the existing chip expansion behavior to restore detail, subject to the existing
   constrained-height behavior.
4. Thread the new state through `CaptureAgendaPlan`, `render`, `contentRowCount`,
   `finish`, and `CaptureAgendaLayoutMetrics`. Include band spacing and insets exactly
   once. Update `CaptureAgendaSectionBlock`, `CaptureAgendaSections.make`, all
   exhaustive view switches, chip targeting, and `CaptureAgendaRowMeasurer`'s candidate
   enumeration/fixpoint measurement. Group ownership must terminate before the band;
   snoozed rows must never be accidentally swallowed by the last Pomodoro group.
5. Append a separate `· alarm N` to an existing saved budget row when snoozed rows
   exist, counting all provided asleep/due/blocked rows. Do not change `TODAY N/C`,
   `over`, cap coloring, or any close-list index. A snoozed-only payload without a
   budget still has its band; do not invent a cap.
6. Add a trailing amber `alarm woke 15:32` to agenda task headlines when their `woke_at`
   is on the snapshot day and its age is in `[0, 60 minutes)`. At 60 minutes remove the
   fresh accent; future, malformed, and prior-day provenance never gets it. Keep the
   accent when a task uses its one-line fold, without inventing task rows in a
   group-only strip. Preserve provenance through row-copy/constructor paths and row
   keys; audit the existing `headerWithNotesChip` reconstruction when adding row fields.
7. Ensure the visible minute repaint runs for snooze countdowns and active wake accents
   even when no current Pomodoro has a countdown. Pure presentation takes injected
   `now`; the app can update these displays without rereading vault files. The next
   phase wires fresh Bob reads at the exact boundary.
8. Add portable tests to the existing presentation/planner suites (or focused snooze
   siblings): ordering after the strip, first fold before Calendar/Now, permanent
   summary at tiny budgets, expansion, all-blocked rows, nullable times, no-ledger and
   prior-day snapshots, budget separation, old-Bob parity, and accent ages just
   before/at/after 60 minutes.
9. Add Mac layout tests in the agenda design/height suites for full, folded, expanded,
   and snoozed-only states at the existing 724/584 content widths. Preserve the existing
   rendered-height tolerance (2 pt), editor eye line, and source lineup numbers. Inspect
   synthetic light/dark renders of asleep, due, blocked, and fresh-woken cases,
   including long titles and accessibility labels. Store any durable visual evidence
   using the artifact workflow.
10. Update `docs/idle-agenda.md` to describe the band, folding, separate count, and
    fresh-wake accent. Run the common gates, including Mac app compilation.

## Phase snooze-waker

1. Add a focused schema-1 wake-result decoder and `BobProcessClient.snoozeWake()` in
   CaptureCore. Decode only fields needed for orchestration while tolerating additive
   fields. Run the exact argv above on a dedicated `snooze-wake` lane with
   `cancelsPreviousInLane: false`, the existing bounded process timeout, and normal
   schema/JSON/exit checks. Follow `isUnsupportedWalksFailure` for precise old-Bob
   command rejection; arbitrary runtime errors must not become permanent capability
   verdicts. Never print task titles in routine diagnostics.
2. Put scheduling decisions in a pure CaptureCore policy with injected clock, and use an
   app-owned `SnoozeWakeScheduler` adapter for timers/Combine/lifecycle events.
   `AppDelegate` retains it alongside `CaptureAgendaStore`, creates it at launch, and
   updates it on Bob executable/vault reconfiguration. Consume publisher-emitted
   snapshots rather than reading `@Published` properties again in a `willSet` sink.
   Follow existing generation-invalidated store code.
3. From a current-day snapshot, arm for `snoozed.next_wake_at + 5 seconds`. An overdue
   target on launch/resume runs promptly; a missing/null/invalid instant or
   absent/old-day snapshot disarms. Timers remain independent of panel visibility, draft
   contents, modal cards, and the Agenda setting.
4. Separate cancelling a pending timer from cancelling an already-started write.
   Serialize wake attempts with at most one pending follow-up; a repeated snapshot,
   retime, hide, or clock notification must not terminate the writer or launch
   overlapping writes. A vault/executable change invalidates old callbacks and timers so
   old-vault results cannot arm the new scheduler. Preserve the app's normal bounded
   process cleanup at quit.
5. Re-arm on snapshot changes, system wake, day changes, and system clock changes.
   Request fresh agenda facts after wake/clock/day changes and after every settled wake
   attempt. At a day boundary drop yesterday's timer/facts; Bob still performs the
   authoritative same-day check when the command runs.
6. Handle unchanged/failed refreshes and `deferred`, `nothing_due`, or `skipped` replies
   explicitly. An unchanged past target must not create a tight loop or suppress future
   legitimate retries forever. Keep a per-target attempt deadline with a 60-second
   minimum retry interval, one timer, and no immediate retry driven by duplicate
   publications. A newer future target arms normally. Use refreshed agenda/returned
   next-wake facts to retire completed work; a null wake target after settlement disarms
   pending wake work. Transient failures retain bounded retries; a confirmed unsupported
   command disables the feature for that client/session until recheck or executable
   replacement. Let Bob handle a wrong-host `skipped`; never bypass its pin.
7. Extend `CapturePanelModel.scheduleCalendarBoundaryRefresh` and its portable boundary
   helper to choose the earliest Calendar boundary and snooze target instant. Parse each
   format correctly. This visible-only read happens at the target time (before the
   independent +5-second write) to show `due · waking`. Keep the one-fire-per-boundary
   protection and existing Calendar behavior. Neither a visible refresh nor an ordinary
   snapshot repaint runs the writer.
8. Use fake-clock/policy tests without real sleeps for arming at +5 seconds, not-before,
   past-due launch, same-target retries, re-time/drop, blocked-only and null times,
   duplicate publications, day rollover, backward/forward clock changes, wake from
   sleep, unsupported capability reset, and generation invalidation. Verify the
   +5-second comparison uses the RFC 3339 offset, including an offset different from the
   process's local zone.
9. Extend `BobProcessClientTests` with a fake executable for exact argv, lane isolation
   from parse/preview/agenda reads, schema mismatch, malformed output, command
   rejection, and valid deferred/skipped replies. Add Mac scheduler integration coverage
   proving a wake still runs with the panel hidden/Agenda off, completion refreshes the
   store, overlapping triggers coalesce, and re-arm never cancels a running write. Every
   async wait has a deadline; no untimed continuations or gates.
10. Update `docs/protocol.md` with the wake command and old-Bob fallback, and
    `docs/develop/architecture.md` with the app-owned scheduler and separate
    non-cancelling lane. Update the idle-agenda refresh description where needed. Run
    all common gates and record final integration evidence.

## Verification and landing for every coding phase

Follow bob-mac-capture's `AGENTS.md` on the checkout used by that phase:

1. Run focused tests while implementing, then `just portable-check`. Format only changed
   lines in the house style. The portable suite cannot typecheck AppKit.
2. When app targets or app tests change, compile a scratch copy on the Mac: read tailnet
   memory first, rsync without `.build` or `.git`, run `Scripts/bundle.sh` and
   `just typecheck-tests`, and remove the scratch copy. Do not use Bryan's working
   checkout or install the scratch app. If the Mac is offline, record that scratch
   verification was unavailable and still require the macOS CI gate; never land Swift
   with no successful compilation.
3. For docs changes run `just docs-check` with `BOB_CLI_DIR` set to the opened matching
   bob-cli checkout, avoiding the wrapper's cache-clone fallback. All mounted pages stay
   in their owner repository. Do not regenerate unrelated docs or reference media.
4. Before landing Swift, use `/sase_monitor` for `Scripts/macos-ci-check` from
   bob-mac-capture. Wait for the staging command itself to exit, then follow the
   monitor/finalizer handoff. The script checks a throwaway exact-SHA CI run and its
   required build/test/bundle/install steps. After every subsequent push, monitor that
   push's exact SHA to green as required by the repository; stale green CI on an earlier
   tree is not evidence for new edits.
5. Record commits, relevant test totals, Mac compilation outcome, exact CI SHA and URL,
   and any visual/integration evidence on the assigned child phase. Use `/sase_final`
   for the host commit declaration; do not manually commit.

The child epic's land agent verifies the combined tree, confirms all three phases and
their CI evidence, and handles source-phase completion for **bob-cli-q5.7**. Acceptance
is the full scope above: old Bob remains usable; Later today survives folding and an
empty ledger; wake accents expire; a hidden-panel +5-second wake runs exactly one Bob
writer; stale/unchanged snapshots cannot spin that writer; and sleep/clock/day changes
recover without displaying yesterday's work.

Before source completion, run `sase bead epic-symbols bob-cli-q5.7` again. Resolve each
entry or re-key it to a genuinely open owner before closing that bead with
`sase bead close bob-cli-q5.7 --note "<verified behavior and evidence>"` under the
host's land assignment. Do not close bob-cli-q5 or another ancestor. Coding phase
workers close only their own assigned child bead and leave source closure evidence for
the land agent.

Record discovered out-of-scope issues as `PROPOSED FOLLOW-UP:` notes on the assigned
phase, with reproduction and clean-base evidence where applicable; do not create task
beads. An identical clean-base check failure is documented for triage rather than used
to abandon completed phase work. Actual new failures and missing required compilation
remain work to resolve before landing.
