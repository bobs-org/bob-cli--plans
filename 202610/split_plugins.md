---
tier: tale
title: Split plugin management into focused Rust modules
goal: Complete bob-cli-3s.5 by splitting plugin management into cohesive modules of
  at most 1500 lines while preserving behavior, coverage, and the five-file size audit.
size: medium
proposed_by: bbugyi200.athena.bob-cli-3s.5
bead: bob-cli-3s.5
status: done
---

- **PARENT:**
  [202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-3s.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/bob-cli-3s.5.md)

# Split plugin management into focused Rust modules

Complete phase bead `bob-cli-3s.5` from `plan:202610/split_largest_rust_files.md`. This
is a behavior-preserving structural refactor of `src/native/plugins.rs`, including its
existing tests. Keep every resulting Rust file at most 1500 physical lines after
formatting. Implement this tale as one bounded work unit; do not split it into another
epic.

This is the last implementation phase. After the plugin split, verify the cumulative
file map and line counts for all five components and run the full checks on that tree.
That verification is part of this phase. It is not a sixth implementation phase, and it
does not authorize editing the four components that earlier phases already split.

## Inspected baseline and scope

The planning checkout is clean at `c2c54a4b5da85a67555f5f7d085ad84d37630400`, the
`capture_clip` split commit. `src/native/plugins.rs` is unchanged from the epic
inventory: 2215 physical lines, 1526 lines before `#[cfg(test)] mod tests`, and 689
lines of tests. Source inspection finds 17 unit tests and 13 integration tests in
`tests/cli/plugins.rs`. These are source counts, not claimed test executions. Capture
executable discovery and focused outcomes before moving code.

`plugins.rs` is still a single file. There is no `src/native/plugins/` directory. The
other four components are already split and under the 1500-line ceiling
(`capture_complete`, `capture_task_toggle`, `task_status_hooks_write`, `capture_clip`).
Leave those trees alone.

The only production callers are:

- `src/native.rs`: `mod plugins;` and `NativeCommand::Plugins => plugins::run(args)`.
- `src/native/completion/tree.rs`: `crate::native::plugins::build_cli()`.

`run` and `build_cli` are the only `pub(crate)` items. Preserve both paths and
signatures. `docs/plugins.md` is the behavior contract. Do not edit it, `Cargo.toml`,
the Justfile, plugin sources, or the linked `bob-plugins` repo. Do not deploy with
`bob plugins sync`. Keep changes outside this component limited to wiring the compiler
actually requires. None is expected.

An initial `sase bead epic-symbols bob-cli-3s.5` found no entries. The Justfile has no
`--epic-symbol` line for this phase. Check again immediately before closure.

## Why this layout

The epic's recommended buckets match the current code. The adjustments below are only
where a literal reading would cycle or split a type from the methods that own its data.

Production dependencies must flow in one direction:

- `cli` uses `git`, `scan`, `sync`, `render`, and `model`.
- `sync` uses `model`, `diff`, `git` (`vault_file_is_dirty` only), and `scan`
  (`read_manifest` and `read_sorted_directory` only).
- `scan`, `diff`, and `git` use `model`.
- `render` uses `model`.
- `model` depends on serde and std only. It does not use `Styler`, child modules, or
  I/O.

`COMMAND_NAME` and the shared path constants (`REPO_PLUGINS_SUBDIR`,
`VAULT_PLUGINS_SUBDIR`, `COMMUNITY_PLUGINS_FILE`, `MANAGED_FILES`) live in `model`.
`cli` and `git` both print `COMMAND_NAME`, and `scan` and `sync` both walk the same
directories. Putting the command name in `cli` would make `git` depend on `cli` while
`cli` calls `pull_repo`.

`FileOutcome` stays in `sync.rs`. It is the per-file result folded into `FileSync`
before the report is returned, and tests never name it. `FileDiff`, `DiffLine`, and
`DiffKind` stay in `model` because both `diff` and `render` consume them.

`SyncReport` aggregation (`files`, `copied`, `skipped`, `unchanged`, `has_backup_paths`,
`has_written_backups`) and `PluginsReport` aggregation (`counts`, `result`,
`issue_summary`) stay on those types in `model`. `files` can remain a private method.
Presentation stays in `render`: move `impl SyncState` (`label`, `glyph`, `paint`) and
`impl VaultState` (`label`, `paint`) there so call sites such as
`plugin.sync.label(...)` do not change and `model` does not take a style dependency.
`FileAction::{is_copy, is_warning}` stay in `model`; `render` calls them.

`read_sorted_directory` and `read_manifest` live in `scan` and are reused by `sync`.
Duplicating the directory walk or the manifest parse would let the two paths drift.
`vault_file_is_dirty` lives in `git` with the pull helpers. Keep its direct
`Command::new("git")` invocation. The pull path keeps `ob::git_command` and
`ob::child_env`. Do not unify those two Git call styles.

`success_json` lives in `render` next to the table. Serde derives stay on the model
types. Diff thresholds (`DIFF_CONTEXT_LINES`, `DIFF_BODY_LINE_LIMIT`,
`MINIFIED_BYTE_THRESHOLD`, `MINIFIED_LINE_THRESHOLD`) stay in `diff` beside the
classifier. `DETAIL_INDENT` stays in `render`. `byte_line_count` stays in `diff`; `sync`
calls it when it builds `FileDiff::NewFile`.

The 689-line suite fits in one `tests.rs` with substantial headroom. Discovery, pull,
and sync tests share `TempDir`, `TEMP_COUNTER`, and the Git fixture helpers. A `tests/`
directory would only reshuffle that shared support. Keep a single counter in `tests.rs`.
There is no production temporary counter to preserve.

Enum variant fields cannot carry their own visibility (rustc E0449, already hit on phase
`bob-cli-3s.3`). Give the enum `pub(super)` and leave variant fields unqualified. Struct
fields that siblings or tests read are `pub(super)` individually.

## Selected file layout

Retain `src/native/plugins.rs` as the single module root. Create the children under
`src/native/plugins/`. Do not also create `plugins/mod.rs`. Counts are approximate
placement budgets, not targets to obtain by squeezing formatting.

| File         | Responsibility                                                                                          | Approximate lines |
| ------------ | ------------------------------------------------------------------------------------------------------- | ----------------: |
| `plugins.rs` | Facade, child declarations, and explicit `run` / `build_cli` re-exports                                 |             30–50 |
| `cli.rs`     | Clap commands and args, option resolution, list/sync dispatch, exit status, pull-before-analysis gate   |           300–360 |
| `model.rs`   | Shared constants, manifests, list/sync reports, state enums, sync outcomes, aggregation methods         |           280–360 |
| `scan.rs`    | Repo discovery, manifest parse, enabled-plugin set, managed-file byte comparison, sorted directory read |           150–200 |
| `git.rs`     | Worktree detection, non-interactive pull, pull summaries, vault dirty check                             |           180–230 |
| `sync.rs`    | Per-plugin and per-file copy decisions, backup-before-overwrite, dry-run, `FileOutcome`                 |           270–340 |
| `diff.rs`    | Text, binary, and minified diff classification and unified-diff truncation                              |           110–160 |
| `render.rs`  | List table, sync report, diff painting, JSON encoding, state label/paint methods                        |           340–420 |
| `tests.rs`   | All 17 current unit tests, fixtures, and the one `TEMP_COUNTER`                                         |           700–780 |

The root re-exports only:

```rust
pub(crate) use cli::{build_cli, run};
```

Do not re-export scan, sync, diff, or render helpers. Tests import the child items they
already call. Use explicit imports in each child. From a child, reach siblings through
`super::` and existing native helpers through `crate::native::env as bob_env`,
`crate::native::ob`, and `crate::native::style::{...}`.

Visibility ceiling inside the component is `pub(super)`, except `run` and `build_cli`.
Items the moved tests or sibling modules call include:

- `model`: `SyncOptions`, `SyncReport`, `PluginSync`, `FileSync`, `FileDiff`,
  `DiffLine`, `DiffKind`, `BackupOutcome`, `FileAction`, `PluginsReport`, `StateCounts`,
  `PluginsResult`, `PluginEntry`, `SyncState`, `VaultState`, `Manifest`, and the fields
  those siblings construct or read. `COMMAND_NAME` and the four path/`MANAGED_FILES`
  constants.
- `scan`: `scan_plugins`, `sync_state`, `vault_state`, `read_manifest`,
  `read_sorted_directory`.
- `git`: `pull_repo`, `print_pull_outcome`, `PullOutcome`, `vault_file_is_dirty`.
- `diff`: `diff_existing_file`, `diff_text`, `byte_line_count`,
  `MINIFIED_BYTE_THRESHOLD`.
- `sync`: `sync_plugins`.
- `render`: `print_plugins_table`, `print_sync_report`, `success_json`.

`OutputFormat`, Clap builders, pull summarizers, and `sync_one_file` stay private to the
child that owns them. Helpers used only by tests keep their current test-only location
in `tests.rs`. The `truncate` unit test stays in that suite and imports
`crate::native::style::truncate` itself. Do not move it into `style` and do not drop it.

Replace the test module's `use super::*` with explicit imports. `use super::*` will no
longer see child functions. Keep test function names, assertions, fixture layout, and
cleanup. Add a regression test only if the split reveals a missing behavior check. Do
not add tests of module organization.

## Implementation steps

1. Re-read `bob-cli-3s.5` and `plan:202610/split_largest_rust_files.md` with audited
   commands. Confirm the tree and the counts above before editing. Do not undo an
   earlier phase's split. Record the base revision. Capture the baseline discovery and
   focused test outcomes listed below before moving production or test code.
2. Extract `model` first: constants, report and outcome types, serde attributes, and
   aggregation impls. Keep struct field order. `PluginsResult` field order is the JSON
   object order (`ok`, `repo`, `bob_dir`, `count`, `synced`, `drift`, `not_installed`,
   `plugins`). `PluginEntry` stays `id`, `version`, `description`, `sync`, `vault`. Keep
   `rename_all = "snake_case"` on `SyncState` and `VaultState`, and `#[serde(default)]`
   on `Manifest`.
3. Extract `diff` and `git` as leaves above `model`. Preserve every pull summary branch,
   the silent up-to-date check, and the dirty-check failure policy (non-success or a
   missing `git` means not dirty).
4. Extract `scan`, then `sync`. Keep the managed-file loop in its current order
   (`manifest.json`, `main.js`, `styles.css`), the folder-name fallback for an empty
   manifest id, id sorting, and the unknown-`--plugin` issue. Keep backup creation ahead
   of the vault write, and keep a backup failure from writing the vault file.
5. Extract `render`, including the moved `SyncState` and `VaultState` presentation
   impls, table widths, and `success_json`. Preserve the expect message
   `"serialize plugins result"`.
6. Move Clap construction, dispatch, and exit-status selection into `cli`. Leave
   `plugins.rs` as the facade with the two re-exports and `#[cfg(test)] mod tests;`.
   Move the whole test body to `tests.rs`. Keep one `TEMP_COUNTER`.
7. Format, compile, and fix only relocation, import, and visibility errors. Run the
   focused checks and `just all`. Review the diff for changes to string literals,
   control flow, serde attributes, exit codes, or assertions. Record the final file map,
   test discovery, commands, results, and any baseline failure as described below.

## Behavior that must survive the move

Contracts in `docs/plugins.md` stay true without editing that file.

- CLI shape: `bob plugins` with no subcommand runs list and accepts list's top-level
  options, including `bob plugins -f json`. Subcommands remain `list` and `sync`.
  Unknown subcommands print `bob plugins: unknown subcommand: ...` to stderr and
  return 2. Clap parse errors still use Clap's exit code and printer. Help text, long
  abouts, examples, shorts, and value names stay as they are. `-f` is format, `-F` is
  force, `-n` is no-pull, `-p` is plugin, `-B` is backup-dir, `-b` is bob-dir, `-r` is
  repo, `-d` is dry-run.
- Path resolution stays `--flag`, then the existing env helper, with tilde expansion:
  repo via `bob_env::plugins_dir`, vault via `bob_env::bob_dir`, backup root via
  `bob_env::plugin_backups_dir`. Sync still appends
  `current_datetime().format("%Y%m%d-%H%M%S")` under the backup root. The default
  helpers themselves stay in `env`.
- Refresh: unless `--no-pull`, both list and sync pull before analysis. A missing `git`
  or a path that is not a worktree skips silently. Pull failures warn on stderr and
  continue with the existing checkout. Successful "Already up to date." and "Already
  up-to-date." stay silent. Other summaries, including Fast-forward with the current
  `file changed` preference, stay on stderr. Pull output never goes to stdout.
  `GIT_TERMINAL_PROMPT=0` stays set on the pull command.
- Discovery: one row per UTF-8 plugin directory under `<repo>/plugins/`, sorted by id.
  Manifest id, version, and description come from `manifest.json`. An empty id uses the
  folder name. A missing or unparseable manifest records `{folder}: {error}`, skips that
  folder, and still lists the others. An unreadable `plugins/` directory is an issue and
  yields no rows. `community-plugins.json` missing, unreadable, or invalid means nothing
  is enabled, not an error. Vault state is `enabled` when the id is listed, `disabled`
  when the folder exists but the id is absent, and `not installed` when the folder is
  absent.
- Sync state byte-compares only managed files that exist in the repo. `synced` means
  each of those files is present and identical in the vault. `drift` means the vault
  folder exists but a managed file is missing or different. `missing` means the vault
  folder does not exist. `data.json` is never read for this comparison.
- List exit status: drift and not-installed are success. Exit 1 only when `issues` is
  non-empty. Table mode prints the table to stdout and each issue to stderr as
  `bob plugins: {issue}`. JSON success is `success_json` of `PluginsReport::result`.
  JSON failure is `{"ok": false, "error": "<issues joined by '; '>"}` and nothing else.
  Header, footer, glyphs, separator, and description truncation stay with the current
  `Styler`, `display_width`, `pad_right`, `terminal_width`, and `truncate` calls.
- Sync copies managed repo files into `<bob-dir>/.obsidian/plugins/<id>/` and never
  opens `data.json` or any other runtime file. Absent managed files are skipped.
  `--plugin` matches the resolved id. An unknown id adds `plugin not found in repo: ...`
  and copies nothing. Results are sorted by id.
- File actions stay `Created`, `Updated`, `Forced`, `Unchanged`, `SkippedDirty`, and
  `Failed`. Identical bytes are `Unchanged` and do not consult Git. A different existing
  file consults `git status --porcelain` in the vault; dirty without `--force` is
  `SkippedDirty` and writes nothing. Dirty with `--force` is `Forced`. A clean
  difference is `Updated`. A missing vault file is `Created`. Overwrites (`Updated` and
  `Forced`) record a backup path first. Dry-run leaves `written: false` and creates no
  directory. A real run creates the backup parent, copies the vault file, and only then
  creates the vault parent and writes repo bytes. Backup or write failure becomes
  `Failed`, appends the current issue text, and does not overwrite the vault file after
  a failed backup.
- Diffs: non-UTF-8, or UTF-8 at least `MINIFIED_BYTE_THRESHOLD` bytes with at most
  `MINIFIED_LINE_THRESHOLD` lines on either side, is `FileDiff::Binary`. Other text uses
  `similar` unified diffs with context 3, body limit 60, and the current `hidden` count.
  New files use `byte_line_count` and the byte length. Rendering keeps the current
  indent, truncation, colors, `pluralize`, `format_bytes`, and the backup arrow line.
  Dry-run says `would copy` and `would back up to`. The summary line keeps `copied`
  versus `to copy`, skipped, unchanged, error count, and the backup footer rules
  (`dry_run && has_backup_paths`, or a real run with `has_written_backups`).
- Sync exit status: a dirty skip is success. Exit 1 only when `issues` is non-empty.
  Issues go to stderr with the `bob plugins: ` prefix. The report itself goes to stdout.

## Verification and handling unrelated failures

Run these on the clean implementation baseline, and again after extraction. Save output
in disposable logs or bead notes. Do not commit generated logs. Each focused filter must
execute a nonzero set of relevant tests.

```sh
cargo test --lib plugins:: -- --list
cargo test --test cli plugins:: -- --list
cargo test --test cli completion -- --list
cargo test --lib plugins::
cargo test --test cli plugins::
cargo test plugins
cargo test --test cli completion
```

Expect 17 original unit tests under `native::plugins::tests::` and 13 integration tests
under `plugins::` in the `cli` binary:

- Unit: `sync_state_detects_synced_drift_and_missing`,
  `vault_state_reads_enabled_disabled_and_not_installed`,
  `scan_reports_states_and_counts`, `unreadable_repo_is_an_error`,
  `pull_repo_skips_non_git_directory`, `pull_repo_fast_forwards_from_remote`,
  `json_shape_is_stable`, `truncate_adds_ellipsis_only_when_needed`,
  `sync_creates_updates_and_leaves_unchanged`, `sync_dry_run_reports_without_writing`,
  `sync_only_filters_to_a_single_plugin`, `sync_unknown_plugin_is_an_error`,
  `sync_preserves_runtime_data_json`, `sync_refuses_then_forces_a_dirty_vault_file`,
  `sync_reports_text_diff_for_changed_files`,
  `sync_summarizes_binary_and_minified_diffs`, `backup_failure_aborts_overwrite`.
- Integration: `plugins_list_renders_table_and_summary`,
  `plugins_default_subcommand_runs_list`, `plugins_list_json_is_machine_readable`,
  `plugins_list_pulls_repo_before_analysis`,
  `plugins_list_no_pull_uses_existing_checkout`,
  `plugins_list_json_stdout_stays_machine_readable_after_pull`,
  `plugins_list_unreadable_repo_reports_error`,
  `plugins_sync_pulls_repo_before_copying`,
  `plugins_sync_dry_run_reports_without_writing`,
  `plugins_sync_backs_up_overwritten_file`,
  `plugins_sync_single_plugin_copies_only_that_plugin`,
  `plugins_sync_preserves_runtime_data_json`,
  `plugins_sync_refuses_dirty_vault_file_then_forces`.

With a single `tests.rs` module, the `native::plugins::tests::*` paths should stay
stable. `cargo test plugins` also selects other `cli` tests whose paths contain
`plugins`, including help-option coverage and
`completion::vault::plugins_come_from_the_repo_checkout`. Those must still pass. Compare
the `--lib plugins::` and `--test cli plugins::` inventories separately so a broader
filter does not hide a lost unit test.

Use the existing temporary repositories, local Git remotes, and isolated backup roots.
Do not use Bryan's real vault, the real bob-plugins checkout, or a network remote.

After the final edit, run:

```sh
cargo fmt
just all
git diff --check
```

`just all` must actually run `cargo fmt --check`,
`cargo clippy --all-targets --all-features`, and the full `cargo test`. Use
`/sase_monitor` for any command that needs a long-running handoff, and wait for that
handoff command itself to exit before ending its turn. Report the checks that actually
ran. This code has no target-specific `cfg` gates. On Linux the Git and filesystem paths
are compiled and tested by the suites above.

Count physical lines after formatting. Every listed file must exist, `plugins/mod.rs`
must not exist, and every file must be at most 1500 lines:

```sh
python3 - <<'PY'
from pathlib import Path

names = (
    'capture_complete', 'capture_task_toggle', 'task_status_hooks_write',
    'capture_clip', 'plugins',
)
root = Path('src/native')
for name in names:
    paths = [root / f'{name}.rs', *(root / name).rglob('*.rs')]
    files = sorted(path for path in paths if path.is_file())
    if not files:
        raise SystemExit(f'MISSING COMPONENT: {name}')
    for path in files:
        count = len(path.read_bytes().splitlines())
        print(f'{count:5} {path}')
        if count > 1500:
            raise SystemExit(f'OVER LIMIT: {path}')
if (root / 'plugins/mod.rs').exists():
    raise SystemExit('competing plugins/mod.rs root')
if not (root / 'plugins.rs').is_file():
    raise SystemExit('missing plugins.rs facade')
PY
```

The four earlier components must still be present and under the ceiling. A layout that
only shrinks `plugins.rs` by leaving a sibling over 1500 does not complete the phase.

Phase `bob-cli-3s.3` recorded a pre-existing process-global `BOB_DAY_FILE` race in
`capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
(notes on that bead: it failed under default parallel `cargo test`, and passed in
isolation and with `--test-threads=4`). The plugin split does not touch that env
handling. This is prior evidence, not permission to assume a new failure has the same
cause. If a check fails, keep the exact diagnostic and reproduce it on the unchanged
base without disturbing the implementation. A failure that reproduces identically on the
clean base does not keep this phase open. Append
`PROPOSED FOLLOW-UP: <summary — reproduction and evidence>` with
`sase bead note bob-cli-3s.5`, citing any existing tracking bead or the phase-3 notes.
Record unresolved flakes and environment limits precisely. Fix failures caused by this
refactor. Do not add unrelated lint or lock cleanup, and do not create task beads.

## Phase completion

After the work and verification above, run:

```sh
sase bead epic-symbols bob-cli-3s.5
```

Resolve every remaining symbol, or re-key its Justfile line to a still-open bead (the
parent epic or a later phase) when that outstanding work belongs there. Do not close
while any entry remains keyed to this phase. Record the before/after file map and
counts, discovered test preservation, commands and results, limitations, and follow-ups
with `sase bead note bob-cli-3s.5` as needed, then run:

```sh
sase bead close bob-cli-3s.5 --note "<final file counts, preserved tests, cumulative five-component audit, checks and limitations actually verified>"
```

Never hand-set this bead's status. Close only `bob-cli-3s.5`. Do not close the parent
`bob-cli-3s` or any ancestor plan bead. A phase description that mentions epic
completion is evidence for the epic land agent, not permission for this worker to close
an ancestor. Do not invoke a manual Git commit skill unless explicitly instructed.
Follow the required `/sase_final` declaration for any normal response that ends the
coding turn.
