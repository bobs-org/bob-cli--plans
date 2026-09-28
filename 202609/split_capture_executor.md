---
tier: tale
title: Split the capture executor into directory modules
goal:
  Preserve capture behavior and all tests while every resulting capture module is at
  most 1500 lines.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-2f.2
bead: bob-cli-2f.2
status: done
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.2.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2f.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2f.2.md)
- **COMMITS:**
  - [e73e2e9](https://github.com/bobs-org/bob-cli/commit/e73e2e985b855f451ed2c41e4581b3200c161674)
    — refactor(capture): split capture executor into directory modules

# Split the capture executor into small directory modules

## Goal and scope

Complete assigned phase `bob-cli-2f.2` (`split-capture`) of the parent epic
`bob-cli-2f`. Replace the current 12,340-line `src/native/capture.rs` with
`src/native/capture/mod.rs` and cohesive child modules. Every resulting Rust file
created or touched by this phase must have at most 1,500 lines, with a practical target
of 300–1,200 where useful. Preserve all capture behavior, existing external Rust paths,
and all 105 unit tests currently in `capture.rs`. Do not split `capture_language.rs` or
`capture_pomodoro_close.rs`; later phases own those files. The parent epic design is
`sase/repos/plans/202609/split_largest_rust_files.md`, section "Phase 2 — Split
src/native/capture.rs".

The old file contains CLI parsing (roughly lines 1–694), capture and item planning
(694–1918), batch planning (1918–2311), Pomodoro operations (2311–6561), sub-bullet
handling (6561–7009), staged commit and insertion (7009–8216), output/error handling
(8216–9370), and a single unit-test module (9371–12340). These are landmarks, not cut
points: move complete items and their comments after checking actual dependencies.
`tests/cli/` already holds the integration tests and is outside this phase.

## Implementation

1. Record the baseline on the current tree: `wc -l src/native/capture.rs`,
   `rg -c '^    #\[test\]' src/native/capture.rs`, and `cargo test --lib -- --list`
   (save/count the full test list for comparison). Inspect each top-level item and the
   old module imports before moving code. Use
   `git mv src/native/capture.rs src/native/capture/mod.rs` so history follows the move.
2. Keep `mod.rs` small: child `mod` declarations, `COMMAND_NAME`, `INBOX_FILE`, `run`,
   the `capture` entry point, and narrow re-exports for callers outside `capture`. Keep
   `pub(crate) use super::capture_language::is_route_token`. Existing paths such as
   `capture::route_label`, `capture::line_spans`, `capture::Placement`,
   `capture::AdjustRange`, and the functions/types imported by
   `capture_schedule_log.rs`, `capture_sections.rs`, `capture_work_log.rs`,
   `capture_pomodoros.rs`, `capture_task_toggle.rs`, `randomize_plan.rs`, and `gkeep/`
   must remain reachable. Prefer `pub(super)` for cross-child items; retain existing
   `pub(crate)` only where callers outside this directory already require it.
3. Move code substantially verbatim into cohesive files under `src/native/capture/`:
   `cli.rs` for clap construction/request parsing; `plan.rs` for batch/item planning and
   capture formatting/priority helpers; `project_note.rs` for project-note planning;
   `batch.rs` for `CaptureBatchPlanner`, snapshots and shared plan/details types;
   `pomodoro_start.rs`, `pomodoro_adjust.rs`, `pomodoro_close.rs`, `pomodoro_link.rs`,
   `pomodoro_insert.rs` for the respective operations; `task_toggle.rs` and
   `ensure_next.rs`; `sub_bullet.rs` for note/sub-bullet placement and managed-log
   parsing; `commit.rs` for staging, rollback, temporary files and clipboard saves;
   `sections.rs` for Markdown section/task/bullet insertion and line spans; and
   `output.rs` for result/error structs and human/JSON output. Split any proposed file
   further if it approaches 1,500 lines, especially `pomodoro_close.rs`; group tiny
   related files rather than leave arbitrary fragments. Preserve item order within moved
   groups, doc comments, and logic. Change only module declarations, imports, needed
   visibility and `super::` paths. Add a short `//!` summary to each new child module.
4. Split the old 105 unit tests into files below `capture/tests/` with `tests/mod.rs`
   for shared fixtures and test-only parse wrappers. A useful partition is
   grammar/routing (old lines 9403–10338), assembly/managed logs plus Pomodoro selection
   (10339–11256), task/bullet placement plus JSON contract (11257–12065), and
   started-Pomodoro placement (12066–12340). Adjust groups to keep every test file below
   1,500 lines, retain every assertion and test name, and use narrow imports from the
   child modules. Check any moved `include_str!`/`include_bytes!` path for new file
   depth.
5. Update only documentation references that would falsely call the new directory
   `capture.rs`, particularly `capture_language.rs` and `capture_task_toggle.rs`. Keep
   references to the later-phase `capture_pomodoro_close` module intact. Do not alter
   unrelated code to make a structural move appear clean.

## Verification and completion

- Run `cargo fmt`; compare the post-move `cargo test --lib -- --list` names/count
  against the saved baseline and confirm all 105 capture unit tests remain. Run
  `cargo test --lib native::capture`, `cargo test --test cli capture`, then `just all`
  (format, clippy with all targets/features, and full test suite). Fix any new
  compiler/clippy warnings caused by the split. Confirm every new/touched Rust source
  file has at most 1,500 lines with a file-level line-count check.
- Compare the diff for accidental logic or assertion edits and check
  `git status --short`; the phase is a structural refactor only. Report the final module
  layout and each line count.
- Before closing `bob-cli-2f.2`, run `sase bead epic-symbols bob-cli-2f.2`; resolve any
  remaining `--epic-symbol` entries or re-key their Justfile line to a still-open bead.
  Close only this phase with
  `sase bead close bob-cli-2f.2 --note "<verified test counts, checks, and line limits>"`.
  Do not close the parent epic or an ancestor. If a check fails identically on a clean
  base tree, record a `PROPOSED FOLLOW-UP:` note on this phase (citing any existing task
  bead), then close the phase. Do not create a new bead directly.
