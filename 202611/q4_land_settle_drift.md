---
tier: tale
size: medium
title: Finish bob-cli-q4 settle drift and close the epic
goal: "Capture and `bob ref today` report post-settle line coordinates and settle every
  note they write. The 17 settle-drift CLI nodes recorded on bob-cli-q4 pass against the
  settle contract, and the bob-cli-q4 epic is closed.

  "
proposed_by: bbugyi200.athena.bob-cli-q4.land
proposed_with: claude/opus
bead: bob-cli-q4
create_time: 2026-11-26 06:05:53
status: wip
---

- **PARENT:**
  [202611/better_o_gesture_settle.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/better_o_gesture_settle.md)
- **BEAD:**
  [bob-cli-q4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q4/README.md)

# Plan: finish bob-cli-q4 settle drift and close the epic

## Context

This is the landing tale for epic `bob-cli-q4` ("Better o / Ctrl+Shift+O and
gesture-settled task structure", `plan:202611/better_o_gesture_settle.md`). All 13 phase
beads are closed. The landing review confirmed their work and their notes. It also filed
every distinct follow-up as `bob-cli-qc`, `bob-cli-qd`, and `bob-cli-qe`, and recorded
the declined proposals in the epic's latest note. That triage is done, so do not redo
it. One piece of epic work is left. The epic's own capture and command-settle phases
(`bob-cli-q4.6`, commit `007ef5ed`; `bob-cli-q4.7`, commit `08a708d3`) left **17
`--test cli` nodes red**, and epic notes #1, #3, #4, #6, and #7 record them. The lander
reproduced them on master `292d60c0`:

```bash
cargo test --test cli -- capture::inbox_route capture::note_pull capture::project_note \
  capture::task_theme_writes ref_library::today task_status_hooks
# 332 passed; 17 failed
```

The failing nodes:

- `capture::inbox_route::ir_dependency_touch_follows_its_routed_task`
- `capture::inbox_route::ir_routed_link_moves_with_repair_stamp_follow_and_open_notes`
- `capture::note_pull::{np2_pick_all_matches_cli_json_and_human_fixtures, np_move_groups_follow_requested_sources_in_cli_fixtures, np16_inbox_choice_and_routed_pick_match_cli_fixtures}`
- `capture::project_note::{capture_move_landing_repairs_moved_reports_after_child_edits, capture_project_note_choice_answers_match_authored_bullets, capture_project_note_task_links_write_the_worked_example}`
- `capture::project_note_offer::*`: six `offer_*` nodes, all panicking at
  `assert_offer_lines_match_final_note`
- `capture::task_theme_writes::new_named_task_reports_and_writes_the_resolved_theme_exception`
- `ref_library::today::today_matches_unnamed_capture_and_named_capture_remembers_theme`
- `task_status_hooks::sync::task_status_hooks_groups_area_project_tasks_after_final_statuses`

The failures have three causes. Two are real behavior gaps; the third is stale tests.

### A. Capture reports pre-settle line numbers (behavior bug)

`settle_capture_notes` in `src/native/capture/plan.rs` runs after `apply_inbox_routes`
and the type-agreement stage. It settles each changed note, but it forwards only two
things through `settled.line_map`: the task-block tracker (`remap_settled_note`) and
`items[].result.parent_line`. Every other 1-based line it emits still points at the
pre-settle bytes:

- `inbox_route.candidates[].moved.line` (in `applied.report`) and
  `RoutedTaskMove.new_line` (in `applied.moves`, which become
  `PlannedCaptureBatch.routed_landings`). `open_notes`, through
  `open_note_for_routed_task` in `src/native/capture/open_notes.rs`, and the jump
  landing in `src/native/capture/jump.rs` read these against post-settle text. So the
  Mac's `⌃↩ Open` lands on the wrong row, and `open_notes[0].block_id` comes out `null`.
  Test evidence: tracker line 9 vs `moved.line` 6.
- `project_note.moved_tasks[].line`, where the test reports 13 but the task sits on line
  18, and `project_note.successors[].line`, where the line is "not a task".
- Any other line coordinate a capture result or the batch emits for a settled note.
  Audit `CaptureItemResult` (`src/native/capture/output.rs`), its nested reports
  (project_note, note_pull, snooze, requirements), and the batch-level reports
  (`section_blocks`, `pomodoro_blocks`, jump landing). Forward each one, or re-resolve
  it by identity.

This breaks the settle contract in the epic plan's capture-settle phase: "Map
`items[].result.line` (and any other emitted line numbers) through `line_map`, so Bob
Mac Capture's ⌃↩ Open lands on the post-settle line, with no Swift changes"
(`decisions:mac-capture-is-a-thin-client`).

### B. `bob ref today` bypasses the settle choke point (writer gap)

`link_existing_task` in `src/native/capture/existing_task_link.rs` plans the same
existing-task Pomodoro link as `bob capture '^route:block-id[#name]'`. It then builds
its own `PlannedCaptureBatch` and commits it, so it never passes through
`settle_capture_notes`. The link raises the task Ready→Next
(`decisions:task-lanes-are-sticky`). Capture then settles the note into
`### Next & In Progress` with a badge row, but `ref today` leaves the note unsettled.
The parity test
`ref_library::today::today_matches_unnamed_capture_and_named_capture_remembers_theme`
expects byte-identical notes, so it fails. The q4.7 writer audit missed this writer.

### C. Stale expectations pin unsettled bytes

Some tests still expect the pre-epic bytes. Their new output is correct under the settle
contract (`docs/task-status-hooks.md` § "Writers settle note-local structure"):

- `task_theme_writes` expects no badge row or `### Next & In Progress` group after
  capture.
- `task_status_hooks::sync` expects capture to leave a stale `⚪ 1 open` badge for
  reconcile to refresh. Capture now writes `⚪ 2 open` itself, so the follow-up
  reconcile must plan nothing for that note.
- `project_note` (choice answers, worked example) expects authored order. Settle now
  moves `[*]` rows into `### Next & In Progress` and adds the badge row.
- The note_pull JSON fixtures under `tests/fixtures/note_pull/` (and inline
  expectations) pin pre-settle `task_blocks[].line` values.

## Ground rules

- Rust is the authority, and the settle pass already matches reconcile byte for byte
  (`tests/fixtures/task_settle/`). Fix what writers report, never what settle produces.
  Never weaken a product invariant just to turn a test green. A test that asserts "the
  reported line holds this task" or "dry-run JSON equals the real run" must keep that
  assertion.
- Do not touch the JS twins or bob-plugins. No plugin change is needed.
- Read `docs.md` and `cli_rules.md` reference memory with `/sase_memory_read` before you
  edit `docs/` or CLI help. Then run `just docs-gen` and `just docs-check`.
- Never edit `Cargo.toml` versions or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).
- Do not edit memory notes (epic decision `memory_settle_record = no`).
- Do not create beads. Put any new discovered work on `bob-cli-q4` as a
  `PROPOSED FOLLOW-UP:` note.

## Steps

### 1. Forward every capture line coordinate through the settle line map (A)

- Change `settle_capture_notes` (or a helper it calls) so the settle line map reaches
  every report that carries a line in a settled note. Pass in the inbox-route outcome
  (`applied.report` and `applied.moves`) before it moves into the batch. Remap these
  through `settled.line_map`, converting 1-based to 0-based and back:
  `RoutedTaskMove.new_line` for `dest == target`, `moved.line` on the matching report
  candidate, and the `project_note` `moved_tasks[].line` and `successors[].line`. Also
  remap any other line field step A's audit finds. When a coordinate has no map entry
  (its line was removed), re-resolve it by block ID in the settled text. Never leave a
  stale number.
- `open_notes`, the jump landing, and `task_blocks` must then agree. Assert
  `open_notes[].line`, `inbox_route…moved.line`, and the tracker's `task_blocks[].line`
  name the same physical line for a routed task.
- Keep dry-run and commit byte-identical, both JSON and files. Both paths already go
  through this choke point.

### 2. Settle `bob ref today` exactly like capture (B)

- Route `link_existing_task` through the same settle as capture before it stages or
  commits. Expose `settle_capture_notes`, or a thin shared wrapper, to
  `existing_task_link.rs` instead of copying it. Remap the result's emitted lines the
  same way. Dry-run must report the settled outcome without writing.
- `bob ref today <ref>` and `bob capture '^route:block-id'` must leave byte-identical
  notes and daily files. Add `ref today` to the writer list in
  `docs/task-status-hooks.md` § "Writers settle note-local structure". Also add a
  `ref today` step to `tests/cli/settle_acceptance.rs` that asserts nothing note-local
  is left for reconcile and tidy stays at zero, following the existing helpers there.

### 3. Re-baseline the stale expectations (C)

- For each node in group C, regenerate the expectation from the settled output. Check
  every changed line number or byte against the final file, never paste blindly. Each
  reported `line` must hold its task, groups must match each task's status, and the
  badge counts must be right.
- `task_status_hooks::sync`: after the Ready capture, assert the badge already reads
  `⚪ 2 open`. Assert the next reconcile plans no grouping or badge change for
  `alpha.md`; keep the later promotion leg as it is.
- `project_note` choice equivalence: keep the choice-vs-authored byte equality. Update
  the order assertion to the settled layout: intake rows in authored order, `[*]` rows
  under `### Next & In Progress`.
- note_pull fixtures: update the JSON (and any human) fixtures under
  `tests/fixtures/note_pull/`, or wherever `note_pull.rs` loads them from. Watch
  `np_move_groups_follow_requested_sources_in_cli_fixtures`: the current output reports
  `alpha` and `beta` at line 16. If both sit in the same `relative_target`, a tracker
  remap is colliding. Fix it in step 1; do not pin it in a fixture.
- Rerun the targeted command from Context until all 17 nodes pass, then run
  `cargo test --test cli -- settle_acceptance ref_library capture`.

### 4. Docs and cleanup

- `docs/capture.md`: state that capture settles every Markdown note it changes. That
  covers status groups and the badge row on eligible area and project notes, plus the
  local future-schedule `[?]` rule. Say the settle runs in the same write and that
  dry-run previews the settled bytes. Say every emitted line coordinate is post-settle:
  `parent_line`, `task_blocks`, `section_blocks`, `open_notes`, `inbox_route` moved
  lines, `project_note` moved and successor lines, and the jump landing. Link
  `task-status-hooks.md#writers-settle-note-local-structure`. Rewrite the paragraph near
  "When `bob task reconcile` … has generated a status-count badge row directly below the
  heading" so it no longer implies only reconcile writes badges. Document the
  `ref today` settle on the ref reference page that describes `bob ref today`.
- Remove the stale
  `#[allow(dead_code)] // Consumed by the capture and command writer phases.` attributes
  on `SettleContext` and its `impl` in `src/native/task_settle.rs`. Those consumers have
  landed. Keep clippy clean.
- Run `just docs-gen` and `just docs-check`.

### 5. Verify

- Run `cargo fmt --all -- --check` and the clippy gate from `just check`.
- Run `just check`. The base tree is already red outside this epic. Note #7 on
  `bob-cli-q4` records lib 8184 passed / 43 failed and CLI 4177 passed / 55 failed at
  `ae78dbb2`: 80 nodes of bob-cli-ph drawer-cutover drift, 1 bob-cli-q9 wispr node, and
  these 17. Pass means the 17 nodes are gone and every remaining failure is in that
  ph/q9 set. Diff your failure list against a clean-base run, or against the node list
  in note #7 and its predecessors. Any new failure, or any remaining failure in capture,
  settle, `ref_library`, `task_status_hooks`, or `settle_acceptance`, is your work. Do
  not run `just check-full`.

### 6. Commit marker

The commit that changes capture's emitted line coordinates and `ref today` bytes is a
JSON and vault-byte contract break. Its Conventional Commit subject must carry `!`, for
example `fix(capture)!: report post-settle lines and settle bob ref today`. Add a
`BREAKING CHANGE:` footer. The footer says capture, `ref today`, task move, split,
merge, gmail, gkeep, and ref writers now write settled notes (status groups, badge row,
future-schedule `[?]`), and that capture JSON line fields are post-settle. It also says
plain Task Split now writes `LINKED FROM` in the managed-link zone. Epic commits
`007ef5ed`, `08a708d3`, and `814b6583` shipped those byte changes without the `!`
marker. No release has gone out since v0.6.0, so this footer puts the break into the
next changelog.

### 7. Close out epic bob-cli-q4 (final step)

Do this in the same turn as the code. It must not wait for this work's own commit SHA,
push, or CI.

1. Run `sase bead epic-symbols bob-cli-q4`. For every listed `--epic-symbol` entry,
   resolve the symbol by wiring it up, privatizing it, adding a non-test pragma, or
   deleting it, per the Symvision epic-whitelist policy. Re-key a Justfile line only to
   a still-open later bead that still needs the exemption. The lander saw none at
   landing time.
2. Close the epic with a verification note:

   ```bash
   sase bead close bob-cli-q4 --note "<verification>"
   ```

   The note summarizes both what the lander verified and what this tale did:
   - All 13 phases closed and verified against source and commits; `bob-cli-py` and
     `bob-cli-pn` closed as fixed.
   - DECISIONS honored: `o_in_group=intake_end`, `settle_cursor=stay`,
     `memory_settle_record=no` (no decisions record written), and `wasm_spike=yes`
     (filed `bob-cli-qe`).
   - Follow-ups `bob-cli-qc` and `bob-cli-qd` (snoozed 7d) filed, and the declined
     proposals recorded in the epic's land-triage note.
   - Integration review: concurrent snooze, tie, Go To, drawer, and freshness commits
     need no change. Snooze add already settles. The bob-plugins commits after
     `8645ed55` are status-neutral.
   - This tale's fixes, with the targeted test commands and their pass counts, plus the
     `just check` failure diff showing only bob-cli-ph/q9 nodes remain.

   If the close is rejected for leftover epic symbols, finish that cleanup and close
   again. If it is rejected because named phases were never completed, investigate:
   every phase is closed today. Never use `--force` just to make the close succeed.

3. Run `just symvision` if the recipe exists. At landing time the justfile had no such
   recipe; say so in your final report.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202611/better_o_gesture_settle.md`. Its path is the PLAN path that
   `sase bead read bob-cli-q4 -r "Need the plan path for closeout"` prints. It currently
   reads `status: wip`. Change nothing else in that file.
5. `bob-cli-q4` has no `parent_bead`, so no ancestor plan needs handling.
