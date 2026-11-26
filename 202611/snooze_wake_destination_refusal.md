---
tier: tale
size: small
title: Refuse unplaceable snooze wakes, then land bob-cli-q5.1.1 and its parent phase
goal:
  A snooze wake that cannot place a never-running Pomodoros destination refuses without
  writing, and epic bob-cli-q5.1.1 plus phase bob-cli-q5.1 close with recorded
  verification.
proposed_by: bbugyi200.athena.bob-cli-q5.1.1.land
proposed_with: claude/opus
bead: bob-cli-q5.1.1
create_time: 2026-11-25 19:09:54
status: wip
---

- **PARENT:**
  [202611/snooze_wake_engine.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snooze_wake_engine.md)
- **BEAD:**
  [bob-cli-q5.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.1.1.md)

# Refuse unplaceable snooze wakes, then land bob-cli-q5.1.1

The land agent for epic `bob-cli-q5.1.1`, the snooze wake engine, verified all five
phases against `plan:202611/snooze_wake_engine.md`. It found one defect the epic caused.
This tale fixes it and then finishes the landing. Nothing resumes the landing after this
tale, so the closeout below is part of this work.

## Context

The lander already made the integration edits below, committed before this tale starts:

- `bob snooze add` now settles each note it rewrites through
  `task_settle::SettleContext`, so a Ready→Next promotion lands in its status group. The
  CLI test `snooze::add_settles_a_promoted_task_into_its_status_group` covers this.
- The `#[allow(unused_imports)]` re-export block in `src/native/snooze/mod.rs` is gone.
  Consumers import through `state::{classify_rows, live_targets}`,
  `when::parse_snooze_when`, and `plan::{plan_add, plan_drop, plan_wake, plan_cleanup}`.
  The test-only `move_subtree_to_entry_at` re-export is now `#[cfg(test)]`.
- `docs/snooze.md` no longer calls the CLI, receipts, and tick "follow-up phases", and
  it names `parse_snooze_when`. The snooze row in `docs/reference/json-outputs.md` is
  trimmed.
- Follow-up triage is recorded on `bob-cli-q5.1.1`. New tasks are `bob-cli-q9` and
  `bob-cli-qa`. Notes were added to `bob-cli-q4`, `bob-cli-q7`, `bob-cli-jz`, and
  `bob-cli-ob`. Do not re-triage.

Governing decisions: `decisions:consumption-keeps-notes-and-recorded-time`,
`decisions:today-is-read-from-the-ledger`, `decisions:task-lanes-are-sticky`, and
`decisions:bob-cli-releases-ride-a-daily-train`. Do not edit `Cargo.toml` versions or
`CHANGELOG.md`. Wake must never change a task line, and every wake destination must sit
in today's `## Pomodoros` so it counts as Today.

## Defect: wake writes outside the ledger and reports success

`plan_wake` in `src/native/snooze/plan.rs` swallows destination failures:

- `resolve_or_create_destination` maps an `insert_named_placeholder` error to
  `(day, 0, name)`. The errors are `NoPomodorosSection`, `MultipleOpenTimedEntries`, and
  `InvalidPomodoroName`.
- `untyped_destination` maps an `insert_unnamed_placeholder` error to `(day, 0)`.
- The `insert_subtree_into_entry(..).unwrap_or_else(..)` call in `plan_wake` swallows
  its error the same way.

The destination then becomes line 1, so the subtree is inserted after the daily note's
first line, outside `## Pomodoros`. The row is still struck, its notes are removed, the
report says `status: "woke"` with `destination: {line: 1, ..., created: true}`, a
receipt is written, and a notification is sent. The lander reproduced this with the CLI
in two cases:

1. A typed task (`[type:: WRITE]`) with no untimed WRITE entry, in a note with two open
   timed entries: `- [ ] (**1100-1200** [t:: 60m]) — WORK` and
   `- [ ] (**1130-1230** [t:: 60m]) — BUILD`. Due row `- **1130** [[jobs#^job]]` with a
   child note, `BOB_NOW=2026-10-08T12:00:00-04:00`.
2. The same daily note with no `## Pomodoros` section at all.

In both cases the output began `# 2026-10-08\n\t- [[jobs#^job]]\n\t  - keep this note`.

## Fix

1. In `snooze/plan.rs`, add
   `WakeRefusal { NoPomodorosSection, MultipleOpenTimedEntries }` with `as_str()` values
   `no_pomodoros` and `multiple_open_timed`. Make `plan_wake` return
   `Result<WakePlan, WakeRefusal>`.
   - Make `resolve_or_create_destination` and `untyped_destination` return `Result`.
   - Map `MultipleOpenTimedEntries` to its own variant. Map every other placeholder or
     insert error to `NoPomodorosSection`. `InvalidPomodoroName` cannot occur for a
     canonical type name, so leave it no new variant.
   - Refuse the whole batch before striking or pruning anything. Wake stays atomic: no
     partial batch.
   - Keep the untyped path unchanged when Pomodoros exists. `insert_unnamed_placeholder`
     already places after the first timed entry and does not refuse multiple timed
     entries.
   - A wake that wakes nothing (no due rows, or prune only) must not refuse merely
     because the ledger is malformed. Resolve destinations only for real jobs.
2. In `snooze/cli.rs`, map `WakeRefusal` to `CliError::refusal(kind, message)`. That is
   exit 2 and the existing JSON error envelope. Teaching copy:
   - `no_pomodoros`: "today's note has no ## Pomodoros section", the same as add.
   - `multiple_open_timed`: "today's ledger has more than one running Pomodoro — close
     one before waking".

   Explicit `wake TASK` and sweeps both refuse. Dry run reports the same refusal.
   Nothing is locked or written, and no receipt or notification is produced. The refusal
   must surface whether it happens in the initial plan or in the re-plan under the lock.
   Update callers of `plan_wake`, including the plan.rs wake-vector runner and its unit
   tests, for the `Result`.

3. Tests:
   - plan.rs unit tests: both refusals return `Err` and leave no postimage. Two open
     timed entries plus an untyped task still wake after the first timed entry, and
     never into the timed entry itself.
   - A `tests/cli/snooze.rs` case for each reproduction above: exit 2, the error `kind`,
     a byte-identical daily note, no `state/bob-cli/snooze/last-wake.json`, and no
     `notify.json` sink.
   - Optionally add rust-only `wake` vectors with an expected refusal. Extend the `wake`
     schema text only if you do, and keep plugin-run groups unchanged.
4. Docs: in `docs/snooze.md` (Mutations `plan_wake` bullet and the CLI paragraph), state
   that a wake whose never-running destination cannot be placed refuses the whole batch
   with `no_pomodoros` or `multiple_open_timed`, exit 2, and writes nothing. Keep
   `docs/snooze.md` out of the docs budget errors.

## Verification

Run `cargo fmt --all`, `just check`, and `just docs-check`.

- At master `6a3e5df6` plus the lander edits, `just check` stops at clippy on a known
  error from active epic `bob-cli-q7`: `-D clippy::manual-contains` at
  `tests/cli/capture/complete_block_id.rs:292`. It is recorded on `bob-cli-q7`; do not
  fix it here. If it still fails, also run `cargo test --workspace --no-fail-fast` and
  confirm the failures are exactly the known set:
  - lib:
    `native::meetings::profile::tests::wispr_summary_renders_one_or_two_pages_with_stable_words`,
    tracked by `bob-cli-q9`;
  - `tests/cli`: the 17 settle-drift nodes recorded on `bob-cli-q4` (inbox_route ×2,
    note_pull ×3, project_note ×3, project_note_offer ×6, task_theme_writes ×1,
    ref_library::today ×1, and the task_status_hooks grouping test).

  No snooze or tick test may fail.

- `just docs-check` is expected to fail only on the tracked reference budgets
  (`capture-syntax.md`, `json-outputs.md`, `bob-navigation-hotkeys.md`) plus the
  `plugin-apis.md` warning. Any snooze page in that list is a regression of this tale.
- Re-run both CLI reproductions above with the rebuilt binary and confirm exit 2 with no
  file changes.

## Closeout (final step)

Do this in the same turn as the code. Do not wait for this work's own commit SHA, a
push, or CI.

1. Run `sase bead epic-symbols bob-cli-q5.1.1`. Resolve each listed entry by wiring it
   up, privatizing it, or deleting it. Re-key it to a still-open bead only if that bead
   needs the exemption. The lander saw none.
2. Close the epic: `sase bead close bob-cli-q5.1.1 --note "<verification>"`. The note
   states:
   - all five phases were verified against the plan;
   - this wake-refusal fix;
   - the lander integration edits: add settles notes, re-export suppressions removed,
     docs refreshed;
   - the triage outcomes recorded on the epic;
   - the exact check results.

   Never use `--force` to make the close succeed.

3. Run `just symvision` if the recipe exists. At planning time it did not.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read bob-cli-q5.1.1 -r "<why>"`
   (`plan:202611/snooze_wake_engine.md`).
5. The parent bead is phase `bob-cli-q5.1` (wake-engine). Verify the child plan
   completed its description:
   - the snooze module, CLI, and guarded writes;
   - never-running placement and struck history;
   - the first-position tick step;
   - receipt, Darwin post, and desktop fallback;
   - the writer audit, shared vectors, and `docs/snooze.md`.

   Then record a note on `bob-cli-q5.1` covering:
   - passing checks and the writer audit (`tests/cli/snooze_writer_audit.rs`);
   - exposed APIs for `bob-cli-q5.2` and `bob-cli-q5.3`:
     `snooze::state::{classify_rows, live_targets}`, `snooze::when::parse_snooze_when`,
     `snooze::plan::{plan_add, plan_drop, plan_wake, plan_cleanup}`, and
     `routing::PlaceInput.never_running`;
   - that relative times round the offset up to five minutes (vector
     `when-relative-45m`: 13:02 + 45m → 13:47), an interpretation of "round up to the
     next 5 minutes" that the outer phases should confirm before copying vectors;
   - the remaining outer work: q5.2 planned-today readers, q5.3 capture `z:`, plugin and
     Mac phases, glossary/guide, and live-vault acceptance.

   Then run `sase bead epic-symbols bob-cli-q5.1`, resolving or re-keying any entries,
   and close only that phase with
   `sase bead close bob-cli-q5.1 --note "<what you verified>"`. Leave the containing
   epic `bob-cli-q5` to its waiting land agent. Never close `bob-cli-q5`.
