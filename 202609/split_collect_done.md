---
tier: tale
title: Split collect_done into a directory module
size: medium
goal: Turn src/native/collect_done.rs into a directory module whose files stay at
  most 1500 lines, with the same behavior and the same 67 unit tests.
proposed_by: bbugyi200.apollo.bob-cli-2f.8
bead: bob-cli-2f.8
status: done
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.8.md)

# Plan: Split src/native/collect_done.rs into a directory module

This is the implementation plan for phase bead `bob-cli-2f.8` (`split-collect-done`) of
epic `bob-cli-2f`. The parent epic already sequenced the work as one file. One agent can
land it from this plan, so the plan is a tale. `medium` matches a bounded multi-file
move whose layout is already known: about 4,300 lines, a fixed external API, and
`just all` as the gate. It is larger than a one-function edit and smaller than another
epic.

Measured this turn, not from the epic's `d0c1692` snapshot: `src/native/collect_done.rs`
is 4,344 lines (code through `print_help` at line 2564, then `#[cfg(test)] mod tests`
with 67 `#[test]` functions). Re-measure with `wc -l` before cutting. If the file is
already at most 1,500 lines, stop and say so. The line numbers below are landmarks in
the current file.

## Rules

Move code verbatim. Keep doc comments, `//` section comments, and the relative order of
items inside each destination file. The move may change only `mod` and `use`
declarations, visibility, and `super::` paths. Leave every function name, test name, and
assertion as it is.

Every resulting file, tests included, stays at most 1,500 lines. Aim for roughly
300–1,200. `git.rs` lands near 220 lines; keep it as its own file because the phase
description calls out git as a module and cutting it further would fragment one
lifecycle. Avoid any other file under about 80 lines.

`src/native.rs` already has `mod collect_done;`, which resolves to
`collect_done/mod.rs`. Leave that declaration in place. Leave every other oversized Rust
file alone. Outside `src/native/collect_done/`, the only expected diff is none: callers
already use the paths re-exported below.

Match `src/native/projects/` (the previous phase): directory module, `use child::*;` in
the root, `use super::*;` in each child, `pub(super)` for sibling sharing, and
`pub(crate) use` only for items that are already `pub(crate)`. `rustfmt.toml` sets
`max_width = 80`. Run `cargo fmt` before the final check.

Give each new submodule, including each test file, a one-line `//!` summary. The current
file has no `//!` docs; leave the root the same way `projects/mod.rs` does.

## Target layout

Start with `git mv src/native/collect_done.rs src/native/collect_done/mod.rs` before any
edit, so history follows.

| File                          | Responsibility                                                                                                                                                                                                                        | Source landmarks                    | Approx. lines |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | ------------- |
| `collect_done/mod.rs`         | `COMMAND_NAME`, `DEFAULT_THRESHOLD`, `Args`, `run`, `run_collection`, `merged_output`, `write_stderr_output`, `parse_args`, `ParseResult`, threshold parsing, `print_help`, `mod` declarations, glob imports, `pub(crate)` re-exports | 16–40, 288–491, 2465–2564           | ~400          |
| `collect_done/plan.rs`        | `CollectionPlan`, `FilePlan`, `LinkRepairPlan` and their impls, `ArchiveWrite`, `apply_file_plan`, `apply_link_repair_plan`, `build_collection_plan`, `validate_dependency_identity_index`                                            | 42–229, 231–234, 492–527, 1047–1246 | ~450          |
| `collect_done/git.rs`         | `GitState`, `GitPrepareError`, `GitDetection`, prepare/detect/touched paths/finish/stage/commit/push, `collect_done_commit_message`                                                                                                   | 237–257, 528–717                    | ~220          |
| `collect_done/archive.rs`     | `DeduplicatedArchiveAppend`, `ARCHIVE_TYPE_LINE`, `DONE_TASKS_KEY`, archive block-id dedup, archive and source frontmatter, append, line endings, `atomic_write` and its temp-path helpers, `read_optional_string`, `fs_error`        | 18–19, 277–281, 718–1046            | ~360          |
| `collect_done/link_repair.rs` | `DEPENDENCY_FIELD_RE`, `MovedBlockTargets`, `MovedBlockTarget`, `LinkRepairResult`, `NoteIndex`, link repair in plans and notes, dependency metadata repair, wiki and markdown link rewriting, `missing_planned_contents`             | 21–26, 1248–2095                    | ~870          |
| `collect_done/transform.rs`   | `Transform`, `BlockIdOccurrence`, `TaskLine`, markdown collection, archive paths and wiki links, `transform_markdown`, block-id scanning, collectible task lines                                                                      | 259–286, 2096–2464                  | ~420          |
| `collect_done/tests/mod.rs`   | Shared fixtures only: `TEMP_COUNTER`, `os_args`, `write_file`, `TempDir`, `current_time_nanos`, `remove_dir_all_if_exists`                                                                                                            | 2584 and 4279–4344                  | ~90           |
| `collect_done/tests/unit.rs`  | Contiguous tests from `parses_default_threshold` through `preserves_crlf_when_adding_done_tasks_frontmatter`                                                                                                                          | 2586–3523                           | ~940          |
| `collect_done/tests/plan.rs`  | Contiguous tests from `scans_markdown_files_with_exclusions_and_threshold` through `duplicate_moved_block_ids_do_not_rewrite_links`                                                                                                   | 3524–4278                           | ~760          |

`DEPENDENCY_FIELD_RE` has a single caller, `repair_dependency_metadata`, so it lives in
`link_repair.rs` with that function. `ARCHIVE_TYPE_LINE` and `DONE_TASKS_KEY` are used
only by the archive frontmatter helpers, so they live in `archive.rs`. `Transform`,
`BlockIdOccurrence`, and `TaskLine` live in `transform.rs`. `GitState` and the other git
enums live in `git.rs`. `ArchiveWrite` stays beside `apply_file_plan` in `plan.rs`
because `run_collection` matches on the value that function returns.

If a recommended file would pass 1,500 lines, cut it again on a cohesion boundary and
record the new layout. If `tests/unit.rs` would pass 1,500, split it at
`creates_archive_frontmatter_for_new_archive_note` into `tests/unit.rs` and
`tests/frontmatter.rs`.

Root skeleton, matching `projects/mod.rs`:

```rust
mod archive;
mod git;
mod link_repair;
mod plan;
mod transform;
#[cfg(test)]
mod tests;

use archive::*;
use git::*;
use link_repair::*;
use plan::*;
use transform::*;

pub(crate) use archive::atomic_write;
pub(crate) use transform::{
    block_ids_in_markdown, is_block_id_byte, split_line_ending,
    trailing_block_id_in_line,
};
```

`run`, `run_collection`, and `DEFAULT_THRESHOLD` stay defined in `mod.rs` and keep
`pub(crate)`.

## Visibility and stable paths

`COMMAND_NAME` becomes `pub(super)` in `mod.rs`. `plan.rs` and `git.rs` already
interpolate it, and external callers do not use the constant.

An item that was private becomes `pub(super)` only when the parent or a sibling names
it. The compiler is the checklist: `cargo check --all-targets` (and the test compile)
will name anything still private. Promote those names and nothing else. Leave an item
private when its only callers moved with it.

These `pub(crate)` items are the whole external surface. Callers must keep compiling
with no edits:

| Item                        | Defined in     | Callers that stay unchanged                                                                           |
| --------------------------- | -------------- | ----------------------------------------------------------------------------------------------------- |
| `DEFAULT_THRESHOLD`         | `mod.rs`       | `src/native/nightly.rs`                                                                               |
| `run`                       | `mod.rs`       | `src/native.rs`                                                                                       |
| `run_collection`            | `mod.rs`       | `src/native/nightly.rs`                                                                               |
| `atomic_write`              | `archive.rs`   | `capture_pomodoro_name.rs`, `capture_task_id.rs`                                                      |
| `block_ids_in_markdown`     | `transform.rs` | `capture/pomodoro_adjust.rs`, `capture_task_id.rs`                                                    |
| `trailing_block_id_in_line` | `transform.rs` | `note_tasks.rs` (and the comment in `gkeep/render.rs`)                                                |
| `is_block_id_byte`          | `transform.rs` | `task_status_hooks/{parse,references,pomodoro}.rs`, `randomize_plan.rs`, `capture_language/tokens.rs` |
| `split_line_ending`         | `transform.rs` | `capture_pomodoro_name.rs`, `capture_task_id.rs`                                                      |

Each moved `pub(crate)` function keeps `pub(crate)` on its definition and is re-exported
from `mod.rs` as shown above. `capture_language/tokens.rs` reaches `collect_done`
through `crate::native::collect_done`; that path keeps working through the re-export.

The current test module imports these parent names, and one test also calls
`super::apply_file_plan` and `super::apply_link_repair_plan`: `archive_contents`,
`archive_relative_path`, `archive_wiki_link`, `block_ids_in_markdown`,
`build_collection_plan`, `dependency_id`, `ensure_source_done_tasks_frontmatter`,
`parse_args`, `repair_dependency_metadata`, `repair_links_in_note`, `source_wiki_link`,
`transform_markdown`, `Args`, `MovedBlockTarget`, `MovedBlockTargets`, `NoteIndex`,
`ParseResult`, `DEFAULT_THRESHOLD`, `apply_file_plan`, `apply_link_repair_plan`. Each of
those that moves out of `mod.rs` must be `pub(super)` so `use child::*` puts it in the
parent and `use super::*;` in the tests can see it. `Args`, `ParseResult`, `parse_args`,
and `DEFAULT_THRESHOLD` stay in `mod.rs`.

Imports follow the same rule as `projects`: the root imports the `std` and `super::`
names its own code uses (`bob_env`, `ob`, `ChildEnv`, `Output`), and each child starts
with `use super::*;`. Add a file-local `use` for anything only that file needs, so the
root does not gain an unused import. In particular, `link_repair.rs` imports
`regex::Regex`, `std::sync::LazyLock`, and `super::super::markdown`. `transform.rs`
imports `super::super::is_always_excluded_note_directory_name`. `git.rs` imports
`Command` and `Stdio` if the root no longer uses them.

## Path fixes the move requires

`repair_dependency_metadata` currently calls `super::markdown::fenced_lines`. Inside
`link_repair.rs`, `super` is `collect_done`, so that call becomes
`markdown::fenced_lines` after `use super::super::markdown;`.

In `task_moves_repair_dependency_ids_in_archive_and_all_dependents`, replace
`super::apply_file_plan` and `super::apply_link_repair_plan` with the bare names. The
test file's `super` is the `tests` module; the functions are visible through
`use super::*;`.

`include_str!` paths are relative to the file that contains the macro.
`extracts_nested_blocks_and_continuations` (it moves into `tests/unit.rs`) currently
uses `../../tests/fixtures/collect_done/...` from `src/native/`. From
`src/native/collect_done/tests/unit.rs` the three fixtures need four parents:

```rust
include_str!("../../../../tests/fixtures/collect_done/nested_blocks.md")
include_str!("../../../../tests/fixtures/collect_done/nested_blocks_source.md")
include_str!("../../../../tests/fixtures/collect_done/nested_blocks_archive.md")
```

If that test ends up in a different file, recompute the depth from that file's directory
to the crate root. The fixture bytes stay the same.

A grep for `collect_done.rs` found no references. After the move, grep `collect_done.rs`
again across `src`, `tests`, and `docs` and update a comment only when it still names
the old file.

## Tests

`tests/mod.rs` follows `projects/tests/mod.rs`: `use super::*;`, the shared fixtures,
then `mod unit;` and `mod plan;`. Each child test file starts with a one-line `//!` and
`use super::*;`. Keep the original order inside each child. The 67 test function names
stay exactly as they are. `cargo test -- --list` paths will gain a `unit::` or `plan::`
segment; the count of listed tests must stay the same.

Record the baseline before the first edit:

```bash
wc -l src/native/collect_done.rs
rg -c '#\[test\]' src/native/collect_done.rs
cargo test -- --list 2>/dev/null | grep -c ': test$'
```

The `#[test]` count is 67. The `cargo test -- --list` number is the package-wide count
the epic requires to match at the end.

## Verification

1. `cargo fmt`
2. `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   `cargo test`), with no new compiler or clippy warnings. If clippy reports `dead_code`
   on an item whose only callers are tests, mark that item `#[cfg(test)]` instead of
   widening it or allowing the lint.
3. Package-wide `cargo test -- --list 2>/dev/null | grep -c ': test$'` equals the
   baseline. `rg -c '#\[test\]' src/native/collect_done` still totals 67.
4. `find src/native/collect_done -name '*.rs' -print0 | xargs -0 wc -l`. Every file this
   phase created is at most 1,500 lines.
5. State the final layout in the close note: each file, its responsibility, and its line
   count.

A `just all` or `just check` failure that reproduces identically on the clean base tree
does not keep the phase open. Confirm that by running the same command on the unmodified
tree, then record it on the phase bead and close anyway:

```bash
sase bead note bob-cli-2f.8 'PROPOSED FOLLOW-UP: <one-line summary — detail>'
```

Cite any task bead that already tracks that failure. Record any other discovered
follow-up the same way. Create no beads.

## Close only this phase

Before closing, run:

```bash
sase bead epic-symbols bob-cli-2f.8
```

At plan time that command reported no `--epic-symbol` entries for `bob-cli-2f.8`. If any
exist when the work is done, resolve each symbol or re-key that Justfile line to a
still-open bead (the parent epic `bob-cli-2f` or a later phase). `sase bead close`
refuses while leftovers remain.

Then close only `bob-cli-2f.8`:

```bash
sase bead close bob-cli-2f.8 --note "<what you verified: layout, line counts, test count, just all>"
```

Leave parent epic `bob-cli-2f` and every ancestor plan bead open. The bead is already
`in_progress`; do not set that status by hand.
