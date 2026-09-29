---
tier: tale
title: Split capture_pomodoro_close into a directory module
goal: The Pomodoro close planner keeps its behavior, tests, and external paths, and
  every file in the new directory module is at most 1500 lines.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-2f.10
bead: bob-cli-2f.10
status: done
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.10](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.10.md)

# Plan: Split capture_pomodoro_close into a directory module

Implement phase `bob-cli-2f.10` only. This is a pure structural split of
`src/native/capture_pomodoro_close.rs` (measured at 2982 lines on this tree; the phase
does not already meet the limit). Do not change behavior, rename items, edit assertions,
or split any other file.

## Outcome

`git mv` the file to `src/native/capture_pomodoro_close/mod.rs`, then move its items, in
their current relative order, into these files:

| File                   | Responsibility                                                                                              | Approx. lines                 |
| ---------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------- |
| `mod.rs`               | Original `//!` docs, `#![allow(dead_code)]`, `close_task_text`, `mod` declarations, `pub(crate)` re-exports | facade, may be under 80 lines |
| `ledger.rs`            | Running-Pomodoro discovery, close timing, ledger link classification, Work Log note grouping                | ~870                          |
| `links.rs`             | Pomodoro-marker rewrite, wikilink tokens, strikethrough spans, bare link recognition                        | ~340                          |
| `linked_tasks.rs`      | `CloseVault`, `ClosePlanner`, `plan_pomodoro_close`, task lookup and completion-field edits                 | ~970                          |
| `tests.rs`             | Current `mod tests` body (18 `#[test]` fns)                                                                 | ~470                          |
| `linked_task_tests.rs` | Current `mod linked_task_tests` body (7 `#[test]` fns)                                                      | ~350                          |

Hard cap is 1500 lines per file, tests included. These cuts sit in the 300–1200 band
except the facade. Do not merge the parsing module back into the ledger: the phase asks
for link and marker parsing as its own file, and putting `link_from_token` /
`target_from_token` in `links.rs` would make `links` depend on ledger types while
`ledger` depends on `links`.

## What stays externally reachable

`src/native.rs` already has `mod capture_pomodoro_close;`. A directory `mod.rs`
satisfies that. Callers must keep compiling with no edits:

- `src/native/capture_pomodoro_start.rs` imports `bare_embedded_link`,
  `bare_plain_link`, `close_task_text`, `lookup_task`, `range_is_struck`,
  `strikethrough_inner_spans`, `strip_pomodoro_markers`, `sub_bullet_range`,
  `wikilink_tokens`, and `CloseVault`.
- `src/native/capture/pomodoro_close.rs` uses `CloseVault`, `find_running_pomodoro`,
  `PomodoroClosePlanError`, `FindRunningError`, `PomodoroClosePlan`, `LedgerLinkRole`,
  `plan_pomodoro_close`, and `sub_bullet_range`.

Re-export every item that is `pub(crate)` today, from the parent module, so those paths
stay `capture_pomodoro_close::Item`. Do not widen any private item to `pub(crate)`.
Sibling-only items become `pub(super)`.

`pub(crate)` re-exports:

- From `ledger`: `RunningPomodoro`, `NamedPomodoro`, `FindRunningError`, `CloseTiming`,
  `LedgerLinkRole`, `ClassifiedLink`, `BlockLinkTarget`, `NextPomodoro`, `WorkLogNode`,
  `WorkLogNoteGroup`, `LedgerClosePlan`, `find_running_pomodoro`, `close_timing`,
  `sub_bullet_range`, `plan_ledger_close`.
- From `links`: `strip_pomodoro_markers`, `WikiToken`, `wikilink_tokens`,
  `strikethrough_inner_spans`, `range_is_struck`, `exact_struck`, `bare_plain_link`,
  `bare_embedded_link`.
- From `linked_tasks`: `CloseTaskRole`, `PomodoroCloseTask`, `PomodoroCloseSummary`,
  `PomodoroClosePlan`, `PomodoroClosePlanError`, `CloseVault`, `plan_pomodoro_close`,
  `lookup_task`.

`close_task_text` stays defined on `mod.rs`, so it needs no re-export.

## Move map

Keep each item's body, doc comments, and `//` comments verbatim. Preserve source order
inside each destination. Change only `mod` / `use`, visibility, and `super::` paths
required by the move.

`links.rs` starts at `enum MarkerPolicy` and runs through `bare_embedded_link` (today
about lines 708–1031). Move `const POMODORO_MARKER` with it; only the marker rewriter
uses that constant. Give the new file a one-line `//!` summary. Its imports are
`std::cmp::Reverse`, `std::ops::Range`,
`super::capture::{leading_spaces_or_tabs_len, list_marker_len}`, and
`super::capture_language`.

Promote these `links` items to `pub(super)`, and no further:

- `rewrite_completed_markers` (called by `plan_ledger_close`)
- `move_only_destination` (called by `classify_sub_bullets`)
- `pomodoro_marker_prefix` and `apply_edits` (called by `retire_embedded_links`)
- `struct MarkerPrefix` and its `start` field (the retire helper reads `prefix.start`).
  Leave `count` and `canonical` private.

`ledger.rs` gets the other three constants (`PLACEHOLDER_LINE`, `EMPTY_SUB_BULLET`,
`HALF_DAY_MINUTES`, `DAY_MINUTES`), every type from `RunningPomodoro` through
`LedgerClosePlan`, and every function from `find_running_pomodoro` through
`target_from_token`, then the Work Log group from `work_log_task_link_target` through
`collect_work_log_note_groups`. `link_from_token` and `target_from_token` stay here.
One-line `//!` summary. Imports: `BTreeSet`, `Range`,
`chrono::{NaiveDateTime, Timelike}`, the `capture` helpers this code already calls
(`adjustment_duration_for_range`, `format_adjusted_range`, `leading_spaces_or_tabs_len`,
`line_spans`, `list_item_body`, `list_marker_len`, `nearest_shallower_list_item_parent`,
`parse_adjustment_range`, `AdjustRange`, `LineSpan`), plus `capture_pomodoros`,
`markdown`, and `pomodoro`. Import the `links` names it calls: `WikiToken`,
`bare_plain_link`, `move_only_destination`, `range_is_struck`,
`rewrite_completed_markers`, `strikethrough_inner_spans`, `strip_pomodoro_markers`,
`wikilink_tokens`.

`linked_tasks.rs` gets `CloseTaskRole` through `retire_embedded_links` (today about
lines 1683–2628), not the test module. One-line `//!` summary. Imports: `BTreeMap`,
`BTreeSet`, `Path`, `PathBuf`, `NaiveDateTime`, `capture::{line_spans, LineSpan}`,
`capture_language`, `capture_task_toggle`, `capture_work_log`,
`note_tasks::{self, BlockIdLookup, NoteTask, NoteTaskSettings, TaskStatusType}`,
`pomodoro`, and `vault_links::LinkResolution`. Import `super::close_task_text`, the
ledger types and functions it names (`RunningPomodoro`, `LedgerClosePlan`,
`LedgerLinkRole`, `BlockLinkTarget`, `WorkLogNode`, `find_running_pomodoro`,
`plan_ledger_close`, `sub_bullet_range`), and from `links` the names `wikilink_tokens`,
`pomodoro_marker_prefix`, `apply_edits`, and `WikiToken`. Qualify
`capture_work_log::WorkLogNode` exactly as the current `work_log_node` function does, so
it does not collide with the ledger struct.

`mod.rs` keeps this text as the module docs:

```rust
//! Pure Pomodoro-close planner: rewrite the running ledger entry, apply linked
//! task effects, and compose dated Work Log entries without writing files.
```

Keep `#![allow(dead_code)]` on `mod.rs` only. Descendants inherit it. Do not add new
allows. Drop imports that the move leaves unused instead of allowing them.

There is no `include_str!` in this file. No comment or doc in the repo names
`capture_pomodoro_close.rs`, so no doc-path edits are required. The intra-doc link from
`bare_embedded_link` to `bare_plain_link` stays valid because both functions move
together.

## Tests

The file contains 25 `#[test]` functions: 18 in `mod tests`, 7 in
`mod linked_task_tests`. Both modules become direct children of
`capture_pomodoro_close`:

```rust
#[cfg(test)]
mod tests;
#[cfg(test)]
mod linked_task_tests;
```

Keeping that depth preserves `super::super::vault_links::target_to_markdown_path` inside
`MemoryVault`. Do not nest the tests under `ledger/` or `linked_tasks/`.

Replace each test module's `use super::*` with explicit imports. `use super::*` only
sees names in `mod.rs`, not private items of the siblings.

- `tests.rs`: `use chrono::{NaiveDate, NaiveDateTime};` and `use super::ledger::*;`. The
  helper `timing_for` calls `parse_adjustment_range`. If the ledger glob does not bring
  that import into the test, add `use super::super::capture::parse_adjustment_range;`.
- `linked_task_tests.rs`: import `BTreeMap`, `Path`, `PathBuf`,
  `chrono::{NaiveDate, NaiveDateTime}`, `tempfile::TempDir`,
  `super::super::vault_links::LinkResolution`, and `super::linked_tasks::*`. Add
  `super::ledger::*` or `super::links::*` only if a named item does not resolve through
  `linked_tasks`.

Do not change test bodies, expected strings, or assertion logic. After the move,
`rg -N '#\[test\]' src/native/capture_pomodoro_close` must still report 25 tests.

## Verification

1. `cargo fmt`, then `cargo fmt --check`. The repo pins `max_width = 80`.
2. `cargo test --lib capture_pomodoro_close::`
3. `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   `cargo test`). No new compiler or clippy warnings.
4. `find src/native/capture_pomodoro_close -name '*.rs' | xargs wc -l` and confirm every
   file is at most 1500 lines.
5. Confirm `git status` shows no edits outside `src/native/capture_pomodoro_close/`. If
   a caller fails to compile, fix the re-export. Do not update the caller.

## Close only this phase

Do not close parent epic `bob-cli-2f` or any ancestor, and do not create beads. Before
closing, run `sase bead epic-symbols bob-cli-2f.10`. There were no `--epic-symbol`
entries when this plan was written. If any are present, re-key each Justfile line to a
still-open bead (the parent epic or a later phase) before closing. `sase bead close`
refuses while leftovers remain.

If `just all` fails in a way that reproduces identically on the clean base tree, record
it on this bead as `PROPOSED FOLLOW-UP: <one-line summary — detail>` (cite any task bead
that already tracks it) and close anyway. Record any other discovered follow-up the same
way. Do not set the bead status by hand.

Close with:

```bash
sase bead close bob-cli-2f.10 --note "<layout chosen, line counts, test count 25, and what just all showed>"
```
