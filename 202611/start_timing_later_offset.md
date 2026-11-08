---
tier: epic
title: Signed Pomodoro start timing — `+N` starts later
goal:
  Inside a start's `<X>`, `-` starts earlier and `+` starts later (a lone sign is one
  five-minute unit), so `=fix2+` writes exactly what `=fix2 ++` writes, `@foo:bar=+3`
  links and starts a task 15 minutes later in one item, `x=` stays a past-only record
  with teaching refusals, and Obsidian `se<X>` keeps byte-for-byte parity.
decisions:
  record_signed_start_timing:
    ask:
      Should bare_plus record the accepted design as the new decision strand
      decisions:start-timing-is-signed?
    memory:
      - decisions:start-timing-is-signed
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
phases:
  - id: signed_timing
    title: Signed `<X>` grammar, later clock, contracts, and link starts in bob-cli
    size: medium
    depends_on: []
    description:
      "signed_timing: extend `<X>` from `[D][-[N]]` to `[D][±[N]]` in bob-cli. Add
      `later_units` to the start spec and summary, move the start later in
      `start_block_minute`, refuse `+0` and two signs, accept every leading-`+` form
      (`=+3`, `=3+2`, `=+fix`, `=3+1#deep-work`, `@foo:bar=+3`, `^foo:bar=3+`), carry
      the sign into fit drafts and the human parse line, teach `x=3+`/`x=+`, bump the
      grammar revision, extend the shared MEET vectors, and document it. Trailing `+`
      after a bare name stays a name byte until bare_plus."
  - id: bare_plus
    title: Trailing `+` after bare names is timing; `#` spells names ending in `+`
    size: medium
    depends_on:
      - signed_timing
    description:
      "bare_plus: a bare start name now ends at its last byte that is neither a digit
      nor `+`, so `=fix2+`, `=fix+3`, `=bugs+2`, and `@foo:bar=fix2+` are timing while
      `=bob+decks` and `=bob+2fa` are unchanged. Refuse `=c++`/`=fix++` with a three-way
      hint, make `=#c++` legal, teach `x=fix+`, keep timing in completion round trips,
      emit trailing timing in bare-name fit drafts, fix the stale `=bugs+2` docs, add
      the differential test, bump the grammar revision, and record
      decisions:start-timing-is-signed."
  - id: link_body
    title: Refuse operator-only bodies after a link start, teaching the glued spelling
    size: small
    depends_on:
      - bare_plus
    description:
      "link_body: an `@route:id[#name]=<X>` item whose remaining text is only
      session-operator tokens (`@foo:bar= ++3`) is refused instead of creating a task
      titled `++3`; a single `++N` body teaches the folded spelling (`@foo:bar=+3`,
      `@foo:bar=fix2+`), and the `^route:id=<X> ++N` near miss teaches the same fold.
      Bump the grammar revision and document it."
  - id: obsidian_parity
    title: Obsidian bob-ledger-tools `se<X>` learns `+` in bob-plugins
    size: small
    depends_on:
      - signed_timing
    description:
      "obsidian_parity: in bob-plugins, teach the bob-ledger-tools `se<X>` snippet
      `se+`, `se+3`, `se3+`, and `se3+2` with the same later-start math, refuse `se+0`
      and two signs, carry the sign into the MEET-conflict fit token, re-copy the shared
      meet_event_run vectors from bob-cli, map signed tokens in the harness, update the
      plugin docs, build, test, and run `bob plugins sync`."
proposed_by: bbugyi200.athena.research.5x.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 05:50:20
status: wip
---

- **PROMPT:**
  [prompts/202611/start_timing_later_offset.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/start_timing_later_offset.md)

# Signed Pomodoro start timing: `+N` starts later

## The rule users learn

> Inside `<X>`, `-` starts earlier and `+` starts later. A sign with no number means
> five minutes. A space makes it a second step.

| Glued: one start                           | Spaced: start, then a second operation |
| ------------------------------------------ | -------------------------------------- |
| `=-2`: start 10 min earlier                | `= -2`: start, then shorten by 10 min  |
| `=+2`: start 10 min later (**new**)        | `= +2`: start, then extend by 10 min   |
| `=fix2+`: FIX starts 5 min later (**new**) | `=fix2 ++`: start, then shift later    |

Read the design source before starting any phase:

```bash
sase artifact read research:202611/start_timing_later_offset/start_timing_later_offset.md "Design source for the signed start timing epic"
```

Bryan accepted every requirement in that report, including its eight adjustments. This
plan fixes the details the report left open and splits the work. Where the two differ,
this plan wins.

Related beads, for context only (do not fix them here):

- `bob-cli-h1` covers past-offset starts wrapping at midnight. `=+3` at 23:55 wraps
  forward to `0010-0035`, exactly as `= ++3` and `=` at 23:58 already do. Add no new
  midnight guard.
- `bob-cli-ik` is the "starts in Nm" lead-time status follow-up. It is out of scope.

## Shared design contract (all phases)

### Grammar and clock

```text
<X>       ::= [duration] [offset]
duration  ::= [0-9]+                 omitted → 5 units (25 min)
offset    ::= "-" [0-9]*             earlier; omitted digits → 1 (`-0` stays legal)
            | "+" [0-9]*             later;   omitted digits → 1; `+0` refused
start = ceil((nowMinutes − 5·earlier + 5·later) / 5) · 5        end = start + 5·duration
```

- One sign per `<X>`. Both signs, or a repeated sign, is a refusal. Never net them out.
- Leading zeros are fine (`+03`). There is no cap on N: `=+24` is two hours later and
  draws no warning.
- `start_block_minute` (`src/native/capture/meet_event_run.rs`) is the only rounding
  path. Add `5·later` there with checked arithmetic. Overflow returns the existing
  "start suffix is too large" error.
- Every offset is a multiple of 5, so whenever `<X>` is non-empty, `B+N` writes the same
  span as `B ++N`.

Worked examples at `BOB_NOW` 09:42 with nothing running (the unshifted start is 09:45):

| Draft              | Reading                                  | Writes                | Same as                       | Phase         |
| ------------------ | ---------------------------------------- | --------------------- | ----------------------------- | ------------- |
| `=+`               | next, 25 min, later 1                    | 0950–1015             | `= ++`                        | signed_timing |
| `=+3`              | next, 25 min, later 3                    | 1000–1025             | `= ++3`                       | signed_timing |
| `=3+`              | next, 15 min, later 1                    | 0950–1005             | `=3 ++`                       | signed_timing |
| `=3+2`             | next, 15 min, later 2                    | 0955–1010             | `=3 ++2`                      | signed_timing |
| `=+fix`, `=3+1fix` | FIX, 25 / 15 min, later 1                | 0950–1015 / 0950–1005 | —                             | signed_timing |
| `=3+1#deep-work`   | DEEP-WORK, 15 min, later 1               | 0950–1005             | —                             | signed_timing |
| `@foo:bar=+3`      | link, start 25 min, later 3              | 1000–1025             | two items `@foo:bar=` / `++3` | signed_timing |
| `=fix2+`           | FIX, 10 min, later 1                     | 0950–1000             | `=fix2 ++`                    | bare_plus     |
| `=fix2+3`          | FIX, 10 min, later 3                     | 1000–1010             | `=fix2 ++3`                   | bare_plus     |
| `=bugs+2`          | BUGS, 25 min, later 2                    | 0955–1020             | `=bugs ++2`                   | bare_plus     |
| `=fix2+~1`         | FIX, 10 min, later 1, drop queued link 1 | 0950–1000             | —                             | bare_plus     |
| `@foo:bar=fix2+`   | link under FIX, 10 min, later 1          | 0950–1000             | —                             | bare_plus     |
| `=fix-`            | FIX, 25 min, earlier 1                   | 0940–1005             | unchanged                     | —             |

### Behavior in every start context

| Context                                                              | Behavior with `+N`                                                                                                                                                      |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fresh start                                                          | Starts the selected entry at the later span: the same note bytes as `… ++N`                                                                                             |
| Restart (`=+`, `=+capture`, later `=capture+`, while CAPTURE runs)   | Fresh timing from the later start, with the existing override warning, like `=3capture` ≡ `=3`                                                                          |
| Swap (`=+fix`, later `=fix+`, while BUILD runs 09:00–09:50 at 09:42) | Fresh timing **0950–1015**, exactly like `=fix-`. It never keeps BUILD's span and shifts it. `=fix ++` stays the keep-then-shift spelling                               |
| Link starts (`@r:id=+N`, `@r:id#name=+N`, `^r:id=+N`)                | Same guards as today: a link start never overrides a running session, and it takes no drop list                                                                         |
| MEET with a calendar run                                             | Only an empty `<X>` follows the calendar (`spec.raw.is_empty()`). `=+meet` (later `=meet+`) is written timing, `event_run.source = explicit`, like `=meet-`             |
| Unnamed start near a meeting                                         | The target is chosen by `[S, S+5)`, where S is the later start (`unnamed_start_target_with`), so a later start can join a meeting the plain `=` would not               |
| Event warnings and fit drafts                                        | Checked against the final span. A fit draft keeps the sign: `{units}+{later}`                                                                                           |
| Same-line chains                                                     | `x =+`, `x =+fix`, `=+3 +2` (later `x =fix+`, `=fix+ +2`) run left to right with no chain-specific code (`session_equals_token` already feeds `is_session_chain_token`) |
| Prose and near misses                                                | `=+ foo` and `=+)` stay prose, like `=- foo` and `=-)`. `Plan =+3` (mid-body) stays prose. `=+3 foo` gets the same counted-token-plus-text error as `=-3 foo`           |
| Undo                                                                 | `--N` shifts back, and `x0` resets a session that has no notes                                                                                                          |

### Reading a `+` after `=`

- A spaced `+` is a separate session operator (`=fix2 ++`).
- A `+` inside a `#` name or an exact-name literal is a name byte (`=3#c++`,
  `@sase:deep-fix#c++`).
- A glued `+` directly after `=` or after leading duration digits is the later offset
  (signed_timing).
- After a bare name (bare_plus), the name ends at its **last byte that is neither an
  ASCII digit nor `+`**, and everything glued after it is timing. This generalizes
  today's rule that a bare name cannot end in a digit:

  | Token                                | Name      | Duration  | Later                                                    |
  | ------------------------------------ | --------- | --------- | -------------------------------------------------------- |
  | `=fix2+`                             | FIX       | 2         | 1                                                        |
  | `=fix+3`                             | FIX       | 5         | 3                                                        |
  | `=bob+decks`                         | BOB+DECKS | 5         | — (unchanged)                                            |
  | `=bob+decks2+3`                      | BOB+DECKS | 2         | 3                                                        |
  | `=fix+read2`                         | FIX+READ  | 2         | — (unchanged)                                            |
  | `=bob+2fa`                           | BOB+2FA   | 5         | — (unchanged: ends in a letter)                          |
  | `=b2b+2`                             | B2B       | 5         | 2                                                        |
  | `=xx+`                               | XX        | 5         | 1                                                        |
  | `=#c++`, `=3#c++`, `=+#c++`          | C++       | 5 / 3 / 5 | — / — / 1 (newly legal: no longer a "redundant `#`")     |
  | `=c++`, `=fix++`, `=fix2++`, `=c++3` | —         | —         | refused with the doubled-plus hint                       |
  | `=+fix2`                             | —         | —         | the existing "one timing" refusal (timing on both sides) |

  The decision is purely lexical. Never pick "name" or "offset" based on whether a
  matching session exists in the vault.

- Bare names keep timing in the trailing position (`=fix2+1`), which is canonical.
  Leading timing stays accepted for bare names, and it is required before `#` names
  (`=3+1#deep-work`). Timing after a `#` literal is already a retired spelling, and its
  existing error rewrites to the leading form (`=#"DEEP WORK"+3` → `=+3#"DEEP WORK"`).

The theme model loses nothing. `split_components` (`src/native/plan_budget/mod.rs`)
already drops empty components, so `C++` and `FIX+` were always the themes `C` and
`FIX`. The vault survey in the research found no ledger name that ends in `+` or
`+digits`.

### Refusals that teach the fix

Each refusal uses the existing diagnostic code for its family: `invalid_pomodoro_start`
for `=…`, `invalid_pomodoro_past` for `x=…`, and `invalid_pomodoro_link` for a link
body. Each message names the typed token and builds every suggestion from the user's own
text. Wording may be polished, but every listed teaching element must stay.

| Typed                                                  | Message content                                                                                                                                                                                              | Phase                                  |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------- |
| `=+0`, `=3+0`, `=fix2+0`                               | "`+0` moves nothing: drop it (`=fix2`), or write `=fix2+` to start 5 minutes later"                                                                                                                          | signed_timing (bare form in bare_plus) |
| `=3-1+2`, `=+1-2`, `=3+1+2`, `=2-1-`                   | "`=3-1+2` takes one offset: `-` starts earlier, `+` starts later (`=3-1` or `=3+2`)", using the token's own numbers and 1 when a count is missing                                                            | signed_timing                          |
| `=fix++`, `=c++`, `=fix2++`, `=c++3`, `@r:id=c++`      | "`=fix++` ends in two `+`: `=fix+` starts FIX 5 minutes later, `=fix ++` starts it and then shifts it later, and `=#fix++` names a Pomodoro `FIX++`" (drop one `+`; put a space before the `++`; prefix `#`) | bare_plus                              |
| `x=3+`, `x=+`, `x=+2`                                  | "`x=3+` records finished work, so it takes no `+`: `=3+` starts a 15-minute session 5 minutes from now, and `x =3+` closes the running session first"                                                        | signed_timing                          |
| `x=fix+`, `x=fix2+`, `x=c++`                           | The same message plus "; spell a name that ends in `+` with `#` (`x=#c++`)"                                                                                                                                  | bare_plus                              |
| `@foo:bar= ++3`, `@foo:bar=fix2 ++`                    | "`++3` after `@foo:bar=` would become the new task's text: write `@foo:bar=+3` to link and start it 15 minutes later, or put `++3` on its own item"                                                          | link_body                              |
| The general shape error (`POMODORO_START_SHAPE_ERROR`) | Lists the new shapes: empty, digits, or optional digits followed by one `-` (earlier) or `+` (later) with optional digits (`=`, `=3`, `=-2`, `=+2`, `=3+1`)                                                  | signed_timing                          |

The two-signs refusal claims the whole glued run of digits and signs, so its diagnostic
range covers all of `3-1+2`. A digit-free run that is prose today (`=-)`, `=- foo`)
stays prose, and so does its `+` twin. `=2-1-` moves from the generic "extra text" error
to the two-signs message; update its row in `docs/capture.md`.

### Contracts

- **Model.** `PomodoroStartSpec` (`src/native/capture_language/model.rs`) and
  `PomodoroStartSummary` (`src/native/capture/batch.rs`) gain `later_units: u64`,
  serialized only when it is nonzero (`skip_serializing_if`). `offset_units` keeps its
  meaning and is now documented as "earlier units", which keeps older readers correct.
  `raw` keeps the typed sign. Example:
  `{"raw":"2+","duration_units":2,"offset_units":0,"later_units":1}`. `schema_version`
  is unchanged. Bob Mac Capture needs no change: it renders bob's spans and resolved
  `start`/`end`, ignores unknown keys, and its grammar gate is `>= 15`
  (decisions:mac-capture-is-a-thin-client).
- **Human parse line.** `format_pomodoro_start` (`src/native/capture_parse.rs`) shows
  the signed offset: `=fix2+ (10m, offset +1u)`, `=-2 (25m, offset -2u)`,
  `=3 (15m, offset 0u)`. The drop variant follows the same pattern.
- **Grammar revision.** Each bob-cli phase that changes accepted text raises
  `GRAMMAR_REVISION` (`src/native/capture_language/mod.rs`) by exactly one from whatever
  master holds when the phase lands. Do not hard-code 17. Then update the
  `capture-parse`/`capture-complete` help text and the generated `docs/reference/cli/*`
  (`just docs-gen`) wherever a revision number is printed.
- **Commits.** Each bob-cli phase is `feat(capture)!:`, because accepted text changes
  meaning. Never touch `Cargo.toml`'s version or `CHANGELOG.md`
  (decisions:bob-cli-releases-ride-a-daily-train).
- **Docs.** Before editing `docs/` or CLI help, read `sase/memory/docs.md` and
  `sase/memory/cli_rules.md` with `/sase_memory_read`. Behavior changes go to the
  contract and reference pages (`docs/capture.md`, `docs/reference/capture-syntax.md`,
  `docs/reference/json-outputs.md` if it lists the start keys), and to guide steps only
  where they become wrong. Run `just docs-gen`, then `just docs-check`.
- **Shared MEET vectors.** Only signed_timing edits
  `tests/fixtures/meet_event_run/vectors.json`. bare_plus and link_body put their MEET
  cases in Rust-only tests, so the copy in bob-plugins never goes stale.

### Out of scope

- `x=` never gains an offset. It records finished work, and inside it `+` already means
  a segment's far edge (`x=~1+`, `x=fix~1+`). Those stay byte-identical.
- No change to Bob Mac Capture. No "starts in Nm" status (`bob-cli-ik`). No midnight
  guard (`bob-cli-h1`). No retirement of `-0`.

## Phase `signed_timing` — bob-cli

**Lexer.** `start_timing_len` (`src/native/capture_language/item.rs`) accepts one sign
of either kind after optional digits: `[0-9]*([-+][0-9]*)?`. It also claims any further
glued sign and digit run as a near miss, which the parser refuses with the two-signs
message. `session_equals_token` already calls it in the leading position, after a `#`
literal (retired), and after `==` (retired), so `=+3`, `=3+2`, `=+fix`, `=3+1fix`,
`=3+1#deep-work`, and `=+#"DEEP WORK"` lex through it. Keep the bare-name peel
(`bare_start_name_len`) unchanged in this phase, so `=fix2+` still reads as the name
`FIX2+` until bare_plus.

**Parser.** `parse_pomodoro_start_suffix` (`src/native/capture_language/tokens.rs`)
fills `later_units`, refuses `+0`, and refuses two signs. Its
`raw.contains('=' | '#' | ':')` guard stays. Update `POMODORO_START_SHAPE_ERROR`
(`markers.rs`). The link-start parser (`tokens.rs`, the function that builds `={raw}`
and calls `session_equals_token`) should need no structural change. Verify
`@foo:bar=+3`, `@foo:bar=3+`, `@foo:bar#fix=+2`, `@foo:bar=+fix`, `^foo:bar=+3`, and
new-task `Write outline @sase:outline=+2`.

**Clock and execution.** Update `start_block_minute`, then thread `later_units` through
every `PomodoroStartSpec`/`PomodoroStartSummary` construction. There are about 14 spec
literals, plus the summary builders in `src/native/capture/pomodoro_start.rs`,
`pomodoro_link.rs`, and `meet_event_run.rs` (its `offset_units: Option<u64>` override
for calendar spans needs a `later_units` twin set to 0). Leave the MEET calendar rule
(`spec.raw.is_empty()`) alone.

**Fit drafts.** `start_fit_token` (`src/native/capture/event_conflict.rs`) writes
`{units}+{later}` when `later_units > 0`, and keeps today's name placement in this phase
(`=2+1fix`, `=2+1#deep-work`). bare_plus switches bare names to trailing.

**Completion and editor.** Leading-timing completion (`completion.rs`, the
`bare_start_name_len(after_digits)` path) must skip a leading `+` run just as it skips
`-`. Accepting `FIX` at `=3+1fi|` yields `=3+1fix`. Editor spans (`editor_pomodoro.rs`,
`editor_classify.rs`) report `pomodoro_start` over `=` plus the signed `<X>`. Editor and
executed parses must agree on every new token.

**Past teaching.** `past.rs` refuses `x=3+`, `x=+`, and `x=+2` with the past-later
message, replacing today's generic `invalid_pomodoro_past` text for those shapes. Build
that message in a helper that bare_plus will reuse for named forms. `x=~1+` and `x=2~1+`
stay unchanged.

**Contracts.** Add `later_units` to the JSON, sign the human line, update the
`capture-parse` help prose that describes `pomodoro_start` as
"`duration_units`/`offset_units`" (it gains `later_units`), and raise the grammar
revision by one.

**Shared vectors.** Add `tests/fixtures/meet_event_run/vectors.json` cases that the Rust
test (`tests/cli/capture/meet_event_run.rs`) runs:

- `=+2`, an unnamed later start that reaches a MEET run through `[S, S+5)` with
  `expected_source: explicit`.
- `=+meet`, which writes explicit timing rather than following the calendar.
- One non-MEET `=+` whose later span newly crosses an event, so the fit draft carries
  `+`.

Keep the ids distinctive (each contains `later`). Update the README's "Pin:" sentence to
name the commit that adds them, and note that bob-plugins maps `=<signed X>` tokens to
`se<signed X>`.

**Tests.** At minimum, cover these:

- Unit tests (`src/native/capture_language/tests/grammar.rs`, `editor_modes.rs`,
  `editor_spans.rs`, `completion.rs`, `past.rs`) for every signed_timing row in the
  tables above, including byte spans.
- Unchanged forms: `= +2`, `=+ foo`, `=+)`, `Plan =+3`, `x=~1+`, `=-2bugs`, and `=fix-`.
- CLI tests under `tests/cli/capture/`: `pomodoro_start.rs`, `pomodoro_start_named.rs`,
  `pomodoro_link.rs`, `parse_pomodoro.rs`, `parse_pomodoro_past.rs`,
  `event_start_warnings.rs`, and `pomodoro_chain.rs` (`x =+`, `=+3 +2`).
- Clock cases: 09:42, an exact boundary with nonzero seconds, 23:55 (wraps to 0010), and
  overflow.
- The restart and swap rows, with the swap `=+fix` writing 0950–1015.
- A transaction case: a failing later item in a bulk draft leaves every touched file
  byte-identical.
- **A differential test.** For each unsigned base B with a non-empty `<X>` (`=3`,
  `=3fix`, `=3#deep-work`, `@foo:bar=3`), run B with `+N` glued in this phase's accepted
  position (`=3+N`, `=3+Nfix`, `=3+N#deep-work`, `@foo:bar=3+N`) against `B ++N` (the
  two-item draft `B` / `++N` for link starts) at the same `BOB_NOW` from the same
  fixture, for N in {omitted, 1, 3}. Compare **final note bytes**, not response
  envelopes.

**Docs.** In `docs/capture.md`:

- The `<X>` table near "`<X>` is exactly the suffix of the Obsidian bob-ledger-tools
  `se<X>` snippet" gains `+`, `+3`, `3+`, and `3+2` rows plus the one-sentence rule.
  Leave the parity sentence as written: obsidian_parity makes it true again.
- The recognition grammar sentence (`[0-9]*(-[0-9]*)?`) and the bare/counted
  definitions.
- The `=`/`=<X>` rows in the grammar-at-a-glance tables.
- The restart/swap result table (add `=+` and `=+bugs` rows).
- The fit-draft list (`=2+1`).
- The `x=` section (the new teaching error).
- The diagnostics catalog entry for `invalid_pomodoro_start`.

Mirror the rows in `docs/reference/capture-syntax.md`. Add `later_units` wherever
`offset_units` is documented. Do not touch `docs/capture.md`'s `=bugs+2` rows yet;
bare_plus rewrites them.

## Phase `bare_plus` — bob-cli

**Name peel.** In `src/native/capture_language/bare_name.rs`:

- `bare_start_name_len` strips the maximal trailing run of ASCII digits **and** `+`
  (where it strips digits today) before its close-shape check. `fix2+` → `fix`,
  `bob+decks2+3` → `bob+decks`, `bob+2fa` → `bob+2fa`.
- `can_spell_bare` also rejects names that end in `+`.
- `trailing_glued_timing` lexes the signed timing (`fix2+~1` → `2+`, `de3+1` → `3+1`).
- Update the module and function doc comments and the unit tests in that file. Today's
  assertions such as `bare_start_name_len("bob+decks2") == Some(9)` stay true.

**Callers.** `session_equals_token` gets the trailing timing through `start_timing_len`
from signed_timing. Then:

- Implement the doubled-plus refusal for `=c++`, `=fix++`, `=fix2++`, `=c++3`, and
  `@r:id=c++`.
- Make `=#c++`, `=3#c++`, and `=+#c++` legal. The redundant-`#` checks in `tokens.rs`
  and `past.rs` key on `can_spell_bare`, so this follows from the `can_spell_bare`
  change.
- Pin `=x+` with a test. It must fail with a clear error and never start a session named
  `X` or `X+`.

**Past.** `past.rs` takes a bare name and then glues a trailing count. When a `+`
follows, refuse `x=fix+`, `x=fix2+`, and `x=c++` with the named past-later message (the
signed_timing helper plus the `#` clause). `x=#c++` records a Pomodoro named `C++`.
`x=fix~1+` and `x=fix3~1+` are unchanged.

**Completion.** `src/native/capture_complete/pomodoros.rs` (the `trailing_glued_timing`
fold) and `completion.rs` must keep the sign and any drop list:

- Accepting `FIX` at `=fi|2+` yields `=fix2+`.
- Accepting `DEEP WORK` / `deep-work` at `=de|3+1` yields `=3+1#deep-work`.
- The same holds with the caret inside the name, on the `+`, and inside the digits.
- Cover bare, digit-ending, hyphenated, multi-word, and plus-bearing names
  (`bob+decks`).

**Fit drafts.** `start_fit_token` composes through `glued_named_start("", name, timing)`
so bare names get trailing timing (`=fix2+1`, `=bugs2`) and `#` names leading timing
(`=2+1#deep-work`). Update the fit-draft list in `docs/capture.md` (`=2bugs` → `=bugs2`)
and the existing fit tests.

**Tests.**

- Every bare_plus row in the token and worked-example tables, with byte spans, in the
  unit suites.
- Flip the `=c++` entries in `src/native/capture_language/tests/editor_modes.rs` to the
  refusal. Editor and executed parse must still agree.
- CLI coverage in `pomodoro_start_named.rs`, `pomodoro_link.rs` (`@foo:bar=fix2+`), and
  `parse_pomodoro_past.rs`.
- Swap `=fix+` while BUILD runs → 0950–1015. Restart `=capture+` ≡ `=+`.
- `=meet+` is explicit timing. Put this in a Rust-only MEET test, not the shared vectors
  file.
- Extend the signed_timing differential test with the trailing forms (`=fix2+N` vs
  `=fix2 ++N`, `@foo:bar=fix2+` vs the two-item draft).

**Docs.**

- Rewrite the stale rows that say `=bugs+2` starts a Pomodoro named `BUGS+2` (the
  bare-name rows near the start of `docs/capture.md`, and the chaining section's
  "`=bugs+2` is one start of the Pomodoro named `BUGS+2`"). `=bugs+2` now starts BUGS 10
  minutes later.
- Document the plus-reading rule and the token table in the named-start section, and `#`
  for names ending in `+`.
- Add `=bugs+` and `=capture+` rows to the restart/swap table.
- Mirror everything in `docs/reference/capture-syntax.md`.
- Raise the grammar revision by one.

**Decision record.**

> [!decision] record_signed_start_timing Write the strand below. When the reviewer
> answered no, skip it; if no human reviewed the plan, record a `PROPOSED FOLLOW-UP:`
> note on the phase bead instead.

Use `/sase_memory_write`, then add the strand `decisions:start-timing-is-signed` (slug
`start-timing-is-signed` in the decisions web) in the house format of its siblings
(frontmatter `keyword: Start Timing Is Signed`, aliases such as `later offset` and
`plus timing`, a one-line `summary`, `metadata.status: accepted`, and `decided:` set to
the landing date). The body has these sections:

- **Applies to:** bob-cli, bob-plugins, Bob Mac Capture.
- **Claim:** start `<X>` takes one sign, `-` earlier and `+` later, where a lone sign is
  one unit and `+0` is refused. A bare start name ends at its last byte that is neither
  a digit nor `+`, and `#` spells names that end in `+`. `x=` takes no offset. Non-empty
  timing is fresh in swaps and MEETs.
- **Why:** the rejected alternatives are resolve-then-shift, glued `++`, leading-only
  `+`, digit-only `+`, a signed `offset_units`, and `x=` offsets. Cite the research ref
  above and the landing commits.
- **Cost:** `=+…` stopped being prose, names ending in `+` need `#`, and every change
  must stay in parity with Obsidian `se<X>`.
- **Reopens when:** ledger names ending in `+` or `+digits` become common, or starts
  need absolute clocks.

Run `sase memory init`. Edit no other memory.

## Phase `link_body` — bob-cli

Today `@foo:bar= ++3` silently creates a task titled `++3` (or fails when `^bar`
exists), because text after a link start becomes the new task's body. Change that
behavior:

- Refuse an `@route:block-id[#name]=<X>` item whose remaining text, on its first
  physical line and in either position, is non-empty and made up entirely of tokens for
  which `is_session_chain_token` holds (for example `++3`, `+2`, `--`, `=fix`; reuse
  that predicate rather than listing operators). Use `invalid_pomodoro_link`, ranged on
  the operator text.
- When that text is exactly one `++[N]` token and the link's `<X>` has no sign, the hint
  folds it into `<X>`, keeping the typed digits:
  - `@foo:bar= ++3` → `@foo:bar=+3`
  - `@foo:bar=3 ++2` → `@foo:bar=3+2`
  - `@foo:bar=fix2 ++` → `@foo:bar=fix2+`
  - `@foo:bar=fix ++` → `@foo:bar=fix+`
  - `@foo:bar#deep-work=3 ++` → `@foo:bar#deep-work=3+`

  Otherwise the hint says to put the operators on their own item.

- `^route:block-id=<X> ++N` is already refused as extra text. Give it the same fold
  hint.
- Prose bodies that merely contain an operator (`@foo:bar= Fix the ++3 bug`) keep
  creating tasks exactly as today.

Cover this in `tests/cli/capture/pomodoro_link.rs` and in `parse_pomodoro.rs` (editor
diagnostics and ranges). Assert that each refusal writes nothing. Document it in
`docs/capture.md`, next to the link-start `=<X>` paragraph and in the
`invalid_pomodoro_link` catalog entry, and in `docs/reference/capture-syntax.md`. Raise
the grammar revision by one.

## Phase `obsidian_parity` — bob-plugins

Open the repo with `/sase_repo` (`bob-plugins`) and read its `AGENTS.md`. Edit only
`plugins/bob-ledger-tools/src/` fragments, then run `npm run build`; never hand-edit
`main.js`.

- **Triggers.** In `010-load-and-constants.js`, change `TRIGGER_RE` from
  `se\d*(?:-\d*)?` to `se\d*(?:[-+]\d*)?`, and change `LEDGER_TRIGGER_RE` to
  `^se(\d*)(?:([-+])(\d*))?$`.
- **Parsing.** `parseTrigger` (`020-time-and-pomodoro.js`) returns the sign, and returns
  `null` (no expansion) for `se+0` and for any second sign (`se3-1+2`). `se-0` keeps
  working.
- **Math.** `computeRange` (`030-snippets.js`) applies the later offset with the same
  `Math.ceil` formula, using `+ later` where it uses `- offset`.
- **MEET fit token.** `eventRunFitToken` (`035-meet-event-run.js`) emits
  `se{count}+{later}` for later starts.
- **Calendar rule.** It stays `expansion.trigger === "se"`, so `se+` is explicit.
- **Other consumers.** Grep every consumer of `dashPresent`, `offsetMultiplier`, and
  `LEDGER_TRIGGER_RE` (the home-chips, off-clock-chips, and vim/snippet fragments
  mention `se<X>`), and make each one sign-aware or confirm it is sign-agnostic.
- **Vectors.** Re-copy
  `plugins/bob-ledger-tools/scripts/fixtures/meet_event_run/vectors.json` byte-for-byte
  from bob-cli master's `tests/fixtures/meet_event_run/vectors.json` (it must contain
  signed_timing's `later` cases). This also fixes the stale `=5#meet` token. Update that
  fixture's README pin.
- **Harness.** In `scripts/test-ledger-tools-meet-event-run.cjs`, `gestureFor` maps
  `^=(\d*(?:[-+]\d*)?)` to `se${suffix}`. Gestures with no Obsidian counterpart go into
  `CAPTURE_ONLY` with a reason.
- **Snippet tests.** In `scripts/test-ledger-tools-snippets.cjs`, `se+`, `se+3`, `se3+`,
  `se3+2`, and `se+03` at 09:42 produce 0950–1015, 1000–1025, 0950–1005, 0955–1010, and
  1000–1025. `se+0` and `se3-1+2` do not expand. Every existing `se`, `se3`, `se-2`, and
  `se2-1` case is unchanged.
- **Docs.** Update `docs/bob-ledger-tools.md` and the `manifest.json` description
  wherever they spell the `se<X>` grammar.
- **Gates.** Run `npm test` and `npm run validate`, then `bob plugins sync` to deploy.

## Verification gates

- **bob-cli phases:** `just check` (fmt, clippy gate, every test binary), then
  `just docs-gen` (commit regenerated goldens and references) and `just docs-check`.
  Spot-check by hand with a scratch vault, using `BOB_NOW="2026-11-07 09:42:00"` and
  `bob capture --dry-run`, for the worked-example rows that phase owns. Also check
  `bob capture-parse -f json` for the `later_units` and span shape.
- **bob-plugins:** `npm test`, `npm run validate`, and `bob plugins sync`.
- **Acceptance for the epic.** Every row of the worked-example, token, context, and
  refusal tables above behaves as written. `=fix2+` and `=fix2 ++`, and `@foo:bar=+3`
  and the two-item draft, produce identical note bytes at the same clock. `x=` gains no
  offset. Obsidian `se<X>` matches bob for every shared vector.
