---
tier: epic
title: Untracked calendar events, calendar-aware MEET starts, and a Calendar band
goal: 'A Google Calendar event tagged #untracked becomes an #untracked Event Task
  that is still linked under a MEET Pomodoro but never counted as tracked time; =
  starts a MEET on the real span of its next back-to-back events; non-MEET sessions
  warn before running into an event; and Bob Mac Capture''s idle view shows a glanceable,
  beautiful Calendar band.

  '
decisions:
  untracked_counts:
    ask: Should an untracked session also drop out of Pomodoro counts, not just minutes?
    choices:
      exclude: Untracked sessions vanish from every aggregate (minutes, counts, homes,
        Day strip)
      minutes_only: Untracked sessions still count as one Pomodoro each; only their
        minutes drop
    default: exclude
    why: '#untracked means not tracked; a lunch counted as a Pomodoro would still
      inflate the day'
    answer: exclude
  obsidian_meet_start:
    ask: Should Obsidian's se<X> + Tab mirror capture's new MEET start behavior?
    choices:
      full: se + Tab on a MEET uses the event-run span and [t::] rule, plus the run-past-event
        Notice
      tracked_only: se<X> + Tab keeps its timing but omits [t::] when the MEET's event
        run is untracked
      none: Obsidian starts stay as today; an untracked MEET started there keeps [t::]
    default: full
    why: one gesture should write the same ledger from both front doors, pinned by
      shared vectors
    answer: full
  warn_untracked:
    ask: Should non-MEET starts and extends also warn when they run past an untracked
      event?
    choices:
      all: Every open Event Task warns, tracked or untracked (a lunch is still a commitment)
      tracked_only: Untracked events never warn (better if they are mostly focus blockers)
    default: all
    why: untracked means off the clock, not optional; the warning never blocks
    answer: all
  memory_event_task_glossary:
    ask: 'Should glossary:event-task say an #untracked Event Task keeps its session
      off the clock?'
    memory:
    - glossary:event-task
    default: true
    corrects: 'Its `[root:: NOTE]` attributes meeting time to an area or project,
      and a Pomodoro Task Link under the daily note''s ledger records the session.'
    answer: true
  memory_untracked_record:
    ask: Should a new record say untracked sessions omit [t::], superseding part of
      synced-meetings-link-by-phase?
    memory:
    - decisions:synced-meetings-link-by-phase
    - decisions:untracked-sessions-omit-t-field
    default: true
    corrects: Back-to-back or overlapping past meetings share one closed MEET whose
      span is the cluster (next start ≤ cluster end).
    answer: true
phases:
- id: untracked-ledger
  title: 'bob-cli: untracked ledger rows and Event Task event helpers'
  size: medium
  depends_on: []
  description: 'untracked-ledger: make a timed row without [t::] untracked in every
    reader and writer, and add the shared Event Task helpers.'
- id: gcal-untracked-tag
  title: 'bob-cli: #untracked in Google Calendar and bob gcal sync'
  size: medium
  depends_on:
  - untracked-ledger
  description: 'gcal-untracked-tag: parse the Google token, render the row tag, split
    past MEET clusters by trackedness, report it in list/sync.'
- id: meet-event-run
  title: 'bob-cli: capture = starts a MEET on its next event run'
  size: medium
  depends_on:
  - untracked-ledger
  description: 'meet-event-run: default MEET starts to the next contiguous event run,
    apply the [t::] rule to every MEET start, add JSON and vectors.'
- id: event-start-warnings
  title: 'bob-cli: warn when non-MEET sessions run past an Event Task'
  size: medium
  depends_on:
  - meet-event-run
  description: 'event-start-warnings: structured run-past-event warnings with fit
    drafts for starts, adjusts, and shifts of non-MEET sessions.'
- id: agenda-events
  title: 'bob-cli: events block in capture-pomodoros --tasks'
  size: medium
  depends_on:
  - untracked-ledger
  description: 'agenda-events: add the additive events block, item event chips, and
    tracked flags to the agenda JSON with a new golden.'
- id: plugins-untracked
  title: 'bob-plugins: render #untracked and keep Obsidian keymaps in lockstep'
  size: medium
  depends_on:
  - gcal-untracked-tag
  - meet-event-run
  - event-start-warnings
  description: 'plugins-untracked: paint #untracked on Event rows, keep \p and home
    chips untracked-aware, and mirror MEET starts per the decision.'
- id: mac-calendar-band
  title: 'bob-mac-capture: Calendar band in the idle agenda'
  size: medium
  depends_on:
  - agenda-events
  description: 'mac-calendar-band: decode the events block and add the folding Calendar
    band, rows, header, chips, and Command-J join.'
- id: mac-rail-and-cards
  title: 'bob-mac-capture: day rail, start-card event runs, conflict banners'
  size: medium
  depends_on:
  - mac-calendar-band
  - meet-event-run
  - event-start-warnings
  description: 'mac-rail-and-cards: draw the day rail and render event runs, far-run
    and conflict banners, and untracked Now cards.'
proposed_by: bbugyi200.apollo.bv
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 11:24:10
status: done
bead_id: bob-cli-f6
---

- **PROMPT:** [prompts/202611/untracked_calendar_events.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/untracked_calendar_events.md)
- **BEAD:** [bob-cli-f6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-f6/README.md)

# Untracked calendar events, calendar-aware MEET starts, and a Calendar band

## Goal

Let Bryan tag a Google Calendar event `#untracked` (lunch, school pickup, the gym) so it
still lands in the day as an Event Task with a Pomodoro Task Link, but its session is
**on the calendar and off the clock**. Along the way, make `=` start a MEET on the real
span of the next back-to-back events, warn when an ordinary session would run into an
event, and give Bob Mac Capture's idle view a glanceable, beautiful summary of what is
coming up today.

## The design in one page

**One fact, one representation.** A timed Pomodoro row _with_ `[t:: Nm]` is tracked; a
timed row _without_ it is **untracked**:

```markdown
- [x] (**1100-1130** [t:: 30m]) — MEET ← tracked: counts 30m
- [x] (**1200-1300**) — MEET ← untracked: on the ledger, off the clock
  - 🍅 [[#^lunch-with-kelly]]
```

This is exactly Bryan's mental model ("which Pomodoros should receive a `t::` property
or not"), it reads honestly in Obsidian, and it is safe: today every reader falls back
to the span when `[t::]` is missing, but a scan of the vault finds **0 of 1,226** closed
timed rows without `[t::]`, so retiring that fallback rewrites no history. The tag on
the Event Task is the _input_; the ledger row is the _record_. Writers decide
trackedness when they write a span; readers only look at the row.

**Where the tag lives.**

- In Google Calendar: a standalone `#untracked` token anywhere in the title
  (case-insensitive), or a description line containing only `#untracked`. Sync strips it
  from the displayed title, the way a resolved `@route` is stripped (strip the tag
  first, then look for the title-ending `@route`, so `Lunch @love #untracked` routes to
  `love`).
- On the Event Task: `#untracked` written at the end of the badge run, before
  `[root::]`, so every existing parser (Rust `split_row_spans`, the Obsidian Event
  painter, the JS clock parser) still finds span, title, and `^id`:

  ```markdown
  - [*] #task **1200-1300** Lunch with Kelly [📅](… "Open in Google Calendar")
    #untracked [root:: love] [type:: MEET] ^lunch-with-kelly
  ```

  The tag is Google-owned exactly like the badges: updated in place while the row is
  untouched, and your vault edit (adding or deleting it) wins and is reported once
  (`kept_edit`). Hand-written Event Tasks can carry it too.

**MEET follows its event run.** When `=` (empty suffix) starts or restarts a session
named MEET, the span is the **event run**: the earliest not-yet-ended, open Event Task
in its lineup plus every following one that touches or overlaps it
(`next.start ≤ run.end`, the `cluster_past` rule) _and shares its trackedness_. The run
decides both the span and whether `[t::]` is written. A counted suffix (`=5#meet`, `=3`)
is the override: explicit timing, but trackedness still follows the run. Non-MEET
sessions are always tracked.

**Non-MEET sessions warn before they run into an event.** Starting, restarting, swapping
in, link-starting, extending (`+N`), or shifting (`++N`/`--N`) a non-MEET session warns
when its new span newly crosses the start of an open Event Task — with a one-token fix:
`⚠ BOB 1150-1215 runs 15m into Standup (1200-1230) · =2 ends at 1200`.

**Dashed means off the clock**, everywhere: the Obsidian Event time pill and
`#untracked` mark, the Obsidian home chip of an untracked session, and the Mac band's
untracked rows and rail capsules all use a dashed outline. One visual word, three
surfaces.

**The Calendar band** (Bob Mac Capture idle view). bob judges, the app paints:

```text
Today · Sun Nov 1 · 3 done · 1h 15m
▌▶ Now  BOB  1150-1215 · 18m left
▌   1 ◯ Fix the capture override preview
📅 Calendar · Standup at 13:00 · in 38m                 4 left · 2h 30m · ↻ 08:02
   ┊12      ▌now ═══BOB═══      13 [Standup][1:1 ]  14        15 [Review····]  16
   38m free · =7 fits
   13:00–13:30  ● Standup                          📹  in 38m     x =meet
   13:30–14:00  ● 1:1 with Sam                     📹
   15:00–16:00  ○ Lunch with Kelly   ┆untracked┆
   +1 later · 2 earlier
○ Next  SASE  …
```

It sits right under the Now card (or the title when nothing runs): first "what am I
doing", then "what is my next hard commitment", then the flexible queue. It answers at a
glance: next commitment and countdown, free time before it (and whether a default
Pomodoro fits), the shape of the rest of the day, a collision between the running
session and the next event, which events are off the clock, a one-key join, and how
fresh the calendar is (`↻ 08:02`, since sync is not periodic).

## Decisions made here (not reviewer choices)

- **Absence of `[t::]` is the representation**; no new field or ledger tag.
- **The tag token is accepted anywhere in the Google title**, not only at the end:
  hashtags read naturally anywhere, and an accidental match is unlikely.
- **Past clusters split by trackedness** in sync (two closed MEETs may overlap; that is
  truthful). The single running MEET sync may create is untracked only when every
  current event is untracked (one running entry is allowed, so mixed current events stay
  tracked; documented as a known limit).
- **Event-run auto-range is keyed on the MEET name** (`normalize_component` == `MEET`),
  the session type sync writes; the same concept drives the warning exemption. An
  empty-suffix _swap_ (`=meet` while BOB runs) keeps today's kept-clock semantics; its
  override warning already offers `x =meet`.
- **Late joins never double count**: the run's start is clamped to the end of today's
  latest closed session when that ends inside the run.
- **Early starts are truthful**: if the run starts after the next 5-minute slot, the
  session is written with the run's future start ("starts in 10m"). A run more than 25
  minutes away (a default Pomodoro would fit first) adds a warning with the `=5#meet`
  escape hatch.
- **Command-J joins the focus meeting** from an empty draft: opening a URL writes
  nothing, so the idle agenda stays read-only.
- **No new capture grammar.** Every behavior is a default of existing operators, so
  `capture-parse`/`capture-complete` and the Mac grammar surface are unchanged.
- **One payload.** Events join `bob capture-pomodoros --tasks` additively (schema stays
  1; fields omitted when empty/true), so the Mac app keeps one fetch, one cache, and one
  refresh pipeline ([[decisions:idle-capture-shows-ledger-agenda]],
  [[decisions:mac-capture-is-a-thin-client]]).
- **Naming collision guard:** `bob stats` already calls days without data "untracked
  points" (`src/native/stats/compute.rs`, `docs/stats.md`). Code for the new concept
  uses `SessionTracking::{Tracked, Untracked}` / "untracked session" and docs always say
  "untracked session".

Rejected alternatives: `[t:: 0m]` (reads like a bug, still counts as a Pomodoro); an
`#untracked` tag on the Pomodoro line (changes the ledger line grammar in Rust, JS, and
Swift parsers); deriving trackedness at read time from links (opaque, needs link
resolution in every reader, cannot be overridden per session); splitting a MEET
placeholder into running + queued entries on start (leaves two open MEETs and trips
`duplicate_open_pomodoro_name`).

## Shared definitions (all phases)

- **Event Task** (unchanged): `src/native/event_task.rs`; a checkbox row in the daily
  note's `## Events` section whose description starts with `**HHMM-HHMM**`. **Untracked
  Event Task**: its line carries a standalone `#untracked` tag token (case-insensitive,
  whitespace-bounded, not inside a Markdown link or code span).
- **Open Event Task**: checkbox not done (`x`/`X`) and not cancelled (`-`).
- **Tracked row**: a timed ledger row whose parentheses hold `[t:: …]` or the legacy
  `⏱` stopwatch (`duration_from_range_text` returns `Some`). **Untracked row**: timed,
  neither. Placeholders `()` are neither.
- **Event run** (computed at write time on the staged daily note): see `meet-event-run`.

## Phase details

### untracked-ledger (bob-cli)

1. `event_task.rs`: add `untracked: bool`, the checkbox status, block id, and clean
   title to the parsed Event Task; add `events_in_note(contents, date)` returning every
   Event Task row (and timed/all-day context rows, flagged as such) in time order, and a
   pure helper `first_event_start_crossed(span, events, exclude_ids) -> Option<…>` (open
   Event Tasks whose start is in `[S, E)`), plus the `cluster`-by-trackedness helper.
   These are the single source for phases 2–5.
2. One shared predicate, e.g.
   `pomodoro::session_tracking(ledger_line) -> Tracked { minutes } | Untracked | Placeholder`,
   used by every reader:
   - `stats/index.rs` `PomodoroSession` gains the tracking state; stats metrics, homes
     attribution, session facts, and the Day strip skip untracked sessions' minutes;
     health gains an additive `untracked_sessions` count. The span-estimate `Partial`
     path for closed sessions is retired (keep `Missing` for range-less rows).

     > [!decision] untracked_counts = exclude Untracked sessions also leave every count:
     > `measure: count` metrics, count-mode lead homes, the Day strip's Pomodoro count,
     > and `completed_summary.count`.

     > [!decision] untracked_counts = minutes_only Untracked sessions keep counting as
     > one Pomodoro each (count metrics, lead homes, `completed_summary.count`); only
     > minutes drop to 0.

   - `capture_pomodoros.rs` `completed_summary_for` / `attributed_minutes` (and
     therefore `bob pomodoro --worked` and the agenda's `completed_summary`) skip
     untracked sessions.
   - `pomodoro_homes`: a closed untracked entry is still attributed with 0-minute parts
     and gains additive `untracked: true` on its homes object. Update
     `tests/fixtures/pomodoro_homes/vectors.json` (`split-rounding-and-duration` pins
     "no `[t::]` uses the span"; change it and add an untracked case) so bob-plugins can
     copy it.

3. Writers preserve absence: `format_adjusted_range` callers in
   `capture/pomodoro_adjust.rs` (adjust, shift) and `capture_pomodoro_close/ledger.rs`
   `close_timing` keep an untracked row untracked (render `(**HHMM-HHMM**)` with any
   other metadata intact).
4. Docs: `docs/stats.md` (minutes measure; "untracked session" wording),
   `docs/pomodoro-homes.md` (R5), `docs/reference/vault-format.md` (ledger: untracked
   rows; Events: `#untracked`), `docs/capture.md` (adjust/shift/ close keep
   trackedness).

   > [!decision] memory_event_task_glossary Add one sentence to the Event Task glossary
   > strand: an `#untracked` tag keeps the session that links it off the clock (a timed
   > ledger row without `[t::]`), so its `[root::]` attributes no minutes. Republish
   > with `sase memory init`.

5. Tests: unit tests for the predicate and helpers; stats goldens with one untracked
   closed row; adjust/shift/close round-trips on an untracked row.

### gcal-untracked-tag (bob-cli)

1. `gcal/model.rs`: extract `#untracked` from title tokens and from a description line
   containing only `#untracked`; keep a cleaned title, then compute `title_route_hint`
   from the cleaned title.
2. `gcal/render.rs`: append ` #untracked` at the end of the badge run (`badge_text`), so
   journal span hashes stay valid for rows without the tag and a newly tagged untouched
   row updates in place. `lifecycle.rs` `split_row_spans` must keep the tag in the badge
   span even when there is no `📅` badge. Context rows strip the token and carry no tag;
   an upgrade to a task renders it.
3. `gcal/plan.rs` `plan_meet_pomodoros`: split past clusters by trackedness and render
   untracked closed MEETs without `[t::]` (`format_meet_span` takes the tracking state);
   a created running MEET is untracked only when every current event is. Sync still
   never retimes existing entries: a tag flip after a session closed is reported once
   and left for a hand edit.
4. `bob gcal list`: VAULT column `task · MEET · untracked`; JSON events and `sync` rows
   gain `untracked`. Pre-existing synced rows whose Google title already contains
   `#untracked` migrate on the next sync (title and badge spans both still match Bob's
   last write, so both update in place) — cover with a lifecycle test.
5. Docs: `docs/gcal.md` (new "Untracked events" section, Sync table, ownership table,
   Root selection order note, Known limits), vault-format Events.

   > [!decision] memory_untracked_record Write
   > `sase/memory/decisions/untracked-sessions-omit-t-field.md` in the decisions-web
   > format (Applies to / Claim / Why with rejected alternatives / Cost / Reopens when /
   > Evidence, citing this plan, `docs/gcal.md`, and the tests): a timed ledger row
   > without `[t::]` is an untracked session; writers decide (sync splits past clusters
   > by trackedness, MEET starts follow the event run) and readers only look at the row.
   > Mark [[decisions:synced-meetings-link-by-phase]] `superseded-in-part` with
   > `superseded_by` and a `[[...]]` back-link naming the retired cluster claim.
   > Republish with `sase memory init`.

6. Tests: `tests/gcal_sync.rs` and `tests/gcal_lifecycle.rs` cases for title token,
   description token, `@route` + tag, mixed past cluster, all-untracked running MEET,
   tag added/removed in Google on untouched and edited rows.

### meet-event-run (bob-cli)

1. **Event run** for a session, on the staged note at `now`: candidates are the
   session's lineup links after drops (`list_queued_links`) that are plain same-note
   links resolving via `event_task_by_id` to an open Event Task whose span has not ended
   (`end_on_timeline > now`). Sort by (start, end, ledger order); take the first; extend
   while `next.start ≤ run.end` and `next.untracked == run.untracked`. Start =
   `max(run.start, end of today's latest closed session if it ends inside the run)`; end
   = run end (midnight-crossing spans use the existing modulo rules).
2. Apply it in `capture/pomodoro_start.rs` when the suffix is empty (no digits, no
   offset, no `@` timing) and the session that ends up running is MEET: plain start of
   the next future Pomodoro, `=meet`/`=#meet` on an open MEET placeholder, and an
   empty-suffix restart of a running MEET. Created MEETs have an empty lineup and keep
   the 25-minute default; an empty run keeps today's behavior.
3. **Trackedness rule** for every start-family write whose session is MEET (including
   counted suffixes, kept-ledger swaps, and link starts): omit `[t::]` exactly when the
   run is non-empty and untracked; a kept-ledger swap keeps its span bytes and drops or
   restores `[t::]` accordingly. All other sessions are tracked.
4. Early start: a run starting after the next 5-minute slot is written with its future
   start. If the run starts more than 25 minutes from now, add a structured warning
   (`pomodoro_event_run_later`): "MEET's next event, Lunch, starts at 1400 (2h 10m from
   now) · capture `=5#meet` to start 25 minutes now". Close safety: `close_timing` never
   moves an end before the start (clamp; warn that `x0` resets instead).
5. JSON (schema 1, additive): `pomodoro_start.tracked` (omitted when true),
   `pomodoro_start.event_run`
   `{time_range, start, end, untracked, source: "calendar"|"explicit", events: [{index, title, time_range, line, block_id, untracked}]}`;
   `pomodoro_blocks[].tracked` (omitted when true). Human output:
   `started MEET 1200-1300 (60m · untracked) · from your calendar: Lunch with Kelly`.
6. Shared vectors `tests/fixtures/meet_event_run/vectors.json` (+ README, pinned commit)
   consumed by a Rust test, for bob-plugins to copy: lineup, statuses, tags, closed
   sessions, now → expected span and tracked.
7. Docs: `docs/capture.md` (lifecycle table, Starting the next Pomodoro, Starting a
   named Pomodoro, Overriding the running Pomodoro, Closing, JSON),
   `docs/reference/capture-syntax.md` if it states "`=` is 25 minutes".
8. Tests: back-to-back tracked run; untracked run then contiguous tracked event (run
   breaks); late join clamp; early start; far-future warning; counted override keeps
   trackedness; restart re-derives; drop `~K` removes an event from the run; ended and
   done events skipped; close-before-start clamp.

### event-start-warnings (bob-cli)

1. After every operator that starts or moves a session's end later — whole- item starts
   (plain, named, created, restart, swap including kept), link starts (`^route:id=`,
   `@route:id=`), `+N`, `++N`/`--N` — when the resulting running session is not MEET,
   compute the open Event Tasks whose start is newly crossed (in the new `[S, E)` but
   not the old one) on today's note, excluding events in that session's own lineup. `-N`
   and closes never warn. Plumb `now` and `warnings` into adjust/shift
   (`plan.rs:770–776` today passes neither).
2. Emit one structured `CaptureWarning` (`code: pomodoro_runs_past_event`) naming the
   earliest crossed event, the overlap, `and N more`, and a fit draft: the largest
   same-operator count that ends at or before that event's start (`=2`, `=2#bugs`,
   `=2-1`, `+K`, `++K`), or none when no positive count fits. Add
   `event_conflict {event:{title,time_range,line,untracked}, overlap_minutes, more, fit}`
   to `pomodoro_start`, `pomodoro_adjust`, and `pomodoro_shift` JSON. Human:
   `⚠ BOB 1150-1215 runs 15m into Standup (1200-1230) · =2 ends at 1200`. Chains warn
   per item, only for new crossings, so `= +2` warns once.
3. Docs: `docs/capture.md` (a "Running into an event" subsection linked from
   start/adjust/shift).

   > [!decision] warn_untracked = all Untracked Event Tasks warn like any other open
   > Event Task.

   > [!decision] warn_untracked = tracked_only Skip untracked Event Tasks when computing
   > crossed events, in capture and in the agenda's `conflict` field; document that
   > untracked events never warn.

4. Tests: start/restart/swap/link start/adjust/shift, MEET exemption, own- lineup
   exemption, done/cancelled events, already-crossed span not re-warned, fit drafts
   including no-fit, `--dry-run` parity.

### agenda-events (bob-cli)

Extend `bob capture-pomodoros --tasks` (schema 1, additive; omitted when the note has no
`## Events` section):

```json
"events": {
  "synced_at": "2026-11-01T08:02",
  "all_day": [{"line": 7, "title": "School holiday", "qualifier": "day 2 of 3"}],
  "rows": [{
    "line": 9, "block_id": "standup", "kind": "task", "qualifier": null,
    "title": "Standup", "time_range": "1300-1330",
    "starts_at": "2026-11-01T13:00", "ends_at": "2026-11-01T13:30",
    "duration_minutes": 30, "phase": "upcoming",
    "status_symbol": "*", "status_type": "TODO", "untracked": false,
    "root": "job", "home": {"label": "job", "hue": …},
    "join_url": "https://meet.google.com/…", "calendar_url": "https://…",
    "location": null,
    "pomodoro": {"line": 31, "role": "next", "index": 1}
  }],
  "focus": 0,
  "gap": {"from": "12:15", "until": "13:00", "minutes": 45, "fits": "=9"},
  "conflict": {"pomodoro_line": 22, "event": 0, "overlap_minutes": 10},
  "start_hint": "x =meet",
  "refresh_at": "2026-11-01T13:00"
}
```

- `phase` is `ended`/`running`/`upcoming` at bob's now; `focus` is the earliest running
  Event Task, else the next upcoming one (null when none); `gap` runs from the running
  session's end (else now's 5-minute slot) to the focus start, with `fits` (largest
  plain `=<X>` that ends by then) only when nothing runs; `conflict` is the running
  non-MEET session crossing the focus start (shared helper from `untracked-ledger`);
  `start_hint` is `=meet` / `x =meet` when the focus is linked under a MEET that is not
  running and is running or starts within 25 minutes; `refresh_at` is the next phase
  boundary. `home` reuses the R9 chip/hue shape the agenda already reports; `join_url` /
  `calendar_url` / `location` come from the row's badges.
- `items[].event` `{time_range, starts_at, ends_at, untracked, title}` on items
  resolving to an Event Task in the daily note; entry `tracked: false` on untracked
  timed entries; `completed_summary.untracked_count` (omitted when 0).
- Human `--tasks` output gains a compact Events block.
- Goldens: new `tests/fixtures/capture_pomodoros/agenda-events.json` (busy day: running
  BOB crossing Standup, untracked lunch, context rows, stamp); existing goldens must
  stay byte-identical. Docs: `docs/capture.md` (`capture-pomodoros`),
  `docs/reference/json-outputs.md`.

### plugins-untracked (bob-plugins, via `/sase_repo`)

1. Event painter (`plugins/bob-ledger-tools/src/291-plugin-gcal-events.js`, `styles.css`
   ~4110–4210): recognize `#untracked` on Event rows; render it as a quiet mark (Lucide
   `timer-off` icon + "untracked", muted ink, dashed outline) and switch the row's time
   pill to a dashed outline, in Live Preview and Reading view. Extend
   `docs/bob-ledger-tools.md` (Calendar events) and the mounted bob-cli plugin reference
   if it is generated from it.
2. `N\p`/`N\P` (`020-time-and-pomodoro.js` `changePomodoroLineUnits` /
   `durationMetadata`) keep an untracked row untracked; `N\o`/`N\O` and Ctrl+Enter
   already do — add regression tests.
3. Home chips (`142-pomodoro-home-chips.js`): untracked sessions contribute 0 minutes,
   chip drawn with a dashed outline, tooltip "untracked · not counted"; copy the updated
   `pomodoro_homes` vectors byte-for-byte.
4. Obsidian MEET starts:

   > [!decision] obsidian_meet_start = full Copy the `meet_event_run` vectors; `se` +
   > Tab with an empty X on a MEET follows the event run span, and every `se<X>` on a
   > MEET follows the `[t::]` rule (`030-snippets.js`, `020-time-and-pomodoro.js`); a
   > Notice mirrors `pomodoro_runs_past_event` for non-MEET `se<X>`, `\p`, and `\o`.

   > [!decision] obsidian_meet_start = tracked_only Copy the vectors for trackedness
   > only: `se<X>` + Tab keeps today's timing but omits `[t::]` when the MEET's event
   > run is untracked. No Notice.

   > [!decision] obsidian_meet_start = none Skip this step; document in the plugin
   > reference that untracked MEETs should be started from capture or have `[t::]`
   > deleted by hand.

5. Add an untracked case to the copied `event_task_close` vectors if bob-cli added one.
   `npm test`, `npm run build`, `build:check`, `validate`, `docs:check`; deploy with
   `bob plugins sync`.

### mac-calendar-band (bob-mac-capture, via `/sase_repo`)

1. Models (`CaptureAgendaModels.swift`): decode `events`, `items[].event`, entry
   `tracked`, `completed_summary.untracked_count` with `decodeIfPresent` and
   `.other(raw)` enum fallbacks; an older bob simply shows no band.
2. Store: schedule a refresh (`.eventBoundary` reason) at `refresh_at` while visible;
   widen the `TimelineView(.everyMinute)` gate to "live countdown or band with a
   running/upcoming event".
3. Presentation + planner: a new `calendar` unit placed after the Now group (after
   title/budget rows when nothing runs), with fold states `full` (header, all-day line,
   rail slot, gap row, rows through the last upcoming, `+N later · M earlier` chips) →
   `compact` (header, rail slot, focus row, `+N` chip) → `oneRow` (header plus "then 1:1
   13:30 · +2"). Ladder insertion: band `full → compact` after Later groups reach
   `oneRow` and before they go into the strip; band `compact → oneRow` after Next groups
   go `oneLineTasks` and before Now does. Never hidden while rows remain; one quiet
   "Calendar clear for the rest of the day · ↻ 08:02" row when every event has ended;
   hidden when there is no `events`. Ended rows appear only when the `earlier` chip
   expands. Add the new row kinds to the four exhaustive switches.
4. Views: header (`calendar` SF Symbol; "Standup at 13:00 · in 38m" or "Lunch until
   13:00 · 23m left", monospaced digits; accessory "4 left · 2h 30m · ↻ 08:02", weekday
   shown when the stamp is from another day); event rows (monospaced time column,
   home-hue dot — hollow and dashed for untracked —, title, dashed "untracked" capsule,
   italic dimmed Maybe/free rows, 📹 join button, trailing countdown or status glyph,
   `start_hint` chip on the focus row); amber conflict row ("BOB runs 10m into
   Standup"); quiet gap row ("38m free · =7 fits"). MEET group items with `event` show a
   time chip instead of the bold span. Command-J in an empty draft opens the focus row's
   `join_url` (check it is free; document in `docs/keyboard.md`).
5. Fixtures: copy `agenda-events.json`; CaptureCore tests for decoding, presentation,
   and planner folds (run `just portable-check` on Linux); app tests on macOS CI. Docs:
   `docs/idle-agenda.md`.

### mac-rail-and-cards (bob-mac-capture, via `/sase_repo`)

1. Day rail (one row, about 16 pt) in the band's `full`/`compact` states: domain from
   the current hour to the later of the last remaining event end and the running session
   end, at least 3 hours, hour ticks in `.caption2` tertiary; event capsules tinted with
   their home hue (tracked filled, untracked dashed outline, Maybe dotted); the running
   Pomodoro as a pink bar on a sublane so a collision with an event is visible; a now
   hairline; the first gap labelled when wide enough. Decorative for VoiceOver (the rows
   carry the facts). Light, dark, and increased-contrast renders.
2. Start/adjust/shift preview cards: render `pomodoro_start.event_run` ("MEET
   12:00–13:00 · from your calendar · Lunch with Kelly · untracked"),
   `pomodoro_event_run_later`, and `event_conflict` banners in the override banner's
   visual language, showing the fit draft. The Now card shows "starts in 10m" when
   `starts_at` is in the future and a dashed "untracked" pill for `tracked: false`.
3. Render fixtures (`CaptureAgendaDesignTests`, 760/620, light/dark) for a busy calendar
   day; refresh `docs/idle-agenda.md` images and `docs/capture-previews.md`.

## Verification

- bob-cli phases: `just check` (or the repo's documented gate), the touched goldens,
  `just docs-check`; `--dry-run` parity for every new capture path.
- Cross-repo lockstep: `pomodoro_homes`, `meet_event_run`, and `event_task_close`
  vectors copied byte-for-byte into bob-plugins and pinned to a bob-cli commit.
- End to end on a scratch vault (`BOB_NOW`, fake `BOB_GCAL_GOG`): sync an untracked
  lunch between two meetings → `=` at 11:58 starts `(**1200-1300**) — MEET`, `x` at
  13:00 completes Lunch and carries Standup, `x =meet` starts
  `(**1300-1400** [t:: 60m]) — MEET`; `bob stats` and `bob pomodoro --worked` exclude
  the lunch; `=` on BOB at 12:45 warns with `=3`.
- Mac: macOS CI green; render artifacts reviewed for light/dark and both widths.
- Releases: bob-cli ships through the daily train
  ([[decisions:bob-cli-releases-ride-a-daily-train]]); every JSON change is additive and
  the app decodes with `decodeIfPresent`, so either side can land first.
