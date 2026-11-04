---
tier: epic
title: Record past Pomodoros with x=
goal: 'Capture gains `x=<E>[!][=<X>]`, which records already-finished Pomodoros in
  one step. Each token starts and closes a past session that ends at the current 5-minute
  block. Consecutive tokens stack back to back. A token never overlaps history, except
  that `!` trims an edge of an older closed row and warns. bob previews, completes,
  and documents the token, and the Mac app renders it.

  '
phases:
- id: grammar
  title: x= lexer, capture-parse contract, and grammar 12
  depends_on: []
  size: large
  description: 'grammar: claim `x=` as a new whole-item session token. This covers
    the lexer, the PomodoroPast model, teaching errors, chain ownership, stack numbering,
    editor and capture-parse output, and grammar 12. Execution refuses the token until
    record lands.'
- id: complete
  title: Pomodoro-name completion inside x=
  depends_on:
  - grammar
  size: small
  description: 'complete: capture-complete offers Pomodoro names at the `<E>` name
    positions of `x=`. It never offers the running session, and replacements stop
    at `!`, `=`, and `@`.'
- id: record
  title: Record engine with explicit-entry close, placement, and output
  depends_on:
  - grammar
  size: large
  description: 'record: execute `x=` stacks. Compute windows, resolve targets, and
    close the explicit entry at C = min(end, now). File the row chronologically, carry
    links only when the close carries them, and add human and JSON output. A strict
    guard refuses every overlap.'
- id: guard
  title: Whole-stack overlap guard and ! edge repair
  depends_on:
  - record
  size: medium
  description: 'guard: replace the strict guard with the full per-token `!` rules.
    Running sessions are protected, the whole stack is validated at once, erase and
    split are refused, and closed-row edges are trimmed with warnings. Pin the user''s
    five examples end to end.'
- id: docs
  title: Capture docs, references, and help for x=
  depends_on:
  - guard
  - complete
  size: medium
  description: 'docs: document recording a past Pomodoro in docs/capture.md and the
    syntax, JSON, and compatibility references. Polish CLI help and regenerate the
    generated pages.'
- id: mac
  title: Bob Mac Capture Record past work card
  depends_on:
  - docs
  size: large
  description: 'mac: decode `pomodoro_past` in Bob Mac Capture and render one chronological
    Record past work card per stack. Style the new spans, and re-run the preview when
    the clock crosses a 5-minute boundary.'
decisions:
  memory_decision_record:
    ask: Add a decisions-web record that past Pomodoros end at now, never touch open
      sessions, and ! only trims edges?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.38.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 11:11:54
status: wip
bead_id: bob-cli-h2
---

- **PROMPT:** [prompts/202611/past_pomodoro_recording.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/past_pomodoro_recording.md)
- **BEAD:** [bob-cli-h2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h2/README.md)

# Plan: Record past Pomodoros with `x=`

## Context

Today the only way to log work that already happened is the two-step `=N-N x`. The
research report `research:202611/recording_past_pomodoros/recording_past_pomodoros.md`
found five problems with it:

- the user has to do mental arithmetic for every stack;
- it restarts a running session;
- it writes overlapping history without warning;
- earlier stack members close at wall-clock time, which prints "ran over" and
  misclassifies Event Tasks;
- a bare `=build x` writes a zero-length row.

Bryan accepted all of the report's requirement adjustments (A1–A9). This plan adopts the
report's recommended answers to its open questions:

- stacks span consecutive items;
- Work Log stamps use the session's own clock C;
- `x=…=0` closes and defers;
- `!` is checked per token;
- calendar MEET rows may be trimmed, and the preview flags it.

Mnemonic for docs and help: _`=` starts now, `x` stops now, `x=` records what already
happened._ Inside the token, everything before the second `=` is time and everything
after it is tasks.

Ground rules for every phase:

- Run `just check` (fmt, clippy, all tests) before declaring done.
- Never edit the Cargo.toml version or CHANGELOG.md (decision
  `bob-cli-releases-ride-a-daily-train`).
- The grammar phase's commit subject is `feat(capture)!: …` because it changes the
  meaning of existing text.
- Any phase that changes clap help or `capture-parse` help regenerates the help goldens
  and `docs/reference/cli/*` with `just docs-gen` in the same commit.
- bob is the only implementation of the grammar (decision
  `mac-capture-is-a-thin-client`).

## Design contract (shared by every phase)

### Grammar

```text
past  := "x=" E [ "!" ] [ "=" X ] [ SP inline-entry ]
E     := "" | D | name | "#" name | "#" literal | D "#" name | D "#" literal
         | name "@" D | "#" name "@" D | "#" literal "@" D
D     := [0-9]+          -- block count ≥ 1; omitted means 5 (25 minutes)
X     := exactly what may follow a whole-item `x` today:
         [N][*P][!M][~K][@N] in any order, with the same wildcards, lexer
         (`lex_close_selection`), and diagnostics
```

- **Where `x=` is recognised.** It is lowercase only. It must start an item, or be a
  token inside a same-line session chain. `X=3` and mid-body `Plan x=3` stay prose.
  Every token that starts with `x=` is claimed: it is either valid or an
  `invalid_pomodoro_past` diagnostic, never prose.
- **Names follow the `=name` rules.** Bare names must start with an ASCII letter and
  must not be close-shaped (`bare_start_name_len`). Names that start with a digit need
  `#` (`x=#2fa`). `#"EXACT NAME"` literals (straight or curly quotes) match by key
  equality.
- **Names stop at `!`, `=`, `@`, `~`, and whitespace.** Neither `!` nor `=` is a
  Pomodoro name byte (`is_pomodoro_selector_byte`, `tokens.rs`). Add a past-specific
  extent function rather than changing `start_name_extent`, which `=`, `=/`, and
  completion share. Quoted literals still run through their closing quote.
- **Give `<E>` its own small lexer.** Do not lex it by passing `"=" + rest` to
  `session_equals_token`. That function would turn `=!`, `=*1`, `=*build`, `==`, and
  `=/` into retired closes, merges, doubled starts, or splits. Reuse the leaf helpers
  instead: digits, `lex_exact_name_after_hash`, `bare_start_name_len`, and
  `canonicalize_exact_name`.
- **The repair `!` binds to `<E>` (A1).** Placing it there keeps the close's own `!`
  wildcard intact, so `x=3=1!` still means "keep 1, complete the rest".
- **Everything after the second `=` goes to the close lexer unchanged.** That includes
  its incomplete states (`x=3=1,` needs `pomodoro_close_task`).
- **An `x=` token is a close for chain purposes.** It can own an inline Work Log entry
  (`x=2#fix=*3 apollo agent`) and child bullets, under exactly the existing close-log
  rules (positional, numbered, star bullets, and the inline/bullet mixing error). Child
  bullets attach to the line's last close, which is now possibly an `x=`.

Token table (claimed tokens; each meaning assumes 09:21, where T = 09:25):

| Typed                             | `<E>`     | Repair | `<X>`       | Meaning                                         |
| --------------------------------- | --------- | :----: | ----------- | ----------------------------------------------- |
| `x=`                              | default   |   –    | plain       | next placeholder, 0900–0925                     |
| `x=3`                             | 3         |   –    | plain       | 0910–0925                                       |
| `x=3!`                            | 3         |   ✓    | plain       | 0910–0925, may trim older closed history        |
| `x=build`, `x=#build`, `x=build=` | BUILD     |   –    | plain       | BUILD 0900–0925                                 |
| `x=2#fix=*3`, `x=fix@2=*3`        | FIX, 2    |   –    | `*3`        | FIX 0915–0925, park task 3                      |
| `x==1!2`                          | default   |   –    | `1!2`       | keep 1, complete 2, defer the rest              |
| `x==!`                            | default   |   –    | `!`         | complete every link                             |
| `x=!` / `x=!=!`                   | default   |   ✓    | plain / `!` | repair; repair and complete everything          |
| `x=3!=1`                          | 3         |   ✓    | `1`         | keep 1, defer the rest, repair                  |
| `x==0`                            | default   |   –    | `0`         | record and defer every link (never a reset, A9) |
| `x=#"DEEP WORK"@3`                | DEEP WORK |   –    | plain       | literal name, 3 blocks                          |

### Teaching errors

Each error below is an `invalid_pomodoro_past` diagnostic in `capture-parse` and a usage
error from `bob capture`. Every message names the typed token and the exact rewrite. The
wording here is a target; implementers may polish it but must keep the rewrite.

| Typed                                                | Message                                                                                                                                                  |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x=3-1`, `x=-2`, `x=3-`, `x=build@3-1`               | ``a past Pomodoro always ends where the next one starts (or now), so it takes no offset; write `x=3` `` (the rewrite drops the dash part: `x=-2` → `x=`) |
| `x=3#fix@2`                                          | ``give the block count once: `x=3#fix` or `x=fix@3` ``                                                                                                   |
| `x=~2`, `x=3~2`, `x=build~2`                         | ``drop links in the close part after a second `=`: `x=3=~2` ``                                                                                           |
| `x=*3`                                               | ``to park task 3, put it after a second `=`: `x==*3` ``                                                                                                  |
| `x=0`, `x=build@0`                                   | ``a past Pomodoro needs at least one block: `x=1` ``                                                                                                     |
| `x=/…`, `x=*build` (merge-shaped)                    | ``x= records one past session; it can't split (`=/`) or merge (`=*`) ``                                                                                  |
| `x=1!2`, `x=!2` (bare count, then a glued selection) | ``write the task selection after a second `=`: `x==1!2` ``                                                                                               |
| `x=fix!2` (a name, then glued text after `!`)        | ``write the task selection after a second `=`: `x=fix=!2`; to also repair, write `x=fix!=!2` ``                                                          |
| `x=3x`, `x=3build`                                   | ``write `#` between the count and the name: `x=3#build` ``                                                                                               |
| oversized `D`                                        | the start family's "too large" wording                                                                                                                   |
| unterminated `x=#"DEEP`                              | an incomplete state (needs `pomodoro_name`), not an error                                                                                                |

Incomplete states:

- `x=#`, `x=3#`, and `x=#"DEEP` need `pomodoro_name`, with an `interactive_placeholder`
  span over `#`, exactly like `=#`.
- `x=build@` needs `pomodoro_start_timing`.
- Close-part dangles (`x==1,`, `x==~`) need `pomodoro_close_task`.
- Inline dangles (`x= 2`) need `pomodoro_close_log_text`.

Reuse these existing need names; add no new ones.

Compatibility breaks, which are why the commit is marked `!`:

- `x=3` was a prose task.
- `x x=3` was a close whose Work Log entry read `x=3`.
- `=3 x=3` was a start shape error.

All three now parse as past-record chains.

### Stacks

A **stack** is a maximal run of consecutive capture items whose kind is `PomodoroPast`.
Same-line chain tokens and blank-line-separated items both count, because chain tokens
already become consecutive items. Any other item or operator ends the stack. That
includes `x`, `=`, `+N`, prose tasks, links, and every other kind.

- `x x=3!`: the stack is `[x=3!]`.
- `x=3 =`: the stack is `[x=3]`, then a start at T with no gap.
- `x=email@1 x=build@3 x=fix@2`: one stack of 3. It reads as a 30-minute timeline typed
  in order.

The draft below is one stack of 2, and each session gets its own Work Log entry:

```text
x=build@2 wired the lexer

x=fix@3 apollo agent
```

Compute stack positions in one shared, pure post-split pass over the draft's items. Call
it from execution (`parse_ordinary_draft`, after the item loop in `draft.rs`) and from
the editor (`parse_for_editor_with` in `editor_parse.rs`), so `capture-parse` and
`bob capture` always agree. Each member records:

- `index`: 1-based, in typed order;
- `size`;
- `offset_blocks`: the sum of the blocks of every later member.

These are text-only facts.

### Windows and the anchor

- **Anchor.** Read the batch's `now` once (`bob_env::current_datetime()`, honouring
  `BOB_NOW`) and drop the seconds. The anchor is `T = ceil5(now)`, which is
  `start_block_minute` with offset 0.
- **Windows.** Member `i` covers `[T − 5·(nᵢ + offsetᵢ), T − 5·offsetᵢ)`, half-open. A
  single `x=3` therefore equals `=3-3` on the clock.
- **Order.** Windows are computed right to left from the text. Targets and task effects
  are then applied left to right, in typed order, so completion order, continuation
  selection, and Work Log order follow what was typed.
- **Close clock.** Each member closes at its own clock, `C = min(window end, now)`.
  Earlier members close at their own end. The last member closes at `now`, at most 4
  minutes before T. So `close_timing` never decrements, and no "ran over" text appears.
- **Midnight (A7).** If `T − 5·(total stack blocks) < 0`, refuse:
  `x=6 would start before midnight (2350); record at most 2 blocks today: x=2` (for a
  stack, name the stack and how far back it can reach). A window may end exactly at
  midnight (T = 1440 when now is 23:56–23:59). That end is written `0000`, as start
  ranges already wrap mod 1440, and every interval computation treats an end ≤ start as
  +1440, as Stats does.

### Targets

- **Unnamed tokens** take `next_future_pomodoro` (`capture_pomodoros.rs`): the first
  open, untimed placeholder. They never use the MEET calendar or start-block preference
  (`unnamed_start_target_with`, `apply_meet_rule`).
  - If there is no placeholder, refuse:
    `no future Pomodoro to record; name one: x=build`.
  - If that placeholder is a `MEET` (it holds calendar meetings, so recording it would
    classify every meeting `not_started` and log an empty MEET), refuse:
    `the next future Pomodoro is MEET (line 7); name the session you worked: x=build, or x=meet to record a meeting`.
- **Named tokens** resolve exactly like `=name`, through `select_capture_name` with
  `#"…"` literal semantics, ambiguity errors, and near-match `did you mean` warnings.
  - An open untimed placeholder is used in place (`source: "placeholder"`).
  - A completed match creates a new "again" row (`source: "again"`).
  - With no match, a new row is created through `insert_named_placeholder`
    (`source: "created"`).
  - If the name resolves to an open _timed_ row (the running session), refuse:
    `x=build names BUILD, which is still open (0900-0925, line 5); close it first: x x=build`.
- **Within a stack**, each member resolves against the staged state left by the members
  before it. So `x=2 x=3` records the same queue twice, and the second member takes the
  links the first one carried. The preview shows this; naming the sessions avoids it.
- **Explicit `x=meet`** uses the typed duration (default 25 minutes), never the event
  run (A8).

### Recording = closing an explicit entry at C

Do not route through the start path: no override, swap, or restart, no MEET rule, no
event-conflict warning, and no "move to current slot". Recording works like this:

1. Write the target's `()` as the canonical `(**HHMM-HHMM** [t:: Nm])` window
   (`replace_placeholder_range` or the created line).
2. Run the existing close engine on **that entry handle** at clock C:
   - Event Task classification uses S = window start and C.
   - Work Log stamps are taken at C (`work_log_stamp(C)`).
   - Close names narrow (R6), and carried names shed.
   - Successors are offered, never auto-linked.
   - Lanes and dependency recovery behave as for any close.
   - The clock key is restamped from counted links (`[t::]` or `[off::]`).
3. Never reset. `x==0` and every reset-shaped selection close and defer, whatever notes
   exist (A9).

**The close seam.** Only two calls look up the running session:
`plan_pomodoro_close_outcome` and `plan_pomodoro_close` in
`capture_pomodoro_close/linked_tasks.rs`. Everything below them already takes
`&RunningPomodoro` plus a `now`. The R6 type lookup (`close_types`, via `find_running`
in `capture/task_theme.rs`) is the third such lookup.

- Add entry-taking variants:
  `plan_pomodoro_close_for_entry(…, entry: &RunningPomodoro, now, …)`. Thread the entry
  through `close_types`, `build_close_summary_json`, and `stage_close_plan`.
- Keep the old wrappers, which call `find_running_pomodoro`, so `x` behaviour stays
  byte-identical.
- The explicit handle matters: during a record, a running session and the past row are
  both open timed rows. The global lookup would refuse that state as "two open timed
  entries".

### Placement and continuation

- **The closed row** is filed chronologically: after the last completed row whose start
  is ≤ the new start, else before the first completed row, else before the first open
  row, else at the section start.
  - Lift this rule out of the private `closed_insert_at` in `gcal/plan.rs` (built on
    `capture_pomodoros::scan`) into a shared ledger module. Have gcal call the shared
    copy too.
  - Completed rows stay before open ones, and no planned row ends up among completed
    rows.
  - Do the move after successors and the headline refresh, or remap their line numbers.
- **The continuation.** A record never plans the future: it creates a continuation only
  when the close carries links.
  - The carried links stay in a continuation **at the source placeholder's slot**, under
    the close's continuation name (after any shed).
  - When nothing carries, the used placeholder simply becomes the closed row, and no
    empty `- [ ] ()` is created even if no later open entry exists. Add a ledger-close
    option for this. `x` keeps its current "always leave a next entry" behaviour.
- **The running session**, if any, is never moved, retimed, or renamed.

### Guard and repair

- Times are half-open. Touching endpoints never conflict.
- Candidates are every timed row in today's `## Pomodoros`:
  - closed rows on either clock;
  - legacy rows with no stamp;
  - calendar MEET clusters;
  - every open timed row.
- Zero-length rows and untimed placeholders occupy no time.
- Rows in other day files are out of scope.

Check each member against history **as it stood before the stack** (after earlier batch
items), all members at once, before any member's effects:

| Situation (per member)                                                                                           | Without `!`                             | With `!`                                    |
| ---------------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------- |
| Overlaps nothing (or only touches)                                                                               | record                                  | **error:** unneeded `!`                     |
| Overlaps an open timed row                                                                                       | **refuse**; teach `x x=…`               | **refuse** (same)                           |
| Overlaps closed rows, and each keeps exactly one nonempty piece after subtracting the union of all stack windows | **refuse**; show the trim it would make | trim each affected row's edge, record, warn |
| The stack covers a closed row completely                                                                         | refuse                                  | **refuse:** would erase it                  |
| The stack would leave a closed row in two pieces                                                                 | refuse                                  | **refuse:** would split it                  |

- **`!` is per member (A4).** It is required exactly on the members whose window
  overlaps a pre-stack closed row, and it is an error on every other member.
  - One refusal lists every offending member, with one line each, so a single retry
    fixes the whole draft.
  - Evaluate the whole stack at once. Taken alone, `x=build@2!` against PLAN 0855–0920
    looks like a split. As part of `x=build@2! x=fix@3!`, it only trims PLAN to
    0855–0900.
- **A trim is a span rewrite only.**
  - Use `parse_adjustment_range` and `format_adjusted_range`
    (`capture/pomodoro_adjust.rs`).
  - The row keeps its clock key (`t` or `off`), name, checkbox, block ID, children, and
    links.
  - Close effects are never re-run.
  - The duration becomes the remaining piece's length, and the minutes taken are the old
    span length minus the new one.
  - Trims are staged before any member records, inside the same batch, so the whole
    batch rolls back on any failure.
- **MEET rows may be trimmed**, flagged `calendar: true` in JSON and `(calendar)` in
  human output. Synced MEET clusters are written once, but verify that `bob gcal sync`
  leaves a trimmed closed MEET row alone (see guard).

Messages (wording targets; every one spells out the fix):

| Case         | Message                                                                                                                                                                       |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| needs `!`    | `x=3 would take 10m from PLAN 0855-0920 (line 4); add ! to take it: x=3! (PLAN becomes 0855-0910)`                                                                            |
| unneeded `!` | `x=3! takes no time from another session; drop the !: x=3`                                                                                                                    |
| open row     | `x=3 overlaps WORK 0900-0925, which is still open (line 5); close it first: x x=3!` (`!` only when the open row has started, because then its closed span will still overlap) |
| erase        | `x=6! would take all of CALL 0920-0925 (line 6); ! only trims a session's edge`                                                                                               |
| split        | `x=build@2! would split PLAN 0855-0920 (line 4) in two; ! only trims a session's edge`                                                                                        |

### Output

**`capture-parse`** reports only text-level facts:

- **Mode.** The item mode is `pomodoro_past`, and `section` carries the typed name, as
  for named starts.
- **The `pomodoro_past` object.** It is added as both a top-level field and an item
  field. Its fields are `raw`, `name` (null when unnamed), `name_is_literal`, `blocks`,
  `minutes`, `repair`, `stack {index, size, offset_blocks}`, and `close`. The `close`
  field is the existing `pomodoro_close` spec shape for `<X>`, including `log` entries.
- **Spans.**

  | Span kind                | Covers                      |
  | ------------------------ | --------------------------- |
  | `pomodoro_past`          | `x=` plus a leading count   |
  | second `pomodoro_past`   | `@D`                        |
  | `pomodoro_name`          | the name only               |
  | `pomodoro_past_repair`   | the repair `!`              |
  | `pomodoro_close`         | the second `=`              |
  | existing close sub-spans | `<X>`, plus log-index spans |

- **Human line.** For example:
  `past  x=fix@3!=*3 (FIX · 15m · 2 of 2 · repair · close: parked 3 · defer the rest)`.
- **Grammar revision.** `GRAMMAR_REVISION` becomes **12**. Update every hard-coded `11`
  in help and tests.

**`bob capture` JSON**, per member:

- `kind: "pomodoro_past"`, `placement: "recorded"`, and `task_line` (the recorded ledger
  line).
- A `pomodoro_past` object. The example below is the first member of
  `x=build@2! x=fix@3!`, recorded against a closed PLAN 0855–0920:

  ```json
  {
    "raw": "x=build@2!",
    "name": "BUILD",
    "target": { "source": "placeholder", "line": 9 },
    "blocks": 2,
    "repair": true,
    "stack": { "index": 1, "size": 2 },
    "anchor": "0925",
    "window": { "start": "0900", "end": "0910", "minutes": 10 },
    "closed_at": "0910",
    "line": 5,
    "trims": [
      {
        "line": 4,
        "name": "PLAN",
        "before": "0855-0920",
        "after": "0855-0900",
        "minutes": 20,
        "clock": "t",
        "calendar": false
      }
    ],
    "close": { "…": "the existing PomodoroCloseSummaryJson" }
  }
  ```

  Field notes:
  - `target.line` is the source row's line before the record.
  - `line` is the recorded row's line after it.
  - `trims` is empty on members that report no trim.

- **No top-level `pomodoro_close` key.** An older Mac would render one as a
  running-session Close card.
- **Batch level.**
  - `pomodoro_blocks` gains string roles `recorded` and `trimmed` (the Mac decodes roles
    as `String`).
  - Each trim is also a structured warning with code `pomodoro_past_trim`.
  - Each trim is reported on the first member, in typed order, whose window overlaps
    that row, with the minutes the whole stack took.

**Human output**, mirroring the close renderer:

- The verb is `recorded` (`would record` in a dry run).
- The suffix is ` · 1 of 2` when the stack has more than one member.
- Task rows, the `next:` line, clock names, and the R6 `from …` suffix are reused
  unchanged.
- Each trim prints exactly once, as a `⚠` warning line.

The same `x=build@2! x=fix@3!` draft renders like this. Task rows and the `next:` line
use today's close formatting; they are illustrative here.

```text
recorded BUILD 0900-0910 (10m) · 2026/20261104.md line 5 · 1 of 2
  nothing queued
recorded FIX 0910-0925 (15m) · line 6 · 2 of 2
  1 [/] Fix agent crash · in progress · carried
  next: FIX at line 8 · carries 1 link
⚠ took 20m from PLAN · 0855-0920 → 0855-0900 (5m) · line 4
```

### Worked examples (`BOB_NOW=2026-11-04 09:21:00`, T = 09:25)

| Draft                                                 | Result                                                                                                                                                                                                                    |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x=build`                                             | BUILD 0900–0925, closed at 09:21                                                                                                                                                                                          |
| `x=build@2 x=fix@3`                                   | BUILD 0900–0910 (closed at 09:10), FIX 0910–0925 (closed at 09:21)                                                                                                                                                        |
| `x=2#fix=*3 apollo agent` ≡ `x=fix@2=*3 apollo agent` | FIX 0915–0925; task 3 parked (stays as 🍅 history under the closed row, not carried) with Work Log `apollo agent` at 09:21; tasks 1–2 deferred and carried in FIX's slot; both spellings give byte-identical vault writes |
| `x==1!2`                                              | next placeholder 0900–0925; task 1 `[/]` and carried, task 2 completed, the rest deferred and carried                                                                                                                     |
| `x=3!` with closed PLAN 0855–0920                     | new row from the next placeholder, 0910–0925; PLAN → `(**0855-0910** [t:: 15m])`; `⚠ took 10m from PLAN`                                                                                                                 |
| `x=3` while WORK runs 0900–0925                       | refused: open row; teaches `x x=3!`                                                                                                                                                                                       |
| `x x=fix@3!` while WORK runs 0900–0925                | WORK closes 0900–0925, then FIX 0910–0925 trims WORK to 0900–0910                                                                                                                                                         |
| `x=3` after `=` at 09:21 (running 0925–0950)          | touching, so it records 0910–0925 before the running row                                                                                                                                                                  |
| `x=3` at 00:05                                        | refused: would start before midnight; teaches `x=1`                                                                                                                                                                       |
| `x=3` at 23:58                                        | records `2345-0000`                                                                                                                                                                                                       |

### Non-goals

- Absolute times (`x=0900-0925`) and backfilling another day.
- A `bob pomodoro record` subcommand.
- Obsidian command parity.
- An `--expect-anchor` submit guard (hold in reserve; the Mac re-runs its preview at
  block boundaries instead).
- Trimming or closing open sessions.
- Splitting or erasing history.

## Phase grammar: x= lexer, capture-parse contract, and grammar 12

Scope: everything text-level. After this phase, `bob capture-parse` fully understands
`x=`, while `bob capture` refuses valid tokens with
`recording past Pomodoros (x=) is not available yet` (removed by record).

1. **Lexer** (`capture_language/item.rs`):
   - Add `EqualsToken::Past { … }` holding the selector, blocks, repair, close-part text
     and offsets, and `len`.
   - Claim it in `session_equals_token` right after the bare-`x`/`x#` checks, before the
     merge and retired-close checks.
   - Teach `is_session_chain_token` and the exhaustive matches
     (`parse_pomodoro_equals_item` and `editor_pomodoro.rs` around
     `parse_editor_close_item`).
   - The close part is lexed by `lex_close_selection` with offsets shifted into the
     token.
2. **Model** (`capture_language/model.rs`):
   - Add `CaptureKind::PomodoroPast { spec: PomodoroPastSpec }`. The spec holds `raw`,
     `name: Option<String>`, `name_is_literal`, `blocks: u32`, `blocks_typed: bool`,
     `repair`, `close: PomodoroCloseSpec` (including `log`), and
     `stack: PastStackPos { index, size, offset_blocks }`.
   - Thread it through every exhaustive match on `CaptureKind`: `type_marker.rs`
     `TypeMarkerSubject::for_kind`, `dependencies.rs` `resolve_execution_ownership`,
     `capture/batch.rs` `plan_capture_to_target`, `capture/plan.rs` line formatting and
     `capture_kind_label` (`"pomodoro_past"`).
   - Add it to the non-exhaustive session lists: `refuse_clip_on_incompatible_kind`,
     `is_template_session_kind`, `capture/relations_merge.rs`,
     `reject_operator_dependencies`, and the `@@`/forced-route refusals that starts use.
3. **Chains and logs** (`capture_language/draft.rs`):
   - `is_chain_close_token` and the owner `rposition` picks treat `Past` as a close, so
     `x=` owns inline entries and child bullets.
   - Reuse the close-log parser for the entry and bullet layouts (exact, child bullets,
     inline), so every existing close-log diagnostic applies.
4. **Stacks:** add the shared `assign_past_stacks` pass (see Stacks). Call it in both
   the execution and editor parses.
5. **Editor and `capture-parse`:**
   - Add `SpanKind::PomodoroPast` and `PomodoroPastRepair` with their labels.
   - Add `EditorMode::PomodoroPast`.
   - Add the `pomodoro_past` field on `EditorParse` and `EditorItemParse`, plus the
     first-item projection (about 30 struct-literal sites).
   - Update `capture_parse.rs`: the wire fields, the human `past` line, and the
     `refusal_diagnostic`.
   - Suppress completion inside the inline Work Log text (`cursor_in_close_log_text`).
   - Teach `editor_close_outcome`.
   - Add the `EditorMode` arms in `rewrite.rs` and `capture_task_wikilinks.rs`: no `@@`
     absorption and no rewrite.
6. **Revision and help:**
   - Set `GRAMMAR_REVISION = 12`.
   - Update the hard-coded `11` in `capture_parse.rs` help and
     `capture_complete/cli.rs`, and the tests that assert 11.
   - Extend `capture-parse` long help with the `x=` mode, object, spans, and teaching
     errors, in the same dense style.
   - Add one recognition sentence to `bob capture --help` (`capture/cli.rs`).
   - Run `just docs-gen`.
7. **Planner placeholder:** add the `PomodoroPast` arm in `plan_capture_item`. It
   returns the temporary usage error above.

Tests:

- **Language** (`capture_language/tests/`: new `past.rs`, plus additions to `chain.rs`,
  `bare_close.rs`, `editor_spans.rs`, `editor_modes.rs`):
  - every row of the token table, teaching-error table, and incomplete list;
  - prose guards (`X=3`, `Plan x=3`);
  - name extents ending at `!`, `=`, `@`, `~`;
  - literals containing `!` and `=`;
  - `x=2#fix=*3` and `x=fix@2=*3` producing equal specs;
  - the three compatibility breaks;
  - chain ownership (`x x=3! entry`, `x=3 =`, `x=build@2 x=fix@3 entry`, child bullets
    on the last `x=`);
  - stack numbering across one line, across adjacent items, and broken by another item;
  - editor and execution agreement.
- **CLI** (`tests/cli/capture/parse_pomodoro_past.rs`): JSON and human `capture-parse`
  goldens for the object, spans, `items[]`, and grammar 12, plus `bob capture 'x=3'`
  returning the temporary refusal with nothing written.

Done when: `capture-parse` reports the full contract above, every listed test passes,
and `just check` is green.

## Phase complete: Pomodoro-name completion inside x=

- **`completion_field_at`** (`capture_language/completion.rs`) returns a new
  `PomodoroPastName` context:
  - after `x=`, `x=#`, `x=D#`, and inside `x=#"…`;
  - with the query bounded by the past name extent;
  - not on the count, the `!`, or anything after the second `=`.
- **`capture_complete/engine.rs`** dispatches that context to `pomodoro_candidates` with
  past semantics:
  - open placeholders and completed names are offered;
  - the running session is never offered (it would be refused);
  - there is no `override` block.
- **Replacement ranges** follow the bare-versus-`#` spelling rules of `=name`. They end
  before any `!`, `=`, or `@D`.

Tests: `capture_complete/tests/pomodoros.rs` cases for each position, the
running-session exclusion, literal names, the replacement ranges, and no completion
after the second `=`.

## Phase record: record engine with explicit-entry close, placement, and output

Scope: execute every valid `x=` member and stack on a ledger. The strict guard is
temporary:

- **Overlap.** Refuse any overlap with any timed row (open or closed), naming the row
  and its line.
- **`!` in this phase.** Refuse `!` on any member: `repair (!) is not available yet`.
  guard replaces both rules.

1. **Close seam.**
   - Add the entry-taking close variants and the `close_types` entry parameter (see
     Recording). Keep the `x` wrappers byte-identical.
   - Pass C as `now` to the close (Event Task clock, Work Log stamps).
   - Bypass the reset branch.
2. **Planner** (`capture/plan.rs`, new `capture/pomodoro_past.rs`):
   - In the batch loop, when a member with `stack.index == 1` is reached, compute every
     member's window from the specs. Check the midnight rule and run the guard pre-pass
     for the whole stack (`parsed_items[i..i+size]`).
   - Then plan each member:
     1. guard the day file and the `## Pomodoros` section;
     2. resolve the target (Targets);
     3. write the window;
     4. close the explicit entry at C;
     5. place the row chronologically;
     6. keep the continuation only when links carry.
3. **Shared placement:** lift `closed_insert_at` and its helpers from `gcal/plan.rs`
   into a shared ledger module used by gcal and capture. Add the "continuation only when
   carrying" ledger-close option.
4. **Output** (`capture/output.rs`, `pomodoro_blocks.rs`):
   - Add `CaptureItemResult.pomodoro_past` with the JSON above (`trims` stays empty in
     this phase).
   - Add the `recorded` block role. The block before-state is `At(source line)` for
     placeholders and `Created` for created and "again" rows.
   - Add the human renderer described above.
   - Remove the temporary refusal from grammar.
   - Add a short "record" sentence to `bob capture --help` with an `'x=build@2 x=fix@3'`
     example, then run `just docs-gen`.

Tests (`tests/cli/capture/pomodoro_past.rs`, using `BOB_NOW`, `BOB_DAY_FILE`, and the
`close_worked_vault`-style fixtures):

- **The user's examples 1–4**, in JSON and human form, with exact ledger bytes. Both FIX
  spellings must be byte-identical.
- **Golden equivalence:** on a quiet ledger whose first open entry is the target, `x=N`
  equals `=N-N x` byte for byte, for every `<X>` shape, at every minute 09:20–09:24 and
  at 09:20:59.
- **Stacks:**
  - two and three members, and stacks across adjacent items;
  - a stack broken by `=`;
  - `x=2 x=3` taking the carried links;
  - earlier members closing at their own end (a 09:15 meeting is `not_started` for BUILD
    0900–0910; no overrun text).
- **Targets:**
  - no placeholder;
  - a MEET-first placeholder;
  - named open, again, and created rows, including the near-match warning;
  - a name resolving to the running session.
- **Running session:**
  - an adjacent running session records the past row before it and leaves it untouched;
  - an overlapping running session is refused.
- **Continuation:**
  - nothing carries, so no empty placeholder;
  - carried links stay in the source slot.
- **Midnight:** 00:05 is refused; 23:58 writes `0000`.
- **Event Task close vectors** still pass unchanged.
- **Atomicity:** byte-identical rollback on every refusal, and the dry run matches the
  real run at a fixed `BOB_NOW`.

## Phase guard: whole-stack overlap guard and ! edge repair

Replace record's strict guard with the full guard and repair (see Guard and repair).

1. **The pre-pass:**
   - Collect the timed rows from the staged ledger. Use `capture_pomodoros::scan`,
     `parse_adjustment_range`, and `session_clock`, applying the midnight normalisation.
   - Classify each member: open overlap, closed overlap, or none.
   - Compute each closed row minus the union of the stack windows, giving 0, 1, or 2
     pieces.
   - Enforce the per-member `!` rule, and return one multi-line refusal that lists every
     offending member with its rewrite.
2. **Trims:**
   - Stage the span rewrites before the first member records.
   - Attribute each trim to the first overlapping member (`pomodoro_past.trims`).
   - Emit the `pomodoro_past_trim` structured warning, the `trimmed` block role, and the
     `⚠ took …` human line, with `(calendar)` for MEET rows.
3. **gcal check:** verify that `bob gcal sync` never rewrites a closed MEET cluster's
   span. Add a regression test that a trimmed MEET row survives a later sync plan
   unchanged. If sync would rewrite it, refuse MEET trims instead, document why, and
   tell the docs phase through the phase bead.

Tests:

- **The guard table:**
  - every row of the guard table;
  - touching endpoints on both sides;
  - several rows trimmed in one capture;
  - the BUILD+FIX stack that would look like a split if checked token by token;
  - off-clock (`[off::]`) victims keeping `off`;
  - MEET victims flagged as calendar rows;
  - legacy unstamped rows;
  - zero-length rows ignored.
- **`!` rules:** a stack mixing needed and unneeded `!`; the open-row refusal with and
  without `!`.
- **Composition:** `x x=fix@3!`.
- **The user's five examples end to end**, in JSON and human form. Example 5 has exact
  PLAN bytes, and the minutes are conserved.

## Phase docs: capture docs, references, and help for x=

Read the `docs.md` and `cli_rules.md` reference memories first.

- **`docs/capture.md`:**
  - Add a new section, `### Recording a past Pomodoro`, right after "Closing the running
    Pomodoro" and its subsections. Cover:
    - the mnemonic;
    - the grammar and token table;
    - teaching errors;
    - stacks and windows;
    - targets;
    - recording at C;
    - placement and continuation;
    - the guard table and messages;
    - human and JSON output;
    - the worked examples;
    - the idioms (`x=3 =`, `x x=fix@3!`, a typed timeline);
    - zsh quoting (single-quote tokens with `!`, `*`, or `~`).
  - Add a row to "Grammar at a glance".
  - Add an `x=` row to the session lifecycle table, and update the mnemonic there.
  - In "Chaining session operators on one line", the seven operators become eight, and
    the examples gain `x x=3!` and `x=3 =`.
  - Note the grammar 12 break wherever the grammar 11 history is listed.
- **`docs/reference/capture-syntax.md`:** token index rows, a Pomodoro section entry,
  and the teaching errors under "Retired and rejected forms".
- **`docs/reference/json-outputs.md`:** grammar 12, the `pomodoro_past` item object, the
  block roles, and the warning code.
- **`docs/reference/compatibility.md`:** it is stale at grammar 10. Add the 11 and 12
  entries.
- **`docs/getting-started.md`:** one line in the Pomodoro walkthrough.
- **Help:** a final review pass over `bob capture --help` and `capture-parse --help` for
  clarity and consistency, then `just docs-gen` and `just docs-check`.

> [!decision] memory_decision_record Use `/sase_memory_write` to add a decisions-web
> strand titled **Past Pomodoros End Now And Only Trim Closed Edges** (slug
> `past-pomodoros-only-trim-closed-edges`). Its claim: `x=` records end at now's
> 5-minute block and stack backward; they never restart, swap, trim, or record over an
> open session; and `!` only trims an older closed row's edge, never splitting or
> erasing it.
>
> The **Why** section cites:
>
> - the research report and its rejected alternatives (the `=N-M x` macro, a trailing
>   `!`, letting `!` touch the running session, fill-free-time);
> - the docs section;
> - the guard tests.
>
> Then run `sase memory init`.

## Phase mac: Bob Mac Capture Record past work card

Work in the linked `bob-mac-capture` repo (open it with `/sase_repo`). Follow its
`AGENTS.md`:

- Run `just portable-check`.
- Type-check the app targets on the Mac host.
- Run docs checks.
- Format only the lines you touch.
- If the Mac is offline, say so.

Build bob from master to generate fixtures.

1. **Decode.**
   - Add `pomodoroPast` to the `capture-parse` item and response models.
   - Add a `PomodoroPastSummary` (with a nested `PomodoroCloseSummary`) to
     `CaptureCommandSuccess`, decoded tolerantly with `decodeIfPresent` / `try?`. An
     older bob must still work.
   - No grammar gate is needed, because the card is keyed on `kind == "pomodoro_past"`.
2. **Card.** Build a `CapturePomodoroPastPresentation` that groups consecutive
   `pomodoro_past` captures by stack into **one "Record past work" card per stack**:
   - a header with the total minutes;
   - a timeline line (`09:00 ── BUILD 10m ── 09:10 ── FIX 15m ── 09:25`), which explains
     and is not a control;
   - one row per member in time order, with the name, exact range, and minutes, plus its
     close task rows rendered by the existing close-row components;
   - trim rows in amber, stating "gives 10m" and `(calendar)` in text, never colour
     alone.

   bob errors keep the existing refusal styling. Map the notification title ("Recorded
   past Pomodoro"), the VoiceOver summary, and the footer verb ("Record").

3. **Spans.**
   - In `captureSemanticCategory(forSpanKind:)`, map `pomodoro_past` with the session
     operators and `pomodoro_past_repair` to a distinct warning-toned category.
   - Make sure name completion fires at the `x=` name spans (`completionSpanKinds`).
4. **Freshness:** while a `pomodoro_past` preview is visible, schedule one dry-run
   refresh at the next 5-minute wall-clock boundary (and the next after that, as long as
   the preview stays visible). The windows move with T.
5. **Docs and tests:**
   - Update `docs/capture-previews.md` (the new card) and the `docs/protocol.md`
     compatibility table (grammar 12, `pomodoro_past`).
   - Add a fixture generator script modelled on the existing
     `Scripts/generate-*-fixtures.sh`. It produces stack, trim, refusal, and parse
     fixtures from a real bob.
   - Add presentation and decoding tests in `Tests/CaptureCoreTests`.
   - Add a design test in `Tests/BobMacCaptureTests`.
