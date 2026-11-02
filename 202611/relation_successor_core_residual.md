---
tier: tale
title: Finish the relation-successor core residuals and land bob-cli-fs.2.1
goal:
  Close the confirmed residual gaps in the Rust relation-successor core (slot placement,
  effective-state projection, relation-winner placement and folds, target reads, and
  real vector and capture-compatibility coverage), verify, then close bob-cli-fs.2.1 and
  its parent phase bob-cli-fs.2 in the same turn.
size: medium
bead: bob-cli-fs.2.1
proposed_by: bbugyi200.athena.bob-cli-fs.2.1.land
proposed_with: claude/opus
status: done
---

- **BEAD:**
  [bob-cli-fs.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/bob-cli-fs.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-fs.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-fs.2.1.land.md)
- **COMMITS:**
  - [fdc34de](https://github.com/bobs-org/bob-cli/commit/fdc34de9bf82600e187ef8f6ebb66473fc4d2bbf)
    — fix(task-complete): finish relation successor core

# Finish the relation-successor core and land bob-cli-fs.2.1

This tale finishes landing child epic `bob-cli-fs.2.1`, the Rust relation-successor core
for phase `bob-cli-fs.2` of `bob-cli-fs` (starts-after relations). All three phases are
closed. The first closeout tale (`plan:202611/relation_successor_core_closeout.md`,
commit `98bfec6a`) fixed four regressions and is only partly done. A fresh land re-audit
at bob-cli `cba88d0d` confirmed in source that the items below are still open. They are
epic work, not follow-ups. One coding agent implements them, and that agent also
performs the closeout at the end. No later lander exists, and nothing resumes this
landing.

## Read first

- `sase bead read bob-cli-fs.2.1 -r "<why>"`: the notes matter. #2 is the original land
  audit, #4 is the partial repair, and the newest note is the re-audit with exact
  residuals and triage. Also read its phases `.1`, `.2`, and `.3`, and the parent phase
  `bob-cli-fs.2`.
- `sase artifact read plan:202611/relation_successor_core.md "<why>"` is the binding
  core contract.
- `plan:202611/relation_successor_core_closeout.md` has detailed acceptance wording for
  every item below.
- `plan:202611/starts_after_relation.md` is the parent contract. Its Placement bullets
  cover `done`, `parked` by `x*`, and `parked` by an unlink. Its `capture_unlink` phase
  says the explicit slot is "the open entry and position of T's first removed link".
- The canonical product docs are `docs/task-relations.md` §20 and
  `docs/task-dependencies.md` §§12.18–12.19. Read the required reference memory with
  `/sase_memory_read`.
- Code:
  - `src/native/task_complete/relation_successors.rs`
  - `src/native/task_complete/relation_successors/{classify,copy}.rs`
  - `src/native/task_complete/successors.rs`, especially `integrate_relation_offers`,
    `relation_recovered_map`, `offer_order_key`, `attach_offer`, the fold collection
    around `project_folds` and `keep_moving_folds`, and `resolve_closing_target`
  - `src/native/task_complete/tests/relation_successor_tests.rs`
  - the fixture `tests/fixtures/relation_successor/vectors.json`

## Hard constraints

- Every capture caller keeps passing `relation: None`. This tale adds no capture event
  detection, disk scan, batching, CLI grammar, JSON field, schema version, or
  human-output change. Downstream phases `bob-cli-fs.3`, `.4`, and `.5` own that work.
- Do not edit `Cargo.toml` versions, `CHANGELOG.md`, memory notes, decision records,
  generated `AGENTS.md`, or vault data. The parent's DECISIONS are final: glossary work
  is already done, and there is no decision record.
- Do not weaken or edit the shared fixture vectors. bob-plugins parity is finished: all
  three `vectors.json` copies are byte-identical, and the navigation truncation fix
  `e9f2fb7` is committed and deployed. Do not touch bob-plugins.
- Recovery stays automatic. Linking, ID minting, `fresh` stamps, and Ready→Next require
  an explicit pick. Unpicked relation-only rows write nothing. Cancellation and
  recovery-only skip relations. Disabled or nontriggering calls do zero target reads.
- Keep the four permanent regression tests from `98bfec6a` green:
  - `classifier_orders_same_lane_by_source_line`
  - `planner_numbers_next_before_ready_across_notes`
  - `recovered_blocked_direct_dependent_keeps_relation_reason`
  - `missing_id_selects_the_exact_source_row`

  Keep the existing SO/PP/KM/LP/recovery/relation-close suites green without changing
  their expectations. At `cba88d0d`, `cargo test --lib task_complete::` reports 187
  passed.

## 1. Effective post-recovery state for R1–R7

The classifier gets its post-recovery state from `relation_recovered_map` in
`successors.rs`, which `classify::blocked_reason` and `classify_source` consume. That
path has three bugs:

- **Raw dependency count.** A source with no legacy verdict falls back to
  `task.depends_on.len()`. A non-dependent Ready source whose `dependsOn` prerequisite
  closed before this gesture is therefore held `blocked`.
- **Placeholder waits.** `waits_on` is filled with empty placeholder `RelationTarget`s
  instead of the real open prerequisites.
- **Pre-recovery text.** R5 checks `has_hide_tag(raw)` on the pre-recovery line, so a
  project-finished pull-in whose `#hide` this gesture removes is held `hidden`. A
  recovered `[?]` project resident also gets relation lane 2 (`relation_lane('?')`),
  when its effective lane is Ready.

The fix:

- **One effective-state helper.** Replace the boolean overlay with one pure projection
  that reuses the existing dependency and project analysis. It reports, per source:
  - effective status and lane;
  - the real remaining open prerequisites, resolved from the snapshot, including
    prerequisites this gesture closed and prerequisites closed earlier;
  - project block;
  - any planned `#hide` or `#prj_is_blocked` removal.
- **Keep the original state.** Keep the original state for final writes and reporting.
  Never run the writer to manufacture this preview.
- **Conservative fallbacks.** A source with `dependsOn` and incomplete dependency data
  stays conservatively held (KM31). Unresolvable prerequisite IDs follow the same rule
  that the existing dependency analysis uses.
- **One identity key.** Key the projection by explicit source identity: path plus line
  for id-less sources, never a bare path. Drop the legacy bare-path fallback in
  `classify_source`.

Tests, each with no picks and with explicit picks:

- a non-dependent relation source whose own prerequisite closed before the gesture
  (offered);
- the same source with unavailable dependency data (held `blocked`);
- a real `[?]` direct dependent recovered by Done (hand-off plus relation reason; this
  regression already exists);
- a blocked project resident recovered by the gesture (Ready lane);
- a hidden project-finished pull-in that also has a starts-after edge. The relation
  wins, and the `#hide` removal still writes.

## 2. Relation-winner placement, dedup bookkeeping, and folds

In `integrate_relation_offers`, a Project, ProjectFinished, or KeepMoving winner is
retagged to `OfferGroup::Relation` and loses `offer.project`. Its old placement anchor
is kept, which is wrong.

- **Rebuild placement.** Rebuild that winner's placement from the winning relation
  trigger, using the same anchor and `attach_offer` path as relation-only rows. Factor
  the anchor selection that is duplicated today into one helper.
- **Placement rules.** For Done events: the struck-bullet slot, then the closing anchor,
  then `inherit`, then implicit. For parked events: the closing context (continuation,
  then next same-name open entry, then a created placeholder), or the explicit slot
  (§3), or implicit. A hand-off keeps its own placement and only gains relation
  metadata.
- **Keep recovery and reasons.** Keep the original recovery effects (project or pull-in
  writes, `#hide` removal) and the original reason separate from the display group. They
  must survive the retag.
- **Recompute folds.** Today `project_folds` and `keep_moving_folds` are computed before
  the merge and never recomputed. Recompute or filter them from the surviving offers
  after dedup, so a project or keep-moving fold whose only offers moved to Relation
  disappears. Preserve the legacy empty-project-group behavior that the existing PP/KM
  tests pin.
- **Tests.** Write integrated tests for real Project, ProjectFinished, and KeepMoving
  overlaps. Run them through `plan_successors` with `keep_moving: true` where needed,
  not with synthetic prelabelled verdicts. Assert:
  - the winning group;
  - the preserved recovery writes;
  - the placement entry and line in exact day bytes, with at least two open entries so a
    wrong anchor cannot pass;
  - the folds.

## 3. Consume explicit parked-event slots

`RelationEvent.slot` (`RelationSlotAnchor { note_path, line }`) has no production read.
`integrate_explicit_slots_and_closing_destinations` asserts only row counts, and its
"deleted slot" case supplies no slot.

- **Define the coordinates.** Document the coordinate semantics on `RelationSlotAnchor`.
  Recommended: `line` is the 0-based line in the **post-gesture** day text where T's
  first removed link bullet stood. That is where a successor bullet is inserted.
  `note_path` must equal the day path.
- **Validate the slot.** The slot is valid only when it falls inside an open Pomodoro
  entry's bullet region in the post-gesture `DayView`, between the entry header and the
  block end inclusive. A valid slot places a picked row there with `EntryRole::Slot` and
  the correct reported entry and line. Any other slot falls back to the implicit
  destination.
- **Order of precedence.** An explicit slot beats the closing context for that event's
  rows. Without a slot, today's behavior is unchanged.
- **No placeholder for previews.** A preview or an unpicked offer never creates a
  placeholder.
- **Type routing.** Type routing remains the final placement authority.
- **Tests.** Assert exact day bytes and the reported entry and line for:
  - a picked row with a valid slot, where the day has at least two open entries and the
    slot is in the non-implicit one;
  - an unpicked row with a valid slot (no writes);
  - a supplied slot whose entry the cleanup deleted (implicit fallback);
  - a slot outside any open entry (implicit fallback).

Since `d32da8fa`, a MEET close may place its created continuation after the last open
Pomodoro. Add one parked-by-close case where `ClosingContext.continuation` points at a
continuation placed last, and check that the picked row follows it.

## 4. Target reads through the full index

`integrate_relation_offers` builds `MapTargetReader` from the held files only. A target
that is in the `NoteIndex` but not held reads `Missing`, so `resolve_edge` drops the
edge. R6 should hold `waits_for` instead.

- **Unread targets stay unreleased.** A resolved target that is not held and not in
  `unavailable_targets` must count as unavailable (held R6, never released, never
  dropped).
- **Truly missing targets.** Only a path or block ID that does not resolve, or a
  resolved note without that block ID, is ignored.
- **Narrow reader.** Either add a narrow cached target-text source to
  `RelationSuccessorInput`, or document and test that the planner's held `note_texts`
  are the complete target data, with every other indexed path treated as unavailable.
  Keep relation and dependency completeness separate. Staged texts, including a deleted
  LINKS line, override snapshots. Never enumerate the vault or read a label cache.
- **Tests.** Write integrated tests through `plan_successors` for:
  - an indexed-but-unheld target, which holds `waits_for`;
  - an ambiguity-causing basename note that is indexed but has no source text, so the
    edge stays unresolved and nothing is offered;
  - repeated edges to one target, read once (count the reads);
  - cheap gates, which make zero reads.

## 5. Real vector coverage and capture compatibility

**Vector coverage.** The current `INTEGRATION_CASES` skip SA14, SA15, SA18, SA20, and
SA21 in the vector walker, and the `seen == expected_ids` check counts the skipped IDs
as covered. Replace the skip with real production assertions for every integration-owned
vector, case, and expected field:

- run SA14, SA15, and SA18 through `plan_successors` against the materialized fixture
  world;
- run SA20 and SA21 through the pure `edge_added_after_release_warning` and
  `relation_offer_survives_final_state` helpers, covering every SA21 case.

Coverage must fail on any vector field that is ignored. Do not dispatch on vector ID to
invent expected values.

**Planner harness.** `run_planner` hardcodes `keep_moving: false` and `bob_dir: None`.
Parameterize it, then add one typed-routing test with a real temporary `bob_dir`, where
relation rows of different task types route to different destinations. That test must
show that routing does not reorder relation groups or numbers.

**Capture compatibility.** Replace
`integrate_capture_compatibility_stays_byte_identical_without_relation`, which calls one
`relation: None` closure twice. The replacement is CLI tests over real temporary vaults
in `tests/cli/capture/`, reusing the existing capture test harnesses and fixtures. Use
three configs:

- `relations.successor_labels` absent;
- `relations.successor_labels` empty;
- populated with a label that overlaps neither the cancel nor the complete labels.

For managed Done, a Pomodoro close, a complete-label close, and cancel recovery, assert
that JSON output, human output, and resulting files are byte-identical across the three
configs. Use deterministic clocks and paths. Cover selected and unselected writes,
fresh/unblocked/keeps cleanup, Ready→Next, missing-ID minting, and repeated park then
done. Where the CLI cannot express a case, reuse the existing engine-level suites rather
than duplicating them. The populated config must not add any relation field to the
output.

## 6. Verify

- Run `just fmt`, the focused `cargo test --lib task_complete::`, the new CLI tests,
  `just check`, and `just docs-check`. Do not run `just check-full`.
- Run long checks in the foreground with a generous explicit timeout, for example
  `timeout` 3600000 ms. If one is killed by its timeout, rerun it with a larger one.
- An unrelated failure needs proof: show that it reproduces identically on the clean
  base. Handle it through `/sase_new_task`, citing `bob-cli-fs.2.1` as the proposing
  bead. Epic-caused failures stay in this tale.

## 7. Closeout (same turn, final step)

Do not wait for this tale's commit, SHA, push, or CI. The host commits after the turn
ends.

1. Re-read `bob-cli-fs.2.1`, its three phases and their notes, and the commits after
   `cba88d0d`. Integrate anything new that should use or conflicts with the relation
   core.
2. Run `sase bead epic-symbols bob-cli-fs.2.1` and
   `sase bead epic-symbols bob-cli-fs.2`; both were empty at the re-audit. Resolve any
   entry (wire it up, privatize it, delete it, or add a valid non-test pragma). Re-key
   an entry only to a verified still-open downstream bead (`bob-cli-fs.3`, `.4`, or
   `.5`) that needs it.
3. Close the epic normally:
   `sase bead close bob-cli-fs.2.1 --note "<what was repaired, tests and checks run with results, integration review, and triage outcomes>"`.
   The triage outcomes to restate:
   - `.1` notes #1/#2: one epic-caused vector-parity issue, resolved by bob-plugins
     `e9f2fb7` and deployed clean from `806ba69`;
   - `.3` note #1: declined, already resolved by `b483c93b`;
   - `bob-cli-fs.8.1.land` note #3: the same parity issue;
   - no distinct non-epic follow-up was found.

   Never use `--force` to make the close succeed. If stale symbols block it, fix them
   and retry.

4. Run `just symvision` if it exists; at re-audit this repo had no such recipe, so
   record that instead.
5. Mark the plan done:
   - run `sase repo open plans -r "<why>"`;
   - re-read the PLAN path from a fresh `sase bead read bob-cli-fs.2.1`;
   - in that checkout, set `status: done` in the frontmatter of
     `202611/relation_successor_core.md`.
6. Run `sase bead read bob-cli-fs.2.1 -r "Need the parent link"`. The parent is phase
   `bob-cli-fs.2`. Verify that the closed child delivered that phase's whole
   policy/copy/classifier/planner scope, and recheck its epic-symbols. Then close only
   that phase:
   `sase bead close bob-cli-fs.2 --note "<core completion, child landing, and verification evidence>"`.
   Do not close outer epic `bob-cli-fs` or mark `starts_after_relation.md` done; its
   waiting land agent owns them. If the parent link has changed or the phase is
   incomplete, add a note on it describing the blocker and report it.
7. Finish with `/sase_final`, declaring commits for bob-cli and the plans repo. Report
   which beads closed and the exact checks run.
