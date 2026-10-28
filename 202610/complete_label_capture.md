---
tier: epic
title: Complete relation sources through the shared capture Done path
goal:
  Finish bob-cli-du.2 by composing complete-label relation commits with existing task
  completion, atomic capture writes, successor choice, and additive output.
phases:
  - id: close-policy
    title: Parse closing-label policy and expose complete verdicts
    size: medium
    depends_on: []
    description:
      "close-policy: share the cancel/complete label parser, drop overlapping keys from
      both lists, add CloseEffect and complete verdicts, and exercise the existing
      config, key, and rule vectors without enabling relation-triggered completion."
  - id: shared-completion
    title: Extract reusable capture completion and aggregate nested outcomes
    size: medium
    depends_on:
      - close-policy
    description:
      "shared-completion: extract complete_existing_task from the whole-item ! adapter,
      add omitted-by-default relation completion output, and generalize successor
      traversal, numbering, net filtering, and detail rendering while preserving
      existing capture behavior."
  - id: relation-composition
    title: Complete relation sources in the staged capture transaction
    size: medium
    depends_on:
      - shared-completion
    description:
      "relation-composition: share closing-edge collection with cancel, enforce
      same-source and cross-item refusals, compose relation writes with
      complete_existing_task, and report nested completion, tracking references,
      warnings, human rows, and successor choices."
  - id: capture-acceptance
    title: Verify capture edge cases and update implemented contract markers
    size: medium
    depends_on:
      - relation-composition
    description:
      "capture-acceptance: finish the scratch-vault acceptance matrix, prove no-config
      and existing !/cancel compatibility, verify rollback and mixed successor
      selection, update capture contract documentation, and prepare evidence for the
      child epic land agent."
proposed_by: bbugyi200.apollo.bob-cli-du.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-du.2
create_time: 2026-10-28 15:26:08
status: wip
---

- **PROMPT:**
  [prompts/202610/complete_label_capture.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/complete_label_capture.md)
- **PARENT:**
  [202610/relation_complete_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_complete_labels.md)

# Complete labels in capture: implementation of bob-cli-du.2

## Outcome and ownership

With `relations.complete_labels: [implemented by, verified by]`, capturing
`@tasks+source ~tasks:target|"implemented by"` completes the existing source. Capturing
`Build the feature ~~reading:source|"implemented by"` creates the feature task and
completes the picked reading task. Both compose the relation write with exactly the
existing `!note:id` completion behavior, inside capture's guarded all-or-nothing batch.
Successors recover automatically and are offered for explicit selection.

This is a child implementation epic for **bob-cli-du.2**, phase `complete-capture` of
`plan:202610/relation_complete_labels.md`. It covers bob-cli only. The child epic's land
agent verifies and closes bob-cli-du.2 after all child phases have landed; child phase
workers close only their assigned child bead. Nobody in this child epic closes
**bob-cli-du** or any ancestor above bob-cli-du.2. Do not manually set bead status.

This division is warranted by four separable obligations: policy/vector compatibility,
extraction of the large existing completion adapter, transaction composition, and
capture acceptance. All dependencies are explicit and serial to avoid overlapping
changes to capture's shared result and planner types. Each phase has a bounded direct
implementation scope; no further design handoff is required.

## Verified starting point

Read the assigned bead and this plan again before changing code; skip work already
landed. At planning time bob-cli-du.2 was in progress with no earlier-work notes, and
`sase bead epic-symbols bob-cli-du.2` reported no entries. The work is unfinished, so no
reopen or close was performed.

Commit `847a3c53` landed prerequisite bob-cli-du.1: §18 and RV10 in
`docs/task-relations.md`, complete-label config and protocol documentation, the inverse
pairs `implemented by` / `implements` and `verified by` / `verifies`, and
`tests/fixtures/relation_complete/vectors.json`. Do not redo these. The fixture has 17
Done, 39 rule, 10 key, and 13 config cases. Existing Rust coverage checks Done bytes and
section presence; policy/config coverage remains to be added.

Relevant implementation seams:

- `src/native/config/relations.rs` contains cancel-only policy and parsing. Several
  capture-complete, parse, list, and rename callers construct `RelationsConfig`
  explicitly, so adding a field requires mechanical updates even outside capture.
- `src/native/task_cancel/verdicts.rs` already separates source eligibility from
  explicit relation-policy eligibility. Complete labels need the corresponding public
  crate-local API in `task_complete::verdicts` for later surface phases.
- `src/native/capture/task_complete.rs::plan_task_complete_item` resolves the token,
  validates the root, performs tree completion, successor planning and ledger
  retirement, stages files, and builds the summary and tracking references.
- `capture/plan.rs` stages `plan_relation_writes` before `plan_relation_cancels`.
  `capture/relations_cancel.rs` collects added/relabeled edges, resolves source
  identity, groups them, checks conflicts, and composes the cancel engine.
- `capture/pull_in.rs` assumes at most one completion summary per capture item.
  `number_draft_offers` preserves engine numbering when there is one item: that is
  insufficient when one relation item completes multiple sources.
- `capture/task_complete.rs` owns the final successor filter and surviving-link set;
  `capture/output.rs` has another completion-row accessor and the human renderer.
  `capture/inbox_route.rs` and block/open-note reporting must see nested completions.
- `capture/dependencies.rs` caches cancel policy; `relations_mirror.rs` currently
  filters warnings by `relations.cancel_labels`, which would miss an overlap warning.

## Inherited decisions and boundaries

Read `docs/task-relations.md` §§17–18, `docs/capture.md`'s `!` and relation sections,
and `docs/task-dependencies.md` §12 before implementation. The audited parent plan and
prior design are `plan:202610/relation_complete_labels.md` and
`plan:202610/relation_cancel_labels.md`; consume them through `sase artifact read`. Read
`docs.md`, `cli_rules.md`, and relevant reference memory through `/sase_memory_read`,
never directly from memory files.

The parent decisions are final:

- **offer_successors=yes.** Complete-label closures participate in the existing offer
  set, `-p/--pull-in`, and `--require-pull-in-choice`. No separate choice protocol.
- **memory_glossary_relation=yes** authorizes only `glossary:task-relation`, and that
  change belongs to the outer epic's rollout. This child epic edits no memory.
- **memory_decision_record=no.** Add no decision record and reopen no design choice.

Other outer phases own capture-complete cues, capture-parse flags, relations list,
rename and reverse migration, plugins, Mac Capture, and live config rollout. Limit edits
in their modules to compilation adjustments required by shared type changes. Do not
enable labels in live config, touch the live vault, edit versions/CHANGELOG, or add a
new CLI option. All tests use scratch vaults and isolated config.

## Required behavior

Only an explicit added edge or relabel into an effective complete key completes its
source. An unchanged re-add, inverse label, relabel away, unlink, reconcile, rename,
hand edit, config change, and sync do not complete or reopen anything. Hiding a label
and typing it with `||` do not disable its effect. The picked target stays unchanged.

The completable status set is Ready `[ ]`, Next `[*]`, In Progress `[/]`, and Blocked
`[?]`. A `^prj` source completes by `!`'s rules; ordinary residents stay open, with the
project count in the message. Closed/custom non-completable statuses only get the link
and `complete_effect: already_closed`, preserving their status and date. Never turn
Cancelled into Done. New sources and recurring open sources are refused.

The shared Done path closes eligible embedded subtasks, retires today's scoped Task
Links, recovers dependents with the existing stamps, and offers successors. Preserve the
engine's actual order (successor planning uses pre-retirement anchors). Do not add a
Work Log, Cancel Log, source freshness stamp, status-group move, alternate Done
rewriter, or independent vault write.

Use canonical codes and §18 CP6 copy:

- `relation_completes_new_task`: names the new source and suggests `~~`.
- `relation_completes_recurring_task`: asks for completion in Obsidian so Tasks can
  create the next occurrence.
- `relation_complete_conflicts_with_item_action`: another action on the same source.
- `relation_close_effects_conflict`:
  `“{S}” can't be both cancelled and completed in one commit`.

The permitted relation-only existing owner syntax is `@note+id`. Distinguish it from the
Pomodoro-linking `@route:id` syntax; do not accidentally reject the CP3 positive case.
Preserve established grammar refusals for whole-item forms while checking all reachable
source conflicts with `!`, `=`, `:`, `@route:id`, `&`, and `*`. For `~~`, an action on
the new target is not an action on the picked source. Identity checks use resolved note
path and block ID, not a basename or token spelling.

## Phase close-policy

1. Generalize the parser in `config/relations.rs` for both lists, preserving cancel
   normalization, spelling, order, duplicate collapse, and existing diagnostics. Add
   `complete_labels`, `CloseEffect { Cancel, Complete }`, `close_effect(key)`, and an
   inverse lookup that returns the configured label together with its effect. Keep
   existing cancel lookup methods as compatible views.
2. Missing/null/empty complete lists are valid empty policy. Non-lists warn and complete
   nothing; invalid entries warn and are skipped. Neither list changes
   `linked_from_lines` or turns a list-entry problem into a `ConfigError`.
3. Remove normalized keys appearing in both lists from both effective lists. Emit one
   warning per overlapping key using CP2's exact message. Inverse lookup and cancel
   behavior must use the filtered sets. Do not silently retain one side.
4. Add `task_complete::verdicts::complete_verdict(source_line, settings, is_new_task)`
   with `Completes { previous_status, project }`, `AlreadyClosed { status }`,
   `Recurring`, and `NewTask`. Reuse existing status/recurrence semantics. Expose the
   policy layer needed to check action/trigger/key/conflicts without performing writes;
   share existing neutral relation enums/helpers with cancel where that avoids drift.
5. Extend fixture tests so config and key cases run against production APIs; run rule
   cases through real eligibility/policy functions, delegating cancel cases to existing
   cancel policy. Do not implement the expected decision tree solely inside tests.
   Preserve the generated Done fixture bytes and existing cancel vectors.
6. Mechanically update all `RelationsConfig` constructors. Do not add new command
   presentation in the outer `complete-surfaces` phase's files.

Acceptance: config/key/rule cases pass, existing task-cancel/config tests pass, Done
vectors still pass, and all targets compile. This phase enables no complete-label
capture writes.

## Phase shared-completion

1. Extract `complete_existing_task` after `!` token resolution. Its input identifies an
   existing source, the raw action token, the request/selection, date, planner, and
   dependency context. Its result exposes the existing `TaskCompleteSummaryJson`, source
   transition details, daily-note effect, task refs, and Pomodoro refs needed by
   adapters. The function owns all completion validation, tree/successor/retirement
   execution, staging, and summary construction. Keep `!` token parsing and its
   whole-item conflict check in `plan_task_complete_item`.
2. Preserve `!`'s already-Done no-op, Cancelled/error behavior, root validation order,
   missing-day behavior, read gates, messages, and JSON. The relation adapter will
   preflight its different already-closed semantics before calling this function. Do not
   simulate capture input, spawn another `bob`, commit files here, or copy the existing
   completion body into a second writer.
3. Add CP7 result plumbing with empty fields omitted: `relation_completes[]` contains
   `{source: {path,line,block_id,text}, target: {path,block_id,text}, labels, action, previous_status, previous_status_name, project_open_tasks?, task_complete, message}`.
   `complete_effect` is optional on relation updates. Keep schema version 1. Every
   existing result constructor gets an empty collection. Only actually completed sources
   will be serialized.
4. Generalize immutable and mutable traversal of unblocked rows/project folds across
   whole-item `task_complete`, `pomodoro_close`, and all nested relation completions.
   Use it consistently in numbering, number-to-key probes, selection validation,
   required-choice errors, top-level `pull_in`, inbox routing, final successor
   filtering, surviving links, and human offer summaries. Search every summary consumer,
   including `capture/inbox_route.rs`, rather than updating one accessor.
5. Fix single-item numbering so multiple source summaries get one stable ordered number
   space. Preserve existing single-engine numbering and tie order for old captures.
   Numeric probe and selected rerun must agree, keys must remain stable, and final
   filtering must run before numbering. Include project fold aggregation.
6. Extract the existing `!` detail rendering for reuse by relation rows. Keep the old
   `!` heading and output byte-identical; relation output will supply its own heading
   and reuse subtasks, ledger, unblocked, still-blocked, and offer details.

Acceptance: existing `!`, `x!`, successor, inbox-route, and cancel suites pass
unchanged. Add focused internal coverage for multiple nested summaries, final filtering,
and number selection; this protects the aggregate behavior before the writer is enabled.
New result fields remain absent on all old capture paths.

## Phase relation-composition

1. Add `capture/relations_complete.rs` and a small shared closing-edge coordinator
   beside `relations_cancel.rs`. Share added/relabeled edge collection, ordered grouping
   by resolved source identity, and source-action conflict analysis. Preserve cancel's
   existing messages, writes, and output. Only share mechanics that are the same; cancel
   still uses its own retirement/recovery engine.
2. Cache the effective close policy once per capture batch. Make complete, cancel,
   overlap, and mirror warnings visible once through the established warning path. The
   existing cancel-only substring filter must not hide overlap warnings. Keep old
   no-complete-config output stable, including cancel diagnostics.
3. After relation LINKS/mirror staging, preflight every candidate close before running
   either effect for that item. Check new/recurring/closed source verdicts, action
   conflicts, and opposite effects on the same source. Group multiple complete edges on
   one source in authored order and call the shared Done path once.
4. Track closing-label intents for the entire draft, so a cancel then complete (or the
   reverse) on the same source in separate items refuses the whole batch. A later staged
   closed status must not mask the contradiction as `already_closed`. Ignore unchanged
   edges, unconfigured keys, and labels dropped by the both-lists rule. The registry is
   per planning pass; numeric offer probes start fresh. Add tests in both item orders
   and for distinct sources, which are allowed.
5. For eligible sources call `complete_existing_task` against the staged relation
   postimages. For closed sources keep only the relation edit and optional
   `complete_effect: already_closed`; do not call `!`'s Cancelled refusal path. Merge
   every returned task/Pomodoro reference and day-file effect into the parent item.
   Re-resolve final source lines and refresh any enclosing item's task/status fields
   after tree closure or daily-note retirement shifts lines.
6. Fill one `relation_completes` row per source actually completed; collect labels in
   authored order and set `task_complete.raw` to the first triggering relation token.
   Each triggering edge gets `complete_effect: completed`. Use the exact existing `!`
   summary object, not a reduced or independently assembled one.
7. Add structured `relation_completed_source` warnings with label and CP6 message. Print
   the bob-authored ✓ row once per source, then shared completion detail rows; suppress
   duplicate warning rendering. Dry runs use the specified would-forms. Render
   already-closed outcomes as link-only feedback without adding completion rows or
   offers. Preserve cancel rows in mixed captures.
8. Add focused scratch-vault tests for each positive direction, all four new refusal
   codes with unchanged vault bytes, same-source deduplication, closed sources, a mixed
   `!`/relation offer draft, numeric/key/none/all selections, and required-choice
   gating. Include a single relation item closing two different sources with offers.

Acceptance: both directions compose through the shared Done function, offers are fully
selectable, opposite effects refuse atomically, and no old path regresses.

## Phase capture-acceptance

Use `tests/cli/capture/relations_cancel.rs`'s isolated-vault and dry/real comparison
pattern for a new registered `relations_complete` suite, factoring fixture helpers only
where useful. Compare parsed JSON while normalizing only `dry_run` and the specified
would-forms; do not mask source identity, counts, selections, or ordering.

Complete the matrix with assertions on observable output and actual vault bytes:

- Each open status including Blocked; closed Done/Cancelled/custom status with date
  preservation; recurring variants; ordinary and `#ref` sources; a `^prj` source with
  untouched ordinary residents; embedded subtasks following `!` rules.
- Added/relabel-into; unchanged retry (including a manually reopened source), relabel
  away and inverse labels; two complete labels on one source; multiple sources;
  distinct-source mixed cancel and complete; same-source mixed effects within one item
  and across items in both orders; false positives when a modifier targets T.
- Same-note/cross-note source, source in today's daily note, missing daily note, mirror
  disabled, archived/Cancelled target, hidden typed label and `||`, CRLF, and New York
  midnight with a controlled clock and config timezone.
- The same fixture executed via `!` and via relation completion yields matching Done
  bytes, tree/ledger/recovery/offer results, allowing only the extra relation edits and
  the documented raw token/outer presentation differences.
- A later item observes staged completion; final successor filtering removes rows whose
  tasks were later closed/reblocked/linked/dropped. Numeric probe/rerun agrees for one
  item with multiple completed sources and for mixed `!`, `x!`, and relation closes.
  Check no choice, none, all, numeric, stable-key, stale/invalid picks, and
  `--require-pull-in-choice`; unpicked recovery must never acquire a Task Link.
- Tracking/output includes nested completed and recovered task blocks, Pomodoro
  retirement, successor-link reasons, and open-note destinations. Exercise inbox routing
  when a recovered candidate is in an inbox.
- A late invalid item, stale disk preimage, and existing injected commit-failure
  mechanism leave/restore all notes and any staged one-off label journal changes. Extend
  the existing transactional harness instead of adding a production failure hook. Dry
  run creates or writes nothing.
- Reconcile/hand edits/config changes never infer Done. Existing no-config capture
  suites pass unchanged; compare representative human/JSON bytes against the clean
  pre-change revision with controlled paths/clock, and assert new fields are absent.
  Include cancel-only config to prove the shipped cancel path remains unchanged.

Update `docs/capture.md` and `docs/reference/json-outputs.md` with the implemented
capture behavior, refusal codes, nested summary, and shared offers. Update only the
matching capture/Rust status markers in §18 and CL2's both-lists clause. Mixed markers
must continue naming unfinished nav/surface phases; do not claim those are shipped.
`docs/reference/configuration.md` already documents the new key; correct only details
the actual implementation requires. No generated glossary or memory edits.

## Validation and completion

Each implementation phase runs the relevant focused Rust and CLI suites. Follow the
repository's test target names (`Cargo.toml` and existing harness registration), and run
formatting and compile/lint checks appropriate to changed types. The acceptance phase
runs `just check` and `just docs-check` after its final edits. Use `/sase_monitor` for
commands expected to outlive the turn; a yielded exec session is not completion.

Investigate failures against the clean base using a sanctioned isolated checkout or
worktree within the workspace. A failure reproduced identically there is recorded as
`PROPOSED FOLLOW-UP: <summary — commands, baseline SHA, and evidence>` on the assigned
bead, citing an existing task if known. Such failures do not keep a completed phase
open. Phase workers create no beads and do not expand this scope to unrelated fixes.

Before closing each assigned phase, run `sase bead epic-symbols <assigned-id>` and
resolve or re-key every remaining exception to a still-open owner. Close only that phase
with `sase bead close <assigned-id> --note "<what was verified>"`.

The child epic land agent checks all child work has landed, confirms the shared
completion call path and no missing result consumers, reviews the test/documentation
evidence, and checks any integration changes. Record the implementation commits and
validation results on bob-cli-du.2. Settle the child epic itself through its normal land
workflow. Run `sase bead epic-symbols bob-cli-du.2`, remove or re-key leftovers, then
close **bob-cli-du.2** with a verification note. That is the only outer bead this land
agent may close. This closeout is explicitly assigned to the child epic land agent,
never a child phase worker. Leave bob-cli-du and all higher ancestors to their own land
agents. Follow SASE's normal final declaration for the worker turn.
