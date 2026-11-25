---
tier: epic
title: "Snoozed Tasks: plan it for today, wake it at a time"
goal: "Bryan can snooze any open, non-Event task to a clock time today from its Task
  Link, its definition, capture, or the CLI. While asleep the task is clearly marked
  with a clock and its time, stays out of Today, the walks, and plan.max_links. At its
  time, bob wakes it into an untimed open Pomodoro, the woken Task Link shows its wake
  time, and one hard-to-miss notice announces it and then goes away on its own.

  "
decisions:
  uncommitted_rows:
    ask:
      How should wake treat a due row whose task is Ready (released, or hand-written on
      an uncommitted task)?
    choices:
      ignore:
        Only Next/In Progress tasks wake; a Ready row never revives a released task
      promote:
        A Ready row wakes and is promoted to Next, exactly like a hand-added Task Link
    default: ignore
    why:
      Release stays final with no exceptions; Alt+Z and bob snooze add commit the task
      for you
    answer: ignore
  wake_notice:
    ask: Which Mac surface should announce a wake?
    choices:
      hud:
        Glass HUD at top-center, 12 s, hover pauses, Open/+15m/× actions; Focus cannot
        hide it
      banner:
        System banner through NotificationService; simpler, but Focus and alert style
        can hide it
    default: hud
    why:
      Bob controls visibility and dismissal end to end, which is what "very likely to
      see" needs
    answer: hud
  hold_in_meetings:
    ask:
      Hold the wake notice during a running calendar meeting and deliver it when the
      meeting ends?
    default: true
    why:
      No interruption mid-meeting and no task titles on a shared screen; the beacon
      still lights
    answer: true
  glossary_snoozed_task:
    ask: Add the new Snoozed Task glossary strand?
    memory:
      - glossary:snoozed-task
    requested:
      As a part of this change, we should add an excellent new glossary term for
      "snoozed task".
    default: true
    answer: true
  decision_record:
    ask:
      Record this design as a new decisions memory record (snoozed tasks wake through
      bob)?
    memory:
      - decisions:snoozed-tasks-wake-through-bob
    default: false
    answer: false
phases:
  - id: wake-engine
    title: bob snooze core, wake writer, and tick step
    depends_on: []
    size: large
    description:
      "wake-engine: the snooze module (row grammar, states, time grammar), the bob
      snooze list/add/drop/wake CLI with guarded writes, never-running wake placement,
      struck history rows, the first-position tick step, the wake receipt plus Darwin
      post plus desktop fallback, the trailing-section writer audit, shared vectors, and
      docs/snooze.md."
  - id: planned-today
    title: Snoozed rows in reconcile, walks, plan, and the agenda JSON
    depends_on:
      - wake-engine
    size: large
    description:
      "planned-today: reconcile treats live rows as recovery-only reach and never
      touches them; freshness, ]s, and bob walks exclude snoozed tasks; bob plan gains
      the snoozed meter and forecast lint; capture-pomodoros --tasks gains additive
      snoozed rows and woke_at."
  - id: capture-z
    title: Capture z:<when> snooze token
    depends_on:
      - wake-engine
    size: large
    description:
      "capture-z: the z:<when> token for new and existing-task captures, the
      existing-task grammar exception, link-now and drop interplay with live rows,
      teaching refusals, completions led by today's event end times, a snooze span kind,
      and docs/capture.md."
  - id: plugin-plumbing
    title: Plugin snoozed index, exclusions, cleanup parity, and Obsidian waker
    depends_on:
      - planned-today
    size: large
    description:
      "plugin-plumbing: the ledger-tools JS mirror of the row grammar and live predicate
      on the shared vectors, isSnoozedToday/isPlannedToday api, ]s/]w/dash exclusions
      and the dash.md query edit, cleanup parity in every link-removing gesture,
      Snoozed-row Ctrl+Enter/Alt+N/Ctrl+Shift+Enter semantics, and the minute-tick bob
      snooze wake spawn with its Notice fallback."
  - id: plugin-marks
    title: Clock marks, woken-link pills, Snoozed-row paint, and plan chip
    depends_on:
      - plugin-plumbing
    size: large
    description:
      "plugin-marks: the alarm-clock definition mark that replaces 🍅 while asleep
      (amber when due), the woke-time pill on woken Task Links, Snoozed-row and heading
      paint, the bob-plan snoozed chip, and the docs/snooze-marks.md display contract
      with vectors."
  - id: snooze-card
    title: Alt+Z Snooze Card and Task Card z
    depends_on:
      - plugin-marks
    size: large
    description:
      "snooze-card: the Alt+Z command and Task Card z opening a Snooze Card with ranked
      time suggestions, a consequence footer, retime/Wake now/Unsnooze on snoozed tasks,
      teaching refusals for Event and Blocked tasks, and a pure JS planner pinned to the
      shared vectors."
  - id: mac-agenda
    title: Mac Later-today band, woken accents, and exact-minute waker
    depends_on:
      - planned-today
      - capture-z
    size: large
    description:
      "mac-agenda: decode the additive snoozed/woke_at agenda fields, add a dim
      Later-today band that folds first, woken-row accents, a snoozed count on the
      budget row, z: span styling, and an always-on scheduler that runs bob snooze wake
      at next_wake_at."
  - id: mac-wake-notice
    title: Presence-aware wake notice on the Mac
    depends_on:
      - mac-agenda
    size: large
    description:
      "mac-wake-notice: the notifier claim file, snooze-woken receipt observer, pure
      presence and meeting hold policy, the wake notice surface with chime and +15m, the
      menu-bar beacon, the silent Notification Center record with withdrawal, Settings,
      and Mac docs."
  - id: close-out
    title: Glossary, guide, decision record, and end-to-end verification
    depends_on:
      - capture-z
      - snooze-card
      - mac-wake-notice
    size: medium
    description:
      "close-out: the Snoozed Task glossary strand and regenerated glossary page, the
      decision record if approved, a how-to guide and concepts paragraph, and a live
      end-to-end run of every acceptance case across bob, Obsidian, and the Mac."
proposed_by: bbugyi200.athena.1fx
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 10:22:50
status: wip
---

- **PROMPT:**
  [prompts/202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/snoozed_tasks.md)

# Plan: Snoozed Tasks

Source research (read it with `sase artifact read`, not from disk):
`research:202611/snoozed_tasks/snoozed_tasks.md` (the design Bryan endorsed) and its
predecessor `research:202611/snoozed_task_links/snoozed_task_links.md`. This plan adopts
that design. The places where it changes or settles the research are listed in
[Adjustments to the research](#adjustments-to-the-research).

## The gap

Some work is committed to **today but cannot start before a clock time**: email Dan
after the 15:30 interview, fix the sink once home at 19:00. Today, such a task either
sits linked in the ledger all day, unactionable and counted against the budget, or is
not planned at all. Nothing tells Bryan when it becomes doable. `job#^email-dan-eod` was
exactly this case: it sat linked among about 18 open links, nothing fired at 15:30, it
was carried into the next day, and then it was cancelled. A **Snoozed Task** is the
missing middle: committed today, not Today until its time, and it resurfaces by itself.

Event Tasks are never snoozed. Their `**HHMM-HHMM**` span, MEET filing, clock-aware
closes, and Calendar band already do this job.

## What Bryan sees

### The vault (Markdown stays the source of truth)

```markdown
## Pomodoros

- [x] (**1445-1530** [t:: 45m]) — MEET
  - 🍅 ~~[[#^yogi-interview]]~~
- [ ] () — WRITE
  - [[job#^email-dan-eod]]
    - Mention the new interview timeline.
  - [[sase_blog_drafts#^what-came-before]]

## Snoozed

- ~~**1530** [[job#^email-dan-eod]]~~ · woke 15:32
- **1900** [[home#^fix-sink]]
  - grab the wrench from the garage first
```

`## Snoozed` sits right after `## Pomodoros`. It is created on demand and deleted when
it holds no rows. Each row is one column-0 bullet: a bold 24-hour target time and one
block link, with the link's notes beneath it. When a row wakes, the link and its notes
move to the top of an untimed open Pomodoro. The row stays behind struck, with the
actual wake time, as the day's provenance.

### One token, four states

The token is an **alarm clock** (Lucide `alarm-clock` in Obsidian, SF Symbol `alarm` on
the Mac, `⏰` in the terminal) plus a time. It is never 💤 (the Reading Board's Snoozed
lane) and never ◷ (scheduled project cards).

| State                                     | Task definition (🍅 slot)                 | `## Snoozed` row paint               | Ledger Task Link                                                         | Mac agenda                           | CLI `bob snooze`     |
| ----------------------------------------- | ----------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------ | ------------------------------------ | -------------------- |
| **Asleep**                                | indigo pill `⏰ 15:30`                    | indigo time pill · faint `in 2h 15m` | (not in the ledger)                                                      | dim "Later today" row                | `15:30  in 2h 15m`   |
| **Due** (time passed, write pending)      | amber pill `⏰ 15:30 · due`               | amber `due · waking`                 | —                                                                        | amber `due · waking`                 | amber `due`          |
| **Blocked** (became Blocked while asleep) | muted pill `⏰ 15:30`                     | muted `waits on prerequisites`       | —                                                                        | muted row                            | `blocked`            |
| **Woken**                                 | 🍅, with the existing 600 ms entrance pop | struck row, dim `woke 15:32`         | warm pill `⏰ woke 15:32`, which settles to a quiet outline after 60 min | amber accent `woke 15:32` for 60 min | `woke 15:32 → WRITE` |

- A definition never wears both marks. If a task has a live ledger link, 🍅 wins and the
  next wake prunes the redundant row.
- Mod+click on the clock jumps to the Snoozed row, mirroring Mod+click on 🍅.
- Tooltips carry the words: `Snoozed until 15:30 · in 2h 15m`,
  `Snoozed until 15:30 · due — waking`, `Woke 15:32 (snoozed until 15:30, 2m late)`.
- Shape plus text carry every state; color is never the only cue. Marks are
  display-only, cannot be copied, do not shift line height, and respect reduced motion
  and forced colors.

### Gestures

| Where                                              | Gesture                                              | Result                                                                                                                                          |
| -------------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| A live Task Link in today's ledger (the main case) | **Alt+Z**                                            | Snooze Card. Enter moves the link subtree into `## Snoozed`.                                                                                    |
| A task definition line                             | **Alt+Z** or Task Card **`z`**                       | Snooze Card. Ready → Next plus `[fresh::]`, exactly like Ctrl+Shift+Enter. Any live link moves into the row.                                    |
| A Snoozed row, or a snoozed task's definition      | **Alt+Z**                                            | Snooze Card in "snoozed" mode: retime, **Wake now**, or **Unsnooze** (drop the row; the lane is unchanged).                                     |
| A Snoozed row                                      | Ctrl+Enter / Alt+N / Ctrl+Shift+Enter                | Complete the task and remove the row / release and remove the row / link it now (normal placement) and remove the row.                          |
| Capture                                            | `@job:email-dan-eod z:1530`, `- Fix sink @home z:19` | Snooze an existing task, or create a task already snoozed.                                                                                      |
| Terminal                                           | `bob snooze add job:email-dan-eod 1530`              | Same as above. `bob snooze` lists today's queue.                                                                                                |
| Event Task or Blocked task                         | any of the above                                     | A teaching refusal: `Event Tasks wait on their own clock (19:00–20:30)`, or `Blocked by ‹X› — snooze waits on the clock, not on prerequisites`. |

The **Snooze Card** has one focused time field that accepts `1530`, `15:30`, `3:30p`,
`3p`, `+45m`, `+2h`, and `+1h30m`. A bare `330` means the next 3:30 today. Relative
times round up to the next 5 minutes.

Ranked suggestions come first from today's remaining **event end times**
(`15:30 · after Yogi interview`), then `+30m`, `+1h`, and `+2h`, then times already used
today. The top suggestion is preselected, so **Alt+Z, Enter** is two keystrokes.

The footer states the consequence:
`leaves WRITE · wakes into WRITE at 15:30 · in 2h 10m · TODAY 9/13 now, 10/13 after`.

Past times and times past midnight are refused with teaching copy:
`same-day only — Schedule (s:) defers to another day`. Enter confirms; Esc, `q` (outside
the text field), and Ctrl+[ close, per `decisions:q-and-ctrl-bracket-mirror-esc`.

### The wake notice (the part that must be seen)

A banner that fires while Bryan is away, or mid-meeting, is a banner never seen. The
design therefore splits the **record** from the **nudge**.

- **Record.** This part is durable and is not a notification. It is the 🍅 flip on the
  definition, the `⏰ woke 15:32` pill on the link, the struck row, and the highlighted
  agenda row. It survives restarts, which is what makes auto-dismissal safe.
- **Nudge.** There is exactly one per wake batch, owned by Bob Mac Capture:
  1. `bob snooze wake` commits the write. Only then does it write an atomic receipt and
     post the Darwin notification `org.bobs.bob.snooze.woken`.
  2. Mac Capture checks **presence**. If Bryan is active, the notice shows now. If the
     screen is locked, the displays are asleep, the session is inactive, or he has been
     idle for at least 3 min, the notice is **held**. It is delivered about 1.5 s after
     he returns, once `bob snooze -f json` confirms the tasks are still woken and open.

     > [!decision] hold_in_meetings During a calendar meeting that is running (per the
     > agenda's Events), the notice is held the same way and delivered when the
     > meeting's span ends.

  3. **The surface.**

     > [!decision] wake_notice = hud A non-activating glass HUD sits at top-center on
     > the screen under the pointer. It never takes focus, never appears as a modal, and
     > is excluded from screen capture (`sharingType = .none`). The alarm glyph rings
     > once (no motion under Reduce Motion).
     >
     > It reads `Ready now · Email Dan with interview updates`, with
     > `woke 15:32 · in WRITE · TODAY 11/13` beneath; `14/13` shows in amber when over
     > the cap. A batch reads `3 snoozed tasks are ready` and lists up to three titles.
     >
     > It offers **Open** (also clicking the body), which uses
     > `obsidian://bob-goto-task`; **+15m**, which re-snoozes through `bob snooze add`;
     > and **×**. A thin countdown line runs 12 s, and hover pauses it.
     >
     > A soft system chime plays by default. VoiceOver announces the notice.

     > [!decision] wake_notice = banner A `UNUserNotificationCenter` banner goes through
     > the existing `NotificationService`. It has the same hold, the same copy,
     > Open/+15m actions, and a stable id. Its visibility is subject to Focus and the
     > alert style the user chose.

  4. A **menu-bar beacon** on the status item (a small amber clock with the count) and a
     **silent Notification Center record** (`.passive`, stable id
     `snooze-woken-<wake_id>`). Both clear when the panel opens, when Open is used,
     after 60 minutes, or at day change. Nothing ever needs a manual dismiss.
  5. **Fallbacks.** When no live Mac Capture has claimed notifications (see the notifier
     claim below), bob itself sends `desktop_notify` (osascript or notify-send).
     Obsidian shows an 8 s `Notice` only when its own `bob snooze wake` spawn reports
     `notified: "none"`. That means never a double banner.

The interruption level is normal. Time Sensitive is deferred: it needs an entitlement
the Developer-ID build may not carry. A notification failure never rolls back a wake.

## Rules (the behavioral contract `docs/snooze.md` will own)

1. **A row is live** while all of these hold:
   - It parses (see [Row grammar](#row-grammar)).
   - It is unstruck.
   - It is in **today's** note.
   - Its link resolves to exactly one open `#task` that is **not an Event Task**.
   - That task is committed: Next `[*]` or In Progress `[/]`.
   - The task has no live link under an open Pomodoro.

   A live row whose target is Blocked `[?]` is **blocked**. It stays parked and wakes
   once its target recovers.

   Every other row is **ignored**: readers treat it as absent, `bob snooze` lists it
   with a reason, and plugins paint it muted with that reason. "Truthful or neutral."

   > [!decision] uncommitted_rows = ignore A row whose target is Ready is ignored. That
   > covers a release that missed cleanup and a hand-written row on an uncommitted task.
   > It never wakes, so a release stays final. `bob snooze` lists it as
   > `ignored · Ready — snooze with Alt+Z or bob snooze add to commit it`.

   > [!decision] uncommitted_rows = promote A row whose target is Ready is live. Wake
   > promotes the target to Next as part of the move, the same rule as hand-adding a
   > Task Link. Cleanup parity is then the only guard against reviving a released task.

2. **Not Today.** Live and blocked rows are never Today. They are not in `today_tasks`
   or `isToday`, do not get 🍅, and do not appear in the `=`/`x` lineups. The Today
   definition in `docs/plan.md` does not change.
   - **Planned today** = Today ∪ live-or-blocked snoozed. Every reader that uses Today
     as an _exclusion_ ("already planned for today") switches to planned-today: the `]s`
     walk scope, recurring and decide rules, the seed, the `]w` lane walk, the
     `bob walks` lane counts, and the dash PENDING/NEXT sections and their counts.
   - Readers that mean _workable now_ keep Today: lineups, the 🍅 mark, the TODAY dash
     section, `plan.max_links`, and the TODAY chip.
   - `bob plan`'s whole-lane NEXT/PENDING meters still include both.
3. **Budget.** Rows never count toward `plan.max_links`. This holds by construction,
   because the budget reads only `## Pomodoros`.
   - The plan reports a separate snoozed meter (`SNOOZED 2 · next 19:00`).
   - A forecast lint `snooze_wake_would_exceed_cap` fires when today's on-clock links
     plus the on-clock live rows exceed the cap. It uses the same on-clock
     classification the link budget uses (`plan_budget::off_clock`). It never changes
     `status`.
   - Wake is never refused for a full budget. After an over-cap wake, the ordinary
     `plan_link_cap_exceeded` applies and the notice says `14/13`.
4. **Snoozing** is a human commitment. Every writer applies the same rule as
   Ctrl+Shift+Enter: Ready or Blocked → Next, and `[fresh::]` is stamped only when the
   task line is rewritten.
   - Snoozing an already-Next or In Progress task writes only the daily note.
   - Live links under today's open entries move as a subtree into the row. The first
     link carries its notes, and any duplicates are removed.
   - An untimed entry left empty is deleted. A running timed entry is never deleted.
   - Closed history lines are never touched.
   - Struck, worked, and recorded-time state stays exactly as
     `decisions:consumption-keeps-notes-and-recorded-time` requires.
   - There is one row per task per day. Re-snoozing retimes the row in place.
     Re-snoozing a woken task replaces its struck row with a new active row.
   - Refused: Event Tasks, Blocked tasks, done or cancelled tasks, non-`#task` lines,
     times not strictly in the future today, and nonexistent DST times. A repeated
     (fold) time means its first occurrence.
5. **Wake** (`bob snooze wake`, idempotent, today's note only):
   1. Lock-free read. Exit at once if there is no section and no unstruck row.
   2. Resolve every row.
   3. Under the vault lock, re-read and re-plan:
      - Prune rows whose target is done, cancelled, or already linked under an open
        Pomodoro.
      - Never move an Event Task row; emit the lint `snoozed_event_task`.
      - Leave blocked rows parked.
      - Leave ignored rows untouched and report each one.
   4. For each due live row (`HHMM <= now`), move the link subtree to the **top** of the
      [destination](#placement), then strike the row with the actual wake time.
   5. Write one preimage-checked atomic write. On lock contention past the bound, return
      `deferred` with exit 0, and the next trigger retries.
   6. Only after the commit, and only when at least one row woke: write the receipt,
      post Darwin, and notify per the channel rule.

   Wake never edits a task line. That means no lane change, no `[fresh::]`, no keep or
   schedule writes, no dependency drops, no `[unblocked::]`, and no successor offers.

6. <a id="placement"></a>**Placement never targets the running entry.** It tries, in
   order:
   1. The first untimed open entry whose type is exactly the task's effective type.
   2. The first untimed open entry that contains the type.
   3. A new `() — TYPE` placeholder at the next-up slot, after the running entry.

   With no effective type, it uses the first untimed open entry. Failing that, it
   creates the same placeholder a bare untyped `@route:id` link would create. The
   destination must be an entry the Today engine counts.

   The explicit **Wake now** gesture (`bob snooze wake TASK`) uses the same placement.
   The "link it now" path, Ctrl+Shift+Enter or capture `@route:id` on a snoozed task,
   keeps ordinary link placement, including running-first, and prunes the row.

7. **History.** Struck rows stay for the day. The heading is deleted only when no rows
   remain. There is no midnight auto-carry: wake reads today's note only, an unwoken row
   becomes history, and the sticky Next lane plus normal review resurface the task.
8. **Cleanup parity** ships in the plugin phases. Every gesture that removes a task's
   live links because the task left today's plan also removes its live row. That
   includes release, complete, cancel, the decay step to Ready, roll and schedule
   setters, a move into a blocked project, and project conversion.
   - Gestures that only rewrite link targets also rewrite the row, the same as any link.
     That covers task move, rename, and `bob task archive`, through whole-note link
     repair.
   - Wake's prune rules make any missed path degrade to a dropped or ignored row, never
     a revived task.
9. **Reconcile** reads rows but never writes them. A live-or-blocked row counts as
   **recovery-only reach**, like the existing recent-activity roots. As a result:
   - A daily-note-resident `[*]` with a live row is kept, not cleared.
   - A snoozed task that recovers from Blocked recovers to Next.
   - It never promotes Ready.
   - It is never consumed as a Blocked link.
   - It never propagates to dependencies.

   Reconcile's struck-reference, duplicate, and childless-entry rules stay scoped to
   `## Pomodoros`.

10. **Single writer, several triggers.** `bob snooze wake` is the only writer that wakes
    rows. It is triggered by:
    - the tick `snooze` step, first in the catalog, on a 5 m cadence;
    - Bob Mac Capture at `next_wake_at`;
    - the bob-ledger-tools minute tick, when a live row is due;
    - Bryan, by running it.

    The automatic sweep (no TASK argument) honors the `tick.writer` pin, falling back to
    `gcal.writer`, exactly as `bob tick` does. A wrong-host sweep is a quiet `skipped`
    with exit 0. `bob snooze wake TASK` runs on any host.

    Concurrent triggers serialize on the vault lock and are idempotent. The latency
    contract is _not-before_: about 1 minute with Mac Capture or Obsidian running, at
    most about 5 minutes from tick alone, and nothing while every host sleeps. Readers
    paint **due** in the meantime.

11. **A trailing section never captures ledger writes.** Every Rust and JS writer that
    appends to `## Pomodoros` must use the section range, which ends at the next `## `,
    never end-of-file. `## Snoozed` is registered in `is_structural_h2_key`, so capture
    never routes into it or treats it as a drawer.
12. **Channel rule** for the nudge, under `snooze.notify: auto | desktop | off` (default
    `auto`):
    - `auto` relies on Mac Capture when a live notifier claim exists. Otherwise bob
      falls back to `desktop_notify`.
    - `desktop` always uses `desktop_notify`.
    - `off` uses nothing.

    The wake JSON reports `notified: "mac_capture" | "desktop" | "none"`.

## Adjustments to the research

These are called out so the reviewer can see every place this plan departs from, or
settles, `snoozed_tasks.md`.

- **A1. Ready rows.** The research contradicts itself. Its C9 says a Ready target means
  "released, drop it", while its wake step 3 says "ensure Next for hand-written Ready
  rows". This plan settles it as a reviewer decision (`uncommitted_rows`). The default
  is the strict C9 reading, refined from "drop" to "ignore": the hand-written row is
  left alone and visibly explained instead of deleted.
- **A2. Wake writes only the daily note.** There is no cross-file write, which means a
  simpler atomic write. This follows from A1 (`ignore`). Under `promote`, wake also
  rewrites the task line, using the capture link writer.
- **A3. Planned-today replaces the research's "whole-lane counts include snoozed"
  note.** `bob walks` lane counts already exclude Today (`walks/mod.rs`
  `counts_in_lane`), so they now exclude snoozed tasks too. `bob plan`'s whole lanes
  include both, as the research intended.
- **A4. One alarm-clock glyph everywhere.** It replaces the research's ◷/⏰ split. ◷
  already means "scheduled" on project cards, and a single icon reads as one concept.
- **A5. The woken pill shows only on links under open entries.** Closed entries carry 🍅
  history already. The struck row keeps provenance.
- **A6. Reconcile reach.** Rule 9 fixes two traps the research did not cover. Without
  it, reconcile would clear a daily-note-resident snoozed `[*]`, and a snoozed task
  recovering from Blocked would recover to Ready. Both would then be ignored at wake.
- **A7. Writer pin and the Obsidian trigger.** The research's optional ledger-tools
  trigger is adopted. Unlike tick, it also covers Linux hosts, and the existing writer
  pin keeps the sweep single-host when Bryan sets one.
- **A8. Meeting hold** (`hold_in_meetings`) extends the research's "silent during a
  meeting" to "held until it ends". It also avoids leaking task titles during a screen
  share.
- **A9. Wake-now placement.** Wake now uses the never-running placement.
  Ctrl+Shift+Enter remains the "start it now" path. Each gesture has one predictable
  meaning.
- **A10. The dash.md query edit.** The research named the dash exclusion but not where
  it lives. The PENDING/NEXT queries are in the vault's `dash.md`, so plugin-plumbing
  edits them, with optional chaining so older plugins degrade safely.

## Contracts

### Row grammar

- The section is the first `## Snoozed` H2 in the note. It ends at the next `## `
  heading or at end of file. It is fence-aware and frontmatter-aware, like
  `pomodoros_section_range`. Later duplicate sections are linted and ignored.
- An active row is `- **HHMM** <block link>`, at column 0, with exactly one plain or
  embedded block link and nothing else.
  - On read, `HH:MM` and `H:MM` inside the bold are accepted. Writers normalize to
    `**HHMM**`.
- A struck row is `- ~~**HHMM** <block link>~~ · woke HH:MM`.
- Child bullets are the link's notes and travel as a subtree.
- Writers keep rows sorted by target time, with ties in link order.
- Malformed rows, extra links, rows inside fences, and indented rows are reported and
  never acted on.
- Several live rows for the same task: the earliest is live and the rest are
  `duplicate`, ignored.
- A Snoozed row never creates a task mention: it is note prose, vector MN5 in
  `docs/task-relations.md`.

### `bob snooze` CLI (noun: today's daily-note Snoozed queue; Daily workflow help section)

| Command                          | Behavior                                                                                                                                                                                            |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob snooze` = `bob snooze list` | Read-only, flags only. Lists today's queue in time order: asleep (countdown), due, blocked, woken (`woke 15:32 → WRITE`), ignored (with reason). The header shows `2 asleep · 1 woke · next 19:00`. |
| `bob snooze add TASK WHEN`       | TASK is `note:block-id` or `[[note#^id]]`. Follows rule 4. Reports the moved link, the lane change, the time, and the forecast.                                                                     |
| `bob snooze drop TASK`           | Removes the live row without waking it. The lane is unchanged (sticky).                                                                                                                             |
| `bob snooze wake [TASK...]`      | No TASK: sweep the due rows (rules 5, 6, 10). With TASK: wake now, regardless of time.                                                                                                              |

- Shared options:
  - `-b/--bob-dir`
  - `-d/--dry-run`, which plans and prints with no lock and no write
  - `-f/--format human|json`
  - `-q/--quiet`
  - `-h`
- Options are sorted, and every option gets a short alias, per `cli_rules.md`.
- `BOB_NOW` and `BOB_DAY_FILE` drive the clock and the note in tests. Use one injected
  clock.
- Exit codes:
  - 0: success, `nothing_due`, `deferred`, or `skipped`.
  - 1: failure.
  - 2: usage error or refusal.
- Human output is colored: an indigo time, amber for due, dim for woken.

### JSON (additive; `schema_version: 1` for new documents)

- **Wake report.** Fields:
  - `schema_version`, `date`, `now`
  - `status`: `woke | nothing_due | deferred | skipped`
  - `woken[]`:
    `{route, block_id, title, target, woke_at, destination: {line, name, created}, over_cap}`
  - `dropped[]` and `ignored[]`: `{route, block_id, target, reason}`, where
    `reason ∈ done | cancelled | already_linked | missing | ambiguous | ready | event_task | duplicate | malformed | not_task`
  - `pending[]`: `{…, state: asleep | due | blocked}`
  - `next_wake_at` (RFC 3339 or null)
  - `plan: {count, cap, over}`
  - `notified`
  - `wake_id`
- **`bob snooze list -f json`** has the same row objects, with `state`, plus
  `next_wake_at`.
- **Receipt.** `$XDG_STATE_HOME/bob-cli/snooze/last-wake.json`, falling back to
  `~/.local/state`. Written atomically with 0600 temp and rename, mirroring
  `highlights_ref/scan_receipt.rs`. Holds:
  - `schema_version`, `wake_id`, `date`, `woken_at`, `host`, `daily_file`
  - `tasks[]`: `{title, route, block_id, target, woke_at, destination, line}`
  - `plan`

  After writing it, post Darwin `org.bobs.bob.snooze.woken` through `darwin_notify.rs`.

- **Notifier claim** (written by Mac Capture, read by bob). Path:
  `$XDG_STATE_HOME/bob-cli/snooze/notifier.json`. Holds
  `{schema_version, pid, app, version, started_at}`. It is live when the pid is alive.
- **`bob plan -f json`** gains
  `snoozed: {count, due, blocked, on_clock, next_wake_at, forecast: {count, cap, over}}`.
- **`bob capture-pomodoros --tasks`** gains:
  - `snoozed: {rows: [{title, route, block_id, target, wake_at, state, notes}], next_wake_at}`,
    holding live and blocked rows only;
  - an optional `woke_at` / `snoozed_target` on agenda task items woken today.

  The schema version is unchanged (the Mac rejects any `schema_version != 1`).

### Conformance vectors

`tests/fixtures/snooze/vectors.json` is owned by bob-cli and documented in
`docs/snooze.md`. Its groups:

- `rows`: parse and states
- `when`: the time grammar against an injected now, including DST gap and fold cases
- `snooze`: an add plan from note plus target state to the resulting note bytes
- `wake`: note plus target states plus now, to note bytes plus report
- `live`: which tasks are planned-today
- `cleanup`: removing a task's row

Plugins copy the file byte-for-byte into their `scripts/fixtures/snooze/` and never
hand-edit it. A change to the vectors therefore lands in bob-cli first.

## Phase wake-engine: bob snooze core, wake writer, and tick step

bob-cli only. First read `cli_rules.md` and `docs.md` with `/sase_memory_read`.

- **Module `src/native/snooze/`.** Pure modules hold the logic; `cli.rs` and the write
  shell stay thin.
  - `parse.rs`: the section range and rows, per the grammar.
  - `when.rs`: the time grammar, the injected clock, and the DST rules, using Bob's zone
    from `src/native/env.rs`.
  - `state.rs`: resolution through `VaultLinkResolver`, and the live, blocked, and
    ignored classification.
  - `plan.rs`: add, drop, and wake plans as content-to-content functions.
  - `receipt.rs`.
  - `cli.rs`.
- **Placement.** Add a never-running variant to `pomodoro_themes/routing.rs` `place`. An
  input flag excludes running entries in all three steps and creates after the running
  entry. Reuse `capture_task_toggle` helpers: `move_subtree_to_entry` adapted to insert
  at the top of the entry, `insert_named_placeholder_after_last_open`,
  `reindent_subtree`, and the type suggestion used by `placement_for_task`.
- **Writes.**
  - `add` and `drop` reuse the capture existing-task link machinery for the task-line
    rewrite (`capture_task_toggle/task_update.rs` `plan_task_next` / `plan_task_link`,
    plus the same freshness stamp capture applies on a picked existing task). They stage
    both files through `capture::write_staged_files`.
  - `wake` follows the `gcal/sync.rs` pattern: a lock-free plan, then
    `ob::acquire_lock_waiting` with a 10 s bound, a re-read and re-plan, and a
    preimage-checked `write_staged_files` with a single retry.
- **CLI registration.** Register in `src/runner.rs` (`Section::DailyWorkflow`, a group
  whose bare form is `list`). Add catalog rows in `src/native/catalog/table.rs` for
  `snooze list` (read) and `snooze wake`. Then run `just docs-gen` for generated CLI
  references.
- **Tick.** In `src/native/tick/catalog.rs`, add `snooze` **first**:
  - `intake = false`, `Cadence::Interval(5)`, a 2 min ceiling;
  - argv `["snooze","wake","-q","-f","json"]`;
  - the readiness gate (`gcal_gate`);
  - an adapter whose summary is `woke 1 · next 19:00` or `nothing due · next 19:00`.

  `deferred` and `skipped` are non-failures. Update the `docs/tick.md` table and config
  example, plus the tick fixtures under `tests/fixtures/tick/`.

- **Notify.**
  - Write the receipt and post Darwin as described in Contracts.
  - Add the `snooze.notify` config key (`config/`, `docs/reference/configuration.md`).
  - The `desktop_notify::send_notification` fallback uses the title `bob snooze` and the
    body `Ready now: <title> (woke 15:32 · WRITE)`; a batch gets
    `3 snoozed tasks are ready: …`.
- **Structural H2.** Add `Snoozed` to `is_structural_h2_key`
  (`src/native/capture_drawers.rs`).
- **Writer audit (rule 11).** Add a fixture daily note with a trailing `## Snoozed`
  holding rows and notes. Prove byte-for-byte that these all still write only inside
  `## Pomodoros` and never touch the Snoozed rows:
  - capture link, start, close, `x=` record, and `=` lineup;
  - gcal Events insertion and MEET filing;
  - reconcile;
  - collect_done link repair, which must rewrite links inside rows on a move, and
    nothing else.

  Fix any end-of-file append you find.

- **Docs.**
  - New root contract `docs/snooze.md`: rules, grammar, CLI, JSON, receipt, latency
    contract, and vectors. Lead with "later-today Task Links", and disambiguate it from
    the Reading Board's Snoozed lane and project `snooze:`.
  - Update `docs/reference/vault-format.md`, `docs/reference/json-outputs.md`, and the
    `mkdocs.yml` nav as docs-gen requires.
  - Run `just docs-check`.
- **Tests.** Rust tests run every vector group. They also cover the acceptance cases in
  [End-to-end acceptance](#end-to-end-acceptance) items 1–3 and 5–7, in their bob-only
  form.
- **Commits.** Commit as `feat(snooze): …`. Never edit the `Cargo.toml` version or
  `CHANGELOG.md`.
- **Gate.** `just check` and `just docs-check`.

## Phase planned-today: Snoozed rows in reconcile, walks, plan, and the agenda JSON

bob-cli only. Builds on the `snooze` module's `live_targets(note) -> {planned, blocked}`
helper. Add that helper in wake-engine if it is missing.

- **Reconcile (rule 9).** Changes are in `task_status_hooks/sync.rs` and nearby.
  - Feed today's live-or-blocked row targets into the same recovery-only path as
    `recent_activity_roots`, or into a sibling set with identical effect on
    `task_transition`. The result: the daily-note `KeptNext` keep and Blocked recovery
    to Next.
  - Never `desired` (no promotion), never consumption, never dependency propagation.
  - Add hooks vectors under `tests/fixtures/task_status_hooks/`:
    - a daily-note `[*]` with a live row is kept;
    - a snoozed Blocked task whose dependency closes recovers to `[*]`;
    - a Ready target is not promoted;
    - rows are byte-identical after reconcile.
  - Update the `--long-about` text.
- **Freshness and walks.**
  - Add `is_snoozed_today` to `freshness::scan` `RowCtx`, computed beside `read_today`.
  - Use planned-today (`is_today || is_snoozed_today`) at every exclusion site:
    - `state.rs` `decide_for` and the walk scope;
    - recurring tier;
    - `placement.rs`;
    - `seed.rs`;
    - `walks/mod.rs` `counts_in_lane`.
  - Add vectors to `docs/freshness.md` and `docs/walks.md` (snoozed excluded from the
    walk and the lane counts; an ignored row does not exclude) and refresh
    `tests/fixtures/walks/`.
- **`bob plan`.**
  - Add the `snoozed` block to `PlanReport` (`plan_budget/mod.rs`).
  - The human line is `SNOOZED 2 · next 19:00 · TODAY 11/13 when all wake` (amber
    `1 due` when due).
  - Add the forecast lint and update `docs/plan.md`: Today is unchanged; planned-today
    is defined for exclusions; add the snoozed meter, the lint, and the dash exclusion
    text.
- **Agenda JSON.** In `capture_pomodoros_agenda.rs`, add the `snoozed` block and the
  `woke_at` / `snoozed_target` item fields, plus goldens under
  `tests/fixtures/capture_pomodoros/`. Update `docs/capture.md`'s agenda section and
  `docs/reference/json-outputs.md`.
- **Gate.** `just check` and `just docs-check`. Commit as `feat(snooze): …`.

## Phase capture-z: Capture z:<when> snooze token

bob-cli only. Read `cli_rules.md` and `docs.md` first.

- **Grammar.** Add the `z:<when>` terminal marker beside `parse_schedule_token`
  (`capture_language/markers.rs`).
  - `when` is the [time grammar](#contracts), shared with `snooze/when.rs`.
  - It works on new-task items that would link: the new task is created Next and its
    link goes to `## Snoozed`, not a Pomodoro.
  - It works on existing-task `@route:id` items, through one documented extra-token
    exception to the existing-task grammar, giving "snooze this task".
  - It writes through the wake-engine planner, so there is a single implementation.
- **Interplay.**
  - `@route:id` on a snoozed task links it now with normal placement and prunes the row.
  - `@route:id!` on a snoozed task drops the row; the lane stays.
  - Re-snoozing retimes the row.
- **Teaching refusals.** These cover:
  - past times ("link it now, or use `s:1` for tomorrow");
  - times past midnight;
  - `z:` combined with `s:<N>` or a `=` start;
  - Event Task targets;
  - Blocked targets;
  - items without a link target.
- **Completions.** After `z:`, completion offers today's remaining event end times
  (`15:30 · after Yogi interview`, from the Events section), then `+30m`, `+1h`, and
  `+2h`, then times used today. Each resolves to a literal HHMM.
- **Spans.** `capture-parse` emits a new `snooze` span kind. Add it to the span
  vocabulary documented for clients.
- **Docs.** Update `docs/capture.md` (marker table, existing-task section, completions)
  and `docs/reference/capture-syntax.md`. Run `just docs-gen` and `just docs-check`.
- **Gate.** `just check`. Commit as `feat(capture): …`.

## Phase plugin-plumbing: Plugin snoozed index, exclusions, cleanup parity, and Obsidian waker

bob-plugins. Open it with `/sase_repo` and read its `AGENTS.md`. Fragments stay at or
under 1000 lines, and new tests are added to the `test:unit` list in `package.json`.

- **Vectors and mirror.** Copy `tests/fixtures/snooze/vectors.json` byte-for-byte into
  each consuming plugin's `scripts/fixtures/snooze/`. In bob-ledger-tools, add a pure
  `snooze` core: section finder, row parse, time grammar, and the live and planned
  predicates. Tests run the `rows`, `when`, `live`, and `cleanup` groups verbatim.
- **Cache and api.**
  - Add a `snoozedCache` beside `todayCache` (`LT/290-plugin-today-and-location.js`
    `rebuildTodayCache`). It holds rows, states, and `snoozedEnteredAt` for animations,
    and refreshes on the same events and the minute tick.
  - Export `api.isSnoozedToday(task)`, `api.isPlannedToday(task)`, and
    `api.snoozed.rows()`. `isToday` stays pure.
- **Exclusions.** Use planned-today in:
  - the freshness rows (`LT/230-plugin-freshness-api.js`,
    `LT/100-freshness-evaluate.js`);
  - the lane walk (`LT/071-lane-walk-model.js` and nav `542`/`544`);
  - `dashboardLaneBudgetFromTasks` (`LT/070`).

  Then edit the vault's `dash.md` PENDING/NEXT/READY query lines and its DataviewJS
  section counts to also skip `...api?.isSnoozedToday?.(task) === true`. This is a
  deliberate vault edit; keep it minimal and let vault-sync commit it.

  Add the walk and footer vectors mirrored from planned-today.

- **Cleanup parity (rule 8).**
  - Teach `planDeferredPomodoroLinkCleanup` (`NH/160`) to also remove the targets' live
    rows in `## Snoozed`, deleting an emptied heading. This covers release, cancel, roll
    and schedule setters, move, and conversion.
  - Do the same for BIP's `planAllOpenPomodoroLinkCleanup` (`BIP/060`, unlink) and the
    link path (`BIP/120`). Linking a snoozed task prunes its row, and
    `pomodoroTaskLinkPresence` reports a `snoozed` presence.
  - Cover the decay writers (`NH/548`, the step to Ready).
  - In TSC Ctrl+Enter (`TSC/070`): completing from a Snoozed row removes the row instead
    of striking it, and completing a snoozed task elsewhere removes its row.
  - Alt+N on a Snoozed row releases the task and removes the row.
  - Ctrl+Shift+Enter on a Snoozed row links now.
  - Add one test per gesture.
- **Obsidian waker.** In the ledger-tools minute tick (`LT/170` around 556–610), when
  `snoozedCache` has a live due row, spawn `bob snooze wake -q -f json` through the
  `bobSpawnJson` precedent (`LT/291`).
  - Spawn at most once per minute, on desktop only.
  - An unsupported old bob disables the waker for the session.
  - On `woken[]` with `notified: "none"`, show one 8 s `Notice`:
    `⏰ Ready now: <title> · woke 15:32 · WRITE`, or a batch line.
- **Deploy.** `npm run validate`, `npm test`, and `npm run docs:check` (with
  `BOB_CLI_DIR` pointing at bob-cli). Then `bob plugins sync`. Commit in bob-plugins as
  `feat(snooze): …`.

## Phase plugin-marks: Clock marks, woken-link pills, Snoozed-row paint, and plan chip

bob-plugins (bob-ledger-tools), plus a new display contract in bob-cli.

- **Contract.** Author `docs/snooze-marks.md` in bob-cli, modelled on
  `docs/today-marks.md` and `docs/event-link-times.md`. It owns principles, placement,
  anatomy, tooltips, interaction, liveness, toggles, and vectors (SM-\*). The JS tests
  run those vectors verbatim. The bob-cli commit lands first.
- **Definition mark.**
  - A sibling of the Today mark in `LT/143`, `275`, `276`, and `277`, in the same slot
    with the same host rules: never inside daily notes, display-only, and the exact
    source mapping. The clock wins over nothing; 🍅 wins over the clock.
  - `span.bob-snooze-mark` draws a CSS-masked Lucide `alarm-clock` from
    `--bob-snooze-mark-glyph`, tinted `currentColor`, plus the time via `::after`
    `attr(data-bob-snooze-time)` with tabular numbers.
  - States: asleep is indigo (`color-mix` of `--color-blue` and `--color-purple`); due
    is amber with one gentle pulse; blocked is muted.
  - On wake, the clock leaves and 🍅 enters with the existing 600 ms pop.
  - Mod+click jumps to the row.
  - Toggles: the Today-marks toggle and a `toggle-snooze-marks` command.
- **Woken-link pill.** A sibling of Event link times (`LT/291d`, `291e`, `291f`). It
  scans the same note's `## Snoozed` struck rows with
  `gcalEventsFindSection(lines, "Snoozed")` and paints links under **open** entries
  only, at `side: -3` (free, because Event Tasks never snooze).
  - Fresh, meaning 60 minutes after `woke`: a warm amber-gold pill `⏰ woke 15:32`.
  - After that, a quiet outline pill.
  - The tooltip explains early or late wakes.
  - Mod+click jumps to the struck row.
  - Add the class to the `ledgerMarkClass` allowlist (`LT/269`).
- **Snoozed-row paint.**
  - Asleep: an indigo time pill and a faint live countdown.
  - Due: amber `due · waking`.
  - Struck: dimmed.
  - Ignored: a muted reason chip, such as `Ready — won't wake`.
  - The `## Snoozed` heading gets a trailing chip: `2 asleep · next 19:00`.
  - Live Preview and Reading view both get the paint, refreshed on the minute tick.
- **Plan chip.** Beside `TODAY n/cap` in the `bob-plan` block (`LT/340`
  `planBlockModel`): `⏰ 2 · 19:00`, indigo, or amber when one is due. It is never
  cap-colored. Click jumps to `## Snoozed`. The tooltip reads
  `2 snoozed · next wakes 19:00 · not counted in TODAY · 11/13 when all wake`.
- **Design bar.** Before landing, take a screenshot of each state in light and dark
  themes using `npm run smoke:obsidian` or a live vault, and attach it to the phase
  bead. Check:
  - no line-height shift;
  - the reduced-motion and forced-colors rules;
  - tooltips never contain `::`.
- **Gate.** The same gates as plugin-plumbing, plus `just docs-check` in bob-cli. Then
  `bob plugins sync`.

## Phase snooze-card: Alt+Z Snooze Card and Task Card z

bob-plugins (bob-navigation-hotkeys hosts the command and card; it reuses BIP's
task-line update and ledger-tools' `api.freshness.stampLine`).

- **Command.** Add `snooze-task` with default hotkey **Alt+Z**. Collision-check
  `.obsidian/hotkeys.json`, the vimrc, and the plugins at install, and never overwrite
  an existing binding. Add Task Card `z` in `resolveTaskCardKey` (`NH/314`) and the row
  order and shortcuts.
- **Contexts.** These are as in [Gestures](#gestures):
  - a live ledger link;
  - a woken link, which re-snoozes it;
  - a closed history link, which snoozes the task and leaves history alone;
  - a task definition;
  - a Snoozed row;
  - Event, Blocked, and done tasks, which get a teaching `Notice`.
- **Card.** Model it on the filtered picker (`NH/200`) and close keys
  (`isTaskCardCloseKeydown`, `bobCardWireCloseKeys`). It holds:
  - the title line `⏰ Snooze “<task title>”`;
  - one input;
  - ranked suggestions, with event ends first, read from the Events rows ledger-tools
    already parses;
  - the consequence footer with the forecast.

  In snoozed mode, the header reads `Snoozed until 15:30 · in 2h`, and the extra rows
  **Wake now** (runs `bob snooze wake TASK`, the one wake writer) and **Unsnooze** are
  added. Use Ctrl+N/P and ↑/↓ to move, Tab to complete, and Enter to confirm.

- **Planner.** A pure JS planner, tested on the `snooze` and `cleanup` vector groups:
  - It writes the daily note through a single editor transaction when that note is open,
    so one Ctrl+Z undoes it. Otherwise it uses a preimage-checked `vault.process`.
  - It writes the task line through the BIP target-plan path (`planTargetTaskUpdate`
    with `forceNext` and the injected `stampLine`).
  - Snoozing never runs a subprocess.
- **Docs.** Add the plugin reference entries for Alt+Z and Task Card `z`, following the
  plugin reference route in `docs.md`. Update `docs/snooze.md` gestures.
- **Gate.** As in plugin-plumbing, then `bob plugins sync`.

## Phase mac-agenda: Mac Later-today band, woken accents, and exact-minute waker

bob-mac-capture. Open it with `/sase_repo` and read its `AGENTS.md`:

- compile every Swift change on the Mac scratch copy;
- run `Scripts/macos-ci-check` before landing;
- watch the exact-SHA CI run to green with `/sase_monitor`.

Bob stays the only fact source, per `decisions:mac-capture-is-a-thin-client`.

- **Models.** In `CaptureCore/CaptureAgendaModels.swift`, decode `snoozed` and the item
  `woke_at` / `snoozed_target` with `decodeIfPresent`. An older bob shows nothing.
- **Band.**
  - A dim **Later today** band, the farthest content, after the strip
    (`CaptureAgendaFitPlanner.swift` render order). Rows read
    `⏰ 19:00  Fix the sink · in 3h 28m`; due rows read amber `due · waking`; blocked
    rows are muted.
  - Its fold steps come **first** in the ladder. First the band folds to one summary row
    (`⏰ 2 later · next 19:00`); that summary never disappears.
  - Update `CaptureAgendaRowKind`, `finish()`, `CaptureAgendaSectionBlock`, and the view
    switches.
- **Accents.**
  - Agenda task rows with `woke_at` within 60 minutes show an amber trailing
    `⏰ woke 15:32`.
  - The budget row appends `· ⏰ 2`.
  - Map the `snooze` span kind in the capture editor's span styling.
- **Waker.** Add an always-on `SnoozeWakeScheduler`, which is not the visible-only
  boundary refresh.
  - Arm it for `snoozed.next_wake_at` plus 5 s from the latest snapshot.
  - Re-arm on snapshot change, system wake, and clock or day change.
  - Run `bob snooze wake -q -f json` on its own non-cancelling `BobProcessClient` lane.
  - An unsupported old bob (the `isUnsupportedWalksFailure` pattern) disables it.
  - Fold `next_wake_at` into the visible boundary refresh as well.
- **Docs.** Update `docs/idle-agenda.md`, `docs/protocol.md` (optional fields and
  commands), and `docs/develop/architecture.md` (lanes) in this repo. Run
  `just docs-check`.
- **Gate.** `just portable-check`, a Mac typecheck of the app targets, then CI green on
  the exact SHA.

## Phase mac-wake-notice: Presence-aware wake notice on the Mac

bob-mac-capture, under the same rules as mac-agenda. The notice design is in
[The wake notice](#the-wake-notice-the-part-that-must-be-seen).

- **Claim.** Write `notifier.json` atomically at launch and remove it at quit, so bob's
  `auto` channel defers to the app.
- **Receipt.** Add `SnoozeWakeReceipt` (CaptureCore: path resolution mirroring
  `RefsScanReceipt.swift`, the 4 MB cap, `schema_version` 1). Add `SnoozeWakeObserver`
  (BobMacCapture: `notify_register_dispatch`, mirroring `RefsScanReceiptObserver.swift`,
  wired like `startScanReceiptObserver`). Dedupe by `wake_id`, persisted in
  UserDefaults. Ignore receipts that are not from today or are older than 4 h.
- **Presence policy.** A pure module in CaptureCore with exhaustive unit tests.
  - Inputs: lock and unlock (`com.apple.screenIsLocked` / `screenIsUnlocked` distributed
    notifications), `screensDidSleep` / `screensDidWake`, session active and resign,
    idle seconds (`CGEventSource.secondsSinceLastEventType`, polled every 2 s only while
    a batch is held), and the running meeting from the latest agenda Events
    (`startsAt`/`endsAt` recomputed against now).
  - Outputs: show now, hold, or drop. A held batch is dropped at day change.
  - Delivery runs 1.5 s after return. Before it, re-verify with `bob snooze -f json` and
    keep only tasks still woken and open.
- **Surface.**

  > [!decision] wake_notice = hud Extract a generic non-activating glass HUD from
  > `RefsCopyHUD.swift`: mouse events on, first-mouse, hover pause, screen-anchored
  > top-center, `sharingType = .none`, and the Reduce Transparency and Reduce Motion
  > fallbacks. Build the snooze HUD from it.

  > [!decision] wake_notice = banner Add a snooze category with Open/+15m actions to
  > `NotificationService` (`interruptionLevel = .active`, `threadIdentifier = "snooze"`,
  > stable id `snooze-woken-<wake_id>`).

  **+15m** runs `bob snooze add <task> +15m`. The notice morphs to
  `Snoozed until 15:47`, then closes. **Open** uses `ObsidianOpenURL.taskGotoURL`. The
  chime is a soft system sound and is never played while a notice is held.

- **Beacon.** In `StatusItemController` / `StatusItemGlyph`, add a small amber clock
  with the count, plus clearing rules. Add the silent `.passive` Notification Center
  record (only when authorized; never prompt), withdrawn with
  `removeDeliveredNotifications` when the beacon clears.
- **Settings.** Add a "Snoozed tasks" section: Announce wakes, Chime, Hold while away,
  and Hold during meetings (default per `hold_in_meetings`).
- **Docs.** Update `docs/settings.md`, `docs/diagnostics.md` (beacon, receipts,
  signposts), and `docs/protocol.md` in this repo.
- **Gate.** As in mac-agenda. Run one manual Mac check, recorded on the bead: lock the
  screen, let a row wake, unlock, and see the notice arrive.

## Phase close-out: Glossary, guide, decision record, and end-to-end verification

bob-cli (memory and docs), plus a live run.

- **Glossary strand** (decision `glossary_snoozed_task`). Use `/sase_memory_write`, then
  run `sase memory init` and `just docs-gen` to regenerate `docs/reference/glossary.md`.
  File: `sase/memory/glossary/snoozed-task.md`. Polish the text against what actually
  shipped, but keep its shape and claims:

  ```markdown
  ---
  keyword: Snoozed Task
  title: Snoozed Task
  summary:
    An open, committed, non-Event task planned for later today; its Task Link waits in
    the daily note's Snoozed section until a clock time, then wakes into the ledger.
  aliases:
    - snooze
    - snoozed row
    - later-today task
  ---

  # Snoozed Task

  _Requested · project_

  aka snooze, snoozed row, later-today task

  An open Next or In Progress task planned for **today, but not before a clock time**.
  Its [[task-link]] waits as one row in the `## Snoozed` section after today's ledger —
  `- **1530** [[note#^id]]`, the link's notes beneath — instead of under a [[pomodoro]].
  While it waits the task is **not Today**: no 🍅, absent from the `=`/`x` lineups, the
  `]s` and `]w` walks, and the dash PENDING/NEXT sections, and not counted in
  `plan.max_links`. Its definition wears an alarm clock and the time (`⏰ 15:30`, amber
  once due) where a Today task wears 🍅. Snoozing — Alt+Z on a Task Link or task, `z:`
  in capture, or `bob snooze add` — raises Ready to Next and stamps freshness, exactly
  like linking.

  At or after the time, `bob snooze wake` — run by `bob tick`, by Bob Mac Capture at the
  exact minute, and by Obsidian — moves the link and its notes to the top of an untimed
  open Pomodoro of the task's type, never the running one, and strikes the row with the
  actual time (`- ~~**1530** …~~ · woke 15:32`), which paints `⏰ woke 15:32` on the
  woken link. The task is now Today and counts normally, and one notice announces it —
  held while Bryan is away — then goes away on its own. A passed time still awaiting a
  safe write is **due**, not ready. Waking never starts work, stamps freshness, drops
  prerequisites, or revives a task that left Next or In Progress.

  Same-day only: an unwoken row becomes history at midnight and the sticky Next lane
  returns the task to review; multi-day deferral is `[scheduled::]`. Never an
  [[event-task]], whose span already says when it applies; not the Reading Board's
  Snoozed lane or a project-frontmatter `snooze:` hold. See `docs/snooze.md`.
  ```

  If `uncommitted_rows = promote` was chosen, replace "revives a task that left Next or
  In Progress" with "revives a released task".

- **Decision record** (decision `decision_record`; only when accepted). Write
  `sase/memory/decisions/snoozed-tasks-wake-through-bob.md` in the house format.
  - Applies to, Claim: rules 1–3, 5, 6, 8, 10, and 12, condensed.
  - Why, with rejected alternatives: the task-line time field (cancelled `21cdc33b`), an
    inline link marker, a timed `SNOOZED` placeholder, a JSON sidecar, counting toward
    the budget, and auto-carry.
  - Cost, Reopens when, and Evidence: this plan, the two research reports, and the
    landing commits.
  - Mark nothing superseded: Today itself is unchanged.
- **Docs.**
  - Add the guide `docs/guides/snooze-a-task-until-later.md`, covering the goal, Alt+Z,
    capture `z:`, what you will see, and what to do when a wake seems late.
  - Add one "Later today" paragraph to `docs/concepts/today-and-the-pomodoro-ledger.md`.
  - Add the nav entries, then run `just docs-check`.
- **End-to-end.** Run every case in [End-to-end acceptance](#end-to-end-acceptance)
  live, on the real vault and the Mac, after the release train ships the bob changes and
  `bob plugins sync`. Record evidence (commands, JSON excerpts, screenshots) on the
  bead.
  - File a task bead with `/sase_new_task` for each discovered defect.
  - File the deferred list below as feature beads only if Bryan asks.

## End-to-end acceptance

1. **Asleep.** A task snoozed to 15:30 at 13:00:
   - is absent from `=`/`x`, TODAY, `]s`, `]w`, the `bob walks` lane counts, and the
     dash PENDING/NEXT sections;
   - is not in `links.count`;
   - keeps its lane and notes;
   - wears `⏰ 15:30`, not 🍅;
   - shows in the plan chip, `bob plan`, the Mac Later-today band, and `bob snooze`.
2. **Wake.** At 15:30 or later, the first eligible trigger:
   - moves the link and notes exactly once to the top of an untimed matching entry;
   - strikes the row with the actual time;
   - flips the definition to 🍅;
   - paints `⏰ woke 15:3x` on the link.

   Exactly one notice fires and auto-dismisses. A second run is a no-op. A 15:40 run
   catches a 15:30 row.

3. **Running entry.** With only a running matching entry, wake creates a new placeholder
   after it. A later plain `x` close cannot record the woken task as worked.
4. **Leaving the plan.** Complete, cancel, release, unlink, decay to Ready, roll,
   schedule, or move before the wake removes or rewrites the row, and the task is never
   revived. A dependency added while asleep keeps the row parked, and the task wakes
   after recovery. An over-cap wake succeeds and shows `14/13`.
5. **Event Tasks.** They are refused on every route. gcal sync, MEET filing, Gmail
   holds, successor offers, close numbering, and task mentions are byte-for-byte
   unaffected.
6. **Rows that do not wake.** Yesterday's rows never wake today. Malformed rows, fenced
   rows, extra links, and ignored rows never execute. Deleting every cache and app
   preference loses nothing but paint. Old binaries and plugins ignore the H2.
7. **Clock edge cases.** DST gaps are refused and DST folds are disambiguated. A wake at
   midnight commits under the old day or defers.
8. **Notice delivery.**
   - Locked or idle Mac: the notice is held and delivered on return, after
     re-verification.
   - With `hold_in_meetings`: held until the meeting ends.
   - Mac Capture not running: osascript fallback.
   - Linux host with Obsidian: a Notice from its own spawn.

## Deferred (not in this epic)

These come after about two weeks of real use:

- named times (`z:home` from `snooze.times`);
- event-anchored wakes and the `starts after` interplay;
- a "missed yesterday" lint;
- a Time Sensitive opt-in;
- a phone channel for the lid-shut evening;
- a dash "Later today" section.
