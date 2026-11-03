---
tier: epic
title: Event link times — the Event Task's time pill beside ledger Task Links to it
goal: 'In a daily note''s Pomodoro ledger, every Task Link that points at one of that
  note''s Event Tasks wears the same time pill the Event Task wears in `## Events`
  (meeting span, past/running/future state, MEET hue, off-clock dash, buffer wings),
  in Live Preview and Reading view, display-only, so a MEET lineup reads as a schedule.

  '
decisions:
  placement:
    ask: Where should the time pill sit on a ledger Task Link row?
    choices:
      before: Leads the link like the Events row, so MEET links read as a column of
        times
      after: Trails the link, like bob's "Swings! (0800-0815)" copy; 🔓/🔑/off must
        skip it
    default: before
    why: Mirrors the Events row (time, name, off chip) and gcal sync's time-ordered
      MEET links
    answer: before
phases:
- id: contract
  title: Authoritative display contract (bob-cli)
  depends_on: []
  size: small
  description: 'contract: write docs/event-link-times.md with which-links, resolution,
    look, placement, tooltip, liveness, toggles, and vector tables; register it and
    add two cross-links.'
- id: pill
  title: The pill in bob-ledger-tools (bob-plugins)
  depends_on:
  - contract
  size: medium
  description: 'pill: fix the shared Event pill clock bug, then implement the contract
    in bob-ledger-tools (pure core, Live Preview widget, Reading sweep, gates, liveness,
    tests, docs, build, sync).'
- id: live_verify
  title: Real-Obsidian e2e and visual review (bob-plugins, bob-cli)
  depends_on:
  - pill
  size: medium
  description: 'live_verify: isolated real-Obsidian e2e with light/dark screenshots,
    then a Live verification section in the bob-cli contract.'
proposed_by: bbugyi200.apollo.ca
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 10:19:35
status: wip
bead_id: bob-cli-gg
---

- **PROMPT:** [prompts/202611/event_link_times.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/event_link_times.md)
- **BEAD:** [bob-cli-gg](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gg/README.md)

# Event link times

## Goal

In `## Events`, every Event Task's leading `**HHMM-HHMM**` span renders as a quiet time
pill (ledger-tools "calendar event painting"). On today's note it dims once the meeting
is past, gets an outline in the MEET hue while the meeting is running, turns dashed when
the event is off the clock (tagged or inherited), and grows hatched `30m` wings for a
`[buffer::]`.

Under `## Pomodoros`, the Task Links that `bob gcal sync` files for those events
(`[[#^swings]]`) render as a bare `^swings` with no time at all, so a MEET lineup gives
no idea when anything happens. Bryan asked to render the same time range that Event
Tasks show next to the Task Links that point at them (vault task
`bob.md#^event-task-link-range`).

```text
before:
- [x] (**0815-0830** [off:: 15m]) — MEET
	- ~~^swings~~
- [ ] () — MEET
	- ^virtual-cfg-appt
	- ^disney-movie-night

after (placement = before):
- [x] (**0815-0830** [off:: 15m]) — MEET
	- ⟦0815-0830⟧ ~~^swings~~                 (past: dimmed; #off: dashed)
- [ ] () — MEET
	- ⟦30m|1830-1900⟧ ^virtual-cfg-appt        (buffer wing; #off: dashed)
	- ⟦1900-2030⟧ ^disney-movie-night
```

(`⟦…⟧` stands for the existing `bob-calendar-event-time` pill and its wings.) Bob Mac
Capture already shows these times: `bob capture-pomodoros --tasks` attaches an `event`
chip to every Task Link that targets one of today's Event Tasks. Obsidian is the surface
that lacks them.

## What exists today (read before implementing)

All paths are in the **bob-plugins** linked repo (open it with `/sase_repo` and read its
`AGENTS.md`) unless they start with `docs/` or `src/native/`, which are bob-cli.

- **The pill to reuse.** `plugins/bob-ledger-tools/src/291-plugin-gcal-events.js`:
  `gcalEventsScanMarkdown`, `gcalEventsTimeState`, `gcalEventsBufferPhase`,
  `gcalEventsNowOnTimeline`, `calendarEventMeetHue()`, `calendarEventSourceLines(ctx)`,
  `refreshCalendarEventPaint()` (dispatches `gcalEventsRefreshEffect` to today's editors
  and re-states Reading records), `calendarEventUpdateClockTimer()` (a 60 s tick while
  today's daily note is active and visible), `setCalendarEventPaintGate()` (sets
  `bob-calendar-events` on the **main window's** body only), and
  `toggleCalendarEventPainting()`. `src/291b-plugin-gcal-event-buffers.js`:
  `gcalEventsTimeAttributes(row, hue)` (every `data-*` attribute plus a `title`),
  `gcalEventsWingModel`, `buildCalendarEventBufferWing`.
  `src/291c-plugin-gcal-event-clock.js`:
  `gcalEventsApplyClockToRow(row, line, path, metadataCache, vault)` (tag, then
  inherited `offclock:`), `gcalEventsClockTooltip`, `gcalEventsFromNoteTitle`. CSS in
  `plugins/bob-ledger-tools/styles.css` (about line 5200):
  `.bob-calendar-events .bob-calendar-event-time` and its `data-state`,
  `data-theme-hue`, `data-off-clock`, and `data-buffer-*` variants,
  `.bob-calendar-event-time-wrap`, `.bob-calendar-event-buffer` wings, and the
  dark-theme rules. User docs: bob-plugins `docs/bob-ledger-tools.md` → "Calendar
  Events".
- **Event Task rows.** `src/035-meet-event-run.js`: `eventRunEventRows(lines)` (every
  checkbox row in `## Events` with a leading `**HHMM-HHMM**`, `#task` optional, fences
  skipped; each row has `lineIndex`, `timeRange`, `startMin`, `endMin`, `buffer`,
  buffered span, `blockId`, cleaned `title`) and `eventRunById(rows, blockId)` (the lone
  row with that id, else null). This matches bob's `parse_event_task_line` in
  `src/native/event_task.rs`.
- **The Mac's rule to stay equal to.** `event_item_chips` in
  `src/native/capture_pomodoros_agenda.rs`: a chip only for a link whose target is the
  agenda's own daily note, only for an Event Task row with a span, and never for a block
  id carried by two Event rows.
- **The ledger-row rule.** `src/148-task-link-marks.js`:
  `taskLinkMarkIsLedgerTaskLinkLine(lines, index, path)` and
  `taskLinkMarkLedgerBodyIsDedicated(body)` accept a direct child bullet, at the first
  child indent, of a column-0 Pomodoro entry (open or closed) in a canonical daily note,
  whose body is exactly one `[[target#^id]]` (alias allowed), optionally after `🍅 `,
  optionally struck `~~…~~`, optionally followed by ` #`. `progressMarkMatchFromBody`
  (`src/137-progress-marks.js`) is narrower (it rejects struck rows), so do not use it;
  completed meetings are usually struck.
- **Neighbours on the same row.** At the link start, Live Preview draws the ledger
  `#TYPE` mark at `side: -2` (`src/268-plugin-ledger-type-marks.js`) and ◐ at `side: -1`
  (`src/268-plugin-progress-marks.js`). At the link end it draws 🔓 at `side: 1`
  (`src/271-plugin-unblocked-glyph.js`), 🔑 at `side: 2`
  (`src/271b-plugin-frees-marks.js`), and, once epic `bob-cli-gf` lands, the off-clock
  chip at `side: 3`. Reading view maps a rendered `li` to its link through
  `progressMarkReadingContext(li, root, {includeStruck, includeClosedEntry, allowLedgerMarks})`
  in `src/269-plugin-progress-marks-reading.js`, which returns
  `{anchor, target, blockId}`. Its `ledgerMarkClass` allow-list names every display-only
  host that may sit beside a ledger link. A host missing from that list makes the row
  stop counting as a ledger row: ◐, 🔓, and 🔑 vanish and a status ring appears.
- **Plumbing.** `applyMarkGateClass` / `registerMarkGatePopoutSync`
  (`src/266-plugin-mark-gate-mirror.js`), `setupGcalEvents()` (291),
  `scheduleLiveWidgetRefresh()` (`src/220-plugin-heading-and-review.js`), mixin
  installation in `src/310-install-methods.js`, test exports in `src/350-exports.js`,
  fragment order in `src/fragments.json`. `src/170-plugin-lifecycle.js` is at the
  1000-line fragment cap, so hook setup from `setupGcalEvents()` instead.
- **Sibling contracts and tests to copy.** bob-cli `docs/frees-marks.md` and
  `docs/task-link-marks.md`; bob-plugins `scripts/test-ledger-tools-frees-marks.cjs`,
  `scripts/test-ledger-tools-gcal-events.cjs`, and the
  `scripts/obsidian-frees-marks-e2e.cjs` / `-lib.cjs` /
  `test-obsidian-frees-marks-e2e.cjs` harness.

### A bug in the pill being echoed

`gcalEventsNowOnTimeline(now, start)` adds 1440 to `now` whenever `now < start` and the
event starts at or after 12:00. Since `13e63d4` (buffer wings), every afternoon or
evening Event pill on today's note therefore paints `past` **before** it starts. At
10:00 an 18:30 meeting is dimmed as past (checked against the current source:
`gcalEventsTimeState("1830-1900", 600, true, true)` returns `"past"`). The link pill
reuses this function, so phase `pill` fixes it first. Only a span whose occupied end
crosses midnight should move an early `now` onto the next-day timeline: shift only when
`occupiedEnd > 1440 && now < occupiedStart && now + 1440 < occupiedEnd`. Pass the
occupied end into the helper. `gcalEventsBufferPhase` uses the same helper.

## Coordination with epic `bob-cli-gf` (off-clock link chips)

`bob-cli-gf` adds a trailing `off` chip to the same ledger rows and is in flight in the
same files: bob-plugins `269` allow-list, `310`, `350`, `fragments.json`, `styles.css`,
`package.json`, `manifest.json`, `docs/bob-ledger-tools.md`; bob-cli
`docs/_meta/contracts.yml`, `mkdocs.yml`, and `docs/task-link-marks.md`. Each phase
starts from the latest master of both repos. On any conflict, keep both features, and
never drop or reorder gf's code. The design does not depend on gf landing: under
`placement = before` the pill sits at the opposite end of the link from gf's chip. When
gf is present, the tests also assert the combined order.

## Design (decided)

### Principles

- **Display-only.** Nothing writes. Without the plugin the vault looks exactly as it
  does today.
- **The same pill.** The inner element is a real `bob-calendar-event-time` span whose
  `data-*` attributes come from `gcalEventsTimeAttributes`, over a row model built the
  same way the Events painter builds its rows. The existing CSS draws it, so a style
  change to the Events pill changes the link pill too. There is no new pill CSS.
- **About the event, not the session.** The pill shows the Event Task's meeting span and
  state. It never reads or judges the Pomodoro entry's own span or `[t::]`/`[off::]`.
- **Equal to the Mac.** A link qualifies exactly when Mac Capture would attach an
  `event` chip: a same-note link to a unique Event Task row with a span.
- **Truthful or neutral.** Unresolved or cross-note links, duplicate ids, context or
  all-day rows, missing source text, and any thrown error show no pill and no warning.

### Which links get a pill

All of these must hold:

1. The rendered note is a canonical daily note (`PLAN_DAILY_PATH_RE`), of any date. The
   pill shows under any Pomodoro entry, open or closed, as the Events pill shows on
   every daily note.
2. The line is a dedicated ledger Task Link row per `taskLinkMarkIsLedgerTaskLinkLine`:
   plain, `🍅`-prefixed, struck, aliased, or followed by ` #`.
3. It is not an embed, mixed prose, a deeper bullet, fenced, in frontmatter or a `%%`
   comment, or in `## Events`, `## Notes`, or any other section.
4. The link targets the rendered note itself: an empty linkpath (`[[#^id]]`), or a
   linkpath that `metadataCache.getFirstLinkpathDest(linkpath, sourcePath)` resolves to
   the same file (`[[2026/20261103#^id]]` written inside `2026/20261103.md`).
5. `eventRunById(eventRunEventRows(lines), blockId)` finds exactly one Event Task row.
6. The "Task Link event times" toggle is on **and** calendar event painting is on (the
   pill is that painting's look).

### Resolution

- **Text:** the row's `timeRange` exactly as written (`1830-1900`), never reformatted.
- **State and phase:** `gcalEventsTimeState` / `gcalEventsBufferPhase` (after the fix),
  with `isToday = isTodayDailyFile(app, renderedFile)` and the row's buffer. Not today
  means no state, the same as the Events pill.
- **Hue:** `calendarEventMeetHue()`, only while running, as on the Events pill.
- **Clock:** `gcalEventsApplyClockToRow(row, line, path, metadataCache, vault)`. `#on`
  wins; then `#off` and its aliases; then the inherited area/project `offclock:` through
  `[root::]`. An off-clock row sets `data-off-clock="true"` (dashed).
- **Wings:** `gcalEventsWingModel` for each side with a buffer, built with
  `buildCalendarEventBufferWing`.
- **Source lines.** Live Preview reads the live editor document, so an unsaved edit to
  the Event row shows on the same frame. Reading view tries, in order,
  `ctx.getSectionInfo(el).text`, `calendarEventSourceLines(ctx)`, and
  `taskThemeSourceTextFor(sourcePath)`. When the last one is empty, request a fill and
  refresh once it lands, with no pill meanwhile. Never read from disk during render.

### Anatomy and look

- Host:
  `span.bob-event-link-time.bob-calendar-event-time-wrap[role=img][aria-label][data-tooltip-position=top][data-block-id]`.
  It contains an optional before-wing `span.bob-calendar-event-buffer`, then
  `span.bob-calendar-event-time` (text `HHMM-HHMM` and the `gcalEventsTimeAttributes`
  attributes), then an optional after-wing. No element inside the host carries a `title`
  attribute: drop the `title` that `gcalEventsTimeAttributes` and the wing models
  produce, because the host's `aria-label` is the tooltip. The host has no listeners in
  its DOM builder (`buildEventLinkTimeElement(doc, model)`), like
  `buildFreesMarkElement`.
- Host-only CSS, not scoped under `.bob-calendar-events`: `user-select: none` (copying a
  row never copies the time), `cursor: default`, `print-color-adjust: exact`, and the
  placement margin below. The pill never changes line height and takes no document
  positions.
- Gates:
  `body:not(.bob-event-link-times) .bob-event-link-time, body:not(.bob-calendar-events) .bob-event-link-time { display: none; }`.
  The JS checks in the builder and the sweep stay the source of truth.

### Placement

> [!decision] placement = before
>
> - **Live Preview:** a zero-width `Decoration.widget` at the link start (the opening
>   `~~` of a struck link, else `[[`, after any `🍅 `) with `side: -3`. A row that also
>   has the ledger `#TYPE` mark (`-2`) and ◐ (`-1`) then reads `pill #TYPE ◐ link`.
> - **Reading view:** insert the host before the link container (the anchor, or its
>   `del`/`s` wrapper when the anchor is that wrapper's only child). Then move it before
>   any contiguous run of `bob-ledger-type-mark` / `bob-progress-mark` siblings that
>   directly precedes the container. Marks that later insert directly before the
>   container land between the pill and the link, so any processor order converges.
> - Host margin: `margin-inline-end: 0.35em`.

> [!decision] placement = after
>
> - **Live Preview:** a zero-width `Decoration.widget` at the link end (after `]]`, or
>   after the closing `~~`, and before any ` #`) with `side: 0.5`, so the row reads
>   link, pill, 🔓 (`1`), 🔑 (`2`), off chip (`3`).
> - **Reading view:** insert the host directly after the link container. Teach the 🔓
>   and 🔑 Reading insertion helpers (and gf's chip helper, if it has landed) to treat a
>   `bob-event-link-time` host directly after the container as part of the link and
>   insert after it, so any processor order converges to link, pill, 🔓, 🔑, off.
> - Host margin: `margin-inline-start: 0.35em`.

In both branches, add `bob-event-link-time` to `ledgerMarkClass` in
`progressMarkReadingContext`, so ◐, 🔓, 🔑, the ring's ledger-row exclusion, and gf's
chip still recognize a row that wears a pill. Grep for every other Reading-view consumer
that inspects ledger row children (the `bob-frees-mark` / `bob-unblocked-glyph`
occurrences map them) and make each one ignore the host too.

### Tooltip (`aria-label`, Obsidian tooltip, never contains `::`)

- Line 1: the Event row's cleaned `title`, cut to 80 characters (79 plus `…`), or
  `Calendar event` when the title is empty.
- Line 2, only when non-empty: the Events pill's own tooltip for the same row (the
  `title` that `gcalEventsTimeAttributes(row, hue)` computes: buffer `Holds …` text and
  `gcalEventsClockTooltip`).
- Any stray `::` becomes `:`.

### Interaction

Copy 🔑's handlers. A plain mousedown in Live Preview places the caret at the widget
position and focuses the editor. Mod+click (primary button) opens the link like
Mod-clicking it: `workspace.openLinkText(linktext, sourcePath, "tab")`. A plain click
never navigates and never writes. No tab stop, no live-region announcement, and no
animation (the Events pill has none either).

### Liveness

- Live Preview rebuilds on `docChanged`, `viewportChanged`, the Live Preview ↔ source
  switch, a file switch, its own refresh effect, and `gcalEventsRefreshEffect`. It never
  rebuilds on selection alone.
- **Clock:** `refreshCalendarEventPaint()` also refreshes link pills: Live Preview
  through the same effect, and Reading view by sweeping link-pill records for today's
  note. That keeps the Events pill's once-a-minute cadence with no new timer.
- One coalesced `scheduleEventLinkTimesRefresh()` (about 120 ms) runs on `metadataCache`
  `changed` / `resolved` (an `offclock:`, `type:`, or `parent:` edit changes
  inheritance), vault `rename` / `delete`, a source-text cache fill, and both toggles.
- The Reading refresh sweeps rendered daily notes in every window (main and popouts). It
  adds, updates, or removes hosts in place, and reconciles detached sections quietly, as
  🔑 does.

### Toggles

- New command `toggle-event-link-times`, "Toggle Task Link event times": session-only,
  default on, `applyMarkGateClass("bob-event-link-times", on)` plus
  `registerMarkGatePopoutSync`. No api namespace.
- "Toggle calendar event painting" also refreshes link pills. Route its
  `bob-calendar-events` class through `applyMarkGateClass` plus
  `registerMarkGatePopoutSync`, so popout windows follow it too. Today
  `setCalendarEventPaintGate()` sets it only on the main window's body, so a popout can
  miss it or miss a later toggle.

### Performance

Scan Event rows once per document version (cache them like `calendarEventScanForView`)
and map them by block id. Resolve each unique linkpath once per build. Add widgets only
inside visible ranges. Never read from disk during render. A node perf test with a
synthetic 200-link ledger and 50 Event rows keeps one Live Preview build under about 25
ms.

### Deferred (record in the contract with triggers)

- Links to another daily note's Event Task. Trigger: Bryan asks. The Mac's chips are
  same-note too.
- Prose links (Events child bullets, `## Notes`, other notes). Trigger: Bryan asks.
- Embedded `![[…]]` ledger rows.
- Showing the event title in place of the `^id` link text (task-link-marks already
  defers readable titles).

## Conformance vectors (the contract copies these; JS tests run them verbatim)

Fixture unless a row says otherwise: today is 2026-11-03 and now is 10:00. The daily
note is `2026/20261103.md`, and its `## Events` holds:

```text
- [x] #task **0815-0830** Swings! #off [root:: abby] [type:: MEET] ^swings
- [ ] #task **0945-1015** Standup [root:: bob] [type:: MEET] ^standup
- [*] #task **1830-1900** (virtual) CFG Appt #off [buffer:: -30m] [root:: body] [type:: MEET] ^cfg
- [ ] **1200-1230** Park [root:: abby] ^park
- **all day** Holiday ^holiday
```

`love.md` is an area with `offclock: true`. `abby.md` is an area with
`parent: "[[love]]"`. `bob.md` is a project holding
`- [ ] #task Fix the parser ^parser`. The note also holds `^gtd` above `## Events`.

**EL — which links.**

| #    | Ledger (under `## Pomodoros`)                                       | Pill?                                   |
| ---- | ------------------------------------------------------------------- | --------------------------------------- |
| EL1  | `- [ ] () — MEET` / `\t- [[#^cfg]]`                                 | yes: `1830-1900`, before wing `30m`     |
| EL2  | `- [x] (**0815-0830** [off:: 15m]) — MEET` / `\t- ~~[[#^swings]]~~` | yes (closed entry, struck)              |
| EL3  | open entry / `\t- 🍅 [[#^standup]]`                                 | yes                                     |
| EL4  | open entry / `\t- [[#^standup]] #`                                  | yes                                     |
| EL5  | open entry / `\t- [[#^standup\|Standup]]`                           | yes                                     |
| EL6  | open entry / `\t- [[2026/20261103#^standup]]`                       | yes (same note by path)                 |
| EL7  | open entry / `\t- [[#^park]]`                                       | yes (hand-written row without `#task`)  |
| EL8  | open entry / `\t- ![[#^standup]]`                                   | no (embed)                              |
| EL9  | open entry / `\t- join [[#^standup]] early`                         | no (mixed prose)                        |
| EL10 | open entry / `\t\t- [[#^standup]]` under another child              | no (deeper bullet)                      |
| EL11 | open entry / `\t- [[bob#^parser]]`                                  | no (not an Event Task)                  |
| EL12 | open entry / `\t- [[#^gtd]]`                                        | no (same-note task outside `## Events`) |
| EL13 | open entry / `\t- [[#^holiday]]`                                    | no (all-day context row)                |
| EL14 | open entry / `\t- [[#^dup]]` where two Event rows carry `^dup`      | no (ambiguous)                          |
| EL15 | open entry / `\t- [[2026/20261102#^swings]]`                        | no (another day's Event Task; deferred) |
| EL16 | `2026/20261102.md` ledger linking its own `[[#^swings]]`            | yes, with no `data-state`               |
| EL17 | `## Events` child bullet or `## Notes` bullet `see [[#^standup]]`   | no (prose; deferred)                    |
| EL18 | EL1 inside a fenced block                                           | no                                      |
| EL19 | a non-daily note with its own `## Events` and `## Pomodoros`        | no                                      |
| EL20 | event-times toggle off, or calendar painting off                    | no (CSS gate and JS)                    |

**ES — state** (the shared `gcalEventsTimeState` / `gcalEventsBufferPhase`, so the
Events pill gets the same answers).

| #    | Span, buffer, now                          | Result                                          |
| ---- | ------------------------------------------ | ----------------------------------------------- |
| ES1  | `1830-1900` at 10:00                       | `future` (today's code says `past`: the bug)    |
| ES2  | `1200-1230` at 10:00                       | `future`                                        |
| ES3  | `0945-1015` at 10:00                       | `running`; `data-theme-hue` is the MEET hue     |
| ES4  | `0815-0830` at 10:00                       | `past`                                          |
| ES5  | `2300-0030` at 23:45                       | `running`                                       |
| ES6  | `2300-0030` at 00:10                       | `running` (a midnight-crossing span shifts now) |
| ES7  | `2300-0030` at 22:00                       | `future`                                        |
| ES8  | `1830-1900`, `-30m`, at 18:10              | `future`, phase `lead-in`                       |
| ES9  | `1200-1300`, `-30m +15m`, at 13:05 / 13:20 | no state, phase `lead-out` / `past`             |
| ES10 | any span, in a note that is not today's    | no state                                        |

The existing `scripts/test-ledger-tools-gcal-events.cjs` state assertions keep passing.

**ER — resolution and parity.**

| #   | Case                                                         | Result                                                          |
| --- | ------------------------------------------------------------ | --------------------------------------------------------------- |
| ER1 | EL1                                                          | `data-buffer-before="30"`, before wing `30m`, `data-off-clock`  |
| ER2 | EL7 (`^park`, `[root:: abby]` inherits love's `offclock`)    | `data-off-clock="true"`                                         |
| ER3 | EL3 (`^standup`, `[root:: bob]`)                             | no `data-off-clock`                                             |
| ER4 | `^park` edited to carry `#on`                                | no `data-off-clock`                                             |
| ER5 | Live Preview, `^standup` span edited to `1000-1030`, unsaved | pill reads `1000-1030` on the same frame                        |
| ER6 | Live Preview, `#off` added to `^standup`, unsaved            | dashed on the same frame                                        |
| ER7 | every pill row                                               | inner attributes equal `gcalEventsTimeAttributes` minus `title` |
| ER8 | the Event row is `[x]` or `[-]`                              | pill still shows (as on the Events row)                         |

**ET — tooltip.**

| #   | Case            | aria-label                                           |
| --- | --------------- | ---------------------------------------------------- |
| ET1 | EL1             | `(virtual) CFG Appt\nHolds 18:00–19:00 · 30m before` |
| ET2 | EL2             | `Swings!`                                            |
| ET3 | EL7             | `Park\nOff-clock · from love`                        |
| ET4 | a 95-char title | first 79 characters plus `…`                         |
| ET5 | a stray `::`    | becomes `:`                                          |

**EP — placement.**

| #   | Case                                      | Result                                                         |
| --- | ----------------------------------------- | -------------------------------------------------------------- |
| EP1 | Live Preview                              | widget at the decided end and `side`; no line-height change    |
| EP2 | a running `[/]` event under an open entry | `before`: `pill ◐ link`; `after`: `◐ link pill`                |
| EP3 | gf's chip present on the same row         | `before`: `pill link off`; `after`: `link pill off`            |
| EP4 | cursor on the row (raw `[[…]]` revealed)  | pill stays at the same end                                     |
| EP5 | a pill-wearing row in Reading view        | ◐ / 🔓 / 🔑 still paint; no status ring appears                |
| EP6 | Event pill and link pill, computed styles | same background, ink, radius, padding, outline style per state |
| EP7 | struck link                               | the strike never runs through the pill                         |

## Phase `contract` — authoritative display contract (bob-cli)

Write `docs/event-link-times.md` in the style of `docs/frees-marks.md` and
`docs/task-link-marks.md`. Sections: What you see (the before/after block, drawn for the
decided placement); Principles; Which links; Resolution (including the source-line
order); Anatomy and look; Placement; Tooltip; Interaction; Liveness; Toggles;
Performance; Deferred; Conformance vectors (EL, ES, ER, ET, EP, copied from this plan,
with each placement-dependent row resolved to the decided branch). Write the decided
placement as plain contract text, with no decision callouts in the published page. In
ES, state that the shared helper fix also corrects the Events pill. Name the JavaScript
mirror files that phase `pill` adds: core `src/291d-event-link-times.js`, mixins
`src/291e-plugin-event-link-times.js` and `src/291f-plugin-event-link-times-reading.js`,
and test `scripts/test-ledger-tools-event-link-times.cjs`. State that there is no api
namespace.

Register the page:

- `docs/_meta/contracts.yml`, "Tasks and projects" group, right after the last
  ledger-mark entry (`off-clock-link-chips` if gf has registered it, else
  `frees-marks`), with the next free `order` and a summary like "Event link times: the
  Event Task's time pill beside ledger Task Links that point at it";
- both `mkdocs.yml` nav lists, at the same spot;
- `tools/docs/tests/test_contracts.py` and `docs/_meta/url_contract.txt` only if the
  docs gate requires them, as the frees-marks contract commit (`b508f0b6`) did.

Add two cross-links, one sentence each and nothing else in those pages:

- `docs/gcal.md` → `## Events` (after the "Who owns each part of a row" table): in
  Obsidian, each MEET Task Link to an Event Task wears that event's time pill
  ([Event link times](event-link-times.md)).
- `docs/task-link-marks.md` → rule 5 (ledger rows): ledger rows that link one of the
  note's Event Tasks wear that event's time pill instead of a ring.

Do not edit guides, memory, `Cargo.toml`, or `CHANGELOG.md`. Run `just docs-gen` if the
registry change regenerates indexes (`docs/README.md`, `docs/develop/contracts.md`),
then `just docs-check` until it passes. Suggested commit subject:
`docs(event-link-times): add ledger Event time pill display contract`.

## Phase `pill` — the pill in bob-ledger-tools (bob-plugins)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Implement bob-cli
`docs/event-link-times.md` exactly. The contract wins; if it is wrong, fix it in bob-cli
in this phase. Edit only `src/` fragments, then `npm run build` (never hand-edit
`main.js`). Keep every hand-edited fragment at or below 1000 lines. Read "Coordination
with epic `bob-cli-gf`" above first.

1. **Fix the shared clock** (`src/291-plugin-gcal-events.js`). Give
   `gcalEventsNowOnTimeline` the occupied end and shift `now` only for a
   midnight-crossing span (see "A bug in the pill being echoed"). Add ES1–ES10 to
   `scripts/test-ledger-tools-gcal-events.cjs`. Correct any sentence in bob-plugins
   `docs/bob-ledger-tools.md` → "Calendar Events" that the fix makes untrue. This fix
   may be its own commit:
   `fix(gcal-events): keep afternoon Event pills future until they start`.
2. **Pure core** `src/291d-event-link-times.js`, listed right after
   `291c-plugin-gcal-event-clock.js` in `fragments.json`. It is synchronous and never
   throws, in the style of `141b-frees-marks.js`.
   - `eventLinkTimeLinks(lines, path, options)` returns a frozen
     `[{line, ch, pathPart, blockId, linktext}]` for every row
     `taskLinkMarkIsLedgerTaskLinkLine` accepts (honouring fences, frontmatter, and
     comments through the ring's document context). `ch` is the decided end: the link
     start (opening `~~`, else `[[`) for `before`, or the link end (after `]]` or the
     closing `~~`, before ` #`) for `after`. `options.isSameNote(pathPart)` decides rule
     4, and `options.isVisible(line, ch)` limits a Live Preview build.
   - `eventLinkTimeModel(eventRow, {isToday, nowMinutes, hue, line, path, metadataCache, vault})`
     returns `{time, row, wings, tooltip}`, or null unless the row is a unique Event
     Task. `eventLinkTimeTooltip(model)` follows the ET table.
   - `buildEventLinkTimeElement(doc, model)` (listener-free, no inner `title`) and
     `EventLinkTimeWidget` (`eq` on time, state, phase, hue, off-clock, buffer, and
     tooltip; handlers copied from 🔑).
3. **Live Preview** in `src/291e-plugin-event-link-times.js`
   (`BobLedgerToolsEventLinkTimesMixin`, installed in `src/310-install-methods.js` after
   the gcal mixins). It holds the setup (called from `setupGcalEvents()`), the toggle
   command, a ViewPlugin with the rebuild rules above,
   `buildEventLinkTimeDecorations(view)` (live document lines, one cached Event-row
   scan, per-build linkpath memo, widgets at the decided `side` in visible ranges),
   `scheduleEventLinkTimesRefresh()`, editor refresh through a `StateEffect` plus
   `gcalEventsRefreshEffect`, and teardown wired beside
   `teardownCalendarEventPainting()`.
4. **Reading view** in `src/291f-plugin-event-link-times-reading.js`: a markdown
   post-processor that records each rendered root and sweeps it, plus a refresh sweep
   over every rendered daily note in every window. For each `li`, call
   `progressMarkReadingContext(li, root, { includeStruck: true, includeClosedEntry: true, allowLedgerMarks: true })`
   and resolve the target against the note's lines in the Resolution order. Add, update,
   or remove hosts in place per Placement, and remove orphaned hosts. Add
   `bob-event-link-time` to `ledgerMarkClass` and to every other ledger-row consumer
   found by the grep in Placement.
5. **Gates and liveness wiring.** Make `refreshCalendarEventPaint()` and
   `toggleCalendarEventPainting()` refresh link pills. Route `bob-calendar-events`
   through `applyMarkGateClass` and `registerMarkGatePopoutSync`. Call
   `scheduleEventLinkTimesRefresh()` from the source-text fill in
   `taskThemeRequestSourceText` and from the mixin's own `metadataCache` and vault
   listeners.
6. **CSS** (`plugins/bob-ledger-tools/styles.css`, after the calendar wing rules): the
   host-only rule, the decided margin, and the two gates. Add no pill look rules; the
   existing `.bob-calendar-events .bob-calendar-event-time` family must draw the inner
   pill unchanged (EP6).
7. **Placement neighbours** (only `placement = after`). Update the 🔓, 🔑, and (if
   present) off-chip Reading insertion helpers so they insert after a
   `bob-event-link-time` host that directly follows the link container.
8. **Exports and tests.** Export the pure helpers through `src/350-exports.js`. Add
   `scripts/test-ledger-tools-event-link-times.cjs` (register it in `package.json`
   `test`) running EL, ES, ER, ET, and EP1–EP5 and EP7 on a fake DOM, as the frees tests
   do. Also cover the toggle and calendar-painting gates, the popout gate sync, the
   "refresh calendar paint also refreshes link pills" wiring, Mac parity (EL11–EL15
   match `event_item_chips`), and the perf bound. Keep the gcal-events, frees,
   unblocked, progress, ledger-type, and task-link-marks tests green, plus gf's chip
   tests if present.
9. **Docs.** In bob-plugins `docs/bob-ledger-tools.md`, add one bullet in "Marks and
   rendered blocks" linking the contract
   (`https://bobs-org.github.io/event-link-times/`), one sentence in "Calendar Events"
   saying ledger Task Links to the note's Event Tasks wear the same pill, and a "Session
   toggles" paragraph for the new command. Add one clause to the
   `plugins/bob-ledger-tools/manifest.json` description and bump its minor version past
   whatever master holds. Then run `npm run docs:gen`.
10. **Gates.** Run `npm run build`, `npm run build:check`, `npm test`,
    `npm run validate`, and `npm run docs:check`. Compare any `npm test` failure with a
    baseline run on the unmodified tree. Report pre-existing failures (for example the
    open `bob-cli-eh` / `bob-cli-et` beads) instead of fixing unrelated code. Then run
    `bob plugins sync`. Suggested commit subject:
    `feat(ledger-tools): show Event time pills beside ledger Task Links`.

## Phase `live_verify` — real-Obsidian e2e and visual review (bob-plugins, bob-cli)

Adapt `scripts/obsidian-frees-marks-e2e.cjs` and its lib (isolated scratch vault and
profile, an owned Obsidian process, CDP probes with timeouts; never the live vault) into
`scripts/obsidian-event-link-times-e2e.cjs` and
`scripts/obsidian-event-link-times-e2e-lib.cjs`. Add an offline self-test,
`scripts/test-obsidian-event-link-times-e2e.cjs`, registered in `npm test` like the
frees one. Run it under `xvfb-run -a`. Bound every screenshot and CDP call with a
timeout (headless Chromium screenshot hangs on apollo are tracked as `bob-cli-fz`).

The scratch vault holds `love.md` (area, `offclock: true`), `abby.md` (area under love),
`bob.md` (project with one ordinary task), and today's and yesterday's daily notes.
Their Event spans are computed from the harness clock so states are deterministic: one
past `#off` event, one running `[/]` event, one future event with `[buffer:: -30m]`, one
hand-written untagged event with `[root:: abby]`, and one all-day context row. Their
`## Pomodoros` sections cover the EL rows (open and closed MEET entries, struck, `🍅`,
` #`, aliased, full-path, embed, mixed prose, deeper bullet, and an ordinary task link).
If the run falls within four hours of midnight, shift the fixture so no span crosses it.
Verify, in Live Preview and Reading view:

- pills exactly per EL/ER, with ET tooltips (a CDP hover whose visible `.tooltip`
  matches the `aria-label`);
- EP2 and EP5 on the running `[/]` row, and EP3 if gf's chip is present;
- EP6: the link pill's computed background, color, border radius, padding, and outline
  style and color match the Events row pill for the same event, in each state;
- the clock: an event edited in Live Preview so its span straddles now flips to
  `running` on the same frame, and `refreshCalendarEventPaint()` repaints link pills in
  both views;
- ES1 live: an afternoon future event's Events pill and link pill both read `future`;
- liveness: setting `abby.md`'s parent to a note without `offclock: true` removes the
  dash within about one second, without reopening the daily note;
- both toggles hide and restore pills in both views and in a popout. A popout's pill has
  the same computed look as the main window's. Row height is unchanged with the gates on
  and off.

Save light and dark screenshots of the ledger in both views and review them for line
height, spacing, alignment, and the strike. If Obsidian or `xvfb-run` is unavailable on
the host, run the offline self-test, record the live run as skipped with the reason, and
say so in the final report.

Then, in bob-cli, add a "Live verification" section to `docs/event-link-times.md` in the
style of `docs/frees-marks.md`'s: what ran, where, measured refresh timings, and
screenshot notes. Run `just docs-check`. Run `bob plugins sync` after any bob-plugins
change. Suggested commit subjects: `test(event-link-times): add isolated Obsidian e2e`
in bob-plugins and `docs(event-link-times): record live verification` in bob-cli.
