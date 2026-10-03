---
tier: tale
title: Split guarded task status writes into focused modules
goal: "Split src/native/task_status_hooks_write.rs into a small facade and focused child
  modules so every resulting Rust file is at most 1500 lines, while guarded-write
  sequencing, platform behavior, the existing entry points, and current test coverage
  stay intact.

  "
size: medium
proposed_by: bbugyi200.athena.bob-cli-3s.3
bead: bob-cli-3s.3
status: done
---

- **PARENT:**
  [202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-3s.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/bob-cli-3s.3.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-3s.3](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-3s.3.md)
- **COMMITS:**
  - [b4a5022](https://github.com/bobs-org/bob-cli/commit/b4a5022515e08c764299614c1c473a5199867608)
    — refactor(native): split task_status_hooks_write into focused modules

# Plan: Split guarded task status writes into focused modules

## Why this is a tale

Phase `bob-cli-3s.3` (`split-task-status-hooks-write` on epic `bob-cli-3s`) owns one
assigned file. The dependency graph below is acyclic and the move is
behavior-preserving, so one coding agent can implement it from this plan. Another epic
would only re-split a phase the parent epic already isolated. `medium` fits a
multi-module visibility-sensitive refactor that is still bounded by one file and the
checks below. Do not set an explicit model.

Measured on this tree at `c9a6f1b`, before any edit:
`src/native/task_status_hooks_write.rs` is 2398 physical lines. The inline test section
starts at the `#[cfg(test)]` on line 1530, so the production body is 1529 lines (over
the 1500 ceiling) and the tests are 869 lines (under it). Extracting tests alone would
leave an oversized production file. Phases 1 and 2 already split their files into
`<name>.rs` plus `src/native/<name>/`. Use that same shape. Leave `capture_complete` and
`capture_task_toggle` as they are, and do not start the later phases (`capture_clip`,
`plugins`).

## Layout

Keep `src/native.rs`'s `mod task_status_hooks_write;`. Replace the body of
`src/native/task_status_hooks_write.rs` with a facade and put children in
`src/native/task_status_hooks_write/`. Do not also add `task_status_hooks_write/mod.rs`.

Move code. Do not rewrite the apply sequence, recovery schema, staging names, or error
strings. Do not drop comments or reformat beyond `cargo fmt`. Adjust `use` paths and
visibility keywords only. Import helper names into each child so call expressions stay
the same.

Dependency direction, which must not cycle:

`model` ← `snapshot` ← `preflight` ← `apply`

`model` ← `staging` ← `recovery` ← `apply`

`apply` also calls `staging` directly. `snapshot` does not call `preflight`, `recovery`,
or `staging`. `preflight` does not call `apply`, `recovery`, or `staging`. `recovery`
does not call `snapshot`, `preflight`, or `apply`. `staging` calls only `model`.

| File                         | Owns                                                                                                                 | Approx. lines |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------- |
| `task_status_hooks_write.rs` | Module docs, the existing clippy allow, `mod` declarations, explicit re-exports, and the private test-facing imports | under 120     |
| `model.rs`                   | Plans, snapshots, sessions, errors, outcomes, the run id, and the two in-flight records                              | about 430     |
| `snapshot.rs`                | Capture, recapture, identity, and stable reads                                                                       | about 220     |
| `preflight.rs`               | Quiet period, read-set checks, and per-file replacement checks                                                       | about 240     |
| `apply.rs`                   | Maintenance lock and the ordered `apply_plan` transaction                                                            | about 170     |
| `recovery.rs`                | Manifests, original-byte retention, hashing, updates, and pruning                                                    | about 270     |
| `staging.rs`                 | Exclusive temporaries, permissions, metadata and xattrs, cleanup, and private file helpers                           | about 320     |
| `tests.rs`                   | The existing inline tests, unchanged assertions                                                                      | about 890     |

These boundaries follow the epic's recommended buckets, with four dependency-driven
departures:

- Content and vault hashing (`sha256_hex`, `vault_hash`) live in `recovery.rs`. Snapshot
  identity is device and inode metadata. The hashes are recovery-manifest fields only.
  Putting them in `snapshot.rs` would add a `recovery` → `snapshot` edge for two
  helpers.
- `StagedWrite` and `AppliedWrite` live in `model.rs`. Apply, preflight, and recovery
  all pass them. Putting `AppliedWrite` in `apply.rs` would cycle `apply` with
  `preflight` and `recovery`.
- `new_run_id`, `RUN_COUNTER`, and `TOOL` live in `model.rs` because
  `ApplySession::production` uses them. Putting them in `apply.rs` or `recovery.rs`
  would cycle that module back into `model`. `SCHEMA_VERSION` stays in `recovery.rs`.
- `ensure_private_dir`, `write_private_file`, and `sync_dir` live in `staging.rs`.
  Recovery writes manifests through them. `recovery` depends on `staging`; `staging`
  does not depend on `recovery`.

Keep one `tests.rs`. The 869-line suite shares `TempDir`, `fixture`, `plan_for`,
`make_session`, and `live_scan_session`, and it sits well under 1500. Splitting it would
publish those fixtures for no coverage gain.

### `model.rs`

Move these types, constants, and helpers:

- `TOOL`, `QUIET_PERIOD`, `RETENTION`, `RUN_COUNTER`
- `ReasonCode`, `InputKind`, `FileIdentity`, `InputState`, `InputSnapshot`
- `PlannedWrite`, `WritePlan`, `ApplySession`, `CaptureError`, `ApplyError`,
  `ApplyOutcome`
- `StagedWrite`, `AppliedWrite`
- `new_run_id`, `remaining_outputs`

Leave every `ApplyError` and `CaptureError` constructor on its type. Leave
`FileIdentity`'s `PartialEq` ignoring `mtime`. Leave `InputSnapshot::matches` on
`InputSnapshot`.

### `snapshot.rs`

Move `capture_required`, `capture_optional`, `planned_write`, `snapshot_for_path`,
`recapture`, `capture_present`, `reject_non_regular`, `read_regular_file`,
`file_identity`, `metadata_fingerprint`, and the Linux-only `O_NOFOLLOW` constant.

### `preflight.rs`

Move `wait_quiet_period`, `preflight`, `preflight_for_replacement`, and
`check_unwritten_output`.

### `apply.rs`

Move `acquire_maintenance_lock` and `apply_plan`. `apply_plan` stays the single ordered
transaction. Preserve this order and the existing warning text:

1. Empty `outputs` returns `ApplyOutcome::NoOp` and creates nothing.
2. `wait_quiet_period`.
3. `before_preflight` callback, when set.
4. `preflight` with an empty applied list.
5. `create_recovery`.
6. `stage_outputs`.
7. `after_staging` callback, when set.
8. `preflight` again. On failure, `cleanup_temps` and attach `recovery_directory` when
   the error does not already have one.
9. For each staged file, in order: `before_replace`, `preflight_for_replacement`,
   `fs::rename` of that temp onto the destination, then `update_manifest` with outcome
   `"partial"`. A manifest-update failure here is a warning, not a returned error.
10. After the loop, `update_manifest` with outcome `"applied"` (warning on failure),
    then `prune_completed` (warning on failure).
11. Return `ApplyOutcome::Applied`.

On a replacement-check failure, clean up the remaining temps, update the manifest to
`"partial"`, and return `ReasonCode::PartialApply` with the recovery directory. On a
rename failure, clean up the remaining temps, remove the failed temp, update the
manifest to `"partial"`, and return `ApplyError::io`. Do not reorder callbacks, fold the
two preflight passes together, or change the `bob {tool}: warning:` lines.

### `recovery.rs`

Move `SCHEMA_VERSION`, `RecoveryManifest`, `RecoveryNote`, `sha256_hex`, `vault_hash`,
`format_system_time`, `unix_secs`, `create_recovery`, `recovery_manifest`,
`write_manifest`, `update_manifest`, and `prune_completed`.

Keep manifest field names, `schema_version: 1`, the `0000.original` / `0000.proposed`
file names, the `manifest.json` name, and the
`$XDG_STATE_HOME/bob-cli/<tool>/<vault-hash>/<run-id>` directory shape. Pruning still
removes only completed (`outcome == "applied"`) records for the same tool, older than
`retention`.

### `staging.rs`

Move `stage_outputs`, `stage_one`, `staged_temp_name`, `create_exclusive_temp`,
`copy_copied_metadata`, `cleanup_temps`, `ensure_private_dir`, `write_private_file`,
`sync_dir`, and both `copy_xattrs` bodies (the Linux `llistxattr` / `lgetxattr` /
`lsetxattr` implementation and the non-Linux stub).

Keep the `.bob-tsh.{run_id}.{index}.{nonce}.tmp` temp pattern, the exclusive
`create_new` open, mode `0o600` while writing, the restored destination mode, private
directory mode `0o700`, and the xattr allow-list (`user.*`, `trusted.*`,
`system.posix_acl_access`, `system.posix_acl_default`) with the `ENOTSUP` / `ENOSYS`
skip.

## Visibility

Keep the facade's module docs and
`#![allow(clippy::result_large_err, clippy::type_complexity)]`. That allow covers this
module and its children. Do not change `ApplyError` or the `Result` types to silence the
lint, and do not add a broader allow. Child modules stay private (`mod`, not `pub mod`).

Re-export every existing `pub(crate)` item from the facade with explicit
`pub(crate) use` paths, not globs. Callers must keep compiling with no edits. If a path
breaks, add the missing re-export. Do not edit the caller.

Re-export:

- From `model`: `QUIET_PERIOD`, `RETENTION`, `ReasonCode`, `InputKind`, `FileIdentity`,
  `InputState`, `InputSnapshot`, `PlannedWrite`, `WritePlan`, `ApplySession`,
  `CaptureError`, `ApplyError`, `ApplyOutcome`, `new_run_id`.
- From `snapshot`: `capture_required`, `capture_optional`, `planned_write`,
  `snapshot_for_path`.
- From `apply`: `acquire_maintenance_lock`, `apply_plan`.

Known external users of that surface, which must stay on `task_status_hooks_write::...`,
are `task_status_hooks` (the facade import is in `task_status_hooks/mod.rs`, and its
children use the names through `use super::*`) and `randomize`. `randomize` sets
`session.tool = "randomize"` on an `ApplySession::production` value so recovery lands
under the randomize root. `task_status_hooks` uses `acquire_maintenance_lock`,
`apply_plan`, `capture_optional`, `capture_required`, `new_run_id`, `planned_write`,
`snapshot_for_path`, and the error, session, snapshot, and plan types. `ReasonCode` is
part of the same public surface because `randomize` matches its variants.

Keep field visibility as it is, except the cross-module cases below. `ApplySession`,
`ApplyError`, `WritePlan`, `PlannedWrite`, `InputSnapshot`, and `FileIdentity` fields
that are `pub` today stay `pub`.

Helpers and fields that are private today and must cross a child boundary become
`pub(super)`, not `pub(crate)`:

- `model`: `TOOL`, `remaining_outputs`, `StagedWrite` and its fields, `AppliedWrite` and
  its fields, `InputSnapshot::matches`, `ApplySession::now`, `CaptureError::io`, and
  `ApplyError::{lock_contention, lock_io, capture, io, vault_changed, quiet_period}`.
  `InputState::Present`'s `identity` and `bytes` fields, and `CaptureError`'s
  `Unsupported` and `Io` fields, become `pub(super)` so `snapshot` and `preflight` can
  build and read them. Tuple variants stay as they are.
- `snapshot`: `recapture`.
- `preflight`: `wait_quiet_period`, `preflight`, `preflight_for_replacement`.
- `recovery`: `RecoveryManifest`, `RecoveryNote`, `sha256_hex`, `vault_hash`,
  `unix_secs`, `create_recovery`, `update_manifest`, `prune_completed`. On
  `RecoveryManifest`, `tool`, `outcome`, and `notes` become `pub(super)`. On
  `RecoveryNote`, `original_hash`, `proposed_hash`, and `state` become `pub(super)`. The
  other manifest fields stay private to `recovery.rs`.
- `staging`: `stage_outputs`, `cleanup_temps`, `ensure_private_dir`,
  `write_private_file`, `sync_dir`.

Leave these private to their new file: `capture_present`, `reject_non_regular`,
`read_regular_file`, `file_identity`, `metadata_fingerprint`, `O_NOFOLLOW`,
`check_unwritten_output`, `recovery_manifest`, `write_manifest`, `format_system_time`,
`SCHEMA_VERSION`, `stage_one`, `staged_temp_name`, `create_exclusive_temp`,
`copy_copied_metadata`, and both `copy_xattrs` functions.

The tests module reaches a few private helpers through the facade. Add private (not
`pub`) facade imports so `use super::*` in `tests.rs` still resolves them:

- `model::TOOL`
- `recovery::{prune_completed, sha256_hex, unix_secs, vault_hash, RecoveryManifest}`

`RUN_COUNTER` stays a single static in `model.rs`. `new_run_id` is the only increment.

## Tests

The inline module has 20 `#[test]` functions:

- `unchanged_read_set_applies_and_records_recovery_bytes`
- `noop_creates_no_recovery_or_staging`
- `equal_length_change_with_restored_mtime_prevents_write`
- `fresh_attempt_after_vault_changed_replans_from_intervening_edit`
- `replacement_inode_prevents_write`
- `symlink_substitution_prevents_write` (`#[cfg(unix)]`)
- `changed_tasks_settings_prevent_write`
- `changed_previous_daily_prevents_write`
- `new_or_deleted_scan_candidate_invalidates_plan`
- `edit_between_staging_and_revalidate_survives`
- `edit_before_later_replacement_reports_partial`
- `exclusive_temp_and_mode_and_foreign_temps` (`#[cfg(unix)]`)
- `staging_failure_preserves_notes_and_foreign_temps`
- `retention_keeps_incomplete_and_prunes_old_completed`
- `quiet_period_skips_wait_for_stable_files_and_status_only`
- `quiet_period_waits_once_then_defers_if_changed`
- `future_mtime_defers_without_sleeping`
- `multiply_linked_output_is_rejected` (`#[cfg(unix)]`)
- `tool_scopes_recovery_root_manifest_and_pruning`
- `live_rescan_sees_new_vault_file`

Move the body of `mod tests` into `src/native/task_status_hooks_write/tests.rs` and
declare `#[cfg(test)] mod tests;` on the facade. Do not wrap `tests.rs` in another
`mod tests`. Keep `use super::*;`. The facade will no longer import `std`, so add the
std imports the test body uses (`fs`, `File`, `io`, `OsStr`, `Path`, `PathBuf`, `Rc`,
`RefCell`, `Cell`, `AtomicUsize`, `Ordering`, `Duration`, `SystemTime`, `UNIX_EPOCH`)
and keep the existing unix `PermissionsExt` import under `#[cfg(unix)]`. Change no
assertion, fixture, helper body, test name, or `cfg` gate. The module path
`native::task_status_hooks_write::tests` stays, so `cargo test task_status_hooks_write`
still selects this suite.

Integration coverage stays where it is under `tests/cli/task_status_hooks/` and
`tests/randomize.rs`. Do not rewrite expected CLI or JSON output.

## Behavior to leave alone

This is a structural split of the guarded writer. Preserve:

- the shared maintenance lock, including contended versus I/O failure reasons
- content and identity checks, symlink and non-regular rejection, and hardlink rejection
  (`nlink != 1`) both at plan time and again before replacement
- `FileIdentity` equality ignoring `mtime`
- Linux `O_NOFOLLOW` on the read path, and the non-Linux identity fallback that returns
  `CaptureError::Unsupported`
- private permissions, xattr copy gates, and the non-Linux `copy_xattrs` stub
- quiet-period wait capped at `QUIET_PERIOD` (2 seconds), skipped for status-only
  writes, and `ReasonCode::QuietPeriod` for a future or unreadable mtime
- recovery schema, path conventions, partial-apply reporting, retention, and the
  tool-specific root `randomize` selects by setting `ApplySession.tool`
- no-op behavior when `outputs` is empty: no recovery directory and no staging
- callback timing (`before_preflight`, `after_staging`, `before_replace`,
  `fail_staging`) and the single `RUN_COUNTER`

Move each `#[cfg(target_os = "linux")]`, `#[cfg(unix)]`, and `#[cfg(not(...))]` block
with the item it gates. Do not merge the two `copy_xattrs` bodies or the unix and
non-unix arms. This checkout is Linux, so unix and Linux tests compile and run here.
Keep the other gates by inspection. Do not claim macOS or non-unix coverage from a run
on this machine.

## Validation

Before editing, record baseline discovery:

```bash
cargo test task_status_hooks_write -- --list
cargo test --test cli task_status_hooks -- --list
cargo test --test randomize -- --list
```

After the move, `cargo fmt`, then confirm the same filters still execute the same tests
(module paths may show `task_status_hooks_write::tests::...`; a lost test is a failure).
The 20 unit tests above must all still be listed. Run:

```bash
cargo test task_status_hooks_write
cargo test --test cli task_status_hooks
cargo test --test randomize
```

Then run `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
and `cargo test`). Use `/sase_monitor` for `just all` and any other cargo run that needs
a long handoff. Fix regressions this split causes. A failure that reproduces the same
way on the clean base tree does not keep `bob-cli-3s.3` open: record it as a
`PROPOSED FOLLOW-UP:` note and cite any task bead that already tracks it.

Count lines after formatting. Every `task_status_hooks_write` Rust file, including
tests, must be at most 1500, with the headroom in the table above. Also confirm the
completed `capture_complete` and `capture_task_toggle` files are still at most 1500.
`capture_clip.rs` and `plugins.rs` are still monolithic and out of scope.

```bash
python3 - <<'PY'
from pathlib import Path
root = Path('src/native')
for name in ('capture_complete', 'capture_task_toggle', 'task_status_hooks_write'):
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
sase bead note bob-cli-3s.3 'PROPOSED FOLLOW-UP: <one-line summary — detail>'
```

Before closing, run `sase bead epic-symbols bob-cli-3s.3`. The planning tree's
`justfile` has no `--epic-symbol` entries. If any appear and are keyed to this phase,
resolve each symbol or re-key that line to a still-open bead (the parent epic
`bob-cli-3s` or a later phase). `sase bead close` refuses while leftovers remain.

Close only `bob-cli-3s.3`. Leave `bob-cli-3s` and every ancestor open. The close note
must include the final file map, line counts, the test commands that ran, and their
results.

```bash
sase bead close bob-cli-3s.3 --note "<what you verified>"
```
