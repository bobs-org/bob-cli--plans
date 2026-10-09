---
tier: epic
title: "`==` Pomodoro override: restart the running session or swap another in"
goal: "`bob capture` and Bob Mac Capture accept `==`, the override twin of every
  whole-item `=` start: `==<X>` restarts the running Pomodoro with fresh `se<X>` timing,
  and `==[<X>]#name` swaps a different Pomodoro in as the running one (taking over the
  running session ledger unless a timing is given) while the old one returns, intact, to
  first future. Every path is atomic, dry-run exact, byte-preserving, and explained in
  both the CLI and the Mac preview.

  "
decisions:
  idle_fallback:
    ask:
      When nothing is running, should every `==` token act exactly like its `=` twin
      instead of refusing?
    default: true
    why: '`==` then always means "make this the current session" and never dead-ends'
    answer: true
phases:
  - id: override-grammar
    title: Lex, parse, and describe the `==` token family
    depends_on: []
    size: medium
    description:
      "override-grammar: teach the capture language the `==` family (`==`, `==<X>`,
      `==[<X>]#name`, `…~<K>`) with `=`-identical claim rules, prose protection for
      Obsidian highlights, teaching near-miss errors, chain support, the additive
      capture-parse `override: true` spec flag, editor needs, and `==#` name-completion
      offsets."
  - id: override-restart
    title: Execute restarts and the idle fallback, with the override JSON contract
    depends_on:
      - override-grammar
    size: medium
    description:
      "override-restart: route override starts in the executor, run the idle fallback,
      restart the running session in place with fresh `se<X>` timing, introduce the full
      `pomodoro_start.override` JSON object and `restarted` human output, and start the
      new docs section."
  - id: override-swap
    title: Execute swaps with ledger takeover and first-future demotion
    depends_on:
      - override-restart
    size: medium
    description:
      "override-swap: implement `==[<X>]#name` (resolve exactly like `=#name`, transfer
      or re-time the session ledger, demote the old running block to first future with
      contents intact), fill the `demoted` JSON, `swapped` human output, `=x0`
      equivalence tests, updated teaching errors, and finish every bob-cli doc surface."
  - id: override-complete
    title: Give the `==#` name picker its override context
    depends_on:
      - override-grammar
    size: small
    description:
      "override-complete: add the additive top-level `override` object (keeps_ledger
      plus the running session) to `bob capture-complete` for `==` name fields, with
      tests and the capture-complete docs."
  - id: mac-override-card
    title: Bob Mac Capture restart and swap preview, footer, and notifications
    depends_on:
      - override-swap
    size: medium
    description:
      "mac-override-card: decode the parse flag and `pomodoro_start.override`, extend
      the start presentation with restart/swap/idle variants, render the before→after
      and demoted rows on the start card, retitle the footer and notifications, and add
      real-bob fixtures, tests, and README rows."
  - id: mac-override-picker
    title: Bob Mac Capture `==#` picker status and row hints
    depends_on:
      - override-complete
      - mac-override-card
    size: small
    description:
      "mac-override-picker: decode the capture-complete `override` object and use it for
      the `==#` picker status line and the running/open row hints, fix the `=#`
      running-row hint to teach `==`, with fixtures and tests."
proposed_by: bbugyi200.apollo.61.w1.w0
decided_by: auto
create_time: 2026-10-09 13:24:47
status: wip
---

- **PROMPT:**
  [prompts/202610/pomodoro_override.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/pomodoro_override.md)

# Plan: `==` Pomodoro override

## Why and the one-sentence model

Today a running Pomodoro can only be resized (`+N`/`-N`), shifted (`++N`/`--N`), reset
(`=x0`, note-free only), or closed (`=x`). There is no way to say "restart this session
now" or "I'm actually doing BUGS, not CAPTURE" without writing history or juggling two
operators. `==` fills that gap.

**Mental model: `=` starts a session; `==` overrides the running one.** Two orthogonal
parts, read left to right:

- **When** — the `se<X>` suffix right after `==` (identical grammar and timing to
  `=<X>`).
- **Which** — an optional `#name` (identical resolution to `=<X>#name`).

Mnemonic for the docs lifecycle table: "`=` starts the next session, `=#name` starts
that one, `==` restarts the running one, `==#name` swaps that one in, `=x` stops the
running one."

Throughout, "ledger" means the parenthesized session ledger on the Pomodoro headline,
`(**0920-0945** [t:: 25m])` — the same span `=x0` clears to `()`.

## Behavior contract (shared by every phase)

### Grammar

| Token                     | A timed session R is running                                                                                                                              | Nothing is running  |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| `==` / `==<X>`            | **Restart** R now with `se<X>` timing (`==` = 25 minutes from now, `==3` = 15, `==-2` = 25 with a 10-minute offset)                                       | exactly `=<X>`      |
| `==#name`                 | **Swap**: `name` takes over R's session ledger byte-for-byte; R returns to first future                                                                   | exactly `=#name`    |
| `==<X>#name`              | **Swap** with fresh `se<X>` timing; R returns to first future                                                                                             | exactly `=<X>#name` |
| any of the above + `~<K>` | drops queued Task Links from the session that ends up running (R for a restart, `name` for a swap), numbered exactly as the `=` start lineup numbers them | exactly `=…~<K>`    |

> [!decision] idle_fallback = no With nothing running, every `==` token refuses (writes
> nothing) with `` nothing is running to override; start it with `=<X>[#name]` ``
> (spelled with the typed suffix/name), instead of acting like its `=` twin.
> `override.action` is then never `"start"`, and the Mac idle caption is not needed.

Why an empty `<X>` differs between the two verbs: a restart exists to re-time, so an
empty suffix means the default 25 minutes (exactly what `=` would write); a swap exists
to change _which_ Pomodoro is running, so an empty suffix keeps the clock (the requested
default). `==5#name` is the explicit "swap in with a fresh 25 minutes" spelling; teach
it in docs and errors.

Lexical rules (all of them mirror `=`, just with the doubled sigil):

- Token = `==` + `se<X>` suffix `[0-9]*(-[0-9]*)?` + optional `#name` (ends at ASCII
  whitespace or `~`) + optional `~<K>`. The second `=` must follow the first directly.
- Claim policy mirrors `=`: an exact token claims its item; counted, named, or
  drop-carrying tokens claim it even with extra text (precise errors, never a task). A
  bare `==` followed by more text stays prose — this protects Obsidian highlights:
  `==important== thing`, `==foo`, `== foo`, `===`, and mid-body `Plan ==3` stay ordinary
  task text.
- Near misses with teaching errors (spell every suggestion with `==`): `==#bugs=3` →
  `==3#bugs`; `==~2#bugs` → `==#bugs~2`; `==3 more` / `==#bugs more` / exact token with
  child lines → "must be the whole item"; `==# bugs` (space) and `==#deep work`
  (multi-word) reuse the `=#` messages; `==x`, `==X1`, `==*`, `==!2` — only when the
  text after the first `=` is itself a well-formed close token — fail with
  `` `==x` is not a close: `==` restarts or swaps the running Pomodoro and never closes it; close it with `=x` ``
  (anything else such as `==xyz` stays prose).
- Incompletes: `==#` needs `pomodoro_name`; `==~` / `==~2,` need `pomodoro_start_task`.
- Chains: `==` tokens are session chain tokens (`==#bugs +2`, `+2 ==#bugs`, `== --1`,
  `=x ==`). This also fixes today's oddity where `=x ==` reports a reorder error whose
  suggestion equals the input.
- Out of scope, unchanged: link/task starts (`^route:id==…`, `@route:id==…` keep their
  existing shape error), `@@` inheritance (never applies), forced
  `--route/--section/--task/--task-ref/--task-section/--clip`, `%`, `s:<N>`, `p:<N>`
  (rejected exactly as on starts).

### Execution (staged daily note; atomic; dry-run computes the same result)

Guards, in order: missing day file → missing `## Pomodoros` section → more than one open
timed entry (existing message) → then:

- **No open timed entry** → the idle fallback: delegate verbatim to the existing `=`
  path (`plan_pomodoro_start_item` / `plan_named_pomodoro_start_item`), including its
  errors (for example "no future Pomodoro").
- **Exactly one open timed entry R** → restart or swap.

**Restart** (`==<X>`, and `==<X>#name` with a non-empty `<X>` whose name resolves to R):

1. Apply `~<K>` to R's lineup (pre-image numbering, `plan_start_drop`).
2. Replace R's whole parenthesized session ledger (the span `=x0` clears, including
   `[t:: …]` and range-local annotations) with the canonical `(**HHMM-HHMM** [t:: Nm])`
   from `compute_pomodoro_start_range`.
3. Move R to the current slot exactly as any start does (normally a no-op).

Property: on a note-free ledger, `==<X>` leaves the day file byte-identical to
`=x0 =<X>`. Unlike `=x0`, a restart never closes a note-bearing session.

**Swap** (`==[<X>]#name`, name resolves to anything other than R):

1. Resolve `name` against the pre-image exactly as `=<X>#name` does (`select_named`:
   open whole slug, else prefix; then completed → "again"; else create; same "did you
   mean" warning). A found open entry must be an untimed placeholder (existing error
   otherwise).
2. With an empty `<X>`, read R's parenthesized ledger bytes; if they cannot be isolated,
   refuse with
   ``cannot read CAPTURE's session ledger at line N; give the new session a timing instead (`==5#bugs`)``.
3. Apply `~<K>` to the target's lineup (a created/"again" target keeps today's
   empty-lineup error).
4. **Demote R**: clear its ledger to `()`, keep the headline suffix and every child byte
   (Task Links, struck links, nested details, stand-alone notes). No note gate (unlike
   `=x0`), no history row, no task-note writes, no Work Log.
5. **Start the target** exactly as `=<X>#name` would with nothing running (create via
   `insert_named_placeholder` when needed, then `start_existing_pomodoro_entry`, which
   moves it to the current slot), except that an empty `<X>` writes R's old ledger bytes
   verbatim instead of the 25-minute default.
6. Move R's block before the earliest other open untimed placeholder (the `=x0` reset
   move), so R is first future.

Properties: on a note-free ledger, `==<X>#name` (non-empty `<X>`) is byte-identical to
`=x0 =<X>#name`, and `==#name` equals `=x0 =#name` except that the target's ledger is
R's old ledger. `==#name` resolving to R refuses:
`` `==#capture` names CAPTURE, which is already running (0920-0945, line 5); restart it with `==` or `==<X>`, or name another Pomodoro to swap in ``.

Invariants for every path: CRLF and a missing final newline are preserved (splice bytes;
do not rebuild via `lines.join`, which `plan_reset` does today); no task lane changes
(no status, Work Log, or task-note writes), so Today is unchanged because both sessions
stay open; `plan.strict` never refuses an override (starts are exempt), while a created
swap target still reports `plan_budget` and the theme-cap warning; later batch items and
chain tokens see the staged result; any failure rolls back the whole batch.

### Worked example (use it verbatim in docs and tests)

`BOB_NOW=2026-10-09 09:32:00`, TAB indentation:

```markdown
## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[bob#^capture-stop]]
- [ ] (**0920-0945** [t:: 25m]) — CAPTURE
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
- [ ] () — SASE
- [ ] () — BUGS
  - [[sase#^fix-flaky]]
```

| Item          | Result                                                                                                                                                   |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `==`          | CAPTURE becomes `(**0935-1000** [t:: 25m])` in place                                                                                                     |
| `==3`         | CAPTURE becomes `(**0935-0950** [t:: 15m])`                                                                                                              |
| `==-2`        | CAPTURE becomes `(**0925-0950** [t:: 25m])`                                                                                                              |
| `==~2`        | CAPTURE restarts at `0935-1000` without `[[bob#^web-capture]]`                                                                                           |
| `==#bugs`     | BUGS takes over `(**0920-0945** [t:: 25m])` and moves under PLAN; CAPTURE becomes `- [ ] () — CAPTURE` right after BUGS, both links intact; SASE follows |
| `==3#bugs`    | Same layout, BUGS is `(**0935-0950** [t:: 15m])`                                                                                                         |
| `==#plan`     | A new PLAN session ("again") takes over `0920-0945`; CAPTURE first future                                                                                |
| `==#capture`  | Refused: CAPTURE is already running (teach `==` / `==<X>`)                                                                                               |
| `==3#capture` | Identical to `==3` (a restart)                                                                                                                           |
| `==#bugs +2`  | Swap, then BUGS extends to `0920-0955`                                                                                                                   |

`==#bugs` writes:

```markdown
## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[bob#^capture-stop]]
- [ ] (**0920-0945** [t:: 25m]) — BUGS
  - [[sase#^fix-flaky]]
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
- [ ] () — SASE
```

### JSON contract (schema version 1; additive only)

**`bob capture-parse`**: the `pomodoro_start` spec (top level and per item) gains
`"override": true`, omitted when false (`#[serde(default, skip_serializing_if = ...)]`,
like `drop`). Mode stays `pomodoro_start`; span kinds stay `pomodoro_start` (covering
both `=` and the suffix), `pomodoro_name`, `pomodoro_start_drop`,
`interactive_placeholder`, so older clients still colour and gate `==` correctly. The
human line spells `==`, and describes the intent (`restart`, or
`swap → BUGS (keeps the running ledger)` / `(fresh 15m)`).

**`bob capture`**: kind stays `pomodoro_start`, `placement: "started"`, `text` is the
raw token, `task_line` is the session now running. The existing `pomodoro_start` fields
describe that session (for a kept ledger: `start`/`end`/`duration_minutes`/`time_range`
from the transferred range, `offset_units: 0`). A new `pomodoro_start.override` object
is present exactly when the token was `==` (plain `=` never carries it):

```json
"override": {
  "action": "swap",
  "ledger": "kept",
  "previous": {"pomodoro_name": "CAPTURE", "pomodoro_line": 5, "start": "0920",
               "end": "0945", "duration_minutes": 25, "time_range": "0920-0945"},
  "demoted": {"pomodoro_name": "CAPTURE", "pomodoro_line": 7,
              "entry_line": "- [ ] () — CAPTURE", "task_links": 2, "has_notes": false}
}
```

- `action`: `"restart"` | `"swap"` | `"start"` (idle fallback: nothing was running, the
  token behaved exactly like its `=` twin; `previous` and `demoted` omitted).
- `ledger`: `"kept"` (swap with empty `<X>`) | `"fresh"` (everything else).
- `previous`: R in the pre-image (`pomodoro_name` omitted when unnamed; `start`/`end` in
  the same format as `pomodoro_start.start`).
- `demoted`: swaps only; R in the post-image. `task_links` counts R's direct-child Task
  Links (`list_queued_links`); `has_notes` is `has_standalone_note`.
- Batch `pomodoro_blocks`: the running session gets `started` (before = its pre-image
  entry, or `created`); a swapped-out R gets `reset`. A restart reports one `started`
  block.

**`bob capture-complete`** (phase override-complete): for a `==` name field the context
stays `pomodoro_start_name` and candidates are unchanged; a new additive top-level
`"override": {"keeps_ledger": true, "running": {"pomodoro_name": "CAPTURE", "line": 5, "time_range": "0920-0945"}}`
appears (`running` omitted unless exactly one session runs; `keeps_ledger` is true when
`<X>` is empty).

### Human output (`bob capture`)

```text
✓ restarted  2026/20261009.md
  CAPTURE 0920-0945 → 0935-0950 (15m) at line 5
  - [ ] (**0935-0950** [t:: 15m]) — CAPTURE
  1 [*] Add support for `=x` syntax! bob.md ^capture-stop
  2 [*] Add capture support for web URLs! bob.md ^web-capture

✓ swapped  2026/20261009.md
  BUGS takes over 0920-0945 (25m) at line 5
  CAPTURE → first future at line 7 · keeps 2 Task Links
  - [ ] (**0920-0945** [t:: 25m]) — BUGS
  1 [*] Fix flaky gkeep test sase.md ^fix-flaky
```

A fresh swap prints `BUGS 0935-0950 (15m) at line 5` (with ` (created)` when created)
and `CAPTURE 0920-0945 → first future …`; append `and its notes` when `has_notes`.
Dry-run verbs: `would restart`, `would swap`. The idle fallback prints the normal start
output plus a dim `nothing was running, so == started it like =`. Reuse the start
renderer's numbered lineup, dropped rows, `Dropped …` summary, and `nothing queued`.

### Teaching errors that should mention `==` (discoverability)

Keep every existing substring (Bob Mac Capture tests pin `=x =#deep-work`) and append:

- `=`/`=<X>` while R runs: `, or capture ==<X> to restart CAPTURE now`.
- `=<X>#name` while another session runs:
  `, or ==<X>#name to swap it in and return CAPTURE to first future`.
- `=<X>#name` naming the running session ("already running"): add
  `== to restart it now`.
- `=x#bugs`: `…or ==#bugs to swap it in without closing CAPTURE`.

Exact wording may be polished; each must name the concrete `==` spelling.

### Design choices made (and rejected alternatives)

- **`se<X>` between `==` and `#`, not clock ranges.** `==0920-0945#bugs` collides with
  the `se<X>` grammar (`0920-0945` already lexes as duration 920 / offset 945 units),
  and `+N`/`++N` compose on the same line for fine-tuning (`==#bugs --1`).
- **One kind, additive object.** Keeping `kind: "pomodoro_start"` keeps the Mac start
  card, footer, picker gating, and `=#` completion working on an older app; a new kind
  would break all of them.
- **Swap demotes instead of closing.** `=x =#bugs` already closes-then-starts (history);
  `==` is for "that session didn't really happen as recorded", so nothing becomes
  history and the old block returns to first future intact.
- **No note gate.** `=x0` closes note-bearing sessions because `=x0` is ambiguous with a
  close; `==` is unambiguous and loses no bytes, so it always demotes and the output
  says "and its notes".

## Phase override-grammar

Work in `src/native/capture_language/` and `src/native/capture_parse.rs` (line numbers
are approximate pointers):

- `item.rs` `session_equals_token` (~888): after the first `=`, consume a second `=` and
  set a new `force`/`override` flag on `EqualsToken::Start`; `len` counts both. Handle
  `==` + close shapes as the claimed near miss above (decide with the existing close
  lexers; only well-formed close tails are claimed).
- `model.rs` `PomodoroStartSpec` (~355): add `override: bool` with
  `#[serde(default, skip_serializing_if = "is_false")]`; set it in
  `parse_pomodoro_equals_item` (named branch ~1604, unnamed ~1710) the way `spec.drop`
  is set. Claim policy: `force` alone does not claim (bare `== foo` stays prose).
- Every place that assumes the token prefix is `1 + suffix.len()` must use the real
  sigil length: `editor_pomodoro.rs` (~274, ~784), `completion.rs`
  `pomodoro_start_name_field` (~419), `start_selection.rs` `start_token_suffix` (~132),
  plus the start-name completion code. Every suggestion that hard-codes `={suffix}` must
  spell the typed sigil: `item.rs` (~1138, ~1179), `markers.rs` (~446, ~474, ~485),
  `start_selection.rs` (~79), `capture_parse.rs` `format_pomodoro_start` (~898).
- Chains: `draft.rs` `session_chain_tokens` / `is_session_chain_token` (item.rs ~958)
  must accept `==` tokens exactly like `=` tokens; inline Work Log trailing operators
  (`=x wired the lexer ==`) must treat `==` like `=`.
- Editor parse (`editor_pomodoro.rs`): `==#` → incomplete `pomodoro_name` with the
  partial spec carrying `override: true` (Bob Mac Capture opens its start picker only
  when the incomplete response has a start spec); `==~` → `pomodoro_start_task`; valid
  tokens → mode `pomodoro_start`. Check `rewrite.rs` (~236, ~293) still round-trips.
- `capture_parse.rs` help prose and Modes/Needs lists; `bob capture-parse` docs section
  in `docs/capture.md` (the `override` spec flag).
- Execution is untouched in this phase: the flag reaches `plan_pomodoro_start_item`,
  which still runs the `=` path, so a running session safely refuses with today's
  message and an idle ledger starts. Do not add interim error text.
- Tests: update every assertion that `==` is prose (`capture_language/tests/chain.rs`
  ~69/~112, `tests/grammar.rs` ~1012, `tests/editor_modes.rs` ~1187/~1278,
  `tests/cli/capture/pomodoro_whole_item.rs` ~573, `tests/cli/capture/parse_pomodoro.rs`
  ~713); add table cases for every grammar row, prose protection (`==important== thing`,
  `== foo`, `===`, `Plan ==3`, `==xyz`), each near miss and incomplete, chains (`=x ==`,
  `==#bugs +2`), capture-parse JSON (`override: true` present only for `==`, spans
  covering `==<X>`), and `==#` name completion replacement ranges.
- Verify: `just check`.

## Phase override-restart

Work in `src/native/capture/` (`pomodoro_start.rs`, `batch.rs`, `start_output.rs`,
`output.rs`, `plan.rs`, `pomodoro_blocks.rs`) and the `capture_pomodoro_close/reset.rs`
helpers:

- In `plan_pomodoro_start_item`, branch on `spec.override` after the day-file, section,
  and multiple-timed guards: no running entry → idle fallback (see the decision
  callout); exactly one → restart when there is no name (the swap phase adds the named
  branch; until then a named override with a running session keeps today's refusal).
- Restart: locate R and its ledger span (reuse `select_running_session` /
  `parse_adjustment_range` or reset's `clear_timing_payload` span logic), apply drops,
  splice the canonical range, then reuse the start's current-slot move. Report the
  lineup through `list_queued_links` / `resolve_queued_links` like any whole-item start.
- Add the full `PomodoroStartOverrideJson` (`action`, `ledger`, `previous`, optional
  `demoted`) to `PomodoroStartSummary` with `skip_serializing_if = "Option::is_none"`;
  populate `action: "restart"` and `action: "start"` here.
- Human output: `restarted` / `would restart` with the before → after range line, and
  the idle note; keep plain `=` output byte-identical.
- `pomodoro_blocks`: one `started` ref with correct before/after indices (debug builds
  panic on wrong refs).
- Teaching error: append the `==<X>` restart hint to the unnamed `=` still-running
  message.
- Docs (`docs/capture.md`): add a "Restarting or swapping the running Pomodoro" section
  after "Starting a named Pomodoro" with the restart half (recognition, timing table
  pointer, guards, worked rows, JSON `override` reference including all fields, human
  output); add `==`/`==<X>` rows to the lifecycle table and the mnemonic.
- Tests (new `tests/cli/capture/pomodoro_override.rs`, registered in
  `tests/cli/capture/mod.rs`): the worked-example restart rows; idle fallback
  byte-identical to `=` (day file and JSON apart from `text`/`override`); `==3` vs
  `=x0 =3` byte equality on a note-free fixture; a note-bearing R restarts with notes
  intact; CRLF and missing final newline; two running entries refuse with the vault
  byte-identical; dry-run JSON equals real-run JSON except `dry_run`; batch rollback.
- Verify: `just check`.

## Phase override-swap

- Add the named override branch: name resolution, the R-is-target cases (empty `<X>` →
  refusal; non-empty → restart), kept vs fresh ledger, demotion with byte splices, the
  first-future move, and the start of the target, in the order specified above. Prefer a
  small dedicated module (for example `src/native/capture/pomodoro_override.rs`) that
  composes `select_named`, `insert_named_placeholder`, `start_existing_pomodoro_entry`,
  `plan_start_drop`, and a byte-safe demote helper factored from `reset.rs` (do not
  change `=x0` behavior).
- Populate `override.demoted` (`task_links`, `has_notes`), `ledger: "kept"|"fresh"`, and
  `pomodoro_blocks` refs: target `started` (or `created` before), R `reset`. Cover an
  unnamed R demoted next to other unnamed `()` placeholders (identical headline text) in
  a debug-build test.
- Human output: `swapped` / `would swap`, the takeover/fresh line, and the
  `→ first future · keeps N Task Links[ and its notes]` line.
- Teaching errors: the named still-running, already-running, and `=x#bugs` additions
  above.
- Docs: finish the new section (swap half, worked example, ledger-unreadable and
  R-is-target errors, `=x0` equivalence statements); "Grammar at a glance" rows for
  `==`/`==<X>`, `==[<X>]#pomodoro`, `==[<X>][#name]~<K>`; the `#`/`=` meaning table rows
  (including `==important==` stays prose and the `==x` error); remove `==` from every
  "stays prose" sentence (~154, ~1405, ~1754, ~3503); amend "Bob never overwrites or
  double-starts an entry" (~795) to name `==` as the explicit override; add `reset` to
  the `pomodoro_blocks` roles list (~3209); the `bob capture --help` paragraph in
  `src/native/capture/cli.rs` (~226); README grammar rows (~345-347).
- Tests: every worked-example swap row; `==3#bugs` vs `=x0 =3#bugs` and `==#bugs` vs
  `=x0 =#bugs` (modulo the ledger) byte equality; again/created targets with
  `created_pomodoro: true`; did-you-mean warning; `==#bugs~2` and created-target drop
  error; a placeholder above R (demoted R still first future); chains `==#bugs +2` and
  `+2 ==#bugs` (inherits the extended ledger); kept ledger with range-local annotations
  moves verbatim; plan budget warning with `plan.strict: true` not refusing; rollback.
- Verify: `just check`, then a manual temp-vault smoke of every worked-example row with
  `BOB_NOW=2026-10-09 09:32:00` in both human and JSON output.

## Phase override-complete

- In `src/native/capture_complete/` (`engine.rs`, `pomodoros.rs`, `model.rs`): when the
  name field belongs to a `==` token, add the top-level `override` object to
  `CaptureCompleteResult` (`keeps_ledger` from an empty suffix; `running` from
  `find_running_pomodoro` on the day file, omitted unless exactly one runs). Plain `=#`
  output stays byte-identical. Candidate rows do not change.
- Tests in `src/native/capture_complete/tests/pomodoros.rs` and
  `tests/cli/capture/complete_editor.rs`: `==#`, `==3#bu`, idle ledger, missing day
  file, plain `=#` unchanged.
- Docs: the capture-complete `pomodoro_start_name` contract in `docs/capture.md`.
- Verify: `just check`.

## Phase mac-override-card

Open the app repo with `sase repo open bob-mac-capture` (if the linked checkout is
absent on the host, `sase repo open gh:bobs-org/bob-mac-capture`). Agent hosts have no
Swift toolchain; macOS CI (`.github/workflows/ci.yml`) is the build gate, so keep Swift
simple (early returns, explicit types in long expressions; see the `primaryActionTitle`
type-checker fix in commit `fec4293`) and re-read every changed file. The app stays a
thin client: every string is built from Bob's JSON.

- `Sources/CaptureCore/CaptureModels.swift`: `PomodoroStartSpec.isOverride` (key
  `override`, default false); `PomodoroStartSummary.overrideOutcome` (key `override`) of
  a new tolerant `PomodoroStartOverride` (action, ledger, previous, demoted). Unknown
  actions decode as nil.
- `Sources/CaptureCore/CapturePomodoroStartPresentation.swift`: add a variant (`start`,
  `restart`, `swap`, `idleStart`) driving title (`Restart CAPTURE` /
  `Restarted CAPTURE`, `Swap in BUGS` / `Swapped in BUGS`), status
  (`Would restart CAPTURE 0920-0945 → 0935-0950 (15m) at line 5`,
  `Would swap in BUGS 0920-0945 (takes over CAPTURE) at line 5`), badge (`Takes over`
  for a kept ledger; existing `New`/`Created`), the idle caption
  (`Nothing was running — starts like =`), `primaryActionTitle` (`Restart` / `Swap`),
  notification title/body (`Restarted CAPTURE` · `0935–0950 (15m) · 2 queued`;
  `Swapped in BUGS` · `0920–0945 · CAPTURE back to first future`), batch suffix, and
  VoiceOver text. `isSessionStart` is unchanged.
- `Sources/BobMacCapture/CapturePanelView.swift` `startPreviewItem`: icon
  `arrow.clockwise.circle.fill` (restart) / `arrow.left.arrow.right.circle.fill` (swap)
  in the existing start pink; a dim `was 0920–0945` caption for restarts; for swaps a
  demoted row (`arrow.uturn.down` ·
  `CAPTURE → first future · keeps 2 Task Links[ and its notes]`); task rows and drop
  summary unchanged.
- `Sources/BobMacCapture/CapturePanelModel.swift`: `primaryActionTitle` early returns,
  the three status chains; `Sources/BobMacCapture/NotificationService.swift`: single
  capture and batch suffix wording.
- Fixtures from a bob built at the swap phase's commit (recipe in
  `Tests/CaptureCoreTests/CaptureTaskCompletePresentationTests.swift`, worked-example
  vault, `BOB_NOW=2026-10-09 09:32:00`): restart dry/submitted, swap kept dry/submitted,
  swap fresh created, idle start, parse `==#bugs`, parse `==#` incomplete; fake-bob
  routes; presentation tests (including "older Bob without `override` decodes as a plain
  start"), panel-model footer tests, notification tests.
- README: Runtime Contract clause, start-card paragraph, footer verbs, editor grammar
  paragraph (now teaching `==`), Notifications section.
- Verify: macOS CI if reachable (`gh run list` after the commit lands); otherwise record
  in the bead notes that Swift verification is pending CI.

## Phase mac-override-picker

- Decode the capture-complete `override` object (`keepsLedger`, optional `running`) in
  `Sources/CaptureCore/` and thread it to the `.pomodoroStartName` rows in
  `CompletionRowContent.swift` and the picker status in
  `CapturePickerPresentation.swift` / `CapturePanelModel.swift`.
- Picker status: `Pick a Pomodoro to take over CAPTURE's 0920–0945` (keeps ledger),
  `Pick a Pomodoro to start now · CAPTURE returns to first future` (fresh), existing
  text when nothing runs or the object is absent.
- Rows under `==#`: open rows' secondary text `Takes over 0920–0945` (keeps ledger); the
  running row `Running 0920–0945` with hint
  `Already running — == restarts it; pick another Pomodoro to swap in` (keeps ledger) or
  `Restarts it now` (fresh). Under plain `=#`, the running-row hint becomes
  `Already running. Restart it with ==, or close it first with =x.` (update
  `CompletionRowContentTests`).
- Fixtures (`==#`, `==3#`, idle) from real bob, fake-bob routes, row and status tests,
  README start-name completion section.
- Verify as in the previous phase.
