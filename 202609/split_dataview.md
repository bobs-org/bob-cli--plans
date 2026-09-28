---
tier: tale
title: Split the Dataview query module
goal:
  Move the Dataview query implementation into cohesive modules of at most 1500 lines
  without changing behavior or tests.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-2f.5
bead: bob-cli-2f.5
status: done
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.5.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2f.5](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2f.5.md)
- **COMMITS:**
  - [2307179](https://github.com/bobs-org/bob-cli/commit/2307179cd17439fc6bb2a14eecbc842189ab0199)
    — refactor(dataview): split query module into cohesive submodules

# Split the Dataview query module

## Goal and scope

Complete phase `bob-cli-2f.5` of the existing Rust-file-splitting epic. Split the
7,088-line `src/native/dataview.rs` into cohesive files under the existing
`src/native/dataview/` directory. Keep `dataview.rs` as the root, preserve CLI, native
query, and Obsidian behavior, and keep every newly created or touched Rust file at 1,500
lines or less. Do not split the existing `index.rs`, `value.rs`, or `tasks/` files;
later epic phases own other large top-level files.

The parent epic requires a structural refactor only: preserve implementation, test
bodies, diagnostics, and external module paths. The root must retain
`pub(crate) fn run`, and `crate::native::dataview::DataviewError` must remain available
to `tasks/filter.rs`, `tasks/index.rs`, and `tasks/result.rs`.

## Implementation

1. Record the current `cargo test -- --list` test count and the test names for
   `native::dataview`. Reconfirm source landmarks before moving anything. Preserve the
   root's existing nine unit tests, possibly in a separate `dataview/tests.rs` if the
   root benefits from it.
2. Make `dataview.rs` a small coordinator with the existing `run`, request dispatch,
   module declarations, and the narrow imports/re-exports needed by its children and
   tests. Move the JavaScript literal and Obsidian process, request, and protocol code
   to `dataview/obsidian.rs`. Move CLI builders, request/input/format/engine/vault types
   and argument validation to `dataview/cli.rs`; keep the existing CLI behavior and help
   text byte-for-byte.
3. Split the native query model and vault evaluation into `native.rs` and `vault.rs` (or
   equivalent files below 1,500 lines); put expression evaluation and call dispatch in
   `eval.rs`. Split the built-in function range into a `functions/` directory by
   coherent families (link/numeric, collection, string/regex, date/duration/currency,
   and comparison/arithmetic), cutting boundaries as needed so no file approaches the
   limit. Move `NativeLexer` and `NativeParser` into `lexer.rs` and `parser.rs`; move
   source matching, markdown path collection, and result path extraction into
   `sources.rs`. Move formatting, warnings, JSON, and markdown output to `output.rs` and
   `render.rs` when the final grouping warrants both. Move `DataviewError` and error
   excerpts into `error.rs`. Give each new submodule a short `//!` summary.
4. Move code verbatim and keep item order within each moved group. Change only module
   declarations, imports, visibility needed between sibling modules, and `super::`
   paths. Use `pub(super)` for sibling-only items and retain the old root path for items
   used by `tasks/`; do not widen other APIs. Keep `index.rs`, `value.rs`, and `tasks/`
   content intact unless a minimal import correction is required. Search for
   `dataview.rs` references in `src`, `tests`, and `docs`, and correct any stale
   file-path descriptions.
5. Run `cargo fmt` and compile/check iteratively to settle cross-module imports without
   changing query logic. Compare the full test list and count with the baseline, run the
   Dataview unit and CLI tests, and run `just all` (`fmt`, `clippy`, and `cargo test`).
   Check for new warnings and confirm each newly created or touched Rust file is at most
   1,500 lines. Report the final file layout, responsibility, and line count.
6. Before closing `bob-cli-2f.5`, run `sase bead epic-symbols bob-cli-2f.5`. Resolve any
   symbols for this phase or re-key their Justfile lines to an open parent/later bead.
   If `just all` fails identically on the clean base, record a `PROPOSED FOLLOW-UP:`
   note on this phase, then close only this phase with
   `sase bead close bob-cli-2f.5 --note "<verified result>"`. Record any other
   out-of-scope findings with the same note prefix rather than creating beads. Do not
   close the parent epic or ancestor plan.

## Acceptance

- `dataview.rs` is a lean coordinator; its original code is distributed by
  responsibility into files under `src/native/dataview/`.
- Every Rust file created or touched for this phase is at most 1,500 lines.
- The test list/count and assertions are unchanged; `just all` passes or an identical
  clean-base failure is documented as prescribed above.
- No new compiler or clippy warnings; the public CLI and old module paths work.
- Only the assigned phase bead is closed, after its epic symbols are cleared.
