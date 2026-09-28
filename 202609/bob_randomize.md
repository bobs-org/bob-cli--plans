---
tier: epic
title: 'bob randomize: bulk re-roll of due prioritized tasks'
goal: '`bob randomize` re-rolls every due P1–P4 Obsidian task to its own random date
  inside that task''s configured priority window. Each touched note is written once,
  with the new date, Blocked status, a 🎲 Schedule Log entry, and status grouping applied
  together. The result is published as exactly one scoped `bob randomize` commit,
  taken between two vault-sync cycles under the shared maintenance lock. The command
  also has a reproducible dry-run preview, polished human output, and a stable JSON
  contract.

  '
phases:
- id: planner
  title: Pure randomize planner and shared task-field helpers
  depends_on: []
  size: medium
  description: 'planner: lift inline-field parsing into a shared task_fields module;
    add priority level lookups, seed mixing, the randomize Schedule Log reason and
    insertion, and hooks helper exposure; build the pure randomize_plan module that
    turns note snapshots into per-note postimages, a reroll list, skip reasons, and
    load data. Includes exhaustive unit tests.'
- id: plumbing
  title: Lock wait, scoped commit, sync report, and writer reuse
  depends_on: []
  size: small
  description: 'plumbing: add a bounded lock wait and a scoped path commit helper
    to ob.rs, a structured report variant of the in-process vault-sync cycle, and
    a tool-name parameter for the guarded writer so recovery records land under bob-cli/randomize.
    Existing hooks, nightly, and vault-sync behavior stays unchanged.'
- id: command
  title: bob randomize command, output, and integration tests
  depends_on:
  - planner
  - plumbing
  size: medium
  description: 'command: add the clap CLI and runner registration; orchestrate lock,
    pre-sync, plan and guarded apply with retries, scoped commit, and post-sync; render
    the human and JSON outputs and exit codes; add tests/randomize.rs integration
    coverage, including git, conflicts, determinism, and task-status-hooks parity.'
- id: docs
  title: Documentation and cross-links
  depends_on:
  - command
  size: small
  description: 'docs: write docs/randomize.md as the full contract, add README index,
    section, workflow, and environment entries, and cross-link projects.md, vault-git-sync.md,
    task-status-hooks.md, and docs/README.md.'
proposed_by: bbugyi200.apollo.2q
create_time: 2026-09-28 10:45:16
status: done
bead_id: bob-cli-2b
---

- **PROMPT:** [prompts/202609/bob_randomize.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/bob_randomize.md)
- **BEAD:** [bob-cli-2b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2b/README.md)

# Plan: `bob randomize`, a bulk re-roll of due prioritized tasks

## Context

Bryan marks lower-priority work with an Obsidian Tasks `priority` field; a task with no
`priority` is implicitly P0. After days or weeks without reviewing tasks, the vault
fills with overdue P1–P4 tasks. Those tasks bury the P0 work that needs full attention.
`bob randomize` is the headless, vault-wide version of the Obsidian `Ctrl+Shift+P`
picker's same-level 🎲 "roll". It re-schedules every due prioritized task to an
independent random date inside that task's priority window from
`~/.config/bob/config.yml`, then publishes the whole change as a single Git commit that
cooperates with `bob vault-sync`.

Design research (read it with `sase artifact read` if you need the background):
`research:202609/bob_randomize_backlog_reroll/bob_randomize_backlog_reroll.md`. A
read-only scan of the live vault on 2026-09-28 found 224 candidates in 28 notes (87 P1,
132 P2, 5 P3). 130 of them are in `sase.md`, all 224 already have a Schedule Log, none
has `due` or `repeat` metadata, 1 is Next, 1 is In Progress, and 40 due P0 tasks must
stay untouched.

The current config (priority property; `value` is what lands in notes):

| Label | Value    | Window      |
| ----- | -------- | ----------- |
| P1    | `high`   | 2–7 days    |
| P2    | `medium` | 8–30 days   |
| P3    | `low`    | 31–90 days  |
| P4    | `lowest` | 91–365 days |

**Hard safety rule for every phase:** never run a live (non-`--dry-run`)
`bob randomize`, `bob vault-sync`, or `bob task-status-hooks` against the real `~/bob`
vault. Tests use temp vaults with `BOB_DIR`, `BOB_NOW`, `BOB_CONFIG_FILE`,
`BOB_VAULT_SYNC_LOCK_FILE`, and `BOB_VAULT_SYNC_STATE_FILE`. A
`cargo run -- randomize --dry-run` against the real vault is read-only and is a fine
final smoke check.

## Product contract (all phases implement this)

### What qualifies

The scan covers every Markdown note the vault walkers already scan, using
`task_status_hooks::markdown_files`. That walker skips `done/`, dot directories,
`_conflicts`, `_generated`, and `_templates`. Task lines come from `note_tasks::scan`,
which applies the Tasks global filter (`#task` by default) and skips frontmatter and
fenced code. Only the task's own line is inspected; child lines are not. Inline fields
are matched anywhere on that line in both `[key:: value]` and `(key:: value)` forms,
with any spacing around `::`. This is the same recognition used by the
`<ctrl+shift+enter>` toggle, `bob projects sync`, and the picker. Each task is
classified in this order:

1. Ignore it (not considered, not reported) when any of these holds:
   - it is not an open task by Tasks status type;
   - it has no `priority` field;
   - it has no `scheduled` field;
   - its block ID is `prj` (`bob projects sync` owns `^prj` scheduling).

   One exception: an open task with no `priority` and exactly one valid `scheduled` on
   or before today increments the summary's **still due P0** count.

2. More than one `priority` field or more than one `scheduled` field: skip as
   `duplicate_field`.
3. The single `scheduled` value is not a strict `YYYY-MM-DD` calendar date: skip as
   `invalid_scheduled`.
4. The `scheduled` date is after the cutoff (`--until`, default today): ignore it, since
   the task is not due.
5. The `priority` value (trimmed) matches no configured level `value`: skip as
   `unknown_priority`, with the value as detail.
6. A `due` or `repeat` inline field, or a `📅` or `🔁` emoji, is on the line: skip as
   `hard_date`. A deadline or recurrence makes `scheduled` more than a soft date.
7. Next `[*]`: skip as `next`. In Progress `[/]`: skip as `in_progress`. Any other open
   symbol besides Ready `[ ]` and Blocked `[?]`: skip as `other_status`.
8. The task's block ID is linked from any open Pomodoro in today's daily note: skip as
   `pomodoro`. A link counts at any depth of the entry's child block, including
   `[[target#^id]]`, `![[target#^id]]`, and aliased forms. The target matches the task's
   note by vault-relative path without `.md` or by file stem, both case-insensitive. An
   empty target means the daily note itself. Over-matching is acceptable because
   skipping is the safe side.
9. `--level` was given and the task's level is not selected: count it as `not_selected`.
10. Anything left is a **candidate**: a Ready `[ ]` or Blocked `[?]` task.

Skip reasons fall into two groups. **Left alone** (`next`, `in_progress`, `pomodoro`,
`not_selected`) is intentional and only reported as counts. **Needs a look**
(`duplicate_field`, `invalid_scheduled`, `unknown_priority`, `hard_date`,
`other_status`) is listed with `path:line` so it can be fixed. Neither group is fatal.

### The roll

- **today**: the same effective day `task-status-hooks` uses. That is the date in the
  `BOB_DAY_FILE` filename when it parses (`pomodoro::day_file_for` plus hooks'
  `daily_anchor_date`), otherwise `bob_env::current_datetime().date()`.
- **until**: `--until DATE|+N`, defaulting to today. It must be today or later. It is
  both the cutoff and the base the windows are rolled from.
- **new date** = `until + level.roll_offset(task_seed)`, using the existing inclusive
  splitmix64 roll in `config.rs`.
- **task_seed**: the mix of the base seed with a stable 64-bit hash (FNV-1a, never
  `std::hash::DefaultHasher`) of three inputs: the vault-relative path, the task line's
  `note_tasks::line_digest`, and the ordinal among identical digests in that note. Dates
  therefore depend on task identity, not scan order or line numbers. A dry run and a
  later live run with the same `--seed` give every unchanged task the same date, even
  after the pre-sync pulls in unrelated edits.
- **base seed**: `--seed` (decimal or `0x`-hex u64), else `BOB_PRIORITY_ROLL_SEED`, else
  `config::roll_seed() & 0xffff_ffff`, so generated seeds print as 8 hex digits. The
  seed is always printed as `0x…` hex.
- If the new date equals the old date (possible only with `min_days: 0`), the task is
  counted as `unchanged` and nothing is written for it.

### What each re-roll writes

Everything is computed in memory, one postimage per note:

1. Replace exactly the 10 date bytes of the `scheduled` value. Keep the bracket or paren
   form, the `::` spacing, and all other text.
2. When the new date is after today, a Ready `[ ]` task becomes Blocked `[?]`
   (`set_task_line_status`), and `[?]` stays `[?]`. This matches the status hooks would
   derive for a future date.
3. Write a Schedule Log entry in the picker's grammar:
   `*2026-09-10 → 2026-10-19* — 🎲 P2 randomize · in **21** (8–30) days`. With `--until`
   after today, append `from <until>`: `… · in **17** (8–30) days from 2026-10-12`.
   - If the task has a direct-child Schedule Log marker, prepend the entry as that
     marker's first child (newest first). Recognize markers with
     `capture::first_direct_managed_log_start`, which also accepts the legacy
     `**Schedule log**` label.
   - Otherwise append `- 🗓️ **SCHEDULE LOG**` plus the entry as the task's last direct
     child, which is how the picker creates a missing log.
   - Indentation reuses the first existing child indentation, then the note's
     `dominant_indent_unit`, then a tab. Bullets are `-`.
4. Apply edits bottom-up so offsets stay valid. Preserve each line's `\n` or `\r\n`
   ending and the file's final-newline state.
5. After all of a note's edits, run `task_status_groups::transform` once with hooks'
   classification, if the note is grouping-eligible. Eligible means `[[area]]` or
   `[[project]]` frontmatter, not a canonical `YYYY/YYYYMMDD.md` daily note, and not
   today's day file. Newly Blocked root tasks then move under `### Blocked` and the
   badge row is regenerated in the same write. `task-status-hooks` is left with nothing
   to change in those notes, so no second commit appears. Pre-existing layout drift that
   `transform` fixes in a touched note is included; hooks would make the same change
   anyway. Grouping warnings are surfaced as warnings.

If any postimage sets `[?]`, the Tasks status registry must pass hooks'
`validate_blocked_status`. Otherwise the run fails before any write.

### Git: one scoped commit that cooperates with vault-sync

Live run sequence, all under one hold of the shared `bob_sync.lock` (the same lock
vault-sync, nightly, and hooks use):

```text
acquire lock           bounded wait (--retry-timeout), visible "waiting…" line
pre-sync               vault_sync cycle in-process: commits pending edits as
                       vault(<host>), recovers merges, fetches/merges, pushes.
                       On failure: abort, nothing written, exit 1, hint --offline
scan + plan            fresh from disk, pure
apply                  guarded writer: 2 s quiet period, snapshot preflight,
                       recovery copies, atomic renames. On vault_changed,
                       quiet_period, or unstable_read with nothing applied,
                       re-scan and re-plan with the same seed within the budget
commit                 git add -- <applied> ; git commit -F - -- <applied>
post-sync              vault_sync cycle again: fetch/merge if the remote moved,
                       push with retries
release lock
```

- randomize never runs `add -A`, merges, rebases, amends, force-pushes, or pushes on its
  own; vault-sync remains the only code that does those.
- After a run, `master` holds, in order: an optional `vault(<host>)` commit from the
  pre-sync, exactly one `bob randomize …` commit containing only the rewritten notes,
  and an optional merge commit if the remote moved.
- Commit message (follows the `bob move-done-tasks <date>` precedent):

  ```text
  bob randomize 2026-09-28: 222 tasks in 28 notes

  P1 85 · P2 132 · P3 5
  until 2026-09-28 · seed 0x7f3a91c2

  130 sase.md
   14 cash.md
  …
  ```

- `--offline` skips both sync cycles but still makes the scoped local commit; the
  background vault-sync publishes it later.
- If the vault is not a Git worktree, the notes are written with a warning and all Git
  steps are skipped.
- A post-sync conflict follows vault-sync's policy: the remote copy wins in place and
  randomize's version is kept under `_conflicts/`. Print a loud warning naming the
  notes, say that re-running re-rolls whatever is still due, and exit 1.
- If the push fails after the commit, print "committed `<sha>` locally; background
  vault-sync will publish it" and exit 1. Never roll back good local edits.
- If the apply is partial (I/O error after some renames), commit the notes that were
  written (each one is self-consistent), post-sync, report the remaining notes and the
  recovery directory, and exit 1. Re-running converges.
- Undo is `git -C ~/bob revert <sha> && bob vault-sync`. It reverts cleanly because
  status and grouping live in the same commit.
- Dry run: no lock, no sync, no writes, no recovery directory, no status file.

### CLI

```text
bob randomize [-d|--dry-run] [-f|--format human|json] [-l|--level LABEL]...
              [-o|--offline] [-r|--retry-timeout SECONDS] [-s|--seed SEED]
              [-u|--until DATE|+N]
```

Options are alphabetical and each has a short alias, per the `cli_rules` memory. `-h`
and `--help` are listed in order like `task-status-hooks` does (`disable_help_flag` plus
an explicit `help` arg).

| Option                        | Behavior                                                                                                                                                        |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-d, --dry-run`               | Preview the full plan without locking, syncing, or writing                                                                                                      |
| `-f, --format`                | `human` (default, colored on a TTY, `NO_COLOR` honored) or `json`                                                                                               |
| `-l, --level LABEL`           | Repeatable. Only re-roll these configured labels (ASCII case-insensitive, e.g. `-l p2`). An unknown label is a usage error (exit 2) that lists the valid labels |
| `-o, --offline`               | Skip both vault-sync cycles; commit locally without pushing                                                                                                     |
| `-r, --retry-timeout SECONDS` | Budget for waiting on the maintenance lock and for re-planning after concurrent-edit races (default `60`; `0` = fail fast)                                      |
| `-s, --seed SEED`             | Base seed (decimal or `0x` hex). A dry run prints the seed that reproduces its dates                                                                            |
| `-u, --until DATE\|+N`        | Treat tasks scheduled through DATE (or N days from today) as due and roll their windows from that date. Must be today or later (exit 2 otherwise)               |

There is no `--bob-dir`. The in-process vault-sync cycle is bound to `BOB_DIR`, and a
flag would let edits and sync target different vaults; `nightly` and `vault-sync` make
the same choice. There is no confirmation prompt, because the dry run, seed replay, and
one revertible commit already make it safe.

The help `long_about` should explain, in a few short paragraphs:

- what qualifies;
- what gets written (date, Blocked, 🎲 Schedule Log, grouping);
- what is left alone (P0, Next, In Progress, today's Pomodoro tasks, `due` and `repeat`
  tasks);
- the lock → sync → commit → sync behavior and how to undo it.

Examples (after_help):

```text
bob randomize --dry-run                 Preview what would move and where
bob randomize --seed 0x7f3a91c2         Apply the dates a dry run showed
bob randomize --level P2 --level P3     Leave P1 tasks for hand triage
bob randomize --until +7                Clear a week for P0 work
bob randomize --dry-run --format json   Machine-readable preview
```

Environment section: `BOB_DIR`, `BOB_NOW`, `BOB_DAY_FILE`, `BOB_CONFIG_FILE`,
`XDG_CONFIG_HOME`, `BOB_PRIORITY_ROLL_SEED`, `BOB_VAULT_SYNC_LOCK_FILE`,
`BOB_VAULT_SYNC_STATE_FILE`, `NO_COLOR`.

Top-level registration: a `randomize` row between `query` and `task-status-hooks` in
`runner.rs` `SUBCOMMANDS` ("Re-roll due prioritized tasks within their priority
windows"), and a `bob randomize --dry-run` example in `AFTER_HELP`, in the same
alphabetical position.

### Human output

Use `style::Styler`, the same helper the other commands use for color. Color roles:

- **Priority heat:** P1 red, P2 yellow, P3 blue, P4 dim.
- **Dates:** cyan.
- **Status:** ✓ green, ⚠ yellow, ✗ red.
- **Secondary text:** dim.

Without color, the layout is identical without ANSI codes. Summary dates use
`%a %b %-d`; per-task lines use ISO dates to match the Schedule Log. Descriptions are
truncated with `…` so each line fits in 100 columns.

Live run:

```text
🎲 bob randomize · Mon Sep 28 · ~/bob

 ✓ Synced vault            committed 2 pending notes separately
 ✓ Re-rolled 222 tasks in 28 notes

   P1  high     85   Wed Sep 30 → Mon Oct 5     2–7 days
   P2  medium  132   Tue Oct 6 → Wed Oct 28     8–30 days
   P3  low       5   Thu Oct 29 → Sun Dec 27    31–90 days

   Next 5 weeks  ▃▅█▇▇▆▄▃▃▂▂▂▂▃▂▂▂▂▂▂▂▂▁▁▁▁▁▁▁▁▁▁▁▁▁   peak 26 · Fri Oct 2
   Notes         sase.md 130 · cash.md 14 · bob.md 12 · +25 more
   Left alone    1 next · 1 in progress
   Still due     40 P0 tasks

 ⚠ 2 tasks need a look
     sase.md:212   duplicate scheduled field
     bob.md:40     unknown priority "highest"

 ✓ Committed 4e5f6a7        bob randomize 2026-09-28: 222 tasks in 28 notes
 ✓ Pushed to origin/master

   seed 0x7f3a91c2 · undo: git -C ~/bob revert 4e5f6a7 && bob vault-sync
```

- The level table lists only levels that re-rolled at least one task. Each row shows the
  min…max of the new dates and the configured window.
- **Next 5 weeks** is a sparkline (`▁▂▃▄▅▆▇█`, scaled to the peak) of all open tasks'
  post-run `scheduled` dates for today+1 … today+35, P0 included, followed by the peak
  day. It shows before anything is written whether P1s pile up on one day.
- A line is omitted when its count is zero. "Left alone" also shows
  `N P1 (not selected)` when `--level` filtered, and `N in today's Pomodoros`.
- A dimmed "waiting for another vault maintenance run…" line prints once when the lock
  is contended.
- **Dry run:**
  - The header gets a `dry run` tag.
  - A per-note task list comes first, notes ordered by count descending then path. Each
    note shows as `sase.md (130)`, followed by one line per task:
    `P2  2026-09-10 → 2026-10-19  <description>`.
  - The summary verbs read "Would re-roll…".
  - There are no git lines.
  - It ends with
    `Nothing was written. Apply these dates with: bob randomize --seed 0x7f3a91c2`,
    adding the same `--level` and `--until` flags when given.
- **Nothing due:** `✓ Nothing to re-roll — no prioritized tasks are due by <until>.`
  plus the "Left alone", "Still due", and "Needs a look" lines. Exit 0.
- **Errors:** go to stderr as `✗ <what failed>: <why>`, followed by one actionable hint.
  Examples: "re-run with --offline to commit locally without syncing", or "another
  maintenance run held the lock for 60 s; try again".

### JSON (`--format json`)

One document on stdout, even on failure. All progress and warnings go to stderr in this
mode.

```json
{
  "schema_version": 1,
  "ok": true,
  "dry_run": false,
  "today": "2026-09-28",
  "until": "2026-09-28",
  "seed": "0x7f3a91c2",
  "levels": null,
  "summary": {
    "rerolled": 222,
    "notes": 28,
    "unchanged": 0,
    "still_due_p0": 40,
    "by_level": [
      {
        "label": "P1",
        "value": "high",
        "count": 85,
        "min_days": 2,
        "max_days": 7,
        "first": "2026-09-30",
        "last": "2026-10-05"
      }
    ]
  },
  "tasks": [
    {
      "path": "sase.md",
      "line": 17,
      "ref": "17:1a2b3c4d",
      "block_id": null,
      "description": "…",
      "level": "P2",
      "value": "medium",
      "min_days": 8,
      "max_days": 30,
      "offset_days": 21,
      "from": "2026-09-10",
      "to": "2026-10-19",
      "status_from": " ",
      "status_to": "?",
      "schedule_log": "prepended"
    }
  ],
  "skipped": [
    { "path": "bob.md", "line": 40, "reason": "unknown_priority", "detail": "highest" }
  ],
  "notes": [{ "path": "sase.md", "tasks": 130, "regrouped": true }],
  "load": [{ "date": "2026-09-29", "count": 5 }],
  "warnings": [],
  "git": {
    "mode": "sync",
    "pre_sync": { "ok": true, "files_committed": 2, "error": null },
    "commit": {
      "sha": "4e5f6a7…",
      "subject": "bob randomize 2026-09-28: 222 tasks in 28 notes",
      "paths": ["bob.md", "cash.md", "sase.md"]
    },
    "post_sync": { "ok": true, "pushed": true, "conflicts": [], "error": null }
  },
  "recovery_directory": "…",
  "error": null
}
```

Field notes:

- **`line`** and **`ref`** use the task's original (pre-edit) position.
- **`schedule_log`** is `prepended` or `created`.
- **`git.mode`** is `sync`, `offline`, `not_a_worktree`, or `dry_run`. Sections that did
  not run are `null`.
- **`error`**, on failure, is
  `{"stage": "lock|config|pre_sync|plan|apply|commit|post_sync", "message": "…"}`.

### Exit codes

- `0`: success, including "nothing to re-roll".
- `1`: runtime failure. This covers config errors, a missing Blocked status, lock
  timeout, pre-sync failure, a partial apply, commit failure, a post-sync conflict, or a
  push failure.
- `2`: usage error. This covers clap errors, an unknown `--level`, `--until` in the
  past, or an unparsable `--seed` or `--until`.

### Design decisions

- **A distinct reason head, `🎲 P2 randomize`, instead of the picker's `🎲 P2 roll`.**
  It names the tool, keeps the picker grammar that `SCHEDULE_LOG_ENTRY_RE` accepts, and
  lets bulk, unreviewed deferrals be counted later.
- **Next, In Progress, and today's Pomodoro tasks are always excluded.** They mean
  "today", and there is no flag to include them.
- **Status and grouping are composed in the same write.** A future `scheduled` date
  makes hooks flip `[?]` and regroup within 15 minutes. Leaving that to hooks would
  create a second 28-note commit, which would also break a clean `git revert`.
- **Manual only.** randomize never runs from `nightly` or cron. Silently deferring tasks
  would erase the overdue signal without review.
- **Out of scope for v1**, and not to be implemented or filed without the user asking:
  - `--balance` (least-loaded-day placement);
  - `--demote`;
  - a report of tasks rolled repeatedly;
  - an Obsidian command that previews `--dry-run --format json`.

## Phase `planner`: Pure randomize planner and shared task-field helpers

Pure code and unit tests only. No CLI registration and no disk writes. Temporary
`#![allow(dead_code)]` on new modules is fine until the command phase wires them in.

1. **New `src/native/task_fields.rs`**, registered in `native.rs`.
   - Move `SCHEDULED_FIELD_RE`, `scheduled_field_matches`, `parse_strict_calendar_date`,
     and `format_calendar_date` out of `capture_task_toggle.rs` and generalize them.
   - Provide `inline_fields(line, key) -> Vec<InlineField>`. Each field records the
     whole-field byte range, the trimmed value's byte range, and the value. Brackets and
     parens are supported; `::` spacing may vary.
   - Provide a helper for "has any of these keys".
   - `capture_task_toggle` uses the shared functions, and its behavior and tests are
     unchanged.
2. **`src/native/config.rs`**:
   - Add `PriorityProperty::levels()`, `level_for_value(&str)` (exact match after trim),
     and `level_by_label(&str)` (ASCII case-insensitive).
   - Reject duplicate level `value`s and duplicate labels at parse time with
     `ConfigError::Invalid`.
   - Make the missing-file message command-neutral:
     `priority levels need <path>; run 'chezmoi apply ~/.config/bob/config.yml'`.
   - Expose seed mixing, `mix64`, as `pub(crate)` or through a small
     `derive_seed(base, parts)` helper.
   - Keep every existing `p:<N>` capture behavior and test green, updating any assertion
     on the old message.
3. **`src/native/capture_schedule_log.rs`**:
   - `randomize_reason(label, rolled_days, min_days, max_days, from: Option<&str>)`
     produces `🎲 P2 randomize · in **21** (8–30) days`, with an optional
     ` from 2026-10-12` suffix.
   - A pure insertion planner prepends an entry line under an existing direct-child
     marker, or appends marker plus entry as the task's last direct child. It reuses
     `capture::first_direct_managed_log_start`, `first_child_indentation`,
     `dominant_indent_unit`, and `line_spans`. It returns the new contents and whether
     it prepended or created. Keep the byte-exact codepoint tests style.
4. **`src/native/task_status_hooks.rs`** (visibility only, no behavior change):
   - Make `markdown_files`, `task_group_classification`, `validate_blocked_status`, and
     `daily_anchor_date` `pub(crate)`.
   - Extract a `pub(crate)` grouping-eligibility predicate over
     `(relative_path, path, contents, daily_path)`. The existing
     `task_grouping_eligible` delegates to it, so there is one source of truth; it
     combines `note_kind`, `canonical_daily_date`, and daily-path matching.
5. **New `src/native/randomize_plan.rs`**, the pure core:
   - Inputs: note snapshots `{path, relative_path, contents}`, plus a context with
     `today`, `until`, `seed`, `&PriorityProperty`, the optional selected labels,
     `&TasksSettings`, the daily path, and today's daily contents (optional).
   - Output `Plan`:
     - per-note `NotePlan {path, relative_path, original, updated, rerolls, regrouped}`
       for changed notes only;
     - a `Reroll` list with every JSON task field;
     - a `Skip` list (reason enum plus detail);
     - `still_due_p0`, `unchanged`, and `not_selected` counts;
     - 35-day `load` data;
     - grouping warnings;
     - a `needs_blocked_status` flag.
   - Implements every rule in "What qualifies", "The roll", and "What each re-roll
     writes". It also includes the open-Pomodoro block-link scan of the daily note,
     reusing `capture_pomodoros::scan` and `pomodoro::pomodoros_section_range`.
6. **Unit tests**, colocated `#[cfg(test)]` modules:
   - **Field scanner:** bracket and paren forms, `[scheduled::2026-09-10]` with no
     space, duplicates, exact byte ranges.
   - **Config:** lookups, rejection of duplicate values and labels, the neutral message.
   - **Reason text:** default, `from` suffix, fixed window `(4–4)`.
   - **Log insertion:** prepend with tabs and with spaces, legacy marker, a marker
     nested under another child that must be ignored, created marker with and without
     other children, CRLF, no final newline.
   - **Plan, qualification:** each skip reason and ignore rule, including `^prj`, done
     or canceled with priority, future-scheduled, P0-due counting, and every `pomodoro`
     link form.
   - **Plan, writes:** `[ ]`→`[?]`; `[?]` stays; `min_days: 0` unchanged; `--until`
     shifting both cutoff and base; `--level` filtering.
   - **Plan, seeds:** stable when unrelated tasks are inserted above (line shift);
     identical lines get distinct rolls; the same inputs give the same plan.
   - **Plan, dates and notes:** date math across month, year, and leap boundaries;
     byte-exact postimage of a project note where a Ready task moves under a generated
     `### Blocked`; ordinary and daily notes edited but not regrouped; `load` counts
     include P0 and the new dates.

Done when `just all` passes and `task-status-hooks`, capture, and toggle tests are
unchanged apart from the updated config message.

## Phase `plumbing`: Lock wait, scoped commit, sync report, and writer reuse

Independent of `planner`. Keep existing behavior byte-for-byte for hooks, nightly,
vault-sync, and move-done-tasks.

1. **`src/native/ob.rs`**:
   - `acquire_lock_waiting(timeout: Duration, on_first_wait: impl FnOnce()) -> Result<File, LockWaitError>`
     polls `try_acquire_lock` with short sleeps (about 250 ms). `LockWaitError` covers
     timeout, open, and acquire failures.
   - `commit_paths(vault, child_env, message, paths) -> Result<Option<String>, String>`:
     `git add -- <paths>`, then a scoped `git diff --cached --quiet -- <paths>` (returns
     `Ok(None)` when nothing changed), then `git commit -F - -- <paths>` with the
     message on stdin, then `git rev-parse HEAD`. Commit only the listed paths, and
     leave other dirty or already-staged files untouched.
   - `detect_git_worktree`-style helper, or reuse `verify_bob_worktree`, so callers can
     tell "not a worktree" from "git missing".
2. **`src/native/vault_sync.rs`**:
   - `pub(crate) struct CycleReport { ok, exit_code, error: Option<String>, files_committed, conflicts: Vec<String>, pushed, local_sha: Option<String> }`
     and
     `pub(crate) fn run_cycle_with_existing_lock_report(child_env, quiet: bool) -> CycleReport`.
   - It runs the same cycle and status-record writing. `quiet` suppresses the
     `print_log` stdout lines and the final stderr error print, since the caller renders
     the error. Warnings stay on stderr.
   - `run_cycle_with_existing_lock` and `run_cycle` keep their exact output and exit
     codes (wrap or share the inner function).
3. **`src/native/task_status_hooks_write.rs`**:
   - Replace the hard-coded `TOOL` and `STATE_SUBDIR` uses with a `tool: &'static str`
     on `ApplySession`. Use it for the manifest `tool`, the
     `$XDG_STATE_HOME/bob-cli/<tool>/…` recovery root, retention pruning (which only
     touches manifests with the same tool), and the `bob <tool>: warning:` prefixes.
   - The hooks production session passes `"task-status-hooks"`, so paths and output are
     identical to today.
   - Document that a caller may pass an empty `scan_paths` with a rescan returning an
     empty list when note membership does not affect its plan. randomize relies on this.
4. **Tests:**
   - Lock wait acquires after release and times out while held; the callback fires
     exactly once.
   - `commit_paths` commits only its paths, leaves another dirty file and a staged file
     uncommitted, and returns `None` when nothing changed. Use tempdir git repos.
   - A writer session with tool `"randomize"` writes under `bob-cli/randomize`, and its
     pruning ignores `task-status-hooks` manifests.
   - Existing vault-sync, nightly, and hooks integration tests pass unchanged.

Done when `just all` passes.

## Phase `command`: bob randomize command, output, and integration tests

1. **New `src/native/randomize.rs`**:
   - A clap CLI exactly as in "CLI", with request parsing: `+N` or ISO `--until`, and
     decimal or `0x` `--seed`.
   - Orchestration exactly as in "Git: one scoped commit…":
     - Load config before taking the lock.
     - Human-mode lock wait line.
     - Pre-sync via `run_cycle_with_existing_lock_report(quiet = true)`.
     - Scan with `capture_required` snapshots. Warn on and skip non-UTF-8 notes.
     - Plan, and validate the Blocked status if needed.
   - Apply through `apply_plan`:
     - The inputs are only the rewritten notes' snapshots.
     - Every write sets `structural_regrouping: true`, so the 2 s quiet period applies.
     - The session uses tool `randomize`.
     - Pass an empty scan set.
     - Retry the scan, plan, and apply after `vault_changed`, `quiet_period`, or
       `unstable_read` with nothing applied, within the remaining `--retry-timeout`
       budget. The lock stays held and the seed is the same.
   - Scoped commit via `ob::commit_paths`, then the post-sync report.
   - The human and JSON renderers, and the exit codes.
   - Remove the temporary dead-code allows from the planner and plumbing phases.
2. **Register the command:**
   - `NativeCommand::Randomize` in `src/native.rs`.
   - The `SUBCOMMANDS` row and `AFTER_HELP` example in `src/runner.rs`. The sorted-table
     test must still pass.
   - `"${root}/bin/bob" randomize --help >/dev/null` in the `justfile` `install-smoke`
     recipe.
3. **New `tests/randomize.rs`** integration tests. Copy the few helpers needed from
   `tests/cli.rs` (bob command builder, temp dir, git helpers,
   `init_vault_sync_pair`-style bare remote plus peer) instead of growing that 32k-line
   file. Fixture vault:
   - a project note with a `## Tasks` section: Ready P1/P2/P3 due tasks, a due P0, a
     future P2, a Next P2, an In Progress P2, and a done P2;
   - an ordinary note;
   - a daily note with an open Pomodoro linking one due P2;
   - the Tasks `data.json` with the Blocked `?` status;
   - a `BOB_CONFIG_FILE` config.

   Pin `BOB_NOW`, `BOB_DAY_FILE`, and `--seed`. Cases:
   - Help lists options alphabetically with short aliases; `bob --help` shows
     `randomize`.
   - A dry run writes nothing, creates no commit or recovery dir, succeeds while the
     lock file is held, and prints the replay command.
   - A live run produces byte-exact postimages: date replaced, `[?]`, log prepended and
     created, block moved under `### Blocked` with an updated badge row. Excluded tasks
     are untouched.
   - Needs-a-look skips (duplicate, invalid date, unknown priority, `due`, `repeat`) are
     reported and left untouched.
   - The same `--seed` makes a dry run and a live run produce the same dates. `--level`
     filters, an unknown level exits 2, `--until +7` shifts cutoff and base, and a past
     `--until` exits 2.
   - Bare remote:
     - an unrelated pre-existing dirty file lands in a separate earlier `vault(` commit;
     - exactly one `bob randomize` commit exists, its changed paths equal the rewritten
       notes, and it was pushed (remote `master` equals local `HEAD`);
     - a peer's non-overlapping pushed change is merged before planning.
   - `--offline` commits without pushing. A non-git vault writes notes and warns. A held
     lock with `-r 1` fails with exit 1 and no writes. An unreachable remote fails the
     pre-sync with exit 1, no writes, and the `--offline` hint.
   - JSON contract: keys, `schema_version`, `git.mode` values, and the `ok`/`error`
     shape on failure.
   - Hooks parity: after a live run, `bob task-status-hooks --dry-run --format json` on
     the same vault reports no status changes or regrouping for the touched notes.
   - Nothing due exits 0. A missing config or missing Blocked status is fatal before any
     write.
   - If feasible: a post-sync conflict. Use a one-shot `post-commit` hook in the test
     vault that makes the peer push a same-line edit to a randomized note. Expect a
     conflict warning, the `_conflicts/` copy, exit 1, and a re-run that converges.

4. Run `cargo run -- randomize --dry-run` against the real vault as a read-only smoke
   check, and eyeball the output with and without `NO_COLOR`. Never do a live run.

Done when `just all` passes.

## Phase `docs`: Documentation and cross-links

Make the docs match what `bob randomize --help` and the implementation actually do; if
they disagree, the command wins.

1. **New `docs/randomize.md`**, the full contract, with these sections:
   - Usage and options.
   - What qualifies, with a skip-reason table.
   - What a re-roll writes, with before and after Markdown for one task in a project
     note.
   - Seeds and previews: the dry-run → `--seed` replay loop.
   - Recipes: a backlog after time away; `--level P2 --level P3` to keep P1s for hand
     triage; `--until +N` to clear N days for P0 work. Explain the P1 pile-up the
     sparkline reveals.
   - Git and vault-sync: the sequence, the commit message, undo, the conflict and
     offline behavior, and "run it where you edit; avoid typing in touched notes for the
     few seconds of the write".
   - Output: a human sample and the JSON schema.
   - Failure modes, environment, and exit codes.
2. **`README.md`**:
   - Add a `randomize` row to the Commands table between `query` and
     `task-status-hooks`, linking to a new `## Randomize` section.
   - Place that section after `## Projects`, with a short summary, examples, and a link
     to the guide. Add it to Contents.
   - Add a Daily workflow note: after time away, run `bob randomize --dry-run`, then
     `bob randomize --seed <seed>`.
   - Update the Environment section so `BOB_VAULT_SYNC_LOCK_FILE`, `BOB_CONFIG_FILE`,
     and `BOB_DAY_FILE` mention randomize, and add `BOB_PRIORITY_ROLL_SEED` if it is not
     documented yet.
3. **Cross-links:**
   - `docs/README.md` guide table row.
   - `docs/projects.md` "Priority property and scheduled rolls": add the
     `🎲 Px randomize` reason and a link.
   - `docs/vault-git-sync.md`: randomize as a lock holder and its sync-sandwiched scoped
     commit.
   - `docs/task-status-hooks.md`: randomize composes Blocked and grouping itself, so
     hooks has nothing left to do in those notes.

Done when the docs are accurate and consistently formatted with the existing guides.

## Verification

- Every phase: `just all` (`cargo fmt --check`,
  `cargo clippy --all-targets --all-features`, `cargo test`).
- End to end: the `tests/randomize.rs` suite, `just install-smoke`, and a read-only
  `bob randomize --dry-run` against the real vault.
