---
tier: epic
title: Complete capture project-note move execution
goal: Project-note capture moves complete parent task blocks atomically, links and
  offers them in draft order, and reports accurate previews and CLI output for bob-cli-gl.5.
phases:
- id: move_execution
  title: Execute ordered parent moves with atomic batch safety
  size: medium
  depends_on: []
  description: 'move_execution: Resolve caret moves against staged parents, render
    task slots, execute one batch move per item, emit move JSON, and enforce all vault
    refusals. Correct chained destination-before-source ordering and cover the move-only
    MV vectors.'
- id: move_links_offers
  title: Link moved tasks and integrate project offers
  size: medium
  depends_on:
  - move_execution
  description: 'move_links_offers: Execute colon moves and selected moved offers through
    existing-task link semantics in draft order with new tasks. Preserve followed
    links, dependency drops, scheduling and project guards, offer identity, and complete
    move reports.'
- id: move_presentation
  title: Finish move output, help, and conformance verification
  size: medium
  depends_on:
  - move_links_offers
  description: 'move_presentation: Add human move output and the no-move discoverability
    hint, regenerate capture help, close remaining MV coverage gaps, and record warm
    dry-run performance. Supply verification evidence to the child epic land agent
    for bob-cli-gl.5.'
proposed_by: bbugyi200.apollo.bob-cli-gl.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-gl.5
create_time: 2026-11-03 16:14:48
status: wip
bead_id: bob-cli-gl.5.1
---

- **PROMPT:** [prompts/202611/capture_move_execution.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_move_execution.md)
- **PARENT:** [202611/capture_move_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_move_bullets.md)
- **BEAD:** [bob-cli-gl.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gl/bob-cli-gl.5.1.md)

# Complete capture project-note move execution

This child epic implements the entire remaining scope of **bob-cli-gl.5**. It belongs
under that assigned phase. Its land agent verifies the combined implementation and
closes bob-cli-gl.5 after the child work is complete; individual phase workers close
only their own assigned beads. Nobody in this child epic closes **bob-cli-gl** or any
other ancestor. Proposal submission, rather than manual bead creation, creates the child
work graph.

## Scope and decisions

An item such as:

```text
Finish the Google exit packet @cash^goog-exit@#admin
- Draft the resignation memo :draft-memo
- ^equity-docs
- :call-ms
- Book the lawyer
```

must create `cash_goog_exit.md` with its task roots in that order, move complete blocks
from `cash.md`, repair links and dependency metadata, stamp moved roots, and link
`call-ms` through the existing-task capture behavior. Preview and commit use the same
planner, and any refusal or failed write leaves the vault unchanged. The `:` project
seal also offers moved tasks alongside newly authored tasks.

The authority is the approved `plan:202611/capture_move_bullets.md`, especially
§§3.5–3.9, 3.11, 4, and 5.5, and the landed contract in
`docs/capture.md#moving-parent-tasks-in`. Read the plan with `sase artifact read`, and
read the assigned bead with `sase bead read` before implementation. This child plan does
not reopen any parent decision:

- `grammar_bump = bump` was already implemented by the grammar phase. Retain the current
  grammar revision; execution adds no further grammar break.
- `memory_decision_record = no` and `memory_glossary_project_note = no`. Edit no memory
  notes, including decisions or glossary strands.
- No new syntax, cross-note moves, nested notes under moves, Mac changes, completion
  changes, release version edits, or CHANGELOG edits belong here. Other outer phases own
  the completion and Mac surfaces.

## What has landed, and why three phases

The assigned phase is in progress with no earlier implementation notes. The tree has the
following verified foundations:

- Commit `b94eec2c` parses move bullets and bumps the grammar. `MoveBullet` already
  carries `sigil`, optional `block_id`, and UTF-8 `range`. Lexical diagnostics and
  capture-parse tests are present. `capture/project_note.rs` still refuses every move
  with the interim message, and the corresponding CLI test remains.
- Commit `cf1e4644` provides `MovePlacement::CaptureSlots(Vec<usize>)`,
  `IdCollisionPolicy::Refuse`, `movable_roots`, and `capture::batch_move`. Phase
  bob-cli-gl.3 is closed and records passing engine, inbox-route, and filtered library
  tests. Use these APIs rather than reimplementing task-block movement.
- The shipped slot API uses **zero-based final task-root positions**, not the
  before-line offsets suggested in the original design. Adapt the renderer to that
  actual interface. `MoveRecord.dest_line` is the landing authority.
- `movable_roots` supplies candidates and a textual disabled reason, but the typed
  `resolve_movable` described by the original design does not exist. Add only the
  missing shared resolution detail needed for stable capture refusals, using the
  engine's scanner and validation rather than another task parser.
- `CaptureBatchPlanner::into_staged_files` currently sorts destination/source sets by
  priority and first touch. That is insufficient for `cash → cash_x → cash_x_y`: both
  new notes are destinations, so first touch puts `cash_x` before `cash_x_y`. The
  contract requires the opposite order. Correct this in the first phase and protect it
  with a regression test; it is necessary for MV11/MV13.
- The current project planner offers only rendered authored rows. It mutates chosen
  authored bullets and re-renders, then links all new tasks in one pass. Moved blocks
  cannot go through that reconstruction, and their link gestures must interleave with
  new tasks. This warrants a separate, sequential links-and-offers phase.

Each phase is bounded implementation work for one coding agent. Dependencies are
sequential because they share the planner and its result types. No extra planning
handoff is intended for these medium phases. Re-check the current tree before editing to
preserve newer note-pull, completion, or output changes that may have landed.

## Shared implementation contract

Keep move state keyed by authored first-level bullet index plus source route and ID.
Carry original draft token ranges and line locations separately from rendered note
lines. Do not recover draft identity by searching task text. Account for later batch
items, multibyte text, and child-line project seals when translating ranges to the
original submitted TEXT.

For each item, derive candidates and offers from the staged parent and authored rows,
freeze offer identities, apply choices, render new rows while retaining move slots,
stage the new note, run the move engine once, and apply effective link gestures in draft
order. After linking, resolve resulting rows by ID in the staged note so
schedule/dependency child edits cannot leave stale landing lines. Never regenerate
existing moved blocks from their display text.

Use the actual engine request shape:

```text
source = parent route
dest = new project note
selectors = one MoveSelector::Block per move
allow_closed = false
placement = MovePlacement::CaptureSlots(final root indices)
id_collision = IdCollisionPolicy::Refuse
```

Call through `capture::batch_move`, preserving its vault-plus-overlay snapshot, snapshot
preimage guards, root verification and `stamp_fresh`. Preserve complete children, logs,
drawers and IDs, and leave collision repair to the existing engine's refusal mode.
Historical daily links may be repaired even without new Task Links. No-move drafts must
avoid new vault-wide scans; the hint reuses the validated parent.

## Phase move_execution

Primary files: `src/native/capture_project_note.rs`,
`src/native/capture/project_note.rs`, `src/native/capture/batch.rs`,
`src/native/capture/batch_move.rs`, `src/native/task_move/plan.rs` and its exports,
`src/native/capture/output.rs`, and relevant capture tests. Extract a focused
`capture/project_note_moves.rs` helper if needed to keep the orchestration readable.

1. Extend rendering to omit move bullets while recording their source indices and final
   root slots. Suppress the placeholder whenever an authored task or move exists.
   Preserve section/drawer behavior and note-only TASKS content. A root slot counts
   tasks, not headings, blank lines, nested lines, or section-note bullets.
2. Resolve moves from `planner.current_contents` and shared engine eligibility. Exclude
   `^prj`, fences/frontmatter, closed and nested rows, and placeholders from eligible
   roots. Distinguish missing, closed, nested and ambiguous IDs using typed scanner
   information; use the existing status-name rules and nearest parent ID or title in
   messages. Do not parse `Display` strings to identify errors.
3. Track completed move origins and destinations with their item number within the
   batch. Repeating `(cash, a)` reports the earlier destination and item; subsequently
   moving `(cash_x, a)` is valid. Check this history before turning a removed source
   into a missing-ID error.
4. Enforce a future project schedule before movement: non-`[?]` roots refuse, `[?]`
   roots can move. Preserve inherited project blocking and the engine's
   `needs_reconcile` report. Refuse root/descendant collisions against authored IDs and
   imported blocks without suffix renaming. Use the contract's exact
   `move_task_missing`, `move_task_closed`, `move_task_nested`, `move_task_ambiguous`,
   `move_task_blocked_birth`, `move_task_already_moved`, and `move_task_id_collision`
   codes and teaching messages through `CaptureError::usage_code` or the appropriate
   existing typed failure constructor.
5. Stage the rendered skeleton and call the shared batch move once for the item's moves.
   Build moved row metadata from the staged landing, counting moved roots in
   `project_note.tasks`. Preserve snapshots across later item edits. Keep other engine
   validation errors intact instead of guessing a misleading capture code.
6. Replace priority-only write ordering with recorded `(source, destination)` edges and
   stable topological ordering so every changed destination precedes its changed source,
   retaining first-touch order among available files. Preserve inbox-route behavior and
   wire edge recording through the shared helper used by both callers. Ensure an
   unorderable graph refuses before writes; never silently drop staged files or fall
   back to unsafe ordering. Use the existing test-only rename failure hook in
   `capture/commit.rs` for rollback coverage.
7. Introduce the additive move result types described below. For caret-only Create
   items, report followed live links without relocating them. During this intermediate
   phase keep the existing interim refusal only for items containing colon moves or
   Offer-form moves. Update its test accordingly; the next phase removes it entirely.

Acceptance: MV1, MV2, MV3, MV6, the runtime collision part of MV7, caret cases of MV9,
MV10–MV15. Test all-move and mixed-order drafts, adjacent moves selected in a different
order from their parent, blocks with children, staged chains, same-parent duplicate
moves, pathless dependencies left behind and co-moved, repair races, and rollback.
Assert byte-identical vault contents after every refusal. Run focused renderer,
task-move, inbox-route and project-note tests, then `just check`.

## Phase move_links_offers

Primary files: `capture/project_note.rs`, its new move helper, `capture/plan.rs`,
`capture/pomodoro_link.rs`, `capture/link_guard.rs`, `capture/dependency_drop.rs`,
`capture/output.rs`, and project-note/link integration tests.

1. Pass the existing batch `DependencyContext` into the project-note planner. Reuse
   `plan_existing_task_pomodoro_link_item` and the capture-level deliberate named-type
   behavior in `plan.rs`; extract a shared adapter if necessary. Calling only the
   low-level placement helper is insufficient: the gesture includes link guards, safe
   local prerequisite drop, schedule pull-forward, lane transitions, stamping, task
   typing, destination merges, and block/dependency result tracking.
2. Build one source-ordered link sequence containing new colon tasks, colon moves, and
   selected offers. Preserve new-task creation semantics, including their lack of an
   existing-task freshness stamp. For moves, run the existing-task gesture after block
   movement. A followed live link stays in its original Pomodoro and reports `planned`;
   it must not be relocated or duplicated by an item name.
3. Count colon moves in both planner-level unused-name checks. Preserve literal names,
   canonical names, per-task type placement when no name is supplied, and merged-name
   propagation across interleaved new and moved tasks. Preserve meaningful
   `pomodoro_refs`, task block refs, dependency-drop/guard details, warnings, and
   aggregate result fields from nested gesture plans.
4. For future-dated projects, an eligible blocked colon move stays blocked and unlinked
   with `not_linked: "project_schedule"`. For inherited project blocking, retain tags
   and guard behavior. A task blocked only by its own open prerequisite receives the
   existing safe drop and then links; do not invent a more permissive dependency drop or
   automatically reconcile the vault.
5. Offer-form planning combines new rendered rows and moved candidate metadata before
   staging. Freeze keys as `<new-note>#^<id>` for moves, use their final-note order for
   numbering, and set `moved_from` on the offer object. Already-live-linked moves and
   colon moves count in `already_planned`; hidden/blocked/closed rows retain existing
   offer exclusions. Include moved and descendant IDs in minted-ID reservations so
   selected ID-less authored tasks cannot collide.
6. Apply a selected moved offer with the same operation as a colon move, leaving
   unselected moves unchanged apart from movement/stamping. Apply choices to existing
   assembled blocks; never reconstruct moved blocks as authored bullets. Preserve the
   existing pre-pick digest and settled-ordinal handling for ID-less authored rows,
   global offer numbering, selection validation, scratch destinations and the batch
   offer-settlement pass. Re-resolve final moved rows by ID after any mutations.
7. Remove the remaining interim refusal and its old test. Complete move JSON status,
   planned/link/guard/detail fields from actual results, including selected caret offers
   (the original sigil remains `^`).

Acceptance: MV4, MV5, the colon case of MV9, and MV16; named and implicit destinations;
Ready/Next/Pending moves; existing links; local prerequisite drops and refused project
drops; all/none/subset selections; mixed colon/new/moved draft ordering; and
preservation of full moved children. Compare `-p all` with equivalent explicit colon
drafts. Run focused project-note, existing-link, dependency-drop and offer-settlement
regressions, then `just check`.

## Phase move_presentation

Primary files: `capture/output.rs`, `capture/project_note.rs`, `capture/cli.rs`,
`tests/cli/capture/project_note.rs` (split new tests into a sibling module if useful),
`tests/fixtures/help/`, and generated documentation.

The output schema is additive; retain the current capture envelope/version exactly. The
original design calls it schema version 1, but the landed JSON reference identifies
capture as unversioned. Do not add or bump a schema field to resolve that wording.
Complete and verify these fields:

- `project_note.moved_tasks`, in draft order: `sigil`, UTF-8 `draft_range`, `item_line`,
  `source`, `source_line`, final landing `line`, `block_id`, `text`,
  `previous_status_symbol`/`previous_status_name`, `status_symbol`/`status_name`,
  `fresh` (date or null), `planned` (`{entry_name, entry_line}` or null), `link` (the
  `ProjectTaskLinkJson` shape or null), `not_linked` (reason or null), and `detail`.
- `project_note.move_summary` when moving:
  `{parent, moved, repaired_links, repaired_notes, needs_reconcile}`.
- `project_note.repaired_files`: `{path, link_count, dependency_token_count}` from
  actual move repair results; avoid double-counting files or confusing dependency
  metadata changes with repaired link counts.
- `project_note.move_hint: {parent, open_count}` only when there are no move bullets and
  at least one movable parent root. Include ID-less/hidden movable roots; exclude
  placeholders, project tasks, closed/nested rows and roots disabled by eligibility.
- Optional `moved_from` on moved offer objects; omit it for ordinary offers.

Human output follows the contract's copy and existing styler:

```text
4 tasks · 2 moved from cash.md · sections Future Work
would move  [/] Collect the equity paperwork  cash.md:7 → ^equity-docs · WORK link follows
would move  [ ] Call Morgan Stanley about the 401k  cash.md:12 → ^call-ms · becomes Next · links under ADMIN
would repair  3 links in 2 notes
hint: '- ^' moves one of cash.md's 12 open tasks into this project
```

Real-run verbs are `moved` and `repaired`. Bob authors all consequence text, including
`links under ADMIN` for an already-Next task and
`stays unlinked until the project starts`. Omit empty consequence separators and handle
singular/plural counts. Preserve existing sync/reconcile hints. Use report state rather
than parsing the draft to generate copy.

Rewrite `capture/cli.rs`'s stale claim that the parent is never modified and add the
move-bullet explanation. Read docs memory, run `just docs-gen`, update the affected help
fixtures using repository conventions, and run `just docs-check`. Reconcile only actual
implementation deviations in `docs/capture.md`; preserve the agreed behavior and
published anchors. Record the shipped root-index slot adapter and any necessary
documentation correction in implementation notes/commit text, not memory.

Audit MV1–MV17 against concrete tests, retaining the grammar phase's existing MV7/MV8
tests instead of duplicating every lexical case. Add MV17 and any missing end-to-end
coverage. At fixed `BOB_NOW`, compare dry-run and real JSON on equivalent vault state,
normalizing only `dry_run`; verify dry runs write nothing. Check UTF-8 draft ranges and
item lines in a later batch item and a child-line seal. Run `just check` and
`just docs-check` on the final tree.

Record warm dry-run timing for the worked draft on a disposable vault copy, including
note count, build profile, warmup and measured samples. Use a representative fixture
vault copied under a temporary directory if a real vault copy is unavailable, and label
its scale honestly. Never mutate the live vault for verification or performance
measurement. Keep one move-engine plan/snapshot pass per item and no new full-vault scan
for no-move drafts; verify by code inspection plus targeted instrumentation only if
needed.

## Verification, follow-ups, and land ownership

Every phase runs its targeted checks and `just check`; help/docs edits additionally run
`just docs-gen` and `just docs-check`. Use `/sase_monitor` for long verification
commands and finish the staging command before ending a turn. Do not leave a phase open
merely because a check fails identically on the unchanged clean base: document the exact
command and matching failure as `PROPOSED FOLLOW-UP:` on the assigned phase, citing an
existing task if known, then close once its own scope is verified. Newly introduced
failures must be fixed. Phase workers create no beads.

Before each phase close run `sase bead epic-symbols <assigned-phase>` and remove or
re-key every remaining suppression to an actually open owner. No transient dead-code
exemption may point at a closed phase.

The child epic land agent must inspect the combined diff and phase evidence, verify
MV1–MV17 coverage and final gates, and confirm the old refusal is absent. It must run
`sase bead epic-symbols bob-cli-gl.5` and resolve/re-key leftovers before closing **only
bob-cli-gl.5** with
`sase bead close bob-cli-gl.5 --note "<verified behavior and checks>"`, after all its
child work is closed. Normal child-epic bookkeeping is limited to that child epic. It
must never close **bob-cli-gl** or another ancestor. Preserve the outer Mac preview
dependency on bob-cli-gl.5; this phase remains open until its whole execution/output
contract has landed.
