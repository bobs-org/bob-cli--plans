---
tier: tale
size: medium
title:
  Finish the Gmail reply-watch landing — planner row identity, line endings, and
  per-target receipts
goal:
  Watched Gmail effects hit the right task row even when the same run files new rows,
  keep task-file line endings byte-identical, and never rewrite already-applied targets
  on a catch-up retry; then epic bob-cli-ii.4.1 and its parent phase bob-cli-ii.4 are
  closed.
proposed_by: bbugyi200.athena.bob-cli-ii.4.1.land
proposed_with: claude/opus
bead: bob-cli-ii.4.1
create_time: 2026-11-08 16:46:34
status: wip
---

- **PARENT:**
  [202611/gmail_reply_watch.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_reply_watch.md)
- **BEAD:**
  [bob-cli-ii.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ii/bob-cli-ii.4.1.md)

# Finish the Gmail reply-watch landing (bob-cli-ii.4.1)

## Why this exists

Epic `bob-cli-ii.4.1` (plan `202611/gmail_reply_watch.md`) has all six phases closed
(`bob-cli-ii.4.1.1`–`.6`, commits `b4147b0b`, `86af3d5e`, `9e0bcdd8`, `457153f0`,
`5a63e9a2`, `b252669b`). Its land agent re-verified the work and already landed these
integration edits (committed before this tale starts):

- fixed the 8 `clippy::collapsible_if` errors the epic introduced in
  `src/native/gmail/sync/file.rs` and `watch_apply.rs`, so `just lint` passes;
- removed the stale watch dead-code allowances and deleted the now-dead
  `EnrollmentMembership::watched`, `WatchScan::watcher_count`, `WatchingTask::watches`,
  and `hold::configured_nudge_for_send`;
- finished follow-up triage and recorded it on the epic bead. Do not redo it.

The review also confirmed three defects that this epic caused. Scratch tests against
`plan_watch_edits` reproduced each one. They are epic work and must be fixed before the
epic closes. This tale fixes them and then performs the whole landing closeout.

Read the epic plan's "Shared implementation contract" through
`sase bead read bob-cli-ii.4.1 -r "<why>"` (it prints the PLAN path) before editing. The
fixes below must honor it: one coalesced scoped commit, no task-line bookkeeping, never
write at a stale line or a guessed row, markers only after durable writes, and body-free
recovery evidence that stays backward-compatible.

## Defect A — a same-run filing insert misplaces watched rows and loses the turn

`build_attempt` in `file.rs` scans watchers from disk under the lock, then calls
`watch_apply::plan_watch_edits` with `filing_tasks`. The planner's buffer for a filing
file starts from the filing snapshot's `updated` text, but existing watchers keep their
**pre-filing disk line numbers**. `capture::insert_task_line` inserts new rows _above_
grouped subsections such as `### Next & In Progress` (pinned by a `file.rs` test), so
every existing row below the insert shifts.

Reproduction: an area note `job.md` contains

```
## Tasks

### Next & In Progress
- [*] #task Old [[mail/aa01|M]] [created:: 2026-11-01]

### Done & Canceled
```

In the same run, a new thread `bb02` is filed into `job.md`, and `aa01` has a new
outgoing move. Result: the planner emits **no edit**, still emits an `aa01` marker
receipt, and reports `held` with an empty `tasks` list. The hold is lost and the turn is
consumed. If the shifted index lands on another `#task` row, `line_matches_task` (which
accepts any line containing `#task` when the block id is absent from it) would write the
effect to the **wrong task**. `resolve_live_task` returning `None` for a blank or
relocated line has the same consume-instead-of-defer problem, because the caller treats
`None` as closed.

Required fix, in `watch_apply.rs`:

- Resolve every target row in the working buffer by its exact scanned text
  (`WatchingTask.text`, which excludes the line ending). Use the scanned line when its
  text matches exactly. Otherwise use the unique line in the buffer whose text matches
  exactly; filing only inserts lines, so unchanged rows keep their text. Anything else
  is unresolved.
- Make resolution three-way: **closed** (an open-status check fails on the matched row:
  consume as today), **live at line N**, or **unresolved/ambiguous**. An unresolved row
  defers its threads: add them to `deferred_threads`, so the marker does not advance and
  the pending evidence is retained. Never write to it. Delete the loose
  `line_matches_task`/`relocate_by_block`-then-`Missing` paths these replace.
- Use the resolved line for the edit, for `WatchTaskRef.line` (the 1-based line in the
  file as written), and when refreshing `plan.snapshot`. Watching and awaiting counts
  come from that snapshot.
- Remove the dead no-op loop `for buffer in buffers.values() { … let _ = buffer; }` in
  `plan_watch_edits`.

## Defect B — watch edits rewrite every CRLF in a task file to LF

`read_line`/`replace_line` in `watch_apply.rs` and the row loop in
`file.rs::apply_born_held` split with `str::lines()` and rejoin with `"\n"`. That strips
every `\r`, so one hold rewrites the whole note's line endings. Reproduced: a 3-line
CRLF `job.md` came back with zero `\r\n`. The comment in `replace_line` claiming the
writers carry the CR is wrong.

Required fix: split with `split_inclusive('\n')` plus `markdown::split_line_ending`, as
`scan.rs::scan_note` does. Hand the pure `hold::apply_hold`/`apply_lift` the content
without its terminator, and re-attach that line's original terminator. Every other byte,
including a missing final newline, stays identical. Share one small helper between
`watch_apply.rs` and `apply_born_held` rather than writing two.

## Defect C — retries after a partial application rewrite already-applied targets

The plan requires that retries "finish deferred targets without repeatedly changing
already-applied targets or losing their subsequent hand edits". `RecoveryWatch` is
thread-level only (`pending_tasks`/`pending_marker`). After a running-session deferral
(or a Defect-A deferral) on one target, the next run re-plans the same turn for
**every** target. Reproduced: two rows on one thread, `^idle` already held by run 1,
Bryan then removes `#awaiting_reply` from `^idle` by hand, and run 2 (session closed)
re-adds it. For a lift, the retry would also re-stamp an already-lifted Ready row's
`[unblocked::]` with a later day. The existing
`two_tasks_on_one_thread_only_running_defers_and_catches_up` test misses this because it
plans run 2 from unchanged disk bytes.

Required fix:

- Add `#[serde(default)] applied_targets: Vec<…>` to `RecoveryWatch`, with a small
  body-free serde struct: vault-relative path plus `block_id` when present, else the
  1-based line. Never a title or task text. Older evidence must load with an empty list.
- When a turn stays pending, `plan_watch_edits` records every target it applied, or
  found already correct, for that turn. On a retry of the **same** turn (same message id
  and internal date), it skips targets listed in `applied_targets` (match path + block
  id when the entry has one, else path + line) and applies only the rest. A newer turn
  supersedes the entry and its `applied_targets`. A target that closed or stopped
  linking is no longer an obligation. When nothing remains deferred, the marker advances
  and the pending entry clears.
- `file.rs::dedup_watch_pending` and the pending-carry paths in `persist_attempt` must
  merge `applied_targets` for the same turn rather than drop them. The marker-only retry
  path (`pending_tasks == false`) keeps its current behavior.
- Send hook: `compose/send_watch.rs::plan_under_lock` passes `&[]` as pending and
  discards `plan.pending`. A running deferral during send therefore makes the next sync
  re-hold every target, including ones the send already held. Pass the account's stored
  watch pending into `plan_for_send`. Persist the resulting `plan.pending` (with
  `applied_targets`) into the account's host-local sync recovery evidence through one
  narrow helper exported from `file.rs`, called under the send's existing maintenance
  lock and after its task writes are durable. It must be idempotent across
  complete-state retries and must never trigger a second Gmail send. The next sync must
  pick that evidence up on its normal recovery path, including watch-only and `-l 0`
  runs.

## Tests

Add focused tests next to the existing ones. Use fake gog and temporary vaults only.

- `watch_apply.rs` unit tests:
  - Defect A: the grouped-area scenario above. The existing `Old` row is held, the
    receipt lists it with its post-insert line, and the new row is unaffected unless its
    own thread holds it.
  - Defect A: a shifted index that lands on another `#task` row writes nothing to that
    row and defers the thread.
  - Defect A: an ambiguous duplicate row defers instead of consuming.
  - Defect B: a CRLF file through hold and through lift, with only the target line's
    content changed and the `\r\n` count unchanged.
  - Defect C: a same-turn retry skips applied targets and preserves a hand edit; a newer
    turn drops `applied_targets`; old evidence without the field deserializes.
- `file.rs` transaction test: run 1 has two tasks on one thread, one running, so it
  holds `^idle` and writes pending with `applied_targets`. Hand-edit `^idle`. Run 2
  (session closed) holds only `^busy`, the edit survives, the marker advances, and
  pending clears. Add the same-run filing-plus-hold scenario as one scoped commit, and
  born-held on a CRLF inbox.
- `tests/gmail_sync.rs`: an end-to-end run where a new thread is filed into an area note
  that already holds a watched Next row whose thread got a reply hold in the same run.
- `tests/gmail_send.rs`: a send with two watchers, one running. `^idle` is held; edit
  `^idle`; the following sync holds `^busy` only and preserves the edit; exactly one
  Gmail send occurs.

Extend `docs/gmail.md` "Running sessions" by one sentence: catching up after a deferral
never rewrites targets the earlier run already applied. Run `just docs-check`.

## Verification

Run `cargo fmt --all`, `just lint`, `cargo test --lib gmail::`,
`cargo test --test gmail_sync`, `cargo test --test gmail_send`, `just docs-check`, then
the canonical `just check`. Do not run `just check-full`.

`native::gmail::sync::guard::tests::real_lock_contention_and_release` is a known
pre-existing flake. It reproduces on the pre-watch base `23217c1d` and is already
recorded as a DISCOVERED ISSUE on `bob-cli-ii`. If it is the only `just check` failure,
rerun it alone, cite it in the close note, and do not fix it here.

Do not build a second checkout of this repo against the same cargo target directory.
During landing review that overwrote this crate's artifacts and produced stale test
binaries; `cargo clean -p bob-obsidian` recovers.

## Closeout (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-ii.4.1` and
   `sase bead epic-symbols bob-cli-ii.4`. Both were empty at landing review. Resolve any
   entry that has since appeared (wire it up, privatize, add a non-test pragma, or
   delete it), or re-key it to a genuinely still-open bead.
2. Close the epic:
   `sase bead close bob-cli-ii.4.1 --note "<verification: phases 4.1.1–4.1.6 reviewed against the plan; land fixes for collapsible_if lint and dead-code allowances; this tale's Defect A/B/C fixes and tests; suites and just check results; flake note>"`.
3. Run `just symvision` if the recipe exists (it was absent from the `justfile` at
   landing review; say so in the note if it still is).
4. Set `status: done` in the frontmatter of the epic's plan file, at the PLAN path
   printed by `sase bead read bob-cli-ii.4.1 -r "<why>"` (`202611/gmail_reply_watch.md`
   in the plans repo).
5. Parent `bob-cli-ii.4` is a **phase** bead ("watch: fetch watched threads every run,
   refresh their mail notes, and apply each new turn to every open linking task, holding
   on your send and lifting with an `[unblocked:: today]` return on their reply,
   including the send finalize hook and the running-session guard"). Verify the closed
   child epic delivered that scope. Then close only that phase:
   `sase bead close bob-cli-ii.4 --note "<what you verified>"`. Do not close
   `bob-cli-ii` or any other ancestor; its own land agent is waiting.

Never use `--force` merely to make a close succeed.
