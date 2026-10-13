---
tier: tale
title: Fix bob-cli build warnings
goal:
  cargo install and cargo build --all-targets emit zero bob-cli warnings, with no
  behavior change.
size: small
decisions:
  guard:
    ask:
      Should `just lint` and `just check` fail when bob-cli emits rustc unused or
      private-interface warnings?
    choices:
      gate:
        Both clippy runs deny rustc unused/private-interface lints; clippy style lints
        stay warnings
      none: Only fix today's warnings; nothing stops new ones from piling up silently
    default: gate
    why:
      Nothing fails on these warnings today, which is how 40-plus of them piled up
      unnoticed
    answer: gate
proposed_by: bbugyi200.apollo.79
decided_by: auto
create_time: 2026-10-13 08:07:09
status: wip
---

# Plan: Fix bob-cli build warnings

## Context

`cargo install --path . --locked` (what `just install` runs) prints 36 rustc warnings
for the `bob-cli` lib plus a yanked-crate warning. Building the test targets
(`cargo build --all-targets`) adds 7 more lib-test warnings and 2 integration-test
warnings. All of them reproduce on Linux at HEAD with the same set the user saw on
macOS. The macOS-only `cfg(target_os)` code (`capture_clip/clipboard.rs`,
`task_status_hooks_write/{snapshot,staging}.rs`) touches none of the items below.

Root causes:

1. **Over-wide re-exports.** `pub(crate) use` lists re-export items that are only used
   by their own module path.
2. **Test-only helpers compiled into release.** Convenience wrappers and accessors that
   only tests call are not gated by `#[cfg(test)]`.
3. **Leftovers from epics.** Fields, variants, and functions that later phases stopped
   reading or calling.
4. **Visibility mismatch.** Three `pub(crate)` output functions take `pub(super)` types.
5. **Yanked lockfile entry.** `chacha20 0.10.0` (via `lopdf → rand 0.10.1`) is yanked,
   and `0.10.2` is a compatible bump.

Nothing fails on these warnings. `just lint` runs clippy without `-D`, and its roughly
200 clippy style warnings hide the rustc ones.

## Policy (apply consistently)

- **Truly dead** (no caller, including tests): delete it.
- **Only tests use it**: gate it with `#[cfg(test)]`. This is the existing idiom; see
  `RefIndex::row_by_path` and `GkeepConfig::for_tests`. Do not move code into test
  modules.
- **Never** silence a warning with `#[allow(dead_code)]` / `#[expect(...)]`, and never
  prefix a field with `_` to hide it.
- Behavior must not change. Every deletion below is a value that is never read or code
  that is never reached.
- When a doc comment points at an item that becomes `#[cfg(test)]` (for example "Same as
  [`classify`]"), move the substantive doc onto the production function. Then non-test
  rustdoc has no link into a test-only item.

Line numbers are from HEAD `576a3e0` and are approximate. Let the compiler confirm each
site.

## Steps

### 1. Lockfile: un-yank chacha20

- Run `cargo update -p chacha20 --precise 0.10.2`. `git diff Cargo.lock` must show only
  chacha20's version and checksum changing.

### 2. Unused re-exports

- `src/native/highlights_ref/mod.rs` (~L154-157): drop `RegionTask` from
  `pub(crate) use region::{...}`.
- `src/native/ref_library/mod.rs` (L52): make it
  `pub(crate) use status::decide_status;`. The `status::TrackerHit` path at ~L671
  already resolves through `status.rs`'s own `use`.
- `src/native/ref_tasks/mod.rs`: drop these from the re-export lists:
  `EditedReadingTask`, `ManagedEmbed`, `InsertedRefTask`, `OrphanRefTask`,
  `RefFollowUp`, `REF_BLOCK_ID_MAX_LEN`, `stamp_close_date_any_id`, and
  `managed_region_line_range`. The items themselves stay because their modules use them.
  - `src/native/ref_tasks/tests.rs` reaches `stamp_close_date_any_id` through
    `use super::*;` (~L576-582). Add `use super::line::stamp_close_date_any_id;` there.
- `src/native/ref_library/status.rs` (~L307-310): delete the unused
  `#[cfg(test)] pub(crate) use crate::native::ref_tasks::{find_trackers, managed_region_line_range, parse_tracker_line};`
  block. It warns in the lib-test build, and no status test uses it.

### 3. Private interfaces (`ref_library`)

- `src/native/ref_library/output.rs`: narrow `print_find_json` (~L61),
  `render_find_human` (~L250), and `render_find_markdown` (~L492) from `pub(crate)` to
  `pub(super)`. Their only caller is `ref_library/cli.rs`. Do not widen
  `find::FindResult` / `FindSummary` instead, because `find` is a private module.

### 4. gkeep

- `src/native/gkeep/imports.rs` (~L367-379): delete `verified_ids`. It has no callers;
  `pull.rs` builds its own local `verified_ids` sets.
- `src/native/gkeep/ledger.rs`:
  - `Journal::ref_parent` (~L363-374): `#[cfg(test)]`. Only the ledger tests (~L834-844)
    read it. The journal still writes `parent` for audit and old-line compatibility.
  - `read_target_tasks` (~L427-440): `#[cfg(test)]`. Move its full doc comment onto
    `read_target_tasks_with_imports`, and give the wrapper a one-line "test convenience:
    no import history" doc.
- `src/native/gkeep/plan.rs`:
  - `classify` (~L115-144): `#[cfg(test)]`, with the same doc move onto
    `classify_with_imports`.
  - `classify_one` (~L211-219): delete. Nothing calls it, including tests.
- `src/native/gkeep/migrate.rs`: delete `FilePlan.has_final_newline` and
  `FilePlan.skips` (~L110, L116-117), the `has_final_newline` local (~L572), and both
  initializers (~L651, L655). `apply_cleanup` already keeps the final newline through
  `split`/`join`, and `skips` is never populated.

### 5. highlights_ref

- `ingest.rs` (~L193-202): delete `fallback_note_for` (no callers). Fold its doc (worker
  retry of a `stuck/` fallback; identical to `IngestError::fallback_note`) into
  `fallback_note_for_with_listen`'s doc.
- `listen.rs`:
  - `ListenCommand::display` (~L269): `#[cfg(test)]`. Only the `resolve` tests use it
    (`.map(ListenCommand::display)`, ~L1625/L1637).
  - In test `run_listen_has_no_fallback_when_sase_missing` (~L1548-1551), replace the
    `{ let mut out = ...; out }` block with `bin.as_os_str().to_os_string()`. This fixes
    the `unused_mut` warning.
- `model.rs` (~L483-492): delete `ScanContext::for_tests`. It is already `#[cfg(test)]`
  and has no callers.
- `sync.rs`: delete `V2ReadingOutcome.refreshed` (~L1703) and its initializers (~L1719,
  L1737, L1792, L1810). The local `refreshed` binding (~L1728) still feeds `embed` and
  `residence`.

### 6. ref_library::migrate_tasks (`plan.rs`)

- Delete `PlannedTask.tracker_start` / `tracker_end` (~L57-59) and their initializers
  (~L398-399). If a local feeding them becomes unused, remove it too.
- Delete `UnmappedRow.stem` (~L70) and its three initializers (~L292, L317, L331). This
  does not touch `PlannedTask.stem` or `report.rs`'s own `stem`, which are both read.
- Delete `MigrationPlan::is_empty` (~L129-131). It has no callers.

### 7. ref_tasks

- `select.rs`: delete the `Selected::V1(TrackerHit)` variant (~L34). `select_for_ref`
  never builds it; v1 hits travel in `status_hits` with `task = None`. Rewrite the stale
  Rule 4 comment (~L176-181) that says the caller maps hits to `Selected::V1`. Keep
  `Selected` as a one-variant enum so call sites stay unchanged; do not collapse it into
  `Option<LocatedRefTask>`. Then fix the match sites:
  - `src/native/ref_library/mod.rs` (~L448): `Some(...Selected::V1(_)) | None =>`
    becomes `None =>`. Keep the v1 comment about `has_usable_tracker`.
  - `src/native/highlights_ref/reading_plan.rs` (~L126-128): remove the
    `Some(Selected::V1(_)) => BirthTask::Refused` arm and its "Unreachable today"
    comment.
  - `src/native/highlights_ref/reading_plan.rs` (~L665): after the change,
    `let Selected::V2(located) = selected else { ... }` is irrefutable and would raise
    `irrefutable_let_patterns`. Make it `let Selected::V2(located) = selected;` and
    delete the unreachable `else` block.
- `walk.rs`: delete `RefTaskIndex.warnings` (~L40) with every write in `build` (~L54-78,
  L475, plus any `build_empty_selection` initializer) and the `warnings()` accessor
  (~L501-503). No production caller ever reads it.
  - When the ref dir is outside the vault, `build` keeps returning the empty index.
  - Discard the second value from `vault_note_paths`.
  - Change the "problems become `warnings`" doc to say problems yield an empty or
    partial index.
  - In `ref_tasks/tests.rs` (~L463-476), rename
    `ref_dir_outside_vault_yields_warning_and_empty_index` to
    `ref_dir_outside_vault_yields_empty_index` and drop the warnings assertion.

### 8. task_complete: gate the recovery parity oracle

- `src/native/task_complete/mod.rs` (L10): `#[cfg(test)] pub(crate) mod recovery;`.
  Since `eb6fa0d`, `successors.rs` is the production engine. Accepted plans keep
  `recovery.rs` only as a test-parity oracle: `plan:202610/prj_task_dep_links.md` says
  "`recovery.rs` (test parity)" and "Mirror both rules in `recovery.rs`", and
  `plan:202610/project_pull_in_line_model.md` says "`recovery.rs` has no production
  caller". Its only users are `task_complete/tests/recovery_tests.rs` and the
  `capture/dependencies.rs` test module, and both are already `cfg(test)`. Gating the
  module clears all four recovery warnings (`RecoveredDependent`, `DependentRecovery`,
  `recover_blocked_dependents`, `line_start_offset`).
- Update `recovery.rs`'s `//!` doc to state that it is a test-only parity oracle kept in
  step with `successors.rs`. Update the `mod.rs` `//!` line that lists "the
  Blocked-dependent recovery" so it names the successor planner.

### 9. task_status_groups

- `parse.rs`: delete `DirectPiece::Other.byte_range` (~L296),
  `DirectPiece::Badges.byte_range` (~L299), and `TaskBlock.byte_range` (~L314), along
  with their initializers (~L370, L433, L504).
  - `Badges` becomes a unit variant `Badges`; update the patterns.
  - The local `byte_range` in the task-block builder still drives `raw` and `cursor`.
- `mod.rs` (~L25-33): delete the three `#[cfg(test)]` constants `MARKER_ACTIVE`,
  `MARKER_BLOCKED`, and `MARKER_CLOSED`. Nothing uses them. Keep
  `LEGACY_GROUP_MARKER_PREFIX`.

### 10. url_routing

- `src/native/url_routing/intent.rs` (~L191-204): delete `split_listen_suffix`. It has
  no callers, including tests.

### 11. Integration tests

- `tests/cli/capture/routing.rs` (~L128-133): the test above the PDG5 comment carries
  two `#[test]` attributes. Delete the first one so the comment sits directly above a
  single `#[test]`.
- `tests/cli/highlights/scan_integration.rs` (~L480): delete the unused
  `let note = vault.join("ref/missing.md");`.

### 12. Guard against regressions

> [!decision] guard = gate In `justfile`, define one variable, for example
> `rustc_gate := "-D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces"`,
> and append `-- {{rustc_gate}}` to the `cargo clippy --all-targets --all-features` line
> in both `lint` and `check`.
>
> - `-D unused` also promotes the group's allow-by-default members. On rustc 1.95,
>   `-D unused` turns `unused_macro_rules` into an error. The two `-A` flags placed
>   after it re-allow those members.
> - Flags after `--` reach only bob-cli's own targets, so dependency artifacts stay
>   cached and clippy's style lints stay warnings.
> - If the README or docs describe what `just lint` / `just check` run, add a short note
>   about the gate.
>
> With `guard = none`, skip this step.

Then run `cargo fmt`; rustfmt may reflow the shortened `use` lists.

## Verification

1. `cargo build 2>&1 | grep -E '^warning'` prints nothing.
2. `cargo build --all-targets 2>&1 | grep -E '^warning'` prints nothing.
3. `grep -A1 'name = "chacha20"' Cargo.lock` shows `0.10.2`, and `git diff --stat` shows
   that Cargo.lock changed by two lines.
4. `just check` passes: fmt, clippy (with the gate when `guard = gate`), and every test
   binary. If tests fail, compare against the base tree (for example with `git stash`)
   before treating a failure as a regression. Recent commits mention known pre-existing
   percent-listen failures.
5. When `guard = gate`, add a throwaway unused import, confirm `just lint` fails on it,
   then revert it.
6. Optional, slow (release build): `just install-smoke` repeats the user's
   `cargo install --locked` path. Expect no `warning:` lines, including the yanked
   chacha20 warning.

Out of scope: the roughly 200 existing clippy style warnings (`collapsible_if`,
`result_large_err`, `too_many_arguments`, and so on), which `cargo install` never shows.
