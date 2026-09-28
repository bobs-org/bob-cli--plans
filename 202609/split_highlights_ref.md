---
tier: tale
title: Split highlights_ref into modules of at most 1500 lines
goal: "Thin src/native/highlights_ref/mod.rs into cohesive submodules and split its unit
  tests so every Rust file in that directory has at most 1500 lines, behavior is
  unchanged, the 59 unit tests in that file still pass, and just all is green.

  "
size: medium
proposed_by: bbugyi200.apollo.bob-cli-2f.4
bead: bob-cli-2f.4
create_time: 2026-09-28 18:19:32
status: wip
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.4.md)

# Split highlights_ref into modules of at most 1500 lines

## Goal

Implement phase bead `bob-cli-2f.4` (`split-highlights-ref`) from epic
`plan:202609/split_largest_rust_files.md`. Move code out of
`src/native/highlights_ref/mod.rs` until every Rust file under
`src/native/highlights_ref/` is at most 1500 lines, aiming for roughly 300–1200. This is
a pure structural refactor of that one root file. Behavior, assertions, and the test
count stay the same.

Measured on this tree before any edit: `mod.rs` is 10240 lines (code through line 7682,
`#[cfg(test)] mod tests` at 7683–10240) and `create.rs` is 1369 lines. Those match the
epic's `d0c1692` landmarks, so the line numbers below are current. `mod.rs` contains 59
`#[test]` functions. `create.rs` contains 21 and stays untouched. The only outside
caller is `src/native.rs`, which calls `highlights_ref::run`. There are no `//!` module
docs, no `include_str!` / `include_bytes!`, and no docs that name
`highlights_ref/mod.rs`.

## Decisions

The epic's outline is the starting point. These cuts differ where the current file's
cohesion or Rust privacy rules require it:

- Keep `create.rs` byte-for-byte. Do not edit it. Its `use super::{...}` list must keep
  resolving. That list is `atomic_save_pdf`, `bob_dir_arg`, `dry_run_arg`,
  `lib_dir_arg`, `parse_marker_with_normalization`, `pdf_text_string`, `ref_dir_arg`,
  `render_marker`, `validate_marker_parent_value`, `validate_required_marker_keys`,
  `xlib_dir_arg`, `CommandError`, `Config`, `MarkerValue`, `Projection`, `Result`,
  `FIELD_ID`, `FIELD_PARENT`, and `FIELD_STATUS`. `create.rs` also reads
  `config.xlib_dir` and `config.lib_dir`.
- Put shared types and their inherent impls in `model.rs`, including the impls the epic
  sketched into `note.rs` (`Config`, `Prefer`, `SyncSource`, `MarkerValue`,
  `ParsedNote`). A private field is visible only in the module that defines the struct.
  Sibling files, including `create.rs`, already read those fields, so the fields they
  read become `pub(super)` either way. Colocating the impls keeps `model.rs` near 900
  lines and leaves `note.rs` as the free-function note pipeline (~620 lines).
- Merge the epic's `doctor.rs` / `intake.rs` pair into one `doctor.rs` (~424 lines).
  Each half is only about 200 lines and they share layout and path checks.
- Split sidecar parse/assets from render/classifiers. Together they are ~1087 lines, at
  the top of the 1200-line aim. `sidecar.rs` is lines 3164–3771 (~608).
  `sidecar_render.rs` is lines 3772–4250 (~479).
- Keep `frontmatter.rs` and `io.rs` as two files, matching the epic. They are ~207 and
  ~243 lines.
- Leave constants and command dispatch in `mod.rs`. The directory already exists, so
  there is no `git mv`.

## Target layout

Move each range verbatim, in order, including structs, enums, and impls that sit inside
the range. Line numbers are inclusive landmarks in today's `mod.rs`. Re-measure with
`wc -l` after `cargo fmt`.

| File                  | Source lines                           | Responsibility                                                                                                                                                                                                               |
| --------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mod.rs`              | 32–120 and 577–676, plus `mod` / `use` | Constants (`COMMAND_NAME` through `COMMON_USER_FIELDS`), `run`, subcommand dispatch (`no_hooks_flag` through `report_result`), `mod create;`, child modules, and `use child::*;` globs. No new essay-length `//!` doc.       |
| `model.rs`            | 120–576, plus impls 5653–6054          | Shared types from `Config` through `GitStatusEntry`, the small impls already beside them, then `impl Config`, `impl Prefer`, `impl SyncSource`, `impl MarkerValue`, and `impl ParsedNote` in that order.                     |
| `hooks.rs`            | 677–844                                | Pre-scan hook configuration, running, and checking.                                                                                                                                                                          |
| `sync.rs`             | 845–1536                               | `sync_pdf`, `scan_library`, `plan_pdfs`, `plan_pdf_sync`, annotation-task plan finalization, `execute_pdf_sync`, image writes, and unchanged-for-write guards. Includes `RoutedTaskGroup` and the `ProcessedTaskIndex` impl. |
| `report.rs`           | 1537–2385                              | PDF sync, scan plan, and write reports, `ScanLine`, `ScanCounts`, and summaries.                                                                                                                                             |
| `doctor.rs`           | 2386–2809                              | `show_marker`, `doctor_vault`, library layout validation, xlib intake, and PDF path collection.                                                                                                                              |
| `guard.rs`            | 2810–3163                              | Output and asset collision checks, `ensure_safe_to_write`, dirty-note allowances, and git status helpers.                                                                                                                    |
| `sidecar.rs`          | 3164–3771                              | Wikilink and parent canonicalization, sidecar discovery and parsing, and image asset resolution.                                                                                                                             |
| `sidecar_render.rs`   | 3772–4250                              | `render_sidecar_highlights`, callouts, and the line classifiers and strippers. Includes the `SidecarAnnotationKind` impl.                                                                                                    |
| `text.rs`             | 4251–4431                              | PDF text artifact cleanup, reflow, and `normalized_identity_text`.                                                                                                                                                           |
| `annotation_tasks.rs` | 4432–5231                              | Candidates, routes, identities, the processed-task index, PDF task lines, and status signals. Includes `PdfTaskStatusConflict`.                                                                                              |
| `projection.rs`       | 5232–5461                              | `resolve_sync_projection`, merge, and diagnostics. Includes `ProjectionMerge`.                                                                                                                                               |
| `cli.rs`              | 5462–5652                              | `print_config_report`, `build_cli`, and the arg builders.                                                                                                                                                                    |
| `note.rs`             | 6055–6674                              | Pipeline metadata, default note body, managed region, tasks-section insertion, timestamps, and config path helpers (`configured_path` through `prefer_from_matches`).                                                        |
| `marker.rs`           | 6675–7232                              | PDF marker read, decode, write, parse, validate, render, and inline value parsing.                                                                                                                                           |
| `frontmatter.rs`      | 7233–7439                              | Note read and parse, frontmatter entries, field predicates, and projection snapshot JSON. Ends just before `ref_note_path`.                                                                                                  |
| `io.rs`               | 7440–7682                              | Path metadata, sha256, atomic write and copy, string rendering, and `change_action`.                                                                                                                                         |

`#[cfg(test)]` helpers stay with the range that contains them, still gated, and become
`pub(super)` because another module calls them:

- `changes_confined_to_frontmatter_or_pdf_task_checkbox` (3011) stays in `guard.rs`.
  Tests call it.
- `sidecar_page_heading` (4024) stays in `sidecar_render.rs`. Tests call it.
- `existing_annotation_task_identities` and `existing_annotation_task_identity`
  (4775, 4782) stay in `annotation_tasks.rs`. `insert_missing_annotation_tasks` calls
  the first.
- `insert_missing_annotation_tasks` (6226) stays in `note.rs`. Tests call it.

## Tests

Split `mod tests` into `src/native/highlights_ref/tests/`, declared from `mod.rs` as
`#[cfg(test)] mod tests;`. Follow `src/native/capture/tests/`: `tests/mod.rs` holds
`use super::*;`, the shared fixtures now at lines 7684–7747 (`string_value`,
`test_projection`, `test_config`, `test_config_for_bob_dir`, `temp_bob_dir`,
`write_test_file`), and the child `mod` lines. Replace the explicit `use super::{...}`
import inside today's test module with that glob. `tests/mod.rs` may be under 80 lines;
it is only the shared fixture root, same as `capture/tests/mod.rs`.

Each child file starts with `use super::*;` and a one-line `//!` summary. Keep every
`super::item` call and every assertion verbatim. Nested `super::item` still resolves
because `tests/mod.rs` imports the parent namespace. Helpers used by only one group move
with that group: `annotation_task_ref_body` and `insert_annotation_tasks` (9023–9048)
stay in `tasks.rs`; `signal_resolution` (9255) stays in `status.rs`.

| Test file             | Source lines              | Coverage                                                                                                                                   |
| --------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `tests/projection.rs` | 7748–8095, then 9636–9712 | Config paths, xlib intake, library layout, pipeline fields, snapshot JSON, deprecated statuses, three-way merge, then `pdf_path_metadata`. |
| `tests/tasks.rs`      | 8096–8292, then 8715–9209 | PDF task line parser, annotation-task candidates, route suffix, processed index, and tasks-section insertion.                              |
| `tests/status.rs`     | 9210–9635                 | PDF task status signals and the narrow checkbox / dirty-note allowance test.                                                               |
| `tests/sidecar.rs`    | 8293–8714, then 9713–9894 | Sidecar parse and render, image assets, text cleanup, block ids, page headings, and quote/comment classification.                          |
| `tests/marker.rs`     | 9895–10240                | Marker parse and render, parent canonicalization, and frontmatter projection.                                                              |

Preserve source order inside each moved group. The second range in `projection.rs`,
`tasks.rs`, and `sidecar.rs` is appended after the first. That is 59 tests. Do not add,
drop, or edit an assertion. Test paths gain a module segment; the `#[test]` count does
not change.

## Move rules

- Move code verbatim. Keep doc comments, `//` comments, and the relative order of items
  inside each moved group. Do not rename items or clean up logic. `cargo fmt` (the repo
  pins `max_width = 80`) is the only formatting pass.
- Change only `mod` / `use` declarations, visibility, and paths the move requires.
- Give every new submodule a one-line `//!` summary. Match the repo's sparse comment
  style.
- In `mod.rs`, declare the children and glob-import them (`use model::*;`, and the same
  for every sibling, including `io` and `sidecar_render`). Keep `mod create;` first.
  `pub(crate) fn run` stays in `mod.rs`.
- Start each sibling with the external and std imports its moved code uses, then
  `use super::*;`, matching `src/native/capture/*.rs`. `mod.rs` keeps only imports that
  the dispatch code still references. Clippy must not report new `unused_imports` or
  `dead_code` warnings. If a `use super::*;` / `use child::*;` pair makes rustc report
  an import cycle, switch that file to explicit `use super::module::Name` imports.
- Visibility: `run` stays `pub(crate)`. Everything else that another file in this
  directory reads, calls, or constructs becomes `pub(super)` and nothing wider. That
  includes struct fields sibling modules already read (for example `Config`'s directory
  fields, which `create.rs` reads). Items used only inside their new file stay private.
  Methods used only inside the impl's file stay private. `pub(super)` on a field that
  moved out of `mod.rs` restores the visibility that field had as a private field of the
  parent module: visible throughout `highlights_ref`, not outside it.
- Do not edit `create.rs`, `src/native.rs`, integration tests, or any other oversized
  Rust file. Later phases own those. No outside import path changes are required if the
  names `create.rs` imports remain in the parent module through the globs and the
  constants left in `mod.rs`.

## Verification

1. Before editing, record `rg -c '#\[test\]' src/native/highlights_ref`. Expect
   `mod.rs:59` and `create.rs:21`.
2. After the move, the directory still has 80 `#[test]` attributes, with `create.rs`
   still at 21 and the new test files totaling 59.
3. `cargo fmt`, then `cargo fmt --check`.
4. `find src/native/highlights_ref -name '*.rs' -print0 | xargs -0 wc -l`. Every file in
   that directory is at most 1500 lines. If `cargo fmt` pushes a file over 1500, split
   it again on a cohesion boundary and keep the same move rules. If a file is over about
   1200 and has a clean seam, split it the same way. Do not add files under about 80
   lines except `tests/mod.rs`.
5. `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   `cargo test`) passes with no new compiler or clippy warnings.
6. State the final layout in the close note: each file, its responsibility, and its
   post-format line count, plus the before/after `#[test]` counts.

A `just all` failure that reproduces identically on the clean base tree does not keep
the phase open. Record it on this phase with
`sase bead note bob-cli-2f.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'` and
close anyway. Do not create beads.

## Close this phase only

After verification, run `sase bead epic-symbols bob-cli-2f.4`. If any `--epic-symbol`
entries remain, resolve each symbol or re-key the Justfile line to a still-open bead
(the parent epic `bob-cli-2f` or a later phase). Then close only this phase:

```bash
sase bead close bob-cli-2f.4 --note "<what you verified: layout, line counts, test counts, just all>"
```

Do not close `bob-cli-2f` or any ancestor. Do not set the phase status by hand before
that close. Closing an assigned phase is unaffected by the parent-close descendant
guard.
