---
tier: tale
title: Split capture task toggle planners into focused modules
goal: "Split src/native/capture_task_toggle.rs into a small facade and focused child
  modules so every resulting Rust file is at most 1500 lines, while pure-planner
  behavior, the existing capture_task_toggle entry points, and current test coverage
  stay intact.

  "
size: medium
proposed_by: bbugyi200.athena.bob-cli-3s.2
bead: bob-cli-3s.2
status: done
---

- **PARENT:**
  [202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-3s.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/bob-cli-3s.2.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-3s.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-3s.2.md)
- **COMMITS:**
  - [c9a6f1b](https://github.com/bobs-org/bob-cli/commit/c9a6f1b453b12730f1a64b4d2b16a2314e383c1d)
    — refactor(capture): split capture_task_toggle into focused modules

# Plan: Split capture task toggle planners into focused modules

## Why this is a tale

Phase `bob-cli-3s.2` (`split-capture-task-toggle` on epic `bob-cli-3s`) is one assigned
file. The dependency graph below is acyclic and the move is behavior-preserving, so one
coding agent can implement it from this plan. Another epic would only re-split a phase
the parent epic already isolated. `medium` fits a multi-module visibility-sensitive
refactor that is still bounded by one file and the checks below. Do not set an explicit
model.

Measured on this tree, before any edit: `src/native/capture_task_toggle.rs` is 2507
physical lines. The inline `#[cfg(test)] mod tests` starts at line 1684, so the
production body is 1683 lines (over the 1500 ceiling) and the tests are 824 lines (under
it). Phase 1 already split `capture_complete` into `capture_complete.rs` plus
`src/native/capture_complete/`. Use that same shape. Do not touch `capture_complete`,
and do not start the later phases (`task_status_hooks_write`, `capture_clip`,
`plugins`).

## Layout

Keep `src/native.rs`'s `mod capture_task_toggle;`. Replace the body of
`src/native/capture_task_toggle.rs` with a facade and put children in
`src/native/capture_task_toggle/`. Do not also add `capture_task_toggle/mod.rs`.

Move code. Do not rewrite planner logic, merge the two task planners, drop comments, or
reformat beyond `cargo fmt`. Adjust `use` paths and visibility keywords only. Import
helper names into each child so call expressions stay the same.

Dependency direction, which must not cycle:

`text` ← `task_update`

`text` ← `links` ← `relocation` ← `ledger`

`ledger` also calls `links` and `text`. `task_update` does not call `links`,
`relocation`, or `ledger`. No shared model module is needed; nothing cycles.

| File                     | Owns                                                                                                               | Approx. lines |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ | ------------- |
| `capture_task_toggle.rs` | Module docs, `#![allow(dead_code)]`, `mod` declarations, explicit re-exports                                       | under 120     |
| `text.rs`                | Mechanical line, subtree, newline, and indentation edits                                                           | about 320     |
| `task_update.rs`         | Status mutation, future `scheduled` retirement, pull-forward Schedule Log text, `plan_task_next`, `plan_task_link` | about 350     |
| `links.rs`               | Insertion and removal planners, implicit open-entry selection, movable-link discovery                              | about 410     |
| `relocation.rs`          | Relocation planner, destination resolution, subtree moves, named-placeholder policy                                | about 420     |
| `ledger.rs`              | `plan_pomodoro_link_ledger` and `list_open_entry_links`                                                            | about 345     |
| `tests.rs`               | The existing inline tests, unchanged assertions                                                                    | about 830     |

`insert_named_placeholder` stays in `relocation.rs`. It returns `LinkRelocationError`
and applies named-entry placement policy (multiple open timed entries,
completed-versus-open anchors). `text.rs` keeps only the mechanical `insert_line_before`
/ `insert_line_after` primitives it calls. That is the dependency-driven departure from
the epic's "placeholder might live in text" suggestion.

Keep one `tests.rs`. The 824-line suite shares `date`, `planned`, `relocated`, and
`relocated_named`, and it sits well under 1500. Splitting it would publish those helpers
for no coverage gain.

### `text.rs`

Move these, and nothing that scans Pomodoros or plans a task:

- `insert_line_before`, `extract_line_range`, `remove_line_range`, `reindent_subtree`,
  `logical_lines`, `physical_line_pieces`, `insert_lines_after`
- `child_block_end_line`, `line_start_offset`, `line_index_at_offset`, `replace_line`,
  `line_has_crlf`, `insert_line_after`
- `unordered_child_indentation`, `pomodoro_child_indentation`

`pomodoro_child_indentation` already receives section bounds. Do not make `text` import
`capture_pomodoros`.

### `task_update.rs`

Move `PULL_FORWARD_ENTRY_EMPHASIS`, `PULL_FORWARD_REASON`, `pull_forward_entry_text`,
`TaskTogglePlan`, `plan_task_next`, `plan_task_link`, `current_status_symbol`,
`set_task_line_status`, `ScheduledFieldMatch`, `scheduled_field_matches`,
`find_single_future_scheduled_field`, and `remove_span_with_space_collapse`.

Leave the two planners intact. `plan_task_link` stamps freshness when the line changes;
`plan_task_next` does not. A shared helper would be a behavior edit.

### `links.rs`

Move `LinkPlanError`, `LinkInsertionOutcome`, `LinkInsertionPlan`, `LinkPlacement`,
`plan_link_insertion`, `LinkRemovalPlan`, `plan_link_removal`,
`select_implicit_open_entry`, `MovableLink`, `find_movable_task_links`,
`insert_link_into_entry`, and `remove_matching_links`.

### `relocation.rs`

Move `LinkRelocationError`, `LinkRelocationAction`, `PomodoroEndpoint`,
`LinkRelocationPlan`, `plan_link_relocation`, `ResolvedRelocationDestination`,
`resolve_relocation_destination`, `destination_endpoint_after_move`,
`move_subtree_to_entry`, `insert_named_placeholder`, `relocation_from_link_plan_error`,
and `endpoint_from_entry`.

### `ledger.rs`

Move `PomodoroLinkLedgerAction`, `PomodoroLinkLedgerPlan`, `plan_pomodoro_link_ledger`,
`OpenEntryLinks`, and `list_open_entry_links`.

## Visibility

Keep the facade's module docs and `#![allow(dead_code)]` so today's dead-code allowance
still covers the tree. Child modules stay private (`mod`, not `pub mod`), matching
`capture_complete.rs`.

Re-export every existing `pub(crate)` item from the facade with explicit
`pub(crate) use` paths, not globs. Callers must keep compiling with no edits. If a path
breaks, add the missing re-export; do not edit the caller.

Re-export at least:

- From `task_update`: `PULL_FORWARD_REASON`, `pull_forward_entry_text`,
  `TaskTogglePlan`, `plan_task_next`, `plan_task_link`, `set_task_line_status`,
  `ScheduledFieldMatch`.
- From `links`: `LinkPlanError`, `LinkInsertionOutcome`, `LinkInsertionPlan`,
  `LinkPlacement`, `plan_link_insertion`, `LinkRemovalPlan`, `plan_link_removal`,
  `select_implicit_open_entry`, `MovableLink`, `find_movable_task_links`.
- From `relocation`: `LinkRelocationError`, `LinkRelocationAction`, `PomodoroEndpoint`,
  `LinkRelocationPlan`, `plan_link_relocation`, `move_subtree_to_entry`,
  `insert_named_placeholder`, `endpoint_from_entry`.
- From `ledger`: `PomodoroLinkLedgerAction`, `PomodoroLinkLedgerPlan`,
  `plan_pomodoro_link_ledger`, `OpenEntryLinks`, `list_open_entry_links`.
- From `text`: `child_block_end_line`.

Known external users of that surface, which must stay on `capture_task_toggle::...`, are
`capture_active_tasks` (`list_open_entry_links` and `OpenEntryLinks` fields),
`randomize_plan` and `capture_pomodoro_close::linked_tasks` (`set_task_line_status`),
`capture_pomodoro_start` (`child_block_end_line`), `capture::ensure_next`,
`capture::task_toggle`, `capture::pomodoro_insert`, `capture::pomodoro_link`,
`capture::pomodoro_start`, and `capture::pomodoro_close`. `capture/mod.rs` only names
the module.

Keep field visibility as it is. `MovableLink` and `OpenEntryLinks` fields are
`pub(crate)`. `ScheduledFieldMatch` fields are private.

Helpers that are private today and must cross a child boundary become `pub(super)`, not
`pub(crate)`:

- `text`: `line_start_offset`, `line_index_at_offset`, `replace_line`,
  `insert_line_after`, `insert_line_before`, `insert_lines_after`, `extract_line_range`,
  `remove_line_range`, `reindent_subtree`, `logical_lines`,
  `pomodoro_child_indentation`.
- `links`: `insert_link_into_entry`.
- `relocation`: `destination_endpoint_after_move`, `relocation_from_link_plan_error`.

Leave these private to their new file: `physical_line_pieces`, `line_has_crlf`,
`unordered_child_indentation`, `remove_matching_links`, `ResolvedRelocationDestination`,
`resolve_relocation_destination`, and every status helper only `task_update` calls.

## Tests

The inline module has 49 `#[test]` functions. Move the body of `mod tests` into
`src/native/capture_task_toggle/tests.rs` and declare `#[cfg(test)] mod tests;` on the
facade. Do not wrap `tests.rs` in another `mod tests`. Keep `use super::*;` so the
re-exports stay in scope. Add `use chrono::NaiveDate;` in `tests.rs` if the facade no
longer imports it. Change no assertion, fixture, or test name. The module path
`native::capture_task_toggle::tests` stays, so `cargo test capture_task_toggle` still
selects this suite.

Integration coverage stays where it is: 10 tests in `tests/cli/capture/task_toggle.rs`
and 5 in `tests/cli/capture/ensure_next.rs`. Do not rewrite expected CLI output.

## Behavior to leave alone

This is a structural split of pure string-to-plan functions. There is no disk I/O to
introduce. In particular, preserve:

- sticky lanes: `plan_task_next` forces Next; `plan_task_link` turns only Ready and
  Blocked into Next and leaves Next and In Progress unchanged
- retirement of exactly one strictly future `scheduled` field, and the underscore
  Schedule Log emphasis (`PULL_FORWARD_ENTRY_EMPHASIS`), which deliberately differs from
  `capture_schedule_log::ENTRY_EMPHASIS`
- freshness stamping only inside `plan_task_link`
- implicit versus named entry selection, insertion idempotence, duplicate removal from
  later open entries, and completed-entry immunity
- subtree moves, destination indentation, CRLF, and a missing final newline

Do not reconcile comments with a different policy. Do not open or edit `bob-plugins`.

## Validation

Before editing, record baseline discovery:

```bash
cargo test capture_task_toggle -- --list
cargo test --test cli capture -- --list
cargo test --test randomize -- --list
```

After the move, `cargo fmt`, then confirm the same filters still execute the same tests
(module paths may show `capture_task_toggle::tests::...`; lost tests are a failure).
Run:

```bash
cargo test capture_task_toggle
cargo test --test cli capture
cargo test --test randomize
```

Then run `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
and `cargo test`). Use `/sase_monitor` for `just all` and any other cargo run that needs
a long handoff. Fix regressions this split causes. A failure that reproduces the same
way on the clean base tree does not keep `bob-cli-3s.2` open: record it as a
`PROPOSED FOLLOW-UP:` note and cite any task bead that already tracks it.

Count lines after formatting. Every `capture_task_toggle` Rust file, including tests,
must be at most 1500, with the headroom in the table above. Also confirm the completed
`capture_complete` files are still at most 1500. The other three epic files are still
monolithic and out of scope.

```bash
python3 - <<'PY'
from pathlib import Path
root = Path('src/native')
for name in ('capture_complete', 'capture_task_toggle'):
    paths = [root / f'{name}.rs', *(root / name).rglob('*.rs')]
    for path in sorted(p for p in paths if p.is_file()):
        count = len(path.read_bytes().splitlines())
        mark = ' OVER LIMIT' if count > 1500 else ''
        print(f'{count:5} {path}{mark}')
PY
```

## Completion

Do not create beads. Record discovered follow-up work on this phase only:

```bash
sase bead note bob-cli-3s.2 'PROPOSED FOLLOW-UP: <one-line summary — detail>'
```

Before closing, run `sase bead epic-symbols bob-cli-3s.2`. The Justfile in this tree
currently has no `--epic-symbol` entries; if any appear and are keyed to this phase,
resolve each symbol or re-key that line to a still-open bead (the parent epic
`bob-cli-3s` or a later phase). `sase bead close` refuses while leftovers remain.

Close only `bob-cli-3s.2`. Do not close `bob-cli-3s` or any ancestor. The close note
must include the final file map, line counts, the test commands that ran, and their
results.

```bash
sase bead close bob-cli-3s.2 --note "<what you verified>"
```
