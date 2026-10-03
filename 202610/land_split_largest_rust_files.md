---
tier: tale
title: Finish landing bob-cli-3s by dropping unused facade re-exports
goal:
  The split facades for capture_task_toggle, task_status_hooks_write, and
  capture_complete re-export only what callers use, so the lib builds show no
  epic-introduced unused-import warnings or suppressions. Epic bob-cli-3s is closed and
  its plan file is marked done.
size: small
proposed_by: bbugyi200.athena.bob-cli-3s.land
bead: bob-cli-3s
status: done
---

- **PARENT:**
  [202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-3s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-3s.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-3s.land.md)
- **COMMITS:**
  - [1305af5](https://github.com/bobs-org/bob-cli/commit/1305af5b48be78dbe0c7f5b5bc05657dac5483b4)
    — refactor(native): drop unused facade re-exports to land bob-cli-3s

# Finish landing epic bob-cli-3s: drop unused facade re-exports, then close the epic

## Context

Epic `bob-cli-3s` ("Split the five largest Rust files into maintainable modules", plan
`plan:202610/split_largest_rust_files.md`) has five closed phases (`bob-cli-3s.1`
through `.5`). The land agent checked them all, and they really are complete:

- Each of the five components keeps its original unit-test names (61, 49, 21, 19,
  and 17) and its original function names (142, 91, 94, 76, and 101), compared against
  the pre-epic base `6192017`.
- Every resulting file is at most 859 lines. No component has a `mod.rs` root or uses
  `include!`.
- `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
  `cargo test`) passes on the cumulative tree.
- No commits other than the epic's own five landed after the epic started, so no
  integration work is needed.
- `sase bead epic-symbols bob-cli-3s` reports no entries.
- The phases' proposed follow-ups are already triaged and recorded as notes on
  `bob-cli-3s`: a +1 on `bob-cli-2e`, and a new flake task `bob-cli-3t`. Do not re-file
  them.

One defect caused by the epic remains. The new facade roots re-export items that nothing
uses through the facade. A non-test `cargo build --lib` therefore prints 7
`unused_imports` warnings, all in these facades; those are the only warnings that build
prints. The lib-test build (`cargo test --lib --no-run`) prints 4 more. Before the epic,
the items were defined in place, so nothing warned. Phase 1 also masked its own unused
re-export with `#[allow(unused_imports)]`. This tale removes those re-exports and then
closes the epic.

This is a structural cleanup. Change no behavior and no item definitions. Leave each
item's declared visibility in its child module as it is, because the items appear in the
signatures of re-exported functions. Do not touch the pre-existing clippy warnings in
moved code (`capture_task_toggle/ledger.rs`, `capture_task_toggle/links.rs`,
`plugins/sync.rs`). The epic plan forbids unrelated lint cleanup.

## Step 1: `src/native/capture_task_toggle.rs`

The facade's `pub(crate) use` blocks fall into three groups. The usage below was
verified with the compiler and `grep` outside `src/native/capture_task_toggle/`:

- **Used by sibling `native` modules (keep re-exported):** `list_open_entry_links`,
  `plan_pomodoro_link_ledger`, `PomodoroLinkLedgerAction`, `find_movable_task_links`,
  `plan_link_insertion`, `plan_link_removal`, `LinkInsertionOutcome`, `LinkPlacement`,
  `LinkPlanError`, `endpoint_from_entry`, `insert_named_placeholder`,
  `move_subtree_to_entry`, `plan_link_relocation`, `LinkRelocationAction`,
  `LinkRelocationError`, `PomodoroEndpoint`, `plan_task_link`, `set_task_line_status`,
  and `child_block_end_line`.
- **Used only by `capture_task_toggle/tests.rs` (through `use super::*;`):**
  `LinkInsertionPlan` (from `links`), `LinkRelocationPlan` (from `relocation`), and
  `plan_task_next`, `pull_forward_entry_text`, `PULL_FORWARD_REASON` (from
  `task_update`). Remove them from the production re-exports. In `tests.rs`, import them
  explicitly from their owning child modules, for example
  `use super::{links::LinkInsertionPlan, relocation::LinkRelocationPlan, task_update::{plan_task_next, pull_forward_entry_text, PULL_FORWARD_REASON}};`.
  If a child item's visibility does not reach `tests.rs`, use the `#[cfg(test)]` facade
  re-export pattern that `src/native/capture_clip.rs` already uses instead. Do not widen
  visibility.
- **Never used through the facade (delete from the re-export lists):** `OpenEntryLinks`,
  `PomodoroLinkLedgerPlan`, `select_implicit_open_entry`, `LinkRemovalPlan`,
  `MovableLink`, `ScheduledFieldMatch`, and `TaskTogglePlan`. Sibling child modules that
  need them already import them from `super::<child>::...`.

Keep the module docs and `#![allow(dead_code)]` exactly as they are; the attribute
predates the epic.

## Step 2: `src/native/task_status_hooks_write.rs`

- **Keep re-exported:** `acquire_maintenance_lock`, `apply_plan`, `new_run_id`,
  `ApplyError`, `ApplyOutcome`, `ApplySession`, `CaptureError`, `InputKind`,
  `InputSnapshot`, `ReasonCode`, `WritePlan`, `capture_optional`, `capture_required`,
  `planned_write`, and `snapshot_for_path`.
- **Delete from the `model` re-export (unused in every build):** `FileIdentity`,
  `InputState`, and `PlannedWrite`.
- **Test-only, so move out of the production facade:** `QUIET_PERIOD` and `RETENTION`
  (now in the `pub(crate) use model::{...}` list); the private `use model::TOOL;`; and
  the private
  `use recovery::{prune_completed, sha256_hex, unix_secs, vault_hash, RecoveryManifest};`.
  `task_status_hooks_write/tests.rs` currently reaches all of these through
  `use super::*;`. Give `tests.rs` explicit imports from `super::model` and
  `super::recovery`, or keep these lines in the facade behind `#[cfg(test)]`. Choose
  whichever compiles without widening visibility.

Keep `#![allow(clippy::result_large_err, clippy::type_complexity)]` and the module docs
unchanged.

## Step 3: `src/native/capture_complete.rs`

Delete the two lines `#[allow(unused_imports)]` and
`pub(crate) use shell::{ShellCompletion, ShellRow};`. Nothing outside
`src/native/capture_complete/` names these types; callers only use the value returned by
`capture_complete::shell_completion(...)`. The capture_complete tests do not use them
either. The `ShellRow` in `src/native/completion/report.rs` is a different type. The
types stay `pub(crate)` in `capture_complete/shell.rs`. Keep the other re-exports
(`build_cli`, `run`, `shell_completion`) unchanged.

## Step 4: Validate

1. Run `cargo fmt`.
2. `cargo build --lib 2>&1 | grep -c '^warning'` must print `0`. Confirm with the full
   output that no warning mentions `capture_task_toggle`, `task_status_hooks_write`, or
   `capture_complete`.
3. `cargo test --lib --no-run` must print no `unused_imports` warning for those three
   facades.
4. Run the focused suites, which must keep their counts:
   - `cargo test --lib capture_task_toggle`: 49 unit tests.
   - `cargo test --lib task_status_hooks_write`: 21 unit tests.
   - `cargo test --lib capture_complete`: 61 unit tests.
   - `cargo test --test cli complete`.
   - `cargo test --test randomize`.
5. Run `just all`. The repo has no `just check` recipe, and `just all` is the validation
   the epic plan names. One known flake can fail this run:
   `native::capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
   (tracked by `bob-cli-2e`), and likewise `native::ob::tests::lock_wait_behavior`
   (tracked by `bob-cli-3t`). If either fails, rerun it in isolation with
   `cargo test --lib <test name>` and record the result. Any other failure must be
   fixed.
6. Rerun the five-component line audit from the epic plan (`python3` script in
   "Validation and completion criteria"). Every file must stay at most 1500 lines.

## Step 5: Close epic bob-cli-3s (final step)

Do these in order, after Step 4 passes:

1. Run `sase bead epic-symbols bob-cli-3s`. It reported no entries at planning time. If
   any entries appear, resolve each one under the Symvision epic-whitelist policy: wire
   the symbol up, privatize it, add a non-test pragma, or delete it. No later open bead
   needs an exemption keyed to this epic, so do not re-key.
2. Close the epic:

   ```bash
   sase bead close bob-cli-3s --note "Verified all 5 phases (3s.1-.5) complete: unit test names (61/49/21/19/17) and fn names (142/91/94/76/101) identical to base 6192017; every split file <=859 lines, no mod.rs roots or include!; no non-epic commits landed during the epic, so no integration needed. Landing cleanup removed epic-introduced unused facade re-exports in capture_task_toggle.rs and task_status_hooks_write.rs (7 lib + 4 lib-test unused_imports warnings -> 0) and the #[allow(unused_imports)] ShellCompletion/ShellRow re-export in capture_complete.rs. just all <result>; focused suites <counts>. Follow-ups: BOB_DAY_FILE race (3s.2/.3/.5) +1'd on bob-cli-2e; ob lock_wait_behavior flake (3s.4) filed as bob-cli-3t; none declined. No epic-symbols entries."
   ```

   Fill `<result>` and `<counts>` with the actual Step 4 outcomes, including any flake
   rerun. Never pass `--force`. If the close is rejected, fix the stated cause and close
   again.

3. Symvision: `just --list` had no `symvision` recipe at planning time. Run
   `just symvision` only if `just --list` now shows one, and record the result.
4. Set `status: done` in the frontmatter of the epic's plan file. Use the path that
   `sase bead read bob-cli-3s -r "Need the plan path to mark it done"` prints under PLAN
   (`→ .../202610/split_largest_rust_files.md`, which is currently `status: wip`). That
   file lives in the git-backed plans store (`sase repo path plans`). If a later
   `sase bead` command fails its pull-rebase refresh because of the uncommitted edit,
   commit it in place with
   `git -C "$(sase repo path plans)" add 202610/split_largest_rust_files.md && git -C "$(sase repo path plans)" commit -m "chore(sdd): mark bob-cli-3s done"`.
5. `bob-cli-3s` has no `parent_bead`, so nothing further needs closing. The landing is
   complete once the epic is closed and its plan file is marked done.
