---
tier: tale
title:
  bob gcal sync links every new Event Task under a past, running, or planned MEET
  Pomodoro
goal:
  Every Event Task that bob gcal sync creates starts [*] or [/], linked in the same
  write under a closed, running, or planned MEET Pomodoro chosen by the meeting's time.
size: medium
decisions:
  past_grouping:
    ask:
      Should sync merge back-to-back or overlapping past meetings into one closed MEET
      entry?
    choices:
      clusters:
        One closed entry per run of touching meetings; Stats splits a merged entry's
        minutes evenly
      per_event:
        One closed entry per meeting with its exact span; more rows, and double-booked
        time counts twice
    default: clusters
    why:
      Groups as far as one truthful span allows; one entry across a gap would log the
      gap as meetings
    answer: clusters
  open_meet_entries:
    ask:
      Should a running meeting and later meetings get separate open MEET entries even if
      two MEETs are open?
    choices:
      one_open:
        Never a second open MEET; later meetings ride the running MEET and carry to a
        placeholder on x
      separate:
        Running MEET plus MEET placeholder; duplicate_open_pomodoro_name lints while
        both are open
    default: one_open
    why:
      Keeps the plan lint quiet; Event Task clock close already splits a shared MEET
      entry on x
    answer: one_open
  memory_meet_record:
    ask:
      Should a new decision record supersede in part the one-appended-MEET claim for
      synced meetings?
    memory:
      - decisions:calendar-events-are-daily-note-event-tasks
      - decisions:synced-meetings-link-by-phase
    corrects:
      Synced calendar events live in their daily note's Events section with root
      attribution and one appended MEET.
    default: true
    answer: true
proposed_by: bbugyi200.apollo.bl
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 21:29:58
status: wip
---

# Plan: `bob gcal sync` files every new Event Task under a past, running, or planned MEET Pomodoro

## Goal

Bryan's request: "whenever the `bob gcal sync` command creates a new event task, a
corresponding past/current/future (depending on the time range of the event) pomodoro is
created with a task link corresponding to each event (with a preference for putting all
events in the same pomodoro when they are all the same type of pomodoro--for example, we
should group all future event tasks that this command creates in the same pomodoro).
This means that every event task that the `bob gcal sync` command creates should have
either a next (`[*]`) or closed (`[/]`) status."

After this change, every Event Task row that sync **creates** (a new row, or a Maybe
context row upgraded in place to a task) is linked in the same write:

| Phase at sync time                              | Pomodoro                                               | Link line     | Row checkbox |
| ----------------------------------------------- | ------------------------------------------------------ | ------------- | ------------ |
| Past: the event ended (`end ≤ now`)             | A closed `- [x] (**SSSS-EEEE** [t:: Nm]) — MEET` entry | `🍅 [[#^id]]` | `[/]`        |
| Current: `start ≤ now < end`, today's note only | The running MEET entry                                 | `[[#^id]]`    | `[*]`        |
| Future: the event has not started               | The open MEET placeholder `- [ ] () — MEET`            | `[[#^id]]`    | `[*]`        |

All past meetings that fit one truthful span share a closed entry (decision
`past_grouping`), all current meetings share one running entry, and all future meetings
share one placeholder.

## Current behavior (read these first)

- `src/native/gcal/plan.rs` — the pure planner. `plan_sync` inserts rows, then
  `plan_meet_links` (bottom of the file) appends a `[[#^id]]` for each `link_eligible`
  task row into `find_meet_entry` (running timed MEET first, else any open MEET),
  creating `- [ ] () — MEET` with `insert_named_placeholder_after_last_open` only in the
  write that adds its first link. `plan_stamp` runs last and shifts
  `pomodoro_entry_line` when it inserts lines. `upgrade_context_row` pushes
  `extra_link_wants` for an upgraded Maybe row.
- `src/native/gcal/sync.rs` — `build_occurrences` sets
  `link_eligible = task && is_today && !has_ended(..)`; `print_json` / `print_human` /
  `record_status` report one `pomodoro` entry; journal `gcal.link` marks are appended
  per `plan.links_added`.
- `src/native/gcal/render.rs` — `render_task_row` writes `[*]` when `linked`, else
  `[ ]`.
- So today: ended meetings get `[ ]` and no link, past days (`-D yesterday`) and future
  days get rows only, and only today's unended meetings get MEET links.

Constraints found while planning (do not regress them):

- `bob task reconcile` (task-status hooks) **exits 1 and writes nothing** when the
  current daily note has more than one non-empty open timed Pomodoro
  (`docs/task-status-hooks.md` § Guard Rails), and `bob capture =…` refuses to start a
  session while one is running. Sync must never create a running entry while any open
  timed entry exists.
- `duplicate_open_pomodoro_name` lints when one component (MEET) is on two open entries
  (`docs/plan.md` § Lints), and an `x` close always creates a same-named placeholder
  right after the closed entry when it carries links
  (`src/native/capture_pomodoro_close/ledger.rs`, `plan_ledger_close_with_parked`).
- Links under completed Pomodoros carry the machine-owned `🍅 ` marker; reconcile
  repairs a missing one. Write it so reconcile has nothing to repair.
- `row_untouched` (`src/native/gcal/lifecycle.rs`) treats a `[/]` row, or one linked
  under a closed Pomodoro, as worked: it is never cancelled, only reported once.
- R9 (`docs/task-status-hooks.md` status table): a `[*]` row in a canonical daily note
  that is unreachable from **today's** open-Pomodoro graph clears to `[ ]`. That only
  matters for `-D tomorrow` (see Days below).

## Target behavior

### Phases

Compute one phase per included **task** occurrence in `build_occurrences`, from the
event's local start/end (`window.local(..)`) and the run's single `now`:

- `Past` when `end ≤ now`.
- `Current` when `start ≤ now < end` **and** the note day is today.
- `Future` otherwise (including a still-running event on another day's note).

Replace `PlannedOccurrence::link_eligible: bool` with the phase (for example
`link_phase: Option<LinkPhase>`, `None` for context rows), and add the event's end in
minutes from the note day's midnight (it may exceed 1440 for a cross-midnight meeting)
so the planner can build spans without re-parsing `time_label`. Keep `start_min`.

### Which rows earn which links

1. **Rows this run creates** — `rows_added` task rows and `rows_upgraded` rows — on
   **any** day (today, `-D yesterday`, `-D tomorrow`): link by phase as in the Goal
   table. The row is rendered with the matching checkbox (`[/]` for Past, `[*]` for
   Current/Future): pass a status char (or the phase) into `render_task_row` instead of
   `linked: bool`. Every created Event Task therefore starts `[*]` or `[/]`.
2. **Rows already in the note without a journaled link or tombstone** (crash-recovery
   adoptions and legacy rows): keep today's rule — only on today's note, only when the
   event has not ended — but route them with the new Current/Future targets. Never
   create a closed entry for an existing row (a closed entry claims the meeting
   happened; only the run that writes the row makes that claim). Never rewrite an
   existing row's checkbox.
3. Journal and tombstones are unchanged: each link appends one `gcal.link` mark, a
   journaled link or tombstone is never re-added, and a removed closed entry's link is
   never re-created.

### Targets and grouping

Process phases in this order within `plan_meet_links` (rename it to something like
`plan_meet_pomodoros` if it reads better): Past, then Current, then Future.

**Past → closed MEET entries.** Sort this run's Past links by (start, end).

> [!decision] past_grouping = clusters Merge while the next meeting's start ≤ the
> cluster's end (touching or overlapping). One entry per cluster: span = cluster start
> to cluster end, `[t:: Nm]` = end − start minutes, children in meeting start order.

> [!decision] past_grouping = per_event One entry per Past link: span = that meeting's
> own span, `[t:: Nm]` = its minutes.

Entry line: `- [x] (**SSSS-EEEE** [t:: Nm]) — MEET` (end `% 1440` for a cross-midnight
meeting, same `format!` shape as `compute_pomodoro_start_range` /
`format_adjusted_range`). Each child is `<indent>- 🍅 [[#^id]]`, with the section's
existing child indentation (the rule `insert_link_into_entry` /
`pomodoro_child_indentation` uses; tab in the fixtures). Never reuse an existing closed
entry, even one named MEET with the same span.

Placement keeps history chronological: insert the new entry block after the child block
of the last completed entry whose start ≤ the new entry's start; otherwise before the
first completed entry; with no completed entries, before the first open entry; in an
empty section, at the section top (the slot capture uses for a first session). Insert
clusters in start order so each lands after the previous one.

**Current and Future → open MEET entries.** Let `R` be the single open timed entry (none
when there are zero or several), and `P` the first open untimed entry whose name is
exactly `MEET` (today's `find_meet_entry` match). A new running MEET is
`- [ ] (**SSSS-EEEE** [t:: Nm]) — MEET` spanning the earliest Current start to the
latest Current end of this run, inserted at capture's current-session slot (after the
last completed entry's child block — including any closed entry this run just added —
else before the first open entry, else the section top; mirror
`create_started_pomodoro_entry` / `new_pomodoro_insertion_index` in
`src/native/capture/`). A new placeholder uses the existing
`insert_named_placeholder_after_last_open(.., "MEET")`.

Shared rules for both choices:

- Never create a running entry while any open timed entry exists (one or several).
- Current links when `R` is MEET → `R`.
- Current links when `R` exists but is not MEET (or several timed entries are open) →
  handled as Future links, plus one warning per meeting, for example
  `Weekly sync is running now, but WORK is the running Pomodoro; linked it under the MEET placeholder`
  (`an unnamed Pomodoro` when the running entry has no name).
- Sync never edits an existing Pomodoro entry line: it never starts, closes, retimes,
  renames, or moves one. It only creates entries and appends links.

> [!decision] open_meet_entries = one_open The run never leaves a second open MEET.
>
> - Current links with no `R`: if `P` exists, link them under `P` (not started; Bryan
>   starts it with `=meet`); otherwise create a running MEET for them.
> - Future links: under the running MEET when `R` is MEET (existing, or created by this
>   run); else under `P`; else create `P`. On `x`, Event Task clock close defers and
>   carries them to a fresh MEET placeholder.

> [!decision] open_meet_entries = separate
>
> - Current links with no `R`: always create a running MEET, even when `P` exists.
> - Future links: always under `P` (create it when missing), even when a running MEET
>   exists. Document in `docs/gcal.md` that `duplicate_open_pomodoro_name` lints while a
>   running MEET and the MEET placeholder are both open.

### Days

- Today: all three phases.
- A past day (`-D yesterday`, or any day before today): every meeting is Past, so the
  note gains closed MEET entries and `[/]` rows.
- A future day (`-D tomorrow`): every meeting is Future, so the note gains a MEET
  placeholder and `[*]` rows. Document that R9 reconcile may clear those `[*]` rows to
  `[ ]` until that day becomes today, when the MEET link raises them again.

### Consequences to document, not fix

- A Past row is worked from birth (`[/]` and a closed-entry link), so a later Google
  decline, delete, or move leaves it alone and reports it once, like any worked task.
  Its Google-side span updates still apply while the spans match the journal.
- `[/]` rows count in the Pending lane and show in `list`'s `past_open` footer until
  Bryan completes or cancels them.
- A closed or running entry built from calendar spans may overlap a session Bryan
  already logged; sync does not clip it.

### Report, JSON, and status

- `SyncPlan`: replace `pomodoro_entry_line` / `pomodoro_created` with a list of touched
  entries
  `{state: closed|running|planned, line (1-based, in the final contents), time: Option<String> (HHMM-HHMM), created, links}`
  in ledger order. Every line must be correct after `plan_stamp` inserts its lines
  (generalize the `pomodoro_0` shifting there to the whole list; the existing
  `stamp_below_pomodoros_does_not_shift_meet_line` and
  `pomodoro_entry_line_shifts_when_stamp_is_inserted` tests must keep passing in their
  new form). Clear the list on a no-op plan as today.
- `sync -f json` (additive, `schema_version` stays 1, not a contract break):
  `pomodoro.entries` lists those entries; `pomodoro.entry_line` and `pomodoro.created`
  keep their keys and now describe the first **open** entry in `entries` (`null` /
  `false` when the run linked only under closed entries); `pomodoro.links_added` counts
  every link, closed entries included.
- Human output: one line per touched entry, in ledger order, replacing today's single
  MEET line — `  + MEET     closed 08:00–08:30 — MEET · linked 1`,
  `  + MEET     running 14:00–15:00 — MEET · linked 1`,
  `  + MEET     created () — MEET · linked 3`, and `=` instead of `+` (and no `created`)
  for an existing entry. Summary: `Pomodoros +N entries, +M links` (`entry` when N is
  1). Keep the `stamp_only` logic correct.
- `record_status` keeps `links_added` = all links.
- Leave `bob gcal list` alone; its `task · MEET` check already finds a `🍅 [[#^id]]`
  line. Verify that with a test rather than assuming it.

## Implementation steps

1. `sync.rs`: phase and end-minute computation in `build_occurrences`; pass them through
   `PlannedOccurrence`; warnings from the planner flow into `warnings`.
2. `render.rs`: `render_task_row` takes the status char; update its doc comment.
3. `plan.rs`: phase-aware link collection (created rows vs existing rows, as above),
   closed-entry clustering and placement, running-entry creation, placeholder reuse or
   creation per the `open_meet_entries` decision, the touched-entry list, and the line
   shifting in `plan_stamp`. Keep the module doc comment accurate. Prefer reusing
   capture helpers (`insert_link_into_entry`,
   `insert_named_placeholder_after_last_open`, `child_block_end_line`,
   `capture_pomodoros::scan`); widen a `pub(super)` capture helper to `pub(crate)` only
   when that is simpler than a small gcal-local helper.
4. `sync.rs`: `SyncReport`, `print_json`, `print_human`, and the dry-run, unchanged,
   retry, and waiting report constructors for the entry list.
5. Update the unit tests in `src/native/gcal/plan/tests.rs` (they set `link_eligible`)
   and add new ones (below).

## Tests

Unit (`src/native/gcal/plan/tests.rs`, pure planner, fixed `now`):

- Past + future rows in one run on an empty ledger: one closed MEET entry with `🍅`
  links before the placeholder; rows `[/]` and `[*]`; journal links for both.
- Past clustering per the decision: back-to-back meetings share one entry with the
  merged span and `[t::]`; a gap makes two entries; entries are in start order.
- Closed entry placement between Bryan's completed sessions by start time, and above an
  open placeholder.
- Current meeting with an empty ledger → running MEET with the meeting's span; with a
  running MEET → linked into it; with a running WORK → linked under the MEET placeholder
  plus the warning, and no second open timed entry.
- The `open_meet_entries` cases: current + future in one run, and a current meeting
  while a MEET placeholder already exists — assert the chosen choice's entries and that
  `one_open` never leaves two open MEETs.
- Cross-midnight meetings: a past one renders `**2330-0030**` with `[t:: 60m]`.
- Upgraded Maybe row that already ended → closed entry and `[/]`.
- Existing unlinked rows: a journaled ended row gets no closed entry; an adopted unended
  row still gets its open-MEET link.
- Idempotency: replanning the output with the resulting journal changes nothing.
- Entry lines in the report stay correct when the stamp inserts lines above and below
  the Pomodoros section.

Integration (`tests/gcal_sync.rs`, `tests/gcal_lifecycle.rs`; fixtures in
`tests/fixtures/gcal/`):

- Update `first_sync_creates_events_meet_and_links`: Product interview (0800-0830, ended
  at 12:00) is now `[/]` under `- [x] (**0800-0830** [t:: 30m]) — MEET` with
  `\t- 🍅 [[#^product-interview]]`; the four future meetings stay under one placeholder.
- Replace `past_day_gets_rows_only_and_no_meet` with a past-day test (closed entries,
  `[/]` rows, today's note untouched) and add a future-day test (placeholder, `[*]`).
- `reconcile_keeps_linked_rows`: after `bob task reconcile`, the past row is still `[/]`
  and its `🍅` link is byte-identical (reconcile repaired nothing).
- A current-meeting run (`BOB_NOW` inside a meeting) followed by `bob task reconcile`
  exits 0 (proves no double running entry), then `bob capture x` after the meeting ends
  completes the meeting per Event Task clock close.
- `closed_meet_session_attributes_body_in_stats` and the lifecycle tests: update the
  expectations the new closed entry changes, and add one asserting a Google delete of a
  Past row leaves it alone and reports it once.
- `-f json`: `pomodoro.entries`, `entry_line`, `created`, `links_added`; `-d` diff shows
  the new entries; `list` reports `task · MEET` for a Past row.

## Docs

Read the `docs.md` reference memory (`sase memory read docs.md`) first.

- `docs/gcal.md`: Sync (phase rules, the targets, "Past days get rows only; only today
  gains MEET links" replaced, the `sync -f json` field list), the ownership table (add a
  "MEET Pomodoro entries | Bob, once" row: created only in the write that adds their
  first link, never edited afterwards), the "Untouched" paragraph and "MEET placeholders
  are created only…" sentence per the chosen `open_meet_entries`, Reconciling
  Google-side changes (worked Past rows), Known limits (overlap with logged sessions; R9
  on future days), and JSON output.
- `docs/reference/vault-format.md` § Events section and Event Tasks: replace "Event
  Tasks linked under today's MEET show `[*]`; a row first synced after its event ended
  shows `[ ]`." and extend the example with a closed MEET entry.
- `docs/reference/json-outputs.md`: the `bob gcal sync` row ("the Pomodoro entry" → the
  MEET Pomodoro entries).
- `docs/vault-git-sync.md:472` if its wording about MEET links becomes wrong.
- Grep `docs/` for other statements this makes false (`MEET placeholder`,
  `first synced after`, `rows only`). Do not hand-edit generated pages; run
  `just docs-gen`.

## Memory

> [!decision] memory_meet_record Use `/sase_memory_write` first (this accepted decision
> authorizes both edits), then:
>
> - Add the `decisions` web strand `decisions:synced-meetings-link-by-phase`,
>   frontmatter shaped like the `decisions:unblocked-tasks-are-offered` strand
>   (`title`/`keyword` "Synced Meetings Are Linked By Phase At Birth", a one-line
>   `summary`, `aliases`, `metadata.status: accepted`). Body: **Applies to** (bob-cli
>   and the vault); **Claim** (the Goal table, grouping per the accepted `past_grouping`
>   and `open_meet_entries` answers, sync only creates MEET entries and appends links
>   and never edits an existing entry, never a second running entry); **Why** (quote
>   Bryan's request from the Goal; the hooks guard and lint constraints above); rejected
>   alternatives (the choices not taken, and `[x]` for ended meetings: sync cannot know
>   Bryan attended, and the request names `[/]`); **Cost** (Past rows are worked from
>   birth and never auto-cancelled, `[/]` rows count in Pending, spans may overlap
>   logged sessions, R9 clears future-day `[*]` until that day); **Reopens when**
>   (same-type Pomodoro duplicates land, or sync can learn attendance); **Evidence**
>   (this plan's archived `plan:` ref, `docs/gcal.md` § Sync, `tests/gcal_sync.rs`).
>   Link `[[decisions:calendar-events-are-daily-note-event-tasks]]`,
>   `[[decisions:event-task-links-close-by-the-clock]]`, and `[[glossary:event-task]]`.
> - Mark `sase/memory/decisions/calendar-events-are-daily-note-event-tasks.md` with
>   `metadata.status: superseded-in-part` and
>   `superseded_by: [decisions:synced-meetings-link-by-phase]`, and append one back-link
>   paragraph: "Superseded in part: Task Links no longer all go under one appended MEET
>   placeholder; an ended meeting lands under a closed MEET entry and a running one
>   under the running MEET. See [[decisions:synced-meetings-link-by-phase]]." Change
>   nothing else in the accepted body.
> - Run `sase memory init`, then `just docs-gen` (it regenerates
>   `docs/develop/decisions.md`).

> [!decision] memory_meet_record = no Leave `sase/memory/` untouched and follow
> `/sase_memory_write` § Declined Changes.

## Commit and release notes

Conventional commit `feat(gcal): …` (no `!`: the JSON change is additive and no CLI flag
changes). Do not edit `Cargo.toml`'s version or `CHANGELOG.md`.

## Validation

```bash
cargo test --test gcal_sync --test gcal_lifecycle --test gcal_list
cargo test gcal
just docs-gen
just docs-check
just all
```

## Out of scope

- Changing Event Task clock close, task-status hooks, the `duplicate_open_pomodoro_name`
  lint, or the bob-plugins Ledger Tools mirror.
- Clipping calendar spans around sessions Bryan already logged.
- Retroactively linking rows created before this change.
- Writing Work Log entries for Past meetings.
