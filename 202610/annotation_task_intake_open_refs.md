---
tier: tale
size: small
title: Import Highlights annotation tasks from every open reference
goal:
  "Annotation #task bullets on ready, next, or wip references become Obsidian tasks,
  including the run that closes them."
decisions:
  decision_record:
    ask:
      Add a decisions memory record saying annotation-task intake follows every open
      reference status?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.8d
decided_by: auto
create_time: 2026-10-16 18:40:00
status: wip
---

# Plan: Import Highlights Annotation Tasks From Every Open Reference

## Context

Bryan left a highlight comment in Highlights on
`lib/blogs/gas_town_emergency_user_manual.pdf`:

```md
- #task Should sase run like claude code with the main agent(s) being the "mayor(s)" ...
```

The sidecar has the bullet, and the rendered reference note shows it, but no Obsidian
task was created. `bob ref sync --dry-run` on that PDF reports
`annotation_tasks_create: 0`.

**Root cause.** `src/native/highlights_ref/sync.rs` only builds annotation-task
candidates when the merged marker/frontmatter projection is `status: wip`, either before
or after the reading-task signal. This applies on both the v1 path
(`intake_allowed_before_signal`, ~line 318) and the v2 path (`pre_signal_wip`, ~line
557). This reference is `status: ready` (its reading task in `sase.md` is `[ ]`), so
intake is skipped.

**Why the gate is now wrong.**

- Bryan asked for the `wip` gate in June 2026 (commit `7a5bee39`, prompt
  `highlight_task_routing_processed_index`) "to make sure that no duplicate Obsidian
  tasks ever get created". The same change added the durable processed ID, `[h:: …]`.
  Together with the vault-wide `processed_task_index` (live, done, and archived `done/`
  tasks, plus legacy `^ht` and `[highlight_task::]` recognition), that ID is what
  actually prevents duplicates. The status gate is only a backstop.
- Under the sticky-lanes model (`decisions:task-lanes-are-sticky`), `[/]`/`wip` is the
  PENDING lane, set by a Pomodoro `=x` outcome. It no longer means "I have started
  reading". A blog read in one sitting, or any reference annotated before an `=x` close,
  never reaches `wip`.
- Closing such a reference loses the work silently and permanently. Going straight from
  `ready`/`next` to `read` has a non-`wip` status both before and after the signal, so
  the pending `#task` bullets are never imported unless the reference is later reopened
  to `[/]`.

**Blast radius.** Only one open (`ready`/`next`/`wip`) reference in the vault has
`#task` bullets: this one. The four other sidecars with `#task` bullets belong to `read`
references, which stay excluded. Widening the gate creates exactly one task on the next
scan.

## Design

Gate intake on "the reference is open", not on "the reference is `wip`":

- **Open** = `ready`, `next`, or `wip`. Blocked `[?]` keeps the selected status, so a
  blocked open reference stays open.
- **Allow intake** when the merged projection is open **before** the reading-task signal
  (this keeps the closing run's import, now also for `ready → read` and `next → read`).
  Also allow it when the projection is open **after** the signal (this keeps a reopen
  importing in the same run, now also for a reopen to `ready`/`next`).
- **Still skip** runs whose status is terminal both before and after the signal: `read`,
  `abandoned`, `legacy`. That keeps the backstop for the finished-library case the June
  gate protected, and keeps the vault-wide `processed_task_index` walk off for finished
  references.
- Dedup, routing (`@name`), v2 residence filing, `[h::]` IDs, and the `🔖` backlinks are
  unchanged.

## Changes

1. **`src/native/highlights_ref/annotation_tasks.rs`.** Add
   `pub(super) fn projection_status_is_open(projection: &Projection) -> bool`, true for
   `STATUS_READY | STATUS_NEXT | STATUS_WIP`. Put it next to `projection_status_is` and
   reuse `status_value_is` or the `as_string` match style already there.
2. **`src/native/highlights_ref/sync.rs`.**
   - v1 path (~lines 314–328): compute both `intake_allowed_before_signal` and the
     post-signal check with `projection_status_is_open`. Rewrite the comment: closing
     runs import from any open status, and a reopen to any open status imports in the
     same run.
   - v2 path (~lines 553–587): rename `pre_signal_wip` → `pre_signal_open`, switch both
     checks to `projection_status_is_open`, and update both comments the same way.
   - `finalize_annotation_task_plans` comment (~line 1082): "non-`wip` PDFs" → "terminal
     (`read`, `abandoned`, `legacy`) PDFs".
   - Keep `STATUS_WIP` uses that are not about intake untouched.
3. **Tests.**
   - `tests/cli/highlights/sync_tasks.rs`: rename
     `highlights_ref_sync_skips_annotation_tasks_for_non_wip_statuses` →
     `highlights_ref_sync_skips_annotation_tasks_for_terminal_statuses`. Narrow its loop
     to `["read", "abandoned", "legacy"]`.
   - Same file: add `highlights_ref_sync_creates_annotation_tasks_for_open_statuses`,
     looping over `["ready", "next"]`. Model it on
     `highlights_ref_sync_creates_tasks_from_pdf_note_task_bullets`: seed the v1 `^ref`
     line with the matching mark (`[ ]` / `[*]`) via `seed_v1_ref_note`, and assert the
     `- [ ] #task …` line with its `🔖` backlink and `[h:: …]`. Also assert that a
     second sync does not duplicate it.
   - `tests/cli/highlights/tasks.rs`: add a closing-run regression next to the existing
     `wip` closing-order test (~line 595). Start from `status: ready` with a `[ ]`
     `^ref` seed, add a `#task` bullet, check the `^ref` task `[x]`, sync, and assert
     the task was imported and the reference is now `read`. Then confirm a further sync
     on the now-`read` reference with a newly added bullet does **not** import it.
   - `tests/cli/highlights/scan_integration.rs`: add a v2 case beside
     `v2_dirty_parent_accepts_default_follow_up`. A `status: ready` PDF whose reading
     task lives in an area note files its default `#task` follow-up into that parent
     note with the full-path `[[ref/…#^h-…|🔖]]` link.
   - Update any other test that asserted no intake for `ready`/`next` (search for the
     old test name and for `annotation_tasks_create: 0` with a `ready`/`next` marker).
4. **Docs.**
   - `docs/highlights-ref-sync.md` ~lines 1064–1071: replace the `wip`-only wording.
     Annotation tasks are imported while the marker/frontmatter-selected status is open
     (`ready`, `next`, or `wip`), before the reading-task checkbox contributes another
     status. A run that closes an open reference to `read`/`abandoned` still imports
     newly added bullets in that run, and reopening a terminal reference imports them
     too. Runs whose status is terminal before and after skip intake.
   - Same file ~line 1365 (the "sync model is deliberately asymmetric" bullets): the
     claim that tasks are created "in the reference note or an explicitly routed
     existing root note only" is stale for v2. Reword it: v1 notes use the reference
     note, v2 uses the reading task's residence (or `mac_inbox`), and explicit `@name`
     routes use that root note.
   - Grep `README.md` and `docs/` for any other `wip`-only intake wording, and fix any
     found.

## Verification

- `just check`, or `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
  `cargo test --no-fail-fast`.
- Read-only check against the real vault with the dev build:
  `cargo run -q -- ref sync --dry-run ~/bob/lib/blogs/gas_town_emergency_user_manual.pdf`
  must report `annotation_tasks_create: 1`, filed into `sase.md` (the reading task's
  residence). `cargo run -q -- ref scan --dry-run` must show no other new annotation
  tasks. Do **not** run a writing sync/scan against `~/bob`; Bryan or the next scheduled
  scan imports it.

## Memory Record

> [!decision] decision_record
>
> Use `/sase_memory_write`, then add the
> `decisions:annotation-tasks-import-from-open-refs` strand (Applies to: bob-cli. Claim:
> annotation `#task` bullets import while the reference is open or on the run that
> closes or reopens it; terminal references never import. Why: `[h::]` dedup supersedes
> the June `wip` gate, and `wip` became the sticky PENDING lane. Evidence: commit
> `7a5bee39`, prompt `highlight_task_routing_processed_index`,
> `decisions:task-lanes-are-sticky`, and this change's commit), then run
> `sase memory init`.
