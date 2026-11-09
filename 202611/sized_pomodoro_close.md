---
tier: epic
title: "Nx: close the running Pomodoro at a stated size"
goal: "Typing `2x!2` closes the running Pomodoro at exactly 2 five-minute blocks from
  its start (or refuses with a named fix), with honest clocks, one card, additive JSON,
  documented grammar, a decision record, and a Bob Mac Capture card that shows the size.

  "
phases:
  - id: grammar
    title: Claim `Nx` in both parsers
    depends_on: []
    size: large
    description:
      "grammar: add the shared size lexer and `PomodoroCloseSpec.size`, claim sized
      whole-item, chain, and link closes plus their teaching errors in the execution and
      editor parsers, extend `capture-parse` spans/JSON and the close-outcome aliases,
      and make `bob capture` refuse sized specs until execution lands."
  - id: execute
    title: Run sized closes with exact-or-refuse
    depends_on:
      - grammar
    size: large
    description:
      "execute: stage exactly 5N minutes from the session start, close at C = min(S +
      5N, now) through all three close entry points, refuse with coded two-line errors,
      add `pomodoro_close.size` JSON, the sized human caption and `tip:`, and bump the
      grammar revision to 20."
  - id: docs
    title: Document `Nx` and record the decision
    depends_on:
      - execute
    size: medium
    description:
      "docs: update capture.md, the capture-syntax reference, and capture/capture-parse
      help, regenerate CLI docs, and add the exact-or-refuse decision record when its
      decision is on."
  - id: mac
    title: Present sized closes in Bob Mac Capture
    depends_on:
      - execute
    size: medium
    description:
      "mac: decode `size`, draw both-direction arrows and the `Nx · as of HH:MM` chip,
      widen the close-alias prefilter for digit prefixes, record fixtures from a real
      bob, and watch macOS CI to green."
decisions:
  chain_scope:
    ask:
      Should a sized close chain only at the front of its line, so text after a close or
      start keeps its meaning?
    choices:
      front: "`x 2x faster` logs `2x faster`; `--1 2x!2` and `2x =` still chain"
      like_x:
        "`2x` chains wherever `x` does but never heads a trail; `x 2x faster` fails"
    default: front
    why:
      After a close or start a sized close can never succeed, so claiming it only breaks
      log text
    answer: front
  decision_record:
    ask:
      Should the docs phase add a decisions record for the sized close's exact-or-refuse
      rule?
    memory:
      - decisions:sized-close-records-exact-blocks
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
proposed_by: bbugyi200.apollo.research.3s.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 05:24:35
status: wip
---

- **PROMPT:**
  [prompts/202611/sized_pomodoro_close.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/sized_pomodoro_close.md)

# `Nx`: close the running Pomodoro at a stated size

## Goal

Add an optional `N` prefix (one or two ASCII digits) to the `x` capture close. `N` is
the size, in five-minute blocks, that the running Pomodoro is recorded at before it
closes. On a 25-minute session, `2x!2` writes the same span and the same task outcomes
as `-3 x!2`, but in one gesture and with honest clocks.

The design follows the research report
`research:202611/sized_pomodoro_close/sized_pomodoro_close.md` (read it with
`sase artifact read file:explicit:e83b2f4f99a787f5d916fccb "<reason>"`). Bryan accepted
every requirement it recommends (R1–R11) and every recommended answer to its questions
(refuse rather than cut; blocks rather than minutes; sized link closes in v1; a dim
human-only `tip:`; three or more digits reserved). This plan adds four refinements,
called out in [Refinements over the research](#refinements-over-the-research).

The promise users learn is one line: **the number before `x` is the number of blocks Bob
records, or Bob tells you why not.** Time goes before `x`; task numbers go after it.

## The contract

### One rule

`Nx` closes the running session as if you had typed `x` the moment it reached N blocks.
Unlike `x`, it may also lengthen the session, and it never records time that has not
happened.

### Exact-or-refuse timing

Let the running session be `[S, E)`, `T` be now's minute of day, and `N ∈ 1..=99`. Let
`e = T − S` normalized into (−720, 720] exactly the way `close_timing` normalizes
`remaining` (`signed_minute_delta`), and `m = 5N`.

1. **Parseable?** A running row whose time range does not parse
   (`parse_adjustment_range` returns `None`) → refuse with code
   `pomodoro_close_size_unparseable`. Plain `x` keeps its "left as written" warning; a
   sized close cannot honor its number, so it refuses.
2. **Started?** `e < 0` → refuse with code `pomodoro_close_size_not_started`.
3. **Elapsed?** `m − e ≥ 5` → refuse with code `pomodoro_close_size_not_elapsed`. This
   allows exactly the five-minute round-up plain `x` already makes. The largest size
   that fits is `fits = ⌈e / 5⌉` blocks (0 when `e = 0`).
4. **Size.** `E′ = (S + m) mod 1440`. Stage the row's range as
   `format_adjusted_range(S, E′, m, &range.metadata)`, the formatter `±N` uses, so the
   clock key (`[t::]` or `[off::]`) and every other metadata byte survive.
5. **Close at `C`.** `lag = e − m` (≥ −4). The close datetime is
   `C = now (seconds zeroed) − max(lag, 0) minutes`, so the date stays right across
   midnight. Run the existing close planner at `C` on the staged text, with today's
   selection, naming, Event Task classification over `(S, C)`, Work Log stamps at `C`,
   carry, continuation, and successor offers all unchanged. At `C`, `close_timing` sees
   `remaining = E′ − C ∈ [0, 5)`, so it never trims again and never reports an overrun.
6. **Never a reset.** `2x0` closes with every plain link deferred. A size claims work,
   matching "explicit modifiers always close".

Every refusal writes nothing: the whole capture batch rolls back through
`CaptureBatchPlanner`, and `--dry-run` previews show the refusal before Return.

| Draft on `0900-0925` @ now             | Result                                          |
| -------------------------------------- | ----------------------------------------------- |
| `2x` @ 09:50 (closing late)            | `0900-0910`, closed at 09:10                    |
| `2x` @ 09:08 (inside the round-up)     | `0900-0910`, closed at 09:08                    |
| `7x` @ 09:50 (claiming an overrun)     | `0900-0935`, closed at 09:35                    |
| `5x` @ 09:40                           | `0900-0925`, closed at 09:25, no `ran 15m over` |
| `2x` @ 09:03                           | Refused: `1x` fits                              |
| `20x` @ 09:50 (typo for `2x`)          | Refused: `10x` fits                             |
| `2x` on a non-grid `0902-0925` @ 09:30 | `0902-0912` (`-3 x` would give 8m)              |
| `2x` on a MEET starting 13:00 @ 12:50  | Refused: not started; `x0` resets it            |

### Refusal copy

Each refusal names the token, the fact, and one fix. The message is two lines: a
self-contained first line, then the fix indented two spaces.

```text
error: `7x` asks for 35m, but BUILD has run 12m since 0900; nothing was written
  `3x` closes at 0915 · `x` closes now
error: `1x` asks for 5m, but BUILD started at 0900 and has run 0m; nothing was written
  `x0` resets it
error: `2x` cannot size MEET: it starts at 1300 and has not begun; nothing was written
  `x0` resets it
error: `2x` needs BUILD's time range, but it could not be parsed; nothing was written
  fix the range, or close with `x`
```

Use the running entry's name, or `the running session` when unnamed (the same fallback
the not-started warning uses). JSON errors carry the `code` above plus
`extra.close_size`:
`{"raw":"7","blocks":7,"minutes":35,"elapsed_minutes":12,"fits_blocks":3}`
(`elapsed_minutes`/`fits_blocks` omitted when not computed).

## Grammar

The form is `[N]x[K][*P][!M][~D][@K]`, plus today's inline Work Log, child bullets,
chains, and link closes. `N` is one or two ASCII digits glued to the `x`. After `x`, the
text must already be close-shaped: empty, or starting with a digit, `,`, `!`, `~`, `*`,
or `@`. Modifier order, wildcards, and the typed-last `@K` stay exactly as today. The
prefix never changes the case rule of the form it prefixes: whole items need lowercase
`x`; link suffixes accept `x` or `X`, as `=X` does today.

| Input                                                       | Behaviour                                                                                                                                                                        |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2x`, `2x!2`, `2x*!2`, `2x1,3!2`, `2x!2@1`                  | Sized close (10m) with today's selection semantics. The prefix is never a task selection: `2x` defers exactly what `x` defers                                                    |
| `02x`                                                       | Valid, 2 blocks (leading zeros match `=03`, `x=03`, `+03`)                                                                                                                       |
| `2x wired the lexer`, `2x` + child bullets                  | Sized close with a Work Log entry stamped at `C`                                                                                                                                 |
| `2x =`, `2x =bugs`, `--1 2x!2`, `2x x=fix~`                 | Session chains. A `--1` shift runs first, so the size counts from the shifted start                                                                                              |
| `@r:id=2x!1`, `^r:id=2x`, `<text> @r:id=2x`                 | Link (or create) first, then the sized close; numbers refer to the post-link lineup. Today these are errors, so nothing collides                                                 |
| `2x0`                                                       | Sized close that defers every plain link. Never a reset                                                                                                                          |
| `2x~`, `2x1,`                                               | The incomplete-selection states `x~` and `x1,` have today, including `needs: ["pomodoro_close_task"]` and completion replacement ranges after the digits                         |
| `0x`, `00x`                                                 | Claimed error: "`0x` needs at least 1 block: write `1x` to close at 5 minutes, or `x0` to reset the session"                                                                     |
| `100x`, `0910x`                                             | Claimed error: "`100x` is too large: a close size is 1–99 five-minute blocks (`12x` is an hour)". Three or more digits stay reserved for a possible clock-time close             |
| Whole-item `=2x`                                            | Claimed error teaching both readings: "`=2x` is not a start: write `2x` to close at 10 minutes, or `=2#x` to start a Pomodoro named X"                                           |
| `2x=3`                                                      | Claimed error: "`2x=3` mixes a close size with a past record: `x=` takes its size after `=`; write `x=3` to record 15 minutes, or `2x =3` to close at 10 minutes and then start" |
| `2x#bugs`                                                   | The same near miss as `x#bugs`, teaching `2x =bugs`                                                                                                                              |
| `2X`, `2X4 lumber`, `2 x!2`, `2xx`, `2x-ray`, `2.5x`, `２x` | Prose in the grammar, as today. Capital `X` is the documented escape; there is no spaced form. (Mac typing still maps an eligible `2xx` to `2x!`; see the Mac phase)             |
| `x made it 2x`                                              | One close whose log reads "made it 2x": a sized token is never a trail head                                                                                                      |
| `x 2x faster`, `= 2x`                                       | Decided by `chain_scope`, below                                                                                                                                                  |

> [!decision] chain_scope = front A sized token chains only when every earlier token on
> its line is an adjust or shift operator (`+N`, `-N`, `++N`, `--N`). Anywhere else it
> is text: `x 2x faster` logs `2x faster`, `= 2x` stays task text (as `= foo` does
> today); every line where a close or start precedes the sized token keeps exactly
> today's parse. `2x =`, `2x x=fix~`, and `--1 2x!2` still chain. Docs state it as: "a
> sized close comes first on its line, after any `±`/`±±` moves".

> [!decision] chain_scope = like_x A sized token is a chain token wherever `x` is (lead
> or chain position), except that it is never a trail head. `x 2x faster` splits into
> two closes and the batch fails at the second; `= 2x` starts and then refuses (nothing
> has elapsed). Docs list these as claimed shapes with the `2X` escape.

Never search-and-replace `\d+x` across a draft, and never rewrite the prefix.
Completion, the comma assist, and every rewrite edit only the selection span.

## Machine contracts

- **`capture-parse` spans.** The existing `pomodoro_close` span grows to cover the
  digits: `2x!2` is `0–2 pomodoro_close` plus `2–4 pomodoro_close_complete`, mirroring
  `pomodoro_start` over `=3` and `pomodoro_past` over `x=3`. For a link suffix the
  digits join whichever span covers its `x` today. An unmodified Mac app therefore tints
  the size as session syntax, and the digits are never a list span, so the comma assist
  cannot turn `12x` into `1,2x`.
- **`capture-parse` spec.** `pomodoro_close.size: {"raw":"2","blocks":2,"minutes":10}`,
  omitted when absent, so every unsized payload (including `pomodoro_past.close`) stays
  byte-identical. `raw` (the spec's existing field) stays exactly as typed: `2x!2`, or
  `=2x!1` on a link. The human line reads
  `close     2x!2 (10m · complete 2 · defer the rest)`; a plain sized close reads
  `close     2x (10m)`.
- **`bob capture` JSON.** One `pomodoro_close` object, never a sibling `pomodoro_adjust`
  item:
  - `planned` is the sized span and `closed` is the result (equal for a sized close),
    `decremented_minutes` is `0`, `remaining_minutes` is `E′ − C`, and `closed_at` is
    `C`. Current Mac builds then render a truthful `0900–0910 · 10m · on time`.
  - Additive member `size`, omitted when absent:

    ```json
    "size": {"raw": "2", "blocks": 2, "minutes": 10,
             "from": {"time_range": "0900-0925", "duration_minutes": 25},
             "delta_minutes": -15}
    ```

    `from` is the running span as staged just before sizing (after earlier chain items
    and any same-item link insertion).

- **Grammar revision** goes from 19 to 20 in the phase that makes `bob capture` execute
  sized closes, never earlier.
- **Release.** Claiming previously valid prose is a grammar break: the grammar phase
  lands as `feat(capture)!:`. JSON changes are additive. No phase edits the Cargo.toml
  version or `CHANGELOG.md` (`decisions:bob-cli-releases-ride-a-daily-train`).

## Presentation

The size lives on the timing line; task numbers stay on the task rows. The two kinds of
number never share a place.

Terminal (`bob capture` human output):

```text
ok closed BUILD 0900-0925 → 0900-0910 (10m · 2x, −15m) · 2026/20261108.md line 7
  1 [*] deferred  Add support for x syntax  bob.md ^capture-stop
  2 [*] → [x]     Web capture               bob.md ^web-capture +1 Work Log
      *2026-11-08 09:10* — wired the lexer
  next: BUILD (created) at line 9 · carries 1 link
```

| Case      | Caption                                              |
| --------- | ---------------------------------------------------- |
| Shrink    | `0900-0925 → 0900-0910 (10m · 2x, −15m)`             |
| Lengthen  | `0900-0925 → 0900-0935 (35m · 7x, +10m)`             |
| Same size | `0900-0925 (25m · 5x)`                               |
| Off-clock | `0900-0925 → 0900-0910 (10m · off-clock · 2x, −15m)` |

The arrow comes from `size.from`, not `decremented_minutes`; a sized close never prints
`(ran Nm over)`. `−` is U+2212, as today.

**Tip.** When a human-output batch holds a `±k` adjust item immediately followed by an
unsized whole-item close of the same Pomodoro line, and the close kept the adjusted span
(`decremented_minutes == 0`) at a size of 1–99 whole blocks, print one dim line after
the close block: `tip: 2x!2 closes at 10m in one step` (the close's selection after its
`x`, without Work Log text). Never in JSON, never for shifts, link closes, or sized
closes.

Mac close card (bob-mac-capture phase), the chip replacing the early/over chip in the
session-operator tint:

```text
Close BUILD
0900–0925 → 0900–0910 · 10m                 [ 2x · as of 09:10 ]
2  Complete  Web capture
1  Deferred  Add support for x syntax
```

Accessible label: "Close BUILD at 10 minutes, 2 blocks, 15 minutes shorter than
planned." (`longer than planned` / `as planned` for the other cases.)

Teaching copy, used by docs and help:

- Hint: "Before `x`: 5-minute blocks · after `x`: task numbers".
- Mnemonic: "`=` starts now, `x` stops now, `Nx` stops at N blocks, `x=` records what
  already happened."

## Refinements over the research

1. **Chain scope** (see `chain_scope`). The research claimed a sized token wherever `x`
   chains, guarding only trail heads. This plan also keeps it out of positions after a
   close or start, where it can never succeed, so `x 2x faster` logs its text instead of
   failing.
2. **No `pomodoro_runs_past_event` on lengthen.** The research suggested firing `+N`'s
   run-past-event warning when `7x` lengthens. `src/native/capture/event_conflict.rs`
   documents that closes never warn, and its `+K fits` advice plans forward time that a
   close has already spent. Linked meetings are already handled by the Event Task clock
   at `C`. Sized closes stay silent there.
3. **Structured refusals.** Refusals carry stable `code`s and `extra.close_size`, so
   clients can render them without parsing prose.
4. **Mac grammar gate unchanged.** The Mac shows the size chip only when Bob sends
   `size` (`decodeIfPresent`), so `CaptureGrammarGate.minimumRevision` stays 15 and
   nothing needs a grammar-20 gate.

## Architecture notes for every phase

- **One owner for the prefix.** Add the size lexer once, in
  `src/native/capture_language/close_selection.rs`, beside `classify_whole_item_close`
  and `close_after_eq_parts`. It returns the size (`raw`, `blocks`, byte range) plus the
  existing selection body after `x`, or a claimed error. Every classifier —
  `classify_whole_item_close`, `whole_item_close_after_x`, `is_plain_whole_item_close`,
  `close_after_eq_parts` / `classify_link_close` / `link_close_after_x`,
  `is_close_hash_token`, `session_equals_token`, `is_session_chain_token`,
  `past_token_len`'s `2x=` near miss, the inline Work Log lexers in `close_log.rs`, and
  the alias locator in `rewrite.rs` — routes through it, so the execution parser
  (`item.rs`, `tokens.rs`) and the editor parser (`editor_pomodoro.rs`,
  `editor_classify.rs`) cannot disagree. Spans that assume a one-byte `x`
  (`token_start + 1`) must use the size's end.
- **Chain position lives in `draft.rs`.** `session_chain_tokens`,
  `split_inline_close_tokens`, and `rewrite.rs`'s `close_lead_ok` share one predicate
  for "may a sized token chain here" (front-of-line rule, or the `like_x` rule if
  chosen), and `is_trail_head_token` is false for sized tokens.
- **Model.** `PomodoroCloseSpec` gains `size: Option<CloseSize>`
  (`#[serde(skip_serializing_if = "Option::is_none")]`), where
  `CloseSize { raw: String, blocks: u32, minutes: u32 }`. `PomodoroCloseSpec::plain`,
  `close_spec_from_lex`, `close_spec_from_incomplete`, and every struct literal set it.
- **Execution.** One shared helper in `src/native/capture/pomodoro_close.rs` (or a
  sibling module) applies [exact-or-refuse](#exact-or-refuse-timing): resolve the
  running entry on staged text with `find_running_pomodoro` (keeping today's no-running
  / several-running errors), parse its range, splice the sized range (factor the splice
  from `plan_pomodoro_adjust_item` rather than copying it), stage it, and return `C`
  plus the `size` JSON. All three close entry points use it: `plan_pomodoro_close_item`,
  `plan_pomodoro_close_link_item` (after the link insertion), and
  `plan_pomodoro_close_task_item` (after the new task's link). A sized close then calls
  `plan_pomodoro_close` (which has no reset branch) at `C` instead of
  `plan_pomodoro_close_outcome`, and `build_close_summary_json` at `C`. `is_reset_spec`
  returns false for a sized spec. The past-record path (`pomodoro_past.rs`
  `record_one_member`) is the template for closing an explicit entry at its own clock.
- **Time in tests.** CLI tests pin time with `BOB_NOW` (see
  `tests/cli/capture/pomodoro_close.rs`).

## Phases

### Claim `Nx` in both parsers

- **Repository:** bob-cli
- **Commit:** `feat(capture)!: claim the sized Nx close in the capture grammar`

Scope: everything in [Grammar](#grammar) and the `capture-parse` half of
[Machine contracts](#machine-contracts), with the architecture in
[Architecture notes](#architecture-notes-for-every-phase).

- Add `CloseSize` and `PomodoroCloseSpec.size`; implement the single size lexer and
  route every listed classifier through it.
- Execution parser: whole-item exact, bullet, and inline-entry close paths in `item.rs`;
  link and task close suffixes in `tokens.rs`; the claimed errors (`0x`, 3+ digits,
  whole-item `=2x`, `2x=3`, `2x#bugs`). Messages live in `markers.rs` and are shared by
  both parsers.
- Editor parser: `editor_pomodoro.rs` and `editor_classify.rs` produce the same spec,
  spans (digits inside `pomodoro_close`), diagnostics (`invalid_pomodoro_close` with
  ranges on the offending bytes), and incomplete states.
- Chain position and trail guard in `draft.rs`, per `chain_scope`.
- Inline Work Log lexers in `close_log.rs` accept a sized owner token.
- `capture-rewrite` close-outcome aliases learn the prefix: `2xx → 2x!`, `2x1s → 2x1*`,
  `^r:id=2xs → ^r:id=2x*`, `--1 2xs → --1 2x*`; `x made 2xx` and `2Xx` stay literal. The
  alias must never touch the size digits.
- `capture-parse` JSON gains `pomodoro_close.size`; its human `close` line shows the
  size (`format_pomodoro_close` in `capture_parse.rs`).
- **Safety guard until `execute` lands:** `bob capture` must refuse any spec with a size
  (whole-item, link, or task close) with a usage error, code
  `pomodoro_close_size_unavailable`: "`2x!2` closes at a size, which this bob cannot
  execute yet; nothing was written". A sized spec must never execute as a plain close.
  Do not bump `GRAMMAR_REVISION`. Do not edit help text or `docs/` (the `docs` phase
  owns both).

Tests: every row of the grammar table in both parsers (unit tests under
`src/native/capture_language/tests/` plus `capture-parse` CLI tests in
`tests/cli/capture/parse_pomodoro_close.rs`), including UTF-8 span offsets after an
emoji line, `02x`, `12x`, `99x`, `2x!2@1`, incomplete `2x1,` / `2x~` needs and
completion ranges, `x made it 2x`, `x 2x faster`, `--1 2x!2` and `2x =` chain splits,
and the `2xx` alias rows in `tests/cli/capture/rewrite.rs`. Add a parity test that runs
each grammar row through both parsers and compares the spec. A `bob capture` test proves
the safety guard refuses and writes nothing. `just check` passes.

### Run sized closes with exact-or-refuse

- **Repository:** bob-cli
- **Commit:** `feat(capture): close the running Pomodoro at a stated size`

Scope: [The contract](#the-contract), the `bob capture` half of
[Machine contracts](#machine-contracts), and the terminal half of
[Presentation](#presentation).

- Replace the `pomodoro_close_size_unavailable` guard with the shared exact-or-refuse
  helper, wired into all three close entry points.
- Refusals with the exact copy, codes, and `extra.close_size` from
  [Refusal copy](#refusal-copy); human output prints both lines.
- `bob capture` JSON `pomodoro_close.size`; `planned`/`closed`/`closed_at`/
  `remaining_minutes`/`decremented_minutes` as specified.
- Human caption with `size.from → closed`, the size chip text, and no overrun line; the
  `tip:` line.
- `is_reset_spec` false for sized specs; `2x0` closes.
- Bump `GRAMMAR_REVISION` to 20 (`src/native/capture_language/mod.rs`) and update any
  test that pins 19.
- No help or `docs/` edits (the `docs` phase owns both).

Tests (`tests/cli/capture/pomodoro_close.rs` or a new `pomodoro_close_size.rs`, with
`BOB_NOW`):

| Area                    | Cases                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Request equivalence     | `2x!2` and `-3 x!2` on `0900-0925` at 09:12 write the same span (`0900-0910`) and the same task outcomes; the only intended differences are Work Log stamps (`C` = 09:10 versus wall 09:12) and the chain's `ran 2m over` line                                                                                                                                                         |
| Exact timing            | 25→10 late (09:50); 25→10 inside the round-up (09:08); 25→35 lengthen; `5x` on 25m with no overrun; non-grid `0902-0925` → `0902-0912`; `[off::]` key and trailing metadata preserved; a unit test of the timing helper for midnight (S 23:50, now 00:07 on the next date, `3x` → `E′` 00:05 and `C` 00:05 on the next date), since a CLI run after midnight reads the next day's note |
| Refusals                | `2x` at 09:03 and `7x` at 09:12 (name `1x` / `3x`); `1x` at the start minute; a not-started MEET; `20x`; an unparseable range; no session; several running. Each leaves every file byte-identical and reports its `code`                                                                                                                                                               |
| Clock effects           | Work Log stamped at `C`; the two-meeting MEET (`0900-1000` holding Design review `0900-0930` and Vendor sync `0930-1000`): `6x` at 10:15 completes Design review and carries Vendor sync as `not_started`; an explicit `!` on Vendor sync's number still completes it                                                                                                                  |
| Selection orthogonality | `2x` vs `x` and `2x!2` vs `x!2` give identical task outcomes; `2x0` closes and defers every plain link                                                                                                                                                                                                                                                                                 |
| Composition             | `2x =` starts at now rounded up with no overlap on a grid session; `--1 2x`; `2x x=fix~`; `^r:id=2x!1` and `<text> @r:id=2x` (numbers after the link); `--dry-run` writes nothing                                                                                                                                                                                                      |
| Output                  | Shrink, lengthen, same-size, and off-clock captions; JSON `size` present only when sized; unsized close JSON byte-identical to before; the `tip:` appears for `-3 x!2` and not for `2x!2`, `++1 x`, or `-3 x` that the close trimmed further                                                                                                                                           |

`just check` passes.

### Document `Nx` and record the decision

- **Repository:** bob-cli
- **Commit:** `docs(capture): document the sized Nx close`

First read the `docs.md` and `cli_rules.md` reference memories.

- `docs/capture.md`: the Grammar-at-a-glance row for `[N]x…`; a new
  `#### Closing at a size` subsection (anchor `closing-at-a-size`) under
  `### Closing the running Pomodoro` with the one rule, the exact-or-refuse table,
  refusal examples, the hint, and the mnemonic; a sentence in
  `#### Event Task links close by the clock` that a sized close's `C` is its sized end;
  rows in the chaining and link-close sections; prose-table rows for `2X4 lumber`,
  `x made it 2x`, the `chain_scope` lines, `0x`, `100x`, `2x=3`, `=2x`; update every
  "never extended" claim about `x` to point at `Nx` for lengthening.
- `docs/reference/capture-syntax.md`: a token-index row, the newly claimed prose shapes
  with the `X` escape, and the reserved 3+-digit shape.
- Help text: the `x` paragraph in `src/native/capture/cli.rs` and the close and mnemonic
  text in `src/native/capture_parse.rs`; refresh help fixtures under
  `tests/fixtures/help/` if they change; run `just docs-gen`, `just docs-check`, and
  `just check`.

> [!decision] decision_record Through `/sase_memory_write`, add the strand
> `decisions:sized-close-records-exact-blocks`, titled "A Sized Close Records Exactly N
> Blocks At Its Sized End".
>
> - **Applies to.** bob-cli and Bob Mac Capture.
> - **Claim.** The exact-or-refuse rule; `C = min(S + 5N, now)` for every clock effect
>   (Event Task classes, Work Log stamps, `closed_at`, the report); never a reset; one
>   item, one card, one JSON close.
> - **Why.** The research report's audit (35% of 356 closes needed a different size; 0
>   of 125 resizes would have been refused) and its rejected alternatives: snap,
>   cut-with-warning, exact-into-the-future, lexing into `-k x`, sigil-or-boundary
>   claiming, suffix sizes, minutes.
> - **Cost.** The first digit-led operator claims `2x4 lumber`-style item starts (escape
>   `2X`); one token carries two kinds of numbers.
> - **Reopens when.** A legitimate close is refused in practice, or Bryan wants a
>   clock-time (`0910x`) or minute spelling.
> - **Evidence.** The research ref, this plan, and the `grammar` and `execute` commits.
>
> Link `[[decisions/event-task-links-close-by-the-clock]]`, whose `C` this record
> defines for sized closes; that record gets no supersession mark. Then run
> `sase memory init`.

> [!decision] decision_record = no Add no memory and file nothing.

### Present sized closes in Bob Mac Capture

- **Repository:** bob-mac-capture (open it with `/sase_repo`; read its `AGENTS.md`
  first)
- **Commit:** `feat(mac-capture): present sized Nx closes`

- Decode `pomodoro_close.size` (with `from` and `delta_minutes`) using `decodeIfPresent`
  in `Sources/CaptureCore/CaptureModels.swift`; old payloads decode unchanged.
- `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`: when `size` is present,
  the session line reads `from → closed · Nm` with an arrow for shrink and lengthen
  alike (no arrow when equal); the `Nx · as of HH:MM` chip (from `closed_at`) replaces
  early/over in the session-operator tint; the accessibility sentence from
  [Presentation](#presentation). Unsized closes render exactly as before.
- Alias prefilter: widen `CapturePanelController.closeAliasPrefilterAllows` so the line
  before the caret matches `(^|[ \t=])[0-9]{0,2}x`, letting Bob decide `2xx → 2x!` and
  `^r:id=2xs → ^r:id=2x*`. Bob still decides every alias; never rewrite the digits
  locally.
- Fixtures: extend `Scripts/generate-close-alias-fixtures.sh` with sized rows and record
  a sized-close `capture --dry-run --format json` payload plus a refusal payload from a
  bob built at bob-cli master (which now includes `execute`); keep the old-bob fixtures.
- Tests: presentation for shrink, lengthen, same size, and an old payload without
  `size`; the refusal message renders both lines legibly; prefilter cases (`2xx`,
  `12xs`, `--1 2xx`, `^r:id=2xs`, prose `Plan 2xx` filtered out); the comma assist never
  turns `12x` into `1,2x`.
- Docs: the close-card section of `docs/capture-panel.md` and the alias rows in
  `docs/keyboard.md`, linking bob-cli's `docs/capture.md#closing-at-a-size`.
- Gate: never land uncompiled Swift. Run `just portable-check`, the mac scratch-copy
  `just typecheck-tests`, and `Scripts/macos-ci-check` before pushing; after the push,
  watch the exact-SHA macOS run to green with `/sase_monitor`.

## Out of scope

- Plain `x` and the `±k x` chain stay unchanged, including plain `x` classifying Event
  Tasks at now on an overrun.
- A clock-time close (`0910x`) and minute sizes (`10mx`): reserved or rejected for v1.
- An Obsidian `N<Ctrl+Enter>` sized close in bob-plugins: an optional later follow-up;
  no plugin consumes capture grammar today.
- `tests/fixtures/event_task_close/vectors.json` needs no new vectors: only the `C`
  passed in is new.

## Verification

- `grammar`: every grammar row parses identically in both parsers; sized specs refuse in
  `bob capture` and write nothing.
- `execute`: the test table above passes; `2x!2` and `-3 x!2` agree on span and task
  outcomes; `bob capture --format json` on an unsized close is byte-identical to before.
- `docs`: `just docs-check` passes; help fixtures match.
- `mac`: macOS CI green on the exact pushed SHA.
