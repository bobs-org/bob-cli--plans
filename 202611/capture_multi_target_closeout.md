---
tier: tale
title: Finish multi-target capture identity, completion, and landing
goal: Repair remaining target-run defects, verify integration, and close bob-cli-gm
  with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-gm.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gm
status: done
---

- **PARENT:**
  [202611/capture_multi_target.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_multi_target.md)
- **BEAD:**
  [bob-cli-gm](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gm/README.md)

# Finish multi-target capture identity, completion, and landing

Complete the remaining work for **bob-cli-gm**, then close that epic in this same coding
turn. This tale has no land agent after you. Do not wait for your commit SHA, a push, or
CI to close the epic: local verification and the final code can land together through
`/sase_final`.

## Scope and binding decisions

The approved contract is `plan:202611/capture_multi_target.md`. Its grammar child is
`plan:202611/capture_target_runs_finish.md`. Read them with `sase artifact read`, and
read bob-cli-gm's current notes before editing. All three direct phases and all
descendants were closed at the landing audit. Most of the feature is implemented; do not
repeat those phases.

Keep final decisions **split_edges = refuse** and **memory_bulk_capture = no**. No
memory edits are authorized. Preserve grammar revision **9**, singleton behavior, exact
source byte ranges, atomic staged planning, one clipboard read and one schedule roll per
run, and the existing first-member legacy fields. Do not edit package versions or
CHANGELOG.md. No Mac or plugin change is needed.

Audit base: `66828cfd`. A fresh origin fetch found master and origin/master identical,
with no PR base divergence. This checkout did not have a compiled bob at audit time; the
globally installed bob still reported grammar 7 and was rejected as evidence. Build and
test the current checkout. The defects below are established by source inspection; first
capture failing regressions against the current source rather than citing the installed
binary.

## 1. Preserve exact resolved block identity

`src/native/capture/plan.rs::validate_fanout_target_group` resolves each member through
`NoteTaskScan::lookup_capture_task`, then calls `to_ascii_lowercase()` on the resolved
block ID before checking its duplicate key. That is inconsistent with
`src/native/note_tasks.rs::by_block_id` and the lexical run identity in
`capture_language/target_runs.rs`, both of which compare the block ID exactly. Two
distinct tasks `^A` and `^a` in the same note therefore collide at planning.

Keep the canonical note-path identity used to detect aliases, but retain the exact
resolved block ID. Add an end-to-end regression in `tests/cli/capture/multi_target.rs`:
two open tasks with IDs differing only by case must each receive the note from
`@work+A @work+a note`; verify the written bytes and both fanout results. Preserve the
existing same-ID and symlink-alias duplicate refusals, including suffix variants. A
refused duplicate must leave every vault file unchanged. Do not change global task-ID
matching rules.

## 2. Complete every member of the parser's selected run

`src/native/capture_language/completion.rs::completion_field_at` still uses the
singleton `extract_terminal_markers`, then calls `completion_marker_index` on that token
stream. It masks type markers but does not mask neighboring percent directives for this
selection. The execution/editor run probes in `item.rs` instead mask directives and use
`extract_terminal_markers_crossing_runs`.

For example, with the cursor after the second plus in `@work+a s:3 @dev+ note`, the
current completion scanner leaves `s:3` in place, selects only the leading `@work+a`,
and returns no field for `@dev+`. The same loss occurs for interleaved `%c`; a leading
schedule before the run also blocks selection. These drafts already belong to the
grammar's target-run contract.

Reuse the shared run selection and offset-preserving masking for completion. Make
complete or partial run members and inner-edge route fragments use their own
route/block/drawer field, including transparent type, directive, relationship, and
schedule/priority tokens. Keep directive completion itself and the other specialized
completion claims ahead of the generic route claim. Avoid changing singleton semantics
by applying the multi-target rules only where they actually select or extend a run; the
original singleton tests are compatibility requirements. Do not introduce a second
capture lexer.

Extend the existing completion matrix with cases such as these (the vertical bar denotes
the cursor, not draft text):

- `@work+a s:3 @dev+| note`
- `s:3 @work+a @dev+b#re| note`
- `@work+a $1 @dev+| note`
- `@work+a %c @dev+| note`
- `@work+a %c @de| note`
- `note s:3 @work+| @dev+b`
- `note @work+| %c @dev+b`
- A three-member run, a child-line trailing run, and a later item after Unicode.

Assert context, target ownership, replacement range, and original byte offsets, then
verify representative cases through `capture-complete` CLI fixtures with real task and
drawer candidates. Preserve the existing `Discuss @de| later` literal behavior,
split-edge diagnostics, first-member bullet drawer menu, separator rewrite matrix, and
non-absorbable `@@` notice. Ensure rewrite uses the same selection for the new
interleaved cases; repair only demonstrated divergences. Add a regression for any such
repair.

## 3. Finish output and integration verification

`capture/output.rs::print_fanout_summary` currently prints its header with a plain
`println!`. Use the established `Styler` dim rendering required by the approved
human-output contract. Preserve exact plain output under `NO_COLOR` and one header per
run. This is a small presentation correction, not a new output format or JSON revision.

The landing audit reviewed the 15 non-gm commits since the first gm commit `d1f34842`,
including note-pull changes `093dcead`/`4db3200c`, project move changes
`6650faa3`/`14a57732`/`ddb0a73f`/`b738d8f2`, shared drawer and transplant helpers
`aac3947c`, attachment repair `dd90f8c7`, and task split/merge work through `e253ce17`.
Preserve the existing integration: task tracker remapping before ordinary forwarding,
staged source/destination snapshots, note-pull offer ownership and numbering, and
omitted optional `moved_from`/`merged_from` fields for ordinary captures. Period-note
changes need no capture integration.

Add focused mixed-draft regressions using the existing note-pull and project-note
fixtures, rather than a new fixture framework:

- A target run before or after a lone-note pull must retain ordered capture results and
  unique offer ownership/numbering; any explicit selections still refer to the
  advertised tasks.
- A project move followed by a run naming tasks at their staged destination must read
  the moved tasks without an on-disk intermediate write. Also check a run before the
  move, so the final task-block report points at the moved location and preserves the
  shared note text.
- A later failing member in such a mixed draft must roll the whole draft back. Dry runs
  must leave vault bytes unchanged.

Refresh the drift review for any new commits since `66828cfd`. Integrate a demonstrated
conflict with these repairs within this tale; do not implement unfinished sibling
task-transfer or merge-label features for bob-cli-go.

## 4. Verify the repairs

Run focused completion/rewrite, target-run grammar, multi-target CLI, and new
mixed-draft regressions. Use locally built commands in an isolated scratch HOME/XDG
vault with `BOB_REF_JOBS_KICK=off` for live verification, never Bryan's vault. Confirm
current `capture-parse` returns grammar 9, one item and two targets, while capture
dry-run returns two results with fanout context.

Run formatting repair as needed, `git diff --check`, and the canonical **`just check`**
gate through the SASE tool/monitor workflow. Do not run `just check-full`. If a monitor
is needed, its continuation must finish this tale's epic closeout rather than waiting
for another land agent.

The previous phase recorded a passing monitored `just check` at this base; rerun after
the new fixes. Generated CLI help whitespace in `task.md` was already repaired by
`66828cfd`; preserve that correction. Regenerate CLI help only if help changes, reading
docs memory first.

`uv run --project tools/docs bob-docs gen --check` currently exits 2 because
`sase/memory/glossary/task-merge.md` has no front matter. This was independently
reproduced on `66828cfd` and corroborated on active causal epic **bob-cli-go**; go.13
owns its accepted glossary output. Recheck `just docs-check` and record the result. This
unrelated existing failure does not require a memory edit or block this epic's completed
repairs. Route any genuinely new failures by cause; never dismiss a failure caused by
the current tale.

## 5. Close bob-cli-gm in this turn

This is the final implementation step, before `/sase_final`. Do not order it after a
commit, SHA, push, or CI result.

1. Re-read bob-cli-gm, its three direct children, nested gm.1.1 and its four children,
   all new notes, and their linked plans through audited commands. Confirm no descendant
   or linked plan is unfinished and all approved requirements and post-audit drift are
   covered. The earlier grammar repair was completed in `ebedfca8`; do not repeat it.
2. Run `sase bead epic-symbols bob-cli-gm`. Resolve every listed exemption (wire up,
   privatize, add a policy-valid non-test pragma, or delete), or re-key it only to a
   still-open later bead that actually needs it. The audit found none and the justfile
   had no symvision recipe; recheck current state.
3. Close normally with
   `sase bead close bob-cli-gm --note "<verification and integration evidence>"`.
   Include the exact checks and all follow-up outcomes below. If symbols block close,
   fix them and retry. If descendants block close, finish their scope; never use
   `--force` merely to succeed or to advance a successful landing.
   Cancellation/supersession requires a deliberate reason and the matching non-done
   resolution, not a shortcut.
4. After closing, run `just symvision` if the recipe exists and confirm clean results;
   if absent, record that fact.
5. Use `/sase_repo` to open the plans repo before editing it. In the printed plans root,
   set `status: done` in **`202611/capture_multi_target.md`**, the plan linked by
   bob-cli-gm. Read plan artifacts through `sase artifact read`. Include the plans
   repository in the final declaration.
6. Re-read bob-cli-gm with reason `Need the parent link`. It had **no parent_bead** at
   audit, so no ancestor close is expected. If the parent link changed, follow the
   original landing rules: a phase parent is verified and closed normally, leaving its
   epic to its land agent; a plan parent requires every descendant, note, linked plan,
   prior landing note, and post-child drift rechecked before normal close, symbol
   cleanup, and plan completion. Stop at an incomplete or ambiguous parent and record
   its blocker.

Carry these already completed `/sase_new_task` dispositions into the close note:

- **gm.1 note #1:** the incomplete-member `block_id` example was reconciled by the
  approved child contract preserving existing singleton `Need::Task`. No independent
  task; preserve that behavior.
- **gm.2 #1, gm.3 #2, gm.1.1.4 #1:** one task-merge glossary front-matter failure,
  independently reproduced and corroborated on active **bob-cli-go**. No duplicate task
  or unauthorized memory edit.
- **gm.3 #1:** task-split catalog, completion kinds, help, and lint fallout was repaired
  by `ebedfca8`. Declined a stale new CI task; confirm current checks after these
  repairs.
- **gm.1.1.4 #2:** catalog count mismatch was repaired by `dd90f8c7` and `ebedfca8` (now
  70). Resolution evidence was appended to existing **bob-cli-de**; no stale failure +1
  or new task.
- **gm epic notes #1–2:** generated `task.md` whitespace was repaired by `66828cfd`;
  preserve it.

The originating lander performed same-type/all-type searches, the last-week task sweep,
and active-epic ownership review before proposing this tale. Any new non-epic discovery
requires `/sase_new_task`; defects caused by this epic remain part of this closeout.
Finish through `/sase_final` with repository commit decisions for all authored changes
after the epic is complete.
