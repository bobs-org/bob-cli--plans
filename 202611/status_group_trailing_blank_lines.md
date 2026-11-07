---
tier: tale
title: Stop status grouping from growing the blank gap after Tasks
goal:
  The gap between a grouped Tasks section and the next heading stays one blank line
  across archive cycles.
size: small
decisions:
  gap_rule:
    ask:
      Cap a rewritten Tasks section's trailing gap at one blank line (vs. only
      discounting a dropped group's separator)?
    choices:
      cap:
        Stops growth and heals inflated notes on next reconcile; deliberate 2+ blank
        gaps get normalized
      targeted:
        Keeps deliberate multi-blank gaps; already-inflated notes never heal and need
        manual cleanup
    default: cap
    why: Matches the emitter's internal blank-run collapse and self-heals the vault
    answer: cap
  vault_cleanup:
    ask:
      Should the coder also hand-trim the extra blank lines in the live ~/bob notes now?
    choices:
      leave:
        Vault untouched; the first reconcile by the fixed bob on each host heals the
        gaps
      trim_now:
        Coder trims the 8 inflated notes now; gaps regrow nightly until the release
        reaches hosts
    default: leave
    why:
      The fix heals these notes itself; agent edits to the live vault risk sync
      conflicts
    answer: leave
proposed_by: bbugyi200.apollo.cw
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 14:07:25
status: wip
---

# Stop `bob task archive` + reconcile from adding a blank line before the heading after Tasks

## Problem

`~/bob/sase.md` has 13 blank lines between the end of its `## Tasks` section (the last
`### Done & Canceled` task) and `## Future Work`. In Obsidian's editing view each one is
an empty line, which shows as a large gap. The vault history shows the gap grows by
exactly one blank line each time `bob task archive` (03:30 nightly) empties a
`### Done & Canceled` group: 1 blank on 2026-10-13, then +1 on 10-15, 10-17, 10-18,
10-19, 10-20, 10-24, 10-26, 10-28, 10-29, 10-31, 11-04, 11-06. It started the day after
commit `2eb25811` ("marker-free self-pruning status groups", 2026-10-12), which added
the section-end whitespace rule.

The same bug affects every area/project note with groups. `bob.md` reached 13 blank
lines, Bryan reset it by hand to 1 on 11-02, and it is back to 5 now. `dev.md`,
`sase_blog.md`, `sase_clans_and_tribes.md`, `sase_install.md`, `sase_memory_audit.md`,
and `sase_sdd_repo.md` are at 2.

## Root cause

Three steps make the loop. Each is reasonable alone, but together they leak one blank
line per cycle:

1. **Archive leaves the separator.** `collect_done::transform_markdown`
   (`src/native/collect_done/transform.rs`) removes only the done/canceled task blocks.
   Afterwards `### Done & Canceled` is still there, followed by the blank line that
   separated it from its list, then the old gap of N blank lines before
   `## Future Work`. That is N+1 blank lines after the empty heading.
2. **Grouping keeps them all as the section's trailing gap.** On the next
   `bob task reconcile` (or `bob task reroll`), `rewrite_container`
   (`src/native/task_status_groups/mod.rs`) drops the empty group, which is correct
   because "empty groups are never emitted". Then `apply_section_end_rule`
   (`src/native/task_status_groups/emit.rs`) re-appends the original section's whole
   trailing whitespace run. That run now includes the dropped heading's separator, so
   N+1 blank lines are kept.
3. **The next close carries the bigger gap.** When a task closes again, the emitter
   writes `### Done & Canceled`, a blank line, and the tasks. It strips its own trailing
   blank lines and appends the stored N+1 run. The gap below the Done group is now one
   line longer than before.

Only bob-cli does this grouping (bob-plugins has no copy of it), so the fix belongs
entirely in bob-cli.

## Fix

> [!decision] gap_rule = cap

In `src/native/task_status_groups/emit.rs`, change `apply_section_end_rule` so the
trailing whitespace it keeps from the original section is **at most one blank line**:

- Original section ends with 0 blank lines (for example `- task\n## Notes`): keep 0.
- Original ends with 1 or more blank lines: keep only the first trailing blank line, raw
  and with its own line ending (CRLF stays CRLF).
- A section that ends at EOF without a final newline keeps that shape, as it does today.

Keep the change inside the section-end rule. `trailing_whitespace_run` can return the
capped slice, or the rule can truncate the run to its first raw line. Do not change the
emitter's body layout. Because the rule runs in each `rewrite_container`, authored topic
containers nested inside `Tasks` are covered too.

Once released, this stops the growth and heals every affected note the first time
reconcile rewrites it. A decorated container whose emitted section now differs from the
original is replaced, so sase.md goes from 13 to 1 and bob.md from 5 to 1.

> [!decision] gap_rule = targeted

Instead of capping, compute the preserved run as the original trailing run minus the
separator blank line that belonged to any empty managed group heading the rewrite drops
at the end of the section. Leave gaps that users wrote themselves untouched.
Already-inflated notes will not heal; say so in the docs change and in the final
summary.

### Non-goals

- Do not change `bob task archive`. It stays a generic task-block mover with no
  knowledge of status groups, and the docs already say archive can leave the layout
  stale until the next reconcile. After the fix, the short-lived post-archive state
  (empty `### Done & Canceled` plus two blank lines) is cleaned up on the next reconcile
  instead of being made permanent.
- Do not touch blank-line handling inside the intake or between groups. Those are
  already regenerated with exactly one separator.
- The canceled "Move runner's override to wait panel!" block in sase.md mixes 2-space
  and tab indentation. That is a separate, unrelated issue; leave it.

## Tests

Unit tests in `src/native/task_status_groups/tests.rs`:

- Keep `section_end_rule_preserves_trailing_whitespace` passing. Its one-blank,
  EOF-without-newline, and CRLF cases are unchanged.
- **Empty Done group left by archive:** a Tasks container with a Next task, then
  `### Done & Canceled`, then 3 blank lines, then `## Future Work`. The output drops the
  empty heading, leaves exactly one blank line before `## Future Work`, and is
  idempotent (use `assert_idempotent`).
- **Inflated gap heals:** a decorated container whose last group's tasks are followed by
  13 blank lines and then `## Future Work` comes out with exactly one blank line.
- **Zero gap stays zero:** `- [*] #task next\n## Notes` stays without a blank line.
- **CRLF cap:** a CRLF section ending with three `\r\n` blank lines keeps one `\r\n`
  blank line.
- **Cycle simulation:** loop 3 times. Add a `[x]` task to the intake, then run
  `grouped(...)`. Strip the Done group's task lines to mimic archive and run
  `grouped(...)` again. After every pass, assert the blank-line count before
  `## Future Work` is exactly 1.

CLI regression test in `tests/cli/task_status_hooks/sync.rs`, or a sibling file in that
module if `sync.rs` is crowded. Reuse the existing reconcile env setup (`--bob-dir`,
`BOB_DAY_FILE`, `BOB_VAULT_SYNC_LOCK_FILE`, `XDG_STATE_HOME`) and the non-repo
`bob task archive --threshold=N` pattern from `tests/cli/move_done.rs`.

1. Start with a project note: a `## Tasks` section holding one Next task and one `[x]`
   task, then `## Future Work`.
2. Run 3 cycles of `bob task reconcile`, then `bob task archive`, then
   `bob task reconcile`. Before each cycle, append a new `[x]` task to the intake.
3. After each reconcile, assert exactly one blank line between the last Tasks content
   line and `## Future Work`.

On today's code, this test fails with 2, 3, then 4 blank lines.

## Docs

`docs/task-status-hooks.md`, in the grouping section next to "Re-running on unchanged
semantic input is byte-identical": add one or two sentences on the section end. A
rewritten container keeps whether it ended with a blank line, plus its line endings and
EOF shape, but the gap before the next heading is normalized to at most one blank line.
A heading for an empty group dropped by the rewrite takes its separator with it. Run
`just docs-check`. Do not edit `CHANGELOG.md` or the `Cargo.toml` version; the release
train owns them. Use a `fix(task-status): ...` commit type.

## Vault follow-through

> [!decision] vault_cleanup = leave

Do not edit `~/bob`. In the final summary, tell Bryan two things. The inflated notes
heal on the first `bob task reconcile` run by a bob that includes this fix. Until then
the gap keeps growing nightly; deleting the blank lines by hand is harmless and stays
fixed once the fixed bob is installed.

> [!decision] vault_cleanup = trim_now

After the code change lands, edit each listed `~/bob` note so its Tasks section ends
with exactly one blank line before the next heading. Run `bob vault-sync status --json`
first and edit only when no sync is running. Change only the blank lines and nothing
else in those notes. Let the vault-sync service commit. Report that the gaps return
(about +1 per night) until the release reaches every host that runs reconcile and
archive.

## Verification

- `cargo test` for the `task_status_groups` unit tests and the new CLI test, then the
  repo's usual full gate (`just` check/test targets) before the commit.
- Run `bob task reconcile --dry-run --bob-dir ~/bob` with the built binary. It must not
  write. Confirm the preview lists `sase.md` and `bob.md` (and the other inflated notes)
  as grouping changes, and nothing else unexpected.
