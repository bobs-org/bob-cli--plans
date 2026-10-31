---
tier: epic
title: Complete project-note task offers for bob-cli-ex.2
goal: Capture @route:id@ creates a project note and offers its eligible tasks through
  the existing pull-in protocol, with selected tasks identical to authored colon bullets
  and complete grammar, documentation, and verification coverage.
phases:
- id: offer_engine
  title: Plan project task offers through the existing colon-bullet writer
  size: medium
  depends_on: []
  description: 'offer_engine: extend rendered project tasks, shared placement, project-note
    output, selection, and draft-level pull-in plumbing. Exercise ProjectNoteForm::Offer
    through internal planner tests while the public colon-seal grammar remains disabled.'
- id: offer_language
  title: Enable the colon seal and verify the public capture protocol
  size: medium
  depends_on:
  - offer_engine
  description: 'offer_language: enable @route:id@ in execution, editor parsing, completion,
    and sealed separator rewrites. Add CLI coverage of selections, destinations, atomic
    failures, and mixed-draft offers, and update CLI help and its generated reference
    pages.'
- id: offer_contracts
  title: Publish project-offer contracts and finish acceptance coverage
  size: medium
  depends_on:
  - offer_language
  description: 'offer_contracts: update capture, project, and successor documentation
    and only the two glossary strands authorized by the parent epic. Verify equivalence
    to authored colon bullets and the remaining acceptance matrix, then provide land
    evidence.'
proposed_by: bbugyi200.apollo.bob-cli-ex.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ex.2
create_time: 2026-10-31 13:56:42
status: done
bead_id: bob-cli-ex.2.1
---

- **PROMPT:** [prompts/202610/project_note_offer_form.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/project_note_offer_form.md)
- **PARENT:** [202610/project_note_seals.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/project_note_seals.md)
- **BEAD:** [bob-cli-ex.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ex/bob-cli-ex.2.1.md)

# Complete bob-cli-ex.2: offer a new project's tasks with @route:id@

## Scope and settled decisions

This is a child epic for phase `bob-cli-ex.2` of `bob-cli-ex`, implementing the
`offer_form` portion of `plan:202610/project_note_seals.md`. Read the assigned bead with
`sase bead read` and that design with `sase artifact read` before implementation. The
parent design's shared contracts remain authoritative; this plan supplies the
implementation sequence and verification details.

The final parent decisions are inherited without reopening them:

- `initial_marks = none`: every offer starts unselected; the Mac card implementation
  belongs to parent phase `bob-cli-ex.4`.
- `memory_glossary = yes`: the authorized notes are exactly `glossary:successor-offer`
  and `glossary:task-successor`. Their changes are specified below and require
  `/sase_memory_write` followed by `sase memory init`.

The work stays in bob-cli. The Mac and plugin implementations belong to their existing
parent-epic phases. Preserve the release-train policy: do not change package versions or
`CHANGELOG.md`.

## Baseline and sizing evidence

At investigation HEAD `76f23471`
(`feat(capture)!: seal project notes with @ instead of +`), prerequisite `bob-cli-ex.1`
is closed and its implementation is present. `bob-cli-ex.2` is in progress and has no
prior implementation/remaining-work notes. Its epic-symbol audit currently reports no
entries. Recheck actual checkout state before starting; do not redo the caret migration.

Already available:

- `ProjectNoteForm::{Create, Offer}`, the span modifier, and the reserved
  `ProjectNoteOfferMarker` exist, but only Create is produced by the grammar.
- `capture/project_note.rs` renders a note, stages it, and links explicit ` :id`
  children using type-aware placement and `apply_new_task` retyping.
- `capture/pull_in.rs::plan_with_pull_in` already handles None/All/Keys, numeric probes,
  unknown picks, and `--require-pull-in-choice` without writes.

Still absent: colon-sealed grammar, project successor rows and folds, task selection
inside the project planner, and inclusion of those rows in the offer/net-pass universe.
This crosses the execution/editor grammars, transaction planner, and existing
completion-offer integration. Three serial medium phases give each coder a bounded
surface. The engine lands before public activation, so no intermediate public form
silently creates a project while ignoring picks.

## End-to-end behavior

Given an area `cash.md`, a daily note with an open Pomodoro, and this draft:

```text
Finish the exit packet @cash:goog-exit@
- Draft the memo
- Collect paperwork ^paperwork
- Call the bank :call-bank
- Private reminder #hide
```

Capture creates `cash_goog_exit.md`. The explicit `:call-bank` behaves exactly as today.
The first two task rows are successor offers in note order, initially unselected; the
hidden row is folded. Default CLI submission and `-p none` create the note with only
explicit colon-bullet links. `-p 1`, `-p <key>`, or `-p all` select additional offers.
`--require-pull-in-choice` with no answer refuses when offers exist and writes neither
the project note nor the daily note.

Every picked row must be rendered and placed exactly as an authored ` :<id>` bullet
under `@cash^goog-exit@`. With fixed time, config, and vault inputs, the resulting files
must compare byte for byte. Created tasks receive no `[fresh::]` stamp. Picks use
existing colon semantics, including scheduled status, explicit `#name` retyping, and
per-task type placement. The lifecycle `^prj` and generated placeholder are never
offered or linked.

## Phase offer_engine

Primary files: `src/native/capture_project_note.rs`,
`src/native/capture/project_note.rs`, `capture/output.rs`, `capture/pull_in.rs`,
`capture/task_complete.rs`, and `task_complete/project_pull_in.rs`.

1. Extend the pure renderer's metadata beyond `named_tasks`. Return task rows with final
   line number, status, optional ID, text, authored-link flag, hidden flag, placeholder
   flag, and enough source identity to update the corresponding authored sub-bullet.
   Include the actual rendered Tasks rows; exclude section headings and note-only
   bullets. Keep nested context and merged TASKS-section behavior intact. Use parsed
   status settings and the existing hide-tag predicate, not substring heuristics. Cover
   source mapping when sections reorder output, duplicate descriptions, custom closed
   symbols, and generated placeholders.
2. Branch on `ProjectNoteForm`. Preserve Create serialization and behavior. For Offer,
   pre-render once without changing selection. Eligible statuses are open
   Ready/Next/Pending (` `, `*`, `/`); omit blocked, terminal, unknown, lifecycle, and
   placeholder rows. Explicit ` :` children count as `already_planned`; hidden tasks
   count as `hidden`. A future project schedule yields the existing `project_schedule`
   link guard and no offers, including when there is no explicit colon child.
3. Freeze offer identities from the pre-pick render using
   `note_tasks::stable_task_key(relative_basename, block_id, description, global_filter, digest_ordinal)`.
   Use the normal cleaned-description ordinal convention so repeated ID-less
   descriptions stay distinct. Keep these keys after picks mint IDs, alter statuses, or
   retype tasks. Number a standalone offer set `1..k` in note order and assign order
   keys `(1, (usize::MAX, usize::MAX), basename, line)`.
4. Read `request.selection`. For each selected row, retain its ID or mint one with
   `capture_block_ids::mint_block_id`, reserving every authored ID, `prj`, and prior
   mints. Set that source bullet's task marker to `link = true`, re-render, and feed it
   through the existing colon-bullet writer. Do not call the Done-successor mutation
   engine: its freshness writes do not apply to newly created tasks.
5. Extract the existing project placement loop into a shared helper returning each
   task's actual destination and insertion metadata. Actual selected/explicit links use
   normal planner staging. Unselected offer destinations use the same helper on scratch
   day text; discard scratch insertions, created Pomodoros, retyping, and IDs. An ID
   allocated only to probe placement must not appear as a committed ID. Respect
   sequential placement and type capacity just as the real link loop does. Keep the
   staged residence visible to `placement_for_task`.
6. Read the day note only if an explicit ` :` bullet exists or there is an eligible
   offer; missing-day and duplicate-link errors keep existing transactional behavior. A
   named Offer form bypasses the old parser/planner check for an explicit colon child;
   instead refuse `#name` only if there is neither an open task to offer nor an explicit
   colon task. Preserve the blocked-project guard.
7. Extend `ProjectNoteSummary` with optional `form: "offer"` and
   `successors: Vec<TaskCompleteUnblockedJson>` emitted for Offer, including `[]`. Omit
   both fields for Create. Store project folds internally for aggregation. Add
   `project_created` beside the existing trigger constants. Each offer row has its
   pre-pick previous status, final status/line/ID, `inbox: false`,
   `hide_removed: false`, and `unblocked_by` naming the new basename's `^prj` and
   project task text. `offer.group` is `project`; `offer.project` is the basename.
   Destinations use existing `typed`, `created`, or `implicit` vocabulary, with fallback
   true for implicit placement. Picked rows set `selected: true`, carry a normal
   successor `link`, and also appear in `project_note.task_links`.
8. Include project rows in `all_unblocked`, `unblocked_vecs_mut`, `all_project_folds`,
   and `completion_summary_count`. Reuse existing draft numbering, refusal payloads,
   pick validation, pull-in totals, and fold output (`sub_projects: 0`). Print
   `New project <basename>` for these human offer rows.
9. Integrate with `apply_successor_net_post_pass`: later capture items that close,
   block, or separately link a new task suppress its offer; selected rows survive only
   if the final task and link survive. Check ID-less lookup and final line positions
   explicitly, since `successor_task_lookup` currently falls back to `row.line`. Keep
   existing Done/relation/Pomodoro-close behavior unchanged.

Test the engine directly using a constructed parsed ProjectNote Offer request; do not
enable syntax just to exercise this phase. Cover rendering, stable keys,
all/none/subset, mint collisions, scratch placement isolation, omitted Create fields,
folds, and offer iterator traversal. Run focused unit tests and existing
project-note/successor CLI regression tests, formatting, and lints. Any temporary
epic-symbol exemption must be keyed to a still-open dependent phase and removed by its
owner; do not introduce general unused-code allowances.

## Phase offer_language

Primary files:
`src/native/capture_language/{tokens,item,line,markers, editor_classify,editor_model,completion,rewrite,type_marker,model}.rs`,
`src/native/capture_block_ids.rs`, public capture protocol/help modules, and their
unit/CLI tests. Inspect the actual dispatch; line references in the parent design
predate the caret migration.

1. Parse `@r:id@`, `@r:id@#name`, and `@r:id@#"Lit Name"` as ProjectNote Offer before
   ordinary colon-link parsing/solo-link conversion. Missing body remains an error;
   route-less `@:id@` is an editor route need and an execution route error. Missing ID
   keeps the family's existing wording. Start/close suffixes are refused through the
   existing project-note errors. Keep ordinary unsealed `@r:id`, typed tasks, and the
   caret form unchanged.
2. Emit mode `pomodoro_project_note` for every Offer form. Its route and ID retain
   `pomodoro_route` / `pomodoro_block_id`, the colon gets `project_note_separator`, and
   the seal gets `project_note_offer_marker`. All token spans and missing-route/ID
   placeholders carry `project_note: "offer"`. `pomodoro_name` keeps theme metadata
   without the modifier. A completed seal offers no picker/needs; a trailing `#` opens
   name completion.
3. Update completion and type-marker handling so either sealed separator has
   `block_id.intent: "project_note"`, excludes the seal from replacement text, and
   includes it in the marker token range. Unsealed intents stay unchanged.
4. Extend separator rewrites: `@r^id@:` becomes `@r:id@`; `@r:id@^` becomes `@r^id@`.
   Validate the stripped candidate's selected marker, matching project mode, and empty
   needs. Same separator, `+` after a seal, named forms, or other suffixes stay literal.
   Preserve the landed plain `@r^id+` editor toggle to `@r+id`; direct execution of
   retired project spellings still refuses them.
5. Complete the parent's diagnostic table. Retired caret and colon `+` spellings teach
   both sealed alternatives, preserve any name suffix, and include the sub-bullet hint
   only without a name. Misordered `#name@`, junk after the seal, `@@` global project
   destinations, and `@r+id@` get their specified teaching diagnostics. Caret link items
   do not acquire the new form.
6. Update human CLI help/examples in capture, capture-parse, capture-complete, and
   capture-rewrite where affected; regenerate CLI reference goldens with
   `just docs-gen`. Remove only the now-obsolete reserved-Offer allowances.

Add CLI tests in `tests/cli/capture/project_note.rs` or a focused sibling module. Use
isolated fixtures and deterministic `BOB_NOW`. Assert preview keys equal submit keys
even after minting/retyping, all/none/numeric/key picks, repeated descriptions,
unknown/out-of-range picks, and choice-required flat `offers` payloads. Assert no file
writes on every refusal and rollback if a later batch item fails.

Exercise named and literal destinations, per-task types, missing daily files, strict
placement refusals, and no real placeholder Pomodoro until a task actually lands there.
Cover mixed numbering with a managed `!` completion and suppression when a later item
closes or links a just-created task. Add full valid/invalid grammar, span byte ranges
(including non-ASCII body offsets), completion, and rewrite matrices. Run
capture-focused unit and CLI suites plus lint/format checks.

## Phase offer_contracts

1. Add “Offer a new project's tasks” under project notes in `docs/capture.md`, with a
   realistic draft, preview JSON, `-p 1,3` / `all` / `none`, choice-required flow,
   explicit colon/caret children, hidden folds, blocked projects, and creation's
   freshness exception. Update span kinds/modifiers, modes, separator toggles, pull-in
   locations, and selection text. Preserve current published anchors.
2. Update `docs/reference/capture-syntax.md`, `docs/task-dependencies.md` sections
   12.1/12.5/12.14, and `docs/projects.md` promotion-successor discussion. Document the
   additive `project_note.form`/`successors`, `project_created`, project fold, stable
   keys, and dual representation of selected rows in `task_links`; clients must not
   count a pick twice. Do not describe the separate Mac implementation as complete.
3. Apply the inherited `memory_glossary = yes` exactly. Audited reads must precede edits
   through `/sase_memory_read`; use `/sase_memory_write` before mutation. In
   `glossary:successor-offer`, add sealed project creation as an offer source, exclude
   `^prj`, and distinguish existing-task freshness stamps from same-capture creation.
   Retain its decisions link. In `glossary:task-successor`, add creation's tasks as
   successors of the new `^prj`, correct the freshness statement, and update its
   summary. These are the only canonical memory notes authorized by the parent decision.
   Run `sase memory init` and `just docs-gen` for generated instructions and glossary
   docs.
4. Audit the parent's full acceptance matrix against tests actually landed. Fill gaps
   rather than duplicating tests. The central acceptance test runs Offer `-p all` and
   Create with the equivalent explicit colon bullets in separate identical scratch
   vaults and compares every written file byte for byte. Cover implicit, explicit-name,
   typed, and scheduled cases (picks only when offers are eligible). Check absence of
   `[fresh::]`, correct final statuses, stable IDs, unselected rows,
   closed/blocked/hidden exclusions, `^prj`, and the placeholder.
5. Run the required verification: `just fmt`, `just lint`,
   `cargo test --test cli capture`, `just test`, and `just docs-check`. Use
   `/sase_monitor` for long commands, preserving exact results and remaining work in the
   handoff. Do not repeat equivalent broad checks without new changes or unresolved
   failures. A failure reproduced identically on the clean base must be recorded as
   `PROPOSED FOLLOW-UP:` on the assigned phase (include an existing tracker ID if
   known); it does not keep completed work open.

## Land and completion ownership

Child phase workers close only their assigned child phase, after its epic-symbol audit.
They never close `bob-cli-ex.2` or `bob-cli-ex`. Do not create standalone follow-up
beads; append `PROPOSED FOLLOW-UP:` notes for land-agent triage.

The child epic's land agent verifies all three phases and the final combined tree,
records commits/check evidence on `bob-cli-ex.2`, and performs its remaining symbol
audit with `sase bead epic-symbols bob-cli-ex.2`. Resolve each listed symbol, or re-key
a genuinely deferred Justfile exemption to a still-open owner before closing. Only once
this complete scope is verified, close the requested phase with
`sase bead close bob-cli-ex.2 --note "<implemented behavior and verification>"`. Leave
the parent `bob-cli-ex` and all ancestor plan beads to their own land agents. Publish
the additive CLI contract evidence so parent phase `bob-cli-ex.4` can consume it; no
manual peer messaging is required.
