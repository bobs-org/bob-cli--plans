---
tier: epic
title: Datetime stamps on every task log entry
goal: 'Every new Schedule, Rotten, Cancel, and Reopen Log entry, from every bob-cli
  and plugin writer, carries one `YYYY-MM-DD HH:MM` stamp of the minute it was written,
  exactly as Work Log entries already do. Older date-only entries stay valid and untouched,
  and every reader, recall band, and display handles both forms.

  '
decisions:
  schedule_layout:
    ask: Where should a Schedule Log entry's write stamp go?
    choices:
      inside: Stamp opens the italic lead before the change; the reason body and old
        readers stay intact
      lead: Stamp is the whole italic lead; the change moves into the body and readers
        must split it off
    default: inside
    why: The reason body stays byte-identical, and old readers keep working during
      rollout
    answer: inside
  reopen_stamp:
    ask: Which minute should a Reopen Log entry's stamp record?
    choices:
      write: The minute bob gmail sync writes the entry; the card alias still shows
        the message's minute
      message: The message's arrival minute, repeating the card alias's time
    default: write
    why: Stamp = write instant for every log; the alias already shows the arrival
      minute
    answer: write
  memory_decision_record:
    ask: Add a decision record for stamped task logs and mark the two Rotten Log records
      it supersedes in part?
    memory:
    - decisions:task-log-entries-are-stamped-at-write
    - decisions:rotten-keeps-are-logged
    - decisions:next-keeps-ask-an-optional-rotten-reason
    corrects: The entry reads `*YYYY-MM-DD* — <reason> · kept N×` with the local review
      date
    default: true
    answer: true
  memory_glossary_rotten_log:
    ask: Correct the Rotten Log glossary term's entry grammar to the stamped form?
    memory:
    - glossary:rotten-log
    corrects: stored as a `🥀 **ROTTEN LOG**` direct child with newest-first entries
      `*YYYY-MM-DD* — <body>`
    default: true
    answer: true
phases:
- id: cli-readers
  title: bob-cli readers, recall, and snapshot accept stamps
  depends_on: []
  size: medium
  description: 'cli-readers: add the shared Rust stamp formatter and parser. Teach
    `bob logs` (parse, score, precision, cache) and the highlights note snapshot to
    read stamped entries in every log. Publish the canonical reader grammar, and mark
    the JSON change with `!`.'
- id: plugin-readers
  title: Plugin Rotten and Schedule readers and recall bands accept stamps
  depends_on: []
  size: medium
  description: 'plugin-readers: make the nav and ledger-tools Rotten Log parsers,
    the nav Schedule Log parser, and both plugins'' recall-age precision accept stamped
    and legacy entries. Displays show the time. Deploy both plugins before any plugin
    writer stamps.'
- id: cli-cancel
  title: bob-cli Cancel Log writers stamp entries
  depends_on:
  - cli-readers
  size: medium
  description: 'cli-cancel: split the cancel engine''s log stamp from `[cancelled::]`,
    and stamp every Rust Cancel writer: relation cancel, merge, offer close, gcal,
    and requirement drop. Regenerate the canonical shared fixtures with a `stamp`
    input.'
- id: cli-schedule-reopen
  title: bob-cli Schedule and Reopen Log writers stamp entries
  depends_on:
  - cli-readers
  - plugin-readers
  size: medium
  description: 'cli-schedule-reopen: stamp the capture `$<N>`, `bob task reroll`,
    and pull-forward Schedule entries and the gmail Reopen entry from one `now` per
    run, in the chosen layout. Mark the capture JSON change with `!`.'
- id: nav-rotten-schedule
  title: Navigation hotkeys stamp Rotten and Schedule entries
  depends_on:
  - cli-readers
  - plugin-readers
  size: medium
  description: 'nav-rotten-schedule: add the generic plugin stamp formatter. Stamp
    every bob-navigation-hotkeys Rotten and Schedule writer and preview from one commit-time
    clock read per gesture, shared with that gesture''s Work Log entries.'
- id: nav-cancel
  title: Navigation hotkeys stamp Cancel entries
  depends_on:
  - nav-rotten-schedule
  - cli-cancel
  size: medium
  description: 'nav-cancel: split nav''s cancel planners'' stamp from `[cancelled::]`.
    Add an optional `stamp` to the `cancel.planCancel` API and stamp every nav Cancel
    writer. Copy the regenerated task_cancel and task_split fixtures.'
- id: cycler-bip
  title: Cycler and Block ID Prompt stamp their entries; requirement twins follow
    bob-cli
  depends_on:
  - nav-cancel
  - cli-cancel
  - cli-schedule-reopen
  size: medium
  description: 'cycler-bip: stamp task-status-cycler''s unblock Schedule entry, its
    planCancel callers, and the cycler-owned requirement-writer twin, copied byte
    for byte to nav. Stamp Block ID Prompt''s pull-forward entry. Copy the regenerated
    task_reqs, req_close, and offer_close fixtures.'
- id: mac
  title: Bob Mac Capture re-syncs fixtures and pins minute recall ages
  depends_on:
  - cli-readers
  - cli-cancel
  size: small
  description: 'mac: re-copy the regenerated req_close vectors, add tests that a minute
    Cancel `last_used` renders a minute age, and refresh the stale day-only comments.'
- id: memory
  title: Decision record and Rotten Log glossary follow the stamped grammar
  depends_on:
  - cycler-bip
  size: small
  description: 'memory: apply the accepted memory decisions (the new decision record
    with its supersession marks, and the Rotten Log glossary grammar), then regenerate
    AGENTS.md and the generated docs.'
proposed_by: bbugyi200.athena.1ho
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-28 08:20:35
status: done
bead_id: bob-cli-rv
---

- **PROMPT:** [prompts/202611/task_log_datetime_stamps.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_log_datetime_stamps.md)
- **BEAD:** [bob-cli-rv](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-rv/README.md)

# Plan: Datetime stamps on every task log entry

## Why and what exists today

Bryan asked that every task log's sub-bullets use a datetime, as the Work Log already
does. Epic `bob-cli-fd` (plan `202611/work_log_timestamps.md`) stamped Work Log entries
`*YYYY-MM-DD HH:MM* — …`. That plan explicitly left the other logs day-only. In the
vault today:

```markdown
- 🛠️ **WORK LOG**
  - _2026-11-15 13:06_ — athena agent
- 🥀 **ROTTEN LOG**
  - _2026-11-26_ — 🤷 no reason given · kept 2× in Next
- ❌ **CANCEL LOG**
  - _2026-11-09_ — We already do this for Leaf projects now.
- 🗓️ **SCHEDULE LOG**
  - _2026-10-30 → 2026-11-18_ — 🎲 P1 → P2 decay · in **19** (8–30) days
- 📬 **REOPEN LOG**
  - _2026-11-27_ — ✉️ reply from Ada · [[261030_job#^m-1a21|2026-11-27 06:40]]
```

The Schedule Log has no write date at all. Its italic lead holds the old and new
`scheduled` dates, so `bob logs` can only guess when a reason was typed (`day-approx`).
There are no other managed log kinds: the Rust registry and both plugin sweeps found
exactly these five markers.

Governing decisions: `decisions:bob-cli-releases-ride-a-daily-train` (bob-cli commits
mark CLI/JSON contract breaks with `!` and never touch the `Cargo.toml` version or
`CHANGELOG.md`), `decisions:rotten-keeps-are-logged`,
`decisions:next-keeps-ask-an-optional-rotten-reason`, and
`decisions:task-decay-is-one-ladder` (they fix the Rotten Log body grammar; this epic
changes only its stamp), and `decisions:mac-capture-is-a-thin-client` (the Mac app
renders bob's strings and parses no new grammar). Bryan's hosts track master, so a phase
"lands" when it reaches master and its plugins are deployed.

### Writers (all must stamp)

| Log      | Repo    | Writer                                                                                                                                                                                                                                                                                                                                                                                                                 | Date it uses today                                                            |
| -------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Schedule | bob-cli | `capture_schedule_log::plan` / `entry_text` (capture `$<N>`, via `capture/plan.rs` and `capture/project_note.rs`)                                                                                                                                                                                                                                                                                                      | rolled `scheduled` date                                                       |
| Schedule | bob-cli | `randomize_plan.rs` (`bob task reroll`) via `plan_entry_insertion`                                                                                                                                                                                                                                                                                                                                                     | from/to `scheduled` dates; `PlanContext` has only `today`/`until`             |
| Schedule | bob-cli | `capture_task_toggle/task_update.rs` `pull_forward_entry_text`, `_FROM → TODAY_` with `_` emphasis                                                                                                                                                                                                                                                                                                                     | today                                                                         |
| Cancel   | bob-cli | `task_cancel/line.rs` `edit_source` / `edit_source_create_log`. **One `date` writes both `[cancelled::]` and the entry**                                                                                                                                                                                                                                                                                               | caller's `today`                                                              |
| Cancel   | bob-cli | callers: `capture/relations_cancel.rs` (`DependencyContext`, no `now`), `task_merge/retirement.rs` cancel and `📥 merged into` provenance (`MergeContext.today`), `task_complete/offer_close.rs` (`OfferCloseInput.today`), `gcal/lifecycle.rs` `cancel_task_row` (`SyncPlanInput.now` exists), `task_block_layout/writers.rs` `drop_req` (via `req_writes.rs` ← `req_close.rs` ← `req_choice.rs` ← `capture/plan.rs`) | today                                                                         |
| Reopen   | bob-cli | `gmail/sync/reopen_plan.rs` `reopen_log_entry` (`ReopenPlanContext` has only `today`)                                                                                                                                                                                                                                                                                                                                  | message arrival date                                                          |
| Rotten   | nav     | `195-rotten-keep.js` `formatRottenLogEntryText` (rejects anything but `YYYY-MM-DD`), `appendFreshnessDecayRottenLogEntry`, `planFreshStampBatchWithLogs`; callers 520, 532, 480 (API wrapper), 540 (decay Keep/Stay/Drop), 548 (Step)                                                                                                                                                                                  | `dateText` / `cardCtx.dateText`; 540:190 and 548:206 read a live `new Date()` |
| Schedule | nav     | `030` `formatScheduleLogEntryText` / `planScheduleLogEntry` (no clock); callers 219, 260 (`planCountedBulletPropertyBatch`), 550:719 `setCountedBulletPropertyValue` (passes no `now`), 390 priority rolls, 570 project-note scheduled, 530/540 decay Less often/Reword, 548 P-level Step, 070                                                                                                                         | from/to values                                                                |
| Cancel   | nav     | `030` `formatCancelLogEntryText` / `planCancelLogEntry`, `240` `planTaskCancelBatch` (one `date` for field and entry), `713` API `cancel.planCancel(content, targets, {date, reason})` v1; callers 540:730 picker Cancel / Task Card Drop, 540:177 decay Drop, 070:94 roll past the last level, 594/596 relation cancel                                                                                                | `valueBaseDate` / `today`                                                     |
| Schedule | tsc     | `020` `formatScheduleLogEntry(removed, today)` → `*REMOVED → TODAY* — 🔓 unblocked by hand`; callers 150, 160, 180                                                                                                                                                                                                                                                                                                     | `getScheduleLogDateString()` (190:695)                                        |
| Cancel   | tsc     | planCancel callers 089g:324, 089i:172; requirement-writer twin `taskReqDrop` (cycler owns it, nav carries a byte copy; throws unless `date` is `YYYY-MM-DD`)                                                                                                                                                                                                                                                           | `context.today`                                                               |
| Schedule | bip     | `040` `formatScheduleLogEntryLine`, `_OLD → TODAY_ — 🍅 …` (`_` emphasis, mirrors bob-cli `task_update.rs`), from `070` `planTargetTaskUpdate`                                                                                                                                                                                                                                                                         | `localTodayParts(options.now)`                                                |

### Readers (must accept both forms before any writer stamps)

| Repo              | Reader                                                                                                                                                                                                                                    | Today, with a stamped entry                                       |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| bob-cli           | `log_history/parse.rs` `parse_stamp`: Rotten/Cancel strict `%Y-%m-%d`; Schedule `FROM → TO` or `TO`, every part a date                                                                                                                    | **Entry silently `excluded.unparsed`** and gone from recall       |
| bob-cli           | `log_history/scan.rs` `score_use` (non-Work kinds parse `last_used` as a date, `unwrap_or(0.0)`), `CACHE_VERSION` (`catalog-v1`), `cli.rs` `format_age` (hours only for Work), `mod.rs` `recency_precision` (`minute`/`day-approx`/`day`) | Wrong scores and ages, stale cache                                |
| bob-cli           | `highlights_ref/note_snapshot/visibility.rs` `format_schedule_inner` / `format_maybe_date` (`parse_strict_calendar_date`, exactly 10 bytes)                                                                                               | Raw stamp text instead of a display date (Work already shows raw) |
| nav               | `010` `ROTTEN_LOG_ENTRY_RE` → `195` `parseRottenLogEntryBullet` → `030` `readRottenLog` (decay header tallies, `EARLIER KEEPS`, refresh summary 192, prompt histories 520/532)                                                            | **Silently dropped**: header reads "none logged yet"              |
| ledger-tools      | `135` `FRESHNESS_ROTTEN_ENTRY_RE` (byte mirror), `freshnessMarkRottenLogShortDate` (244), and Reading-view `…SummaryFromListItem` (346–375, which rebuilds the line and needs an `<em>` that holds the date)                              | **Silently dropped**: tooltip line hidden                         |
| nav               | `010` `SCHEDULE_LOG_ENTRY_RE` → `030` `parseScheduleLogEntryBullet` → `getPriorityRollStreak` (030:880, which reads only `reason`)                                                                                                        | Depends on layout (see Design)                                    |
| nav, ledger-tools | Recall-age precision hard-coded as `kind === "work" ? "minute" : "day"` (nav 183:31–37, 184:222; lt 279:57–63)                                                                                                                            | Day ages for stamped Cancel/Rotten/Schedule rows                  |
| Mac               | `LogRecallBandPresentation.age(lastUsed:)` treats any `:` as minute precision                                                                                                                                                             | Already correct. Only its comments are stale                      |

Already fine: `periods/brief.rs` (first 10 chars), merge provenance check, gmail reopen
and gcal dedup (they match card ids, row status, or reason text, never the stamp),
`vault_merge_union.rs`, the agenda JSON (it passes text through verbatim), and every
Cancel Log consumer in the plugins (marker-only).

## Design

### The stamp

The stamp is exactly the Work Log stamp, `YYYY-MM-DD HH:MM`: zero-padded, 24-hour, with
seconds truncated (never rounded). It has no weekday, seconds, `T`, or offset. Every log
writes its stamp as the italic lead of the entry and keeps its existing body grammar:

```markdown
- [-] #task Port the importer [cancelled:: 2026-11-28] ^port
  - ❌ **CANCEL LOG**
    - _2026-11-28 14:32_ — Already supported.
    - _2026-10-25_ — Entries written before this change keep their date
  - 🥀 **ROTTEN LOG**
    - _2026-11-28 14:32_ — 🍂 Next → Ready · after 2 keeps
    - _2026-11-27 08:05_ — Still blocked on Ada · kept 2× in Next
  - 🗓️ **SCHEDULE LOG**
    - _2026-11-28 14:32 · 2026-11-18 → 2026-12-02_ — 🎲 P2 randomize · in **14** (8–30)
      days
    - _2026-11-10 09:05 · 2026-11-18_ — 🎲 P0 → P1 · in **8** (2–7) days
    - _2026-10-30 → 2026-11-18_ — legacy entry, unchanged
```

### Schedule Log layout

With `inside` (the default), the italic lead is `STAMP · TRANSITION`. The separator is
`·` (a space, U+00B7, a space). TRANSITION is the legacy text: `FROM → TO`, or `TO` when
there was no previous value. When there is no schedule value at all, the lead is just
`STAMP`. This also fixes the existing decay-card Reword / Less often bug, which writes
`** — reason` on an unscheduled task. The body is exactly the reason, so `bob logs`
recall, the Ctrl+Enter roll streak, and machine-text classification read the same bytes
as today. Old nav builds keep the roll streak: their regex lets `from` absorb
`STAMP · FROM` (or `to` absorb `STAMP · TO`) and still captures the reason. Pull-forward
entries keep their `_` emphasis (`_STAMP · FROM → TODAY_ — 🍅 …`), so recall keeps
skipping them.

> [!decision] schedule_layout = lead The italic lead is just `*STAMP*`, as in every
> other log. The transition becomes the start of the body, followed by `·` and then the
> reason:
> `*2026-11-28 14:32* — 2026-11-18 → 2026-12-02 · 🎲 P2 randomize · in **14** (8–30) days`,
> `*2026-11-10 09:05* — 2026-11-18 · 🎲 P0 → P1 · …`, or `*2026-11-28 14:32* — reason`
> when there is no value. Pull-forward is `_STAMP_ — FROM → TODAY · 🍅 …`. Readers split
> a leading `DATE → DATE · ` or `DATE · ` off the body only when it is a strict-date
> transition. Recall, roll-streak classification, and machine-text checks run on the
> reason after that split. `log_history`'s Schedule body is the reason, never the
> transition. The highlights snapshot renders the stamp as the lead and the transition
> as body text. Every vector below uses the **lead** column. `cli-schedule-reopen`
> already waits for `plugin-readers`, because under this layout old nav builds lose the
> roll streak on stamped entries.

### Reopen Log stamp

With `write` (the default), the stamp is the minute `bob gmail sync` writes the entry
(its run's `now`). The card alias `[[stem#^m-id|YYYY-MM-DD HH:MM]]` keeps showing when
the message arrived. The duplicate check stays keyed on the card id.

> [!decision] reopen_stamp = message The stamp is the message's arrival minute in
> `ctx.timezone`, the same value the card alias prints. `ReopenPlanContext` needs no new
> clock. Document that the Reopen Log is the one log whose stamp records the triggering
> event, not the write.

### Semantics

Every phase implements these rules exactly:

1. **Write instant.** The stamp is the minute the entry is written into the task note:
   the gesture's commit (Enter in a prompt or card, the `bob` run, the checkbox). It is
   never the time a prompt or card opened, a session start, or a daily-note filename
   date. (Under `reopen_stamp = message`, Reopen is the exception.)
2. **One instant per gesture, across all logs.** One clock read is formatted once. Every
   entry that one gesture writes carries a byte-identical stamp, across every target and
   every log kind. For example:
   - a decay Step writes Rotten + Schedule;
   - a decay Drop writes Rotten + Cancel;
   - Ctrl+Shift+P writes Schedule + Work;
   - a multi-item `bob capture` run uses the `now` it already resolves once at
     `capture/plan.rs:420`;
   - one `bob task reroll` run;
   - one gmail or gcal sync run.
3. **Clocks stay where they are.** bob-cli uses `env::current_datetime()` (`BOB_NOW`,
   then `DATE`, then the configured `timezone`, then the host zone). Plugins use host
   local time through an injectable `now` seam, as the Work Log writers already do.
4. **Field dates stay dates and keep their sources.** `[cancelled::]`, `[scheduled::]`,
   `[fresh::]`, the Schedule transition dates, and every keep or decay count keep their
   current values and sources. Only the log stamp is new. Where one value fed both a
   field and the entry (the cancel engines, `taskReqDrop`), split it into `date` (field)
   and `stamp` (entry).
5. **Previews are live.** Prompt and card previews format the clock at render time with
   the same formatter the writer uses, and they render the exact bytes it writes. That
   includes fixing the decay Stay preview's double spaces (nav 195:453). The written
   stamp is read again at commit.
6. **History is immutable.** Legacy date-only entries are never backfilled, rewritten,
   or flagged. Moves, splits, merges, promotions, and reversals still carry log subtrees
   byte for byte.
7. **Strict writers, tolerant readers.** Writers emit only the forms in this plan, and
   they reject a date-only stamp argument so an unconverted caller fails loudly. Readers
   accept exactly the stamped and legacy forms. They reject one-digit hours, seconds,
   `T`, and offsets.
8. **Displays show the time when there is one.** A surface that reformats a stamp
   renders its date as it does today and appends ` HH:MM` when the stamp has a time,
   matching the lane-walk Work Log chip (`🛠️ Nov 1 14:32 — …`). Examples: the decay card
   header `last Nov 28 14:32 — …`, the freshness-mark tooltip, and the highlights
   snapshot (now including Work entries).
9. **DST.** Stamps are wall-clock time, so they can repeat during the fall-back hour.
   Entry order (newest first, by insertion) stays the authority.

### Conformance vectors

These tables are the parity contract between the Rust, JS, and Swift copies. Each phase
copies the rows for its implementation into its tests verbatim. The **formatter**
vectors are the Work Log ones (`2026-11-01 14:32:59` → `2026-11-01 14:32`;
`2027-01-01 00:00:00` → `2027-01-01 00:00`).

**Writers** (stamp `2026-11-28 14:32`):

| Writer           | Inputs                                  | Entry (`inside`)                                                        | Entry (`lead`)                                        |
| ---------------- | --------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------- |
| Rotten keep      | `Still blocked on Ada`, kept 2× in Next | `*2026-11-28 14:32* — Still blocked on Ada · kept 2× in Next`           | same                                                  |
| Rotten step      | Next → Ready after 2                    | `*2026-11-28 14:32* — 🍂 Next → Ready · after 2 keeps`                  | same                                                  |
| Cancel           | `Already supported.`, date `2026-11-28` | `*2026-11-28 14:32* — Already supported.` + `[cancelled:: 2026-11-28]`  | same                                                  |
| Schedule         | from `2026-11-18`, to `2026-12-02`      | `*2026-11-28 14:32 · 2026-11-18 → 2026-12-02* — R`                      | `*2026-11-28 14:32* — 2026-11-18 → 2026-12-02 · R`    |
| Schedule         | to `2026-12-01` only                    | `*2026-11-28 14:32 · 2026-12-01* — R`                                   | `*2026-11-28 14:32* — 2026-12-01 · R`                 |
| Schedule         | no value (empty `to`)                   | `*2026-11-28 14:32* — R`                                                | same                                                  |
| Pull-forward     | from `2026-12-05`, today `2026-11-28`   | `_2026-11-28 14:32 · 2026-12-05 → 2026-11-28_ — 🍅 …`                   | `_2026-11-28 14:32_ — 2026-12-05 → 2026-11-28 · 🍅 …` |
| Reopen (`write`) | message `2026-11-28 10:00`              | `*2026-11-28 14:32* — ✉️ reply from Ada · [[c#^m-1\|2026-11-28 10:00]]` | same                                                  |

In the table, R stands for the unchanged reason text. The pull-forward `🍅 …` reason
text is unchanged.

**Readers** (any list marker, as today):

| Line                                                                             | Log              | Parsed                                                                   |
| -------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------ |
| `- *2026-11-28 14:32* — Already supported.`                                      | any non-Schedule | date `2026-11-28`, time `14:32`, body `Already supported.`               |
| `- *2026-10-25* — Already supported.`                                            | any non-Schedule | date `2026-10-25`, time none                                             |
| `- *2026-11-28 14:32 · 2026-11-18 → 2026-12-02* — R`                             | Schedule         | stamp `2026-11-28 14:32`, from `2026-11-18`, to `2026-12-02`, reason `R` |
| `- *2026-11-28 14:32 · 2026-12-01* — R`                                          | Schedule         | stamp, to `2026-12-01`, reason `R`                                       |
| `- *2026-11-28 14:32* — R`                                                       | Schedule         | stamp, no transition, reason `R`                                         |
| `- *2026-10-30 → 2026-11-18* — R`                                                | Schedule         | no stamp, from, to, reason `R` (legacy)                                  |
| `- *2026-11-06* — R`                                                             | Schedule         | no stamp, to `2026-11-06`, reason `R` (legacy)                           |
| `- *2026-11-28 9:05* — x`, `*2026-11-28 14:32:10* — x`, `*2026-11-28T14:32* — x` | any              | rejected                                                                 |

Under `lead`, rows 3 and 4 use their `lead` spellings and parse to the same fields.

**Recall** (`bob logs list`, today `2026-11-28`):

| Kind     | Entry                                                           | `text`               | `last_used`                                           |
| -------- | --------------------------------------------------------------- | -------------------- | ----------------------------------------------------- |
| cancel   | `*2026-11-28 14:32* — Already supported.`                       | `Already supported.` | `2026-11-28 14:32`                                    |
| cancel   | `*2026-10-25* — Already supported.`                             | `Already supported.` | `2026-10-25`                                          |
| rotten   | `*2026-11-28 14:32* — Still blocked · kept 2× in Next`          | `Still blocked`      | `2026-11-28 14:32`                                    |
| schedule | `*2026-11-28 14:32 · 2026-11-18 → 2026-12-02* — Waiting on Ada` | `Waiting on Ada`     | `2026-11-28 14:32`                                    |
| schedule | `*2026-12-05 → 2026-12-09* — Waiting on Ada`                    | `Waiting on Ada`     | `2026-11-28` (legacy: earliest date, capped at today) |

**Displays:** the decay header or tooltip `last` fragment for
`*2026-11-28 14:32* — Still blocked · kept 2× in Next` gains the time after the date,
for example `Nov 28 14:32`. A legacy `*2026-11-25* — …` keeps today's rendering.

### Phase graph and rollout

Readers go first. `cli-readers` and `plugin-readers` run in parallel. No writer phase
starts before the readers for its log have landed and deployed:

- `cli-cancel` waits for `cli-readers`. No plugin or Mac code parses Cancel stamps.
- `cli-schedule-reopen` and `nav-rotten-schedule` wait for both reader phases.
- `nav-cancel` follows `nav-rotten-schedule`, which owns the shared stamp plumbing that
  the decay Drop needs, and `cli-cancel` (fixtures).
- `cycler-bip` follows `nav-cancel` (the `planCancel` stamp option and nav's twin copy),
  `cli-cancel` (fixtures and the Rust `drop_req` it mirrors), and `cli-schedule-reopen`
  (the Rust pull-forward bytes it mirrors).
- `mac` follows the fixture and precision changes.
- `memory` runs last.

Plugin phases deploy only their own plugins, once per plugin id:
`bob plugins sync --repo <checkout> -p <id>`.

**Expected transient:** `cli-cancel` regenerates canonical fixtures that bob-plugins
copies byte for byte. Until `nav-cancel` and `cycler-bip` copy them, the plugin
canonical-comparison tests run beside a bob-cli checkout
(`test-task-reqs-twin-copies.cjs` and friends) report drift. That is expected. Do not
file it as a flake or CI bead.

## Phase cli-readers: bob-cli readers, recall, and snapshot accept stamps

Work in bob-cli.

- Add one shared module, for example `src/native/task_log_stamp.rs`:
  - `task_log_stamp(now: NaiveDateTime) -> String` (`%Y-%m-%d %H:%M`).
  - A strict parser
    `parse_task_log_stamp(&str) -> Option<(NaiveDate, Option<NaiveTime>)>` that accepts
    exactly 10 or 16 bytes and checks digit positions.
  - A Schedule lead splitter that returns `{stamp, from, to}` for both the chosen layout
    and the legacy forms.

  `capture_work_log::work_log_stamp` delegates to the formatter. Add the formatter and
  reader vectors as unit tests.

- `log_history`:
  - `parse_stamp` accepts both forms for Work, Rotten, and Cancel. A stamped entry's
    `last_used` and `sort_key` are the minute. A legacy entry's are as Work's legacy
    handling is today.
  - Schedule uses the splitter. A stamped entry's `last_used` is the stamp minute;
    legacy keeps the earliest-date-capped-at-today rule.
  - `score_use` uses one minute-age path for every kind.
  - `recency_precision` returns `minute` for all four kinds.
  - `cli.rs` `format_age` shows hours for any stamp with a time.
  - Bump `CACHE_VERSION`.
  - Add the recall vectors to `log_history/tests.rs`. Keep the classify registry test
    passing.
- Highlights snapshot (`note_snapshot/visibility.rs`): `format_maybe_date` renders a
  stamp as `format_display_date(date) + " HH:MM"`. `format_schedule_inner` splits a
  leading stamp and renders `<stamp> · <from> → <to>`. Add snapshot tests for every log,
  including the Work Log, which shows raw text today.
- Docs:
  - `docs/reference/vault-format.md`: one canonical "Task log entries" grammar section
    beside the managed-children registry. It covers the stamp, the per-log reader
    grammar (both forms), and semantics 1–9. Writer phases add their "writers emit"
    sentences. Fix the stale `docs/projects.md` ~1180 claim that the Work summary is
    `*YYYY-MM-DD*`.
  - `docs/log-recall.md`: the precision table, the stamp sentences, and frecency ages.
  - `docs/reference/json-outputs.md`: `recency_precision` and `last_used`.
  - Run `just docs-check`.
- Commit `feat(logs)!: …`, because the `recency_precision` values and the `last_used`
  shape change for three kinds. Keep `schema_version: 1`, since clients already handle
  minute `last_used`. Verify with `just check`.

## Phase plugin-readers: Plugin Rotten and Schedule readers and recall bands accept stamps

Work in bob-plugins (open it with `/sase_repo` and read its `AGENTS.md`). Edit `src/`
fragments only, then run `npm run build`. Keep every fragment at or below 1000 lines.
nav 010, 030, 470, and 480 are close to the cap, so split a fragment and register it in
`fragments.json` rather than exceed it.

- nav `010` `ROTTEN_LOG_ENTRY_RE`, and its byte mirror in ledger-tools `135`
  `FRESHNESS_ROTTEN_ENTRY_RE`: accept an optional ` HH:MM` (strict two digits) after the
  date inside the italics. `parseRottenLogEntryBullet` (195) and `readRottenLog` (030)
  return `{date, time, …}`, with `time: null` for legacy entries.
- ledger-tools: `freshnessMarkRottenLogShortDate` and the Reading-view
  `…SummaryFromListItem` accept a stamped `<em>`. Render displays per semantic 8 through
  one helper per plugin, so the text and DOM paths cannot drift. Do the same for the nav
  decay header and `EARLIER KEEPS` rows.
- nav `SCHEDULE_LOG_ENTRY_RE` / `parseScheduleLogEntryBullet`: parse the chosen layout
  and both legacy forms into `{stamp, from, to, reason}`. `getPriorityRollStreak` keeps
  reading `reason` only. Keep accepting `**…**` emphasis as today.
- Recall bands (nav 183, 184; ledger-tools 279): drop the `kind === "work"` precision
  hard-code. Treat a `last_used` with ` HH:MM` as minute precision for every kind, as
  the Mac band does.
- Fixtures and tests:
  - Add the stamped reader vectors to `scripts/fixtures/rotten-log-grammar.json`
    (`parse` and `tallies`; `format` stays for the writer phase).
  - Add the Schedule reader vectors to `test-navigation-hotkeys-schedule-log.cjs`.
  - Add a roll-streak test where stamped and legacy entries mix.
  - Add display vectors to `test-navigation-rotten-log.cjs` and
    `test-ledger-tools-freshness-mark*.cjs`.
  - Add minute-age rows to `test-navigation-log-recall.cjs` and
    `test-ledger-tools-log-history.cjs`.
- Docs: `docs/bob-navigation-hotkeys.md` (~450, header `last <date>`) and
  `docs/bob-ledger-tools.md` (~296, tooltip) show the time. Run `npm run docs:check`.
- Verify:
  - Run `npm run build`, `npm run build:check`, and `npm test`. Compare failures with
    the known set (bob-cli-f1, bob-cli-et, bob-cli-eh, bob-cli-f8) before claiming new
    ones.
  - Deploy `-p bob-navigation-hotkeys` and `-p bob-ledger-tools`.

## Phase cli-cancel: bob-cli Cancel Log writers stamp entries

Work in bob-cli.

- `task_cancel`: `TaskCancelInput` gains `stamp: String`. `edit_source` and
  `edit_source_create_log` write `*{stamp}* — …` and keep `date` for `[cancelled::]`
  only.
- Thread one stamp from each caller's single `now`:
  - `DependencyContext` gains it, built where `capture/plan.rs` builds the context from
    the run's `now`, for `relations_cancel.rs`.
  - `MergeContext` gains it, for the cancel and `📥 merged into` provenance in
    `task_merge/retirement.rs`. Fill it at `task_merge/command.rs`,
    `capture/relations_merge.rs`, and `capture/transfer.rs`.
  - `OfferCloseInput` gains it (`capture/offer_verdicts.rs`).
  - gcal `cancel_task_row` uses `SyncPlanInput.now`.
  - `drop_req` in `task_block_layout/writers.rs` gains `stamp: &str`, threaded through
    `req_writes.rs`, `req_close.rs`, and `req_choice.rs` from `capture/plan.rs`.
- Shared fixtures (bob-cli is the source of truth):
  - Add a `stamp` input beside the existing `date`/`today` inputs.
  - Regenerate `tests/fixtures/task_cancel/vectors.json`, `offer_close/vectors.json`,
    `req_close/vectors.json` (and its `README.md`),
    `task_reqs/{writes,layout,vectors}.json`, and `task_split/vectors.json` (only if a
    case writes a new entry). Existing entries carried as input stay date-only.
  - Update the fixture READMEs and the vector-index docs that describe the inputs.
  - Do not copy into bob-plugins or the Mac app; later phases do that.
- Tests:
  - Update unit tests and
    `tests/cli/capture/{relations_cancel,req_close,successor_verdicts,…}.rs` and
    `tests/gcal_lifecycle.rs`. Every CLI test that writes an entry pins `BOB_NOW`.
  - Add one multi-target gesture test with one shared stamp.
  - Add one test where `BOB_NOW=2026-11-29 00:10:00` and the field and stamp agree on
    the date.
- Docs: `docs/task-relations.md` (~1346–1353, ~1507–1528), `docs/task-merge.md` (~59,
  ~106), `docs/task-requirements.md` (~46, ~244–248), `docs/task-dependencies.md` (the
  SV and RV vector inputs), `docs/gcal.md` (~1103–1108), `docs/projects.md` Cancel
  examples, and the Cancel writer sentence in `vault-format.md`. Run `just docs-check`.
- Commit `feat(cancel)!: …`, because capture JSON strings for req close and section
  merge carry entry text. Verify with `just check`.

## Phase cli-schedule-reopen: bob-cli Schedule and Reopen Log writers stamp entries

Work in bob-cli.

- `capture_schedule_log`:
  - `entry_text(stamp, from, to, reason)` emits the chosen layout; an empty `to` emits
    `*STAMP* — reason`.
  - `plan(indent, scheduled, reason, stamp)` is used by `capture/plan.rs` and
    `capture/project_note.rs`, which pass `task_log_stamp(now)`.
  - Update the module's mirror comments (they cite a stale `main.js` path; point them at
    the nav `src/030-…` formatter).
- `bob task reroll`: sample `current_datetime()` once per run in `randomize.rs` and
  carry the stamp through `Report`/`PlanContext` to `randomize_plan.rs`. The roll base
  (`today` / `BOB_DAY_FILE`) and the `from <until>` suffix are unchanged.
- Pull-forward (`capture_task_toggle/task_update.rs`): `pull_forward_entry_text` takes
  the stamp from the capture `now`, keeping `_` emphasis. Snooze and other callers pass
  their own clocks' stamp.
- Reopen (`gmail/sync/reopen_plan.rs`): `reopen_log_entry` stamps per `reopen_stamp`.
  Under `write`, `ReopenPlanContext` gains a stamp from the sync run's `now`
  (`gmail/sync/file.rs`). The dedup check stays keyed on the card id. Add
  `📬 REOPEN LOG` to the vault-format registry list, which omits it today.
- Tests:
  - Writer vectors as unit tests.
  - Update `tests/cli/capture/{priority,task_toggle,ensure_next,…}.rs`,
    `tests/randomize.rs`, and `tests/gmail_sync.rs` with pinned `BOB_NOW`.
  - Add one `$<N>` test and one reroll batch test asserting a single shared stamp.
- Docs: `docs/capture.md` (~650–656 `$<N>` example, ~7891–7899 pull-forward, ~8307–8313
  `schedule_log` JSON and its "schema is unchanged" note), `docs/randomize.md`
  (~166–180), `docs/gmail.md` (~1108–1111), `docs/projects.md` Schedule examples, and
  the Schedule and Reopen writer sentences in `vault-format.md`. Run `just docs-check`.
- Commit `feat(capture)!: …`, because the strings in `schedule_log.lines` change. Verify
  with `just check`.

## Phase nav-rotten-schedule: Navigation hotkeys stamp Rotten and Schedule entries

Work in bob-plugins, plugin `bob-navigation-hotkeys`, under the same fragment rules (nav
480, 510, 540, 550, 570, 580, and 700 are at 987–999 lines).

- Add a generic `formatTaskLogStamp(date)` (the Work Log contract). `formatWorkLogStamp`
  becomes an alias of it. Use one injectable commit-time `now` seam per gesture, and
  share the same stamp with any Work Log entry the gesture writes.
- Rotten:
  - `formatRottenLogEntryText({stamp, …})` and
    `appendFreshnessDecayRottenLogEntry(…, stampText, body)` accept only
    `YYYY-MM-DD HH:MM`.
  - `planFreshStampBatchWithLogs` takes `opts.stampText`, while `dateText` keeps
    stamping `[fresh::]`.
  - Stamp the callers in 520, 532, 480 (additive optional `stampText` on the API
    wrapper, sampling the clock when it is absent), 540 Keep/Stay, and 548 Step. Replace
    the live `new Date()` reads at 540:190 and 548:206 with the gesture's single read
    (their `valueBaseDate` semantics stay).
- Schedule:
  - `formatScheduleLogEntryText({stamp, from, to, reason})` and
    `planScheduleLogEntry(content, line, {stamp, …})` emit the chosen layout, with an
    empty `to` producing `*STAMP* — reason`.
  - Stamp every caller: 219, 260 (reuse the `options.now` already sampled for the Work
    Log), 550:719 (sample at commit), 390 (picker `fixedWorkLogNow`), 570, 530/540 Less
    often/Reword, 548 P-level Step, and 070.
  - Leave the test-only precomputed decay plans (050) alone, except their fixtures.
- Previews: `370-picker-schedule-work-log.js`, `380-picker-cancel-and-lane.js`, 192, and
  195 render live stamps and the exact written bytes (semantic 5).
- Guard:
  `rg -n "formatRottenLogEntryText\(|appendFreshnessDecayRottenLogEntry\(|planScheduleLogEntry\(|formatScheduleLogEntryText\(" plugins/bob-navigation-hotkeys/src`
  shows every writer receiving a `formatTaskLogStamp` stamp.
- Tests:
  - Move `rotten-log-grammar.json` `format` to stamped vectors.
  - Update the pinned-byte suites (`schedule-log`, `rotten-log`, `rotten-keep*`,
    `decision-card-*`, `roll-decay`, `quick-roll*`, `priority-*`, `schedule-review`,
    `schedule-work-log`, `task-card-review`, `freshness`, `morning-review-settle`) with
    a fixed `now`.
  - Add three tests: a decay Step shares one stamp across Rotten and Schedule;
    Ctrl+Shift+P shares one stamp across Schedule and Work; a card opened at 23:59 and
    committed at 00:01 stamps `00:01` on the new date.
  - The `obsidian-decay-ladder-e2e` expectations follow.
- Docs: `docs/bob-navigation-hotkeys.md` (~313, ~426–453), `docs/task-card.md`, and
  `docs/decay-ladder.md` examples. Run `npm run docs:check`.
- Verify: `npm run build`, `npm run build:check`, and `npm test` (known failures as
  above), then deploy `-p bob-navigation-hotkeys`.

## Phase nav-cancel: Navigation hotkeys stamp Cancel entries

Work in bob-plugins, plugin `bob-navigation-hotkeys`.

- `formatCancelLogEntryText({stamp, reason})` and `planCancelLogEntry` require a stamp.
  `planTaskCancelBatch(content, session, {date, stamp, …})` keeps `date` for
  `[cancelled::]` only.
- `cancel.planCancel(content, targets, {date, reason, stamp})`: an additive optional
  `stamp`. When it is absent, sample the plugin clock once per call, so an older caller
  still writes a stamp. Follow `docs/develop/plugin-apis.md`'s versioning rule (additive
  options keep v1 unless that doc says otherwise) and document the field there.
- Stamp the callers: 540:730 (picker Cancel / Task Card Drop, through `…FromEditor` /
  `…FromLinkPicker`), 540:177 (decay Drop, sharing one stamp with its Rotten entry),
  070:94 (roll past the last level), and 594/596 relation cancel.
- Copy bob-cli's regenerated `task_cancel` and `task_split` fixtures byte for byte.
  Teach their harnesses the new `stamp` input.
- Tests: update `cancel-log`, `cancel-task`, `relation-cancel-writer`,
  `offer-close-apis`, `roll-decay` cancel, and `task-split-carry-api`. Add a decay-Drop
  test with one shared stamp across Cancel and Rotten. Add a `planCancel` test without
  `stamp` that still writes a stamp.
- Docs and verify as in nav-rotten-schedule, then deploy `-p bob-navigation-hotkeys`.

## Phase cycler-bip: Cycler and Block ID Prompt stamp their entries; requirement twins follow bob-cli

Work in bob-plugins, plugins `task-status-cycler` and `block-id-prompt`, plus the byte
copies they own in nav. tsc 089i is at 1000 lines and 190 at about 990, so split before
you add code.

- task-status-cycler:
  - `formatScheduleLogEntry(removed, today, stamp)` writes the chosen layout for
    `🔓 unblocked by hand`. Read the stamp once per gesture through a generic seam,
    generalizing `getPomodoroWorkLogStamp()` (for example `getTaskLogStamp()`), from the
    150, 160, and 180 callers.
  - The planCancel callers (089g, 089i) pass `stamp`.
- The requirement-writer twin `taskReqDrop` (`task-reqs-writers-twin.js`, cycler-owned)
  takes `stamp` beside `date`, mirroring bob-cli's new `drop_req` byte for byte. Copy it
  unchanged into nav. Update its callers (tsc 193, 198c, 198d, and the nav equivalents).
- block-id-prompt: `formatScheduleLogEntryLine` writes the pull-forward entry,
  byte-identical to bob-cli's `pull_forward_entry_text`, with the stamp from the
  existing `options.now`, read once in `planTargetTaskUpdate`.
- Copy bob-cli's regenerated `task_reqs/*` (every plugin copy), `req_close` (nav and
  cycler), and `offer_close` (cycler) fixtures byte for byte.
  `test-task-reqs-twin-copies.cjs` passes beside a bob-cli checkout.
- Tests: update `test-task-status-cycler-blocked-schedule.cjs`, the req-drop / req-close
  / successor / open-line / task-reqs-vectors suites,
  `test-block-id-prompt-target-update.cjs`, and the pomodoro-insertion/link suites. Add
  tests for one stamp shared across a multi-target close, and for the 23:59→00:01 commit
  case.
- Docs: `docs/block-id-prompt.md` (~75), `docs/task-status-cycler.md` (~151–152), and
  plugin-apis if touched. Run `npm run docs:check`.
- Verify:
  - Run `npm run build`, `npm run build:check`, and `npm test`.
  - Deploy `-p task-status-cycler`, `-p block-id-prompt`, and
    `-p bob-navigation-hotkeys` (its twin copy changed).
  - If the ledger-tools fixture copy is test-only, it needs no deploy.

## Phase mac: Bob Mac Capture re-syncs fixtures and pins minute recall ages

Work in bob-mac-capture (open it with `/sase_repo` and read its `AGENTS.md`), in
`Sources/CaptureCore` and `Tests` only.

- Re-copy `Tests/Fixtures/req_close/vectors.json` from bob-cli and confirm it with
  `cmp`. Re-copy `log-recall-vectors.json` only if bob-cli changed it.
- `LogRecallBandPresentation`:
  - Keep the `:` heuristic.
  - Update the comments at ~69–70, ~83, and ~105 to say that every kind may carry minute
    stamps.
  - Add `LogRecallStateTests` rows: a Cancel `last_used` of `2026-11-28 14:32` renders a
    minute or hour age, and `2026-10-25` keeps a day age. Add a direct `minuteAge` test.
- Refresh the stale `*YYYY-MM-DD* — ` comments in
  `CapturePomodoroClosePresentation.swift` (~98–110).
- Verify:
  - Run `just portable-check` (and `just docs-check` if docs change).
  - After pushing, watch the exact-SHA macOS GitHub Actions run to green with
    `/sase_monitor`.

## Phase memory: Decision record and Rotten Log glossary follow the stamped grammar

Work in bob-cli. Use `/sase_memory_write`, and apply only the memory decisions that were
answered yes.

> [!decision] memory_decision_record Add the
> `decisions:task-log-entries-are-stamped-at-write` strand (status accepted, decided on
> the epic's approval date).
>
> - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, and the vault.
> - **Claim:** every Work, Schedule, Rotten, Cancel, and Reopen Log entry leads with a
>   `YYYY-MM-DD HH:MM` stamp of its write instant (with the chosen Schedule layout and
>   Reopen rule). One gesture uses one stamp across all of its logs, field dates stay
>   dates, writers are strict, readers are tolerant, and legacy entries are never
>   backfilled.
> - **Why:** Bryan's request in this epic's prompt, and the Work Log precedent
>   (`plan:202611/work_log_timestamps.md`). Cite this plan and the phase commits.
> - **Rejected alternatives:** keeping the other logs day-only (that plan's semantic 8),
>   seconds, offsets, backfilling, and the unchosen Schedule layout.
> - **Cost:** wider rows, and a mixed history of stamped and date-only entries.
> - **Reopens when:** minute precision proves noise in recall or reviews.
>
> Then add `decisions/task-log-entries-are-stamped-at-write` to the `superseded_by` list
> of both `decisions/rotten-keeps-are-logged.md` and
> `decisions/next-keeps-ask-an-optional-rotten-reason.md` (both are already
> `superseded-in-part`). In each body, add one back-link sentence saying that its
> `*YYYY-MM-DD*` entry stamp was retired, and leave the rest untouched.

> [!decision] memory_glossary_rotten_log In `glossary/rotten-log.md`, replace
> "newest-first entries `*YYYY-MM-DD* — <body>`" with "newest-first entries
> `*YYYY-MM-DD HH:MM* — <body>` (entries from before stamps keep `*YYYY-MM-DD*`)".

Then run `sase memory init`, `just docs-gen` (which regenerates
`docs/reference/glossary.md` and `docs/develop/decisions.md`), and `just docs-check`.

## Non-goals

- Backfilling, reformatting, or relative-time rendering of existing vault entries.
- New log kinds, new CLI flags, or a `bob logs` schema bump.
- Plugin timezone awareness. `bob logs`' own clock (`Local::now()`, which ignores
  `BOB_NOW`) also stays as it is.
- Switching pull-forward entries from `_` to `*` emphasis.

## Known pre-existing issues (for the land phase to triage, not fix here)

- `periods/brief.rs`: `in_rotten` never resets when a later sibling log starts
  (~1897–1917, ~2041–2058). The "latest reason" is the oldest entry in a newest-first
  log (~1777, ~2052). The fixtures write entries oldest-first, which hides this.
- `📬 REOPEN LOG` is missing from the `task_children` registry and from `task_split`'s
  marker list, so tidy, NOTES scoops, or splits can treat it as an authored note.
- Pull-forward inserts under the first managed log of any kind
  (`first_direct_managed_log_start`, `task_update.rs` ~88 and ~190), while
  `docs/capture.md` ~7893 says Schedule Log only.
