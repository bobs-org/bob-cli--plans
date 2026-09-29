---
tier: tale
size: medium
title: Split the task status grouping transform into four modules
goal:
  Keep task status grouping behavior and its 36 unit tests intact while reducing every
  resulting Rust file to at most 1500 lines.
proposed_by: bbugyi200.apollo.bob-cli-2f.9
bead: bob-cli-2f.9
create_time: 2026-09-28 20:22:15
status: wip
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.9.md)

# Split task status grouping

## Scope and constraints

Complete phase `bob-cli-2f.9` of the existing split-largest-Rust-files epic. The current
`src/native/task_status_groups.rs` is 2,991 lines, with 36 `#[test]` cases. This is a
structural refactor only. Preserve all algorithms, comments, assertions, observable
output, external `native::task_status_groups` paths, and test names. Keep visibility as
narrow as possible. Do not change any other oversized file or close the parent epic.

## Implementation

1. Record the baseline test count and inspect external uses of `task_status_groups`.
   Move the file with
   `git mv src/native/task_status_groups.rs src/native/task_status_groups/mod.rs` so
   history follows the module. Keep the original module documentation and external
   constants, types, and `transform` entry point in `mod.rs`.
2. Cut the original file into four cohesive files, preserving item order within each
   moved group: `mod.rs` for the API, transform, container rewrite, and child
   classification (roughly original lines 1–712); `parse.rs` for marker and badge
   parsing, parsed containers, direct pieces, task blocks, subtrees, and list item
   parsing (roughly 713–1366); `emit.rs` for the grouping plan, emission, newline
   handling, source lines, and heading tree (roughly 1367–2021); and `tests.rs` for the
   entire current unit test block (roughly 2022–2991). Add a one-line module doc to each
   child. Use minimal `mod`, `use`, and `pub(super)` changes so sibling modules can
   access the moved items; keep the old external API accessible from the parent. Adjust
   only paths that the move makes necessary, including any relative fixture paths if
   encountered. If actual item dependencies suggest moving an item across the proposed
   boundary, prefer cohesive ownership while keeping four files and the 1500-line limit.
3. Run `cargo fmt`, compare the resulting test list and 36 local tests with baseline,
   and run `just all` (format check, Clippy on all targets and features, and the full
   test suite). Resolve any new compiler or Clippy warnings from the split. Check every
   touched Rust file is at most 1500 lines, and review the diff for unchanged test
   assertions and transform logic.
4. Run `sase bead epic-symbols bob-cli-2f.9`; resolve any symbols still assigned to the
   phase or re-key their Justfile lines to a still-open bead. Record any unrelated
   clean-base check failure as a `PROPOSED FOLLOW-UP:` note on this phase. Close only
   `bob-cli-2f.9` with
   `sase bead close bob-cli-2f.9 --note "<verification and final file line counts>"`.

## Acceptance

- `task_status_groups` retains its external paths and behavior, including all 36 unit
  tests.
- The four resulting files each contain at most 1500 lines.
- `just all` passes, with no new compiler or Clippy warnings, or an identical clean-base
  failure is documented on the phase bead.
- No `--epic-symbol` entry remains keyed to `bob-cli-2f.9`, and only the assigned phase
  is closed.
