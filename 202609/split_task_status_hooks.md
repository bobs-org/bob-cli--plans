---
tier: tale
size: medium
title: Split task_status_hooks into modules of at most 1500 lines
goal: 'Turn src/native/task_status_hooks.rs into a directory module whose every file
  is at most 1500 lines, without changing behavior, test assertions, or the paths
  outside callers already use.

  '
proposed_by: bbugyi200.apollo.bob-cli-2f.6
bead: bob-cli-2f.6
status: done
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.6.md)

# Split `src/native/task_status_hooks.rs`

Implement phase bead `bob-cli-2f.6` (parent epic `bob-cli-2f`). This is a pure
structural refactor of one file. The measurements below were taken on the clean tree at
`2307179`: `src/native/task_status_hooks.rs` is 6107 lines (production code through line
4204, then 60 `#[test]` functions from line 4205 through 6107). That matches the epic
landmark, so the line numbers below are current.

## Outcome

- `src/native/task_status_hooks.rs` is gone. The module is
  `src/native/task_status_hooks/mod.rs` plus the submodules in the layout below.
- Every Rust file this change creates or edits has at most 1500 lines. Target roughly
  150–1200 lines per file.
- Behavior, names, doc comments, `//` section comments, assertion text, and the relative
  order of items inside each moved group stay as they are.
- The 60 unit tests all still run. A workspace-wide `cargo test -- --list` test count
  taken before the move matches the count after it.
- `just all` passes: `cargo fmt --check`, `cargo clippy --all-targets --all-features`,
  and `cargo test`, with no new compiler or clippy warnings.
- Callers outside the directory keep compiling unchanged: `src/native.rs`
  (`mod task_status_hooks;` and `task_status_hooks::run`), `src/native/randomize.rs`
  (`daily_anchor_date`, `markdown_files`, `read_tasks_settings`,
  `validate_blocked_status`), `src/native/randomize_plan.rs` (`TasksSettings`,
  `grouping_eligible_note`, `task_group_classification`), and `src/native/note_tasks.rs`
  (`TasksSettings`, `TaskStatusType`, `read_tasks_settings`).

## Layout

Use a directory module, the same shape as `gkeep/` and `highlights_ref/`. `git mv` the
file to `mod.rs` before editing so history follows. Do not use the `dataview.rs` +
`dataview/` pair.

`mod.rs` owns the facade: the four top-level constants (`POMODORO_MARKER`,
`COMMAND_NAME`, `DEFAULT_GLOBAL_FILTER`, `TASKS_SETTINGS`), `run`, `print_clap_error`,
`build_cli`, `OutputFormat`, and `Request` (current lines 1–238), plus `mod`
declarations, private `use` globs, and the `pub(crate)` re-exports. Give `mod.rs` no
extra logic.

The epic's sketch put daily-note selection in `retry.rs` and left the non-retry tests as
two files. Re-cut those two points. Daily-note selection is sync behavior
(`sync_task_statuses` is the production caller; `note_kind` and `canonical_daily_date`
are also used by grouping). The non-retry tests are about 1568 lines, which is over the
file cap, so they become two test files plus a shared fixture module. Everything else
follows the epic sketch.

| File                 | Responsibility                                                                                                 | Source lines (current file) |
| -------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------- |
| `mod.rs`             | Constants, CLI, `run`, `OutputFormat`, `Request`, submodule declarations, re-exports                           | 1–238                       |
| `model.rs`           | Report, result, scan, and plan types through `RankedStatus`                                                    | 240–672                     |
| `retry.rs`           | `RetryEnv`, delay, logging, `retry_loop`, `run_with_retries`, and the 13 retry tests                           | 741–972, tests 4206–4539    |
| `sync.rs`            | Daily-note selection and `note_kind`, then `sync_task_statuses` and dependency transitions                     | 673–740, then 973–1568      |
| `pomodoro.rs`        | Logical lines, Pomodoro scan, link tokens, marker expectations, strikethrough spans, empty-Pomodoro plan/apply | 1569–1870 and 2417–2458     |
| `settings.rs`        | Read, parse, and validate Tasks settings                                                                       | 1871–2008                   |
| `structure.rs`       | Fenced and bullet extents, duplicate-line removal, structural plan/apply, `reindent_segment`                   | 2009–2416 and 2459–2474     |
| `parse.rs`           | Markdown walk, task-line and metadata parsing, dates, block ids, `task_blocks`                                 | 2475–2772                   |
| `references.rs`      | Archive catalog, `ReferenceContext`, resolution, edges, reachability, desired statuses, change rows            | 2773–3343                   |
| `compose.rs`         | `compose_outputs`, grouping eligibility and reports, guarded apply, capture-error mapping                      | 3344–3648                   |
| `output.rs`          | Human and JSON printing, `SyncError`, `print_error`                                                            | 3649–4204                   |
| `tests/mod.rs`       | Shared fixtures only                                                                                           | helpers at 4540–4612        |
| `tests/sync.rs`      | Daily, reference, parse, and status-transition tests (20)                                                      | listed below                |
| `tests/structure.rs` | Pomodoro, duplicate, strike, move, cancel, and empty-block tests (27)                                          | listed below                |

`reindent_segment` stays in `structure.rs`. Only `apply_structural_plan` calls it.
`plan_empty_pomodoro_removals` and `apply_empty_pomodoro_plan` move together, in that
order, to `pomodoro.rs`.

Inside `sync.rs`, keep the daily-note block (`daily_anchor_date`,
`canonical_daily_date`, `previous_daily_path`, `note_kind`) above `sync_task_statuses`,
which is their original relative order. Do not interleave them with the retry code that
currently sits between them.

If a finished file is over 1500 lines, split that file again on the same cohesion rules
before declaring the phase done. Do not create a file under about 80 lines unless the
unit is genuinely standalone. `settings.rs` is a small but real I/O boundary (about 140
lines) and should stay its own file.

## Visibility and imports

Follow the `src/native/dataview.rs` facade pattern.

- Each child file starts with a one-line `//!` summary. The original file has no `//!`
  docs; do not invent a long module doc on `mod.rs`.
- `mod.rs` declares every child, then `use child::*;` so sibling code and tests can say
  `super::name`. Children call siblings through those parent names. Do not import a
  sibling module directly unless two public names collide.
- A name used only inside its new file stays private.
- A name used by a sibling module or by `tests/` becomes `pub(super)`.
- Struct and enum fields that another submodule or a test constructs or reads become
  `pub(super)`. `SyncResult` is the large case: `empty_sync_result` builds every field.
  Leave fields that are already `pub(crate)` unchanged.
- Do not widen any item to `pub(crate)` unless it already is. Do not narrow an existing
  `pub(crate)` item.
- Re-export the existing external surface from `mod.rs` so outside paths are unchanged:
  - `run` stays `pub(crate)` in `mod.rs`
  - `pub(crate) use model::{TaskStatusDefinition, TaskStatusType, TasksSettings};`
  - `pub(crate) use sync::daily_anchor_date;`
  - `pub(crate) use settings::{read_tasks_settings, validate_blocked_status};`
  - `pub(crate) use parse::markdown_files;`
  - `pub(crate) use compose::{grouping_eligible_note, task_group_classification};`
  - `pub(crate) use output::SyncError;`

- `TaskStatusDefinition` and `SyncError` have no outside callers today. Still re-export
  them, because they are already `pub(crate)` at `task_status_hooks::…`.
- Parent-private constants stay in `mod.rs`. Children may name `super::COMMAND_NAME`,
  `super::POMODORO_MARKER`, `super::TASKS_SETTINGS`, and `super::DEFAULT_GLOBAL_FILTER`.
- Fix imports per file. Drop imports a file no longer uses. Expect clippy to flag
  `dead_code` and `unused_imports` until visibility matches actual use.
- `#[cfg(test)]` helpers that exist only to build retry fakes (`MockEnv`,
  `empty_sync_result`, `sync_error`, `mock_env`, `scripted_attempts`) stay inside
  `retry.rs`'s test module. Do not lift them into production code.

There is no `include_str!` or `include_bytes!` in this file. Nothing in `docs/` or other
Rust files names the path `task_status_hooks.rs`. After the move, search `src`, `tests`,
and `docs` for that filename anyway and update a comment only when it would now point at
the wrong file.

## Tests

Record the baseline before moving anything:

```bash
cargo test -- --list 2>/dev/null | grep -c ': test$'
```

Keep all 60 tests. Do not change assertion strings, inputs, or helper behavior. Within
each destination file, keep the original source order of helpers and tests.

`retry.rs` gets `#[cfg(test)] mod tests` with `use super::*;` (plus `super::super` names
for types that live in other submodules). Move these helpers and tests together:
`empty_sync_result`, `sync_error`, `mock_env`, `scripted_attempts`, and

- `retry_ceiling_progression_caps_at_thirty_seconds`
- `retry_delay_stays_within_upper_half_of_ceiling`
- `random_unit_interval_varies_and_stays_in_unit_range`
- `allowed_transient_reasons_are_retried_without_applied_files`
- `terminal_and_unknown_reasons_are_never_retried`
- `partial_apply_is_never_retried_even_with_no_applied_files`
- `any_error_listing_applied_files_is_never_retried`
- `retry_loop_retries_lock_contention_then_succeeds`
- `retry_loop_stops_immediately_on_terminal_reason_without_noise`
- `retry_loop_never_starts_another_attempt_once_budget_is_exhausted`
- `retry_loop_clamps_sleep_to_remaining_budget`
- `retry_loop_with_zero_budget_makes_one_attempt_and_never_sleeps`
- `retry_decision_log_includes_run_attempt_reason_and_recovery_directory`

`tests/mod.rs` holds only the shared fixtures, in this order: `reference`, `identity`,
`test_settings`, `resolved`, `resolved_paths`, `date`. Declare `mod sync;` and
`mod structure;`. Re-export the parent engine with `pub(super) use super::*;` so the
child test files can `use super::*;` and still see `pub(super)` items. Parent `mod.rs`
declares `#[cfg(test)] mod tests;`.

`tests/sync.rs` (20 tests, original order):

- `previous_daily_selection_uses_latest_canonical_earlier_date`
- `dated_day_file_overrides_effective_anchor_and_malformed_name_falls_back`
- `recent_links_include_completed_live_links_but_exclude_retired_links`
- `same_note_recent_links_resolve_in_each_daily_context`
- `note_kind_uses_shared_area_and_project_frontmatter_predicates`
- `rolling_reachability_is_cycle_safe_and_includes_dependencies`
- `resolves_exact_and_unique_case_insensitive_basenames`
- `ambiguous_basename_does_not_resolve`
- `dotted_note_names_keep_the_full_basename`
- `fenced_column_zero_content_does_not_end_dependency_scan`
- `parses_task_markers_and_preserves_status_offsets`
- `parses_bracket_and_parenthesized_task_dependency_metadata`
- `parses_only_calendar_valid_scheduled_metadata_in_supported_forms`
- `future_schedule_uses_the_calendar_day_after_the_anchor`
- `task_dependency_index_matches_tasks_duplicate_and_missing_id_semantics`
- `replacement_changes_only_status_and_preserves_crlf`
- `transition_matrix_promotes_monotonically_and_clears_only_unreferenced_next`
- `blocked_transition_precedence_and_recovery_are_explicit`
- `desired_statuses_merge_parents_and_propagate_stronger_intermediates_through_cycles`
- `recovery_rank_defaults_blocked_roots_to_next_and_propagates_in_progress`

`tests/structure.rs` (27 tests, original order):

- `extracts_only_block_links_under_open_pomodoros`
- `duplicate_lines_use_canonical_task_identity_and_first_open_owner`
- `deleted_conflict_line_cannot_claim_an_unrelated_task`
- `duplicate_cleanup_ignores_distinct_unresolved_and_ineligible_links`
- `full_line_deletion_preserves_children_crlf_and_final_line_ending`
- `deleted_completed_duplicate_is_not_retired_moved_or_reinserted`
- `struck_references_are_retired_and_spans_are_paired`
- `completed_fallback_does_not_take_mixed_live_bullets`
- `parses_embedded_alias_and_mixed_block_links`
- `parses_and_normalizes_pomodoro_marker_prefixes_per_link`
- `dependency_reference_requires_a_sole_transcluded_block_link`
- `moves_completed_mixed_bullet_subtree_to_current_and_strikes_only_done`
- `repairs_completed_pomodoro_links_in_place_and_is_idempotent`
- `repairs_markers_by_owner_and_marks_completed_fallback_moves`
- `conflicting_duplicate_statuses_are_not_normalized`
- `canceled_reference_removal_deletes_complete_mixed_content_items`
- `canceled_subtrees_compose_with_nested_and_moving_bullets`
- `canceled_subtree_deletion_preserves_crlf_and_no_final_newline`
- `duplicate_deleted_lines_do_not_report_canceled_reference_edits`
- `direct_child_scan_counts_plain_children_but_ignores_fences`
- `empty_pomodoro_deletion_removes_full_blocks_and_preserves_crlf_eof`
- `entries_emptied_by_duplicate_cleanup_are_removed_in_same_pass`
- `moving_last_child_removes_source_but_retains_destination`
- `empty_timed_entries_are_not_current_targets_or_ambiguity_inputs`
- `two_non_empty_timed_entries_still_match_the_ambiguity_guard`
- `completion_classification_accepts_conventional_and_custom_done_only`
- `cancellation_classification_uses_recognized_tasks_status_types`

Integration tests under `tests/cli/task_status_hooks/` already live in their own files.
Leave them untouched. Their `include_str!` paths are relative to those files and must
keep working because those files do not move.

## Constraints

- Move code verbatim. Change only `mod` and `use` lines, visibility, and `super::` paths
  required by the move. Do not rename items, reorder items inside a moved group, or
  clean up logic.
- Edit only this module, plus a comment elsewhere if a search shows it names the old
  filename incorrectly. Leave `task_status_hooks_write.rs`, `projects.rs`, and every
  other oversized file alone. Phase `bob-cli-2f.7` is already in progress on
  `projects.rs`.
- `rustfmt.toml` pins `max_width = 80`. Run `cargo fmt` on the moved files rather than
  hand-wrapping.
- The workspace tree was clean at planning time. If other agents have since edited
  unrelated files, leave those edits in place.

## Verification

1. Compare the saved `cargo test -- --list` count with a fresh count. Both totals match,
   and `native::task_status_hooks` still lists 60 tests.
2. `find src/native/task_status_hooks -name '*.rs' | xargs wc -l` shows every file at
   most 1500 lines.
3. `cargo fmt`, then `just all`.
4. `git diff --stat` shows the deletion of `src/native/task_status_hooks.rs` and the new
   directory. `src/native.rs`, `randomize.rs`, `randomize_plan.rs`, `note_tasks.rs`, and
   `task_status_hooks_write.rs` stay unchanged unless a filename comment had to be
   corrected.

## Close-out

Close only `bob-cli-2f.6`. Do not close parent epic `bob-cli-2f` or any ancestor plan
bead.

Before closing, run `sase bead epic-symbols bob-cli-2f.6`. Planning time found no
`--epic-symbol` entries for this phase. If any are present when the work finishes,
resolve each symbol or re-key that Justfile line to a bead that is still open (the
parent epic or a later phase). `sase bead close` refuses while leftovers remain.

Then:

```bash
sase bead close bob-cli-2f.6 --note "<what you verified: file list and line counts, test count unchanged, just all green>"
```

Do not create beads. Record any discovered follow-up on this phase before closing:

```bash
sase bead note bob-cli-2f.6 'PROPOSED FOLLOW-UP: <one-line summary — detail>'
```

A `just all` failure that reproduces identically on the clean base tree does not keep
the phase open. Note it as a `PROPOSED FOLLOW-UP:` (cite any task bead that already
tracks it) and close the phase anyway.
