---
tier: epic
title: Capture snooze token and atomic row operations
goal: Capture can snooze new and existing tasks, link or drop snoozed rows safely,
  and complete same-day times with documented editor and JSON contracts.
phases:
- id: grammar
  title: Parse snooze intents and expose editor spans
  size: medium
  depends_on: []
  description: 'grammar: add the shared terminal z:<when> intent, existing-task exception
    and snooze-drop spelling, editor spans and diagnostics, and parser tests; guard
    execution until the mutation phase consumes the intent.'
- id: mutation
  title: Apply capture snooze operations atomically
  size: medium
  depends_on:
  - grammar
  description: 'mutation: connect snooze add/drop to capture snapshots for existing
    and new tasks, preserve row notes when linking now, implement retime and drop,
    and verify refusals and whole-batch atomicity.'
- id: completion_docs
  title: Complete snooze times and publish the capture contract
  size: medium
  depends_on:
  - mutation
  description: 'completion_docs: add event-first z: completion with literal replacements,
    document the grammar and additive JSON, and run integrated capture and documentation
    verification.'
proposed_by: bbugyi200.athena.bob-cli-q5.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.3
create_time: 2026-11-25 19:34:24
status: done
bead_id: bob-cli-q5.3.1
---

- **PROMPT:** [prompts/202611/capture_snooze_token.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_snooze_token.md)
- **PARENT:** [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:** [bob-cli-q5.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.3.1.md)

# Capture snooze token for bob-cli-q5.3

## Objective and ownership

Complete the `capture-z` phase of `plan:202611/snoozed_tasks.md`, assigned bead
`bob-cli-q5.3`. This is a child epic of that phase, not a replacement for the outer
snooze epic. All implementation is in bob-cli. The three phases are sequential because
they share grammar models and execution dispatch; each is a bounded implementation task.
The child epic's land agent verifies the complete scope and closes `bob-cli-q5.3`. Phase
workers close only their own assigned beads. Nobody working this child epic may close
outer epic `bob-cli-q5`.

Use `sase bead read bob-cli-q5.3 -r "Need original capture-z scope and final decisions"`
and `sase artifact read plan:202611/snoozed_tasks.md "Need approved snooze contracts"`
for authoritative context. Read `docs/snooze.md` for the landed core contract.

## Current evidence and size

At planning time HEAD was `03ca3f7e`, the working tree was clean, and `bob-cli-q5.3` was
in progress without earlier implementation notes. The prerequisite `bob-cli-q5.1` is
closed. Its implementation includes:

- `src/native/snooze/when.rs`: `SnoozeClock`, `parse_snooze_when`, `ResolvedTime`, and
  teaching refusal messages.
- `src/native/snooze/plan.rs`: pure `plan_add`, `plan_drop`, `plan_cleanup`, and
  `plan_wake`; `TaskFact`, `IdentityMap`, and file postimages.
- `src/native/snooze/parse.rs` and `state.rs`: section/row parsing and resolved task
  identities and states.
- Structural `## Snoozed` registration, CLI, tests, shared vectors, and trailing-section
  writer coverage. Commit `03ca3f7e` also settles touched notes on add. Reuse this work.

Capture has no snooze marker, span, completion context, or mutation adapter.
`capture_language/markers.rs` currently extracts schedule and priority;
`editor_model.rs::SpanKind` has `Schedule` but no snooze. Existing-task links are
dispatched through `capture/plan.rs` and `capture/pomodoro_link.rs`. Their ordinary link
guard can drop local prerequisites, which snooze must never invoke to make a Blocked
task eligible.

There were no `--epic-symbol` entries for `bob-cli-q5.3` at planning time. The installed
SASE command rejected `memory read roster:decisions`; the roster was inspected through
`sase memory web show decisions -f json` and the governing records were then read
through audited `sase memory read`. Do not change SASE tooling or memory as part of this
feature.

## Governing decisions and fixed boundaries

Honor the outer epic's final choices: `uncommitted_rows = ignore`, `wake_notice = hud`,
`hold_in_meetings = yes`, `glossary_snoozed_task = yes`, `decision_record = no`. This
child epic needs no memory edits. The glossary work belongs to the outer docs phase; do
not edit any memory note, including the rejected snooze decision record.

These records govern this implementation:

- `decisions:mac-capture-is-a-thin-client`: Bob owns grammar, completion, preview, and
  one aggregate transaction. Preserve schema version 1 and grow wire objects additively.
  Mac span styling belongs to `bob-cli-q5.7`.
- `decisions:capture-sigils-say-where-what-how` and
  `decisions:capture-reserves-clock-tags`: keep route/type/directive ownership and
  literal clock-tag behavior intact.
- `decisions:capture-link-drops-blocking-deps`: ordinary link-now retains its existing
  behavior. Snooze is a separate intent and refuses Blocked targets before dependency
  removal or schedule pull-forward.
- `decisions:task-lanes-are-sticky`: snooze commits Ready to Next; dropping a snooze row
  leaves the lane unchanged.
- `decisions:consumption-keeps-notes-and-recorded-time`: resolve identities, keep
  authored notes and recorded history, and remove only eligible entries emptied by the
  operation.
- `decisions:today-is-read-from-the-ledger`: a snoozed task has no open ledger link and
  contributes nothing to the current Today link budget.
- `decisions:open-pomodoro-names-follow-task-links`: settle touched live entries through
  the existing capture settlement path after removing or inserting links; report final
  positions from settled snapshots.
- `decisions:event-buffers-widen-the-span`: keep meeting and buffered clocks distinct.
  The requested event-end suggestions use the event's meeting end from the Events
  parser, not a reparsed MEET ledger range.

Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before relevant changes.
Add no subcommands or flags. Never edit package versions or `CHANGELOG.md`. Use
`feat(capture): ...` for additive changes; a genuinely breaking wire/CLI change would
require `!`, so avoid one here.

## Finished behavior

With a fixed clock before 15:30:

| Input                               | Result                                                                                                                          |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `@job:email-dan-eod z:1530`         | Commit the existing eligible task and place its link subtree in `## Snoozed`.                                                   |
| `Fix sink @home:fix-sink z:19`      | Create Next task `^fix-sink` and snooze it until 19:00, without inserting a Pomodoro link.                                      |
| `- Fix sink @home z:19`             | Support the outer plan's route-only example: create the task, mint its block ID with the existing helper, and snooze it.        |
| `@job:email-dan-eod z:+1h`          | Retime its active row; one row per resolved task identity.                                                                      |
| `@job:email-dan-eod` while snoozed  | Link now using ordinary placement, including running-first; carry the row's child notes to that link and remove the active row. |
| `@job:email-dan-eod!` while snoozed | Drop its active snooze row without waking, releasing, completing, or cancelling the task.                                       |

The route-only new-task example means `z:` itself requests commitment for an ordinary
task creation with a routable destination. Resolve its target through normal capture
routing, and use `capture_block_ids::mint_block_id` against the staged note's complete
used-ID set when no authored block ID exists. Preserve an authored ID when present. This
makes the outer example concrete without a second ID algorithm. A non-task capture
(note, drawer/bullet, reference, record, standalone session operation, requirement, or
navigation) has no snoozable task target and refuses `z:`. Do not reinterpret those
forms as tasks. Respect existing forced-destination and item-kind restrictions.

An item takes at most one lowercase, terminal `z:<when>` marker. Support the established
terminal-marker positions around the route, with UTF-8 byte offsets into the original
draft. Child prose and code literals do not become parent markers. Empty or malformed
claimed `z:` tokens remain incomplete/invalid editor states and fail execution with
teaching copy; they must not silently become task text or ordinary link captures.

Use the core time parser for `1530`, `15:30`, `3:30p`, `3p`, `19`, `+45m`, `+2h`,
`+1h30m`, and bare `330`. Use one Bob-zoned injected clock per operation, including
`BOB_NOW` and `BOB_DAY_FILE`. Do not duplicate rounding, same-day, DST-gap, or fold
rules. Clock-dependent validation belongs at the shared time-resolution boundary, not in
a second lexer.

Refuse past times with the core same-day explanation plus capture advice to link now or
use `s:1` for tomorrow. Refuse next-day times, duplicate `z:`, `z:` with `s:<N>`
(including `s:0`), a resolved priority schedule, session start/close suffixes, or a drop
suffix. Refuse Event Tasks with their clock explanation; refuse effective Blocked
targets, including inherited project blocking, and done/cancelled/non-task targets.
Every refusal aborts the whole draft without writes. A Pomodoro selector cannot choose a
wake destination: refuse a named-session suffix combined with `z:` and teach the bare
snooze form. A task's effective type still governs its later wake placement.

The `!` form above is a narrow snooze-row drop, not the existing `@route+id!` toggle. If
no active snooze row exists, return a teaching not-snoozed refusal; never invent a
completion or toggle fallback. Struck wake history remains history. Ordinary capture
behavior without these new forms stays unchanged.

## Phase grammar

Own `capture_language/` marker/model/strict/editor work, capture-parse output plumbing,
and parser tests. Add a dedicated small snooze grammar module if it makes the marker and
drop rules easier to share.

1. Carry an optional raw snooze specification and source range through
   `TerminalMarkers`, `ParsedCaptureText`, draft/item assembly, and the editor model.
   Keep lexing independent of filesystem state. Audit the parallel extraction path in
   `target_runs.rs` and rejection paths for project/task children so intent is neither
   lost nor applied to the wrong row. Do not add multi-target snooze or new
   project-creation operations beyond the defined task forms.
2. Admit existing-task `@route:id z:<when>` as the documented extra-token exception
   while retaining the body's new-task meaning. Apply the same safe exception to the
   existing `^route:id` alias; keep its first-token and whole-item rules. No authored
   children on an existing-task snooze. Parse the narrow `@route:id!` drop explicitly
   before ordinary target validation; do not relax block-ID validation globally.
3. Emit additive `snooze` spans for the full marker and a consistent incomplete/invalid
   diagnostic. Keep existing schema/grammar revisions unless a verified compatibility
   requirement forces otherwise. Add syntax-shape refusals for incompatible markers and
   unsupported kinds.
4. Until phase mutation consumes these intents, put one explicit execution guard at the
   capture dispatch boundary: a recognized snooze intent cannot fall through to ordinary
   writes. The next phase removes this transitional refusal. Do not add dead-code
   allowances or fake success.
5. Test strict/editor agreement for both marker orders, existing versus body-bearing
   forms, Unicode offsets, escaped/code/child text, duplicate and partial markers,
   incompatibilities, and the drop-versus-toggle distinction. Cover new-task routes with
   and without an authored ID.

Run focused `capture_language` unit tests, capture-parse CLI regressions, formatting,
and the repository lint gate. Record the intent fields and execution guard location in
the phase completion note for mutation.

## Phase mutation

Own a focused `capture/snooze.rs` adapter plus dispatch/result integration in
`capture/{mod,plan,batch,output,pomodoro_link}.rs` and CLI tests. Reuse the existing
capture transaction; never spawn `bob snooze` or use its disk write shell inside
capture.

1. Build `TaskFact` and the resolved `IdentityMap` from `CaptureBatchPlanner`'s current
   snapshots and existing dependency/type resolvers. Include read guards for all facts
   that decide eligibility. Earlier draft edits and newly minted IDs must be visible. Do
   not fall back to stale on-disk facts after staging a task or daily-note edit.
2. For existing-task snooze, resolve/validate the target and time first, then call
   `snooze::plan::plan_add`. Stage its postimages back into the batch. Ready becomes
   Next with the standard freshness stamp; Next and In Progress task lines remain
   byte-identical unless the user explicitly requested another already-supported field
   edit. Never drop dependencies or pull schedules forward to make snooze succeed.
3. For new-task snooze, reuse normal capture routing, authored-child handling, type and
   clock classification, ID collision checks, and task construction. Mint an absent ID
   from the current staged contents. Stage the task before invoking the pure add
   planner. Avoid making an intermediate ledger link or a spurious Pomodoro. Ensure the
   final task has Next and the same freshness semantics as an explicit commitment.
4. For ordinary link-now on a snoozed target, extract its active row's notes using
   shared row/subtree helpers, run ordinary link placement, attach the notes to the
   actual destination, then prune active rows by identity. Calling `plan_drop` alone
   would delete the row's notes; calling `plan_wake` would choose the wrong destination.
   Extend the pure core helper surface only where necessary to preserve those notes. No
   double note transfer, no duplicate link, and no wake receipt or notice.
5. Route snooze-drop through `plan_drop`; preserve the task lane and task line. Retime
   through `plan_add`, including re-snoozing a struck/woken row. Keep unrelated history
   and trailing sections byte-identical.
6. Preserve capture's settlement pass, final line positions, budget calculation,
   whole-draft atomic commit, and preview parity. Add an optional result object for
   snooze action, resolved time, task identity, and daily destination/row information;
   document its exact shape in the next phase. Omit it for unrelated items. Human/JSON
   previews must state snooze/drop accurately and must not report a nonexistent
   Pomodoro.
7. Remove the grammar phase's execution guard only when every accepted intent reaches a
   complete planner branch.

Add `tests/cli/capture/snooze.rs` and register it in the capture test module. Use
existing snooze vectors and fixed-clock CLI fixtures as the oracle. Cover Ready/Next/In
Progress, local and inherited Blocked refusal with unchanged prerequisite bytes,
Event/done/cancelled/non-task refusal, same-note target/daily coalescing, relative/DST
parsing through the core, retime, link-now into a running entry with nested notes, drop
and heading cleanup, duplicate identity spellings, struck history, and a full budget.
Verify mint collisions against earlier items and non-task anchors. Demonstrate dry-run
byte identity, a later invalid item rolling back an earlier snooze, sequential
operations on the same task, and stale preimage refusal. Run targeted capture/snooze
regressions and `just check`.

## Phase completion_docs

Own the snooze cursor-completion context, candidates, rendering, completion tests, and
final public documentation. Sequential dependency on mutation keeps the documented
response shapes aligned with the working adapter.

1. Add `CompletionContext::Snooze` in the shared lexical completion path and a candidate
   variant in `capture_complete/{model,engine,render}.rs`. Use the existing replacement
   protocol with a range over the value after `z:`; `replacement` is literal four-digit
   HHMM. Labels retain explanatory text, e.g. `15:30 · after Yogi interview` or
   `+1h · 16:00`. Document the context, range, candidate fields, and initial selection.
2. Read today's Events with `event_task::events_in_note`; never call Google Calendar or
   parse rendered agenda strings. Rank future event meeting ends chronologically, then
   resolved `+30m`, `+1h`, `+2h` in that order, then future times already used in
   today's Snoozed section. Deduplicate by resolved minute, retaining the earliest
   group's label and stable order. Ignore malformed rows, all-day/no-clock events, and
   invalid or non-future times. Relative suggestions near midnight must not overflow.
3. Use the core zoned clock/parser for validation and replacement values. Filter a typed
   prefix without destroying source-group ordering. Keep replacement byte ranges correct
   inside Unicode/multiline drafts, and do not claim a `z:` embedded in a URL, code, or
   child prose. Missing daily/Events/Snoozed data still permits valid relative
   suggestions.
4. Test event-first ordering with running and upcoming events, ties, cancelled/closed
   event handling consistent with the Events model, offsets/partial values, reused
   times, midnight filtering, and accepting a candidate into a draft that
   `capture-parse` and dry-run both accept.
5. Update `docs/capture.md`: marker table, time/refusal section, narrow existing-task
   exception, new-task identity behavior, drop/link-now, spans, completion vocabulary,
   and response/preview examples. Update `docs/reference/capture-syntax.md` and the
   capture entries in `docs/reference/json-outputs.md` as needed. Link to
   `docs/snooze.md` rather than duplicating its entire clock and row contract. Keep the
   agenda subsection edits from parallel outer phase `bob-cli-q5.2`.
6. Run `just docs-gen`, `just docs-check`, and `just check`. Use `/sase_monitor` for
   long checks and follow its staging/finalization workflow; do not end a turn with an
   unhanded-off live command.

## Land verification and bead completion

The child land agent integrates all three phases, removes every transitional guard, and
validates the behavior table against a disposable fixture vault. Check that normal
link-now is still running-first, automatic wake remains never-running, and snoozed rows
are absent from current Today counts. Do not implement the outer planned-today, plugin,
or Mac phases here.

Require passing `just check` and `just docs-check` on the final integrated tree. Reuse
successful unchanged-tree evidence; do not repeat broad tests without a new change or
unresolved failure. If a failure reproduces identically on the clean base, record it on
the assigned phase as
`PROPOSED FOLLOW-UP: <summary — reproduction and existing tracker if any>`; that
pre-existing failure does not prevent closing the completed scope. Do not create task
beads.

Before each phase closes, run `sase bead epic-symbols <assigned-id>` and resolve/re-key
every remaining allowance to a still-open owning bead. Before the land agent closes
`bob-cli-q5.3`, run `sase bead epic-symbols bob-cli-q5.3` again. After its child phases
and child epic are complete, close only the original phase with
`sase bead close bob-cli-q5.3 --note "<implemented behavior and concrete verification>"`.
Do not close `bob-cli-q5`, do not reopen the in-progress assigned phase, and never
hand-edit bead status. Finish each root turn through `/sase_final`.
