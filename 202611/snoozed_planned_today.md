---
tier: epic
title: Snoozed reader integrations for bob-cli-q5.2
goal:
  Snoozed tasks survive reconciliation without propagation, stay out of review walks,
  and appear in plan forecasts and schema-1 agenda output while Today remains
  ledger-derived; verify and close the delegated bob-cli-q5.2 phase.
phases:
  - id: reader-recovery
    title: Share snoozed reads and preserve recovery-only reach
    size: medium
    depends_on: []
    description:
      "reader-recovery: extract reusable snapshot-backed snoozed row resolution from the
      landed core, then integrate direct recovery-only reach into task reconcile without
      promotion, dependency propagation, consumption, or Snoozed-section writes. Add
      focused classification and reconcile preservation tests and contract help."
  - id: review-exclusions
    title: Exclude planned-today tasks from freshness and walks
    size: medium
    depends_on:
      - reader-recovery
    description:
      "review-exclusions: thread is_snoozed_today through freshness scan, evaluation,
      placement, and seed, and use planned-today at review and lane-walk exclusion
      sites. Preserve ledger-only Today and whole-lane plan meters. Add executable
      freshness and walks vectors and publish their contracts for the plugin phase."
  - id: plan-forecast
    title: Report the snoozed meter and nonbinding wake forecast
    size: medium
    depends_on:
      - reader-recovery
      - review-exclusions
    description:
      "plan-forecast: extend PlanReport and every report construction path with snoozed
      counts, next wake, and an on-clock forecast; add the forecast lint and human line
      without changing current budget status, Today, or whole-lane meters. Cover
      missing-ledger, clock-policy, and cap-boundary cases and update docs."
  - id: agenda-provenance
    title: Publish later-today agenda rows and wake provenance
    size: medium
    depends_on:
      - reader-recovery
      - plan-forecast
    description:
      "agenda-provenance: add the schema-1 capture-pomodoros --tasks snoozed block and
      optional woke_at/snoozed_target item fields, including notes and zoned wake
      instants. Add agenda goldens and one cross-command lifecycle regression, update
      JSON contracts, and leave complete verification evidence for this child epic's
      land agent to close bob-cli-q5.2."
proposed_by: bbugyi200.athena.bob-cli-q5.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.2
create_time: 2026-11-25 19:37:32
status: wip
---

- **PROMPT:**
  [prompts/202611/snoozed_planned_today.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/snoozed_planned_today.md)
- **PARENT:**
  [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)

# Snoozed reader integrations

Complete the `planned-today` phase of `plan:202611/snoozed_tasks.md`, assigned as
`bob-cli-q5.2`. A task committed for later today must survive reconciliation and stay
out of review queues, while the plan and idle agenda show when it will wake. After wake,
its ordinary ledger link supplies Today membership and wake provenance.

This is a child epic, not a new design. Its four bounded implementation phases are
medium because each owns one reader concern, its tests, and its documentation.
Dependencies are deliberately ordered: the shared read interface lands first, the walk
contract precedes plan documentation, and plan/agenda changes serialize their shared
JSON reference and generated help. No phase requires a plugin or Swift edit.

## Starting evidence and scope

- Planning baseline: `03ca3f7e`. The tree was clean. `bob-cli-q5.2` is in progress with
  no earlier implementation notes; no `--epic-symbol` entries name it.
- `bob-cli-q5.1` is closed and its core is present: `snooze::{parse,state,when,plan}`,
  guarded CLI, delivery, tick, fixtures, and trailing-section writer tests. Reuse these
  rather than reimplementing wake or time parsing.
- `state::live_targets(&ClassifiedRows)` already returns `{planned, blocked}`. It is a
  pure classifier projection, not the parent's hypothetical note-loading helper.
  Resolution and useful row details currently live privately in
  `snooze/cli.rs::evaluate`, `ledger_identities`, and related helpers.
- None of the assigned integrations currently references snoozed state:
  `task_status_hooks`, `freshness`, `walks`, `plan_budget`, or the agenda modules.
- Existing `recent_activity_roots` feeds `desired_statuses` over dependency edges.
  Merely extending that set would propagate snooze reach to prerequisites, violating the
  approved design. Keep a separate direct snoozed recovery set.
- Existing freshness has separate ordinary, recurring, and checklist overlays; changing
  only `walk_scope` would miss overlays. The seed also distinguishes backlog scheduling
  from its today bucket; do not collapse those semantics.

At implementation start, reread this bead and check history/current source for work that
landed since this baseline. The predecessor's notes report unrelated failures tracked by
`bob-cli-q9`, `bob-cli-q4`, and `bob-cli-q7`; these are leads, not permission to assume
any new failure is preexisting.

Scope is bob-cli reader integration and its contracts. Capture `z:`, plugin indexes,
dash vault edits, visual marks, Mac UI/wakers/notices, glossary work, and live-vault
acceptance belong to the outer epic's other phases. No memory edits are needed here.
Never edit Cargo versions or CHANGELOG.md.

## Binding decisions

The outer review is final: `uncommitted_rows=ignore`, `wake_notice=hud`,
`hold_in_meetings=yes`, `glossary_snoozed_task=yes`, and `decision_record=no`. Do not
re-ask these choices. Only the outer glossary phase may use its authorized
`glossary:snoozed-task` edit; this child epic changes no memory.

Governing records, already consulted at planning time:

- `decisions:today-is-read-from-the-ledger`: Today stays dedicated links under open
  Pomodoros; planned-today is an additional exclusion predicate.
- `decisions:task-lanes-are-sticky`, `decisions:projects-block-from-prj-line`,
  `decisions:subprojects-inherit-project-blocking`, and
  `decisions:unblocked-tasks-resurface-as-ticklers`: preserve normal blocking and
  recovery precedence, daily-note clearing exceptions, and unblock-field skips.
- `decisions:consumption-keeps-notes-and-recorded-time`: keep existing ledger
  consumption/history rules; snoozed rows never enter ledger consumption.
- `decisions:ready-is-freshness-gated`, `decisions:review-walk-is-tiered`, and
  `decisions:recurring-occurrences-walk-when-due`: retain buckets, stage ordering,
  due-only review, and recurring/checklist behavior except the explicit snoozed hold.
- `decisions:review-attention-follows-the-walk` and
  `decisions:w-count-reads-lane-excess`: recompute attention/counts from the actual
  remaining queues and lane excess; never patch display counts separately.
- `decisions:off-clock-follows-tag-then-note-default` and
  `decisions:off-clock-note-defaults-filter-by-task-type`: the forecast uses the
  existing link-clock resolver, including inherited type-filtered defaults.
- `decisions:idle-capture-shows-ledger-agenda` and
  `decisions:mac-capture-is-a-thin-client`: bob supplies all new facts, additively, and
  existing roles, numbering, resolution, and task-note rendering remain intact.

The installed CLI lacks the documented `roster:decisions` selector. The catalog was read
via `sase memory web show decisions`, followed by audited reads of the governing records
above. This compatibility issue does not require a memory edit.

## Shared contract

Use resolved `(vault-relative path, block_id)` identities, including aliases, embeds,
and same-note links. `is_snoozed_today` means membership in the classifier's `planned`
set: Asleep, Due, or Blocked. Ready, terminal, Event, malformed, unresolved, ambiguous,
duplicate-loser, history, and previous-day rows do not add membership. A live ledger
link wins over an unstruck snoozed row. Planned-today is `is_today || is_snoozed_today`;
never insert snoozed identities into `today_tasks`.

Inject one Bob-zone clock and an explicit note date. Missing note/section yields an
empty snoozed view; a Snoozed section without a Pomodoros section can still be read.
Reuse the core's earliest-live `next_wake_at`; blocked and ignored rows do not arm it.
Read-only callers must not take write locks, wake rows, produce receipts, or notify.

## Phase reader-recovery

1. Add a reusable reader under `src/native/snooze/` by extracting the minimal shared row
   resolution/classification from `cli.rs`. Keep the pure `parse`/`state` APIs and
   guarded write boundaries. Prefer a snapshot/lookup adapter so hooks use their
   already-scanned note contents, and agenda/freshness can reuse cached reads. Do not
   force each consumer to parse CLI JSON or independently recreate eligibility.
2. Return parsed row/subtree details, classified state, resolved identity, and the
   planned/blocked sets. Preserve access to clean titles and struck-history times for
   later readers. Use existing task settings, duplicate-ID handling, Event recognition,
   and resolver semantics. Normalize absolute/relative paths at the adapter boundary.
   Migrate the snooze CLI to shared logic where needed, preserving existing outputs and
   write validation; do not broadly redesign its planners.
3. In `task_status_hooks/sync.rs`, classify today's snoozed rows against the scanned
   snapshot. Keep their direct identities separate from both `direct_desired` and
   `recent_activity_roots`. At each task's recovery calculation, membership offers
   `RankedStatus::Next`; retain any stronger legitimate ledger/activity rank. For a
   daily-note Next task, direct snoozed membership also supplies the keep predicate.
   Current derived blocking still wins. Ready never receives `desired` from snooze.
4. Never pass this set through dependency traversal or Pomodoro-reference structures.
   Never consume, strike, prune, deduplicate, re-order, or otherwise edit Snoozed rows.
   Existing guarded snapshots must cover every source used in reconciliation.
5. Add fixtures/tests in `tests/cli/task_status_hooks/` and its fixtures: daily Next
   kept; blocked target stays parked while reasons remain and recovers to Next once
   gone; unrelated blocked prerequisite gets no snooze recovery; Ready unchanged;
   stronger legitimate recovery rank retained; ignored and historical rows have no
   effect. Compare the entire Snoozed subtree byte-for-byte, including notes and struck
   history; retain LF/CRLF writer-audit coverage.
6. Update hooks long help and the affected reconciliation contract in
   `docs/task-dependencies.md`, plus the read/recovery rule in `docs/snooze.md`.
   Regenerate owned help with `just docs-gen`.

Verification: existing snooze unit/CLI and writer-audit tests, focused hooks recovery,
blocking, and consumption tests; `just check` and `just docs-check` before closure.
Record the concrete reader API and adapter expectations on the phase for consumers.

## Phase review-exclusions

1. Add `is_snoozed_today` beside `RowCtx::is_today` in `freshness/scan.rs`, populate it
   once per snapshot from the shared reader, and thread it through `FreshnessRow` and
   relevant placement scope inputs. Keep `is_today` truthful.
2. Apply planned-today at the ordinary scope, recurring overlay, `decide_for`,
   placement, and seed's existing Today exclusions. Audit every `is_today` use and
   constructor, not only the named functions. A snoozed PRE/POST row must also be held
   out, even though ordinary Today-linked PRE/POST rows remain eligible; add a
   snoozed-specific checklist guard rather than changing the Today exception.
3. Keep seed's established meaning: where Today prevents assigning a spread-out backlog
   date, planned-today does too. Preserve its existing today-bucket behavior, counters,
   and idempotence; never run the seed on the real vault or reseed old stamps. Test
   these semantics in an isolated fixture.
4. Change `walks/mod.rs::counts_in_lane` to exclude planned-today. Keep `bob plan`'s
   full NEXT/PENDING counts independent. `s` attention/tiers and `w` excess follow the
   filtered counts through their normal derivation.
5. Add named, executable conformance cases to the Rust state/placement tests and CLI
   freshness/walks tests, documenting the same inputs/outputs in `docs/freshness.md` and
   `docs/walks.md`. Cover asleep, due, blocked, Ready-ignored, linked-wins, recurring,
   PRE/POST, prior day, and wake returning to ordinary Today. Refresh
   `tests/fixtures/walks/` only for intended changes. Give the outer plugin phase stable
   vector identifiers to mirror; do not introduce a seventh group into the strictly
   validated six-group snooze fixture merely for walk tests.

Verification: focused freshness/placement/seed/walks tests, then `just check` and
`just docs-check`. Preserve existing JSON schema versions for additive/internal inputs.

## Phase plan-forecast

1. Add a typed `snoozed` block to `plan_budget::PlanReport`:
   `{count, due, blocked, on_clock, next_wake_at, forecast: {count, cap, over}}`.
   `count` is unique planned identities (live plus blocked); `due` and `blocked` count
   those states. `on_clock` counts only Asleep/Due identities that use the current
   on-clock link-budget classification. Blocked rows remain visible in the meter but do
   not join the automatic-wake forecast until eligible again.
2. Forecast is `links.count + snoozed.on_clock`, with `cap = max_links` and
   `over = count > cap`. Use `plan_budget::off_clock` / `LinkClockContext`, honoring
   explicit #on/#off, inherited boolean and type-filtered defaults, and current type
   resolution. Exclude ignored, history, duplicate, already-linked, and off-clock
   snoozed rows from the addition. Do not alter budget enforcement or wake
   placement/refusal. Exact cap is allowed.
3. Add `snooze_wake_would_exceed_cap` to warnings when that forecast exceeds cap. It
   never changes `PlanReport.status`, `links`, `today`, `today_tasks`, or whole-lane
   NEXT/PENDING meters. Use a shared report-enrichment path and update all callers:
   `plan_budget/cli.rs`, its empty-report path, and hooks `sync.rs`'s report path. Do
   not nest this enrichment under a `has_section` guard for Pomodoros.
4. Human output adds `SNOOZED 2 · next 19:00 · TODAY 11/13 when all wake`, with amber
   due text when applicable and sensible no-next/zero cases. JSON is additive; retain
   the current plan schema version. Update help and test fixtures together.
5. Update `docs/plan.md`: define planned-today for exclusions, preserve Today, explain
   the meter/forecast and dash exclusion requirement owned by the plugin phase. Document
   exact types in `docs/reference/json-outputs.md` and cross-reference `docs/snooze.md`.
   No actual vault dashboard edit belongs here.

Verification: human and JSON tests for no rows, no daily note, Snoozed-only note,
asleep/due/blocked mixtures, aliases/duplicates, explicit and inherited clock policy,
exact/over-cap, stable Today and full lanes. Run `just docs-gen`, `just check`, and
`just docs-check`.

## Phase agenda-provenance

1. Extend `capture_pomodoros.rs::CapturePomodorosResult` only for `--tasks` with
   `snoozed: {rows, next_wake_at}`. Each row is
   `{title, route, block_id, target, wake_at, state, notes}` for Asleep/Due/Blocked
   only, sorted by target then source order. `route` uses resolved vault-relative
   spelling; `target` is `HHMM`; `wake_at` is the target day's RFC 3339 instant in Bob's
   zone, using the core's fold/gap policy. `next_wake_at` excludes blocked.
2. Use existing agenda line rendering for `notes: AgendaLine[]` from the row's authored
   subtree, preserving depth, kind, and text. The rows are separate from Pomodoro
   entries and operator lineups. Never synthesize `=`/`x` numbers for them. Reuse note
   caches and resolution; avoid repeated whole-vault scans per row.
3. Add optional `woke_at` and `snoozed_target` to `AgendaItem` in
   `capture_pomodoros_agenda.rs`. Match well-formed struck history from today's Snoozed
   section by resolved identity to a resolved open task under an open Pomodoro. Preserve
   the core's `woke_at` spelling (`HH:MM`) and target (`HHMM`); the existing agenda date
   supplies the day. Ignore malformed/unresolved history; for hand-authored duplicate
   history choose the latest actual wake, source-order tie-break, and pin the rule with
   a fixture. Do not expire these facts at 60 minutes: accent aging is a client display
   rule, and historical facts remain available.
4. With `--tasks`, absent note/section gives empty rows and null next wake. A
   Snoozed-only note still produces rows. Without `--tasks`, omit the new block and
   leave the base payload unchanged. Keep schema_version exactly 1 and omit absent
   per-item fields. Preserve existing roles, numbers, events, budget, and warnings.
5. Add goldens under `tests/fixtures/capture_pomodoros/` and cases in
   `tests/cli/capture/pomodoros_agenda.rs`: asleep/due/blocked notes, history matched
   through alternate link spellings, no/empty ledger, malformed and ignored rows,
   past-day override, and non-`--tasks` compatibility. Add one fixture lifecycle
   regression comparing snooze list, reconcile, freshness, walks, plan, and agenda
   before/after an actual `bob snooze wake`, with repeat wake idempotence and no
   task-line freshness mutation. Keep receipt/notification side effects isolated.
6. Update `docs/capture.md`'s agenda section, generated companion help as needed,
   `docs/reference/json-outputs.md`, and the reader contract in `docs/snooze.md`.
   Publish a compact sample payload on the phase for the outer Mac phase to consume.

Verification: agenda goldens and unchanged base-command/lineup tests, snooze lifecycle
integration, `just docs-gen`, `just check`, and `just docs-check`.

## Landing and completion

Each worker closes only its assigned child phase after checking its own epic-symbols.
The child epic's land agent integrates all outputs and verifies the complete scope of
`bob-cli-q5.2`; it then runs `sase bead epic-symbols bob-cli-q5.2`, resolves or rekeys
any remaining entries, and closes that outer phase with an evidence note. That land
agent is responsible for this delegated completion, never `bob-cli-q5` or an ancestor of
the outer epic. Ordinary child workers do not close ancestors.

Final checks are `just check` and `just docs-check` on the integrated tree. Use
`/sase_monitor` for checks that require a handoff, and `/sase_final` to finish each
turn. Fix introduced failures. For a failure reproduced identically on a clean base,
record `PROPOSED FOLLOW-UP:` on the assigned phase with reproduction and any existing
tracking bead; it does not keep the phase open. Do not create task beads here. Evidence
must distinguish new passing regressions from known unrelated failures.

The completion note must confirm: Snoozed bytes survive reconcile; snoozed recovery does
not propagate; review/walk exclusions agree; Today and current budget are unchanged
until wake; the separate forecast follows clock policy; agenda schema 1 contains
later-today rows and wake provenance; no plugin, Mac, memory, or release work from
neighboring outer phases was claimed complete.
