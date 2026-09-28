---
tier: tale
title: Split tests/cli.rs into one cli integration target
goal:
  Replace the 35334-line tests/cli.rs file with a single Cargo test target at
  tests/cli/main.rs, a shared support module, and per-command test modules of at most
  1500 lines, keeping all 515 tests and the same behavior.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-2f.1
bead: bob-cli-2f.1
create_time: 2026-09-28 17:02:03
status: wip
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.1.md)

# Plan: Split tests/cli.rs into one cli integration target

This tale is the implementation of phase bead `bob-cli-2f.1` (parent epic `bob-cli-2f`,
design `plan:202609/split_largest_rust_files.md`, phase `split-cli-tests`). Do that
phase only. Leave the other nine oversized Rust files alone. Do not close `bob-cli-2f`
or any ancestor. Do not create beads.

## Outcome

`tests/cli.rs` is gone. Cargo still builds one integration-test binary named `cli`,
discovered from `tests/cli/main.rs`. `cargo test --test cli` still selects that binary.
The file is not fanned out into separate `tests/*.rs` targets.

Every Rust file this change creates or edits has at most 1500 lines. Aim for about
300–1200. A file under about 80 lines is too small unless it is a directory `mod.rs` or
`main.rs` whose only job is `mod` declarations. No test, assertion, helper body, doc
comment, or section comment changes except `mod` / `use` / visibility, `super::` paths,
`include_str!` relative paths, and the two `tests/randomize.rs` comments named below.

## Baseline (re-measure before moving anything)

At the start of implementation, on this tree:

- `wc -l tests/cli.rs` is 35334.
- The file contains 515 `#[test]` attributes.
- Helpers and types that are not tests sit in three regions: the preamble at lines 1–38
  (`TEMP_COUNTER`, bin consts, `LegacyHelpCase`), three priority helpers around lines
  6070 and 6830–6850 (`write_priority_config`, `priority_scheduled_offset_days`,
  `date_offset_days`), the shared helper block at lines 27875–28881 (`bob_command`
  through `remove_dir_all_if_exists`, including `TempDir`), and the Pomodoro-close
  helpers at lines 33652–33750 (`close_worked_vault`, `run_close_json`,
  `run_close_expect_error`).
- Five `include_str!("fixtures/...")` calls: three `fixtures/task_status_hooks/...`
  inside `task_status_hooks_syncs_fixture_and_is_idempotent`, and two
  `fixtures/tasks_parity/vault/...` inside
  `project_schedule_tasks_flip_between_dash_and_blocked_queries_when_due`. `fixture()`
  uses `env!("CARGO_MANIFEST_DIR")` and does not change.
- `tests/randomize.rs` mentions `tests/cli.rs` on its module doc (about line 5) and on
  the `init_vault_sync_pair` doc (about line 352). It keeps its own helper copies. Do
  not make it import `tests/cli/support`.
- `Cargo.toml` `include` already has `/tests/**`. There is no `[[test]]` table. Do not
  add one.
- `just all` is `cargo fmt --check`, then `cargo clippy --all-targets --all-features`,
  then `cargo test`. `rustfmt.toml` sets `max_width = 80`.

Record, before editing:

```bash
cargo test --test cli -- --list 2>/dev/null | grep -c ': test$'
```

The after count must equal that before count. Also require 515 `#[test]` attributes
under `tests/cli/`. On this Linux host the list count should be 515, because the three
`#[cfg]` tests are `unix` or `not(target_os = "macos")`.

If `tests/cli.rs` is already at most 1500 lines when implementation starts, stop and say
so. It is not, as of this plan.

## Target tree

`tests/cli/main.rs` holds the crate-level `//!` line and `mod` declarations only. Nested
directories use `mod.rs` the same way (`capture/mod.rs`, `highlights/mod.rs`,
`task_status_hooks/mod.rs`, `projects/mod.rs`): one-line `//!` plus `mod` lines. Those
index files may be under 80 lines.

Each test module starts with the imports it actually uses and `use crate::support::*;`.
Keep the original relative order of items inside each destination file.

Estimated body lines are the current test bodies plus about 30 lines of imports. They
are a budget, not a promise. After `cargo fmt`, split again on the boundary named in the
table if any file exceeds 1500, and prefer staying under 1400.

| File                             | Tests | Responsibility                                                                         | Body lines, approx. |
| -------------------------------- | ----: | -------------------------------------------------------------------------------------- | ------------------: |
| `support.rs`                     |     0 | Shared helpers used by two or more test modules                                        |           see below |
| `help.rs`                        |    25 | Cache extraction, native-only help, legacy and script help, top-level and nightly help |                 977 |
| `help_options.rs`                |    20 | Alphabetical option and subcommand help listings                                       |                 718 |
| `task_status_hooks/sync.rs`      |     6 | Fixture sync, grouping, rank, duplicate prune                                          |                1154 |
| `task_status_hooks/structure.rs` |    10 | Empty Pomodoros, guard rails, archive references, in-place strike, compose             |                 955 |
| `task_status_hooks/blocked.rs`   |     5 | Blocked status, schedules, recovery guard                                              |                 487 |
| `task_status_hooks/retry.rs`     |    10 | Lock, retry budget, cron redirection                                                   |                 647 |
| `dataview.rs`                    |    18 | `dataview_*` behavior tests                                                            |                 998 |
| `projects/list.rs`               |     2 | `projects_list_*`                                                                      |                 195 |
| `projects/sync.rs`               |    15 | `projects_sync_*` except the schedule tail below                                       |                 987 |
| `projects/schedule.rs`           |     5 | Schedule propagation, dash/blocked parity, sole `prj` task, per-file schedule errors   |                 341 |
| `plugins.rs`                     |    13 | `plugins_*` behavior tests                                                             |                 592 |
| `highlights/create.rs`           |    11 | `highlights_create_*`                                                                  |                 576 |
| `highlights/scan_hooks.rs`       |    10 | Pre-scan hook and legacy hook-key tests                                                |                 497 |
| `highlights/scan.rs`             |    15 | The other `highlights_ref_scan_*` tests                                                |                1025 |
| `highlights/sync.rs`             |     9 | Marker frontmatter sync and dirty-note guards                                          |                 466 |
| `highlights/sync_tasks.rs`       |    11 | Sidecar render and annotation-task routing                                             |                1139 |
| `highlights/tasks.rs`            |    10 | `highlights_ref_task_*` and `highlights_ref_status_*`                                  |                1105 |
| `highlights/marker.rs`           |    16 | Remaining `highlights_ref_*` (doctor, marker edit, frontmatter, tombstone)             |                1124 |
| `move_done.rs`                   |    11 | `move_done_*`                                                                          |                 922 |
| `pomodoro.rs`                    |    10 | `pomodoro_*`, `tmux_*`, `script_pomodoro*` binaries (not help tests)                   |                 233 |
| `vault_sync.rs`                  |    17 | `vault_sync_*`, conflict dir, renamed commands, `nightly_*`, `executable_stubs_*`      |                 806 |
| `capture/parse.rs`               |    24 | `capture_parse_*` except the Pomodoro protocol tests                                   |                1099 |
| `capture/parse_pomodoro.rs`      |     7 | `capture_parse_pomodoro_{adjust,shift,close,start}_*`                                  |                 947 |
| `capture/rewrite.rs`             |    12 | `capture_rewrite_*` and `capture_complete_and_rewrite_*`                               |                 387 |
| `capture/complete_query.rs`      |    12 | Route, marker, section, task, and global completion JSON                               |                 695 |
| `capture/complete_editor.rs`     |    18 | Pomodoro-name completion, task-section completion, cursor and empty-success paths      |                 986 |
| `capture/project_note.rs`        |     6 | Names containing `project_note`                                                        |                 641 |
| `capture/priority.rs`            |    12 | `capture_priority_*`, `capture_without_priority_*`, plus the three priority helpers    |                 537 |
| `capture/task_marker.rs`         |     5 | Task block-id marker and retired `::` marker                                           |                 222 |
| `capture/task_toggle.rs`         |     5 | `capture_task_toggle_*` except `ensure_next`                                           |                 526 |
| `capture/ensure_next.rs`         |     4 | `capture_task_toggle_ensure_next_*` and `capture_task_toggle_named_ensure_next_*`      |                 707 |
| `capture/batch.rs`               |    11 | `capture_batch_*` and tests whose names contain `global_`                              |                 469 |
| `capture/clip.rs`                |    15 | Clip, history, clipboard, and `%1`                                                     |                1093 |
| `capture/authored.rs`            |    19 | Names containing `authored`                                                            |                 729 |
| `capture/sub_bullet.rs`          |     7 | `capture_sub_bullet_*`                                                                 |                 741 |
| `capture/bare.rs`                |    11 | `bare_terminal` and `bare_hash`                                                        |                 451 |
| `capture/sections.rs`            |    14 | Forced section, task section, `capture_sections_*`, `capture_tasks_*`                  |                1062 |
| `capture/targets.rs`             |     5 | `capture_targets_*`                                                                    |                 244 |
| `capture/task_id.rs`             |     4 | `capture_task_id_*` and `capture_task_sections_*`                                      |                 519 |
| `capture/routing.rs`             |    26 | Remaining `capture_*` inbox, schedule, dry-run, JSON, and bullet routing               |                 923 |
| `capture/pomodoro_link.rs`       |     9 | Linked-task and named-Pomodoro execution, including malformed marker                   |                 570 |
| `capture/pomodoro_name.rs`       |     8 | `capture_pomodoros_*`, `capture_pomodoro_name_*`, human start name, link solo grammar  |                1064 |
| `capture/pomodoro_start.rs`      |    15 | `capture_pomodoro_start_*`                                                             |                1047 |
| `capture/pomodoro_whole_item.rs` |     6 | `capture_pomodoro_whole_item_start_*`                                                  |                1152 |
| `capture/pomodoro_adjust.rs`     |     5 | `pomodoro_adjust` execution                                                            |                 532 |
| `capture/pomodoro_shift.rs`      |     5 | `pomodoro_shift` execution                                                             |                 798 |
| `capture/pomodoro_close.rs`      |     3 | `pomodoro_close` execution, plus the three close helpers                               |                1076 |

Help tests are every test whose name is
`cache_extraction_writes_expected_files_and_modes`,
`all_top_level_subcommand_help_is_safe_and_plain`,
`public_help_surfaces_do_not_list_long_only_options`,
`legacy_binary_help_is_safe_and_plain`, `script_fallback_help_is_safe_and_plain`,
`pomodoro_help_documents_show_stale_option`,
`nightly_help_exits_before_operational_work`, `highlights_ref_subcommand_help_works`,
`top_level_help_lists_commands_alphabetically_with_examples`, or that ends in
`_help_is_native_only`, `_help_is_native_only_and_defaults_to_run`,
`_help_lists_options_alphabetically`, `_help_lists_subcommands_alphabetically`,
`_help_lists_subcommands_and_options`, or `_help_lists_subcommand_and_options`.
Option-list names go to `help_options.rs`. The others go to `help.rs`. Assign help
before any command-family rule so `capture_parse_help_*` does not land in
`capture/parse.rs`.

`projects/schedule.rs` receives exactly these five tests:
`projects_sync_propagates_scheduled_task_properties_at_date_boundary`,
`projects_sync_then_task_status_hooks_blocks_and_recovers_propagated_tasks`,
`project_schedule_tasks_flip_between_dash_and_blocked_queries_when_due`,
`projects_sync_shows_sole_prj_task_when_schedule_is_due`,
`projects_schedule_errors_are_per_file_and_leave_invalid_file_untouched`.

`highlights/scan_hooks.rs` receives `highlights_ref_scan_*` tests whose names continue
`runs_configured_pre_scan`, `dry_run_reports_env_pre_scan`, `empty_env_disables`,
`fails_when_pre_scan`, `no_hooks_`, `hook_child_`, or `rejects_legacy_pre_scan`.

`highlights/sync_tasks.rs` receives `highlights_ref_sync_*` tests whose names contain
`sidecar`, `textbundle`, `linked_sidecar`, `annotation_task`, `pdf_note_task`, `routed`,
`legacy_highlight_task`, `manual_sections`, `missing_routed`, `skips_annotation`,
`skips_vault_scan`, or `skips_legacy`.

`capture/complete_query.rs` receives `capture_complete_global_*`,
`capture_complete_all_tasks_*`, `capture_complete_inherited_*`,
`capture_complete_completes_a_marker_*`, `capture_complete_scopes_*`,
`capture_complete_route_*`, `capture_complete_task_block_id_*`,
`capture_complete_section_*`, and `capture_complete_task_json_*`. Every other
`capture_complete_*` test goes to `capture/complete_editor.rs`, except
`capture_complete_and_rewrite_*`, which stays in `capture/rewrite.rs`.

`capture/pomodoro_link.rs` receives `capture_pomodoro_linked*`,
`capture_pomodoro_link_uses*`, `capture_pomodoro_dry_run*`,
`capture_pomodoro_preflight*`, `capture_pomodoro_missing*`,
`capture_malformed_pomodoro*`, and `capture_named_pomodoro*`. `capture/pomodoro_name.rs`
receives `capture_pomodoros_*`, `capture_pomodoro_name_*`,
`capture_human_names_the_starting*`, and `capture_pomodoro_link_solo*`.

Apply rules in the order above so a name that mentions Pomodoro only in passing (project
note, priority render, clip, authored bullets, bare terminal marker) stays with that
feature. Parse and complete protocol tests stay in the parse and complete modules, not
in the execution modules.

The 515 tests must all be assigned. A name that matches none of these rules is a bug in
the splitter. Fix the assignment. Do not drop the test.

## How to split

1. `git mv tests/cli.rs tests/cli/main.rs` before any edit, so history can follow the
   file.
2. Split with a throwaway script under `/tmp`, not a committed generator. Detect real
   items only: column-0 `fn`, `struct`, `enum`, `const`, `static`, and `impl`. Ignore
   column-0 `type:` lines. Those are YAML inside raw strings, not Rust items. Attach the
   attribute and comment lines directly above each item, including `#[cfg(...)]` and
   `#[test]`.
3. Move the priority helpers with the priority tests, and the three close helpers with
   the close tests. Keep both `#[cfg(unix)]` and `#[cfg(not(unix))]` copies of
   `assert_unix_mode` and of `set_mode` together.
4. Put every other helper, const, static, and `TempDir` into `support` when at least two
   destination modules call it. A helper used by one module stays private in that
   module. `LegacyHelpCase` stays private in `help.rs`.
5. Make shared items `pub(crate)`. `TempDir`'s `path` field stays private. `new` and
   `path` become `pub(crate)`. Do not widen anything to `pub`.
6. Rewrite each `include_str!("fixtures/...")` to a relative path from the destination
   file to `tests/fixtures/...`. From `tests/cli/task_status_hooks/sync.rs` and
   `tests/cli/projects/schedule.rs` that is `../../fixtures/...`. Do not move fixture
   files.
7. Give `main.rs` and each new module a one-line `//!` summary. The original file has no
   crate docs to preserve.
8. Update the two `tests/randomize.rs` comments so they name `tests/cli/support.rs` (and
   `tests/cli/support.rs::init_vault_sync_pair`) instead of `tests/cli.rs`. Then
   `rg -n 'tests/cli\.rs' src tests docs` and update any other comment that would now
   point at a file that does not exist. Do not rewrite comments that merely describe
   behavior.
9. Run `cargo fmt`. Trim unused imports until
   `cargo clippy --all-targets --all-features` reports no new warning caused by the
   move. Watch `dead_code` and `unused_imports` on helpers that used to be visible
   because everything lived in one file.

Preserve the assertion in `capture_pomodoro_link_solo_grammar_and_atomic_execution` that
ends in `|| true` (source line 31821 today). Do not simplify it and do not add
`#[allow]`. Prior phase notes already treat `clippy::overly_complex_bool_expr` on that
expression as pre-existing, cited against `bob-cli-28` and `bob-cli-v`. Confirm those
beads with `sase bead search` before citing them. If `just all` fails only on that same
expression, check that `git show HEAD:tests/cli.rs` still contains it, record
`PROPOSED FOLLOW-UP:` on `bob-cli-2f.1` naming the existing bead, and continue the
close. Any new clippy error or warning is in scope and must be fixed by restoring a
visibility or import, not by editing test logic.

## Verification

- `find tests/cli -name '*.rs' | xargs wc -l` shows no file over 1500 lines, and
  `tests/cli.rs` is gone.
- `rg -c '#\[test\]'` across `tests/cli` sums to 515, and
  `cargo test --test cli -- --list` matches the count recorded before the move.
- `cargo package --list --allow-dirty` still lists the new files under `tests/cli/`.
- `just all` passes, aside from the pre-existing `|| true` diagnostic if that is the
  only failure and it reproduces from `HEAD`.
- In the close note, list each new file, what it holds, and its line count.

Before closing, run `sase bead epic-symbols bob-cli-2f.1`. If any `--epic-symbol` entry
remains, resolve it or re-key that Justfile line to a still-open bead (the parent epic
or a later phase). Then close only this bead:

```bash
sase bead close bob-cli-2f.1 --note "<what you verified>"
```

Discovered follow-up work, including a clean-base check failure, goes on this bead as
`sase bead note bob-cli-2f.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. Do not
file a new bead.
