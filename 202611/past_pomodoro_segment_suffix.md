---
tier: epic
title: Place past Pomodoros on today's timeline with the x= segment suffix
goal: '`x=E[~|~n|~n+|~~][!][=X]` records past work against an edge of today''s timeline
  (the tail since the last closed Pomodoro, a closed Pomodoro, a gap, or the head
  before the first row). Placement is exact and guarded, every cut is reported, and
  a plain `x=` stays byte-for-byte unchanged. bob ships a segment picker, a typed
  `-` to `~` assist, and a stale-preview guard. Bob Mac Capture renders the suffix,
  the picker, and a card that names the segment.

  '
phases:
- id: timeline
  title: Timeline catalog and pinned layout engine
  depends_on: []
  size: medium
  description: 'timeline: add a pure pomodoro_timeline module that turns today''s
    ledger into numbered segments (closed Pomodoros, gaps, tail, head, walls) and
    lays a run of windows out from a pin with exact, capped, and cut lengths; heavily
    unit-tested, not yet wired.'
- id: grammar
  title: Segment suffix lexer, pins, spans, and teaching errors
  depends_on: []
  size: medium
  description: 'grammar: lex the optional suffix at the end of E, carry it on PomodoroPastSpec,
    compute one pin per run with misplaced/double-pin diagnostics, emit the pomodoro_past_segment
    span and capture-parse JSON, and replace the old ~/dash teaching errors; execution
    refuses suffixed tokens until record lands.'
- id: record
  title: Record pinned runs end to end and ship grammar 14
  depends_on:
  - timeline
  - grammar
  size: medium
  description: 'record: wire the timeline and layout into plan.rs for pinned runs
    (snapshot first, unpinned runs untouched), add guard hints and the same-capture
    trim rule, report segment and cut in capture JSON and human output, document the
    feature, and bump the grammar to 14.'
- id: assists
  title: Segment picker, dash-to-tilde rewrite, and submit guards
  depends_on:
  - record
  size: medium
  description: 'assists: add the pomodoro_past_segment completion context and candidates
    (plus zsh), the - to ~ arm of capture-rewrite, the hidden --expect-past-windows
    stale-preview guard, and the shell-expanded-tilde refusal, with docs and help.'
- id: decision
  title: Decision record for segment placement
  depends_on:
  - record
  size: small
  description: 'decision: author the accepted decisions strand that records segment
    placement, one pin per run, exact typed counts, the now cut, and unchanged ! semantics,
    then regenerate memory.'
- id: mac
  title: Bob Mac Capture segment typing, picker, card, and guard
  depends_on:
  - assists
  size: medium
  description: 'mac: add - to the rewrite trigger keys, style the segment span, render
    picker rows, decode and show the segment line and cut note on the past card, and
    send --expect-past-windows with a re-preview on refusal; regenerate fixtures and
    update protocol docs.'
decisions:
  decision_record:
    ask: Add a new accepted decisions strand recording the x= segment-suffix design?
    memory:
    - decisions
    default: true
    requested: Review the past_pomodoro_segment_suffix.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file
    answer: false
proposed_by: bbugyi200.athena.18s
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 14:25:22
status: done
bead_id: bob-cli-hj
---

- **PROMPT:** [prompts/202611/past_pomodoro_segment_suffix.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/past_pomodoro_segment_suffix.md)
- **BEAD:** [bob-cli-hj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hj/README.md)

# Plan: Place past Pomodoros on today's timeline (`x=` segment suffix)

## Why and what

Today `x=E[!][=X]` can only record work that ended _now_: every run of `x=` items ends
at `T = ceil5(now)`. Logging work that ended an hour ago, or claiming the first ten
minutes of a session already closed, still means editing the daily note by hand. This
epic adds an optional **segment suffix** at the end of `E` that says _where_ the
recorded session sits. The count already says how long, the name says which session, and
the part after the second `=` says what happened to its tasks.

The design follows the consolidated research report
`research:202611/past_pomodoro_segment_suffix/past_pomodoro_segment_suffix.md`. Bryan
accepted every requirement in it (R1–R12 and the _Recommended solution_), including its
answers to the four open questions:

1. Typed overflow out of a gap trims a neighbour under `!`.
2. A run of `x=` items hangs from one pin.
3. Suffixes resolve against the staged snapshot just before the run.
4. Three-or-more-digit indexes are reserved for a future `~HHMM` clock suffix.

Every phase worker should read that report with `sase artifact read` before coding. This
plan is the binding contract wherever the two differ.

Mnemonic for docs and errors: **_before the second `=` is when; after it are tasks. `~`
steps back along today's timeline; `+` hangs the session on the segment's far end._**

## The shared contract

Every phase implements against this section. Phase sections say only which slice each
one owns.

### Vocabulary (R1)

- A **segment** is one piece of today's timeline. There are four kinds:
  - **Pomodoro**: a closed timed row.
  - **gap**: an uncovered stretch.
  - **tail** (`~`): the time since the last closed Pomodoro.
  - **head** (`~~`): the time before the first row.
- The syntax is the **segment suffix**.
- Never call these "blocks", which already means five minutes. Never call them an
  "anchor", because JSON `pomodoro_past.anchor` already means `T`.

### Grammar

```text
x=E[s][!][=X] [work-log entry]

E  = [N] [name | #name | #"LIT"] [N]      (unchanged)
s  = "~"          tail: start where your last closed Pomodoro ended
   | "~" n        segment n: start at its start
   | "~" n "+"    segment n: end at its end
   | "~~"         head: end where your first Pomodoro starts
n  = 1..99, no leading zero               (3+ digits reserved for ~HHMM)
```

- The suffix is always the last part of `E`. It comes after the count and name and
  before the repair `!` and the second `=`.
- `~` already ends an unquoted name (`past_name_extent`), and `#"…"` literals still
  protect their bytes. So `x=#c++~1+` is the name `c++` with the suffix `~1+`, and
  `x=#deep-work~2` is `deep-work` with `~2`.
- `x=2~1!=!1` splits as `E=2~1`, repair `!`, then close `!1`. This is unchanged
  `split_past_token` behavior.
- The CLI never accepts `-` as a suffix (R9). Only the editor rewrite turns a typed `-`
  into `~`.

### The timeline model

Inputs are the staged contents of today's `## Pomodoros` section, `now` (truncated to
the minute), and `T = ceil5(now)`. bob builds the timeline as follows:

1. **Rows.** Take every timed row (`collect_timed_rows`: zero-length rows are skipped,
   and a midnight-ending span is unwrapped to minute 1440) whose start is `< T`. Closed
   or open, on- or off-clock, MEET or not. Untimed `()` placeholders are not on the
   timeline. Rows starting at or after `T`, such as a future synced MEET, are ignored
   (R3).
2. **Walls.** Open timed rows get no number and are never written. A wall covers
   `[start, T)`, so a running session that overran its planned end still covers up to
   now (R2).
3. **Pomodoro segments.** These are the closed rows, each with its true span.
   Overlapping closed rows each get their own number.
4. **Gaps.** A gap is a maximal stretch that no row covers (closed spans plus wall
   coverage), lying between the earliest row start and the latest closed-row end. A gap
   shorter than 5 minutes is slack: it gets no number, and its neighbours count as
   adjacent (R4).
5. **Numbering.** Pomodoro segments and numbered gaps are ordered by end time, newest
   first. Ties go to the later start first, then the later line first. `~1` is always
   the closed row that ended last.
6. **Tail** (`~`). The tail exists only when some closed row exists. Its start is the
   latest closed-row end (`~1`'s end). Its end is the smallest of `T` and, for each
   wall, `max(wall.start, tail.start)`. A wall that already covers the tail's start
   therefore empties it. The tail keeps its name even when it is shorter than 5 minutes,
   and it keeps true off-grid minutes.
7. **Head** (`~~`). The head exists when at least one row exists and the earliest row
   start is greater than 0. It spans `[0, earliest row start)`.

The worked example used everywhere below has `BOB_NOW` 09:13 and `T` = 09:15. PLAN
(07:30–08:00) and BUILD (08:30–09:00) are closed, and CHORE then READ are planned
untimed placeholders:

```text
          head                 ~3 PLAN         ~2 gap          ~1 BUILD     tail
   |.......................|###############|...............|###############|.........|
  0000                    0730            0800            0830            0900      0915 = T
 ends here:                ~~              ~3+             ~2+             ~1+      x=
 starts here:              ~3              ~2              ~1              ~
```

### Placement and length (R5)

Each suffix resolves to a **pin**. The pin records its edge (`start` or `end`), the
minute `at` it pins, and the segment's **far edge** (the other end of the segment).
Start pins also carry a **forward limit**:

| Suffix | Edge  | `at`              | Far edge                         |
| ------ | ----- | ----------------- | -------------------------------- |
| `~`    | start | tail start        | tail end (`T` or a wall's start) |
| `~n`   | start | segment _n_ start | segment _n_ end                  |
| `~n+`  | end   | segment _n_ end   | segment _n_ start                |
| `~~`   | end   | head end          | 0 (midnight)                     |

- The **forward limit** of a start pin is `T`, or the start of the first wall that
  begins at or after `at`, whichever comes first. Its cut kind is `now` or `wall`.
- **Requested length.** A typed count requests `5·N` minutes. An untyped count requests
  25 minutes.
- **A typed count is exact.** It may run past its segment into a neighbour. The existing
  guard then decides (`!` trims one edge, and erase, split, or open overlap refuses).
- **An untyped count stops at the far edge.** Its length is
  `min(25, room to the far edge)`. When that room is ≤ 0, the item refuses (it never
  becomes a zero-length record).
- **Nothing records past now.** In a start-pinned run, a window whose end passes the
  forward limit is cut to that limit. A member that would _start_ at or after the limit
  refuses. This also applies to typed counts (`x=read4~` at 09:13 records 09:00–09:15).
- **Nothing starts before midnight.** In an end-pinned run, a start below 0 refuses
  through the midnight gate, which names the largest count that fits. An end pin later
  than `T` refuses.
- **Every cut is reported.** `cut` is `now` or `wall` when the forward limit bound the
  window, and `segment` when an untyped count stopped at the far edge. In the tail the
  far edge _is_ the forward limit, so tail cuts read `now` or `wall`.
- **Recorded minutes are the true minutes** of the final window (`end − start`, which
  may be off the 5-minute grid). That value appears in `[t:: Nm]` and `window.minutes`.
  The lexical `blocks` keeps its meaning.
- **Unchanged rules.** The record still closes at `C = min(window end, now)`, and Work
  Log and Event Task clocks use `C`. Target resolution, chronological filing, the trim
  rewrite, and close effects are all unchanged. The suffix never selects tasks from the
  session it carves.

### Runs hang from one pin (R8)

A maximal run of consecutive `x=` items remains one contiguous block in typed order.

- **No suffix in the run.** The run ends at `T`, byte-for-byte as today.
- **A start suffix (`~`, `~n`) on the run's first item** pins its start. Members flow
  forward in typed order, each untyped member capped at the pinned segment's far edge,
  and the whole run respects the forward limit.
- **An end suffix (`~n+`, `~~`) on the run's last item** pins its end. Members flow
  backward in reverse typed order, each untyped member capped at the far edge.
- **A suffix anywhere else, or two suffixes in one run,** is a lexical error
  (`invalid_pomodoro_past`) on the offending item. It names the item, states the rule,
  and gives a concrete fix:
  - ``x=b~2 can't place this run: a run of x= items hangs from one place, so a start suffix (~, ~n) goes on its first item and an end suffix (~n+, ~~) on its last; write `x=a~2 x=b` or capture them separately``
  - `x=b~1! adds a second place to this run: a run of x= items hangs from one place; keep one suffix or capture them separately`

Because every run stays contiguous, `preflight_overlaps` keeps its continuous-union
logic unchanged.

### Snapshot (R7)

Each pinned run resolves its suffix against the staged snapshot that the overlap guard
already reads (earlier items in the same draft are staged). All suffixes in one run
share that snapshot, so numbers never move within a run. As a result, `x x=2~1!` closes
the running session and then carves the first ten minutes of _that_ session.
`capture-complete` reads the vault, not the staged draft. When the two differ, the
dry-run card is authoritative.

### Safety: `!` keeps exactly its meaning (R6)

- `!` is required exactly when a window overlaps closed history and is an error
  otherwise. It trims one edge, never erases or splits, and never touches an open row.
- Carving into a Pomodoro segment always needs `!`. A gap never does unless a typed
  count overruns it.
- One addition: **`!` never trims a row this capture already recorded.** A later run
  that would trim, erase, or split a row recorded by an earlier run in the same capture
  refuses, with this text:
  `x=1~1! would trim CHORE 0800-0810, which this capture just recorded; capture them separately`
- **Pinned-run hints** extend the existing guard messages:
  - A typed overflow without `!` adds `or fit the gap: x=6~2`, using the largest count
    that fits the pinned segment (≥ 1).
  - An erase of the pinned Pomodoro segment adds `carve at most 5 blocks: x=5~1!`, using
    the largest count that leaves a nonempty piece (≥ 1).

### Acceptance examples

All examples use the day above (`BOB_NOW=2026-11-04 09:13`). The six requested examples
must match **exact ledger bytes**, and `--dry-run` must report the same windows as the
real run.

| Draft         | Records                                            | Side effects                        |
| ------------- | -------------------------------------------------- | ----------------------------------- |
| `x=~=!1`      | CHORE 0900–0915 (15m, 25m cut at now), closed 0913 | completes CHORE task 1              |
| `x=read2~=!1` | READ 0900–0910, closed 0910                        | completes READ task 1               |
| `x=2~1!=!1`   | CHORE 0830–0840                                    | BUILD → 0840–0900; completes task 1 |
| `x=2~1+!=!1`  | CHORE 0850–0900                                    | BUILD → 0830–0850; completes task 1 |
| `x=2~2=!1`    | CHORE 0800–0810                                    | completes task 1                    |
| `x=2~2+=!1`   | CHORE 0820–0830                                    | completes task 1                    |

Cases the request left open:

| Draft                                    | Result                                                                             |
| ---------------------------------------- | ---------------------------------------------------------------------------------- |
| `x=~~`                                   | CHORE 0705–0730                                                                    |
| `x=~2`                                   | CHORE 0800–0825 (default fits the 30-minute gap)                                   |
| `x=6~2`                                  | 0800–0830, filling the gap                                                         |
| `x=8~2`                                  | refuses: takes 10m from BUILD; `add ! to take it: x=8~2!`, `or fit the gap: x=6~2` |
| `x=8~2!`                                 | 0800–0840; BUILD → 0840–0900                                                       |
| `x=~1`                                   | refuses: would take 25m from BUILD; add `!`: `x=~1!`                               |
| `x=~1!`                                  | 0830–0855; BUILD → 0855–0900                                                       |
| `x=6~1!`                                 | refuses: would take all of BUILD; `carve at most 5 blocks: x=5~1!`                 |
| `x=2~2!`                                 | refuses: takes no time from another session; drop the `!`                          |
| `x=read4~`                               | READ 0900–0915 (15m, 20m cut at now)                                               |
| `x=~` with WORK running since 0905       | CHORE 0900–0905 (5m, 25m cut at WORK)                                              |
| `x x=2~1!` with BUILD running since 0830 | `x` closes BUILD at 0913; CHORE 0830–0840; BUILD → 0840–0913                       |
| `x=read2~2 x=fix`                        | READ 0800–0810; FIX 0810–0830 (20m, 25m cut to fit ~2)                             |
| `x=email1 x=read2~2+`                    | EMAIL 0815–0820; READ 0820–0830                                                    |
| `x=~4`                                   | refuses and lists the segments (below)                                             |
| first row at 0020, `x=~~`                | 0000–0020 (20m, 25m cut to fit ~~); `x=5~~` refuses at the midnight gate           |

### Teaching errors

Lexical errors are owned by `grammar`, and timeline errors are owned by `record`:

| Typed                                          | Teaches                                                                                                                                                                    |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x=~0`, `x=3~0`                                | `~` already names the tail: `x=~`, `x=3~`                                                                                                                                  |
| `x=~+`, `x=~0+`                                | to end now write `x=`; to fill since your last Pomodoro write `x=~`                                                                                                        |
| `x=~~+`, `x=~~2`                               | `~~` already ends at your first Pomodoro and takes no number: `x=~~`                                                                                                       |
| `x=2~01`                                       | `x=2~1`                                                                                                                                                                    |
| `x=2~123`, `x=2~0800`                          | reserved for a future clock suffix                                                                                                                                         |
| `x=~2fix`, `x=~2#deep-work`                    | the suffix comes last in `E`: `x=fix~2`, `x=#deep-work~2`                                                                                                                  |
| `x=2~1~2`, `x=2~~1`                            | one segment suffix per item                                                                                                                                                |
| `x=3~1!+`                                      | `+` belongs to the suffix, before `!`: `x=3~1+!`                                                                                                                           |
| `x=2~1,3`                                      | a comma list is a drop: `x=2=~1,3` (existing teaching kept)                                                                                                                |
| `x=2-1`                                        | x= takes no offset; to start at segment 1 write `x=2~1`, to end now write `x=2`                                                                                            |
| `x=-`, `x=3-`                                  | x= takes no offset; to start where your last Pomodoro ended write `x=~` (`x=3~`), to end now write `x=` (`x=3`)                                                            |
| `x=deep-work`, `x=deep~work`                   | `x=#deep-work`                                                                                                                                                             |
| run errors                                     | see [Runs hang from one pin](#runs-hang-from-one-pin-r8)                                                                                                                   |
| no closed row, with `~`/`~n`/`~n+`             | `x=2~1 counts back from your last closed Pomodoro, but nothing is closed today yet; write x=2 to end now` (adds `, or x=2~~ to end where WORK starts` when any row exists) |
| empty tail                                     | `x=~: nothing to record between BUILD (ended 0900) and WORK (running since 0900)` (or `… and now (0915)`)                                                                  |
| no head                                        | `x=~~: PLAN starts at 0000, so there is no time before your first Pomodoro`; no rows: `nothing is on today's timeline yet; write x= to end now`                            |
| index out of range                             | `x=~4: today has 3 segments: ~1 BUILD 0830-0900, ~2 gap 0800-0830, ~3 PLAN 0730-0800; the time before 0730 is ~~`                                                          |
| untyped member with no room                    | `x=fix has no room left in ~2 gap 0800-0830; give it a count (x=fix2) or capture it separately`                                                                            |
| member starts at or after the limit            | `x=fix would start at 0915, after now; shorten the run` (or `after WORK started (0905)`)                                                                                   |
| bash-expanded `~` (argv starts with `x=$HOME`) | `your shell expanded ~; quote the capture: bob capture 'x=~'`                                                                                                              |

### Labels: one source for the picker, errors, and human output

The `timeline` module owns label text so that every surface agrees. Times use the
existing ASCII `HHMM-HHMM` style:

- Pomodoro: `~1 BUILD 0830-0900` (unnamed rows say `session`)
- Gap: `~2 gap 0800-0830`
- Tail: `~ since BUILD 0900-0915`
- Head: `~~ before PLAN 0000-0730`

### Wire contract

All changes are additive. `schema_version` stays 1, and the frozen `capture-*` names are
unchanged. **Unsuffixed drafts produce byte-identical JSON**, because every new key is
omitted when it does not apply.

**`capture-parse`** (lexical):

- `pomodoro_past.segment` takes one of three shapes:
  - `{"raw":"~2+","kind":"index","index":2,"edge":"end"}`
  - `{"raw":"~","kind":"tail","edge":"start"}`
  - `{"raw":"~~","kind":"head","edge":"end"}`
- `pomodoro_past.stack.pin` (`"start"` | `"end"`) and `stack.lead_blocks` (the sum of
  earlier members' blocks) appear only for runs that carry a suffix. `offset_blocks`
  keeps its meaning.
- A new span kind, `pomodoro_past_segment`, covers the suffix bytes from `~` through an
  optional `+`.

**`capture` / `--dry-run`** (resolved):

- `pomodoro_past.segment` appears on the member that carries the suffix:
  `{"raw":"~1","kind":"pomodoro","index":1,"edge":"start","start":"0830","end":"0900","minutes":30,"name":"BUILD","line":4}`.
  `kind` is one of `pomodoro`, `gap`, `tail`, or `head`. `index`, `name`, and `line`
  appear only when they apply.
- `pomodoro_past.stack.pin` appears for pinned runs.
- `pomodoro_past.window` gains `requested_minutes` and `cut` (`"now"` | `"wall"` |
  `"segment"`), both only when the window was cut. `anchor` stays `T`.
- The human header gains a segment clause and a cut note:
  - `would record CHORE 0830-0840 (10m) · 2026/20261104.md line 5 · at the start of ~1 BUILD 0830-0900`
  - `would record CHORE 0900-0915 (15m, 25m cut at now) · … · since BUILD`
  - The other clause forms are `at the end of ~2 gap 0800-0830` and `before PLAN`. The
    cut forms are `cut at WORK` and `cut to fit ~2`.

**`capture-complete`**:

- The new context is `pomodoro_past_segment`. Rows come in timeline order: tail (when
  nonempty), `~1…~N`, then head.
- Each row carries `replacement`, `label`, `kind`, `edge`, `start`, `end`, `minutes`,
  and `needs_repair` (true for `pomodoro` rows), plus `index`, `name`, and `line` when
  they apply.
- Accepting a row inserts the suffix only, never `!`.

**`capture-rewrite`**: the `expand_sigil_alias` rule gains a `-` arm (rule code
`expand_sigil_alias`).

**Internal**: `bob capture --expect-past-windows <HHMM-HHMM[,…]>` is a hidden internal
subprocess argument with no short alias.

### Compatibility and shipping

- `GRAMMAR_REVISION` goes from 13 to 14, and only when `record` makes suffixes record.
- Every claimed spelling (`x=~…`, `x=…~…`) was an error before, so commits are
  `feat(capture): …` without `!`. They ship on the daily release train, with no version
  or CHANGELOG edits.
- An older bob with a newer app reports the current teaching error and writes nothing. A
  newer bob with an older app still shows the right window and trims, without the
  segment line.

## Phase `timeline`: timeline catalog and pinned layout engine

Add `src/native/capture/pomodoro_timeline.rs` as `pub(crate)` and register it in
`src/native/capture/mod.rs`. It is pure: it takes contents and minutes and does no I/O.

1. **Move the row reader.** Move `TimedRow`, `parse_range_minutes`, and
   `collect_timed_rows` out of `src/native/capture/pomodoro_past.rs` into the new module
   (`pub(crate)`). Make `pomodoro_past.rs` import them. Behavior is identical, and the
   existing past tests must pass untouched.
2. **Model.**
   - Types: `SegmentKind { Pomodoro, Gap, Tail, Head }`; `Edge { Start, End }`;
     `Segment { kind, index: Option<u32>, start: i64, end: i64, name: Option<String>, line: Option<usize> }`
     (minutes in the 0..=1440 frame); `Wall { start, name, line }`.
   - `Timeline { now, t, rows, walls, numbered: Vec<Segment>, tail: Option<Segment>, head: Option<Segment>, tail_wall: Option<Wall> }`.
   - `Timeline::build(contents: &str, now_min: i64, t: i64) -> Timeline` implements
     [The timeline model](#the-timeline-model) exactly.
3. **Resolution.** `SegmentQuery { Tail, Start(u32), End(u32), Head }` is the module's
   own input type, mapped from the lexical suffix later by `record`.
   `Timeline::resolve(&self, query) -> Result<Pin, TimelineError>` returns
   `Pin { segment, edge, at, far_edge, forward_limit, limit: LimitKind::{Now, Wall(Wall)} }`.
   `TimelineError` is structured (`NothingClosed { first_row: Option<(name, start)> }`,
   `EmptyTail { after: Segment, wall: Option<Wall> }`,
   `NoHead { first: (name, start) }`, `NoRows`, `OutOfRange { count }`) so `record` can
   format the teaching table with the raw token.
4. **Layout.**
   `layout_run(pin: &Pin, members: &[MemberLength]) -> Result<Vec<Laid>, LayoutError>`.
   - `MemberLength { requested_minutes: u64, typed: bool }`.
   - `Laid { start, end, requested_minutes, cut: Option<Cut> }`, with
     `Cut { Now, Wall, Segment }`.
   - `LayoutError { NoRoom { member }, PastLimit { member }, BeforeMidnight { member, max_blocks }, EndPinAfterNow }`.
   - It follows [Placement and length](#placement-and-length-r5): start pins flow
     forward and end pins flow backward.
5. **Labels and listing.** `Segment::label()` and `Timeline::listing()` produce the
   [label](#labels-one-source-for-the-picker-errors-and-human-output) and out-of-range
   listing text. `Timeline::picker_rows()` returns tail (nonempty only), `~1…~N`, then
   head.
6. **Dead code.** Until `record` wires the module, put
   `#![cfg_attr(not(test), allow(dead_code))]` at its top with a one-line comment;
   `record` removes it.
7. **Unit tests** (in-module or `src/native/capture/tests/`):
   - The example day: numbering, tail, head, and every pin for `~`, `~1..~3`,
     `~1+..~3+`, `~~`.
   - Edge cases: no rows; only future MEETs (rows at or after `T` ignored); back-to-back
     rows (no gap); a 2-minute sliver (unnumbered, neighbours adjacent); overlapping
     closed rows (each numbered, gaps from coverage); off-clock rows; a row starting at
     0000 (no head); a running wall before and at `T`; a wall covering the tail start
     (empty tail); an empty tail just after a close at `T`; a row ending at 0000 (minute
     1440); an off-grid MEET end (true minutes kept); a stale open row from early
     morning (no gaps, empty tail).
   - Layout: every acceptance-table window; every cut path (untyped at the far edge,
     typed at now, typed at a wall); `NoRoom`; `PastLimit`; the midnight gate with
     `max_blocks`; and both pin directions for runs of 2–3 members.

Gate: `just check` (or the repo's fmt + clippy + test equivalents) is green. There is no
user-visible change.

## Phase `grammar`: segment suffix lexer, pins, spans, and teaching errors

This phase works in `src/native/capture_language/` plus the parse wire. It never reads
the vault.

1. **Lexer** (`past.rs`).
   - After the count/name/trailing-count prefix and the retired-`@` check, when the rest
     starts with `~`, lex the suffix. It must consume the rest of `E`. Return
     `PastEValid.segment: Option<PastSegment>` and `segment_range` (relative, `~`
     through `+`).
   - Implement every lexical row of [Teaching errors](#teaching-errors). Every rewrite
     keeps the typed count, name, repair `!`, and close part, like the existing
     rewrites.
   - The old `rest.starts_with('~')` "drop links" teaching stays only for a comma list.
   - Update the dash teachings (`past_rest_error`) to the new two-reading copy. The
     dashed bare name keeps its `#` teaching, and `x=deep~work` (bare name, `~`,
     letters, and a valid `deep-work` selector) teaches `x=#deep-work`.
2. **Model** (`model.rs`).
   - `PastSegment { raw: String, kind: PastSegmentKind::{Tail, Index(u32), Head}, edge: PastEdge::{Start, End} }`
     serializes to the wire shapes above.
   - `PomodoroPastSpec.segment: Option<PastSegment>` (skipped when `None`).
   - `PastStackPos` gains `pin: Option<PastEdge>` and `lead_blocks: u32`. The wire omits
     both for unpinned runs.
3. **Stacks and pins.** Change `compute_past_stacks` to take each item's blocks plus
   optional suffix edge. It returns positions (`pin`, `lead_blocks`) and pin diagnostics
   `(item index, message)` using the two run messages in the contract, whose rewrites
   move the suffix text to the legal end member.
   - Execution (`draft.rs`): a diagnostic is a usage error on that item.
   - Editor (`editor_parse.rs`): it becomes an `invalid_pomodoro_past` diagnostic on
     that item.
   - Every other item's parse is unchanged.
4. **Spans.** In `editor_pomodoro.rs`, emit `SpanKind::PomodoroPastSegment`
   (`"pomodoro_past_segment"`, added in `editor_model.rs`) over the suffix bytes. Also
   update the execution-side builders in `item.rs` that construct `PomodoroPastSpec`.
5. **Parse wire.**
   - `src/native/capture_parse.rs`: add `PastWire.segment`, and give `PastStackWire` its
     optional `pin` and `lead_blocks`.
   - The human `format_pomodoro_past` shows the suffix.
   - Rewrite the `capture-parse --help` paragraph so that `~`/`~~`/`+` before the second
     `=` is described as the segment suffix (no longer only drop teaching), including
     the new span kind and the run rule.
6. **Interim execution refusal.** In `src/native/capture/plan.rs`, a past run with any
   suffixed member refuses (usage) with
   `x=2~1 places a past Pomodoro on today's timeline, which this build cannot record yet`.
   `record` deletes this. `GRAMMAR_REVISION` stays 13.
7. **Docs.** In `docs/capture.md` §Recording a past Pomodoro, update only the "Common
   teaching errors" table to the rows this phase owns. Drop the old
   `x=~2`/`x=3~2`/`x=build~2` drop row, keep the comma-list drop row, and replace the
   dash row. `record` writes the placement narrative.
8. **Tests.**
   - `src/native/capture_language/tests/past.rs`: a table covering every valid suffix
     shape (`x=~`, `x=3~`, `x=fix~2`, `x=fix2~2+`, `x=#"DEEP WORK"~1`, `x=#c++~1+`,
     `x=~~`, `x=2~1!=!1`) and every lexical teaching row.
   - `tests/cli/capture/parse_pomodoro_past.rs`: JSON `segment`, `stack.pin`,
     `lead_blocks`, the span ranges, run diagnostics (misplaced start, misplaced end,
     double pin), and proof that unsuffixed JSON is byte-identical.

## Phase `record`: record pinned runs end to end and ship grammar 14

1. **Wiring** (`src/native/capture/plan.rs`, `pomodoro_past.rs`).
   - For a run whose `stack.pin` is set:
     1. Read the pre-stack snapshot first, using the existing "today's daily note does
        not exist" error path.
     2. Build `Timeline` with the same `now`/`T` as `compute_preflight`.
     3. Map `PastSegment` to a `SegmentQuery`, resolve it, and run `layout_run` with
        each member's requested length and typed flag.
     4. Format every `TimelineError`/`LayoutError` into the teaching copy using the
        member's raw token and the existing `capture_error_context` prefix.
     5. Build `MemberWindow`s with true minutes.
   - Unpinned runs keep the current order and code path exactly: preflight before the
     snapshot, with the same errors and bytes.
   - Factor `compute_preflight` so both paths share the `MemberWindow` formatting.
   - Delete the `grammar` interim refusal and the `timeline` dead-code allow.
2. **Guard additions.**
   - Add the pinned-run hints (`or fit the gap`, `carve at most N blocks`) to
     `preflight_overlaps` messages.
   - Add the same-capture rule: the batch keeps the windows recorded by earlier runs in
     this capture. A closed row whose span equals one of them refuses any trim, erase,
     or split with the contract copy.
   - The guard logic is otherwise untouched.
3. **Output** (`src/native/capture/output.rs`).
   - `PomodoroPastSummaryJson.segment` is resolved from the pin's `Segment`.
   - Add `PomodoroPastStackJson.pin` and
     `PomodoroPastWindowJson.{requested_minutes,cut}`, all `skip_serializing_if`.
   - In `print_human_pomodoro_past_success`, add the segment clause and cut note.
4. **Grammar.** Set `GRAMMAR_REVISION = 14` (`src/native/capture_language/mod.rs`).
5. **Docs.** First read the `docs.md` and `cli_rules.md` reference memories.
   - `docs/capture.md` §Recording a past Pomodoro:
     - The shape line becomes `x=E[s][!][=X]`.
     - Replace the now-false sentence "every past Pomodoro ends at the same boundary".
     - Add a "Placing it on today's timeline" subsection with the picture, the suffix
       table, the length rules, runs and pins, the snapshot rule, the empty-day
       refusals, both example tables, and a note to single-quote captures in shells
       (`bob capture 'x=~'`).
   - Update the session-chain shape (`x=E[!][=X]` → `x=E[s][!][=X]`) and
     `docs/getting-started.md` if its sentence becomes false.
   - `docs/reference/json-outputs.md` gets the new fields, and
     `docs/reference/compatibility.md` gets grammar 14.
   - Gate on `just docs-check`.
6. **Tests.** Add `tests/cli/capture/pomodoro_past_segment.rs`, registered in
   `tests/cli/capture/mod.rs` and using the helpers and fixture style of
   `pomodoro_past.rs`.
   - The six requested examples as exact ledger bytes plus JSON, with the dry run equal
     to the real run.
   - Every row of the open-cases table and every timeline teaching row.
   - Runs with both pins, and number stability within a run.
   - The `x x=2~1!` snapshot.
   - The same-capture refusal.
   - Whole-batch byte-identical rollback when a later member fails.
   - Every existing `pomodoro_past.rs` test still passes unchanged (the unsuffixed
     byte-identity proof).

## Phase `assists`: segment picker, dash-to-tilde rewrite, and submit guards

1. **Completion field** (`src/native/capture_language/completion.rs`).
   - Add `CompletionContext::PomodoroPastSegment`.
   - In the `x=` branch, a cursor inside the suffix (from its `~` through digits, `+`,
     or a second `~`, before `!` or the second `=`) yields a field whose replacement is
     the suffix range. The query is the text after the first `~` up to the cursor.
   - The name field keeps ending at `~`.
2. **Candidates.**
   - In `src/native/capture_complete/`, add `Candidates::PastSegment` and the render
     context name `pomodoro_past_segment`.
   - In `pomodoros.rs`, `pomodoro_past_segment_candidates` reads today's day file with
     the same `BOB_DAY_FILE`/date resolution as `pomodoro_candidates_inner`. It builds
     `Timeline` at `current_datetime()` and maps `picker_rows()` to the
     [wire rows](#wire-contract).
   - Filtering rules:
     - An empty query lists every row.
     - `~` lists only the head.
     - Digits prefix-match `index`.
     - Digits followed by `+` keep the exact index as an `edge: end` row with a `~n+`
       replacement, and drop the tail and head rows.
   - Add the engine dispatch.
   - Add a zsh arm in `shell.rs` that shows each label as the description.
   - Update the `capture-complete` help paragraph in `capture_complete/cli.rs`.
3. **Rewrite** (`src/native/capture_language/rewrite.rs`).
   - Add a `-` arm to `expand_sigil_alias`. It swaps the typed `-` for `~` only when the
     swapped text opens the `PomodoroPastSegment` field at the cursor. For a new `~`,
     the field starts at the swapped byte. For `x=~-` → `x=~~`, the field starts at the
     prior `~`.
   - It never fires when the `-` extends a plain `#name`, because `-` is name content
     there.
   - Required swaps: `x=-`, `x=2-`, `x=fix-`, `x=fix2-`, `x=#"DEEP WORK"-`, `x=~-` →
     `x=~~`, and `x=2~-` → `x=2~~`.
   - Required non-swaps: `x=#deep-`, `x=~2-`, `x=2!-`, `x=2=-`, `x=3 -2`, a `- bullet`
     line, prose, and `=2-` (the start offset).
4. **Stale-preview guard.**
   - Add a hidden `--expect-past-windows <LIST>` to `bob capture`
     (`src/native/capture/cli.rs`, alphabetical, `.hide(true)`, no short alias) and
     plumb it on the request.
   - After each past run's windows resolve, pinned or not, compare them in draft member
     order with the next entries, before staging trims.
   - A mismatch, a count mismatch at the end, or a malformed list refuses through a new
     `CaptureError` constructor, modeled on `override_refused`: I/O class (exit 1), code
     `pomodoro_past_windows_changed`, message
     `your timeline changed since the preview: x=2~1! now means 0900-0910`.
   - The CLI behaves identically without the flag.
5. **Shell-expanded tilde.** In `src/native/capture/shell_guard.rs` (argv drafts only,
   before planning): when a draft token starts with `x=` followed by the non-empty
   `$HOME` value, refuse with code `shell_expanded_tilde` and the contract message. Its
   rewrite puts `~` back in place of `$HOME`. Document that a `pushd`-style `x=~1`
   expansion cannot be detected.
6. **Docs.**
   - In `docs/capture.md`, add the picker and the `-` → `~` typing assist to the
     placement subsection and to the editor-assist docs where the `.`/`;`/`,` aliases
     are described (find them with grep).
   - Update `docs/reference/json-outputs.md` for the completion context and error codes.
   - Run `just docs-gen` if help output changed, then `just docs-check`.
7. **Tests.**
   - Field ranges and queries.
   - Candidate rows for the example day, including `+`, `~`, and digit filtering, plus
     empty days.
   - zsh output.
   - Every required swap and non-swap.
   - The expect guard: match, mismatch, count mismatch, malformed value, omitted flag.
   - The `$HOME` guard, positive and negative.

## Phase `decision`: decision record for segment placement

> [!decision] decision_record

Use `/sase_memory_write` first. Create
`sase/memory/decisions/past-pomodoros-hang-from-segment-edges.md` in the format of its
siblings (frontmatter: `keyword`, `aliases`, `summary`, `metadata.status: accepted`,
`metadata.decided: 2026-11-05`; body: Applies to, Claim, Why, Cost, Reopens when,
Evidence).

- **Claim:** A past Pomodoro may be placed by a segment suffix that counts closed rows
  and gaps back from now. A run hangs from one pin, typed counts are exact, untyped
  counts stop at the segment's far edge, nothing records past now, and `!` still only
  trims one edge.
- **Why:** List the rejected alternatives from the research's _Alternatives considered_:
  absolute clock (kept as a follow-up), name-relative anchors, separate numbering, the
  `@` sigil, a canonical or executed `-`, a Swift-side dash rewrite, independent
  placement in runs, and untyped-fills-segment.
- **Evidence:** the research ref, this plan, and the `record` commit.
- **Links:** `[[mac-capture-is-a-thin-client]]` and
  `[[capture-sigils-say-where-what-how]]`.

Then run `sase memory init`.

> [!decision] decision_record = no

Make no memory edit. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead instead.

## Phase `mac`: Bob Mac Capture segment typing, picker, card, and guard

Open `bob-mac-capture` with `/sase_repo`, and read its `AGENTS.md` first (compile rules,
mac typecheck, `Scripts/macos-ci-check`, and watching the exact-SHA macOS run to green
with `/sase_monitor` after every push). Install a bob built from this epic's master for
fixtures.

1. **Typing.**
   - Add `-` to both rewrite trigger allowlists: the key-down guard in
     `Sources/BobMacCapture/CapturePanelController.swift` and
     `CapturePanelModel.noteSeparatorToggleIntent`. It is ungated, like `.` and `,`, and
     bob decides.
   - Verify (and test) that the swap is its own undo step, so ⌘Z restores the dash.
   - Verify that rapid `--`, and `-1+` followed by an immediate Return, never let a
     stale rewrite reply edit a newer draft, and that pending rewrites settle before
     submit. Extend the existing generation and draft checks if a gap shows.
   - IME marked text stays untouched.
2. **Highlighting.** Add a `pomodoro_past_segment` style in
   `CaptureEditorPalette.swift`. It belongs to the `pomodoro_past` family and is visibly
   distinct from `pomodoro_close_drop`, so that `~1` before the second `=` never looks
   like a drop.
3. **Picker.**
   - Add a `pomodoro_past_segment` context in `CompletionRowContent.swift` and
     `CaptureCompletionMenu.swift`.
   - Row layout: `replacement` in monospace, `label`, `start–end`, `minutes`, and a text
     badge (`tail`, `gap`, `head`, or `needs !`). Use theme colour for Pomodoro rows and
     a lighter outlined treatment for gaps, always paired with text.
   - Accept inserts bob's replacement unchanged.
4. **Models and card.**
   - Decode `segment`, `stack.pin`, `window.requested_minutes`, and `window.cut` with
     `decodeIfPresent` (`CaptureModels.swift`).
   - `CapturePomodoroPastPresentation` gains a segment line
     (`at the start of ~1 BUILD 0830–0900`, `at the end of ~2 gap 0800–0830`,
     `since BUILD`, `before PLAN`) and a cut note (`15m · 25m cut at now`), both with
     accessibility labels. Do not add a new panel.
5. **Stale-preview guard.**
   - When the latest `capture-parse` grammar is ≥ 14 and the preview held
     `pomodoro_past` members, submit with `--expect-past-windows` built from the preview
     windows in member order. This is a positive capability gate, because older bobs
     file unknown flags as task text.
   - On error code `pomodoro_past_windows_changed`, keep the draft, show bob's message,
     and re-run the dry-run preview.
   - The `CaptureGrammarGate` minimum stays 5.
6. **Fixtures and docs.**
   - Regenerate and extend fixtures with `Scripts/generate-pomodoro-past-fixtures.sh`
     (segment parse, segment preview with a trim, a cut preview, segment completion, and
     the windows-changed error).
   - `docs/protocol.md`: add a grammar 14 row, the new fields and context, the `-`
     trigger, and the expect guard.
7. **Gates.**
   - Run `just portable-check`.
   - Run the mac scratch typecheck, `bundle.sh`, and `just typecheck-tests` per
     `AGENTS.md`.
   - Run `Scripts/macos-ci-check` and `just docs-check`, then push and watch macOS CI to
     green. If the mac is offline, say the macOS targets are unverified.

## Verification across the epic

- After `record`: run `bob capture --dry-run -f json` on each of the six requested
  examples, using a fixture day at `BOB_NOW=2026-11-04 09:13`. It reports the contract
  windows, and the real run writes the same bytes the tests pin.
- After `assists`: `bob capture-rewrite --cursor` on `x=2-` returns `x=2~`, and
  `bob capture-complete` at `x=~|` lists `~ since BUILD …`, `~1 BUILD …`, `~2 gap …`,
  `~3 PLAN …`, and `~~ before PLAN …`.
- After `mac`: in the app, typing `x=2-` shows `x=2~` with the picker open. Choosing
  `~1` and typing `!` shows the card line `at the start of ~1 BUILD 0830–0900` with the
  BUILD trim row, and Return records it.
