---
tier: epic
title: Pre/post-event buffers for calendar events
goal:
  "A Google Calendar description line like `@love #untracked -30 +15` gives an Event
  Task a beautifully rendered buffer that every occupied-time surface honors."
phases:
  - id: buffer-grammar
    title:
      "bob-cli: description directive lines, the [buffer::] field, and the shared
      conflict predicate"
    depends_on: []
    size: medium
    description:
      "buffer-grammar: HTML-safe combinable directive lines in gcal/model.rs,
      EventBuffer and buffered spans plus fixpoint title cleaning and the conflict
      predicate in event_task.rs, and the shared event_buffer vectors."
  - id: gcal-sync-buffers
    title: "bob-cli: bob gcal sync renders buffers and files MEETs on buffered spans"
    depends_on:
      - buffer-grammar
    size: medium
    description:
      "gcal-sync-buffers: render [buffer::] in the Google-owned badge run, keep that run
      intact on reroutes, use buffered spans for sync phases and MEET clusters, show
      buffers in list/sync output, and document them in docs/gcal.md."
  - id: capture-buffers
    title: "bob-cli: MEET event runs and run-into-event warnings honor buffers"
    depends_on:
      - buffer-grammar
    size: medium
    description:
      "capture-buffers: buffered MEET event runs and far-run text, conflict
      detection/overlap/fit/messages via the shared predicate, additive buffer JSON,
      meet_event_run and event_task_close vector cases, and docs/capture.md."
  - id: agenda-buffers
    title: "bob-cli: capture-pomodoros --tasks events block carries buffers"
    depends_on:
      - buffer-grammar
    size: medium
    description:
      "agenda-buffers: per-row and chip buffer objects, buffer-aware
      gap/conflict/start_hint/refresh_at, the human Events block label, and a new
      agenda-events-buffered golden."
  - id: plugins-buffers
    title: "bob-plugins: hatched buffer wings and a buffer-aware JS mirror"
    depends_on:
      - buffer-grammar
      - capture-buffers
    size: medium
    description:
      "plugins-buffers: copy and pin the vectors, make 035-meet-event-run.js
      buffer-aware, paint hatched lead-in/lead-out wings on Event time pills in Live
      Preview and Reading view, update docs, and deploy with bob plugins sync."
  - id: mac-buffers
    title: "bob-mac-capture: buffer wings on the Calendar rail and buffer-aware cards"
    depends_on:
      - capture-buffers
      - agenda-buffers
    size: medium
    description:
      "mac-buffers: tolerant buffer decoding, hatched rail wings and buffer-aware
      firstGap, a row buffer capsule with accessibility text, buffer-aware event-run and
      conflict cards, fixtures, renders, and docs."
decisions:
  memory_buffer_record:
    ask:
      Write a buffers decision record and mark synced-meetings-link-by-phase superseded
      in part?
    default: true
    memory:
      - decisions:synced-meetings-link-by-phase
      - decisions
    corrects:
      Back-to-back or overlapping past meetings share one closed MEET whose span is the
      cluster (next start ≤ cluster end).
    answer: true
  memory_event_task_glossary:
    ask: Add one sentence about the [buffer::] field to the Event Task glossary strand?
    default: false
    memory:
      - glossary:event-task
    answer: false
proposed_by: bbugyi200.apollo.c5
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-01 21:42:29
status: wip
---

- **PROMPT:**
  [prompts/202611/gcal_event_buffers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_event_buffers.md)

# Pre/post-event buffers for calendar events

## Goal

Let Bryan write `-30` and/or `+15` in a Google Calendar event's description to reserve
30 minutes before and 15 minutes after the event (travel, prep, wind-down). Bob then
treats the event as occupying `start − 30 … end + 15` everywhere a time range means "I
am busy". That covers MEET starts from `=`, run-into-event warnings, free gaps, sync's
past/current/future phases, and the MEET Pomodoro spans it writes. The meeting itself
keeps its real time. Buffers render beautifully on the Event Task, in Obsidian, in
`bob gcal list`, and in Bob Mac Capture's Calendar band. A single description line such
as `@love #untracked -30` must work.

Context: epic `bob-cli-f6` (`plan:202611/untracked_calendar_events.md`) built
`#untracked`, the MEET event run, run-into-event warnings, the agenda `events` block,
and the Calendar band. This epic extends each of them.

## The design in one page

**Two spans, one rule.** Every Event Task has its **meeting span**, the bold
`**1200-1300**`, which is never rewritten. A buffer adds a **buffered span**:
`max(00:00, start − before)` to `end + after`. The rule is:

> The buffered span is what the event _occupies_. The meeting span is when the meeting
> _happens_.

| Uses the buffered span                                                                 | Keeps the meeting span                                                                     |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| MEET event run (`=` default range, early start, far-run warning, run clustering)       | Timed close classification (`ended_inside`/`running`/…, shared `event_task_close` vectors) |
| Run-into-event warnings and fit drafts (capture, Obsidian Notice)                      | Agenda row `phase` (`ended`/`running`/`upcoming`) and `focus`                              |
| `bob gcal sync` phases (past/current/future) and the MEET clusters and spans it writes | The bold span text, its journal hash, and Google ownership of the time                     |
| Agenda `gap`, `conflict`, `start_hint`, `refresh_at`                                   | Row titles, `#untracked`, roots                                                            |

The meeting clock stays bare on close for this reason: a MEET closed at 13:00, when
lunch ends at 13:00 and its `+15` drive back runs to 13:15, must still complete lunch.
Widening close would carry a finished meeting through its travel time.

A buffer inherits its event's trackedness. With `#untracked -30`, the lead-in is off the
clock too. A tracked meeting's buffer is meeting time attributed to its `[root::]`.

**Typed the way you think.** In the Google description, a **directive line** is any line
whose every token (split on whitespace or commas) is a directive. A line with any other
token is ordinary text and is ignored whole.

| Token                      | Meaning                                    |
| -------------------------- | ------------------------------------------ |
| `@route`                   | Root route (existing; the first one wins)  |
| `#untracked`               | Off the clock (existing; case-insensitive) |
| `-N`, `-Nm` (also `−`/`–`) | Pre-event buffer of N minutes              |
| `+M`, `+Mm`                | Post-event buffer of M minutes             |

So all of these work: `@love #untracked -30`, `-30 +15`, and `@love, -30` on one line;
or the directives spread over several lines. N and M have 1–3 digits; each side is
capped at 720 (12h); the first value for each side wins (the same reading order as
`@route`). Buffers are read from the description only. Numbers in titles collide with
real titles (`Q4 +1`, `Room -2`), and the user asked for descriptions.

Descriptions are HTML when written in Google's web editor. Before reading directives,
Bob normalizes:

- `<br>`, `<p>`, `<div>`, and `<li>` boundaries become line breaks;
- other tags are stripped;
- common entities (`&nbsp;`, `&amp;`, `&lt;`, `&gt;`, `&quot;`, `&#39;`, numeric) are
  decoded.

Today `@love<br>#untracked` is silently missed; this fixes that for every directive. A
stray `+1` dial-in line never qualifies: phone lines carry other tokens.

**Spelled the same everywhere it is text.** Bob writes a Google-owned inline field at
the end of the badge run, after `#untracked` and before `[root::]`:

```markdown
- [*] #task **1200-1300** Lunch with Kelly [📅](… "Open in Google Calendar") #untracked
  [buffer:: -30m +15m] [root:: love] [type:: MEET] ^lunch-with-kelly
```

A side that is zero is omitted (`[buffer:: -30m]`, `[buffer:: +15m]`), and an event
without buffers has no field. Hand-written Event Tasks may carry the field too; capture
reads it from the vault, never from Google. Every human surface spells a buffer the way
it was typed, with a typographic minus:

- `bob gcal list`: `12:00–13:00 −30m +15m`
- warnings: `Lunch (1200-1300 −30m +15m)`
- the agenda's Events block
- the Mac row capsule

Why a field and not glyph tokens:

- `escape_title` already neutralizes `::`, so a title can never forge one.
- Rust (`inline_fields`) and JS (`pomodoroThemeInlineFields`) already scan fields.
- The title cleaners already strip trailing fields.
- Dataview can query it.

Rejected alternatives:

- **Rewriting the bold span to the buffered span.** It loses the meeting time and breaks
  span parsers in Rust, JS, and Swift.
- **`⇤30m`-style glyph tokens.** They are forgeable from titles and need new escapes and
  strippers in two languages.
- **Two fields `[before::]` `[after::]`.** They are noisier and split one fact in two.

**Drawn the same everywhere it is drawn: hatched wings.** A buffer is drawn as lighter,
diagonally hatched "wings" fused to the event's time shape: the Obsidian time pill, and
the Mac rail capsule. This is the established visual language for travel time. Wings use
the event's home hue. They are dashed when the event is untracked, matching f6's "dashed
means off the clock". A wing lights up with the hue outline while its buffer is in
progress (lead-in or lead-out).

**Google-owned like the badges.** The field lives in the badge span. Its ownership:

- It is updated in place while that span is untouched.
- A vault edit wins and is reported once (`kept_edit`).
- Rows synced before this feature migrate on the next sync, as `#untracked` did.
- Context rows (Maybe/free) carry no buffer.
- A Maybe row upgraded to a task renders it.

While here, fix a latent ordering bug. A root reroute calls `set_task_root`, whose
`trailing_metadata_start` walks back over `#untracked`. That inserts `[root::]` before
the tag and splits the Google run, so the next sync falsely reports a badges edit. Gcal
row writes must never insert metadata into or before the Google run.

**Warnings follow the occupied time.** A non-MEET session's new span `[S, E)`
_conflicts_ with an open Event Task when two things hold. First, it overlaps the event's
buffered span. Second, the meeting is not already in progress at `S`
(`start < S < end`): starting mid-meeting stays quiet, as today, because a meeting you
left early stays open after a timed close. It warns when the new span conflicts and the
old one did not (`conflicts(new) && !conflicts(old)`). For events without a buffer this
is exactly today's "start newly crossed" rule, so every existing test and message stays
byte-identical. With buffers it covers four cases:

- running into a lead-in;
- starting inside a lead-in and running into the meeting;
- starting inside a lead-out;
- running from before the event through all of it.

The fit draft ends at the buffered start, and it is offered only when `S` is before the
buffered start. Messages:

```text
⚠ BOB 1110-1135 runs 5m into Lunch's −30m buffer (1200-1300) · =4 ends at 1130
⚠ BOB 1110-1215 runs 45m into Lunch (1200-1300 −30m) · =4 ends at 1130
⚠ BOB 1305-1330 runs 10m into Lunch's +15m buffer (1200-1300)
⚠ BOB 1150-1215 runs 15m into Standup (1200-1230) · =2 ends at 1200     (no buffer: unchanged)
```

**MEET follows its buffered run.** In the event run, three things use the buffered span:
candidates (buffered end > now), sort, and clustering (`next.buffered_start ≤ run.end`,
same trackedness). A `+15` after Standup 1000-1030 therefore bridges to a 1045 1:1, and
the run is 1000-1115. `=` at 11:25 for lunch with `-30 +15` writes `(**1130-1315**)`.
The late-join clamp and the early-start rule are unchanged; the far-run warning measures
from the buffered start.

## Shared definitions (all phases)

- **EventBuffer** (`src/native/event_task.rs`): `{before_min: u16, after_min: u16}`,
  each `0..=720`.
  - Parse from the vault field `[buffer:: …]`: tokens are whitespace-separated, each
    `-N`/`−N`/`–N` or `+M`, with an optional `m`. Only the first `[buffer::]` field
    outside code spans and Markdown links counts. An invalid value means no buffer.
  - Render the canonical field as `[buffer:: -30m +15m]`, with the before side first and
    zero sides omitted.
  - Render the human label as `−30m +15m` (U+2212).
- **Buffered span**: on the day's timeline, `start = max(0, start_min − before)` and
  `end = end_on_timeline + after`. The total is clamped below 24h, so `HHMM-HHMM` stays
  unambiguous. Its text is `HHMM-HHMM`, mod 1440, like other timeline spans.
- **Directive line** (Google description, after HTML normalization): see the table
  above. It yields `route`, `untracked`, `before`, and `after`, and is the only
  description reader. `title_route_hint` and the title `#untracked` strip are unchanged.
- **Conflict predicate** and **newly** rule: as above. It is one pure helper in
  `event_task.rs` that returns, earliest buffered start first:
  - the event;
  - `overlap_minutes` (new span ∩ buffered span, with overnight wrap as in today's
    helpers);
  - `buffer_part` (`before` or `after` when the overlap lies wholly inside one buffer,
    else none);
  - the fit target (the buffered start, when `S` precedes it).

  Capture, the agenda, and the JS mirror all use it.

- **JSON `buffer` object.** In capture and agenda payloads it is
  `{"before_minutes", "after_minutes", "time_range", "starts_at"?, "ends_at"?}` and is
  omitted when there is no buffer. `time_range`/`starts_at`/`ends_at` are the buffered
  span; `starts_at`/`ends_at` appear only in agenda payloads. gcal `list` and `sync`
  JSON always carry the key: an object `{"before_minutes", "after_minutes"}`, or `null`,
  which matches gcal's null style. Every change is additive and schema versions stay 1.
- **Shared vectors** `tests/fixtures/event_buffer/vectors.json` plus a `README.md` with
  the pinned commit. They have three sections:
  - `description`: raw (incl. HTML) → route/untracked/before/after;
  - `row`: vault line → buffer, buffered span, clean title;
  - `conflict`: session new/old span plus a day's Events → crossed block id, overlap,
    `buffer_part`, fit.

  A Rust test consumes them, and bob-plugins copies them byte-for-byte.

## Phases

### buffer-grammar (bob-cli)

1. `src/native/gcal/model.rs`: replace `description_route` and
   `description_has_untracked` with one
   `description_directives(description) -> Directives` reader.
   - It runs HTML normalization, then directive lines (whitespace/comma tokens).
   - The first route and the first value per buffer side win.
   - `GcalEvent` gains `buffer: EventBuffer`.
   - `untracked` keeps its title/description OR.
   - `route_hint` comes from directive lines, so `@love #untracked -30` routes,
     untracks, and buffers.
2. `src/native/event_task.rs`:
   - Add `EventBuffer` with parse/field/label, `buffered_span`, and `buffer` on
     `EventTask`/`EventNoteRow`.
   - Make `clean_event_title` strip trailing fields, `#untracked`, and Markdown links in
     any interleaving until nothing changes, so `[buffer::]` never leaks into a title in
     any order.
   - Add the conflict predicate. Re-express `event_starts_crossed` /
     `first_event_start_crossed` on it (or retire them), keeping today's results for
     unbuffered events.
3. Write the shared vectors, with a Rust test that consumes them. The vectors must
   include:
   - HTML (`<br>`, `<p>`, `&nbsp;`);
   - comma separators;
   - `−`/`–`;
   - `m` suffix;
   - a 4-digit token that kills its line;
   - a phone line;
   - first-wins conflicts;
   - the 720 cap;
   - the midnight clamp;
   - an overnight event;
   - every conflict shape above, plus the unbuffered parity cases.
4. Unit tests for model parsing, including `Lunch @love #untracked` in the title plus
   `-30 +15` in the description.
5. Docs: in `docs/gcal.md` "Untracked events" and "Root selection", replace "a
   description line containing only `@route`/`#untracked`" with directive lines
   (combinable, HTML-safe). The Buffers section lands in `gcal-sync-buffers`.

### gcal-sync-buffers (bob-cli)

1. Render the field.
   - `gcal/render.rs`: `badge_text(..., untracked, buffer)` appends ` #untracked` and
     then ` [buffer:: …]`. Unbuffered rows hash exactly as before, so journals stay
     valid.
   - `render_task_row` writes badges, the Google run, `[root::]`, `[type::]`, then
     `^id`.
   - Context rows never carry the field.
2. Keep the Google run intact.
   - `gcal/lifecycle.rs` `split_row_spans`: the badges span keeps `#untracked` and
     `[buffer::]` with or without a `📅` badge. Generalize
     `split_trailing_untracked_tag` to the trailing Google run.
   - Add a gcal row metadata setter used by render and by the reroute in
     `plan.rs::apply_span_updates`:
     - it replaces an existing `[root::]` in place;
     - otherwise it inserts after the Google run, before `[type::]`/`^id`;
     - `[type:: MEET]` goes after the root;
     - it never moves tokens Bryan placed.
   - Regression test: reroute a `#untracked [buffer::]` row, and the next sync reports
     no `kept_edit`.
3. Phases and MEET spans on the buffered span. `sync.rs` `row_time` / `link_phase_for`
   and `PlannedOccurrence` carry both spans:
   - past means buffered end ≤ now;
   - current means buffered start ≤ now < buffered end (today only);
   - `WantedLink.start_min/end_min`, `cluster_past`, `format_meet_span`,
     `closed_insert_at`, and the created running MEET span use the buffered span;
   - the meeting span still drives the bold time label and its hash.
4. Output.
   - `bob gcal list`: the TIME cell reads `12:00–13:00 −30m +15m` (buffer label dimmed
     on a TTY), and JSON events carry `buffer`.
   - `sync -f json` rows carry `buffer`.
   - Human sync rows show the label; MEET lines show buffered spans.
5. Tests in `tests/gcal_sync.rs` and `tests/gcal_lifecycle.rs`:
   - the description forms;
   - a new row with a buffer;
   - a buffer added or changed in Google on an untouched row (updates in place);
   - a buffer on an edited row (kept, reported once);
   - an unbuffered row's journal hash unchanged;
   - a lead-out run creates a running MEET with the buffered span;
   - a past cluster bridged by a `+15`;
   - a mixed-trackedness split with buffers;
   - a Maybe-to-task upgrade gains the field.
6. Docs:
   - `docs/gcal.md`:
     - a new "Buffers" section (grammar, the two-spans rule table, a rendered row, what
       each surface does);
     - the Sync phase table in buffered terms;
     - the ownership-table row for `[buffer::]`;
     - Known limits: a description you cannot edit means a hand-added `[buffer::]` is
       kept as your edit; a pre-buffer stops at midnight; the first value wins.
   - `docs/reference/vault-format.md`: Events field order and the `[buffer::]` grammar.
   - `docs/reference/json-outputs.md`: the gcal rows.

   > [!decision] memory_buffer_record Write
   > `sase/memory/decisions/event-buffers-widen-the-span.md` in the decisions-web format
   > (Applies to / Claim / Why with the rejected alternatives above / Cost / Reopens
   > when / Evidence citing this plan, `docs/gcal.md`, `docs/capture.md`, and
   > `tests/fixtures/event_buffer/vectors.json`). The claim is the two-spans rule table.
   > Mark [[decisions:synced-meetings-link-by-phase]] `superseded-in-part` with
   > `superseded_by` and a `[[...]]` back-link: its phases and clusters now use the
   > buffered span. Republish with `sase memory init`.

   > [!decision] memory_event_task_glossary Add one sentence to the Event Task glossary
   > strand: `[buffer:: -30m +15m]` widens the span the event occupies (MEET runs,
   > warnings, gaps, sync phases), while timed close keeps the bare span. Republish with
   > `sase memory init`.

### capture-buffers (bob-cli)

1. `src/native/capture/meet_event_run.rs` uses the buffered span for:
   - candidates;
   - sort;
   - clustering;
   - the run span;
   - early start;
   - the far-run warning.

   The far-run message for a pre-buffered first event reads
   `MEET's next event, Lunch (1200-1300 −30m), starts at 1130 (2h 10m from now) · …`.
   Unbuffered text is unchanged.

2. JSON and human output for the run:
   - `event_run.time_range/start/end` are the written buffered span.
   - `event_run.events[]` gain `buffer`.
   - Human output appends the label:
     `from your calendar: Lunch with Kelly (1200-1300 −30m +15m)`.
3. `src/native/capture/event_conflict.rs` uses the shared predicate for detection,
   overlap, and fit:
   - fit ends at the buffered start;
   - messages follow the four forms above;
   - `event_conflict.event` gains `buffer`, and `event_conflict` gains `buffer_part`
     (omitted when none).

   This covers start/restart/swap/link starts, `+N`, and `++N`/`--N`. Existing messages
   for unbuffered events stay byte-identical.

4. Vectors.
   - `tests/fixtures/meet_event_run/vectors.json` gains these cases (and its README pin
     is updated):
     - pre-buffer early start;
     - a `+15` bridging to the next event;
     - a buffer never bridging trackedness;
     - now inside a lead-out;
     - a buffered far run.
   - `tests/fixtures/event_task_close/vectors.json` gains "a buffered event classifies
     by its meeting span".
5. Tests: every start family and adjust/shift with buffers, MEET exemption, own-lineup
   exemption, mid-meeting quiet, lead-out start, overnight, and `--dry-run` parity.
6. Docs in `docs/capture.md`:
   - the lifecycle table cell;
   - "MEET starts follow the calendar" (buffered runs, an example);
   - "Running into an event" (the conflicts rule, the four messages, JSON
     `buffer`/`buffer_part`).

### agenda-buffers (bob-cli)

`bob capture-pomodoros --tasks` events block (`src/native/capture_pomodoros_agenda.rs`):

1. Task rows gain `buffer` (with `starts_at`/`ends_at`). `phase` and `focus` keep the
   meeting span.
2. Hints and boundaries:
   - `gap.until` is the focus's buffered start.
   - `gap.from` also skips past any open Event Task whose buffered span is in progress
     then (a running lead-out).
   - `conflict` uses the shared predicate against the running non-MEET session (only
     when the focus is the earliest conflict) and gains `buffer_part`.
   - `start_hint` is `x =meet` once the focus's buffered start has passed, and `=meet`
     when it is 0–25 minutes away.
   - `refresh_at` includes buffered boundaries.
3. `items[].event` chips gain `buffer`.
4. The human Events block shows the label after the time, and keeps rows visible until
   their buffered end.
5. A new golden `tests/fixtures/capture_pomodoros/agenda-events-buffered.json`
   (`tests/cli/capture/pomodoros_agenda.rs`) covers:
   - an untracked lunch with `-30 +15`;
   - a `+15` standup;
   - a BOB running into a lead-in;
   - a lead-out in progress.

   Existing goldens stay byte-identical.

6. Docs: `docs/capture.md` (`capture-pomodoros`) and `docs/reference/json-outputs.md`.

### plugins-buffers (bob-plugins, via `/sase_repo`)

1. Copy `event_buffer` vectors with their README pin. Re-copy `meet_event_run` and add
   the README pin it lacks. Re-copy `event_task_close`.
2. The `plugins/bob-ledger-tools/src/035-meet-event-run.js` mirror:
   - `eventRunEventRows` reads `[buffer::]` (reuse the inline-field scanner) and exposes
     the buffered span;
   - `eventRunCleanTitle` gets the fixpoint strip;
   - `computeMeetEventRun` uses the buffered spans;
   - `eventRunNewCrossings` uses the conflict predicate;
   - so do overlap, fit, and notice text, for `se<X>`, `\p`, `\o`, and `\O`.

   Tests consume the vectors.

3. Painter. Put new code in a new fragment such as `292-plugin-gcal-event-buffers.js`,
   registered in `src/fragments.json`, so 291 stays under the 1000-line cap.
   - The scan detects the field and gives the time pill `data-buffer-before`,
     `data-buffer-after`, and a tooltip `Holds 11:30–13:15 · 30m before · 15m after`.
   - Live Preview renders wing widgets beside the pill (side -1/+1). Each is a hatched
     segment (a `repeating-linear-gradient` in the event hue at low alpha) labelled
     `30m`/`15m` in UI-smaller tabular figures, with outer corners rounded, flush to the
     pill, and dashed when the row is untracked.
   - Reading view inserts the same spans around the `<strong>`.
   - The raw field is hidden on inactive lines (shown while editing, like the `#` of
     `#untracked`). In Reading view, Dataview's `.dataview.inline-field` for `buffer` is
     hidden.
   - `data-buffer-phase="lead-in"|"lead-out"` lights the active wing with the hue
     outline on the existing 60s refresh. Past fades only after the buffered end.
   - Light, dark, and reduced-motion friendly (no animation).
4. Task-status-cycler Ctrl+Enter keeps the meeting span. Run the copied close vectors.
5. Docs in `docs/bob-ledger-tools.md`: "Calendar Events" (wings, states) and
   "Calendar-aware starts" (buffered runs, notices).
6. Gate: `npm test`, `npm run build`, `build:check`, `validate`, `docs:check`; deploy
   with `bob plugins sync`.

### mac-buffers (bob-mac-capture, via `/sase_repo`)

1. Models: decode `buffer` on event rows, `items[].event`, `event_run.events[]`,
   `event_conflict.event`, and `buffer_part`. Use the tolerant
   `(try? decodeIfPresent(...)) ?? nil` pattern so a malformed buffer never fails a
   snapshot.
2. Rail (`makeCalendarRail`, `railEventCapsule`):
   - Wings are thinner hatched or low-opacity extensions flush to the capsule, in the
     home hue. They are dashed when untracked, and outlined in the hue while active.
   - The domain extends through buffered ends.
   - `firstGap` uses buffered spans, so it agrees with bob's `gap`.
3. Rows:
   - A quiet caption capsule `−30m +15m` (monospaced digits, hairline) after the title.
     It is tinted with the hue while its buffer is in progress.
   - The accessibility label adds "30-minute buffer before and 15-minute buffer after,
     holds 11:30 to 13:15".
   - The MEET group time chips append the label.
4. Preview cards:
   - The event-run caption reads
     `MEET · 11:30–13:15 · from your calendar · Lunch with Kelly −30m +15m · untracked`.
   - The conflict banner headline uses `buffer_part` ("runs 10m into Lunch's +15m
     buffer").
5. Fixtures and tests:
   - Copy `agenda-events-buffered.json`.
   - Add CaptureCore decode, presentation, and planner tests (`just portable-check` on
     Linux).
   - Add `CaptureAgendaDesignTests` renders (760/620, light/dark, high-contrast).
6. Docs: `docs/idle-agenda.md` (rail wings, row capsule) and `docs/capture-previews.md`.
7. If the Mac host is unreachable, report the macOS targets as unverified.

## Phase graph

| Phase             | Repo            | Size   | Depends on                      |
| ----------------- | --------------- | ------ | ------------------------------- |
| buffer-grammar    | bob-cli         | medium | —                               |
| gcal-sync-buffers | bob-cli         | medium | buffer-grammar                  |
| capture-buffers   | bob-cli         | medium | buffer-grammar                  |
| agenda-buffers    | bob-cli         | medium | buffer-grammar                  |
| plugins-buffers   | bob-plugins     | medium | buffer-grammar, capture-buffers |
| mac-buffers       | bob-mac-capture | medium | capture-buffers, agenda-buffers |

## Verification

- bob-cli phases: `just check` (or the documented gate), the touched goldens and
  vectors, and `just docs-check`. Every new capture path has `--dry-run` parity.
- End to end on a scratch vault (`BOB_NOW`, fake `BOB_GCAL_GOG`), with Lunch 1200-1300
  described as `@love<br>#untracked -30 +15` and a 1400 Standup described `+15`:
  - sync writes `… #untracked [buffer:: -30m +15m] [root:: love] …`;
  - `bob gcal list` shows `12:00–13:00 −30m +15m`;
  - `=` on BOB at 11:10 warns
    `runs 5m into Lunch's −30m buffer (1200-1300) · =4 ends at 1130`;
  - `x =meet` at 11:30 starts `(**1130-1315**) — MEET`;
  - `x` at 13:00 completes Lunch (meeting clock);
  - `bob capture-pomodoros --tasks -f json` shows `buffer` and a gap ending at the next
    buffered start.
- Cross-repo lockstep: the `event_buffer`, `meet_event_run`, and `event_task_close`
  vectors are byte-identical in bob-plugins and pinned to a bob-cli commit.
- Releases: bob-cli ships on the daily train
  ([[decisions:bob-cli-releases-ride-a-daily-train]]). All JSON is additive and the app
  decodes buffers tolerantly, so either side can land first.
