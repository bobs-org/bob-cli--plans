---
tier: tale
size: medium
title:
  "Land epic bob-cli-31 (task freshness): fix the epic-caused regressions, then close it"
goal:
  cargo test is green again, the freshness code adds no clippy warnings, block-id-prompt
  stamps only lines it rewrites, the cycler's follow-up stamp is guarded, and epic
  bob-cli-31 is closed with its plan file marked done.
proposed_by: bbugyi200.apollo.bob-cli-31.land
bead: bob-cli-31
create_time: 2026-09-30 23:51:14
status: wip
---

- **PARENT:**
  [202609/task_freshness_review.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/task_freshness_review.md)
- **BEAD:**
  [bob-cli-31](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-31/README.md)

# Plan: Land epic bob-cli-31 (task freshness)

## Context

Epic `bob-cli-31` ("Task freshness: a rolling review lease for Ready tasks", plan
`plan:202609/task_freshness_review.md`) has all ten phase beads closed (`bob-cli-31.1` …
`bob-cli-31.10`). The land agent verified each phase against its notes, the source, and
the epic's commits:

- bob-cli: 32d7007, 3cd4d44, f103979, 66c4e4c, fcf1f6a, 779cc0c, 663a0bc, 52969e5.
- bob-plugins: 8fd0f90, 3fc6a5f, 7e13c02, 566a3ef.

It also checked the vault artifacts, the deployed plugin copies, the chezmoi
`freshness:` block, and `bob freshness list` on the live vault.

- **Passes:** bob-plugins `npm test` 945/945 and `npm run validate` 6/6; bob-cli
  integration tests (685 CLI plus the parity suites); `cargo fmt --check`.
- **Integration:** the only unrelated commits that landed during the epic are
  bob-cli-2z's `=x` Work Log commits (c7ce096, 15c6341, 0ce41b9). They keep the epic's
  `[/]` stamp in `ClosePlanner::apply_startable` intact and add no new existing-task
  rewrite path. bob-plugins has no unrelated commits since the epic started.

Four epic-caused problems remain. They are this plan's work. Follow-up triage is already
finished and recorded on `bob-cli-31` (do not create beads for it).

1. **Five broken bob-cli unit tests.** Commit 3cd4d44 (capture-stamps, `bob-cli-31.3`)
   made `ClosePlanner::apply_startable` stamp `[fresh:: <close date>]` on every `[/]`
   line. The expected lines in `src/native/capture_pomodoro_close/linked_task_tests.rs`
   (written before the epic) were never updated, because that phase ran only the CLI
   tests. `cargo test --lib` fails these:
   - `worked_example_updates_tasks_and_writes_dated_work_logs` (assert near line 130)
   - `day_file_can_also_be_a_task_note_and_receives_its_work_log` (near 320)
   - `close_plan_preserves_crlf_in_changed_task_notes` (near 333)
   - `selection_in_progress_and_complete_updates_both_tasks` (near 471)
   - `typed_entry_lands_in_task_work_log_as_typed_subset` (near 606)

   For example, the actual output is
   `- [/] #task Ready [fresh:: 2026-09-28] ^ready\r\n`, but the test expects
   `- [/] #task Ready ^ready\r\n`.

2. **New clippy warnings from the epic.** About 18 lib warnings are in code the epic
   added. Other warnings on master are pre-existing (tracked by task `bob-cli-v`) or
   come from bob-cli-2z (for example `POMODORO_CLOSE_INTERNAL_BULLETS_ERROR` in
   `capture_language/markers.rs`); leave those alone.
3. **block-id-prompt stamps lines it did not otherwise rewrite.** The plan,
   `docs/freshness.md` § "Who stamps" ("Ctrl+Shift+Enter and `^^` when they rewrite the
   task line") and the plugin README all say the stamp applies only when the gesture
   already rewrites the task line. `bob capture`'s `plan_task_link` follows that rule.
   But `planTargetTaskUpdate` in `plugins/block-id-prompt/main.js` applies the injected
   `stampLine` unconditionally. A stamp-only change then sets `hasChanges`, so linking a
   Next task that already has a block ID rewrites its note just to stamp it. The test
   `planTargetTaskUpdate stamp-only change marks hasChanges` in
   `scripts/test-block-id-prompt.cjs` pins the wrong behavior.
4. **task-status-cycler's follow-up stamp has no guard.** The plan let the cycler stamp
   in a follow-up edit after the Tasks `set-status-symbol-to-*` command (option b), but
   only "when the line still holds the same task with the expected new status".
   `setActiveCheckboxStatus` (around line 10400 of `plugins/task-status-cycler/main.js`)
   calls `this.stampFreshnessOnEditorLine(editor, taskStatus.line)` after
   `tryExecuteTasksCommand` without that check. If Tasks inserts a recurrence line or
   deletes the line (`onCompletion`), it could stamp a different task.

## Step 1 — bob-cli: fix the five unit tests

In `src/native/capture_pomodoro_close/linked_task_tests.rs`, update every expected `[/]`
task line in the five tests above to carry `[fresh:: 2026-09-28]` (the close date those
tests use). Use the canonical placement from `docs/freshness.md`: immediately before the
trailing Tasks suffix (the leftmost Tasks-key field such as `[created::…]`, tag, or
`^id`).

- `- [/] #task Add support for \`=x\` syntax! [created::2026-09-26] ^capture-stop`→`-
  [/] #task Add support for \`=x\` syntax! [fresh:: 2026-09-28] [created::2026-09-26]
  ^capture-stop`
- `- [/] #task Ready ^ready\r\n` → `- [/] #task Ready [fresh:: 2026-09-28] ^ready\r\n`

Take the exact expected strings from the `left:` values that
`cargo test --lib capture_pomodoro_close::linked_task_tests` prints. Confirm that each
one only adds the stamp; change no production code. If any failure is something other
than the added `[fresh:: …]`, stop and investigate before editing.

## Step 2 — bob-cli: clear the epic's clippy warnings

Run `cargo clippy --all-targets --all-features` and make these epic-introduced
diagnostics disappear without changing behavior:

- **Unused re-exports:**
  - `src/native/freshness/mod.rs`: the `placement::{…}` and `state::{…}` re-export
    lists.
  - `src/native/dataview.rs`: `parse_details`, `TaskDetails` in the `tasks::{…}`
    re-export.
  - `src/native/task_status_hooks/mod.rs`: `TaskMetadata` in
    `parse::{markdown_files, task_metadata, TaskMetadata}`.

  Remove re-exports nothing uses. Make ones that only tests use `#[cfg(test)]`.
  Production callers use `freshness::stamp_fresh` (in `capture_task_toggle.rs` and
  `capture_pomodoro_close/linked_tasks.rs`), so keep that one.

- **Dead code that is part of the documented contract or the test vectors:**
  - `freshness::placement::set_refresh` and the `RefreshEdit::Set` / `Clear` variants
    (the Rust half of the P12 vector);
  - `freshness::state::collect_lints`;
  - the `FreshnessRow.status` / `.created` fields and the other `created` field near
    `state.rs:74`;
  - `FreshnessConfig::interval` / `stale_daily_budget` in
    `src/native/config/freshness.rs`.

  Prefer deleting what is truly unneeded. For contract helpers that tests exercise, use
  `#[cfg(test)]` or a scoped `#[cfg_attr(not(test), allow(dead_code))]` with a one-line
  reason comment. Match the module-level `allow(dead_code)` idiom already used in
  `task_fields.rs` and `capture_task_toggle.rs` only if narrower options don't fit.

- **Style lints:**
  - `collapsible_if` at `freshness/placement.rs:~242`, `freshness/seed.rs:~203`,
    `config/freshness.rs:~93` and `:~101`;
  - `let_and_return` at `freshness/seed.rs:~480`;
  - `items_after_test_module` at `freshness/seed.rs:~576` (move the items above the test
    module);
  - `explicit_auto_deref` / needless borrow at `freshness/scan.rs:~171`;
  - `useless_format` at `freshness/scan.rs:~121`;
  - `type_complexity` at `freshness/scan.rs:~304` (add a type alias).

Acceptance: no clippy diagnostic remains in `src/native/freshness/**` or
`src/native/config/freshness.rs`, and none remains on the three re-export lines. The
only clippy error left is the pre-existing deny at
`tests/cli/capture/pomodoro_name.rs:808` (`|| true`). Epic `bob-cli-28` owns it; do not
fix it here. Run `cargo fmt` afterwards.

## Step 3 — bob-cli verification

Run `cargo fmt --check`, `cargo test` (lib, CLI and parity suites, all green), and the
clippy command above. Report the before/after warning count.

## Step 4 — bob-plugins: block-id-prompt stamps only lines it rewrites

Open the repo with `sase repo open bob-plugins -r "<why>"`, read its `AGENTS.md`, and
work only in the printed path.

1. In `planTargetTaskUpdate` (`plugins/block-id-prompt/main.js`), compute
   `const lineRewritten = removedFutureSchedule || statusChanged || Boolean(newBlockId);`
   after the status and block-ID edits. Call `applyFreshStampLine` only when
   `lineRewritten` is true. Keep `freshnessChanged` and `hasChanges` as they are; a
   stamp can then never be the only change. Update the comment above the stamp to say it
   applies only when the gesture already rewrote the line, matching `bob capture`.
2. In `scripts/test-block-id-prompt.cjs`, replace
   `planTargetTaskUpdate stamp-only change marks hasChanges` with a test that the same
   input (`- [*] #task Ship it ^ship`, `activationEligible`, a stamper) leaves the
   content unchanged, never calls the stamper, and reports `freshnessChanged: false` and
   `hasChanges: false`. Add a case where a Next task without a block ID gets a new block
   ID (`newBlockId`) and is stamped. Keep the other freshness tests passing.
3. Bump `plugins/block-id-prompt/manifest.json` from `1.16.0` to `1.17.0` and the
   version cell of its row in the repo `README.md`. The row text already describes the
   intended rule ("when they rewrite its line").

## Step 5 — bob-plugins: guard the cycler's follow-up stamp

1. In `setActiveCheckboxStatus` (`plugins/task-status-cycler/main.js`), Tasks-command
   branch:
   - record `editor.lineCount()` before `tryExecuteTasksCommand(commandId)`;
   - after it, call `this.stampFreshnessOnEditorLine(editor, taskStatus.line)` only when
     the line count is unchanged and
     `getTaskStatusForLine(editor.getLine(taskStatus.line), taskStatus.line)?.symbol === nextSymbol`.

   Guard defensively if `lineCount` is missing (no stamp). Keep `return true`. Update
   the comment: this is the plan's option (b) and its same-task guarantee.

2. Add tests to `scripts/test-task-status-cycler.cjs` near the existing
   `cycler freshness:` tests. Fake the Tasks command through `plugin.app.commands` so
   `executeCommandById` mutates the test editor, and cover:
   - a normal open-to-open rewrite is stamped;
   - a command that inserts a line (recurrence) is not stamped;
   - a command that deletes the line is not stamped, and the following task is
     untouched;
   - a command that leaves a different status is not stamped.
3. Bump `plugins/task-status-cycler/manifest.json` from `1.18.0` to `1.19.0` and its
   `README.md` version cell.

## Step 6 — bob-plugins verification and deploy

Run `npm test` and `npm run validate` (all pass). Deploy both plugins with
`bob plugins sync -n -r "<opened bob-plugins path>" -p block-id-prompt` and
`... -p task-status-cycler`. Confirm with `cmp` that the vault copies under
`~/bob/.obsidian/plugins/<id>/` match the repo files.

## Step 7 — close the epic (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-31`. At land time it printed "No --epic-symbol
   entries for bob-cli-31". If entries appear, resolve each one (wire it up, privatize
   it, add a non-test pragma, or delete it) or re-key it to a still-open bead that needs
   it.
2. Close the epic. Never use `--force` merely to make the close succeed:

   ```bash
   sase bead close bob-cli-31 --note "<verification>"
   ```

   The note should summarize:
   - all ten phases were verified against notes, source and commits;
   - the bob-cli-2z integration was checked;
   - the five `linked_task_tests` expectations were fixed;
   - the epic's clippy warnings were cleared (give the counts);
   - block-id-prompt 1.17.0 stamps only rewritten lines;
   - the task-status-cycler 1.19.0 follow-up stamp is guarded;
   - test results for both repos and the deploys;
   - the follow-up triage already recorded on the epic: the `bob-cli-28` corroboration,
     new task `bob-cli-33`, and the declined proposals with reasons.

3. Run `just symvision` if the recipe exists. It did not exist in bob-cli at land time;
   say so in the final response if it is still missing.
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path shown by
   `sase bead read bob-cli-31 -r "<why>"` (`plan:202609/task_freshness_review.md` in the
   plans sidecar). Change only that field.
5. `bob-cli-31` has no `parent_bead`, so nothing further cascades.

## Final response: Bryan's checklist (not automated)

- **In Obsidian:**
  - The status bar shows `⟳ N due · N new · ✓ N today`, and its counts match
    `bob freshness list`.
  - `]s` / `[s` and Ctrl+Alt+J/K walk the queue across notes.
  - Alt+F stamps with no other change; Alt+Shift+F stamps and advances.
  - `freshness.md` groups NEW → DUE, and the REVIEW chip links to it.
  - `fresh` shows as a muted pill.
- **Undo check:** on an open `#task` line, press Alt+] and then Ctrl+Z once. If that
  removes only the `[fresh::]` stamp and leaves the new status, the cycler's follow-up
  edit is a separate undo step. File a follow-up only if that bothers you.
- **Tune the load:**
  - Optionally triage `gkeep_inbox.md`, then set `task_refresh: 2` on `gkeep_inbox.md`
    and `mac_inbox.md`.
  - Consider `task_refresh: 14` on `sase.md`.
  - Set `freshness.stale_daily_budget` only if mornings run long.
- **MacBook and athena:** reinstall `bob` and run `bob plugins sync`.
- **Two-week trial:** keep the design if, on at least 10 of 14 mornings, NEW reaches 0
  before planning and the ritual takes ≤10 minutes.
