---
tier: epic
title: Finish Rust leaf-project keep-moving offers
goal: Complete bob-cli-fb.2 with tested leaf detection, successor offers, capture
  selection and reporting, correct batch behavior, and verified performance.
phases:
- id: engine
  title: Implement the keep-moving successor engine
  size: medium
  depends_on: []
  description: 'engine: implement leaf detection, cost gates, K1-K8, candidate classification,
    stable ordering, picks, and separate keep-moving rows in the shared Rust engine,
    with LP and pure KM tests.'
- id: capture
  title: Connect capture planning and the JSON contract
  size: medium
  depends_on:
  - engine
  description: 'capture: wire both capture close paths, dependency completeness, config,
    separate JSON arrays, selection and refusal, group folds, task-block roles, and
    Pomodoro diff reasons; add focused CLI tests.'
- id: finish
  title: Complete batch semantics, human output, and verification
  size: medium
  depends_on:
  - capture
  description: 'finish: implement KM21 with later-item provenance, render the approved
    human copy, complete cross-surface regression coverage, and verify repository
    gates and the release-mode performance target.'
proposed_by: bbugyi200.apollo.bob-cli-fb.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fb.2
create_time: 2026-11-01 13:00:25
status: wip
bead_id: bob-cli-fb.2.1
---

- **PROMPT:** [prompts/202611/leaf_keep_moving_rust.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/leaf_keep_moving_rust.md)
- **PARENT:** [202611/leaf_project_keep_moving_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/leaf_project_keep_moving_offers.md)
- **BEAD:** [bob-cli-fb.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fb/bob-cli-fb.2.1.md)

# Finish the Rust keep-moving phase

## Assignment and scope

This is a child implementation epic for **bob-cli-fb.2**, the Rust engine and capture
reporting phase of **bob-cli-fb**. It implements the Rust portion of
`plan:202611/leaf_project_keep_moving_offers.md`. The runtime should attach this child
epic to bob-cli-fb.2. This proposal creates no task beads by hand.

The original phase has no prior-work notes or recorded remainder. At inspection, the
checkout is clean and its HEAD is `d6b16f4f`, which landed the contract, documentation,
and authorized glossary work. There are no `keep_moving` implementation symbols in the
Rust code and no `--epic-symbol` entries for bob-cli-fb.2. Recheck current code and bead
notes before implementation to avoid repeating work that landed after this proposal.

Use three sequential medium phases because the shared engine, capture integration, and
batch/presentation behavior each have their own substantial tests and failure modes.
Each phase must leave a compiling tree. The first phase supplies a tested engine behind
a false capture input until the second phase connects its output and config. The final
phase completes the batch and presentation contract before the assigned Rust phase can
close. This is all bob-cli work; the parent epic's plugin and Mac phases remain
separately owned.

Final parent decisions are binding:

- `offer_lanes = open`: offer Pending, Next, and Ready, in that order.
- `memory_glossary = yes`: the authorized changes already landed in the contract phase.
- `memory_decision_record = no`: do not create a decision record.

No memory edits are needed or authorized by this child plan. Do not edit generated agent
instructions, `Cargo.toml` versions, or `CHANGELOG.md`. The JSON change is additive and
stays schema version 1. Keep all existing flags and selection grammar.

## Contract to implement

`docs/task-dependencies.md` §§12.15-12.17, including KM1-KM32 and LP1-LP12, are the
behavioral authority. §§12.2-12.5 define keys, routing, picking, and output. The shared
design is already approved; do not reopen its choices.

A leaf is a recognized, nonterminal project outside `done/`, with one open `^prj`, whose
`HomeResolver::resolve_note` result has Project kind, its own path as key, and a
different `root_parent`. Middle projects can be leaves; task `[root::]` does not affect
leafness. Reuse the resolver's wrapper, root, area, ambiguity, cycle, and depth rules.

For one managed Done gesture, C is its open-to-Done tasks, including closed embedded
subtasks. Read the day baseline before strikes, retirement, drops, or carry-forward, but
after an explicit link step in the same item. Read task/project statuses from the staged
post-gesture notes. For each eligible project N containing C:

1. PRED(N) consists only of completed non-`^prj` residents with their own baseline live
   link; it must be nonempty.
2. No baseline live link may target other open work resident in N, including `^prj`.
   Ignore already-closed targets and tasks resident in sub-projects. Plain, embedded,
   and deferred links in open Pomodoros, including planned placeholders, count.
3. N's `^prj` must remain open and unblocked by its dependencies, schedule, and
   inherited project tag. N must not be empty. A `project` group or `project_finished`
   row for N suppresses keep-moving. Cancel and every exclusion in §12.15 never trigger
   it.
4. Both `plan.link_unblocked` and `plan.keep_moving` must be true.

Classify N's remaining open task residents, excluding C and `^prj`. Fold blocked rows
first, then future-scheduled rows, then hidden rows. Unknown/custom marks are ignored.
Offer Pending, Next, and Ready; include eligible refs, nested subtasks, and due
recurring tasks. Do not enumerate descendant notes. Deduplicate by task identity with
priority `hand_off > project > project_finished > keep_moving`; omit a group with no
selectable rows. Order keep-moving after every existing group, including project-created
offers, then by anchor, note path, lane, and task line.

Reuse existing keys, selection, D7 type routing, anchors, and placement. Picks mint only
missing block IDs, raise Ready to Next while preserving Next/Pending, stamp freshness
and clear keeps/unblocked fields, and insert plain Task Links. Unpicked rows do not
recover, mint, stamp, strip tags, or otherwise edit their task lines. All keep-moving
rows have trigger `last_planned`, group `keep_moving`, project N, and PRED(N) in
`unblocked_by`; they never enter `unblocked[]` or `still_blocked[]`.

## Verified implementation seams

- `src/native/task_complete/successors.rs` already owns `DayView`, `DependentVerdict`,
  `OfferDraft`, keys, numbering, selection, routing, placement, and writes. Its current
  gate returns early when there is no completed dependency ID or project pull-in.
  Keep-moving must open that gate too. Add the new algorithm in `keep_moving.rs`, not as
  another large block in `successors.rs`.
- `project_pull_in.rs` and `project_close.rs` already supply recognized project
  surfaces, EMPTY/SCHED, project blocking, candidate recovery, and row emission. Reuse
  their predicates without changing existing project behavior.
- `SuccessorInput` currently has no snapshot-completeness field. An empty snapshot
  cannot distinguish an actually empty dependency graph from an unavailable one. Add
  explicit completeness information: unavailable input must never be interpreted as
  proof that prerequisites are closed.
- Both `capture/task_complete.rs` and `capture/pomodoro_close.rs` currently stop all
  successor planning when the required dependency snapshot is unavailable. KM31 needs a
  scoped exception: safe keep-moving candidates still offer, while dependency-bearing
  candidates fold blocked and legacy recovery remains suppressed. Never resume legacy
  dependency recovery against incomplete data.
- The actual SL20 implementation is `apply_successor_net_post_pass` in
  `capture/task_complete.rs`; `capture/plan.rs` calls it before block diffs finish.
- `capture/output.rs` has the cross-summary iterators `all_unblocked`,
  `unblocked_vecs_mut`, and `all_project_folds`. `capture/pull_in.rs` uses them for
  numeric probes, key validation, draft numbering, counts, and choice refusals. Extend
  this plumbing to both arrays, including nested relation completions, without merging
  the serialized arrays or accidentally printing keep-moving as unblocked.
- `OfferOrderKey` currently contains group, anchor, path, and line. Update all producers
  consistently when adding lane order. `capture/project_note.rs` constructs its own keys
  for `project_created`; keep-moving must sort after those too.
- `surviving_successor_links` currently returns one set and `finish_with_unblocked`
  assigns one reason. Preserve the surviving link's class so a keep-moving pick gets
  `reason: "keep_moving"`, and existing picks keep `"unblocked"`.

## Phase engine: tested shared rule

Primary files: new `src/native/task_complete/keep_moving.rs`,
`src/native/task_complete/mod.rs`, `successors.rs`, narrowly shared helpers in
`project_pull_in.rs`/`project_close.rs`, and new
`src/native/task_complete/tests/keep_moving_tests.rs` registered in `tests/mod.rs`.

1. Implement `is_leaf_project` using `HomeResolver`, with an in-memory `HomeSource` for
   LP tests. Production resolution must honor staged frontmatter for touched notes and
   read only reached ancestors when disk fallback is necessary. Do not implement a
   second root-parent algorithm or add eager vault discovery.
2. Implement the G1-G3 no-I/O pre-gate over held completed-note and baseline day text.
   G1 checks project/nonterminal completed non-`^prj` work with a block ID; G2 checks
   for at least one otherwise eligible resident; G3 checks the predecessor's block
   fragment in the held day text. Resolve leafness only after these checks, and parse
   live links/build the required index only after the cheap rejection path.
3. Add `SuccessorInput.keep_moving`, explicit dependency-snapshot completeness, and
   `OfferGroup::KeepMoving`. Add `SuccessorPlan.keep_moving: Vec<UnblockedRow>` and
   distinct keep-moving folded counts (`hidden`, `blocked`, `scheduled`) per project.
   Adapt current constructors/tests to compile; existing capture callers pass false for
   keep-moving until the capture phase is complete. Do not introduce new blanket
   dead-code allowances.
4. Draft offers after existing hand-off, project, and pull-in classifications so K7 and
   dedup see them, but before routing, key assignment, selection, and placement. Update
   the planner's early gate. Reuse existing verdict/placement machinery; keep all new
   detection and classification in the new module.
5. Preserve the no-write property of unpicked keep-moving verdicts. Existing
   `apply_verdicts` and `emit_rows` have recovery-specific paths: distinguish the new
   class explicitly, emit it into the separate vector, and do not write recovery
   metadata. Use the normal selected-offer writers for picks.
6. Handle incomplete dependency snapshots explicitly. Fold candidates whose own
   prerequisites cannot be proved closed. If the project root's blocking state cannot be
   proved safe, suppress the group. Candidates with no dependency fields remain eligible
   where the project can be proved safe. Preserve cancellation recovery.
7. Extend ordering with a lane component used only for keep-moving; keep existing
   groups' relative order unchanged. Keep stable ID-less keys and deterministic mint
   collision handling, and adapt project-created order-key constructors mechanically.

Tests must cover LP1-LP12 and every pure engine KM case: own-link vs another task's
link, root links, old/struck/closed-entry links, placeholder and closing anchors,
carried/dropped baseline work, nonempty recognition, root blocking, folds, lane order,
dedup, selection/no selection, recurring/ref/nested residents, two-project ordering, and
unavailable prerequisites. Cite vector IDs. Assert unpicked task text equality, absence
of mints/freshness/unblocked writes, and Pending preservation on picks. Exercise type
routing and stable keys through the existing machinery. Add a counting source or
equivalent test seam to verify the cheap gate performs no ancestor reads.

Acceptance: targeted engine tests and existing successor/project-pull-in tests pass, and
`just check` is green apart from a proven identical clean-base failure. Capture behavior
remains at its prior state until the next phase connects the new result.

## Phase capture: runtime and JSON integration

Primary files: `src/native/config/plan.rs`, `capture/dependencies.rs`,
`capture/task_complete.rs`, `capture/pomodoro_close.rs`, `capture/output.rs`,
`capture/pull_in.rs`, `capture/plan.rs`, `capture/pomodoro_blocks.rs`,
`capture/task_blocks.rs`, and new `tests/cli/capture/keep_moving.rs` registered with the
existing CLI test module tree.

1. Add `plan.keep_moving`, default true, with the same boolean/null parsing and invalid
   value warning/fallback path as `link_unblocked`. Cache its loaded value per batch.
   Test missing, null, true, false, and invalid scalar values.
2. Wire the pure pre-gate into both successor callers. A keep-moving candidate must run
   the planner even without a dependency ID or project-empty trigger. Keep the day
   baseline at the existing anchor point, including the explicit-link step for that
   item. Preserve `pre_day` versus `post_day` for Pomodoro closes and all complete-label
   entry points. Cancellation, parks, hooks, and project creation do not gain triggers.
3. Build the dependency snapshot only when legacy gates need it or eligible keep-moving
   candidates/project safety need dependency proof. Pass explicit completeness into the
   engine. On failure, retain `unblocked_check: "unavailable"` and suppress legacy
   recovery, but allow KM31's independently safe rows. Clarify the existing blanket
   `unblocked_check` sentence in §12.5 to name this approved exception; do not change
   the KM31 rule or any decision.
4. Always serialize `keep_moving: []` or populated rows on `task_complete` and
   `pomodoro_close`, including nested `relation_completes`. Use the existing row shape
   and engine-provided trigger/group/predecessors. Keep `unblocked[]` unchanged.
   Disabled/no-trigger cases have empty arrays and no keep-moving group.
5. Include both arrays in shared offer traversal, draft numbering, numeric selection
   probes, stable-key selection, stale-key errors, refusal payloads, and counts. Extend
   fold aggregation by group and project so project folds cannot overwrite keep-moving
   folds. Include blocked/scheduled counts for this group; omit groups with no surviving
   selectable rows. Preserve numbering after a numeric probe.
6. Set `day_file` and continuation details when keep-moving picks write the day. Add
   `TaskBlockRole::KeepMoving` for selected rows and preserve the reason per surviving
   Pomodoro link. Extend existing per-row net traversal to the new arrays; full
   later-item group invalidation is owned by the finish phase.
7. Add focused CLI coverage for `!`, Pomodoro completion, retargeted close, and nested
   complete labels; separate array shape; all/none/key/number picks; mixed groups,
   including project-created rows; folded counts; no-candidate silence; both kill
   switches; and write-free `--require-pull-in-choice` (KM30). Test unavailable-snapshot
   behavior through a controlled fixture/seam, not file permissions dependent on the
   test runner. Assert dry-run leaves the fixture unchanged and dry-run/real-run JSON
   agree for the same initial state and flags.

Acceptance: single-gesture capture JSON and selection satisfy the contract, safe
keep-moving rows survive KM31, the G1-G3 rejection path adds no index/snapshot work, and
focused CLI plus existing SO/PP/SL suites pass. No CLI or JSON breaking change.

## Phase finish: net batch, human output, and verification

Primary files: `capture/task_complete.rs` (or a small extracted net helper module),
`capture/plan.rs`, `capture/output.rs` (or a focused new rendering module), related
fold/reason plumbing, and the new keep-moving CLI tests.

1. Implement KM21 with source-item/later-item provenance. A later item completing,
   reblocking, linking, or dropping a candidate removes its row. A later item linking
   any open resident of N, or closing N's `^prj`, removes N's whole group. Preserve the
   existing final task/link checks and dependency/project-empty rules.
2. Do not implement group invalidation as merely “any final live link in N”: links
   selected by the originating close must survive, including several picks in one group.
   Track item order and the provenance of later task/link effects using existing
   planned-item metadata or a narrow explicit record if needed. Test a later explicit
   action on an already-selected row as well as a later action on a different resident.
   Report only surviving rows, counts, folds, and link reasons; net reporting does not
   undo task edits legitimately performed by earlier items.
3. Render keep-moving separately for direct completion, Pomodoro completion, nested
   relation completion, and human refusal output. Reuse offer numbering, destinations,
   and styling. Lead with why; never label these rows “unblocked” or use the unlock
   glyph. Use lane names rather than raw marks for these rows. Preserve unrelated
   existing output. The approved forms are:
   - Header:
     `Keep N moving · ‹T› was its last task planned today (ready tasks, not dependents)`.
   - Pick suffix: `· keeps N moving`.
   - Fold row: `not offered  2 blocked · 1 scheduled later · 1 hidden`, omitting zero
     parts.
   - Use the §12.15 predecessor-cleaning/truncation and multiple-predecessor caption
     rules, ⏭ where a glyph is shown, and existing Next color styling.
   - No selectable rows means no header/fold-only group. Explicit none never writes task
     fields or produces a linked notification.
4. Complete KM coverage across pure and CLI suites. KM24 is the cycler's re-plan case;
   verify the Rust equivalents through fresh planning, stale picks, and transactional
   stale-file refusal. KM26-KM28's Obsidian UI/walk/undo portions belong to other
   phases; do not claim they ran in Rust. Test KM32's zero-added-read path and the
   unlinked project-close gate. Keep the original SO, PP, and SL assertions intact.
5. Exercise batch cases with no picks, one pick, multiple picks, later sibling links,
   later completion of just one offered row, later root close, reblocking, drops, nested
   complete labels, and multiple project groups. Verify remaining number/key selections,
   absence of stale group counts, and final diff reasons. Include a mixed
   `hand_off`/keep-moving human refusal to catch accidental “unblocked” labeling.
6. Run formatting, the full `just check`, and `just docs-check`. Use `/sase_monitor`
   before long builds/checks as required by the single-turn runner; wait for each
   handoff command itself to exit. Run targeted tests during implementation so the final
   full gate is not repeatedly rerun without a new change or failure.
7. Build release mode on apollo and measure a triggering `!` dry run against a temporary
   copy of `~/bob`, never against the live vault. Follow vault reference memory before
   accessing it. Keep scratch work inside the permitted workspace and isolate config,
   clock/day fixtures, and output paths. Inspect the copy for a suitable linked leaf
   task or stage the documented `sase_badges:draft-spec` fixture only in that copy.
   Record exact command, release revision, repetitions and timing distribution, plus
   proof that JSON contains the KM1 group; the triggering `!` target is ≤70 ms. Compare
   representative nontriggering closes against the base if a regression appears. Do not
   report an unavailable measurement as passing.

Acceptance: all Rust-expressible KM and LP cases pass, human copy matches §12.15,
single/batch JSON selection and net results agree, full gates pass or a failure is
proven identical on the clean base, and the performance result is recorded.

## Landing and bead lifecycle

Every phase closes only its own assigned child phase after checking
`sase bead epic-symbols <assigned-id>` and resolving or re-keying leftovers. Prefer no
temporary unused-symbol exemptions; if unavoidable, key them to a still-open consumer or
this child epic and remove them as soon as consumed.

Discovered out-of-scope work goes in a `PROPOSED FOLLOW-UP:` note on the phase assigned
to that worker, not a new bead. Record any pre-existing gate failure with the exact
command, matching clean-base reproduction, and any known tracking bead. Such an
identical base failure does not leave a completed coding phase open.

The **child epic's land agent** verifies all three phases together and records the
coverage, JSON example, gates, and timing evidence on bob-cli-fb.2. That designated land
agent, as requested for a child epic, runs `sase bead epic-symbols bob-cli-fb.2`,
resolves or re-keys all remaining entries, then closes **bob-cli-fb.2 only** with
`sase bead close bob-cli-fb.2 --note "<what was verified>"`. Child phase workers must
not perform that ancestor close. The land agent also finishes its own child epic via the
normal lifecycle; it must not close bob-cli-fb or any ancestor above bob-cli-fb.2. The
parent epic's land agent retains plugin/Mac integration and parent closure.
