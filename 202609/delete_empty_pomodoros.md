---
tier: tale
title: Delete empty Pomodoros during task-status reconciliation
goal:
  The current daily ledger is pruned of Pomodoros with no real sub-bullets during every
  task-status-hooks reconciliation.
size: medium
proposed_by: bbugyi200.athena.0m9
create_time: 2026-09-17 09:04:21
status: wip
---

# Plan: Delete empty Pomodoros during task-status reconciliation

## Goal

Extend `bob task-status-hooks` so its current-daily-note structural cleanup removes
every recognized top-level Pomodoro that has no real sub-bullets. In particular, a
duplicate placeholder such as the second `- [ ] () — GTD` in `~/bob/2026/20260917.md`
must disappear on a live run while the first GTD entry, which owns a task-link child,
remains intact.

Keep this behavior inside the existing guarded, retryable task-status-hooks write path.
`--dry-run` must report the same planned cleanup without changing the vault, and a
second live run must be byte-identical/no-op.

## Required behavior

- Apply the cleanup only to recognized, column-zero open or completed ledger entries in
  the selected current daily note's real `## Pomodoros` section. Preserve nested and
  fenced checkbox lookalikes, entries outside the section, and checkbox states that the
  existing Pomodoro parser deliberately does not recognize as ledger entries.
- Treat a Pomodoro as non-empty when it has at least one actual Markdown list-item
  child, regardless of whether that child contains a resolvable task block link. Ignore
  fenced-example bullets. Use the same direct-child/list-parent semantics as the
  existing Pomodoro scanners so capture and cleanup do not disagree about what
  constitutes a child.
- When an empty entry owns indented continuation content but no list-item child, remove
  its whole entry block so cleanup cannot leave orphaned indented text. Keep surrounding
  section content, line-ending style (including CRLF), and final-newline state stable.
- Empty entries must not count toward the multiple-open-timed-Pomodoro guard and must
  not be selected as destinations for completed-reference relocation. Multiple genuinely
  non-empty open timed entries must continue to fail before any write.
- Run empty-entry pruning after duplicate-line removal, canceled-reference subtree
  removal, completed-reference retirement/relocation, and marker repair have been
  composed. This must remove entries that start empty as well as entries whose last
  child is removed or moved away during the same invocation, while retaining an entry
  that receives a moved child in that invocation.
- Rescan the fully normalized daily contents before building direct desired statuses,
  dependency reachability, and recent-activity state. Removed entries and their removed
  content must never influence status transitions.
- Preserve the existing input-count meaning of compatibility fields such as
  `open_pomodoros` and `references`; expose empty-entry deletion through a new,
  deterministic result collection instead of silently changing those fields.

## Implementation

1. In `src/native/task_status_hooks.rs`, enrich the Pomodoro model or share the existing
   `capture_pomodoros` scanning primitives so each recognized entry records whether it
   has a real direct child list item. Keep the current fence, section, and
   top-level-entry exclusions aligned across scanners.
2. Adjust the timed-open ambiguity check and structural relocation target selection to
   consider only entries that are non-empty before structural reconciliation. This lets
   an empty timed entry be pruned instead of causing a false ambiguity or receiving work
   merely because it would otherwise be the current/fallback target.
3. Add a deterministic post-structural empty-Pomodoro cleanup plan. Build it from the
   daily text produced by the existing structural plan, correlate entries back to the
   original scan in document order for stable reporting, remove each empty entry's full
   block, and then use that final text for the existing post-rewrite Pomodoro rescan.
   Apply this same two-stage transformation in `compose_outputs` after daily checkbox
   edits so dry-run analysis and guarded live output cannot diverge.
4. Add a serializable `removed_empty_pomodoros` result item containing the original
   one-based line number and Pomodoro line text. Thread it through `SyncResult`, the
   retry-test empty result helper, change/no-op detection, a dedicated human
   `would remove`/`removed empty Pomodoros` section, and the summary count. Keep
   ordering in source-document order and ensure one report item per removed entry.
5. Keep all writes flowing through the existing `ComposedOutput` and
   `apply_guarded_outputs` machinery; do not introduce a direct daily-note write or a
   separate lock/recovery path.

## Tests

Add focused unit coverage in `src/native/task_status_hooks.rs` for the scanner and
two-stage rewrite:

- open placeholder, open timed, and completed entries without children are removed;
- a child bullet without a link keeps its parent, while fenced and nested lookalikes do
  not falsely make an entry non-empty;
- full entry-block deletion preserves neighboring content, CRLF, and absence of a final
  newline;
- an entry emptied by canceled/duplicate cleanup or by moving its last child is removed
  in the same pass, while a relocation destination that gains a child is retained;
- empty timed entries are excluded from relocation and ambiguity decisions, but two
  non-empty timed entries still trigger the guard.

Add or extend CLI integration coverage in `tests/cli.rs` with a temporary daily note
that mirrors the real regression: two `- [ ] () — GTD` entries, only the first with a
child, plus representative completed/timed entries. Assert that:

- JSON dry-run leaves bytes untouched and reports the second GTD in
  `removed_empty_pomodoros` with its original line number/text;
- human dry-run uses `would remove`, and a live run uses `removed`, includes the new
  summary count, and removes only the empty entry;
- the live result uses the normal guarded write/recovery reporting and the second run
  reports `already in sync`;
- the command still rejects multiple non-empty open timed entries, while empty timed
  entries no longer cause the ambiguity guard.

Audit existing task-status-hooks integration fixtures that use childless Pomodoros as
neutral setup. Where a test is intended to remain a write-free/no-op or to exercise a
guard unrelated to this feature, give the entry a harmless non-link child bullet; where
deletion is now part of the intended outcome, update the expected daily bytes, JSON
arrays, human summaries, and applied-file/recovery assertions explicitly. Do not weaken
unrelated status, retry, or concurrency assertions.

Run targeted tests first, then the repository checks:

```bash
cargo test task_status_hooks
cargo test --test cli task_status_hooks
just all
```

## Documentation

- Update the `task-status-hooks` long help in `src/native/task_status_hooks.rs` and the
  overview in `README.md` to include removal of childless current-daily Pomodoros.
- Add an Empty Pomodoros rule to `docs/task-status-hooks.md` describing scope,
  child-list semantics, cleanup ordering, ambiguity behavior, dry-run/live behavior, and
  idempotence.
- Extend the documented human and JSON output contracts with `removed_empty_pomodoros`,
  including the original-line-number guarantee and the new summary count.

## Acceptance criteria

- With the 2026-09-17 ledger shape described above, only the childless second GTD entry
  is selected for deletion; the first GTD and every other entry with a child remain.
- Dry-run, live application, retries, guarded snapshots, and recovery copies all use one
  identical composed cleanup result.
- A successful rerun is a no-op, and all formatting, lint, unit, and integration checks
  pass.
