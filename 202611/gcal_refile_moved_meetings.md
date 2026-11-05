---
tier: tale
title: Re-file MEET Pomodoros when Google moves an event
goal:
  A Google Calendar time or buffer change re-files the event's Bob-written MEET filing
  in the same sync, never touching sessions Bryan owns.
size: medium
decisions:
  closed_meets:
    ask:
      Also re-file untouched Bob-written closed MEETs (retime them, or reopen born-done
      rows moved later)?
    default: true
    why:
      The Swings move after a born-done sync would otherwise leave a stale closed MEET
    answer: true
  memory_decisions:
    ask:
      Add the meet-filing-follows-google-until-edited record and mark the three records
      it partly supersedes?
    memory:
      - decisions:meet-filing-follows-google-until-edited
      - decisions:synced-meetings-link-by-phase
      - decisions:bob-gcal-is-read-only-and-proposes-once
      - decisions:ended-synced-meetings-are-born-done
    default: true
    requested:
      MEET pomodoros should always be corrected in response to Google Calendar event
      updates when it is safe to do so (which should be basically always, right?)
    answer: true
proposed_by: bbugyi200.athena.18i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 08:34:03
status: wip
---

# Re-file MEET Pomodoros When Google Moves An Event

## Why this plan exists (the answer to "is there a good reason?")

No. The behavior that made Bryan fix the MEET by hand is a gap, not a safety rule.

What happened in `2026/20261105.md` (vault commits `4d37dfdb8`, `3e9fa6667`,
`8753da32a`):

1. 05:37: sync created `^swings` at `**0815-0830**` as `[*]` and linked `[[#^swings]]`
   under the future `- [ ] () — MEET` placeholder.
2. 08:17: Bryan moved the event to 08:00–08:15 in Google and sync ran.
   `apply_span_updates` (`src/native/gcal/plan.rs`) rewrote the bold span to
   `**0800-0815**` and re-slotted the row. The event was already over, but the link
   stayed under the future placeholder and the row stayed `[*]`.
3. 08:18: Bryan closed a MEET by hand (`0805-0820`), then edited it to `0800-0815`.

Root cause: Bob updates every other part of an Event Task row from Google until Bryan
edits it (time, title, badges, `#off`/`#on`, `[buffer::]`, description bullets,
`[root::]`). MEET filing is the exception: it is still written once.
`collect_wanted_links` only files rows created in this run, or open-MEET links for
unlinked rows that have not ended. A journaled `gcal.link` record blocks any later
filing, and nothing re-files a link when the time changes. The docs say so explicitly:
"Sync never starts, closes, retimes, renames, or moves an existing Pomodoro."

The decision records give a sound reason only for MEETs that are Bryan's record of what
actually happened. A session he started, closed, retimed, or annotated must not be
rewritten. That reason does not cover:

- a link sitting in an untimed `() — MEET` placeholder (a plan, not a record), or
- a closed MEET that still reads exactly as Bob wrote it.

Because `^swings` inherits off-clock (`abby` → `love`, `offclock: true`), the 15-minute
`bob task reconcile` settle would have filed `0800-0815 [off:: 15m]` by about 08:30. An
on-clock meeting moved the same way is never corrected.

"Basically always" safe is right, with three firm limits that this plan keeps:

- Sync never touches a running session.
- Sync never touches a MEET or link Bryan edited.
- Sync never acts unless Google changed the event. An unstarted meeting whose time
  simply passed keeps today's missed-meeting policy.

## Contract

**Trigger.** In one sync run, `apply_span_updates` applies a Google change to a
journaled Event Task row's _occupied span_: its meeting time (`time` span replaced) or
its `[buffer::]` (badge span replaced and the parsed buffer differs). These never
trigger a re-file:

- the clock passing, with no Google change;
- a kept edit (`kept_edit` on `time`/`badges`);
- title, root, description, or clock-tag-only updates;
- context rows.

This works on any note day. The phase rules are the birth rules from `link_phase_for`,
applied to the row's new buffered span.

**Filing classification** of the moved row's same-note ledger links (any decoration:
plain, 🍅, struck), measured on the note _before_ this run's ledger edits:

| Filing     | Definition                                                                                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Planned    | Every link is a dedicated child line (body is exactly `[[#^id]]`) with no children of its own, under an open untimed `() — MEET` entry; and `lifecycle::row_untouched` holds for the row |
| Bob closed | Exactly one link, a dedicated `🍅 ~~[[#^id]]~~` line under a **Bob-shaped closed MEET** (below), and the row is **born done** (below)                                                    |
| Running    | Any link sits under an open timed entry                                                                                                                                                  |
| Yours      | Anything else (a closed MEET that is not Bob-shaped, a link line with child notes, a shared line, a non-MEET entry, a touched row)                                                       |

A **Bob-shaped closed MEET** is a completed entry that meets all of these:

- Its header is byte-equal to `format!("- [x] {span} — MEET")` from
  `format_meet_span(start, end, off_clock)`. Here `start..end` is the cluster
  (`cluster_past` rules) of its members' buffered spans _as the rows read before this
  run_, and `off_clock` is the header's own clock key.
- Its children are exactly one dedicated `🍅 ~~[[#^id]]~~` line per member, with no
  grandchildren and nothing else.
- Every member is a same-note Event Task in the born-done state.

Settle-written MEETs that were not trimmed also match. Trimmed ones do not, so they are
treated as Yours.

A row is **born done** when all of these hold:

- it is `[x]` with `[completion:: <note day>]`;
- there is no 🍅 in its own child block;
- it has no other ledger link.

**Actions:**

| Filing     | New phase         | Action                                                                                                                                                                                                                                                                                                                                         |
| ---------- | ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Planned    | Future or Current | None (the placeholder is timeless, and birth also reuses it for Current)                                                                                                                                                                                                                                                                       |
| Planned    | Past              | Remove the placeholder link lines and file the row as a Past birth: it joins `cluster_past` and gets a struck 🍅 link under a new closed MEET in ledger time order. The row becomes `[x]` + `[completion:: <note day>]`. A childless untimed MEET placeholder is removed                                                                       |
| Bob closed | Past              | Remove the whole Bob-shaped MEET and re-file all of its members as Past wants (moved members at their new spans, the others at their current spans). Clusters re-form, which may retime, split, or merge them. Rows stay `[x]`                                                                                                                 |
| Bob closed | Future or Current | Remove the whole Bob-shaped MEET. Re-file the other members as Past wants. Reopen the moved row: `[x]` → `[*]`, `remove_completion_fields`. Route its link with the birth rules for its phase (running MEET, placeholder, or a created running MEET, never while another timed entry is open; otherwise placeholder plus the existing warning) |
| Running    | any               | None, silently (by-the-clock close classification already handles it)                                                                                                                                                                                                                                                                          |
| Yours      | any               | None. `push_kept_once(..., "kept_meet")` with detail `kept your MEET Pomodoro; Google moved it to HH:MM–HH:MM`                                                                                                                                                                                                                                 |

> [!decision] closed_meets = no The two "Bob closed" rows become Yours. They are
> reported with `kept_meet` and never edited, and step 2 skips the Bob-shape check and
> the reopen.

Invariants that must keep holding:

- The run never leaves a second open MEET.
- Sync never creates a running entry while any open timed Pomodoro exists.
- A re-file never adds a second link for the same block ID.
- Event Task link order is still sorted by `link_order` after the re-file.
- A rerun with no Google change is a no-op, apart from the normal per-minute stamp rule.
- `-d` shows the exact diff and writes nothing.
- An incomplete snapshot still writes nothing.

## Implementation steps

1. **Capture moves** (`src/native/gcal/plan.rs`). In `apply_span_updates`, when
   `apply_spans` replaces the time span or changes the parsed buffer, push a
   `MovedRow { calendar_id, event_id, block_id, old_line }` onto a new
   `SyncPlan.moved_rows`. `old_line` is the row text before this run's rewrite. Clear it
   in the no-op branch of `plan_with_clock` alongside the other vectors. Use the
   `event_task` span and buffer parsers. Do not re-parse by hand.

2. **New module `src/native/gcal/refile.rs`** (`plan.rs` is already about 2.7k lines).
   Add `plan_refile(input, working, plan, clock) -> Vec<WantedLink>` (make `WantedLink`,
   `cluster_past`, `format_meet_span`, and `resolved_off_clock` `pub(super)` as needed).
   It:
   - runs after `apply_lifecycle` and `apply_effective_clocks`, and before the ledger
     inserts in `plan_meet_pomodoros`;
   - classifies each `MovedRow` (above), using `capture_pomodoros::scan`,
     `child_block_end_line`, and `lifecycle::row_untouched`;
   - removes Bob filing:
     - dedicated placeholder link lines;
     - whole Bob-shaped closed MEET blocks;
     - then `lifecycle::remove_childless_untimed_meet`;
   - rewrites row status:
     - `capture_task_toggle::set_task_line_status` plus
       `task_complete::tree::add_or_replace_completion_field` for Past;
     - `set_task_line_status(.., '*')` plus `remove_completion_fields` for a reopen;
   - returns the re-file wants with their phases and effective clocks, which are
     resolved exactly like `collect_wanted_links` does;
   - pushes `kept_meet` notices for Yours.

3. **Route re-file wants through birth placement** (`plan_meet_pomodoros`). Merge the
   re-file wants into the `past` / `current` / `future` buckets before clustering and
   routing. They must bypass the `state.link` / created-row gates in
   `collect_wanted_links`. Dedupe by block ID against the birth wants. Count re-filed
   links in a new `plan.links_refiled` instead of `plan.links_added`, so
   `append_journal` (`src/native/gcal/sync.rs`) appends no new `gcal.link` records for
   them, because the occurrence already has one. Include re-file targets in
   `plan.pomodoro_entries`. Treat a non-zero `links_refiled` as a change in the
   `records_empty` / `changed` logic.

4. **Report** (`src/native/gcal/sync.rs`):
   - JSON `pomodoro.links_refiled` (additive; `schema_version` stays 1).
   - `SyncReport` / `SyncStatus` carry the count.
   - The human summary appends `, ↪N refiled`; a refile-only summary reads
     `✓ Events updated · Pomodoros ↪1 refiled`, following the existing sort-only
     pattern.
   - The moved row's `rows[].action` stays `updated`.
   - `kept_meet` notices flow through the existing kept/warnings path.

5. **Tests.** Add unit tests in `src/native/gcal/plan/tests.rs` and end-to-end tests in
   `tests/gcal_sync.rs` (and `tests/gcal_lifecycle.rs` where kept notices live):
   - **Swings regression.** `^swings` `[*]` `0815-0830` with a love-routed (inherited
     off-clock) root, under `() — MEET` beside `^unami-w-abby`. Google moves it to
     0800-0815 and sync runs at 08:17. Expect:
     - `- [x] (**0800-0815** [off:: 15m]) — MEET` with `🍅 ~~[[#^swings]]~~`, in ledger
       time order;
     - the row `[x]` with `[completion:: 2026-11-05]`;
     - the placeholder keeps `[[#^unami-w-abby]]`;
     - JSON `links_refiled: 1` and `links_added: 0`;
     - a rerun is a no-op.
   - The same move on-clock writes `[t:: 15m]`. A sole placeholder child removes the
     placeholder.
   - Planned → Future or Current: no ledger change, and the row time still updates.
   - Bob closed, past → past: born done at 08:31 under `0815-0830`, moved to 0800-0815,
     and the MEET becomes `0800-0815`. A two-member cluster that the move splits becomes
     two closed MEETs. A move that merges two Bob MEETs becomes one MEET.
   - Bob closed → Future: the row reopens `[*]` with no completion, the link goes under
     the placeholder (created when missing), and the closed MEET is gone. → Current
     while a FIX Pomodoro runs: placeholder plus the existing warning, and no running
     MEET is created.
   - Yours stays untouched, with `kept_meet` reported once across two changed runs, for
     each of:
     - an edited MEET span;
     - a note child under the MEET;
     - an extra non-Event link;
     - a row unchecked by hand;
     - a link line with a child note under the placeholder.
   - Running: a link under a running MEET is a silent no-op.
   - A kept time edit (Bryan edited the bold span) does not re-file.
   - A buffer-only Google change on a Bob closed MEET widens the MEET.
   - `-d` prints the diff and leaves the note and journal unchanged.
   - With `closed_meets = no`, the Bob-closed cases assert `kept_meet` and unchanged
     ledgers instead.

6. **Docs** (behavior change → contract/reference only).
   - `docs/gcal.md` § Sync: replace "Sync never starts, closes, retimes, renames, or
     moves an existing Pomodoro …" with the re-file contract. Sync still never starts or
     closes a session Bryan owns, and it re-files only Bob-written filing after a Google
     move.
   - § Who owns each part of a row: "MEET Task Link" and "MEET Pomodoro entries" become
     "Google until you edit it" with the re-file rule.
   - The "Untouched" paragraph: a move now re-files Bob filing, while a decline or
     delete still leaves a born-done row alone.
   - § Off-clock events: fix "sync never retimes existing MEET entries" so the tag-flip
     rule (`kept_off_clock`) stays but no longer claims MEETs are never retimed.
   - § Reconciling Google-side changes: add a bullet.
   - § JSON output and the `sync -f json` paragraph: `links_refiled`.
   - § Known limits: a Bryan-edited MEET is kept and reported once as `kept_meet`;
     trimmed settle MEETs count as yours.
   - Run `just docs-gen` (it regenerates `docs/develop/decisions.md` from the decisions
     web) and `just docs-check`. Do not edit `CHANGELOG.md` or the `Cargo.toml` version.

7. **Decision record** (use `/sase_memory_write`; each note is covered by its memory
   decision).

   > [!decision] memory_decisions Add the new record and mark the three older records
   > below.
   - Add the `decisions:meet-filing-follows-google-until-edited` strand. Copy the
     frontmatter shape of `gcal-root-follows-google-until-edited.md`:
     `origin: requested`, `scope: project`, `keyword`, aliases `gcal meet refile` and
     `moved meeting refile`, `metadata.status: accepted`, `decided: 2026-11-05`. Give it
     Applies to / Claim / Why / Rejected alternatives / Cost / Reopens when / Evidence.
     - **Claim:** the contract above, compressed.
     - **Why:** the Swings evidence above.
     - **Rejected:**
       - write-once filing (stale MEETs, hand fixes);
       - re-file on every run (it would auto-close on-clock meetings nobody attended);
       - re-file by journaled MEET records (it misses settle MEETs and adds a record
         kind);
       - moving links out of Bryan-edited sessions.
     - **Cost:** a born-done row moved later is reopened; a hand reorder or hand-written
       MEET that happens to be Bob-shaped is treated as Bob's.
     - **Reopens when:** attendance becomes knowable, or Bryan wants moved meetings to
       keep their original filing.
   - Mark these three records `superseded-in-part`, each with `superseded_by` gaining
     `decisions:meet-filing-follows-google-until-edited` and one back-link sentence
     naming the retired clause:
     - `synced-meetings-link-by-phase` (never retimes or moves);
     - `bob-gcal-is-read-only-and-proposes-once` (link written once);
     - `ended-synced-meetings-are-born-done` (never changes an existing row's checkbox).
   - Link `[[decisions:ended-off-clock-events-settle]]` and `[[glossary:event-task]]`.
   - Run `sase memory init`, then `just docs-gen`.

8. **Verify:** `just fmt`, `just lint`, `just test`, and `just docs-check` all pass.

## Out of scope

- Clock-tag flips on worked rows stay `kept_off_clock`.
- Cross-day moves keep the cancel path, which still leaves born-done rows alone.
- Reconcile's off-clock settle is unchanged.
- No changes to bob-plugins or Bob Mac Capture: the JSON change is additive, and Bob
  Ledger Tools only runs `sync -q -f json`.
