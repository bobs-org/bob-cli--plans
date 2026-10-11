---
tier: epic
title: Daily Stats engine for bob-cli-6l.2
goal: Complete bob-cli-6l.2 with a single-pass read-only stats engine, six configurable
  metric kinds, all eleven built-ins, honest schema-1 reports, tested aggregation,
  and engine documentation; the child epic land agent closes only that outer phase.
phases:
- id: contracts
  title: Define the stats model and configurable metric registry
  size: medium
  depends_on: []
  description: 'contracts: implement the schema-1 report and request types, isolated
    stats configuration, eleven built-ins through the same entry validator as custom
    metrics, and formula parsing and dependency validation. Establish the interfaces
    below; do not add a CLI or a placeholder compute implementation.'
- id: inputs
  title: Build the single-pass vault and journal index
  size: medium
  depends_on:
  - contracts
  description: 'inputs: reuse the Tasks per-file parser and Pomodoro scanner to read
    each Markdown file once, include archives with source residence, preserve date
    diagnostics, resolve parent links from the same walk, and index every host journal
    with deterministic ordering and health counters.'
- id: series
  title: Implement ranges, grouping, summaries, and display strings
  size: medium
  depends_on:
  - contracts
  description: 'series: implement pure range and prior-window resolution, ISO/calendar
    grouping, pooled ratios, boolean day counts, rolling medians, summaries, deltas,
    and shared formatting over synthetic daily series. Today never enters summaries.'
- id: vault_metrics
  title: Evaluate Pomodoro, task, checklist, and field metrics
  size: medium
  depends_on:
  - inputs
  description: 'vault_metrics: evaluate four derived metric kinds from the shared
    index, including since, quality, parts, archive-invariant task filters, checklist
    outstanding intervals, and typed daily fields. Add hand-calculated tests per kind.'
- id: journal_formulas
  title: Evaluate review answers and formulas
  size: medium
  depends_on:
  - vault_metrics
  description: 'journal_formulas: implement journal distinct-row and last-answer selectors,
    recorded-writer compatibility, review-close gap detection, and checked formula
    evaluation with null and quality propagation. Test multi-host reader vectors and
    the existing R1-R19 writer vectors without changing the journal writer.'
- id: integration
  title: Assemble compute, prove the schema, and document the engine
  size: medium
  depends_on:
  - series
  - journal_formulas
  description: 'integration: wire the completed components into compute(&StatsRequest),
    add the golden fixture vault and independent expected report values, document
    all engine contracts and limitations in docs/stats.md, and verify read-only behavior,
    deterministic output, and the repository checks. Leave the enclosing phase and
    all ancestor closure to the child epic land agent.'
proposed_by: bbugyi200.apollo.bob-cli-6l.2
parent_bead: bob-cli-6l.2
create_time: 2026-10-11 07:36:40
status: done
bead_id: bob-cli-6l.2.1
---

- **PROMPT:** [prompts/202610/daily_stats_engine.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/daily_stats_engine.md)
- **BEAD:** [bob-cli-6l.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6l/bob-cli-6l.2.1.md)

# Daily Stats engine for bob-cli-6l.2

## Objective and ownership

Complete the engine phase of `plan:202610/daily_stats.md`, assigned as `bob-cli-6l.2`.
Deliver one read-only
`pub(crate) fn compute(&StatsRequest) -> Result<StatsReport, StatsError>` in
`src/native/stats/`, its configuration, all six metric kinds, all eleven built-ins, the
schema-1 JSON model, meaningful tests, and engine documentation. CLI registration and
rendering belong to the enclosing epic's later phases.

This is a **child epic of bob-cli-6l.2**, not a replacement for the Daily Stats epic.
There are no earlier-work notes on bob-cli-6l.2 and no engine implementation in the
inspected tree. The phase spans several independently testable subsystems; six bounded
medium implementation phases avoid putting that whole implementation in one follow-up
coder. The graph is:

```text
contracts -> inputs -> vault_metrics -> journal_formulas --+
         \-> series -------------------------------------+-> integration
```

All dependencies are declared in frontmatter. Each phase must leave a compiling tree
with its own tests. The child epic land agent verifies the combined engine and closes
**bob-cli-6l.2** after the child work is complete; it must not close `bob-cli-6l`, any
other outer phase, or any ancestor plan bead. Descendant phase workers never close
bob-cli-6l.2 themselves. Do not set reservation/status fields by hand. Do not manually
create task beads: record out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on the
worker's assigned phase for land-agent triage.

Before implementing, read the assigned bead and the authoritative design through
`sase bead read` / `sase artifact read`. Sources actually consulted for this plan:

- `plan:202610/daily_stats.md`, especially Data model, Metric kinds, Engine
  architecture, JSON contract, Reliability, and Phase engine. It overrides the older
  research where they differ.
- `research:202610/bob_daily_stats/bob_daily_stats.md`. The initially missing local
  research artifact became readable after opening the research sidecar with
  `sase repo open`; continue to use audited artifact reads.
- `bob-cli-6l.1` completion notes, `docs/stats.md`, and
  `tests/fixtures/stats/review_answers.json` for what the journal phase shipped.

The enclosing epic's decisions are final: include `#ref` / `^ref` reading tasks in Done,
Created, and Dropped, with their own `ref` part; the future Stats chip is approved but
outside this child epic. Both memory proposals were declined. **Edit no memory notes,
generated agent instructions, plugin code, or live vault files.** No new decision gate
or memory proposal is needed here.

## Findings and implementation seams

- `src/native/dataview/tasks/index.rs::parse_file` is the existing full per-file Tasks
  parser. It handles frontmatter, comments, fences, hierarchy, and configured statuses.
  Expose a narrow reusable parser context through `dataview/tasks/mod.rs` instead of
  copying it or calling the whole-vault query/index per metric.
- `TasksSettings` and `StatusRegistry` currently have restricted visibility. Keep them
  behind that context, read settings once, and require Dataview format. Reuse the
  non-Dataview error wording from `note_ready` (the `bob ready` module).
- `RichTask` already has created/scheduled/due/start but lacks done/cancelled. Add both
  optional dates and update its one test literal in `freshness/seed.rs`. Preserve
  missing/invalid date provenance for stats before converting TaskDate to Option; do not
  reverse-engineer dates by forking the Tasks regex/parser.
- `capture_pomodoros::scan` supplies section presence, rows, states, names, and source
  line numbers. `completed_summary_for` uses recorded `[t::]` before
  `capture_pomodoros_agenda::range_minutes`. Extract a small shared duration helper if
  needed so stats can additionally expose fallback quality without changing existing
  capture results. `plan_budget::split_components` owns composite names.
- `collect_done::link_repair::source_relative_path_from_archive` maps
  `done/<path>_done.md` back to its source. Expose a narrow crate-visible wrapper;
  retain the existing archive convention instead of inventing a second one.
- `vault_links::NoteIndex::from_paths` works from the paths already collected; `resolve`
  handles paths, basenames, ambiguity, and wiki targets. Read frontmatter parent links
  from the same file buffers and follow them with a cycle guard.
- Add raw `stats: Option<serde_yaml::Value>` to `config::RawConfig`, beside plan and
  freshness, and a dedicated loader following `config/plan.rs`.
- Register the engine module in `src/native.rs`; do not add a `NativeCommand`, clap
  entries, help fixtures, or CLI commands. Internal tests can exercise the crate-private
  compute entry point before the CLI phase consumes it.
- The shipped shared fixture is `tests/fixtures/stats/review_answers.json` (R1–R19), not
  the planned `stats_journal/vectors.json`. Keep that file stable and add reader cases
  alongside it. Actual writer spellings include `decay:notNow`, `decay:level:0`, and
  `decay:lessOften`. Accept these existing events; journal outcome vocabulary is
  additive, and unknown outcomes contribute to `other`.

## Common contracts established by contracts

Use ordinary Rust typed structs/enums, deterministic maps/order, and serde for the
public report; avoid an untyped JSON blob shared by all modules. Internal names can
vary, but keep these boundaries stable for dependent workers:

- `StatsRequest`: vault path, config path, optional metric selection (overview,
  all-visible, or explicit IDs), optional range, grouping, offset, and compare. Obtain
  the clock once via `env::current_datetime()` so `BOB_NOW` applies; expose a pure
  helper accepting a fixed date/time for unit tests.
- `StatsError`: distinguish vault I/O (future CLI exit 1) from configuration,
  request/unknown-metric, and non-Dataview format failures (future CLI exit 2), with a
  stable code and message for the schema-1 error envelope.
- `MetricDef` / registry: validated kind-specific parameters, id, label, unit,
  direction, palette slot, source, origin, optional explicit since, definition, limits,
  and formula dependencies/AST. A registry build returns warnings as well as valid
  entries and their evaluation order.
- Daily values: date, integer `value` or null, optional ratio `num`/`den`, quality
  (`ok`, `partial`, `missing`, `not_tracked`), non-ok reason, and dimension/key integer
  parts. Today is provisional. Formatting/grouping adds display, end, rolling median,
  and bucket partial flags; evaluators never format terminal UI.
- `StatsIndex`: parsed task rows with source residence and date diagnostics, daily-note
  section/ledger/field inputs, resolved parent relationships, journal events by day, and
  once-per-input health counters. No evaluator performs I/O.
- `StatsReport` follows the enclosing design's full JSON schema: ok=true,
  schema_version=1, today, range with prior metadata, metric reports (definitions,
  summary, points, prior_points, parts_summary, warnings), health, and global warnings.
  Dates serialize as ISO strings. Ratio points include num/den; grouped points include
  partial. Empty parts may be omitted. No consumer must recalculate a display string,
  summary, percentage, or delta.

### Configuration and registry

Default range is `4w`; default overview is
`[worked, reviewed, done, created, net, review_closed]`; hide defaults to empty. Support
optional global since and user metrics. Ignore unknown keys, including plugin-only
`bob_path`. Missing config uses defaults. A malformed/non-mapping stats block or invalid
top-level range, overview, hide, since, or metrics container is a ConfigError naming
`stats.<key>` and the config path (the block itself names `stats`). Malformed metric
entries produce local warnings naming `stats.metrics.<id>.<key>`, leaving all unrelated
metrics usable. A bad stats block must not break plan/freshness/highlights loaders when
their own blocks are valid.

Built-ins must be literal config entries run through the same parser as custom entries,
not a separate evaluator switch on metric IDs:

| ID            | Kind and parameters                                  | Unit / color / direction   |
| ------------- | ---------------------------------------------------- | -------------------------- |
| worked        | pomodoro, minutes                                    | minutes / orange / up      |
| reviewed      | journal, reviewed, decision tiers                    | count / violet / up        |
| done          | tasks, done, recurring exclude                       | count / green / up         |
| created       | tasks, created, recurring exclude                    | count / blue / neutral     |
| net           | formula, `created - done - dropped`                  | signed / gray / down       |
| review_closed | checklist, tags gtd+post, match Morning review, bool | bool / violet / up         |
| dropped       | tasks, dropped, recurring exclude                    | count / gray / neutral     |
| habits        | checklist, tags gtd+pre, ratio                       | ratio / green / up         |
| keep_rate     | journal, keep_rate, pending+next                     | ratio / violet / neutral   |
| review_time   | pomodoro, minutes, themes GTD                        | minutes / orange / neutral |
| sessions      | pomodoro, count                                      | count / orange / neutral   |

A custom entry with a built-in ID replaces the whole definition and is marked
`redefined`; it is not a partial patch or a silent fallback if invalid. Custom IDs are
`custom`; otherwise origin is `builtin`. Hide suppresses lists/default selection, not
dependency evaluation or explicit selection. Formula unknown references and cycles
invalidate affected definitions with local warnings, including downstream dependents;
unrelated definitions remain available.

All six kinds support common label/unit/direction/color/since/description fields where
semantically compatible. Palette values are orange/violet/green/blue/gray, not hex.
Validate kind parameters and unit compatibility. Retain validated parameters for the
future metrics catalog. Parse formulas as integer literals, metric IDs, unary minus,
parentheses and `+ - * /`, never arbitrary code.

One naming conflict in the original table needs a concrete representation: `kind: tasks`
is the metric discriminator, so its optional task-class filter will be
**`task_kind: task|prj|ref`**. Do not generate duplicate YAML `kind` keys. Document this
spelling with a working example and validation tests.

## Phase inputs: one read per note

Collect Markdown paths once, excluding dot directories plus `_conflicts`, `_generated`,
and `_templates`, but including `done/`. Do not follow directory symlinks into loops or
other trees. Read task settings once; read each selected Markdown file once, with
bounded parallel reads following `ref_tasks/walk.rs`'s scoped-thread approach. Restore
deterministic path order before merging. All tasks, parent links, fields, and ledgers
derive from those buffers. Read only `.bob/stats/review/*.jsonl` as the explicit dot-dir
exception. Surface real I/O failures; do not turn an unreadable vault into a zero
report.

Canonical daily paths are exactly `YYYY/YYYYMMDD.md` with a valid calendar date and
matching directory year. Similarly named files elsewhere are ordinary notes. Index
frontmatter and inline daily fields without mistaking fenced examples for values. For
`source: either`, frontmatter takes precedence; inline is fallback only when the key is
absent, not when a present value is invalid.

Reuse the Tasks parser's statuses/global filter and skip code/comments exactly as it
does. Keep original status type; DONE and CANCELLED are distinct from the broad
`is_done()` helper (which includes NON_TASK). Archive residence maps back to source
before note filtering and parts; source parent ancestry governs `in:`. Preserve
diagnostics for missing, malformed, and future relevant event dates. Health counts each
affected task once per limitation, not once per metric, selected date, or grouping
bucket. Retain schema counters invalid_dates and closed_without_date; an additive
missing_created counter can report that separate limitation without mislabeling absence
as malformed input.

Journal parsing validates v=1, kind=review.answer, required strings, valid day, and
offset-aware timestamp. Keep day verbatim rather than deriving it from ts. Skip/count
malformed or unsupported lines, including a truncated final JSON line; blank lines are
not events. Missing journal directory is an untracked series, not an I/O error. Report
valid per-host event count, latest timestamp, and its day. Compare parsed timestamps by
instant (offsets differ); stable path/line tie-breaking makes identical-timestamp
conflicting answers deterministic.

Test fences/comments, configured custom DONE status, non-task status, settings read
errors, canonical versus lookalike daily paths, exclusions, archive mapping, parent
cycles/ambiguous basenames, malformed dates, journal truncation, and deterministic merge
order. Add a test-only read counter or injected reader to prove one Markdown content
read per report input pass.

## Phase vault_metrics: derived daily facts

Each evaluator returns daily values over a requested inclusive date span plus its
inferred since and limitations. Explicit metric since overrides inference; global since
is a floor. Dates before effective since are null/not_tracked. Infer since from the
earliest qualifying kind input: a canonical Pomodoros section, relevant valid task event
date, checklist instance, or daily field. No input ever means a fabricated ancient
tracking date.

**Pomodoro.** Only closed `[x]`/`[X]` top-level ledger rows count. Minutes use recorded
t first, then the existing range fallback with partial quality/reason. Missing daily
note or missing section is null/missing; an existing empty/open-only section is zero.
Minutes split equally over canonical uppercase composite theme components, remainder to
the first; unnamed is `(unnamed)`. Apply themes and exclude_themes to components, so
GTD's allocation alone feeds review_time. Count mode counts matching sessions once even
when multiple components match. Keep totals and theme parts internally consistent and
document count-mode parts. An unreadable duration must be surfaced as a
limitation/partial result, not silently described as complete observed zero minutes.
Test t overriding range, overnight ranges, uppercase X, absent section, empty section,
and odd-minute splits.

**Tasks.** Created counts any status on created date; Done requires status type DONE and
completion date; Dropped requires CANCELLED and cancelled date. Exclude recurring by
default, with only/include options. Always exclude the canonical daily note's `^gtd`
row, also after mapping an archived row's residence. Apply whole-token
tags_any/tags_all, task_kind, and `in:` paths/basenames/parent ancestry. Parts are kind
and source residence note. Kind precedence: ^prj, then #ref/^ref, then task. Ref tasks
are included by default. Dates before since are untracked; other dates are zero when no
matching events occurred. Missing, malformed or future dates are never imputed. Test
reopened tasks, NON_TASK versus custom DONE, recurrence modes, all three kinds, archived
history, parent matches, and exclusions.

**Checklist.** Match all specified tags as exact case-insensitive tokens and optional
normalized description substring. Chore identity strips inline fields, tags and block
ID, collapses whitespace, and lowercases. An instance is outstanding on
`[scheduled else created, completion]`, open through today. Met on D means a completion
on D; due means met or outstanding. Deduplicate chore identity across instances: met/due
counts chores, not rows. This includes late completion days, missed days, and deferred
instances without assuming daily recurrence. Ratio returns basis points plus num/den,
count returns met, bool returns all due met; nothing due is null/missing. Per-chore
daily parts must distinguish met, missed and not-due for the future grid (absent chore
key is not-due). Return limitations for undatable/cancelled instances rather than
inventing a history that the source cannot establish. Test
skipped/late/deferred/duplicate instances, case/whitespace normalization, tag-prefix
exclusions, and the POST substring.

**Field.** Read only the day's canonical daily note, with frontmatter/inline/either
source. Validate integer counts, minutes as integer or `30m`/`1h 30m`, and bool 0/1 or
true/false/yes/no. Absent or invalid is null with a reason; no silent coercion, negative
count/minutes, or unbounded parse panic. Test source precedence, inline bracket/plain
forms, missing notes and invalid values.

## Phase journal_formulas: recorded facts and composition

Reviewed counts distinct key/day in selected tiers: decision means every tier except
pre/post, all means every tier, and a list is an exact tier filter. Outcome parts use
each key's last selected event, unknown outcomes under other; tier parts count distinct
keys per tier. Repeated lines and multiple hosts do not inflate the daily total. Accept
both actual shipped decay spellings and the design's normalized spellings as recognized
outcomes without rewriting files.

Keep rate uses the last event per key/day among pending/next rows, intersected with any
configured tier restriction. Keep means outcome exactly `keep`; denominator is distinct
selected keys, zero denominator yields null/missing. Basis points round
deterministically; periods pool num and den, never average daily percentages. Use the
earliest valid event day across hosts as journal since unless explicitly
overridden/floored.

Evaluate the review_closed definition as a dependency even if not selected or hidden, so
a known closed-review day with **no event in any tier from any host** becomes
null/missing plus journal_gap for Reviewed/Keep rate. PRE/POST-only days have events:
Reviewed is real zero and Keep rate has no denominator, not a gap. Before journal since
retain not_tracked rather than flagging historical plugin gaps. If the user
invalidates/redefines review_closed, honor the registry result and report the resulting
limitation; never hide a second built-in evaluator.

Formulas evaluate dependencies in validated order with checked arithmetic and
half-away-from-zero division. Null input propagates null and worst input quality;
division by zero or overflow is null with an actionable reason, never a panic or wrap.
Pin quality precedence and inferred formula since consistently (all inputs must be
tracked, so inferred since is the latest dependency since). Literal-only formulas can
use explicit/global since, otherwise the requested range start. Do not eagerly evaluate
formulas against partially populated dependency maps.

Load R1–R19 from the existing fixture and prove the recorded/countable cases feed reader
behavior correctly. Add hand-authored reader vectors for duplicate keys, two hosts,
out-of-order timestamps and offsets, equal-timestamp ties, malformed and truncated
lines, unknown outcomes/tiers, pre/post-only days, keep then done, gap day with POST
met, pre-tracking dates, and formula null/cycle/division cases.

## Phase series: periods and presentation

Implement without vault access using the common daily model. Ranges accept `7d`, `4w`,
`3m`, `1y`, all positive `Nd/Nw/Nm/Ny`, `all`, and inclusive explicit ISO date
endpoints. Convert months to whole weeks using ceil(N*52/12), years to N*52 weeks (3m=91
days, 1y=364). Reject zero/negative/overflowing lengths, bad dates, inverted intervals,
and invalid offsets with usage errors rather than panics.

A preset yields N complete days ending yesterday plus a separate provisional today
point. Offset K shifts complete-day bounds backward K\*N days and omits today when K>0.
Prior is the immediately preceding equal-length complete window. Explicit ranges clamp a
future end to today, and only actual complete days feed summary/prior comparison. Reject
a fully future range after clamping. `all` starts at the earliest effective since among
selected metrics; include hidden dependencies for evaluation but not selection-based
start. Handle an empty or today-only history explicitly with null summaries and no
invented prior data. The schema's preset range end remains yesterday even when
includes_today=true.

Grouping auto is day through 91 complete days, week through 371, month beyond. Weeks
start Monday (ISO year boundaries included); month buckets use calendar months.
Edge-clipped buckets carry partial=true. Sum count/minutes/signed, pool ratio num/den,
and count boolean met days with `N of M` display where M is days with data. All-null
buckets remain null; mixed known/missing input is a partial aggregate with a reason
rather than implicit zero. Keep today's point separate when grouping so provisional data
never contaminates a completed bucket or period summary. Summaries always use raw
complete daily values, not bucket totals masquerading as daily observations.

Rolling median windows are 7 day points, 4 week buckets, or 3 month buckets; omit null
and provisional observations and require ceil(window/2) data points. Compute sufficient
preceding input for the first displayed point's rolling window, even without compare.
Provisional today has null rolling_median.

Summary: numeric count/minutes/signed headline is daily median; bool is days met / days
with data; ratio is pooled num/den. Also total, mean, p25, p75, days_with_data,
days_met, prior headline, and delta. Use sorted linear-interpolated percentiles (index
(n-1)\*p) and half-away rounding when an integer report value is required; test
even-sized inputs. All-missing summaries have null values. Parts summaries exclude
today, rank deterministically by total then key, and give top five plus other for
note/theme dimensions, with preformatted shares.

All display strings live in display.rs: minutes `4h 05m`/`45m`/`0m`, plain counts,
signed `−6`/`+9`, bool met/missed, ratio whole percentages, null `–`. Headlines add /day
or `N of M days` as specified. Deltas: ±1% is approximately same, ordinary numeric
changes are percent, >=2x or <=0.5x use multipliers, zero/nonzero prior is new, null
prior unavailable; ratios use percentage points, bool uses days, and net uses
shrinking/growing/flat by total sign. Include sign and accessible words naming the prior
period. Avoid nonsensical percent ratios when a custom signed metric crosses zero;
document that edge's neutral wording. Tests cover all thresholds, leap day,
year/ISO-year boundaries, clipped months, offset/prior alignment, pooled 1/1 plus 0/9 =
10%, and today's exclusion.

## Phase integration: usable compute and evidence

Wire request validation, config/registry, one index read, inferred since/range, required
dependency evaluation, complete daily series, grouping and formatting into compute.
Compute only selected metrics plus necessary dependencies, include the prior summary
regardless of compare, and fill prior_points only for compare. Preserve
requested/default metric order. Unknown explicit metric IDs are errors; invalid
configured entries already have warnings and must not disable unrelated overview
metrics. Health and warnings are deterministic and deduplicated.

Add `tests/fixtures/stats/vault/` with explicit Dataview Tasks settings (make sure
dot-dir files are tracked) and hand-calculated expectations. Exercise all eleven
built-ins plus a custom metric of each kind through the same compute path. Use a fixed
clock, e.g. 2026-10-11, and an M-vector table explaining each expected value; never
derive expected JSON by snapshotting the implementation. Cover the entire enclosing
plan's fixture list, including archive moves leaving results unchanged, reopening
removing Done, invalid dates counted once, a real zero versus missing/untracked, odd
composite minutes, checklist misses/late completion, field parsing, journal
duplicates/gaps, formulas, ranges, grouped ratios, and today. Test schema shape,
optional ratio/bucket fields, metrics origin/parameters, and error envelopes.

Prove compute does not alter a temporary fixture vault or config and returns identical
JSON for repeated fixed-clock calls. Verify shared duration parity against an
independent sum for recorded-t fixture rows. Add an opt-in real-vault parity test
(following tests/tasks_real_vault_parity.rs) callable from internal tests without
requiring an unimplemented stats CLI. Its oracle independently counts closed Pomodoro t
values and event-date tasks; normal tests skip it. Live-vault execution and <=300ms
measurement are rollout acceptance, not CI time assertions or permission to edit the
vault in this child epic.

Fill docs/stats.md with the catalog, exact exclusions/limits, six kinds, working
commented YAML, validation/invalid-value behavior, task_kind spelling, data
quality/since rules, window examples, grouping, summary math and formatting, schema-1
example, reader/writer vector references and M-vector table. Preserve the existing
journal documentation and clarify normalized versus shipped decay spellings. Record
limitations: reopened tasks leave Done, deleted tasks disappear from Created, Keep
imports retain their origin date, renaming a chore changes identity, and dates cannot
reconstruct missing history. List later/rejected metrics from the enclosing design. No
memory edits or CLI documentation claiming that the future command already exists.

## Verification and land procedure

Each phase runs the focused Rust tests covering its changed behavior and format and lint
checks. Integration runs the canonical `just check` (cargo fmt --check, cargo clippy
--all-targets --all-features, cargo test --no-fail-fast). Use `/sase_monitor` for
commands too long for a turn and wait for the monitor handoff command itself to exit. Do
not end a turn leaving a command session running.

When checks fail, fix failures caused by these changes. A failure reproduced identically
on a clean base is recorded as a PROPOSED FOLLOW-UP with the command, base revision,
failure, and any already-tracking task ID; it does not leave an otherwise complete phase
open. Preserve user/other-worker changes while checking the base. There is no plugin
testing or deployment in this child epic.

Before closing each assigned child phase run `sase bead epic-symbols <phase-id>`;
resolve any symbol or re-key it to a still-open owner. No `--epic-symbol` entries were
present for bob-cli-6l.2 at planning time. Avoid introducing placeholders that only pass
because a phase is open.

The child epic land agent reviews every acceptance item above, consolidates verification
on bob-cli-6l.2, and checks `sase bead epic-symbols bob-cli-6l.2` immediately before its
final closure. Resolve all remaining entries or re-key only genuinely later-owned items.
Once child work is complete, close bob-cli-6l.2 with
`sase bead close bob-cli-6l.2 --note "<implementation and checks verified>"`. Keep the
Daily Stats parent bob-cli-6l and every ancestor plan open for their own land agents.
Closing the assigned outer phase is this child land agent's terminal deliverable; a
worker's phase notes are evidence, never ancestor-close authority.
