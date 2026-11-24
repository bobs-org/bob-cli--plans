---
tier: tale
title: Observe Bob-filed Gmail threads open at birth and explain paused archive gates
goal:
  Closing a freshly filed Gmail Task before the next sync archives it normally, and any
  paused archive gate tells tick what to run.
size: medium
decisions:
  repeat_pause_exit:
    ask:
      Should an unchanged paused archive gate stop failing the sync after the first run
      that sees it?
    choices:
      every_run:
        Every paused run exits 1 as documented; cron mails each Gmail tick until
        confirmed
      first_sighting:
        Only a new pause digest exits 1; repeats report in JSON and exit 0, mailing once
    default: every_run
    why:
      A pause blocks all auto-archiving and becomes rare after the fix, so repeating is
      acceptable
    answer: every_run
proposed_by: bbugyi200.athena.1fu
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-24 16:03:39
status: wip
---

# Gmail archive: observe Bob-filed threads open at birth, and explain paused gates

## Problem

The Mac's `bob tick` Gmail step failed on every hourly run from 08:05 to 11:05 on
2026-11-24. Each run sent a cron email that said only:

```
bob tick: warning: failed
  ✗ gmail     exit 1
      │ failed
```

Reconstructed from the vault's Gmail journal
(`.bob/gmail/kellys-macbook-pro-local.jsonl`) and vault git history:

1. 07:54: the daily-note Gmail sync that Ledger Tools runs
   (`bob gmail sync -q -f json -r 30`) filed a new thread ("Yogi - Senior SWE -
   Technical Screen") into `job.md` as an open `- [ ]` Gmail Task. That run journaled
   `gmail.save` and `gmail.filed` but no `gmail.archive_observed` for the thread.
2. By about 07:58 Bryan had cancelled the task (`[-]`).
3. 08:05: the next sync was the first to look at the thread for archiving, and it found
   the thread already closed. `ArchiveJournal::project` classifies a close with no open
   observation in the current epoch as `HistoricalBacklog`. That marks the plan row
   `historical: true`, and `archive_preview::apply_gate` pauses the whole batch with
   `backlog_unconfirmed` until a matching `-c DIGEST` arrives.
4. The live paused branch in `archive_apply::apply_live` pushes `paused` rows and sets
   `outcome.failed = true` without adding any warning. `SyncReport::attach_live_archive`
   then sets the untyped failure kind `failed`. Tick's Gmail failure detail is the first
   `failure_kinds` entry plus the first warning (docs/tick.md), so all it could show was
   `failed`.
5. This repeated every hourly sync until Bryan un-cancelled the task at 11:50. The 12:05
   sync then observed it open, and a later normal close archived it at 13:05.

### Root cause

The thread could not be observed open in the same run that filed it. `command.rs`
freezes a pre-hydration `SavedIdSnapshot` before filing ("IDs introduced by this run
stay unsaved until the next sync"). `apply_live` applies that snapshot with
`scan.with_saved_id_snapshot(...)` before it calls `ensure_baseline`.
`with_saved_id_snapshot` clears `saved_ids` for any thread missing from the snapshot,
which includes every thread filed in this run. `ArchiveJournal::event_for_open` →
`valid_observation_evidence` → `valid_ids` then rejects the empty set, so no open
observation is written.

As a result, every Bob-filed thread has an open-observation blind spot that lasts until
the next live sync (up to an hour on tick's cadence). Closing its task inside that
window, which is common when triaging fresh mail such as cron-failure emails:

- pauses all automatic archiving, not just that thread's, because the gate is
  batch-wide;
- fails every Gmail tick, and therefore sends a cron email every hour;
- gives no actionable message.

Why the fix is safe: the saved-ID freeze exists so that a **close generation** never
archives message IDs that appeared during the same run. An **open observation** stores
no saved IDs. Its journal value is `account`, `thread_id`, `enable_epoch`, `state`,
`observation_id`, `after_generation`, and `tasks`; see `event_for_open` in
`src/native/gmail/sync/archive_state.rs`. Its saved-ID check is only a
validity/completeness check. Recording the open observation from the unfrozen,
post-filing scan therefore cannot widen any archive set. The later close still freezes
IDs from that run's own pre-hydration snapshot, which includes the IDs saved at filing.

## Changes

### 1. Record open observations from the post-filing scan (root-cause fix)

In `src/native/gmail/sync/archive_apply.rs` `apply_live`, on the modify-grant path:

- Keep the raw `scan_archive_evidence(...)` result, from before
  `.with_saved_id_snapshot(...)`, and pass that raw scan to `ensure_baseline`.
  Everything else (the `has_modify_grant` archivable check, the preview,
  `persist_closes`, terminal planning, let-go) keeps using the snapshotted scan.
- When `ensure_baseline` appends and the function re-reads enrollment and re-scans,
  re-apply the snapshot to the fresh scan for planning as it does today.
- Add a comment at the `ensure_baseline` call explaining that open observations carry no
  saved IDs, so threads filed in this run are observed open at birth. Without this, a
  task closed before the next sync would be misread as historical backlog.
- Do not change `valid_observation_evidence`, `event_for_open`, the journal schema,
  `with_saved_id_snapshot`, or the dry-run path (`-d` never writes the journal).
- Check that the filing run still exits 0 and adds no new human or JSON archive rows.
  The newly observed thread is open, so it should project as a watching hold, the normal
  state for open threads.

### 2. Make a paused archive gate actionable

In `archive_apply.rs`, every live branch that pauses the batch must push one body-free
warning. That means both the modify-grant branch and the read-only (`!has_modify_grant`)
branch, for `backlog_unconfirmed` and `over_limit`. Use a shared helper with a stable
shape, for example:

```
Gmail archive paused (backlog_unconfirmed): 1 closed thread needs confirmation; preview with `bob gmail sync -d`, then confirm with `bob gmail sync -c <digest>`
```

- Include the gate reason, the count of paused rows, and the plan digest when one
  exists, so the cron email can be copied and run as is. Use thread IDs or counts only,
  never subjects or task text; journals and reports must stay body-free.
- The `confirm_mismatch` branch already warns. Leave its text alone.
- Make sure the new warning does not start with any prefix that `parse_warning_kind` in
  `src/native/gmail/sync/report.rs` recognizes, so it never adds a bogus kind.

In `src/native/gmail/sync/report.rs`:

- `attach_live_archive`: when any row is `paused`, insert the typed failure kind
  `archive_paused` instead of the generic `failed` fallback. Keep `deferred` as is. Keep
  the `failed` fallback for other untyped causes when the set is still empty.
- `attach_archive_preview` (dry-run pause): use the same `archive_paused` kind and add
  the same warning through the shared helper, so `-d` and live runs agree.
- Exit codes do not change (see the `repeat_pause_exit` decision).

Tick needs no code change. `child_detail` already renders the first `failure_kinds`
entry plus the first warning, so the cron email will read `archive_paused` followed by
the pause warning. `archive_paused` is not in tick's transient set, so it still reports
`failed`.

> [!decision] repeat_pause_exit = first_sighting Persist the last paused-gate digest
> that failed a run in Gmail sync health state (`sync-state.json`, next to
> `archive_review_seen`, through `health.rs`'s existing state read/write). A live run
> whose paused digest equals the stored one still reports its rows, warning, digest,
> `ok: false`, and `failure_kinds: ["archive_paused"]` in JSON, but returns exit 0. A
> new digest, or a pause after a non-paused run, exits 1 and replaces the stored digest.
> Clear the stored digest on any run whose gate is open. Update docs/gmail.md ("A pause
> ... exits 1") and docs/tick.md to describe the first-sighting rule. Add an integration
> test that runs the same paused backlog twice: exit 1, then exit 0 with the same JSON
> rows. Then change the task status and assert that the new digest exits 1 again.

### 3. Docs

Before editing docs, read the `docs.md` reference memory with `/sase_memory_read`.

- `docs/gmail.md`, "Archive on close": after the baseline sentences, state that a live
  sync records an open observation for every thread it files in that same run, so
  closing a freshly filed task before the next sync still archives automatically.
  Historical backlog then means only threads that were already closed when the archive
  phase first saw them, for example after first enablement, an `archive_label` rename,
  or a manual `bob gmail save` whose task closed before any live sync ran. In the
  `-d`/`-c` paragraph, say that a pause also emits one warning naming the gate reason,
  the paused-thread count, and the confirm command with the digest.
- `docs/gmail.md`, sync JSON paragraph: add `archive_paused` (any paused archive gate)
  to the `failure_kinds` list.
- `docs/reference/json-outputs.md`: only if its `failure_kinds` text lists kinds
  explicitly (it currently does not). Otherwise leave it.
- `docs/tick.md`: optionally add a one-line example of a paused Gmail failure detail
  next to "A Gmail failure carries the first `failure_kinds` entry plus the first
  warnings line."

Run the repo's docs checks (`just docs-check`) after editing.

## Tests

- **New integration test** in `tests/gmail_archive_apply.rs`, for example
  `same_run_filing_is_observed_open_and_a_quick_close_archives`. Use an archive-enabled,
  modify-grant `FakeGog` env where `bob gmail sync` itself files a new enrolled thread
  into an open task. Reuse the routed or inbox filing setup from `tests/gmail_sync.rs`
  (for example around `sync_routes_to_the_matching_area_note`), not the `save` + `-l 0`
  adopt helper.
  - Run 1: assert exit 0, a `gmail.filed` event, and a `gmail.archive_observed` event
    with `state: open` for that thread in the journal.
  - Close the task (`[x]` and, separately or parameterized, `[-]`).
  - Run 2: assert exit 0, `archive[0].action == "archived"`, no `paused` rows, and no
    `archive_digest`.
  - Show that the test fails without change 1. Before the fix, run 2 pauses with
    `backlog_unconfirmed`.
- **Extend** `live_pauses_backlog_without_confirm_and_creates_no_labels`. That
  historical-backlog path still applies when observations were skipped with `-n`. Assert
  `failure_kinds == ["archive_paused"]` and that `warnings` contains the pause warning
  with the gate reason, the count, and the same digest as `archive_digest`. The existing
  `-c` confirmation must still archive.
- Add a similar assertion for the `over_limit` pause if a test already covers it, and
  for the dry-run pause in `tests/gmail_archive_preview.rs`
  (`dry_run_pauses_backlog_and_confirms_with_the_digest`).
- **Unit tests** in `report.rs` for `attach_live_archive` and `attach_archive_preview`:
  a paused plan yields `archive_paused`, other untyped failures still yield `failed`,
  and a deferral keeps `deferred`.
- **Tick adapter unit test** in `src/native/tick/adapters.rs` `mod tests`: a Gmail
  document with `ok: false`, `failure_kinds: ["archive_paused"]`, and the pause warning
  yields `Outcome::Failed` with detail `archive_paused\n<warning>`.
- Existing archive-apply, archive-preview, and Gmail sync suites must stay green,
  especially the tests that rely on historical backlog after `-n`, first enablement, and
  label rename (`archive_label_rename_rebaselines_and_keeps_old_receipt_identity`).

Validate with `just fmt`, `just lint`, and `just test` (or `just check`).

## Commit

Use a non-breaking `fix(gmail): ...` conventional commit. The new `failure_kinds` value
is additive. Never edit `Cargo.toml`'s version or `CHANGELOG.md`; the daily release
train owns both.

## Out of scope

- Manual `bob gmail save` followed by closing the task before any live sync still
  produces historical backlog. Saves do not run the modify-grant archive baseline. The
  docs change above names this case, and the new warning makes it actionable. Fixing it
  would need the save path to probe the modify grant and write observations, which is a
  separate change.
- Splitting the batch-wide gate so that fresh closes archive while backlog rows wait.
- No Mac or vault recovery is needed. The stuck pause cleared at 12:05 on 2026-11-24,
  and the Gmail step has passed every run since.
