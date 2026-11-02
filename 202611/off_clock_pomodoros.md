---
tier: epic
title: Off-Clock Pomodoros, recorded and counted on their own clock
goal: "Sessions spent on #offclock tasks (lunch, the gym, school pickup) keep their time
  as [off:: Nm] Off-Clock Pomodoros instead of vanishing; a Pomodoro goes off the clock
  exactly when every Task Link it keeps targets an #offclock task; bob stats and the
  Stats dash gain a Worked · Off-clock · Total clock while every glanceable work signal
  stays Worked; ended off-clock events settle on their own; and one dashed visual
  language marks off-clock time everywhere.

  "
decisions:
  settled_status:
    ask:
      When an ended off-clock event settles into a closed session, should its Event Task
      row be completed?
    choices:
      done:
        Completed ([x] + done date), as a timed close completes a meeting that ended
        inside it
      in_progress:
        Marked [/] like a past row sync creates; you complete or cancel it yourself
    default: done
    why:
      settling exists so off-clock life needs no gestures; [/] would park every lunch in
      Pending
    answer: done
  memory_off_clock_glossary:
    ask: Should a new glossary strand define the Off-Clock Pomodoro?
    memory:
      - glossary:off-clock-pomodoro
    default: true
    requested:
      Add an excellent new glossary term for "untracked pomodoro" (or whatever we decide
      to call these).
    answer: true
  memory_glossary_corrections:
    ask:
      Should the pomodoro and event-task glossary strands describe off-clock sessions as
      [off::] Off-Clock Pomodoros?
    memory:
      - glossary:pomodoro
      - glossary:event-task
    default: true
    corrects:
      An `#untracked` tag keeps the session that links it off the clock (a timed ledger
      row without `[t::]`), so its `[root::]` attributes no minutes.
    answer: true
  memory_off_clock_record:
    ask:
      Should a new off-clock-pomodoros-count-apart record supersede
      untracked-sessions-omit-t-field?
    memory:
      - decisions:off-clock-pomodoros-count-apart
      - decisions:untracked-sessions-omit-t-field
    default: true
    corrects:
      Stats, homes, `bob pomodoro --worked`, and the agenda skip untracked sessions'
      minutes and counts.
    answer: true
  memory_settle_record:
    ask: Should a new ended-off-clock-events-settle decision record describe settling?
    memory:
      - decisions:ended-off-clock-events-settle
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
phases:
  - id: off-clock-ledger
    title: "bob-cli: [off:: Nm] ledger key, #offclock tag, and the off-clock vocabulary"
    size: medium
    depends_on: []
    description:
      "off-clock-ledger: replace SessionTracking with SessionClock, read and write
      [off:: Nm], accept #offclock/#off-clock/#untracked, rename every surface to
      off-clock, give off-clock homes real minutes, and land the glossary and record."
  - id: off-clock-rule
    title:
      "bob-cli: a session is off the clock iff every Task Link it keeps is #offclock"
    size: medium
    depends_on:
      - off-clock-ledger
    description:
      "off-clock-rule: stamp the clock key at every capture start, close, and
      running-lineup edit from the counted-links rule, explain mixed sessions, and pin
      it in shared pomodoro_clock vectors."
  - id: stats-clock
    title: "bob-cli: bob stats -k worked|off-clock|total and the clock JSON contract"
    size: large
    depends_on:
      - off-clock-ledger
    description:
      "stats-clock: add the request lens and pinned clock param, relabel and neutralize
      metrics, emit the clock object, off_clock sub-shares, clock-aware sessions and
      health, coverage dates, CSV and human output."
  - id: settle-ended-events
    title: "bob-cli: bob task reconcile settles ended off-clock Event Tasks"
    size: medium
    depends_on:
      - off-clock-ledger
    description:
      "settle-ended-events: reconcile turns ended #offclock Event Tasks still parked
      under an unstarted placeholder into closed [off:: Nm] MEETs trimmed around
      on-clock sessions, plus its decision record."
  - id: plugins-off-clock-ledger
    title:
      "bob-plugins: [off::] writers, Obsidian start/close rule, Event marks, home chips"
    size: medium
    depends_on:
      - off-clock-rule
    description:
      "plugins-off-clock-ledger: teach ledger writers [off::], apply the clock rule on
      se<X> starts and Ctrl+Enter closes from copied vectors, paint #offclock, and draw
      dashed home chips with minutes."
  - id: plugins-stats-clock
    title:
      "bob-plugins: Worked · Off-clock · Total control in the Stats dash and the Day
      strip hint"
    size: large
    depends_on:
      - stats-clock
      - plugins-off-clock-ledger
    description:
      "plugins-stats-clock: segmented clock control with per-view totals, off-clock and
      total renderings in hue with dashed tails, and a quiet off-clock hint under the
      Day strip's WORKED fact."
  - id: mac-off-clock
    title:
      "bob-mac-capture: off-clock vocabulary, fallback decoding, and the title chip"
    size: medium
    depends_on:
      - off-clock-rule
    description:
      "mac-off-clock: decode off_clock with untracked/tracked fallbacks, rename every
      pill, capsule, and spoken label to off-clock, and add a dashed off-clock chip to
      the agenda title line."
proposed_by: bbugyi200.apollo.research.2z.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-01 21:46:31
status: wip
---

- **PROMPT:**
  [prompts/202611/off_clock_pomodoros.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/off_clock_pomodoros.md)

# Off-Clock Pomodoros, recorded and counted on their own clock

## Goal

`bob-cli-f6` made a session spent on an `#untracked` Google Calendar event (lunch, the
gym) a timed row without `[t::]` that every reader ignores. The time is lost: Bryan
cannot ask "how much time went to family this week?", the day has holes where life
happened, and the ledger says lunch ran 1200–1300 while no total includes it.

This epic keeps that time and counts it apart. It adopts the research recommendation in
`research:202611/off_clock_pomodoro_stats/off_clock_pomodoro_stats.md` in full (Bryan
agreed to all of its requirements) and splits it into phases that distinct agents can
land.

## The design in one page

**Name.** An **Off-Clock Pomodoro** is a Pomodoro whose time Bob records but keeps off
the work clock. "Untracked" stops being true once the time is recorded, and "tracked"
already means two other things in Bob: the `#task · tracked task` mark, and the Stats
`not_tracked` quality ("untracked points"). "Off the clock" is the phrase f6 already
used, and dashed outlines already mean it.

**One positive key per timed row.** `[off:: Nm]` is the exact counterpart of `[t:: Nm]`:

```markdown
## Pomodoros

- [x] (**1030-1055** [t:: 25m]) — BUILD ← Worked 25m
  - 🍅 [[bob#^fix-sync]]
- [x] (**1200-1300** [off:: 60m]) — MEET ← Off-clock 60m
  - 🍅 ~~[[#^lunch-with-kelly]]~~
- [ ] (**1900-2000** [off:: 60m]) — MEET ← running, off the clock
  - [[#^grocery-store]]
```

| Timed row                         | Reads as                                 | Worked | Off-clock | Health               |
| --------------------------------- | ---------------------------------------- | ------ | --------- | -------------------- |
| `(**S-E** [t:: Nm])` or legacy ⏱ | on the clock                             | N      | —         | —                    |
| `(**S-E** [off:: Nm])`            | off the clock                            | —      | N         | —                    |
| `(**S-E**)`, no key               | f6-era off-clock                         | —      | span      | `unstamped_sessions` |
| both `[t::]` and `[off::]`        | malformed (old writer retimed a new row) | t      | —         | `clock_conflicts`    |
| `()`                              | placeholder                              | —      | —         | —                    |

The key states intent: a hand-typed span or a deleted `[t::]` becomes a visible
`unstamped` row instead of time that silently moves, an old writer that appends `[t::]`
produces a detectable conflict, and Dataview can `sum(rows.off)` exactly as 119 of
Bryan's notes already `sum(task.t)`. A bare row keeps f6's meaning (off the clock), so
the rows f6 writers produce until this lands stay correct. No migration is needed: the
vault has 0 bare closed rows and no `[off::]`/`[u::]` fields today.

**The tag.** `#offclock` is canonical. `#untracked` and `#off-clock` are permanent
aliases, accepted case-insensitively in Google titles and descriptions and on any task
line. Sync renders the canonical spelling, so untouched synced rows migrate in place.

**The rule.** Let **K** be the Task Links a session _keeps_. The session is off the
clock iff K is non-empty and every link in K resolves to a task whose own line carries a
standalone `#offclock` tag (or alias). Otherwise it is worked.

- An unlinked session stays worked (taken literally, "all links are off-clock" is true
  of an empty set and would push every unlinked work session off the clock).
- An unresolved link fails toward worked.
- K is what the session actually keeps: at a MEET start with an event run, the run's
  events (future lineup events are queue, not participation); at any other start, the
  lineup; at close, the links that remain after by-the-clock classification and the
  close selection move deferred and dropped links out.
- Writers stamp the key when they write the span (starts, closes, sync, settling).
  Closed rows are history: later tag edits are reported as drift, never applied.
- A mixed session is worked in full, never split per link. Capture says why, live in the
  preview: `◌ LUNCH stays on the clock: it also links Fix the parser`.

**A second clock, not a junk drawer.** Off-clock is a filter over sessions, not a
bucket: lunch rooted at `love` groups under `love` when you look at off-clock time,
which answers "where did the off-clock time go?". `bob stats -k worked|off-clock|total`
(default `worked`) re-aims every Pomodoro metric that does not pin `clock:`, and bob
relabels each metric so no surface prints "Worked" over lunch. **Total** is a sum of
sessions, not elapsed time (two closed sessions may overlap), so it reports overlap and
is never called occupied or free time.

**The work signal stays pure.** Worked, Sessions, the Day strip facts, the menu bar,
`bob pomodoro --worked`, and the agenda's `completed_summary` stay Worked. Off-clock
time gets a quiet dashed hint there, never a new fact.

**Off-clock time gets data.** An `#offclock` event nobody starts never becomes a
session. So `bob task reconcile` (every 15 minutes on the Mac) settles an ended
off-clock Event Task still parked under an unstarted placeholder into a closed
`[off:: Nm]` MEET for its span, trimmed so it never overlaps an on-clock session.

**Dashed means off the clock, everywhere.** In text the glyph is `◌`. The ledger key,
Event marks, home chips, the Mac band, the Stats dash, and the Day strip hint all speak
it:

```text
$ bob stats
Today  Worked 4h 10m · Sessions 9 · …
…
◌ 1h 30m off-clock · bob stats -k off-clock

STATS   7D  4W  3M  1Y  All   ‹ ›   ☐ vs prior   ⊞ Table
        [ ● Worked 6h 30m ][ ◌ Off-clock 1h 30m ][ ◐ Total 8h ]
```

## Decisions made here (not reviewer choices)

- **Name and vocabulary:** Off-Clock Pomodoro; tag `#offclock` (aliases `#untracked`,
  `#off-clock`); key `[off:: Nm]`; CLI values `worked | off-clock | total`; JSON
  booleans `off_clock` on entities (sessions, events, homes, gcal rows); `clock` is
  reserved for the stats lens. Code says `SessionClock::{Worked, OffClock, …}` and
  `off_clock`; "untracked" survives only in alias parsing and deprecated JSON keys.
- **Readers trust the key's number.** `[off:: N]` counts N minutes, like `[t:: N]`.
  Switching class moves the recorded number; it never re-derives minutes from the span.
- **Deprecated f6 keys stay one release.** Every f6 JSON key (`tracked`, `untracked`,
  `untracked_count`, `untracked_sessions`, homes `untracked`) keeps being emitted beside
  its `off_clock` replacement until the first bob-cli release that contains this epic
  has shipped. Clients read the new key first. No f6 key has shipped in a release yet
  (latest tag `v0.4.0`), but the plugins and the Mac app already decode them.
- **No write gate.** Today's plugin writers keep unknown range metadata byte-for-byte,
  so an `[off::]` row written before `plugins-off-clock-ledger` deploys survives every
  Obsidian gesture. The only transient flaw is a stale `[off:: N]` number after an
  Obsidian `\p`; it disappears when the plugin phase deploys.
- **The rule's pure core lives with the key** (`off-clock-ledger`), so the start/close
  writers (`off-clock-rule`), stats drift (`stats-clock`), and the plugins share one
  definition.
- **Settling runs in `bob task reconcile`**, not `bob gcal sync`. Sync runs only on
  demand and once when the daily note is created; reconcile already owns ledger
  structure and runs every 15 minutes on the Mac.
- **Out of scope:** exempting off-clock links from `plan.max_links`, counting off-clock
  Event Task completions in task metrics, and the menu bar indicator.

Rejected: absence of `[t::]` as the meaning (f6; silent and indistinguishable from a
deleted key); `[u:: Nm]` (only worth it if the tag stayed `#untracked`); a `[t:: 0m]`
row; deriving the clock at read time from links (f6 rejected it; every reader would
resolve links, and tag edits would rewrite history); re-clocking today's closed rows in
reconcile (fights hand overrides); per-link minute splits (stores a guess); a binary
"include untracked" switch (cannot say "off-clock only"); a formula
`total = worked + off_clock` (formulas have no homes); calling the third view "All"
(already a range chip) or "Occupied" (false under overlap); a Day strip that follows the
dash's lens (the strip would change meaning on every note).

## Shared definitions (all phases)

- **Off-clock tag:** a standalone `#offclock`, `#off-clock`, or `#untracked` token,
  case-insensitive, whitespace-bounded on the left, followed by whitespace, end of line,
  or one of `])}:.,;!?`, not inside a code span or Markdown link (today's
  `line_has_untracked_tag` rules in `src/native/event_task.rs`). A nested tag
  (`#offclock/x`) does not qualify, nor does a tag inherited from a parent task or note.
- **Counted Task Links (K's candidates):** dedicated Task Link bullets beneath the
  session, plain, struck/done, `🍅`-marked, or embedded (`docs/pomodoro-homes.md` R3 and
  `glossary:task-link`; reuse `pomodoro_homes::session_links`). Not counted: deferred
  `[[…]]#` bodies, mentions, LINKS/LINKED FROM entries, and Depends-On links.
- **Clock verdict:** `OffClock` iff K ≠ ∅ and every link resolves to exactly one task
  whose own line has the off-clock tag; else `Worked` with a reason: `Unlinked`,
  `Mixed { first_on_clock }`, or `Unresolved { first }`.
- **Clock diagnostics:** `unstamped` (bare closed timed row), `conflict` (both keys),
  `drift` (a closed row whose key disagrees with the verdict recomputed now, counted
  only when every link in K resolves).

## Phase details

### off-clock-ledger (bob-cli)

Make the key, the tag, and the vocabulary real everywhere f6 touched, without changing
any number Worked reports.

1. **Classifier.** Replace `SessionTracking`/`session_tracking`
   (`src/native/capture/pomodoro_adjust.rs` ~806–823) with
   `SessionClock { Worked { minutes }, OffClock { minutes, stamped }, Placeholder }`
   plus a `ClockStamp { Keyed, Bare, Conflict }` diagnostic, per the table above. Parse
   `[off:: …]` like `[t:: …]` (case-insensitive key, `parse_adjustment_duration`
   values). Update every caller: `capture_pomodoros.rs` (`completed_summary_for`,
   `attributed_minutes`, entry flags), `capture/pomodoro_blocks.rs`
   (`headline_is_tracked`), `capture/meet_event_run.rs`, `pomodoro_homes/session.rs`,
   `stats/index.rs` (`PomodoroSession` carries the clock; stats output must stay
   byte-identical in this phase: off-clock sessions still leave every metric).
2. **Writers keep the key.** `format_adjusted_range` keeps whichever key the row has and
   updates its minutes (adjust `+N`/`-N`, shift `++N`/`--N`, split, close retimes); a
   bare row is stamped `[off:: Nm]` from the span; a conflict row keeps `[t::]` and
   drops `[off::]`. Replace `set_ledger_tracking` with `set_ledger_clock(ledger, clock)`
   that moves the recorded number between keys. No writer ever produces a bare timed row
   or both keys again. The f6 MEET event-run rule now writes `[off:: Nm]` where it
   omitted `[t::]`.
3. **Tag aliases.** Generalize `line_has_untracked_tag` into `line_has_off_clock_tag`
   (one scanner, three spellings); `clean_event_title` strips any alias;
   `EventTask.untracked` becomes `off_clock`. Add the pure rule core: a
   `counted_session_links` helper and `clock_verdict(&[LinkFact]) -> ClockVerdict` over
   caller-resolved link facts, with unit tests for every edge.
4. **gcal.** `gcal/model.rs` accepts all three spellings in titles and description-only
   lines (strip the tag, then read the title-ending `@route`); `render.rs` writes
   ` #offclock` at the end of the badge run, so untouched rows written as `#untracked`
   migrate in place on the next sync (cover in `tests/gcal_lifecycle.rs`); `plan.rs`
   writes `(**SSSS-EEEE** [off:: Nm])` for off-clock past clusters and a created
   off-clock running MEET. Rename the once-only report `kept_untracked` to
   `kept_off_clock`. `bob gcal list` VAULT reads `task · MEET · off-clock`; list and
   sync JSON gain `off_clock` beside the deprecated `untracked`.
5. **Agenda and capture JSON (schema 1, additive).** `capture-pomodoros --tasks`:
   `pomodoros[].off_clock` (omitted when false; deprecated `tracked: false` kept),
   `completed_summary.off_clock: {count, minutes}` (omitted when 0; deprecated
   `untracked_count` kept; `count`/`minutes` stay Worked), `events.rows[].off_clock`,
   `items[].event.off_clock`. Capture: `pomodoro_start.off_clock`,
   `pomodoro_start.event_run.off_clock` (and its events),
   `event_conflict.event.off_clock`, `pomodoro_blocks[].off_clock`, each beside its
   deprecated key. Human copy says `off-clock`:
   `started MEET 1200-1300 (60m · off-clock) · from your calendar: Lunch`
   (`start_output.rs` `untracked_label`), the agenda's `· off-clock` event detail.
   Refresh the touched goldens (`tests/fixtures/capture_pomodoros/agenda-*.json`).
6. **Homes.** A closed off-clock session attributes its **real minutes** with
   `off_clock: true` on the homes object (deprecated `untracked: true` kept). Update
   `tests/fixtures/pomodoro_homes/vectors.json`: replace `untracked-zero-minutes` with
   `off-clock-real-minutes`, add `off-clock-bare-span` and `clock-conflict-worked`
   cases. `stats/homes.rs` keeps passing 0 minutes for off-clock sessions until
   `stats-clock` (Worked must not move in this phase).
7. **Vectors.** `tests/fixtures/meet_event_run/vectors.json`: off-clock runs now expect
   `[off:: Nm]`; replace `expected_tracked` with `expected_off_clock`; update the
   README. Add an off-clock MEET case to `tests/fixtures/event_task_close/vectors.json`
   if its harness covers ledger keys.
8. **Docs.** `docs/reference/vault-format.md` (the key table, the tag and aliases),
   `docs/capture.md` (start/adjust/shift/close wording and JSON fields), `docs/gcal.md`
   (rename "Untracked events" to "Off-clock events" and keep the old `#untracked-events`
   anchor resolving; Sync table; ownership table; Known limits),
   `docs/pomodoro-homes.md` (R5 real minutes, vector table),
   `docs/reference/json-outputs.md`, and the pomodoro paragraph of `docs/stats.md` ("an
   off-clock session leaves every Worked metric"). Never write "untracked session" in
   new prose; say Off-Clock Pomodoro (term) or off-clock session.
9. **Memory** (`/sase_memory_write`, then `sase memory init` and `just docs-gen` so
   `docs/reference/glossary.md` and `docs/develop/decisions.md` regenerate):

   > [!decision] memory_off_clock_glossary Add the `glossary:off-clock-pomodoro` strand
   > (aliases: off-clock session, untracked pomodoro, untracked session), close to: "A
   > [[pomodoro]] whose time Bob records but keeps off the work clock — lunch, the gym,
   > school pickup. Its timespan carries `[off:: Nm]` where a worked session carries
   > `[t:: Nm]`: `- [x] (**1200-1300** [off:: 60m]) — MEET`. A session goes off the
   > clock exactly when it keeps at least one [[task-link]] and every one resolves to a
   > task tagged `#offclock` (aliases `#untracked`, `#off-clock`); an unlinked session,
   > an unresolved link, or one on-clock link keeps the whole session worked. Writers
   > decide when they write the span (starts, closes, and `bob gcal sync` for an
   > off-clock [[event-task]]); a closed row's key is history, and later tag edits are
   > reported, never applied. Worked, Sessions, `bob pomodoro --worked`, and the
   > [[day-strip]] facts leave it out; its [[root-parent]] still receives its minutes.
   > Drawn dashed; `◌` in text. A bare closed `(**HHMM-HHMM**)` row is an older
   > off-clock session counted by its span. Not a `()` placeholder, a Stats day before
   > tracking began (`not_tracked`), or the tracked-task mark on a `#task` row."
   > `stats-clock` and `settle-ended-events` each add their own sentence when they land.

   > [!decision] memory_glossary_corrections In `glossary:pomodoro`, "representing one
   > work session" becomes "representing one timed session, on the clock (`[t::]`) or
   > off it (`[off::]`, an [[off-clock-pomodoro]])".

   > [!decision] memory_glossary_corrections In `glossary:event-task`, the `#untracked`
   > sentence becomes: "An `#offclock` tag (alias `#untracked`) makes the session that
   > links it an [[off-clock-pomodoro]], so its `[root::]` attributes off-clock
   > minutes."

   > [!decision] memory_off_clock_record Write
   > `sase/memory/decisions/off-clock-pomodoros-count-apart.md` ("Off-Clock Pomodoros
   > Are Recorded And Counted Apart"; Applies to / Claim / Why with the rejected
   > alternatives above / Cost / Reopens when / Evidence citing this plan, the research
   > report, `docs/reference/vault-format.md`, `docs/stats.md`, and the vectors). Claim:
   > a timed row with `[off:: Nm]` is an Off-Clock Pomodoro; writers stamp it by the
   > counted-links rule at start, close, sync, and settling; closed rows are frozen and
   > drift is reported; Worked, Sessions, `--worked`, the Day strip, and the agenda
   > summary exclude it (`untracked_counts = exclude` still governs Worked); the
   > Off-clock and Total clocks count it; a bare closed span reads as off-clock and is
   > flagged unstamped. Cost: mixed sessions count as worked, history is frozen at write
   > time, Total can double-count overlaps. Reopens when per-task minute accounting is
   > needed, or off-clock should come from an area property instead of a task tag. Mark
   > `decisions:untracked-sessions-omit-t-field` `superseded` with `superseded_by` and a
   > `[[decisions:off-clock-pomodoros-count-apart]]` back-link stating that absence of
   > `[t::]` no longer means off the clock and that off-clock time is now counted apart.

10. **Tests.** Classifier table (all five row shapes, case-insensitive key, `1h 30m`
    values); retime round-trips keep each key; alias parsing and title cleaning; the
    verdict core; gcal title/description aliases, canonical render, and in-place
    migration; agenda and capture JSON carry both keys; homes vectors; a guard that
    `bob stats` JSON and human output on the stats fixture vault are unchanged. Mark the
    commit `!` (homes parts change meaning).

### off-clock-rule (bob-cli)

Apply the rule at every managed write, and make its outcome visible before Bryan
commits.

1. **Starts.** Generalize the f6 MEET rule (`capture/meet_event_run.rs`
   `finalize_meet_start`, applied via `apply_meet_rule` from `pomodoro_start.rs` and
   `pomodoro_link.rs`) into one `finalize_session_clock` for every start-family write
   (plain, named, created, restart, swap including kept-ledger swaps, link starts): a
   MEET with a non-empty event run uses the run's clock; every other session applies the
   verdict to its counted lineup links, resolved against the planner's staged vault view
   (same-note Event Tasks via `event_task_by_id`, other notes through the capture
   resolver). A kept ledger switches keys with `set_ledger_clock`. An `#offclock` "Walk
   the dog" start writes `[off:: 25m]`.
2. **Closes.** In `capture_pomodoro_close` (`plan_pomodoro_close` →
   `plan_ledger_close_with_parked`), compute K from the links the closed entry keeps
   after by-the-clock classification and the selection (deferred and dropped lines
   removed; a worked link also carried forward still counts), resolve through the
   existing `CloseVault`, and write the verdict's key on the closed row with its final
   minutes. `close_timing` must switch keys even when it does not retime
   (`remaining < 5`). The close summary names the clock: `(60m · off-clock)`.
3. **Running lineup edits.** Capture gestures that add links to or drop links from the
   running session (`pomodoro_link.rs`, `~K` drops) re-stamp its key. Correctness never
   depends on this (open rows never count); it keeps the Now card honest.
4. **Mixed notice.** When K holds at least one off-clock link but the verdict is worked,
   add a structured warning `pomodoro_clock_mixed` at start and close:
   `◌ LUNCH stays on the clock: it also links Fix the parser` or
   `◌ LUNCH stays on the clock: [[bob#^gone]] does not resolve`. It appears in
   `--dry-run`, so the Mac live preview shows it before Return.
5. **Vectors.** Add `tests/fixtures/pomodoro_clock/vectors.json` + README (pinned
   commit; mirror target `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_clock/` and
   task-status-cycler for closes), driven by a Rust test through `bob capture -f json`.
   Each case: `notes` (path → contents, so other-note links resolve), `day`, `now`,
   `token`, `expected_day`, `expected_off_clock`, `expected_warning`. Cover: unlinked;
   all off-clock (non-MEET); mixed; unresolved; alias spellings; nested tag rejected;
   MEET run with a future off-clock event queued behind an on-clock run; close after a
   not-started off-clock event is deferred; close of a mixed session after dropping the
   on-clock link; kept-ledger swap; bare row restamped at close; conflict row at close.
6. **Docs.** `docs/capture.md`: a "Which clock a session is on" subsection (the rule,
   the four edges, examples, the notice) linked from Starting, Closing, Overriding, and
   the JSON section; `docs/reference/capture-syntax.md` if it states start timing rules.
7. **Tests.** Every vector case, plus `--dry-run` parity and kept-ledger, restart, and
   link-start paths.

### stats-clock (bob-cli)

Give stats a second clock while `bob stats` with no flag stays byte-identical whenever
the window has no off-clock time.

1. **Pinned param.** Pomodoro metric configs gain `clock: worked | off-clock | total`
   (`registry.rs` `parse_pomodoro`, `Option<Clock>`; unset follows the lens), validated
   like `recurring:` (a bad value invalidates only that metric; `clock:` on a
   non-Pomodoro kind is an `invalid_metric` warning). Example:
   `family_time: {kind: pomodoro, clock: off-clock, in: [love]}`.
2. **Request lens.** `bob stats` and `bob stats show` gain
   `-k, --clock <worked|off-clock|total>` (default `worked`; hidden value aliases `on`,
   `off`, `untracked`, `all`), threaded through `StatsRequest`. It re-aims every
   Pomodoro metric without a pinned `clock:`; it never touches task, checklist, journal,
   or formula metrics or formula inputs (key evaluations by `(id, clock)` so a displayed
   lensed metric and an unlensed formula input can coexist). Options stay alphabetical
   (`-k` between `-h` and `-o`; `tests/cli/help_options.rs`), and completion follows.
3. **Minutes and coverage.** Worked counts `[t::]` minutes; Off-clock counts `[off::]`
   minutes and bare spans; Total counts both. Off-clock and Total series are
   `not_tracked` before the **off-clock adoption date** (the first day with an off-clock
   session; a metric's `since:` still wins when later), so typical-day medians are not
   dragged to 0 by years without data.
4. **Relabel and direction** (`compute.rs` where `MetricReport` copies label and
   direction; ids never change): `worked` → Worked / Off-clock / Total; `sessions` →
   Sessions / Off-clock sessions / Total sessions; any other label → `Label` /
   `Label · off-clock` / `Label · total`. Off-clock and Total views report direction
   `neutral`.
5. **JSON (schema 1, additive).**
   - Top-level
     `clock: {value, label, since, options: [{value, label, description, minutes, display}]}`:
     the active view, the adoption date (null when the vault has no off-clock session),
     and each view's total over the current window, so one object draws the whole
     control.
   - `parameters.clock` on each Pomodoro metric (the effective clock).
   - Pomodoro metric points and daily points carry `off_clock` (omitted when 0): the
     bucket's off-clock amount for the metric's filter — the sub-share of `value` under
     `total`, the amount `value` leaves out under `worked`, omitted under `off-clock`.
     Part summaries carry `off_clock` under `total`.
   - `sessions[]` carry real minutes and `off_clock: true` when off the clock; the view
     filters them (Worked omits off-clock rows). Mark the commit `!`.
   - Health: `off_clock_sessions`, `unstamped_sessions`, `clock_conflicts`,
     `clock_drift`, `overlap_minutes` plus a capped `overlaps[]` lint list (date, lines,
     minutes), each omitted when 0/empty; deprecated `untracked_sessions` kept. Drift
     uses the shared verdict core over the session's counted links.
   - Homes in stats (`stats/homes.rs`) attribute real minutes for off-clock sessions
     under the off-clock and total views.
6. **Human and CSV** (`render.rs`). Worked view: unchanged, plus one dim line when the
   window has off-clock time: `◌ 1h 30m off-clock · bob stats -k off-clock`. Off-clock
   view: header suffix `· off the clock`; an empty window says
   `No off-clock time in this window`. Total view: the headline breaks down as
   `8h · 6h 30m worked + 1h 30m off-clock`. Non-zero conflicts, drift, unstamped, and
   overlap surface through the existing health output. CSV appends a trailing `clock`
   column (effective clock for Pomodoro rows, empty otherwise); update the exact-header
   test and mark the commit `!`.
7. **Docs.** `docs/stats.md` (Metric kinds, `stats:` config `clock:`, Data model and
   quality, Schema-1 contract, `bob stats` human/CSV/JSON, the M- and H-vector tables
   for new cases); `just docs-gen` regenerates `docs/reference/cli/stats.md`. Use
   "off-clock" for the new concept; "untracked points" keeps meaning `not_tracked`.

   > [!decision] memory_off_clock_glossary Add one sentence to the
   > `glossary:off-clock-pomodoro` strand: the Off-clock and Total clocks of
   > `bob stats -k` and the [[stats-dash]] count it, grouped by [[root-parent]] like any
   > session. Republish with `sase memory init` and `just docs-gen`.

8. **Tests.** Extend the stats fixture vault (`tests/fixtures/stats/vault/`) so one day
   has worked, `[off::]`, bare, and conflict rows, and two overlapping sessions; assert
   every label under every view (one stray "Worked" over lunch would undo the trust this
   feature builds), pinned-vs-lens precedence, formula isolation, coverage dates,
   sub-shares adding up, sessions filtering, health counts, CSV, human output, and
   byte-identical default output on a vault with no off-clock time.

### settle-ended-events (bob-cli)

1. **Rule.** In `bob task reconcile`'s structural planning
   (`src/native/task_status_hooks/structure.rs` `plan_structural_changes`), on the
   current ledger at `now`: every open, off-clock Event Task whose span has ended and
   whose only Task Link sits under an open untimed placeholder `()` settles. Cluster
   settling events like sync's past clusters (`next.start ≤ cluster end`); subtract
   every on-clock timed session's span (closed or running); each remaining piece of at
   least 5 minutes becomes a closed `- [x] (**SSSS-EEEE** [off:: Nm]) — MEET` inserted
   in time order (reuse gcal's closed-MEET insertion), linking `🍅 [[#^id]]` for each
   event it overlaps; the placeholder links move out (the empty-Pomodoro rule then
   deletes a childless placeholder). A cluster fully covered by on-clock sessions does
   not settle. Cancelled, done, and unlinked rows never settle. Idempotent.

   > [!decision] settled_status = done A settled event's row is completed (`[x]` plus
   > its done date) through the same completion path a timed close uses for a meeting
   > that ended inside the session; reconcile then retires its link as struck.

   > [!decision] settled_status = in_progress A settled event's row becomes `[/]`, like
   > a past row sync creates, and stays in the Pending lane until completed or
   > cancelled.

2. **Output.** Human: `◌ settled Lunch with Kelly 1200-1300 off the clock`; JSON adds a
   `settled[]` array (event title, line, span, created entry line, minutes). `--dry-run`
   previews it.
3. **Docs.** `docs/task-status-hooks.md` (the rule in "In practice", output), a pointer
   from `docs/gcal.md` "Off-clock events" ("cancel the row before it ends to keep it off
   the record"), Known limits (an event ending after the last reconcile before midnight
   does not settle).
4. **Memory:**

   > [!decision] memory_settle_record Write
   > `sase/memory/decisions/ended-off-clock-events-settle.md` ("Ended Off-Clock Events
   > Settle Into Closed Sessions"): claim, why (the off-clock clock needs data; Bryan's
   > first instinct was no gesture at all), rejected alternatives (settling in sync,
   > which rarely runs after events end; settling at capture time; no settling), cost
   > (the calendar is not proof of attendance; a skipped event counts unless cancelled
   > before it ends; per `settled_status`, the row status written), reopens when
   > (attendance becomes knowable, or settling should also cover running sessions).

   > [!decision] memory_off_clock_glossary Add one sentence to the
   > `glossary:off-clock-pomodoro` strand: an ended `#offclock` Event Task nobody
   > started settles into one on the next `bob task reconcile`.

5. **Tests.** Reconcile tests: settle one event; back-to-back cluster; trimmed around a
   closed on-clock session (one and two pieces); fully covered; running on-clock
   session; placeholder with mixed future and ended links; idempotent rerun; dry-run.

### plugins-off-clock-ledger (bob-plugins, via `/sase_repo`)

Read the repo's `AGENTS.md`; edit fragments and rebuild, never `main.js`.

1. **Key.** `plugins/bob-ledger-tools/src/010-load-and-constants.js` gains an `[off::]`
   regex beside `DURATION_FIELD_RE`; `020-time-and-pomodoro.js` (`durationField`,
   `formatDurationField`, `removeDurationMetadata`, `pomodoroDurationMinutes`,
   `changePomodoroLineUnits`) learns both keys with the Rust writer rules (keep the key,
   stamp a bare row `[off::]`, resolve a conflict to `[t::]`). `\o`/`\O` already keep
   metadata; add regression tests.
2. **Starts.** `035-meet-event-run.js` writes `[off:: Nm]` for an off-clock run;
   `030-snippets.js` / `300-plugin-vim-and-snippets.js` apply the verdict to a non-MEET
   `se<X>` lineup using the plugin's existing task resolver (the one home chips use).
3. **Closes.** task-status-cycler's Pomodoro Ctrl+Enter
   (`plugins/task-status-cycler/src/060-pomodoro.js` `buildPomodoroCompletionPlan`, with
   `061-event-task-clock.js`) computes K after its by-the-clock classification and
   switches the key when the class changes.
4. **Tag and copy.** `eventTaskUntrackedTagRange`/`eventTaskLineIsUntracked`/
   `eventRunCleanTitle` accept all three spellings; the Event painter
   (`291-plugin-gcal-events.js`, `styles.css` ~4235–4287) marks them "off-clock" (Lucide
   `timer-off`, dashed pill; title "Off-clock · on the calendar, off the clock"). Rename
   helpers, classes, and data attributes to the off-clock vocabulary.
5. **Home chips.** `142-pomodoro-home-chips.js`: off-clock sessions draw dashed chips
   **with their minutes**, tooltip `love · 60m off the clock`, aria "(off-clock)"; fix
   Reading view (`274-plugin-pomodoro-home-chips-reading.js`), which never passes the
   flag today.
6. **Vectors.** Copy byte-for-byte from bob-cli and pin each README to the bob-cli
   commit: new `pomodoro_clock` (ledger-tools starts; task-status-cycler closes),
   updated `meet_event_run` (add the missing README and pin), `pomodoro_homes` (fix the
   stale `b71a6c15` pin), and `event_task_close` if bob-cli changed it.
7. **Ship.** `docs/bob-ledger-tools.md` and the manifest description, version bumps for
   each touched plugin, `npm run docs:gen`, `npm test`, `npm run build:check`,
   `npm run validate`, `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`, then
   `bob plugins sync`.

### plugins-stats-clock (bob-plugins, via `/sase_repo`)

1. **Control.** A labelled radio group (segmented control) under the range chips, drawn
   entirely from `report.clock.options`:
   `● Worked 6h 30m · ◌ Off-clock 1h 30m · ◐ Total 8h`. Shown only when `report.clock`
   exists and `clock.since` is set (older bob or a vault with no off-clock time keeps
   today's dash). Arrow keys and Space as for any radio group, plus one free key that
   cycles views. The choice is remembered on this device for the dash only
   (`app.saveLocalStorage`, like the Day strip bars). `238-plugin-stats-block.js` is
   near the 1000-line fragment cap: put the control in a new fragment (the homes toggle
   in `239-plugin-stats-homes-render.js` is the pattern).
2. **Fetching.** `statsBuildArgs`/`statsFetchArgsFor` add `-k` only for non-default
   views, so the default request and its cache key (shared with the Day strip) are
   unchanged. Switching back is instant from the argv-keyed cache; prefetch the other
   two views when idle. Bars morph as on a range change; reduced motion turns it off.
3. **Off-clock view.** Tiles, focus chart, window bars, and "Where the time went" draw
   each root's own hue as a dashed outline over a light tint (`love` stays love's
   color); the header adds `· off the clock`; empty windows say "No off-clock time in
   this window". Replace hard-coded `"worked"` titles and meta text with bob's labels.
4. **Total view.** Each bar, lane segment, and root row is solid for Worked with a
   dashed tail for Off-clock from bob's `off_clock` sub-shares (no client arithmetic);
   hover and focus: `Thu 29 · 8h total · 6h 30m worked · 1h 30m off-clock`. Session
   drill-down rows mark off-clock sessions dashed.
5. **Copy and a11y.** Comparisons in Off-clock and Total views are neutral ("45m more
   than the prior period", no celebratory ▲ color); aria sentences name the view; dashes
   survive `forced-colors` (border style, not color).
6. **Day strip.** WORKED and both lanes are unchanged and keep the default request. When
   the day's `worked` point carries `off_clock`, a quiet dashed `◌ +1h off-clock`
   appears under WORKED; clicking it opens the dash in the Off-clock view with the day
   pinned (`openStatsForDay` sets the clock).
7. **Docs and fixtures.** bob-cli `docs/stats.md` "Obsidian Stats block", "Window bars",
   and "Day strip" (bob-cli docs, edited from the bob-cli workspace);
   `docs/bob-ledger-tools.md`; regenerate `scripts/fixtures/stats/canonical.json` from
   bob-cli per its README; add synthetic off-clock/total fixtures and the preview
   scripts' cases; tests for the control, argv, cache sharing, both renderings, the
   hint, and persistence. Ship as in `plugins-off-clock-ledger`.

### mac-off-clock (bob-mac-capture, via `/sase_repo`)

Read the repo's `AGENTS.md` (compile rules, format only changed lines).

1. **Decode.** `CaptureAgendaModels.swift` and `CaptureModels.swift`: decode every
   `off_clock` key with `decodeIfPresent`, falling back to the deprecated `untracked` /
   `tracked == false` / `untracked_count`; add
   `completed_summary.off_clock {count, minutes}`. Rename Swift properties to
   `offClock`.
2. **Copy.** Every visible and spoken "untracked" becomes "off-clock": the Now header
   pill and its label ("off-clock session"), Calendar row capsules and labels, the start
   preview's event-run text. Dashes stay as they are.
3. **Title chip.** When `completed_summary.off_clock.minutes > 0`, the agenda title line
   adds a dashed `◌ 1h off-clock` chip after the Worked summary; decorative chips stay
   out of VoiceOver except through the title's label.
4. **Notice.** Confirm the start/close previews show the `pomodoro_clock_mixed`
   structured warning in the existing warning style; add a test.
5. **Tests and docs.** Update `Tests/Fixtures/agenda-events.json` to carry both keys;
   add an old-keys-only decoding test, an `[off:: 60m]` headline tokenizer test
   (`CapturePomodoroLineTokens.swift` is key-agnostic; refresh its comment), and copy
   assertions. `docs/idle-agenda.md`, `docs/capture-previews.md`. Run
   `just portable-check` and compile the macOS targets per `AGENTS.md`; if the Mac is
   unreachable, report the macOS targets as unverified.

## Compatibility and rollout

- Order: `off-clock-ledger` → (`off-clock-rule` ∥ `stats-clock` ∥ `settle-ended-events`)
  → `plugins-off-clock-ledger` and `mac-off-clock` after `off-clock-rule` →
  `plugins-stats-clock` after `stats-clock` and `plugins-off-clock-ledger` (both plugin
  phases touch bob-ledger-tools' build output, so they run in sequence).
- bob-cli ships only on the daily release train
  ([[decisions:bob-cli-releases-ride-a-daily-train]]); new JSON keys need no `!`; the
  homes-minutes, `sessions[]`, and CSV changes do. Agents never edit the Cargo version
  or `CHANGELOG.md`.
- Every client reads the new key first and falls back to the f6 key, so bob-cli and each
  client can land in either order. The last phase to land records a
  `PROPOSED FOLLOW-UP:` to remove the deprecated f6 keys after the first release that
  contains this epic.

## Verification

- bob-cli phases: `just check` (or the repo's documented gate), the touched goldens and
  vectors, `just docs-gen` then `just docs-check`, and `--dry-run` parity for every new
  capture and reconcile path.
- End to end on a scratch vault (`BOB_NOW`, fake `BOB_GCAL_GOG`): sync
  `Lunch @love #untracked` 1200–1300 → the row renders `#offclock`; at 11:58 `=` on the
  MEET writes `(**1200-1300** [off:: 60m]) — MEET`; `x` at 13:00 keeps `[off:: 60m]`;
  `bob stats` shows unchanged Worked plus `◌ 1h off-clock · bob stats -k off-clock`;
  `bob stats -k off-clock` attributes 60m to `love`; `-k total` breaks down the
  headline. A second, unstarted `#offclock` event that ends settles on
  `bob task reconcile`. A `=` start linking an off-clock task and a work task warns
  `pomodoro_clock_mixed` and writes `[t::]`.
- Cross-repo lockstep: `pomodoro_clock`, `meet_event_run`, and `pomodoro_homes` vectors
  copied byte-for-byte into bob-plugins with README pins.
- Plugins: `npm test`, `build:check`, `validate`, `docs:check`, `bob plugins sync`.
- Mac: `just portable-check` plus macOS target compilation; render the agenda fixtures
  in light/dark.
