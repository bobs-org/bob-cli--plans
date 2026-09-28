---
tier: tale
size: medium
title: Split the capture language grammar into focused modules
goal:
  Preserve the capture grammar's behavior and public module paths while splitting its
  11,615-line source and 163 unit tests into Rust files of at most 1,500 lines each.
proposed_by: bbugyi200.apollo.bob-cli-2f.3
bead: bob-cli-2f.3
create_time: 2026-09-28 17:46:53
status: wip
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.3.md)

# Split `src/native/capture_language.rs`

## Scope and baseline

The assigned phase is `bob-cli-2f.3` of the existing file-splitting epic. The current
file has 11,615 lines: grammar and editor implementation through line 7,034, then 163
unit tests. This is a pure structural refactor of that file. Keep the existing root path
`native::capture_language` for its callers, preserve error text and serialized shapes,
and do not change assertions or behavior. Leave the other oversized files for their own
phases.

## Implementation

1. Record the baseline test count and inspect all top-level items, call sites, test
   helpers, and `#[cfg(test)]` boundaries. Move the file with `git mv` to
   `src/native/capture_language/mod.rs` before editing. Preserve the existing root `//!`
   documentation and add concise module summaries.
2. Move model types and shared token interfaces into `model.rs`; draft line splitting,
   authored line classification, global declarations, and inheritance into `draft.rs`;
   item and session-operator parsing into `item.rs`; line resolution and error
   constructors into `line.rs`; route/caret/selector parsers into `tokens.rs`; and
   terminal-marker extraction, schedules, priorities, clips, and marker messages into
   `markers.rs`. Keep item order and logic intact. Use `pub(super)` for sibling-only
   items and root re-exports for existing `pub(crate)` caller paths.
3. Split the editor implementation by cohesion: editor model/span/diagnostic types; core
   line, global, and item parsing; special Pomodoro item parsing; token classification
   and diagnostics. Keep each editor source below 1,500 lines and retain the single
   grammar shared with execution. Move cursor completion to `completion.rs` and
   absorption/text edits to `rewrite.rs`.
4. Split the test module into focused files for editor spans/diagnostics, grammar and
   markers, completion, draft/authored children, global declarations, and rewrite. Put
   shared test helpers in a test root. Preserve each original test and assertion
   verbatim, with only import/path changes needed by the module move. Keep every
   resulting source and test file at most 1,500 lines, aiming for headroom.
5. Resolve visibility and import issues without widening external access beyond what
   existed. Check external references in `capture`, `capture_parse`, `capture_complete`,
   `capture_rewrite`, and other callers; preserve their `capture_language::...` paths.
   Update only misleading references to the old file layout if necessary.

## Verification and completion

Run `cargo fmt`, targeted capture-language tests, and `just all` (format, Clippy, and
full test suite). Compare the final unit-test count to the baseline 163 and verify that
the full test count has not fallen. Check line counts for every resulting file and
inspect the diff for accidental logic or assertion edits and new warnings. Before
closing the assigned phase, run `sase bead epic-symbols bob-cli-2f.3` and resolve or
re-key any remaining symbols to an open bead. Close only `bob-cli-2f.3` with a note
listing the verification performed; any unrelated clean-base failure belongs in a
`PROPOSED FOLLOW-UP:` note on that phase.
