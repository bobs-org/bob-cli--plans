---
tier: tale
title: Execute past Pomodoro stacks with explicit-entry close
goal: Complete bob-cli-h2.3 with atomic past recording, chronological placement, strict
  overlap refusal, and human and JSON output.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-h2.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-h2.3
status: done
---

- **PARENT:**
  [202611/past_pomodoro_recording.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/past_pomodoro_recording.md)
- **BEAD:**
  [bob-cli-h2.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h2/bob-cli-h2.3.md)

# Execute past Pomodoro stacks for bob-cli-h2.3

## Objective and scope

Complete **bob-cli-h2.3**, the record phase of `plan:202611/past_pomodoro_recording.md`.
`bob capture 'x=…'` must record past sessions, close their tasks at each session's own
clock, file the completed rows chronologically, and report the result in human and JSON
output. Preserve ordinary `x` behavior. This tale is one bounded implementation: the
grammar, name selection, task-close engine, atomic batch planner, and block tracker
already exist.

The parent epic's decisions are final: `memory_decision_record = no`; **edit no memory
files**. Do not change Cargo.toml versions or CHANGELOG.md. No new command or flag is
needed. Full overlap repair belongs to **bob-cli-h2.4**; this phase refuses all timed
overlaps and every repair `!`.

Before coding, reread
`sase bead read bob-cli-h2.3 -r "Need current phase scope and notes"` and the parent
plan through `sase artifact read`. Recheck the current tree for newly landed work. The
planning inspection found a clean tree at `cf2e3c9f`, grammar commit `ce8f53b1`, no
prior-work notes on the phase, and the temporary execution refusal in
`src/native/capture/plan.rs`. Do not repeat the grammar phase. The phase is already in
progress; do not change its status or reopen it routinely.

## Existing implementation and integration points

- `capture_language/model.rs` already provides `PomodoroPastSpec` and `PastStackPos`;
  execution and editor parsing already assign stacks. Reuse these text-level facts.
- `capture/plan.rs::plan_capture_batch` reads `now` once and plans items against staged
  snapshots. Its loop also tracks Pomodoro/task blocks and successor offers.
- `capture_pomodoro_close/linked_tasks.rs::plan_pomodoro_close_outcome` performs the
  reset check; `plan_pomodoro_close` independently finds the running entry. Below that
  lookup, close operations already accept the selected entry and clock.
- `capture/task_theme.rs::close_types` is another running-entry lookup. Its R6 naming
  logic must also accept the explicit recorded entry.
- `capture/pomodoro_close.rs::plan_pomodoro_close_item` composes close summaries, staged
  writes, dependency recovery, successor offers, and refreshed headline refs.
- `capture_pomodoro_close/ledger.rs::plan_ledger_close_with_parked` currently makes a
  continuation when links carry **or** there is no later entry. Its MEET-specific
  continuation placement moves that continuation to the end of the open queue.
- `gcal/plan.rs::closed_insert_at` contains the shared chronological placement rule.
- `capture/output.rs` has both close rendering and helpers that traverse close summaries
  for offers, deduplication, and batch reconciliation; extending only the serializer
  would miss these effects.

## Implementation

### 1. Make close planning accept an explicit entry

Add an entry-taking close variant, for example `plan_pomodoro_close_for_entry`,
receiving `&RunningPomodoro`, the close clock, and a small typed ledger-close policy.
Keep the current public wrappers using `find_running_pomodoro` and their existing reset
behavior. Share their implementation below entry resolution rather than duplicating task
effects. The record path calls the close-only variant, bypassing the reset branch even
for `x==0`, with or without notes.

Add an entry-taking `close_types` variant and reuse its existing theme lookup. Carry the
explicit entry through summary construction, staging, and successor processing; the
record path must never consult the global running-entry finder after writing its
temporary timed row. A separate running row may legitimately remain present.

The policy for ordinary `x` keeps current behavior. The policy for records creates a
continuation **only when links carry**, at the selected placeholder's slot, including
for explicit MEET records. Keep naming/narrowing/shedding, clock stamping from counted
links, task transitions, Work Logs, and dependency recovery in the shared close engine.
Successors remain offers unless the caller explicitly picks them through the existing
selection contract; never auto-link new work.

### 2. Preflight each whole stack before applying any member

Create `capture/pomodoro_past.rs`, wire it into the capture module, and replace the
temporary refusal dispatch with record planning. At a stack's first item, inspect the
full consecutive slice using its existing stack metadata before any member's task
effects. Use the ledger staged by earlier, non-stack items. Keep this preflight as a
clear seam for h2.4 to replace with guard-and-trim planning.

Read the batch clock once, truncate seconds for records, and compute `T = ceil5(now)`.
With checked arithmetic, member i has window
`[T - 5*(blocks_i + offset_i), T - 5*offset_i)` and close clock
`C_i = min(window_end_i, now)`. Do not reread wall time for members. Preserve the
ordinary operators' timing behavior outside this path. Reject overflow and a stack whose
start precedes today's midnight, with the typed token/stack and a useful maximum-block
rewrite. Allow T=1440 near midnight; format that endpoint as `0000` while retaining an
unwrapped minute value for interval arithmetic.

Read the day file and require its Pomodoros section. Reject every repair member with
`repair (!) is not available yet`. Scan every timed row in that section, including open,
closed, off-clock, legacy unstamped, and MEET rows. Refuse any half-open overlap and
name the conflicting row and line. Touching endpoints and zero-length rows occupy no
conflicting time. Normalize genuine midnight-ending ranges across 1440, while
distinguishing a zero-length range instead of treating it as a full day. No history
rewrites or trims belong in this phase.

### 3. Resolve and record members in typed order

After preflight, resolve each member against the state staged by its predecessors:

- Unnamed: use `next_future_pomodoro`, the first open untimed placeholder. Do not use
  the start path's preference or MEET rules. Refuse a missing placeholder with
  `no future Pomodoro to record; name one: x=build`. Refuse a MEET-first placeholder and
  teach an explicit session name or `x=meet`.
- Named: reuse `select_capture_name`, including exact quoted literals, ambiguity
  refusals, and near-match warnings. Use an open untimed placeholder in place
  (`source: placeholder`); a completed match creates a fresh row (`source: again`); no
  match inserts a named placeholder (`source: created`). Refuse a match to any open
  timed row, even when its span does not overlap, and teach closing it first.

Use `replace_placeholder_range` or the shared equivalent to write the canonical
requested window, build the explicit entry handle, and close it at C. Never call
start/restart/swap/MEET-event-run planning. Explicit `x=meet` uses its typed duration
(25 minutes by default). Earlier members' Event Task classifications and Work Log stamps
must use their own C, not the batch clock; no record reports an overrun or decrements
its requested span. Reset-shaped selections close and defer.

Reuse the close item's recovery/offer/staging pipeline, including its pre-close
baseline, without cloning a second task-status implementation. Keep all edits in
`CaptureBatchPlanner`; any later member, selection, log, or other batch failure must
leave filesystem contents unchanged.

### 4. File completed blocks chronologically and retain source identity

Extract `gcal/plan.rs::closed_insert_at` and only its needed generic helpers into a
shared ledger module. Have both gcal and capture use it, preserving gcal behavior:
insert after the last completed row whose start is <= the new start, otherwise before
the first completed row, otherwise before the first open row, otherwise at the section
start. Preserve complete child blocks and the note's line endings.

After close effects, successor handling, and headline refresh, remove the selected
completed block from its source and insert it using that rule on the remaining ledger.
Its carried continuation stays in the vacated source slot under the shed name. No carry
means no synthesized empty continuation. The independent running session retains its
bytes, span, name, children, and relative place among open rows; its numeric line can
naturally shift when a completed row is inserted before it.

Remap the recorded headline, continuation, nested close summary, task references, and
block refs through the move. Work Logs in the daily note can move the whole section, so
do not identify a row solely by its old absolute line or name. Preserve the block
tracker's original identity: `At(source line)` for a placeholder and `Created` for
fresh/again rows. Add `Placement::Recorded` and the `recorded` block role; the `trimmed`
role is h2.4's work.

### 5. Expose nested record output and update the temporary help

Add optional `CaptureItemResult.pomodoro_past`, absent on other captures, with:

```text
raw, name,
target { source: placeholder|again|created, line },
blocks, repair,
stack { index, size },
anchor: HHMM,
window { start: HHMM, end: HHMM, minutes },
closed_at: HHMM,
line,
trims: [],
close: existing PomodoroCloseSummaryJson
```

`target.line` identifies the selected/inserted source before recording; `line` is the
resulting recorded headline. Use `kind: pomodoro_past`, `placement: recorded`, and the
recorded ledger text as `task_line`. Do **not** serialize a top-level `pomodoro_close`
for a record; older clients would render the wrong card. Keep `trims` as an empty array
ready for the next phase. Preserve existing non-record JSON and the grammar revision
already set to 12.

Provide shared accessors for close summaries, or otherwise update every applicable
`all_*`, mutable traversal, offer/deduplication, and line-refresh path to include the
nested `pomodoro_past.close`. Verify a later batch item can reconcile offers from an
earlier record, and that no traversal reports a summary twice.

Reuse close rendering for task rows, `next:`, clock labels, and R6 `from ...`.
Parameterize the header for `recorded` / `would record`, plus ` · i of n` for stacks.
Retain the ordinary close renderer's output unchanged. Remove the temporary "not
available yet" help statement; add a short record description and the example
`x=build@2 x=fix@3` in `capture/cli.rs`. Read `docs.md` through the memory skill, then
run `just docs-gen` and keep generated help/docs in sync. Broader documentation and Mac
UI work remain with the epic's separate phases.

## Verification and acceptance

Add `tests/cli/capture/pomodoro_past.rs` and register it in the CLI test module. Use
isolated vault fixtures, fixed `BOB_NOW` and `BOB_DAY_FILE`, and exact byte assertions
for durable effects. Replace the temporary valid-token refusal test in
`parse_pomodoro_past.rs` with coverage appropriate to executable records; retain its
parser-contract tests. Add focused lower-level tests where entry selection, ledger
policy, or placement is best isolated.

Required coverage:

1. At 09:21, `x=build` records 0900-0925 and closes at 0921; `x=build@2 x=fix@3` records
   0900-0910 and 0910-0925, closing at 0910 and 0921. Assert human and JSON fields,
   exact ledger bytes, no overrun/decrement, and both FIX spellings (`x=2#fix=*3` /
   `x=fix@2=*3`) producing identical writes.
2. Compare `x=N` with `=N-N x` on quiet ledgers across 09:20-09:24 and 09:20:59,
   including close selection and log variants. Use fixtures where their shared contract
   applies. Explicitly test intentional differences separately: records never reset,
   never create an empty continuation, and keep MEET carry in its slot.
3. Two/three-member stacks, blank-line adjacency, a stack broken by `=`, and `x=2 x=3`
   consuming carried links in order. An Event Task starting at 09:15 is `not_started`
   for the earlier 0900-0910 member; its Work Log uses 0910.
4. Missing file/section/placeholder, MEET-first refusal, named placeholder/again/new,
   literals, ambiguity, near-match warning, and named timed-row refusal. Explicit MEET
   duration stays typed. Test R6 narrowing/shedding and on/off-clock stamping.
5. An adjacent running session is byte-preserved while the past block is filed before
   it. Overlapping open/closed/off-clock/legacy/MEET rows refuse; touching boundaries
   and zero-length rows do not. Every `!` refuses, including a later member. A conflict
   found only in a later member still aborts the entire batch.
6. No-carry closes leave no empty placeholder; carried and deferred links remain at the
   source slot even when other placeholders or a running row precede it. Check
   chronological insertion before/between/after completed blocks and a daily-note task
   whose Work Log insertion shifts the section. Assert block identities and reported
   line numbers, not only names.
7. `x==0` always closes and defers with and without notes; explicit selection, wildcard,
   parked, dropped, and unresolved-link behavior stays consistent with the existing
   close engine. Dependency recovery and successor offers still work through the nested
   summary and across later batch items.
8. Midnight: 00:05 with 3 blocks refuses and offers a valid maximum; 23:58 records
   2345-0000. Seconds do not move the anchor unexpectedly. Check overflow refusal.
9. Every refusal preserves all vault bytes, including when an earlier batch item would
   have edited another file. Fixed-clock dry-run predictions match execution apart from
   the dry-run flag and human tense, with no writes from preview.

Run relevant close/record/parse/block-tracker and gcal placement tests during
development. Existing Event Task close vectors and ordinary close/reset regressions must
pass unchanged. Finish with `cargo fmt --all`, `just docs-gen` (after help edits),
`just docs-check`, and **`just check`** (fmt, clippy, all tests). Use `/sase_monitor`
for long verification commands and wait for the handoff staging command itself to exit.
Do not claim completion on an unfinished test process.

If a check failure reproduces identically on the clean base, record it as
`PROPOSED FOLLOW-UP:` on bob-cli-h2.3, with reproduction evidence and any existing
tracking bead, and finish the phase as instructed. Do not repair unrelated failures or
create task beads. New out-of-scope findings also go only into
`sase bead note bob-cli-h2.3 'PROPOSED FOLLOW-UP: ...'`.

## Completion boundary

When the implementation and verification are complete, run
`sase bead epic-symbols bob-cli-h2.3`. Resolve each remaining symbol or re-key its
Justfile entry to an appropriate still-open epic/phase before closing. Record the guard
integration seam and verification evidence, then run
`sase bead close bob-cli-h2.3 --note "<specific behavior and checks verified>"`. Close
only this assigned phase; do not close bob-cli-h2 or any ancestor. Finish through
`/sase_final` as required by the host. No implementation is part of the planning turn
that proposes this tale.
