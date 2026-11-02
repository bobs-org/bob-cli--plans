---
tier: epic
title: Complete Stats clock selection and reporting
goal: Finish bob-cli-fl.3 with Worked, Off-clock, and Total metrics, clock-aware session
  diagnostics, and complete CLI and JSON contracts.
phases:
- id: clock_evaluation
  title: Evaluate Stats metrics on separate clocks
  size: medium
  depends_on: []
  description: 'clock_evaluation: add request and pinned metric clocks, isolated formula
    evaluation, coverage, labels, and the clock/sub-share JSON contract.'
- id: session_diagnostics
  title: Report clock-aware sessions and clock diagnostics
  size: medium
  depends_on:
  - clock_evaluation
  description: 'session_diagnostics: filter session drill-downs by clock, attribute
    real off-clock minutes, and report unstamped rows, conflicts, drift, and overlaps.'
- id: clock_interfaces
  title: Expose and document all three Stats clocks
  size: medium
  depends_on:
  - session_diagnostics
  description: 'clock_interfaces: add CLI clock selection and completion, human and
    CSV output, integration coverage, documentation, and the inherited authorized
    glossary sentence.'
proposed_by: bbugyi200.apollo.bob-cli-fl.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fl.3
create_time: 2026-11-02 01:23:54
status: wip
bead_id: bob-cli-fl.3.1
---

- **PROMPT:** [prompts/202611/stats_clock.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/stats_clock.md)
- **PARENT:** [202611/off_clock_pomodoros.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/off_clock_pomodoros.md)
- **BEAD:** [bob-cli-fl.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fl/bob-cli-fl.3.1.md)

# Complete bob-cli-fl.3: Stats clocks

## Outcome and scope

Complete the `stats-clock` phase of `plan:202611/off_clock_pomodoros.md`, assigned as
`bob-cli-fl.3`. `bob stats -k worked|off-clock|total` and `bob stats show -k ...` will
select the clock of unpinned Pomodoro metrics. Reports will explain the selected clock,
retain off-clock minutes and homes, and diagnose suspect ledger rows without writing the
vault. The default work signal stays Worked.

This child epic has three sequential, directly implementable phases. The metric
evaluation change spans request/registry, dependency evaluation, coverage, series, and
serialization. Session diagnostics is a separate resolver/interval problem; the final
phase consumes both contracts to finish CLI output and documentation. Each phase is
bounded to one coding agent and must leave a compiling, tested tree.

The outer epic's final DECISIONS apply unchanged. In particular,
`memory_off_clock_glossary = yes` already authorizes the one Stats sentence in
`glossary:off-clock-pomodoro`. It is inherited authorization, not a new reviewer choice.
This phase has no need to change the other memory notes authorized to the outer epic.

Ledger writers, settling ended events, plugin UI, the Mac client, task completion
counts, and the menu bar belong to other outer-epic phases. This plan changes the
bob-cli Stats reader and its contracts.

## Evidence and starting point

- Read `bob-cli-fl.3` and the parent plan through audited SASE commands before work. At
  planning time it was `in_progress`, with no earlier-work notes or child plan.
- Commit `8a9ec685` landed the `bob-cli-fl.1` ledger foundation. Reuse `SessionClock`,
  `clock_stamp`, `counted_session_links_from_scan`, and `event_task::clock_verdict`;
  recheck the current checkout before implementing.
- `stats/index.rs` stores `PomodoroSession.clock` but only populates recorded minutes
  for Worked. `metrics.rs` skips every `session.untracked`.
- `StatsRequest` has no clock. `MetricSpec::Pomodoro` has no pinned clock. `compute.rs`
  currently keys evaluations only by metric id; mutating that one entry for the view
  would silently change formula inputs.
- `homes.rs::finish` and `session_facts.rs` report zero minutes for off-clock rows and
  do not filter the sessions list by the requested clock.
- `value.rs`, `series.rs`, and `report.rs` have no off-clock sub-shares. `render.rs`
  emits the old nine-column CSV header.
- `tests/cli/stats.rs::current_stats_contract_stays_worked_only` is a temporary
  foundation-phase guard that asserts no top-level clock object. Replace that obsolete
  assertion with the new schema contract while preserving numerical and human-output
  regression coverage.
- `sase bead epic-symbols bob-cli-fl.3` reported no entries. Check again before
  completion; another phase may add temporary entries in the meantime.
- The parent plan is the complete normative design. Its cited research artifact
  `research:202611/off_clock_pomodoro_stats/off_clock_pomodoro_stats.md` failed an
  audited read as missing during planning; this plan does not rely on unseen text.

## Shared contract and implementation rules

### Clock identity and ownership

Use a Stats lens enum with `Worked`, `OffClock`, and `Total`; distinguish it from the
ledger's `SessionClock`, which classifies a single row and includes Placeholder.
Canonical serialized/config/CLI values are `worked`, `off-clock`, and `total`. CLI-only
hidden value aliases are `on`, `off`, `untracked`, and `all`, respectively. All output
uses the canonical values and off-clock vocabulary.

A Pomodoro metric's optional configured clock wins over the request clock. The effective
clock appears in `parameters.clock`, including Worked. Invalid clock values, and
`clock:` on any other metric kind, invalidate only that configured metric with an
`invalid_metric` warning through the existing registry machinery. Tasks, fields,
checklists, journals, formulas, and their existing presentation do not follow the
request lens.

Formula dependencies evaluate in their configured/default context: an unpinned Pomodoro
dependency means Worked; a pinned dependency retains its configured clock. Key cached
evaluations by metric identity and effective clock, with a clock-independent key/context
for other kinds. Supply the existing formula and journal evaluators an id-indexed
projection of the canonical dependency values where useful. Evaluate directly selected
Pomodoro metrics separately when their effective lens differs. Reuse one
`StatsIndex`/vault scan, not multiple `compute` calls or vault reads. Nested formulas
and hidden inputs obey the same rule.

### Minutes, filters, and coverage

Trust the classifier's recorded number: `[t:: N]`/legacy stopwatch is Worked,
`[off:: N]` is Off-clock, a bare closed timed row is Off-clock by its span, and a row
with both keys is Worked using `[t::]`, with a conflict diagnostic. Closed placeholders
contribute neither clock. Do not reinterpret closed history from present-day task tags
or change the existing handling of malformed/missing data as an incidental refactor.

Apply the clock before accumulating each metric, using its existing theme/home filters
and rounding rules. Counts count selected sessions, not minutes. Real off-clock minutes
flow through the same home attribution and theme split helpers as Worked; retain
deterministic allocation and existing multi-home count semantics.

Worked retains its current inferred coverage. The Off-clock adoption date is the first
eligible closed off-clock session in the indexed history, including bare legacy spans,
independent of the requested window or an individual metric's home filter. Off-clock and
Total metrics cannot have usable points before adoption; a later metric `since:` or
global floor can move their start forward. Preserve `not_tracked` through prior windows,
grouped points, rolling medians, and typical days instead of filling earlier years with
zeros. With no adoption evidence, expose a null adoption date and do not invent one;
test the no-data path explicitly.

### JSON contract

Keep `schema_version: 1`. Add the top-level object:

```text
clock: {
  value: "worked" | "off-clock" | "total",
  label: "Worked" | "Off-clock" | "Total",
  since: ISO date | null,
  options: [{value, label, description, minutes, display}, ...]
}
```

The three options occur in Worked, Off-clock, Total order and report closed-session
minutes for the displayed window, including today's closed sessions when today is
included. They are independent of selected metric ids, configured metric filters, hidden
metrics, or display grouping. Exclude prior/median lead-in and running rows. Total sums
session minutes and may overlap; its description must not imply occupied or free time.
Keep window totals distinct from existing complete-day metric headline summaries.

Relabel effective Pomodoro metrics without changing ids:

| Id         | Worked           | Off-clock          | Total          |
| ---------- | ---------------- | ------------------ | -------------- |
| `worked`   | Worked           | Off-clock          | Total          |
| `sessions` | Sessions         | Off-clock sessions | Total sessions |
| any other  | configured label | label · off-clock  | label · total  |

Off-clock and Total Pomodoro metrics have `direction: neutral`; Worked retains its
definition's direction. Apply this by each metric's effective clock, so pinned metrics
stay correct in a differently lensed report. Definitions and human copy must not
describe off-clock values as worked minutes.

Pomodoro `points` and `daily_points` gain numeric `off_clock`, omitted when zero or in
the Off-clock view: in Total it is the off-clock sub-share of `value`; in Worked it is
the amount excluded by that metric's filters. Keep prior points consistent as well. The
value's unit matches its metric (minutes or session count). Total `parts_summary`
entries gain their own nonzero off-clock sub-share, including theme, root parent, area,
and project dimensions. Carry exact per-day/per-dimension contributions through the
existing aggregation pipeline; do not re-split aggregate minutes and introduce rounding
drift. Non-Pomodoro metrics gain no such fields.

`sessions[]` follows the request clock, carries real minutes, and adds `off_clock: true`
only for off-clock rows. Its existing homes object retains deprecated `untracked` beside
`off_clock`. The sessions list remains an unfiltered drill-down of the displayed window,
not an intersection of selected metric filters.

Extend the existing homes health block with optional `off_clock_sessions`,
`unstamped_sessions`, `clock_conflicts`, `clock_drift`, `overlap_minutes`, and
`overlaps[]` entries containing date, lines, and minutes. Keep deprecated
`untracked_sessions` equal to the off-clock count during the compatibility window. Omit
zero counts/empty lists. Diagnostics describe the displayed window even when the active
lens hides the offending rows.

The parent's byte-identical default-output goal applies to human output and existing
numerical semantics on a clean no-off-clock fixture. JSON deliberately adds the
specified fields, and CSV deliberately adds the clock column; do not suppress required
fields to preserve the obsolete whole-response shape.

## Phase clock_evaluation

Own `stats/request.rs`, `registry.rs`, `index.rs`, `metrics.rs`, `compute.rs`,
`value.rs`, `series.rs`, `report.rs`, and the minimal `homes.rs` changes needed for
metric attribution. A small dedicated stats clock module is appropriate if it keeps
selection/labels/window totals in one place.

1. Add the default Worked request clock and validated optional Pomodoro config clock.
   Keep the public CLI flag for `clock_interfaces`; engine tests construct requests
   directly. Update struct literals and catalog parameter serialization.
2. Retain both clocks' actual minutes in the index without duplicating classification or
   parsing tags in evaluators. Record the adoption date from eligible history.
3. Implement evaluation contexts and formula isolation in both the empty-date coverage
   probe and the real evaluation pass. Preserve journal gap dependencies.
4. Implement selection, home/theme filtering, adopted coverage, and exact off-clock
   contributions. Propagate them into daily/grouped/prior points and Total part
   summaries; clear them wherever the point becomes unavailable.
5. Build the top-level clock options independently of the selected metrics and attach
   effective parameters, labels, and neutral directions to metric reports.
6. Add deterministic engine/series/config tests for all three clocks, pinned precedence,
   invalid config, nested/hidden formula dependencies, mixed display plus formula
   inputs, adoption boundaries, grouped/prior points, and per-part sums. Update the
   foundation-only no-clock guard to the additive contract.

Acceptance: direct engine requests produce correct values and metadata for all three
clocks from one read-only index; formulas and non-Pomodoro metric values are stable
across request clocks. The default CLI still runs successfully. This phase does not
claim the entire outer phase is finished while session/reporting work remains.

## Phase session_diagnostics

Own the remaining `stats/index.rs`, `homes.rs`, `session_facts.rs`, `report.rs`, and
`compute.rs` session/diagnostic wiring, plus focused fixtures/tests. Depend on
`clock_evaluation` so the effective clock and duration plumbing already exists.

1. Populate real closed-session minutes and homes, mark off-clock rows, and filter the
   drill-down by the request clock. Keep catalog entries required by pinned metrics even
   when their session rows are filtered from the view.
2. Retain `ClockStamp` facts from the original row for conflict/unstamped counts. A bare
   completed placeholder is not an unstamped timed session.
3. Resolve the already-indexed counted Task Links, including same-note blocks and
   archived tasks, to exactly one current task line; reuse existing note/link resolution
   and `line_has_off_clock_tag`. Feed `LinkFact`s to `clock_verdict`. Record drift only
   for fully resolved lineups; unresolved or ambiguous links keep their existing
   resolver diagnostics and do not manufacture drift. Empty lineups follow the shared
   Unlinked verdict. Never rewrite the row.
4. Detect overlap using actual timed intervals, independently of keyed minute overrides.
   Touching endpoints are not overlaps. Use normalized ranges (including overnight
   spans), deterministic date/line ordering, and a bounded detail list (50 entries is
   sufficient). Keep the aggregate exact even when details are capped; document its
   definition and test triple overlaps so a pairwise listing cannot accidentally inflate
   aggregate minutes. Do not subtract overlap from any clock total. Avoid quadratic
   all-history comparisons on `-r all`.
5. Keep window-scoped counts independent of lens and of the number of displayed Pomodoro
   metrics. Extend existing health warnings so renderers can surface each diagnostic
   without a second vault pass.
6. Test keyed-vs-span duration differences, bare rows, conflicts, both directions of
   resolved tag drift, alias parsing, unresolved/ambiguous/deferred links, empty
   lineups, placeholders, adjacent/overlapping/overnight spans, and capping. Test all
   sessions filters, real home shares, compatibility aliases, and omission of empty
   diagnostic fields.

Acceptance: session sums and home minutes agree with their clocks, diagnostic counts are
deterministic and bounded in serialized detail, and vault bytes remain unchanged. Commit
the sessions contract change with a Conventional Commit `!`.

## Phase clock_interfaces

Own `stats/cli.rs`, `render.rs`, relevant completion metadata/tests,
`tests/cli/stats.rs`, `tests/cli/help_options.rs`, the Stats documentation, and the
single authorized glossary addition. Depend on `session_diagnostics`.

1. Expose `-k, --clock` on bare Stats/overview and show, default Worked. Accept hidden
   aliases while help/completion lists only the three canonical values. Place `-k` after
   `-h` and before `-o` to match existing short-option help order. `completion/tree.rs`
   already reuses Stats clap commands; add value descriptions in `completion/kinds.rs`
   where needed and test the actual completion protocol.
2. Finish overview and detail human output: Worked gets one dim
   `◌ 1h 30m off-clock · bob stats -k off-clock` line only when the window has off-clock
   time; Off-clock headers say `· off the clock` and empty windows say
   `No off-clock time in this window`; Total shows the worked/off-clock breakdown with
   matching periods/units. Keep non-Pomodoro and pinned metric labels honest. Surface
   nonzero conflict, drift, unstamped, and overlap health through existing
   health/warning output, without duplicate lines.
3. Append CSV `clock` as the final column. Pomodoro rows carry their effective canonical
   value; other metric rows carry an empty final field. Preserve quoting, existing
   columns, and newline behavior. Update exact-header tests and mark this contract
   change with a Conventional Commit `!`.
4. Extend Stats fixture coverage with worked, stamped off-clock, bare, conflict, and
   overlapping rows on one day, tagged/resolvable tasks and root parents. Preserve the
   original no-off-clock fixture as a regression oracle; a dedicated clock fixture or a
   temporary fixture overlay avoids rewriting unrelated golden expectations. Check
   explicit ranges, presets, offsets, all-range, grouping, today's closed vs running
   rows, and comparison windows through the CLI.
5. Assert every built-in/custom label in all lenses, pinned-vs-request precedence,
   unchanged formulas/tasks/fields/checklists/journals, null/dated adoption,
   zero-omitted sub-shares, exact filtered split sums, sessions, diagnostics, all output
   formats, alias equivalence, and no vault writes. Compare legacy numerical fields and
   exact default human output on the no-off-clock fixture, allowing only the specified
   additive JSON and CSV changes.
6. Update `docs/stats.md`: metric kinds/config, coverage and quality, schema-1 contract,
   human examples, CSV header, and M/H vectors. Update the JSON-output reference if its
   Stats summary/examples need adjustment. Read `cli_rules.md` and `docs.md` through
   `/sase_memory_read` before these changes, then run `just docs-gen` to regenerate
   `docs/reference/cli/stats.md`.
7. Use `/sase_memory_write` and read `glossary:off-clock-pomodoro` through
   `/sase_memory_read`. Under inherited `memory_off_clock_glossary = yes`, add: “The
   Off-clock and Total clocks of `bob stats -k` and the [[stats-dash]] count it, grouped
   by [[root-parent]] like any session.” Do not duplicate it if another authorized phase
   has already added it. Run `sase memory init` and `just docs-gen` to republish
   generated instructions and documentation.

Acceptance: `bob stats`, `bob stats overview`, and `bob stats show` expose the complete
parent contract, documentation and generated help agree, and all targeted tests and
required gates pass subject to the clean-base exception below.

## Verification and completion

Use focused Rust tests while developing (Stats unit modules and CLI Stats/help/
completion tests), with fixed `BOB_NOW` and `NO_COLOR=1` for output comparisons. Check
the actual crate/test target names before invoking filters. Run formatting,
`just check`, and `just docs-check` for the complete result; the current repository uses
`cargo fmt --all` to apply Rust formatting. Use `/sase_monitor` for long commands and
wait for the handoff command itself to exit. Record exact commands and results. No live
vault mutation is required.

Before resolving a failed check, reproduce suspected unrelated failures on an authorized
clean base. A failure identical on that base is recorded on `bob-cli-fl.3` as
`PROPOSED FOLLOW-UP: ...`, citing an existing task bead when known, and does not keep
completed work open. Workers do not create task beads.

Each child phase closes only its own assigned bead after its tests and
`sase bead epic-symbols <assigned-id>` pass. Resolve every owned symbol or re-key it to
an explicitly still-open owner before closing. Record remaining discoveries as
`PROPOSED FOLLOW-UP:` notes for land-agent triage.

The child epic's land agent verifies all three phases together against every item of the
parent's `stats-clock` section, including compatibility aliases and the authorized
memory sentence. It runs `sase bead epic-symbols bob-cli-fl.3`, resolves or legitimately
re-keys any remaining entries, and records the verification on `bob-cli-fl.3`. Once this
child epic and its descendants are complete, the land agent closes the original assigned
phase using `sase bead close bob-cli-fl.3 --note "<what was implemented and verified>"`.
That closure is the purpose of this child epic. Neither child phase workers nor this
land agent close `bob-cli-fl` or any higher ancestor. Never hand-edit bead status, and
do not reopen `bob-cli-fl.3` merely because it was already reserved.

Submit each normal turn through `/sase_final`; use `bead_action: keep` while only
planning or while any assigned scope remains unfinished. Do not manually change the
crate version or CHANGELOG.
