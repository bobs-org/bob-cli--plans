---
tier: epic
title: 'bob gcal: Google Calendar events as daily-note Event Tasks'
goal: 'Every day, the Google Calendar events Bryan said yes to appear as Event Tasks
  in a `## Events` section above `## Pomodoros`. Each one is rooted where its time
  belongs (`meet` by default, or a root the event names) and linked under one appended
  MEET Pomodoro. A plain close of a MEET session completes, carries, or defers each
  meeting by the clock. `bob gcal` (calendars, doctor, list, login, logout, sync)
  makes setup a single read-only login, makes every sync idempotent, and never overwrites
  Bryan''s edits. The Mac runs sync automatically and Obsidian shows the Events beautifully.

  '
decisions:
  events_paint:
    ask: Should Obsidian paint Events rows (time pills, now/past states, muted context
      rows)?
    default: true
    why: Events sit right above the ledger, so they should read as its sibling, not
      as raw Markdown
    answer: true
  launch_agent:
    ask: Install a Mac LaunchAgent that runs bob gcal sync every 5 minutes?
    default: true
    why: The daily-note trigger alone misses invitations you accept later in the day
    answer: true
  glossary:
    ask: Add an Event Task glossary strand and list Event Tasks as an Area Note residence
      exception?
    memory:
    - glossary
    default: true
    corrects: The only exceptions are each daily note's templated `^gtd` task and
      closed tasks
    answer: true
  decision_records:
    ask: Record the three gcal decisions (daily-note Event Tasks, clock close, propose-once)
      in decisions?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: meet-area
  title: Modernize meet.md and prepare the vault
  depends_on: []
  size: small
  description: 'meet-area: turn ~/bob/meet.md into a top-level violet area with task_type
    MEET and a ## Tasks section, add the .bob/gcal journal git rules, commit under
    the sync lock, and record meet''s hue in the homes docs.'
- id: event-close
  title: Event Task grammar and the clock-aware MEET close (Rust)
  depends_on: []
  size: medium
  description: 'event-close: add the shared Event Task parser and clock classifier,
    make bob''s x close resolve unlisted Event Task links by the clock, report it
    in JSON and human output, add shared vectors, document the Events grammar, and
    write the glossary and clock-close memory.'
- id: event-close-js
  title: Clock-aware Ctrl+Enter close in Obsidian
  depends_on:
  - event-close
  size: medium
  description: 'event-close-js: mirror the clock rule in task-status-cycler''s Pomodoro
    completion using a copy of the shared vectors, test it, document it, and deploy
    with bob plugins sync.'
- id: gcal-auth
  title: bob gcal skeleton, gog transport, login, logout, doctor, calendars
  depends_on: []
  size: medium
  description: 'gcal-auth: extract gkeep''s hardened subprocess runner, add the gcal
    module, config block, and gog transport with the BOB_GCAL_GOG test seam, ship
    login, logout, doctor, and calendars with beautiful output, register the command,
    and write the setup contract and guide.'
- id: gcal-list
  title: Event fetch, inclusion rule, root precedence, and bob gcal list
  depends_on:
  - gcal-auth
  size: medium
  description: 'gcal-list: fetch one DST-safe local day per calendar as a complete
    snapshot, normalize events, classify them as task, context, or omitted, choose
    and validate roots (route token, colour, calendar, default), and ship the read-only
    list as the bare default.'
- id: gcal-sync
  title: bob gcal sync writes Events rows and MEET links
  depends_on:
  - gcal-list
  - event-close
  - meet-area
  size: medium
  description: 'gcal-sync: render Event Task and context rows, keep the per-host journal,
    plan inserts, adoption, tombstones, and the appended MEET placeholder with links
    written once, write through a locked preimage-checked transaction with a no-op
    guarantee, and ship sync, list''s VAULT column, docs, and the daily-note decision
    record.'
- id: gcal-lifecycle
  title: Reconcile Google-side changes and learn series roots
  depends_on:
  - gcal-sync
  size: medium
  description: 'gcal-lifecycle: three-way update of untouched managed spans, cancel
    untouched tasks for declined, deleted, or moved events with a created CANCEL LOG,
    handle Maybe and Yes transitions, learn series roots from vault edits, list unresolved
    past meetings, and record the propose-once decision.'
- id: obsidian-events
  title: Obsidian trigger, ledger guard, and Events painting
  depends_on:
  - gcal-sync
  size: medium
  description: 'obsidian-events: run bob gcal sync when today''s note is created,
    add a gcal-sync toast adapter, stop the section-move gesture from dropping task
    rows into the ledger, paint Events rows when chosen, test, document, and deploy.'
- id: rollout
  title: Mac rollout, config, and handoff
  depends_on:
  - meet-area
  - event-close-js
  - gcal-lifecycle
  - obsidian-events
  size: small
  description: 'rollout: add the gcal config block and (when chosen) the Mac LaunchAgent
    in chezmoi, reword the gtd_daily calendar checklist row, update and check the
    Mac best-effort, and hand Bryan the short login checklist.'
proposed_by: bbugyi200.apollo.research.2w.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-29 19:40:01
status: wip
bead_id: bob-cli-ea
---

- **PROMPT:** [prompts/202610/bob_gcal_daily_events.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_gcal_daily_events.md)
- **BEAD:** [bob-cli-ea](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ea/README.md)

# Plan: `bob gcal`, Google Calendar events as daily-note Event Tasks

## Why

Calendar commitments are the one part of the day Bob cannot see. Today Bryan does this
by hand: the `gtd_daily` `#pre` row "Review Calendar for today and tomorrow + Create
@EVENT notes", the `bob.md ^sync-gcal-meets` note ("Use `[root::meet]` for 'meeting
tasks'!"), and hand-made MEET sessions in daily notes. This epic automates that habit.

The design follows the consolidated research report
`research:202610/bob_gcal_daily_events/bob_gcal_daily_events.md`. Bryan accepted every
recommendation in it, including its open-question defaults:

- Meetings that ended inside a session close as Complete.
- Maybe answers become context rows.
- One root edit on a recurring meeting teaches the series.
- `meet` is violet.
- Missed meetings stay open and are surfaced, never auto-cancelled.

This plan applies the report against today's code. It also corrects six points the
report predates or missed:

- **Task types moved.** Since `8d7309ff`, the task field is `[type:: MEET]` and the area
  default is `task_type: MEET`. The legacy `[theme::]` and `theme:` forms are only read,
  never written.
- **Capture can't link daily-note tasks.** Its link writer refuses a task that lives in
  the daily note ("routed note and Bob daily note must be different files"). Sync builds
  its `[[#^id]]` links from the crate-visible `capture_task_toggle` helpers.
- **No placement helper fits.** Nothing places a new named entry after the last open
  entry; this epic adds one.
- **Cancel can't record a reason.** `plan_task_cancel` drops the reason unless a
  `❌ **CANCEL LOG**` child already exists, and no Rust code creates one. This epic adds
  an opt-in creator.
- **No shared escaper or runner.** gkeep's title escaper and its hardened subprocess
  runner are module-private. This epic moves both to shared modules; nothing is copied.
- **Hooks delete empty entries.** Task-status hooks delete every childless ledger entry.
  So sync creates a MEET placeholder only when it is about to link at least one meeting.

## What Bryan sees

````markdown
- [*] #task [[gtd_daily]] [root:: gtd_daily] [created::2026-10-30] ^gtd

```bob-plan

```

## Events

- **all day** School holiday
  [📅](https://www.google.com/calendar/event?eid=… "Open in Google Calendar")
- [*] #task **0800-0830** Product interview
  [📅](https://www.google.com/calendar/event?eid=… "Open in Google Calendar")
  [📹](https://teams.microsoft.com/… "Join call") [root:: job] [type:: MEET]
  ^product-interview
- **0915-1000** Toddler class · _maybe_
  [📅](https://www.google.com/calendar/event?eid=… "Open in Google Calendar")
- [*] #task **1830-1900** Dentist
  [📅](https://www.google.com/calendar/event?eid=… "Open in Google Calendar")
  [📍](https://www.google.com/maps/search/?api=1&query=… "Main St Dental") [root:: body]
  [type:: MEET] ^dentist

## Pomodoros

- [ ] () — PLAN
  - [[#^gtd]]
- [ ] () — MEET
  - [[#^product-interview]]
  - [[#^dentist]]
````

The flow:

1. Obsidian opens today's note, and `## Events` shows the day's real commitments, each
   rooted where its time belongs.
2. A MEET placeholder sits after the planned work, so a bare `=` still starts PLAN.
3. Type `=meet` when a meeting starts, then press Ctrl+Enter (or capture `x`) when it
   ends. The meeting that ended inside the session is done and its minutes land on its
   root. Later meetings are deferred to the carried MEET placeholder, and nothing is
   left to clean up tomorrow.

## Design

### Inclusion rule (which events, and as what)

Each event goes through these checks in order. The first match decides:

| Event                                                                                                                                     | Result                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `eventType` not `default` / `fromGmail` (focus time, out of office, working location, birthday), or `status: cancelled`                   | omitted                                                         |
| Your answer is declined or needs-action                                                                                                   | omitted (`list -a` shows why)                                   |
| Timed, under 24 h, starts on this local day, busy (`transparency` ≠ `transparent`), and you accepted it, organize it, or it has no guests | **Event Task** (checkbox, root, MEET link)                      |
| Same, but answered Maybe                                                                                                                  | **context row** (`gcal.tentative: task` makes it an Event Task) |
| All-day, multi-day (shown on every day it covers), or free, and you said yes or maybe                                                     | **context row**                                                 |

- **Your answer** comes from your own entry in `attendees[]` (`self: true`). An
  organizer whose `organizer.self` is true counts as yes. So does an event with no
  `attendees` list.
- **Day ownership.** A timed event under 24 h belongs only to the local day it starts
  on, even when it crosses midnight. A multi-day event appears as a context row on each
  day it covers.

### Event Task grammar (owned by bob-cli, `docs/reference/vault-format.md`)

- **Event Task:** a `#task` line, inside a canonical daily note's `## Events` H2, whose
  text after the checkbox and an optional `#task ` tag starts with `**HHMM-HHMM**`.
  - A span whose end is earlier than its start crosses midnight.
  - Hand-written Event Tasks qualify, so the clock close works without Google.
- **Written task row:**
  `- [<s>] #task **HHMM-HHMM** <title> [📅](<htmlLink> "Open in Google Calendar")[ [📹](<join> "Join call")][ [📍](<maps url> "<location>")] [root:: <root>][ [type:: MEET]] ^<block-id>`
- **Written context row:**
  `- **all day** <title>[ · *day N of M*][ · *maybe*][ · *free*] [📅](…)`, or the same
  with `**HHMM-HHMM**` for a timed Maybe or free event. It has no checkbox and no root.
- **Clock language.** Times are 24-hour local times in the vault's `timezone:`. Day
  windows run from local midnight to local midnight, computed through the time zone,
  never as +24 h, so DST days are right. Rows are sorted all-day first, then by start
  time.
- **Badges:**
  - `📅` is the identity link, Google's `htmlLink`. It is also the adoption key.
  - `📹` appears only when there is a join link. Use `hangoutLink`, else the video entry
    point in `conferenceData`, else a URL found in `location`.
  - `📍` appears for a non-URL location, as a Maps search URL.
  - Descriptions and attendee lists are never imported.
  - `📅` is also the Obsidian Tasks due signifier, but only when a date follows it.
    gcal-sync adds a regression test proving no Bob parser reads the badge as a due
    date.
- **Titles are data.** Escape them through the shared escaper (moved from gkeep), which
  also neutralizes `🔁` and Tasks emoji signifiers. A resolved `@route` token is
  stripped from the title.
- **Fields:**
  - `[root:: X]` goes on every Event Task.
  - `[type:: MEET]` is written only when the PT13 suggestion for the line, with its
    root, is not MEET. So `[root:: meet]` needs no type field once `meet.md` declares
    `task_type: MEET`, and `[root:: job]` does need one.
  - Fields are inserted with the shared setters (`set_task_theme`, then
    `set_task_root`), giving `… [root:: job] [type:: MEET] ^id`.
  - No `[created::]`, `[scheduled::]`, or `[id::]`.
- **Block IDs** come from `capture_block_ids::mint_block_id` over the bare title. They
  are unique within the note, use `event` as the fallback stem, and never change after
  creation.
- **Status:**
  - `[*]` when sync links the row under MEET. This matches the sticky-lanes promotion,
    and the R9 daily-note carve-out clears an unlinked `[*]` later.
  - `[ ]` when the event had already ended before its first sync.

### Choosing the root (first valid match wins)

1. **Series memory** (gcal-lifecycle). Your earlier edit to `[root::]` on another
   occurrence of the same recurring event.
2. **An `@route` token in the event:**
   - either the title's final whitespace-separated token (`Dentist @body`), or a
     description line that is exactly `@body`;
   - resolved through `parent_notes::resolve_parent`, which takes exact stems, then
     project aliases, and rejects inbox kinds and terminal projects;
   - an unresolvable token stays in the title as prose and adds a `route_unknown`
     warning.
3. **The event's colour**, through `gcal.colors`. Keys are Google's event-colour names
   or ids: `lavender`=1, `sage`=2, `grape`=3, `flamingo`=4, `banana`=5, `tangerine`=6,
   `peacock`=7, `graphite`=8, `blueberry`=9, `basil`=10, `tomato`=11. Colour is the only
   channel that works on invitations you can't edit.
4. **The calendar**, through `gcal.calendars[].root`.
5. **The default**, `gcal.root` (default `meet`).

Every candidate must be valid before it is written:

- **What valid means:** not an `is_invalid_root_value`, resolves through the homes
  resolver relative to the daily note, is not `@unfiled`, and is its own root parent (an
  area at any depth, or a root project).
- **What happens otherwise:** the candidate falls back to the next source with a warning
  naming R11's reason and fix.
- **If even the default fails:** no root field is written, and `doctor` fails loudly.
- **Once written, `[root::]` is Bryan's.** Bob never rewrites it.
- **What the source labels look like** in list and sync output: `via @body`,
  `via colour tomato`, `via calendar Health`, `default`, or
  `learned from your 2026-10-29 edit`.

### MEET placeholder and links

- **Only today gains links.** Other days get rows only.
- **Which rows get a link:** an Event Task row whose event has not ended at sync time
  gets exactly one `\t- [[#^id]]` link. It is written once, ever, and recorded in the
  journal. A removed or moved link is never re-added.
- **Which entry the link goes into:**
  1. An open entry whose canonical name is exactly MEET, preferring a running timed one.
  2. Otherwise a new `- [ ] () — MEET`, inserted right after the child block of the last
     open entry. With no open entry, it goes after the last entry. In an empty section,
     it becomes the first line.
  - Placement uses a new `capture_task_toggle` helper beside `insert_named_placeholder`.
  - The entry is created only in the same write that adds its first link, so sync never
    makes a second open MEET (`duplicate_open_pomodoro_name`).
- **MEET is not exempt.** It stays inside the plan budget.

### Closing MEET by the clock (event-close, event-close-js)

**Which links the rule covers.** Plain-marker links in the closing session whose target
is an Event Task in the same daily note: `[[#^id]]`, or a link that resolves to this day
file. Links to other notes are unchanged.

**The inputs:**

- **S** is the session's start, taken from its range.
- **C** is the close time:
  - Rust `x` uses `now`.
  - Ctrl+Enter uses now when the note is today's note, and the session's written end
    otherwise.
  - An untimed `()` close has no clock, so the rule is skipped.
- All values are in minutes from the note's midnight. Add 1440 to the event end when the
  span crosses midnight, and to C when C < S.

**The clock classes:**

| Class          | Test (in order) | Default outcome                                                           |
| -------------- | --------------- | ------------------------------------------------------------------------- |
| `not_started`  | start ≥ C       | **Deferred**: removed from the session, carried, not counted, not started |
| `ended_before` | end ≤ S         | **Deferred**                                                              |
| `ended_inside` | end ≤ C         | **Complete** `[x]`, counted                                               |
| `running`      | otherwise       | today's default (In Progress, carried)                                    |

**How the rule meets close selections:**

- An explicitly numbered link always wins: `x<N>`, `*P`, `!M`, `~K`.
- If the user didn't number a link:
  - `not_started` is Deferred, even under the `!` / `*` wildcards.
  - `ended_inside` is Complete.
  - `ended_before` is Deferred unless a wildcard applies.
  - `running` keeps whatever it would get today: InProgress for a bare `x`, Deferred
    when an `x<N>` list leaves it unlisted, and the wildcard's outcome under `!` / `*`.

**Unchanged:**

- Embedded and `#`-deferred markers keep their authored meaning.
- The `x0` reset path, Work Log, successor offers, and the duplicate-target conflict
  check stay as they are.

**JSON.** `pomodoro_close.task_links[]` gains an additive, optional
`clock: {class, span}` (span such as `"0800-0830"`) on Event Task links. It is present
even when an explicit selection won. `source` keeps its existing values.

**Human output.** Event Task rows get a dim caption:

- `ended 08:30`
- `running until 11:00`
- `starts 14:00`
- `ended 07:30 · before this session`

### How sync works

**One bounded, idempotent reconciliation of one day** (default today, `-D` for another
day whose note exists):

1. **Preconditions.**
   - If `gcal.writer` is set and this host differs, exit 2 with a hint. `-d` is allowed
     anywhere.
   - If the day's note or its `## Pomodoros` section is missing, exit 0 with
     `waiting for 2026/20261030.md`. Bob never creates daily notes; Templater does.
2. **Fetch outside any lock.** Fetch one calendar per gog call, all pages, with a field
   mask that requests only `attendees(self,responseStatus)`. Any failure means the
   snapshot is incomplete: write nothing and exit 1. A 403, an offline host, or a
   missing calendar is never evidence of cancellation.
3. **Classify, choose roots, and fold the journal.** Journal state is keyed by
   (occurrence key, day). The occurrence key is calendar id plus the event's instance
   id, which survives a reschedule.
4. **Plan against an in-memory snapshot of the note:**
   - insert new rows in time order;
   - adopt a row whose `📅` URL already appears in the note without a journal record
     (crash recovery);
   - tombstone a journaled row whose line is gone (never re-create it);
   - add MEET links once.
   - gcal-lifecycle adds span updates, cancels, and RSVP transitions.
5. **No-op check.** A byte-identical plan writes nothing and appends nothing.
6. **Write.**
   - Take `ob::acquire_lock_waiting` (30 s), re-read and re-plan, then
     `capture::write_staged_files`, which is preimage-checked and all-or-nothing.
   - On a `NoteChanged` refusal, re-plan once; then exit 1.
   - Append the journal only after the note write succeeds.
   - Record host-local status (last run, outcome, error) under
     `$XDG_STATE_HOME/bob/gcal/` for `doctor`. This is never written to the vault.
   - Commits are left to vault-sync, as with capture.

**Who owns each part of a row:**

| Part                          | Owner      | Sync behavior                                                                                                 |
| ----------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------- |
| Checkbox                      | Bryan      | Never changed, except cancelling an _untouched_ task (gcal-lifecycle)                                         |
| Time span, title, badges      | Google     | Updated in place only while byte-identical to what Bob last wrote; else kept and reported once as `kept_edit` |
| `[root::]`                    | Precedence | Written once. A difference from what Bob wrote is Bryan's edit; on a recurring event it teaches the series    |
| `[type::]`, block ID          | Bob, once  | Never rewritten                                                                                               |
| Child bullets (meeting notes) | Bryan      | Never touched                                                                                                 |
| MEET Task Link                | Bob, once  | Never re-added or moved; removed only by a cancel                                                             |
| The whole row                 | Bryan      | Deleted or moved away: tombstoned, never re-added                                                             |

**"Untouched"** means not done, not In Progress, never linked under a closed Pomodoro,
and with no 🍅 history. A worked task is never cancelled; it is reported once.

**The journal** is `.bob/gcal/<host>.jsonl`, written through `pref_journal` (`v: 1`,
`ts`, `host`, `kind`; union-merged; malformed lines skipped with a bounded warning).

- Record kinds: `gcal.row`, `gcal.link`, `gcal.gone`, and (gcal-lifecycle)
  `gcal.series_root`.
- Records carry the occurrence key, iCalUID, series id, day, note path, block id, row
  kind, short SHA-256 hashes of each managed span, and the root with its source.
- Records never carry titles, descriptions, attendees, or tokens.

### The `bob gcal` command

`bob gcal` sits in the Integrations help section after `gkeep`. Its membership rule:
_commands that read Google Calendar or project it into the vault._

**Conventions:**

- Options are alphabetical, and every public long option has a short alias.
- Exit codes follow gkeep: 0 for success (including "nothing to do" and "waiting"), 1
  for a runtime failure, 2 for usage or setup errors.
- JSON carries `schema_version: 1` and never includes tokens, descriptions, or
  attendees.
- Output uses ● dots in R8 area hues for roots and ■ swatches for types (R7: places wear
  dots, names wear swatches), honors `NO_COLOR`, and stays silent on a broken pipe.

| Subcommand                                          | Purpose                                                                                                                                             | Writes                |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| `calendars [-f table\|json]`                        | Calendars with id, name, access role, colour, synced?, and mapped root, so config is copy-paste                                                     | —                     |
| `doctor [-b DIR] [-f human\|json]`                  | Checklist with a fix per failure                                                                                                                    | —                     |
| `[list] [-a] [-b DIR] [-D DAY] [-f table\|json]`    | **Bare default.** The day's events beside the vault: time, RSVP, root and its source, and what sync would do. `-a` adds omitted events with reasons | —                     |
| `login [-a EMAIL] [-c FILE] [-m]`                   | One-time consent plus a live check                                                                                                                  | token store (via gog) |
| `logout [-a EMAIL]`                                 | Remove Bob's Calendar grant only; vault untouched; prints the Google revoke URL                                                                     | token store           |
| `sync [-b DIR] [-D DAY] [-d] [-f human\|json] [-q]` | Reconcile one day. `-d` prints the exact unified diff of the note                                                                                   | daily note, journal   |

**Rejected subcommands:**

- `pull`: in gkeep it means drain; this is a reconcile.
- `prune`: cancelling belongs inside sync.
- `root`: the task line already is the UI.
- `setup`: folded into `login` and `doctor`.
- `open`, `watch`, `push`, and event writes: out of scope; `gog calendar` already does
  them.

```text
$ bob gcal
bob gcal · Fri 2026-10-30 · primary · synced 3 min ago on kellys-macbook-pro
TIME         EVENT               RSVP         ROOT                    VAULT
all day      School holiday      ✓ own        —                       context
08:00–08:30  Product interview   ✓ going      ● job    colour tomato  task · MEET
09:15–10:00  Toddler class       ? maybe      —                       context
12:00–12:30  Design review       · no reply   —                       — reply in Calendar
18:30–19:00  Dentist             ✓ own        ● body   @body          task · MEET
2 tasks (2 under MEET) · 2 context · 1 awaiting your reply

$ bob gcal sync
bob gcal sync · 2026/20261030.md
  + task     08:00–08:30  Product interview   ● job   via colour tomato
  + task     18:30–19:00  Dentist             ● body  via @body
  + context  all day      School holiday
  + context  09:15–10:00  Toddler class       maybe
  + MEET     created () — MEET · linked 2
✓ Events +2 tasks, +2 context · Pomodoros +1 entry, +2 links

$ bob gcal login
Google Calendar · one-time setup · read-only

1/3  OAuth client   ✓ bob-gcal (Desktop app)
2/3  Sign in        Opening your browser…
                    ✓ granted: view your calendars (read-only)
3/3  Check          ✓ 8 calendars (syncing 1: primary) · today: 2 tasks, 2 context

Token stored by gog in the macOS Keychain. Remove it with: bob gcal logout
Next: bob gcal sync -d    preview today's Events

$ bob gcal doctor
bob gcal doctor · kellys-macbook-pro
✓ gog           v0.44.0 · /opt/homebrew/bin/gog
✓ client        bob-gcal
✓ account       read-only Calendar access
✓ token         refreshes
✓ calendars     primary reachable
✓ default root  meet → meet.md · area ● violet · suggests ■ MEET
✓ today         2026/20261030.md has ## Pomodoros
✓ journal       .bob/gcal/*.jsonl tracked and union-merged
✓ writer        this host
· last sync     3 min ago · +2 tasks
```

A failing check prints `✗ <check>  <problem>` plus `  fix: <one command or step>`.
`invalid_grant` explains the 7-day expiry for consent screens left in Testing: "publish
the consent screen, then run `bob gcal login`".

### Login (easy on day one and in week ten)

`bob gcal login` drives the `gog` CLI (gogcli, `github.com/openclaw/gogcli`, ≥ v0.44.0)
under an isolated `--client bob-gcal` token bucket with read-only scope
(`--services calendar --readonly`, which grants `calendar.readonly`). Removing or
re-consenting Bob's grant therefore never touches the Gmail token that `sase_gmail`
uses.

1. **Client.** Use an already-stored `bob-gcal` client when one exists. Otherwise
   `-c FILE` runs `gog auth credentials set FILE --client bob-gcal`. Otherwise, on a
   TTY, offer to reuse another stored gog client ("Reuse gog client 'default' for
   Calendar? [Y/n]"). Otherwise print a four-step checklist and exit 2:
   1. Enable the Calendar API in the Cloud project.
   2. Set the consent screen to External and **In production** (unverified is fine for
      personal use).
   3. Download a Desktop app client.
   4. Run `bob gcal login -c PATH`.
2. **Account.** `-a`, else `gcal.account`, else the sole gog account, else a TTY prompt,
   else exit 2.
3. **Sign in.** Run
   `gog auth add <email> --client bob-gcal --services calendar --readonly` with
   inherited stdio, adding `--manual` under `-m`. When `SSH_CONNECTION` is set without
   `-m`, first print a one-line hint to use `-m`. The device-code flow is not offered
   because Google forbids Calendar scopes there.
4. **Check.** List calendars and today's classified counts. An `accessNotConfigured`
   error prints the Console link that enables the API.

Login is idempotent: when a valid grant already exists, it reports "already connected"
and runs only the check.

### Configuration (`~/.config/bob/config.yml`)

```yaml
gcal:
  account: you@example.com # optional when gog holds exactly one account
  writer: kellys-macbook-pro # optional; the one host label allowed to sync
  root: meet # default [root::]
  calendars: # default: [primary]
    - id: primary
    # - id: <calendar id from `bob gcal calendars`>
    #   root: body
  colors: {} # e.g. {tomato: job, sage: body}
  tentative: context # context | task
  timeout_secs: 60
```

- A missing block means defaults, and unknown keys are ignored.
- A wrong-typed value is a setup error (exit 2) that names the key, and it never breaks
  other config loaders. Parse it like `plan:`: a hand-walked `Option<serde_yaml::Value>`
  on `RawConfig`.
- The host label is `vault_sync::hostname_slug()`, the label vault-sync uses in commit
  subjects. `doctor` prints it.

### Where it runs

The Mac is the only writer:

- Templater creates daily notes there.
- Vault-sync's "remote wins" conflict policy would quarantine Bryan's live ledger edits
  if another host wrote today's note.
- The browser login and Keychain are local.

Other hosts may run `list`, `calendars`, `doctor`, and `sync -d` after their own login.

## Conventions for every phase

**bob-cli:**

- Gate with `just check`. When CLI help or docs change, run `just docs-gen` and
  `just docs-check`.
- Read the `cli_rules.md` and `docs.md` reference memory before touching CLI or docs.
- Every change here is additive. Use Conventional Commit subjects without `!`. Never
  edit the Cargo.toml version or CHANGELOG.md.
- **Tests never touch Google.** `BOB_GCAL_GOG` points at a fake gog script, modeled on
  `tests/gkeep_support/` with an isolated `HOME`, `BOB_DIR`, `BOB_CONFIG_FILE`, and lock
  and state paths. Its default is `env_remove`d.
- Recorded fixtures are redacted: no real titles, emails, ids, or URLs. Docs use
  placeholders such as `you@example.com` (see `docs/_meta/privacy_baseline.txt`).

**bob-plugins** (open it with `/sase_repo`, then read its `AGENTS.md`):

- Edit only `plugins/<id>/src/*.js` fragments. Register new fragments in
  `src/fragments.json`, keep each fragment at or under 1000 lines, and never hand-edit
  `main.js`.
- Run `npm run build` and `npm test`. Add each new test file to `package.json`'s list.
- Deploy with `bob plugins sync`.
- Shared vectors are byte-for-byte copies of bob-cli's, with a `README.md` naming the
  source commit.

**Vault (`~/bob`):**

- Edit directly; it is not a repo checkout.
- Check `git -C ~/bob status --short` first.
- Apply and commit while holding the maintenance lock:
  `flock -w 120 "${BOB_VAULT_SYNC_LOCK_FILE:-${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock}" …`.
- Then run `bob vault-sync`, and confirm `bob vault-sync status --json` shows no new
  conflicts.

**Linked and external repos** (bob-plugins, chezmoi, gogcli): open only through
`/sase_repo`.

**Memory:**

- Edit only under an accepted memory decision, using `/sase_memory_write`, then run
  `sase memory init`.
- Decision records follow the house shape: Applies to, Claim, Why (with rejected
  alternatives), Cost, Reopens when. They cite this epic's plan and the research report.

## Phase meet-area: Modernize meet.md and prepare the vault

This phase edits `~/bob` directly, plus one bob-cli docs line.

1. **`~/bob/meet.md` frontmatter.** Today the note is an untyped zorg note with
   `parent: [[work]]`. Make it a top-level area, keeping every `generated_from_zorg` /
   `zorg_*` key unchanged, and follow `job.md`'s key order:

   ```yaml
   parent: "[[area]]"
   type: "[[area]]"
   color: violet
   task_type: MEET
   aliases:
     - Meetings
   tags:
     - meet
   id: meet
   ```

   - Use the canonical `task_type:`, not the legacy `theme:`.
   - `violet` is the only free R8 slot besides red. `dev` already owns blue.

2. **`~/bob/meet.md` body.** The change is additive:
   - Replace the bare "Meeting Notes" first line with a one-paragraph intro: "Meetings,
     appointments, and calls. Calendar events land in each day's `## Events` with
     `[root:: meet]` unless the event names another root (see `bob gcal`)."
   - Keep the "Related notes" list.
   - Insert an empty `## Tasks` H2 above the first dated H2.
   - Change no legacy line or `^z-…` id. About 515 inbound `[[meet…` references must
     keep resolving.
3. **Journal git rules.**
   - In `~/bob/.gitignore`, add `!/.bob/gcal/*.jsonl` with a comment, beside the themes
     and relations entries.
   - In `~/bob/.gitattributes`, add `.bob/gcal/*.jsonl merge=union` with the same style
     of comment.
4. **Commit** under the lock (message
   `vault: make meet a top-level area and allow the gcal journal`), then sync.
5. **Verify.**
   - `bob types suggest meet` shows `■ MEET suggested · from meet`.
   - `bob stats show worked -r 1w -f json` reports no `area_hue_collision` and no
     `area_color_invalid`.
   - `meet` resolves to the vault-root `meet.md`. R11 tries the exact vault path before
     basenames, so no `_generated/tag_pages/*/meet.md` can shadow it. `bob gcal doctor`
     re-checks this later through the real resolver.
6. **bob-cli docs.** In `docs/pomodoro-homes.md` R8, add `meet: violet` to "Bryan's
   current top-level area assignments". Commit
   `docs(homes): record meet's violet area hue`.

## Phase event-close: Event Task grammar and the clock-aware MEET close (Rust)

1. **`src/native/event_tasks.rs`** (new; register it in `src/native.rs`). It holds the
   one Rust definition of Event Tasks, and gcal reuses it:
   - `events_section_range(lines) -> Option<Range<usize>>` mirrors
     `pomodoro::pomodoros_section_range`: skip frontmatter and fences, take the first
     `## Events` H2, and end at the next H2.
   - `parse_event_span(line) -> Option<EventSpan { start_min, end_min, crosses_midnight }>`
     reads a checkbox line, an optional `#task `, then `**HHMM-HHMM**` followed by
     whitespace or end of line. Hours run 00–23 and minutes 00–59. A span with end ==
     start is zero-length.
   - `event_task_at(rel_path, contents, line_index) -> Option<EventTask>` uses the
     canonical-daily check that task-status hooks already use (`canonical_daily_date`),
     plus section membership and the span.
   - `ClockClass { NotStarted, EndedBefore, EndedInside, Running }` and
     `classify(span, session_start_min, close_min)` implement the table in Design,
     including midnight normalization.
2. **Close integration** (`src/native/capture_pomodoro_close/`):
   - `plan_pomodoro_close` builds a `CloseClock { session_start, close_at }` from the
     running entry's range and `now`.
   - `number_task_links` / `apply_close_selection` take `Option<&CloseClock>`. For each
     Plain-marker link whose target is this day file, they find the target block's line
     in `contents`, attach `clock: Option<(ClockClass, EventSpan)>`, and resolve the
     outcome with the rules in Design, extending `outcome_for`.
   - Keep the duplicate-target conflict check, Work Log, carry, successor offers, and
     the `x0` reset unchanged.
   - Add the additive JSON `clock` object and the human captions.
3. **Shared vectors** in `tests/fixtures/event_task_close/vectors.json`, with a
   `README.md` naming the JS mirror. The Rust tests read them through `include_str!`.
   - `classify` cases. Each case is (S, C, span) mapped to a class. Cover:
     - boundary equalities: start == C is not started, end == S ended before, and end ==
       C ended inside;
     - zero-length events;
     - meetings that cross midnight;
     - a session that crosses midnight.
   - `close` cases. Each case is a mini daily note (Events + Pomodoros), `now`, and an
     optional typed selection, mapped to the expected note and per-link outcomes.
     - Cover the bare close with all four classes.
     - Cover a session whose only links are deferred meetings. Pin today's Rust output
       for it; the JS mirror must match.
     - Cover a link to an Event Task in another daily note, which stays unchanged.
     - Cover an untimed close.
     - Mark cases that use typed selections (`x2`, `x!`, `x*`, `x!1`) `rust_only`.
4. **Tests:** unit tests for the parser and classifier, plus `apply_close_selection` and
   full `bob capture 'x'` acceptance runs with `BOB_NOW`.
5. **Docs:**
   - `docs/capture.md`, in "Closing the running Pomodoro": a short "Event Task links
     close by the clock" subsection with the class table and selection precedence, the
     `clock` JSON field, and the captions.
   - `docs/reference/vault-format.md`: the `## Events` section and the Event Task
     grammar, saying hand-written rows qualify.
   - `docs/reference/json-outputs.md`, if it lists `task_links` fields.
6. **Memory:**
   - > [!decision] glossary
     >
     > - Add a `glossary` strand **Event Task** (aliases: event row, calendar task): the
     >   grammar, `## Events` residence, its `[root::]`, MEET linking, and the clock
     >   close. Link `[[area-note]]`, `[[pomodoro]]`, `[[task-link]]`, and
     >   `[[root-parent]]`.
     > - Edit the Area Note strand's exception sentence to list each daily note's Event
     >   Tasks under `## Events` beside `^gtd`, linking `[[event-task]]`.
   - > [!decision] decision_records Add
     > `decisions/event-task-links-close-by-the-clock.md`.
     >
     > - **Applies to:** bob-cli and bob-plugins.
     > - **Claim:** the class table and the precedence rules.
     > - **Why:** a plain close today marks every linked meeting worked and In Progress,
     >   carries it, and splits the minutes evenly. Rejected: per-meeting placeholders
     >   (they still carry on a plain `x`, and duplicate names are blocked); manual
     >   peeling.
     > - **Cost:** Rust and JS must stay in lockstep through the vectors; back-to-back
     >   meetings in one session still split time evenly.
     > - **Reopens when:** same-type Pomodoro duplicates land, or meetings need minute
     >   precision.
7. **Gate:** `just check`, `just docs-gen`, `just docs-check`. Commit
   `feat(capture): close Event Task links by the clock`.

## Phase event-close-js: Clock-aware Ctrl+Enter close in Obsidian

This phase works in the bob-plugins plugin `task-status-cycler`.

1. **Pure helpers.** Add a new fragment, for example `src/065-event-task-clock.js`.
   Mirror the Rust Events section range, span parse, and `classify` exactly.
2. **Hook.** In `completeActivePomodoroTask` (`src/170-plugin-pomodoro.js`), add a pass
   after `classifyPomodoroSubBullets` and before
   `startPomodoroNonTranscludedTaskBullets`. For each plain-link bullet whose target is
   an Event Task in the same note:
   - **`ended_inside`:** close the target the way the link gesture does
     (`replaceResolvedTranscludedTaskLine(…, "x")`), strike the bullet, and record it in
     `closed`.
   - **`not_started` / `ended_before`:** append `#` so the existing plan defers and
     carries it.
   - **`running`:** leave it alone.

   The existing re-read then feeds `buildPomodoroCompletionPlan` unchanged. Supply S and
   C as Design specifies, and skip the pass for untimed sessions.

3. **Parity.**
   - Copy the vectors to
     `plugins/task-status-cycler/scripts/fixtures/event_task_close/vectors.json`, with a
     README naming the bob-cli commit.
   - Add `scripts/test-task-status-cycler-event-clock.cjs`, which runs every
     non-`rust_only` case through the built plugin.
   - The Work Log and the emptied-entry behavior must match the Rust bytes.
4. **Docs.** Update the plugin reference page for Ctrl+Enter Pomodoro completion with
   the clock rule.
5. **Ship.** Run `npm run build`, `npm test`, and `bob plugins sync`. Commit
   `feat(task-status-cycler): close Event Task links by the clock`.

## Phase gcal-auth: bob gcal skeleton, gog transport, login, logout, doctor, calendars

1. **Shared runner.**
   - Move gkeep's private `run_request`, `spawn_adapter`, `kill_adapter_tree`, and
     `kill_stragglers` (`src/native/gkeep/adapter.rs`) into a crate-visible module, for
     example `src/native/subprocess_runner.rs`. Its API should look like
     `run(spec: RunSpec{program, args, stdin, env, timeout}) -> RunOutput{status, stdout, stderr, timed_out}`.
   - Keep the process group, timeout kill, straggler cleanup, and ETXTBSY retry.
   - gkeep's behavior and tests must stay byte-identical.
   - Leave `highlights_ref/clip_adapter.rs`'s copy alone, and file a follow-up through
     `/sase_new_task`.
2. **Module `src/native/gcal/`**, mirroring gkeep:
   - `mod.rs`: `GcalError`, with runtime exits of 1 and setup exits of 2.
   - `cli.rs`: the tree for calendars, doctor, login, and logout. `list` and `sync`
     arrive later; until then a bare `bob gcal` prints help.
   - `config.rs`, `gog.rs`, `root.rs`, `login.rs`, `logout.rs`, `doctor.rs`,
     `calendars.rs`, and `ui.rs` (glyphs, R8 hue dots, type swatches, `NO_COLOR`).
3. **Config:** the `gcal:` block exactly as in Design. Document it in
   `docs/reference/configuration.md`.
4. **Transport (`gog.rs`):**
   - **Executable.** Use `BOB_GCAL_GOG`, else `gog` on `PATH`, resolved to an absolute
     path for doctor.
   - **Common argv** (no shell):
     `--no-input --json --color never --readonly --client bob-gcal --account <email>`.
   - **Operations:** `version`, which needs at least 0.44.0; `credentials_list`;
     `accounts` (`gog auth list --json`); and `calendars`.
   - **Error mapping.** Map stderr and JSON errors to `gog_missing`, `gog_too_old`,
     `no_client`, `not_logged_in`, `invalid_grant` (with the Testing-mode hint),
     `insufficient_scope`, `api_disabled`, `network`, `timeout`, and `bad_json`.
   - **Hygiene.** Never log tokens and never put secrets in argv.
   - **Verify against source.** Before coding, confirm every flag and JSON shape against
     gog v0.44.0 source: open `gh:openclaw/gogcli` through `/sase_repo`. The events list
     is in `internal/cmd/calendar_list.go`, which passes `calendar.Event` through and
     honors a raw `--fields` mask.
5. **`root.rs`:**
   `validate_root_candidate(vault, daily_rel, name) -> Result<RootTarget, RootRejection>`.
   It covers `is_invalid_root_value`, `HomeResolver::resolve_target`, not `@unfiled`,
   and own-root checks, with R11's reason and fix text. `doctor` uses it now; list and
   sync use it later.
6. **`login`, `logout`, `doctor`, `calendars`:** implement them as Design specifies,
   with human and JSON output.
   - Doctor checks: gog, client, account and scope, a token refresh (a live calendars
     call), configured calendars, default-root validity and its type suggestion, today's
     note, journal git rules, and writer host.
   - Doctor's JSON shape is
     `{schema_version: 1, ok, host, checks: [{id, status, summary, fix}]}`.
   - Match gkeep doctor's exit-code semantics.
7. **Registration:**
   - `src/native.rs`.
   - `src/runner.rs` SUBCOMMANDS: Integrations, after gkeep.
   - Completion `tree.rs` and `kinds.rs` (format choices).
   - Catalog: a `gcal-doctor` row, plus exclusions for login, logout, and calendars.
   - Help tests. Then run `just docs-gen`.
8. **Docs:**
   - A new contract, `docs/gcal.md`, with setup, config, doctor, and calendars; later
     phases append to it.
   - A new guide, `docs/guides/connect-google-calendar.md`. It covers the Cloud Console
     checklist, the publish-don't-Test warning, `-c`, `-m` over SSH, and logout and
     revoke.
   - `mkdocs.yml` nav, `docs/_meta/contracts.yml`, and `docs/reference/json-outputs.md`.
9. **Tests.**
   - Add `tests/gcal_support/mod.rs` with a `FakeGog` that records argv, responds per
     subcommand, and can set the exit code or sleep.
   - Cover these login paths: an existing client, `-c`, reusing another client, the
     checklist, already connected, account resolution, and the `-m` passthrough.
   - Cover these doctor states: each failure kind, an invalid default root, a missing
     journal rule, and a foreign writer.
   - Cover calendars rendering and config errors.
10. **Commit** `feat(gcal): add bob gcal login, logout, doctor, and calendars`.

## Phase gcal-list: Event fetch, inclusion rule, root precedence, and bob gcal list

1. **`gog.rs` events.**
   - `events(calendar_id, from, to)` runs
     `gog calendar events <id> --from <RFC3339> --to <RFC3339> --all-pages --max 250 --fields <mask>`.
   - The mask is items(id, iCalUID, recurringEventId, originalStartTime, status,
     summary, description, location, htmlLink, hangoutLink,
     conferenceData(entryPoints(entryPointType,uri)), start, end, transparency,
     eventType, colorId, attendees(self,responseStatus), organizer(self)), plus
     nextPageToken.
   - `event(calendar_id, event_id)`, for gcal-lifecycle lookups.
   - Any failure fails the whole snapshot.
2. **`model.rs`.** Define `GcalEvent`, with these fields:
   - `key: OccurrenceKey{calendar_id, event_id}`, `ical_uid`, `series_id`, and
     `original_start`;
   - `status`, `title`, and `route_hint` (the parsed description `@route` line only; the
     description itself is dropped);
   - `location`, `html_link`, and `join_link`;
   - `start` / `end` as `Timed(DateTime<FixedOffset>) | AllDay(NaiveDate)`, with an
     exclusive all-day end;
   - `transparency`, `event_type`, `color_id`, and
     `my_answer: Yes | Maybe | No | NoReply`.
3. **Day window.** `-D/--day today|tomorrow|yesterday|YYYY-MM-DD` resolves through the
   configured time zone and honors `BOB_NOW`. Both local midnights are computed through
   the zone.
4. **`classify.rs`.** Implement the inclusion rule and day ownership. It returns
   `Task | Context(why) | Omitted(why)`.
5. **`root.rs` precedence.** Implement sources 2–5, with route-token parsing and title
   stripping, colour names and ids validated at config load, and the per-candidate
   fallback warnings. Leave source 1 as a hook for gcal-lifecycle.
6. **`list.rs`.**
   - Show the table (mock in Design) without the VAULT column, which gcal-sync adds.
   - `-a` adds omitted events with reasons.
   - JSON is
     `{schema_version, day, calendars, events: [{time, title, answer, inclusion, reason, root: {value, source, warnings}}], counts}`.
   - Before login, bare `bob gcal` prints a three-line "not connected yet · run
     `bob gcal login`" message and exits 2.
7. **Registration.** Make `list` the bare default (it is flag-only and read-only), add
   the catalog row `gcal-list`, and update completion and help. Then run
   `just docs-gen`.
8. **Docs:** add the inclusion rule, the root precedence, and the list contract to
   `docs/gcal.md`, and update `json-outputs.md`.
9. **Tests.** Use recorded, redacted fixtures under `tests/fixtures/gcal/` covering:
   - accepted, Maybe, declined, and unanswered events;
   - an organizer with no guests;
   - free, all-day, and multi-day events;
   - every non-default event type;
   - DST forward and back;
   - an event declared in a foreign time zone;
   - a meeting crossing midnight;
   - the exclusive all-day end;
   - malformed JSON;
   - a page failure that makes the snapshot incomplete;
   - each root source and each fallback.
10. **Commit** `feat(gcal): add bob gcal list`.

## Phase gcal-sync: bob gcal sync writes Events rows and MEET links

1. **Shared escaper.**
   - Move gkeep's `escape_task_text` and its helpers into a crate-visible module, for
     example `src/native/task_text_escape.rs`. gkeep's output must stay byte-identical.
   - Add a gcal pass that neutralizes `🔁`, Tasks emoji signifiers, and priority emojis.
2. **`render.rs`.** Render the task and context row grammar exactly as specified in
   Design:
   - badge URL encoding and link-title quoting;
   - block IDs from `mint_block_id` over the bare title, against IDs already in the note
     plus those reserved in this run;
   - the PT13-driven `[type:: MEET]` decision (`pomodoro_themes::suggest_for_task` on
     the line without a type field);
   - the field order;
   - the status (`[*]` when linking, otherwise `[ ]`).
3. **`journal.rs`.**
   - Use the `pref_journal` namespace `{dir: ".bob/gcal", noun: "gcal"}`.
   - Write the records `gcal.row`, `gcal.link`, and `gcal.gone`.
   - Fold them into state keyed by (occurrence key, day).
   - Hash spans as SHA-256, truncated to 16 hex characters.
4. **`plan.rs`** is pure: the note snapshot, events, journal state, config, and `now` go
   in, and `SyncPlan{contents, records, report}` comes out.
   1. Find `## Events`, or create it immediately above `## Pomodoros` with tidy blank
      lines. An existing `## Events` elsewhere is used where it is.
   2. Handle each included occurrence:
      - **Journaled, and its line is present** (matched by block id for tasks, or by the
        `📅` URL for context rows): do nothing in this phase.
      - **Journaled, and its line is gone:** write one
        `gcal.gone{reason: "line_removed"}`.
      - **Not journaled, but the note already has a line with its `📅` URL:** adopt the
        line by recording it, and record its link too if one exists.
      - **Otherwise:** insert the row at its time-ordered slot, after the last parsable
        row that starts at or before it. Never reorder existing lines.
   3. On today only, link each eligible task row once, using the MEET rules in Design.
      Add the new placement helper `insert_named_placeholder_after_last_open` in
      `capture_task_toggle/relocation.rs`, and append links with the existing link
      helpers using tab indentation.
   4. When nothing changed, append no records.
5. **`sync.rs`.** Implement the write pipeline from Design: writer check, waiting exit,
   fetch, lock, re-read, plan, `write_staged_files`, a single retry, journal append, and
   the host-local status file.
   - `-d` prints a unified diff from the same plan and writes nothing.
   - `-q` prints nothing on success, no-op, or waiting.
   - JSON is
     `{schema_version, ok, day, path, status: written|unchanged|waiting|dry_run, rows: […], pomodoro: {entry_line, created, links_added}, warnings: […]}`.
6. **`list` and `doctor`.**
   - Add list's VAULT column: `task · MEET`, `task`, `context`, `waiting for note`, or
     `removed by you`.
   - Add the header's last-sync line, from the host-local status, else the newest
     journal record's host and time.
   - Add doctor's last-sync check.
7. **Registration.**
   - Add the catalog row `gcal-sync` (network; invalidates tasks; PreviewFirst with
     `--dry-run`).
   - Update completion and help, then run `just docs-gen`.
8. **Docs:**
   - `docs/gcal.md`: the sync contract and ownership table.
   - `docs/reference/vault-format.md`: the written rows, badges, and context rows.
   - The guide: a first-sync walkthrough.
   - `json-outputs.md`.
9. **Memory.** > [!decision] decision_records
   > Add `decisions/calendar-events-are-daily-note-event-tasks.md`.
   >
   > - **Applies to:** bob-cli, bob-plugins, and the vault.
   > - **Claim:** synced events live in their daily note's `## Events`, a recorded
   >   residence exception beside `^gtd`. Attribution comes from `[root::]`, and links
   >   go under one appended MEET.
   > - **Why:** daily residence keeps meetings out of the review walk, freshness, Ready
   >   caps, and sticky Next (R9). Rejected: durable series tasks in `meet.md`, a
   >   read-only fence, and one placeholder per meeting.
   > - **Cost:** a residence exception; meetings count against the plan budget.
   > - **Reopens when:** Event Tasks need review or lanes, or same-type duplicates land.
10. **Tests:**
    - A first sync creates Events, MEET, and links. A second run is byte-identical and
      appends nothing to the journal.
    - A deleted row or removed link is never re-added.
    - Crash adoption.
    - Reusing an open MEET, and reusing a running MEET.
    - Placement after the last open entry, and a bare `bob capture '='` still starts
      PLAN.
    - A past day gets rows only. A missing note waits.
    - An incomplete snapshot writes nothing.
    - A preimage change retries once. A non-writer host refuses.
    - `job` gets `[type:: MEET]`; `meet` gets none.
    - Hostile titles (`#task`, a trailing `^id`, `%%`, `::`, `🔁`, `📅 2026-10-30`, a
      leading `#` or `-`).
    - Integration:
      - a closed MEET session linking a `[root:: body]` row attributes to `body` in
        `bob stats`;
      - task-status hooks keep linked rows `[*]`;
      - `capture-pomodoros --tasks` agenda text reads cleanly;
      - Dataview finds no due date from the `📅` badge.
11. **Commit** `feat(gcal): add bob gcal sync`.

## Phase gcal-lifecycle: Reconcile Google-side changes and learn series roots

1. **Three-way span updates.** For each journaled row that is still present and still
   included:
   - Re-render its time, title, and badge spans.
   - A span whose current hash equals the last-written hash, and whose render changed,
     is replaced in place.
   - A span that differs from both is `kept_edit`, reported once (journal flag).
   - A time change re-slots the row together with its child block to keep time order.
   - The checkbox, `[root::]`, `[type::]`, and block ID are never rewritten.
2. **Cancels.**
   - Extend `task_cancel` with an opt-in `create_cancel_log`. It inserts
     `❌ **CANCEL LOG**` plus `*YYYY-MM-DD* — <reason>` as the first child when no log
     exists (docs/projects.md shape). Existing callers stay unchanged.
   - For an untouched task whose event was declined, cancelled, deleted, or moved to
     another day, run `plan_task_cancel` with `source_path == daily_path`.
   - Reasons: "Declined in Google Calendar", "Deleted in Google Calendar", and "Moved to
     Fri 2026-10-31 in Google Calendar".
   - Stage the postimages.
   - In the same write, remove an untimed MEET entry the cancel left childless.
   - A worked task is left alone and reported as `worked_kept` once.
3. **Disappearances and RSVP transitions.**
   - **A journaled occurrence missing from the snapshot:** look it up with
     `event(cal, id)`. A 404 or 410, or `status: cancelled`, means deleted. A start on
     another day means moved. A failed lookup makes the snapshot incomplete, so nothing
     is written.
   - **Declined events and deleted context rows:** remove an untouched context row.
   - **Maybe → Yes:** upgrade an untouched context row in place to a task row, with a
     link when today's meeting hasn't ended yet.
   - **Yes → Maybe:** leave it and report `now_maybe`.
   - **Series splits:** treat them as cancel plus insert, never as a fuzzy match.
4. **Series learning (root source 1).**
   - When a journaled task's `[root::]` differs from the root Bob recorded, and the
     event recurs, append
     `gcal.series_root{series, root, learned_from: {day, block_id}}`.
   - New rows in that series use the learned root (still validated). Sync reports it,
     for example `Weekly 1:1 → job (learned from your 2026-10-29 edit)`.
   - A later edit, including a change back, re-teaches the series.
5. **`list` footer.** List open Event Tasks from the previous 7 daily notes, for
   example:
   `2 past meetings still open: Thu Dentist, Tue Weekly 1:1 · close or cancel them in their notes`.
   Nothing auto-cancels a missed meeting.
6. **Doctor.** Add the last error from host-local status, and the count of malformed
   journal lines.
7. **Docs:**
   - `docs/gcal.md`: transitions, cancel reasons, and series learning.
   - The writers list for `❌ **CANCEL LOG**` in `docs/projects.md` and
     `docs/task-dependencies.md`: add `bob gcal sync`.
8. **Memory.** > [!decision] decision_records
   > Add `decisions/bob-gcal-is-read-only-and-proposes-once.md`.
   >
   > - **Applies to:** bob-cli and the vault.
   > - **Claim:** read-only Google scope; Bob writes each row and link once;
   >   Google-owned spans update only while untouched; the vault owns everything else;
   >   deleted rows are tombstoned; only untouched tasks are cancelled; an incomplete
   >   snapshot writes nothing; the Mac is the single writer.
   > - **Why:** rejected two-way sync, iCal URLs, and Obsidian calendar plugins.
   > - **Cost:** Google-side edits to touched rows are reported, not applied.
   > - **Reopens when:** Bob needs to RSVP or create events.
9. **Tests:**
   - each transition;
   - kept edits, reported once;
   - a worked task that is never cancelled;
   - cancel-log creation, and capture's relation cancel unchanged;
   - removal of an emptied MEET;
   - a failed lookup that writes nothing;
   - series learning and re-teaching;
   - the footer.
10. **Commit** `feat(gcal): reconcile calendar changes and learn series roots`.

## Phase obsidian-events: Obsidian trigger, ledger guard, and Events painting

This phase works in bob-plugins.

1. **Trigger (bob-ledger-tools).**
   - When today's canonical daily note is created in this session (a vault `create`
     event), wait up to 15 s for Templater to give it `## Pomodoros`. Then run
     `bob gcal sync -q -f json` once through `bobSpawnJson`; bob stays the only writer.
   - Exit 2 (not connected, or not the writer host) logs only to the console. Exit 1
     shows at most one Notice per day: "Calendar sync failed · run `bob gcal doctor`".
   - Add a setting toggle, "Sync Google Calendar when today's note is created", on by
     default.
2. **Toast (bob-command-mode).** Add `cmRegisterOutcomeAdapter("gcal-sync", …)`
   rendering "Calendar · +2 tasks, +1 context · MEET +2 links". The catalog row appears
   automatically.
3. **Ledger guard (bob-navigation-hotkeys).** The section-move gesture (Ctrl+Shift+],
   `findNextMarkdownSection` / `getNextSectionBulletInsertion`) never moves a column-0
   `#task` row into `## Pomodoros`, where it would read as a ledger entry. It shows a
   Notice instead. This protects every section above the ledger, including Events.
4. > [!decision] events_paint bob-ledger-tools paints `## Events` in Live Preview and
   > Reading view, modeled on `buildPomodoroThemeDecorations`
   > (`src/279-plugin-pomodoro-theme-chips.js`) and its reading counterpart:
   >
   > - **Time pill.** A leading `**HHMM-HHMM**` or `**all day**` becomes a pill in the
   >   ledger's muted style, with tabular numerals and an en dash.
   > - **Now and past (today's note only).** A running meeting's pill gets an accent
   >   ring in the MEET type's hue. Past meetings dim. Refresh on a one-minute timer
   >   only while today's note is visible.
   > - **Context rows.** They render in muted text, and `*maybe*`, `*free*`, and
   >   `*day N of M*` become small muted tags.
   > - **Badges.** The `📅`, `📹`, and `📍` badges sit at 0.85em and brighten on hover.
   > - **Theme and toggle.** Provide light and dark CSS rules, and a setting toggle
   >   "Paint calendar events".
   > - **Pure core.** Put the scanning in a pure fragment so a Node test covers it.
5. **Tests, docs, deploy.**
   - Add Node tests for the trigger gating, the guard, and (when painting) the scanner.
   - Update the plugin reference pages.
   - Run `npm run build`, `npm test`, and `bob plugins sync`.
   - Commit per plugin (`feat(ledger-tools): …`, and so on).

## Phase rollout: Mac rollout, config, and handoff

1. **chezmoi config** (open through `/sase_repo`). In `home/dot_config/bob/config.yml`,
   add a commented `gcal:` block beside `gkeep:`:
   - `account`;
   - `writer`, set to the Mac's host label, which appears in vault-sync commit subjects
     in `git -C ~/bob log`;
   - `root: meet`, `calendars: [primary]`, `colors: {}`, and `tentative: context`.
2. > [!decision] launch_agent Add
   > `home/Library/LaunchAgents/com.bbugyi.bob-gcal-sync.plist`, modeled on
   > `com.bbugyi.bob-vault-sync.plist`:
   >
   > - It runs `exec /Users/bbugyi/.cargo/bin/bob gcal sync -q` with `StartInterval 300`
   >   and `RunAtLoad`.
   > - It reuses the same `HOME` and `PATH`, which must include gog's directory.
   > - It logs to `/var/tmp/com.bbugyi.bob-gcal-sync.{out,err}`.
   > - It is installed and loaded the same way as the vault-sync agent; check the repo's
   >   `.chezmoiignore` and `run_onchange` scripts.
   >
   > When this decision is off, skip the plist. The Obsidian trigger and manual runs
   > remain.
3. **Vault.** Reword the open `gtd_daily` `#pre` occurrence from "Review Calendar for
   today and tomorrow + Create @EVENT notes" to "Review Calendar for today and
   tomorrow", keeping every field. Commit it under the lock.
4. **Mac, best-effort over `ssh mac`** (the Mac is often offline):
   - Update bob to a master build the way that host installs it.
   - Run `chezmoi update`, then `bob gcal doctor`, and confirm the only failing checks
     are the ones login fixes.
   - If the Mac is unreachable, say so in the handoff.
5. **Bryan's checklist.** Write it in the phase bead notes and the final message:
   1. In Cloud Console, enable the Calendar API and confirm the consent screen is
      External and In production.
   2. On the Mac, run `bob gcal login`, or `bob gcal login -c <client json>`. Approve
      the Keychain prompt the first time the LaunchAgent runs.
   3. Run `bob gcal sync -d`, then `bob gcal sync`.
   4. Run `bob gcal doctor` on apollo, which reports the Mac as writer.
   5. Colour privacy: recolour one invitation and ask its organizer whether their copy
      changed. If it did, keep `gcal.colors` empty.
   6. Close `bob.md ^sync-gcal-meets` when happy.

## Not in v1

- Incremental `syncToken` sync, push channels, or pre-creating future days.
- Any write to Google: RSVP, create, or edit.
- More than one MEET entry per day, or per-meeting placeholders.
- Exempting Event Task links from `max_links`, until busy days trip the cap.
- A Mac Capture time chip, as an additive `capture-pomodoros --tasks` field.
- A native Rust HTTP client. Bob has no HTTP or TLS stack, and gog covers the transport.
