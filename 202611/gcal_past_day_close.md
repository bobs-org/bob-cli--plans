---
tier: tale
size: medium
title: "bob gcal sync: a past-day sync closes Bob-filed meetings"
goal:
  Syncing a previous day completes its ended, still-open, Bob-filed Event Tasks and
  closes their MEET Pomodoros with the buffered time ranges, idempotently, leaving each
  closed row in its born-done end state.
decisions:
  running_meets:
    ask: Should a past-day sync also close a stale Bob-shaped open timed MEET entry?
    choices:
      close:
        A Bob-shaped stale running MEET closes in place; an edited one is kept and
        reported once
      keep:
        Every open timed MEET is left alone and reported once; only placeholder filings
        close
    default: close
    why:
      A sync-created running MEET on a past note is stale; the shape gate protects
      edited sessions.
    answer: close
  unlinked_rows:
    ask:
      Should a past-day sync complete open journaled rows that have no MEET link at all?
    choices:
      keep:
        Unlinked open rows stay open, reported once, and keep surfacing in list's
        past-open footer
      complete:
        Unlinked open rows are completed on the note's day with no MEET and no recorded
        minutes
    default: keep
    why:
      A removed link is a propose-once edit; completing with no session records
      attendance, no time.
    answer: keep
  memory_supersession:
    ask:
      Write the new decision record and mark the retired meet-filing clause
      superseded-in-part?
    default: true
    memory:
      - decisions
    corrects:
      Clock-tag-only, title, root, or description changes, a kept time or buffer edit,
      and the clock passing without a Google change never trigger re-filing.
    answer: true
proposed_by: bbugyi200.apollo.dn.f0
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 12:05:07
status: wip
---

# `bob gcal sync`: a past-day sync closes Bob-filed meetings

## Problem

Bryan's request (2026-11-12):

> Can you now help me make it so syncing a previous day always closes out the
> appropriate tasks and pomodoros? For example, the `bob gcal sync -D yesterday` command
> should close the open event task it contains and close the MEET pomodoro with the
> appropriate time range.

Today, an Event Task synced _before_ its meeting ended stays open forever once its day
passes, because every existing closing mechanism has a different trigger:

- **Born done** (`decisions:ended-synced-meetings-are-born-done`) applies only when sync
  _creates_ a row whose buffered span already ended. A row created while the event was
  Current or Future is `[*]` under an open MEET placeholder (or a running MEET) and is
  never revisited.
- **Refile** (`decisions:meet-filing-follows-google-until-edited`,
  `src/native/gcal/refile.rs`) completes a row and files a struck link under a closed
  MEET only when _Google changes_ the event's meeting time or parsed buffer. The record
  and `docs/gcal.md` § Sync state explicitly that "the clock passing without a Google
  change never trigger[s] re-filing".
- **Settle** (`decisions:ended-off-clock-events-settle`,
  `src/native/task_status_hooks/settle.rs`) runs from `bob task reconcile`, but
  `plan_settle` returns early when `anchor != now.date()` — it only ever touches
  _today's_ ledger — and it only settles _off-clock_ events.

So after an evening of meetings nobody started, `bob gcal sync -D yesterday` reports
"Events unchanged", and yesterday's note keeps `[*]` rows under an open
`- [ ] () — MEET` placeholder until Bryan closes them by hand (they linger in
`bob gcal list`'s `past_open` footer).

## Goal

On a sync of a **past note day** (note day strictly before `now.date()` in vault-local
time — the same `input.today` / `input.now` already carried by `SyncPlanInput`), every
included, journaled Event Task row that is still open and whose MEET filing is still
Bob-owned is converged to its **born-done end state**: row `[x]` with
`[completion:: <note day>]`, a struck `🍅 ~~[[#^id]]~~` link under closed
`- [x] (**SSSS-EEEE** [t:: Nm]) — MEET` / `(**SSSS-EEEE** [off:: Nm])` clusters built
from buffered spans and split by clock verdict, and the emptied untimed placeholder
removed. A second run is a byte-identical no-op. Today's and future notes are untouched
by this pass: today keeps its live-session semantics (timed close classification,
reconcile's off-clock settle, running MEETs).

Invariant to state in docs and assert in tests: after a past-day sync, an eligible row
looks exactly as if it had first been synced by this run (the born-done shape), so the
existing Bob-shaped-closed-MEET recognition in `refile.rs::bob_closed_members` keeps
working on later Google moves.

## Current implementation map (read these before coding)

- `src/native/gcal/plan.rs` — `build_plan` orchestration: `apply_lifecycle` (cancels for
  absent events) → `apply_effective_clocks` → `refile::plan_refile` (returns
  `Vec<WantedLink>`) → `plan_meet_pomodoros` (merges birth links from
  `collect_wanted_links` with refile wants; Past links go through `cluster_past` +
  `format_meet_span` into closed MEETs with struck links via
  `append_links(..., true, ...)`) → `link_order::sort_event_task_links` → `plan_stamp`.
  The no-op guard (`records_empty`) and the `changed` clearing block list every plan
  vector.
- `src/native/gcal/refile.rs` — `classify_filing` buckets a row's ledger mentions into
  `Planned` (one exact plain, childless `- [[#^id]]` bullet under the single open
  untimed `() — MEET` placeholder, row untouched), `BobClosed`, `Running` (any mention
  under an open timed entry), `Unlinked`, `Yours`. `Filing::Planned` + `LinkPhase::Past`
  already does exactly what this feature needs per row: `lifecycle::remove_meet_link`,
  `complete_row` (sets `[x]` + completion field), then a
  `WantedLink { phase: Past, refiled: true }` for the placement pass; afterwards
  `lifecycle::remove_childless_untimed_meet` cleans the emptied placeholder.
  `report_kept`/`plan::push_kept_once` give journal-flag-backed once-only notices.
- `src/native/gcal/sync.rs` — `link_phase_for` computes the phase from real local
  datetimes (`buffered_end <= now` ⇒ Past), so midnight-crossing events whose buffered
  span has not ended yet are _not_ Past even on a past note. Report structs/JSON
  (`links_refiled` at ~110/465/683/1849/1967, summary strings at ~2100–2190), host
  status record, and the multi-day wrapper live here.
- `src/native/gcal/lifecycle.rs` — `row_untouched` (status `[ ]`/`[*]`, no 🍅 in the
  row's subtree, not linked under any closed Pomodoro), `remove_meet_link`,
  `remove_childless_untimed_meet`.
- Tests: `tests/gcal_sync.rs` (fake gog via `tests/gcal_support`), see
  `moved_past_event_refiles_its_bob_written_closed_meet` for the refile test pattern and
  JSON assertions.

## Design

### Trigger and eligibility

Add a **clock-passed close pass** that runs inside plan building, after `plan_refile`
and before `plan_meet_pomodoros`, only when `note_day(input) < input.now.date()`. It
examines every included occurrence (`occurrence.task == true`) with
`link_phase == Some(LinkPhase::Past)` whose journaled/adopted row is present in the
note, skipping:

- block IDs already produced by this run (birth links, refile wants, `plan.moved_rows`)
  — a Google-moved row keeps going through the existing refile path untouched by this
  pass;
- rows that fail `lifecycle::row_untouched` (done, cancelled, `[/]`, 🍅 history, or
  linked under a closed Pomodoro) — these are Bryan's record;
- rows without a parseable `**HHMM-HHMM**` span (all-day rows);
- rows that `unique_event_by_id` cannot resolve to exactly one Events row.

The phase check uses the occurrence's `link_phase`, so an early-morning `-D yesterday`
run never closes a midnight-crossing event whose buffered end is still in the future;
the next run closes it.

### Per-filing behavior

Classify each candidate's filing with the same machinery as `classify_filing` (refactor
it so it works from a `block_id` + current line instead of requiring a `MovedRow`; the
refile call site passes the moved row's old line as before):

| Filing                                                                                                                                                      | Action                                                                                                                                                                                                                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Planned` (untouched link under the open untimed placeholder)                                                                                               | Close: `remove_meet_link`, `complete_row` with the note's day, emit a Past `WantedLink` flagged `closed: true`. The placement pass clusters these by buffered span (`cluster_past`, split by clock verdict) into closed MEETs with struck links — identical to born-done output. `remove_childless_untimed_meet` then drops the emptied placeholder. |
| `Running` (mention under an open **timed** entry)                                                                                                           | Per `running_meets` decision, below.                                                                                                                                                                                                                                                                                                                 |
| `Unlinked` (no ledger mention)                                                                                                                              | Per `unlinked_rows` decision, below.                                                                                                                                                                                                                                                                                                                 |
| `Yours` (hand filing: shared/non-exact bullet, link with child notes, several entries, or under Bryan's closed session while the row is somehow still open) | Keep, and report once (see Notices).                                                                                                                                                                                                                                                                                                                 |
| `BobClosed`                                                                                                                                                 | Cannot occur for an open row (`bob_closed_members` requires born-done members); treat as `Yours` defensively.                                                                                                                                                                                                                                        |

> [!decision] running_meets = close A **Bob-shaped stale running MEET** is an open entry
> whose header is exactly `- [ ] (**SSSS-EEEE** [t:: Nm]) — MEET` (or the `[off::]` form
> — the shape `format_meet_span` writes when sync creates a running MEET, which a
> capture-started session never matches because capture leaves the end open), and whose
> every physical child line is a dedicated, childless, exact plain `- [[#^id]]` link
> resolving to a same-note Event Task, every member Past phase and `row_untouched`.
> Close it **in place**: status to `[x]`, keep its start, extend its end to
> `max(current end, members' max buffered end)`, recompute the `[t::]`/`[off::]`
> duration from the final span (keep its existing clock key), strike each member link to
> `🍅 ~~[[#^id]]~~`, and complete each member row with the note's day. Do not remove and
> rebuild the entry, and do not split it by clock. An open timed MEET that fails the
> shape gate — or any non-MEET open timed entry holding a candidate's link — is kept and
> reported once.

> [!decision] running_meets = keep Every open timed entry is left exactly as it is;
> candidates filed under one are kept and reported once. Only placeholder-filed rows
> close.

> [!decision] unlinked_rows = keep An open, untouched, journaled row with no ledger
> mention stays open (a removed link is never re-added, per the existing propose-once
> rule) and is reported once. It keeps surfacing in `bob gcal list`'s `past_open`
> footer.

> [!decision] unlinked_rows = complete Such a row is completed `[x]` with
> `[completion:: <note day>]` and reported in `rows` as `closed`; no MEET entry or link
> is written and no minutes are recorded anywhere.

### What this pass never does

- Never touches today's or a future day's note (strictly `note day < today`).
- Never trims spans against logged sessions: clusters are the born-done shape
  (`decisions:synced-meetings-link-by-phase` already accepts that a closed MEET built
  from calendar spans may overlap a logged session; reconcile's trimmed settle shape is
  deliberately not reproduced, so Bob-shaped recognition keeps working).
- Never cancels anything (absent events keep going through the existing lifecycle pass,
  which runs earlier).
- Never appends a new `gcal.link` journal record for a moved link (same rule as refile)
  and never journals the checkbox change; idempotency comes from the row no longer being
  open on the next run.
- Never changes `bob task reconcile` (settle stays today-only and off-clock).

### Notices

Each kept candidate (filing `Yours`, `Running` under the keep branch or a failed shape
gate, `Unlinked` under the keep branch) is reported once per occurrence via
`push_kept_once` with a new flag `kept_open` and a detail that names the cause and the
hand fix, following the existing `kept_meet` voice, e.g.
`left ^id open: its MEET link is yours; close or cancel it in the note`. `-F` clears the
day's notice flags, so a forced sync may re-report — existing behavior for every
`kept_*` flag.

### Report, JSON, and status surface

Mirror the `links_refiled` plumbing end to end, additively (JSON `schema_version` stays
1; this is not a contract break, so no `!` commit marker):

- `SyncPlan`: new `rows_closed: Vec<ClosedRow>` (`time_label`, `title`, `completion`
  day; modeled on `CancelledRow`) and `links_closed: Vec<PlannedLink>`. `WantedLink`
  gains `closed: bool`; the placement pass routes `closed` links into
  `plan.links_closed` (exactly as `refiled` routes into `links_refiled` at
  plan.rs:3044). Add both to the `records_empty` guard and the `changed` clearing block.
  The emptied-placeholder count rides the existing `entries_removed`.
- JSON report: each closed row appears in `rows` with `action: "closed"`; `pomodoro`
  gains `links_closed` (count). The multi-day wrapper needs no change (per-day reports
  carry the new fields). Closed rows no longer count as `in_sync`.
- Human report: one row line per closed row and a summary segment, following the
  existing report style (e.g. `✔ closed  19:00–20:00  Dinner w/ Alice` rows; summary
  `✓ Events updated · Pomodoros +1 entry, ✔2 closed`; a close-only run reads
  `✓ Events updated · Pomodoros ✔2 closed`). Reuse the dim/color conventions of
  neighboring rows; `-q` still silences success output; `-d` shows everything only as
  the unified diff plus the same report.
- Host status (`status.json`) and its consumers: add `links_closed` beside
  `links_refiled` (sync.rs ~1849).

### Interactions to preserve

- `-F` on a past day: reset rebuilds the day (restored past rows are born done), then
  this pass closes any kept open Bob-filed rows in the same plan.
- Multi-day runs (`-D yesterday -D today`): the pass applies per past day only; day
  ordering and the wrapper document are unchanged.
- Waiting/held days, the account guard, and the Events stamp rules are untouched.
- `link_order::sort_event_task_links` runs after placement, as today.

## Steps

1. **Refactor** `refile.rs::classify_filing` (and `ledger_mentions` callers as needed)
   to classify by `block_id` + an optional old line, so both the Google-move path and
   the new pass share it. No behavior change; the refile tests stay green.
2. **Implement the close pass** (new `plan_clock_passed_close` in `refile.rs` or a
   sibling `close.rs` module) per the Design section, returning extra `WantedLink`s;
   call it from `plan.rs::build_plan` right after `plan_refile`, merging its wants into
   the same `plan_meet_pomodoros` call with block-ID dedup against birth and refile
   wants. Implement the `running_meets` and `unlinked_rows` branches selected above.
3. **Plumb reporting**: `ClosedRow`, `links_closed`, `kept_open` notices, JSON fields,
   human rows and summary segments, status record, no-op guards.
4. **CLI help**: extend `sync_command`'s `long_about` with one clause (a past day also
   closes Bob-filed open meetings) and add `bob gcal sync -D yesterday` to
   `sync_after_help` examples; keep cli_rules conventions (no new options). Regenerate
   the CLI reference with `just docs-gen`.
5. **Docs** (`docs/gcal.md`): § Sync — rewrite the "A past day (`-D yesterday`) gains
   closed MEET entries…" paragraph to cover closing previously synced rows, scope the
   "clock passing … never trigger[s] re-filing" sentence to same-day syncs, update the
   running-entry and unlinked-row sentences per the selected branches, and document the
   new JSON fields and summary forms. § Reconciling Google-side changes — adjust "A
   running entry is left alone" / "a passing clock does not trigger this re-file" to
   name the past-day exception, and extend the "Missed meetings are never
   auto-cancelled" bullet with the new closing path. § Off-clock events — note that a
   past-day sync also closes an ended `#off` event nobody started.
   `docs/task-status-hooks.md` § Settled off-clock events — one cross-reference sentence
   (settle is today-only; a past-day `bob gcal sync` closes older ones).
6. **Tests** (`tests/gcal_sync.rs` unless noted; follow the `moved_past_event_refiles…`
   pattern):
   - Placeholder-filed `[*]` row on yesterday closes: `[x]` +
     `[completion:: <note day>]`, struck link under
     `- [x] (**SSSS-EEEE** [t:: Nm]) — MEET` with the buffered cluster span, placeholder
     removed; JSON asserts `action: "closed"`, `pomodoro.links_closed`, and the summary
     string.
   - Back-to-back events form one cluster; a mixed on/off-clock pair splits into
     overlapping `[t::]` and `[off::]` MEETs; an `#off` event closes into `[off:: Nm]`.
   - Rerun is byte-identical and reports zero `links_closed` (idempotency); the
     resulting rows satisfy the born-done shape (a subsequent Google move refiles them
     via `BobClosed`).
   - Touched rows (`[x]`, `[/]`, 🍅 history) and `Yours` filing are kept with one
     `kept_open` notice that does not repeat on the next run.
   - `running_meets` branch: Bob-shaped stale running MEET closes in place (span end
     extended, duration recomputed, links struck, rows completed); a running MEET with
     an added child note is kept + noticed. (Keep branch: both are kept + noticed.)
   - `unlinked_rows` branch per selection.
   - Today's note: an ended event under the placeholder stays open (scope guard); a
     future day stays untouched.
   - Early-morning `-D yesterday` with a midnight-crossing event whose buffered end is
     later than now: row stays open this run.
   - Multi-day `-D yesterday -D today`: wrapper carries yesterday's closes, today
     unchanged; `-F -D yesterday` closes kept rows after the reset; `-d` previews the
     close without writing.
   - `bob gcal list` `past_open` footer drops the closed meetings (`tests/gcal_list.rs`)
     if that assertion is cheap to add.
7. **Memory**:

   > [!decision] memory_supersession

   Add the strand `sase/memory/decisions/past-day-sync-closes-bob-filed-meetings.md` —
   claim per the Design section (including the selected branches), why (Bryan's
   2026-11-12 request; a past-day sync runs after every event ended by construction,
   voiding the earlier "sync usually does not run after an event ends" rejection;
   reconcile never touches past ledgers), rejected alternatives (extend reconcile to
   past days; trim like settle; close today's ended events too), cost (attendance is
   unknowable: a skipped meeting is closed and counted; uncheck or cancel it by hand),
   reopens-when (sync can learn attendance), evidence (this plan, `docs/gcal.md` § Sync,
   `tests/gcal_sync.rs`), with `[[...]]` links to
   `decisions:ended-synced-meetings-are-born-done`,
   `decisions:meet-filing-follows-google-until-edited`,
   `decisions:ended-off-clock-events-settle`, and `glossary:event-task`. Mark
   `sase/memory/decisions/meet-filing-follows-google-until-edited.md` with
   `metadata.status: superseded-in-part` plus `superseded_by` and a body back-link
   stating that the "clock passing … never trigger[s] re-filing" clause is retired for
   past note days. Run `sase memory init`. Keep both bodies otherwise untouched.

## Validation

- `cargo check`, then `cargo test --test gcal_sync` (and `--test gcal_list` if touched)
  plus any refile/settle-adjacent unit tests in the touched modules.
- `just fmt lint`.
- `just docs-gen` after the help change; `just docs-check` — if it fails only with the
  pre-existing `docs/reference/plugins/index.md` mount collision (tracked on the
  `bob-cli-it` bead), note that and proceed.
- `git diff --check`.
