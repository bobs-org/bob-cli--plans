---
tier: epic
title: "`=` overrides the running Pomodoro; `==` retires"
goal: "A whole-item `=` start restarts or swaps the running Pomodoro by itself, `==`
  becomes a retired teaching error, and Bob Mac Capture shows a clear, unmissable
  override warning and never commits an override whose warning was not on screen.

  "
decisions:
  swap_clock:
    ask:
      When `=#bugs` swaps BUGS in while CAPTURE runs, should BUGS keep CAPTURE's running
      clock?
    choices:
      keep:
        BUGS takes over CAPTURE's clock, as `==#bugs` does today; `=5#bugs` starts fresh
      fresh:
        BUGS always starts a fresh 25 minutes; the clock takeover goes away entirely
    default: keep
    why: You asked for `==#name` to keep the running clock; `=` inherits that.
    answer: keep
  decision_record:
    ask:
      Add a decisions-web record for `=` overriding the running Pomodoro behind a
      warning?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: eq-overrides
    title: "bob: `=` restarts or swaps the running session"
    depends_on: []
    size: medium
    description: "eq-overrides: route every whole-item `=` start with one running
      session through the existing restart/swap planners, add
      `override.previous.overdue`, `override.close_first`, the yellow human warning
      line, the hidden `--refuse-override` guard, and the state-based capture-complete
      override context; `==` keeps working.

      "
  - id: retire-double
    title: "bob: retire `==`"
    depends_on:
      - eq-overrides
    size: medium
    description: 'retire-double: make every claimed `==` token a teaching error that
      names its `=` rewrite in both parsers and in chains, drop the parse-spec
      `override` flag and the idle `action: "start"` fallback, respell every `==`
      message, and migrate the tests.

      '
  - id: docs
    title: "docs: teach the `=` override"
    depends_on:
      - retire-double
    size: medium
    description: "docs: rewrite the Pomodoro start, named-start, and override sections
      of docs/capture.md plus README.md around `=`, document the new JSON fields, the
      warning line, `--refuse-override`, and the retired `==` row, quoting bob's real
      output.

      "
  - id: mac-warning
    title: "Mac: override banner and orange Return"
    depends_on:
      - retire-double
    size: medium
    description: "mac-warning: add the override banner to the start card, tint the
      Return button orange for overrides, respell `==` copy to `=`, drop the idle and
      parse-flag leftovers, regenerate fixtures from the new bob, and update tests and
      README.

      "
  - id: mac-guard
    title: "Mac: commit an override only after its warning was shown"
    depends_on:
      - mac-warning
    size: small
    description:
      "mac-guard: pass `--refuse-override` on every submit whose on-screen preview did
      not show an override, and turn the refusal into a calm re-preview so the second
      Return commits with the banner visible."
proposed_by: bbugyi200.apollo.91
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 16:26:19
status: wip
---

- **PROMPT:**
  [prompts/202610/eq_overrides_running_pomodoro.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/eq_overrides_running_pomodoro.md)

# Plan: `=` overrides the running Pomodoro; `==` retires

## Why

`==` is not needed. Look at what it does that `=` does not:

- **Nothing is running.** Every `==` token already behaves exactly like its `=` twin.
- **One session is running.** `=` refuses ("… is still running …; close it with `x`
  first"), and `==` restarts or swaps it.

So the only thing `=` adds today is the refusal, and the refusal is just a safety net.
This epic replaces that safety net with two better ones:

- a warning you cannot miss, shown before you commit;
- a commit-time guard, so an override never happens without that warning on screen.

After that, `=` covers starting, restarting, and swapping on its own. `==` retires with
a teaching error, the same way `=x` just did in epic bob-cli-9b.

**Mental model:** "`=` starts the session that should be running now. If another session
is running, `=` overrides it (restart or swap) and warns you first. To close the running
session instead, put `x` in front: `x =`."

## Where things stand

- **Execution:** `src/native/capture/pomodoro_start.rs`.
  - `plan_pomodoro_start_item` refuses `=`/`=<X>` while a session runs unless
    `spec.r#override` is set. `plan_named_pomodoro_start_item` does the same for
    `=<X>#name`.
  - `plan_override_restart_item` and `plan_override_swap_item` implement `==`.
  - The idle fallback adds `override.action: "start"`.
- **Lexing:** `session_equals_token` (`src/native/capture_language/item.rs`) lexes `==`
  as `EqualsToken::Start { r#override: true }`.
  - The sigil is also spelled in `editor_pomodoro.rs`, in `completion.rs`
    (`named_start_name_at`, `pomodoro_start_override_at`), in `start_selection.rs`, and
    in `markers.rs`.
- **JSON:**
  - `capture-parse` emits the spec flag `pomodoro_start.override: true`.
  - `bob capture` emits the `pomodoro_start.override` object (`action`, `ledger`,
    `previous`, `demoted`).
  - `capture-complete` emits a top-level `override` (`keeps_ledger`, optional
    `running`), but only for `==` name fields.
- **Bob Mac Capture already presents overrides from bob's JSON.**
  - The start card has restart and swap variants
    (`CapturePomodoroStartPresentation.swift`, and `startPreviewItem` in
    `CapturePanelView.swift`).
  - The swap picker copy comes from the completion `override`.
  - Live preview runs `bob capture --dry-run --no-clip --format json`.
  - Submit does not wait for the preview, so `=` then a fast Return can commit before
    any preview is drawn.
- **In flight: epic bob-cli-9b (bare `x` close).**
  - Its `src/` phases have landed. Its docs (9b.3) and Mac (9b.4) phases rewrite the
    same `docs/capture.md`, `README.md`, Mac copy, and fixtures this epic touches.
  - Spell every close `x` (`x =`, `x =#bugs`).

## Design (applies to every phase)

### Behavior

"Running" means exactly one open timed entry in the staged daily note, which is the test
`==` uses today. Chains and batches see staged state, so in `x =` the `x` closes the
session first, and the `=` is a plain start that never warns.

| Token                                  | Nothing running                            | One session R running                                                       |
| -------------------------------------- | ------------------------------------------ | --------------------------------------------------------------------------- |
| `=` / `=<X>`                           | Start the next future Pomodoro (unchanged) | **Restart** R now with `se<X>` timing (`=` = 25 minutes)                    |
| `=#name` (another session)             | Start it, 25 minutes (unchanged)           | **Swap** it in; R returns intact to first future (timing per `swap_clock`)  |
| `=<X>#name` (another session, counted) | Start it with `se<X>` (unchanged)          | **Swap** it in with fresh `se<X>` timing                                    |
| `=<X>#name` naming R, counted          | —                                          | **Restart** R (same as `=<X>`)                                              |
| `=#name` naming R                      | —                                          | Per `swap_clock`                                                            |
| `…~<K>`                                | Drops from the started session (unchanged) | Drops from the session that ends up running, numbered like the start lineup |
| More than one timed session open       | Refused (unchanged)                        | Refused (unchanged)                                                         |

Rules:

- **Byte-identical results.** Restart and swap produce exactly the day file that `==`
  produces today. Reuse `plan_override_restart_item` and `plan_override_swap_item`, and
  change only their messages.
- **Link and task starts are unchanged.** `^route:id=`, `@route:id=<X>`, and
  `Text @route:id=3` still refuse with "finish the current Pomodoro first". A link start
  that overrode would re-time the running session as a side effect of linking a task.
  The whole-item `=` stays the one explicit override gesture.
- **Strict plans.** `plan.strict` never refuses an override, and a created swap target
  still reports `plan_budget`. Both are unchanged.

> [!decision] swap_clock = keep An empty timing keeps the running clock only when the
> override changes _which_ session runs. Re-running the same session always re-times it,
> because keeping its clock would change nothing.
>
> - `=#bugs` hands R's ledger to BUGS byte-for-byte (`ledger: "kept"`), and `=5#bugs` is
>   the "fresh 25 minutes" spelling.
> - `=#capture` while CAPTURE runs refuses with
>   `` `=#capture` names CAPTURE, which is already running (0920-0945, line 5): restart it with `=` or `=<X>`, close it with `x`, or name another Pomodoro to swap in``.
> - `=3#capture` restarts CAPTURE (same as `=3`).
> - The unreadable-ledger refusal stays, and teaches `=5#bugs`.

> [!decision] swap_clock = fresh An empty timing always means a fresh 25 minutes.
>
> - `=#bugs` while R runs swaps BUGS in at a fresh 25 minutes. `ledger` is always
>   `"fresh"` (keep the key for older clients).
> - `=#capture` while CAPTURE runs restarts CAPTURE exactly like `=`; it never refuses.
> - Delete the kept-ledger path: `read_session_ledger_bytes`, the unreadable-ledger
>   refusal, and the kept branches in the human output.
> - `capture-complete` reports `keeps_ledger: false` everywhere.
> - On the Mac, delete the `Takes over` badge, the `Takes over 0920–0945` row text, and
>   the "take over" picker status.

### Retiring `==`

- **One teaching error.** Every `==` token that claims its item today fails with a
  single teaching error that names the `=` rewrite (the token with one `=` removed).
  This covers `==`, `==<X>`, `==#name`, `==<X>#name`, `…~<K>`, `…@<X>`, the near misses
  `==#bugs=3` and `==3 more`, and the incompletes `==#` and `==~`:
  ``  `==3#bugs` is retired: `=` restarts or swaps the running Pomodoro by itself; write `=3#bugs` ``
  The retired error wins over every other diagnostic on that token.
- **The `==x` family** (`==x`, `==X1`, `==*`, `==!2`) stays claimed. Its message drops
  the clause "`==` restarts or swaps…" and reads
  ``  `==x` is not a close: close the running Pomodoro with `x` ``, keeping today's
  rewrite normalization.
- **Prose is unchanged.** `== foo`, `==important== thing`, `==foo`, `===`, and
  `Plan ==3` stay ordinary task text, which protects Obsidian highlights.
- **Editor:** mode `pomodoro_start` with an `invalid_pomodoro_start` diagnostic over the
  whole token. Report spans the way the retired `=x` forms do. Offer no
  `pomodoro_start_name` completion and report no `pomodoro_name` or
  `pomodoro_start_task` needs.
- **`bob capture`:** an item error, of the same error class as the retired `=x` forms.
  In a chain such as `x ==`, the `==` token fails on its own range.

### JSON contract (schema version 1, additive)

**`bob capture`:**

- `pomodoro_start.override` is present exactly when a start restarted or swapped a
  running session. That now comes from `=`; plain starts never carry it.
  - `action` is `"restart"` or `"swap"`.
  - `"start"` is no longer emitted, but clients must keep tolerating it.
- New `override.previous.overdue: true` (omitted when false). It is true when `now` is
  at or after R's end, where R's start is read on today's date and R's end is that start
  plus R's duration.
  - `2350-0015` at 23:55 is not overdue.
  - A start later than `now` (a session shifted forward) is not overdue.
  - A range that does not parse is not overdue.
- New `override.close_first`: the draft spelling that closes R first and then runs this
  start. It is always `"x "` plus the typed token: `"x ="`, `"x =#bugs"`,
  `"x =3#bugs~2"`. bob owns this grammar; clients only display it.
- New hidden flag `--refuse-override`, for subprocess use.
  - It mirrors `--require-pull-in-choice`: it is hidden and has no short alias (the
    `cli_rules` exception for internal subprocess arguments).
  - When any item would restart or swap, bob refuses the whole batch before writing,
    identically in dry-run.
  - The error is I/O-class (exit 1) with `code: "pomodoro_override_refused"` and a
    message such as
    ``refusing capture: `=` would restart the running CAPTURE (0920-0945) and --refuse-override is set; capture `x =` to close it first``
    (a swap says "would swap out").

**`bob capture-parse`:** drop the `pomodoro_start.override` spec flag. It was omitted
when false, so nothing else changes.

**`bob capture-complete`:** the top-level `override` on a `pomodoro_start_name` field
becomes state-based.

- It is present when exactly one session runs in the daily note and no earlier item, or
  earlier same-line chain token, in the draft is a whole-item session operator (`x…`,
  `=…`, `+N`/`-N`, `++N`/`--N`).
- Otherwise it is omitted. The dry-run preview stays the authority for what happens.
- `keeps_ledger` keeps today's rule: an empty leading suffix and no `@` part.
- `running` is always set when the object is present.

### Human output (`bob capture`)

A yellow line (`styler.yellow`) goes directly under the header of every restart or swap.
Everything else stays byte-identical. The idle line
`nothing was running, so == started it like =` is deleted.

```text
✓ would restart  2026/20261009.md
  ⚠ overrides the running CAPTURE 0920-0945 · capture `x =` to close it first
  CAPTURE 0920-0945 → 0935-1000 (25m) at line 5
  …
✓ swapped  2026/20261009.md
  ⚠ overrode the running CAPTURE 0920-0945
  BUGS takes over 0920-0945 (25m) at line 5
  CAPTURE → first future at line 7 · keeps 2 Task Links
```

Variants:

- **Overdue:** `⚠ overrides CAPTURE 0920-0945, still open past its end · capture `x
  =` to close it first` on a dry run, and
  `⚠ overrode CAPTURE 0920-0945, still open past its end` on a real run.
- **Unnamed R:** "the running session 0920-0945", or "the session 0920-0945, still open
  past its end".
- **Real runs** never print the close-first hint.

### Bob Mac Capture: one warning, impossible to miss

The warning lives in one place, the start card, and the Return button echoes it. bob
supplies every fact and every grammar string. The app never computes clocks or composes
tokens.

```text
┌──────────────────────────────────────────────────────────────────┐
│ ▲  Overrides the running CAPTURE                      0920–0945  │ ← orange banner
│    Its clock restarts without closing it                         │
│    Type ( x = ) to close it first                                │
│ ↻  Restart CAPTURE                              0935-1000 (25m)  │
│    2026/20261009.md · line 5                                     │
│    was 0920–0945                                                 │
│    ① ◉ Add support for x syntax                bob ^capture-stop │
└──────────────────────────────────────────────────────────────────┘
                                   Discard   Preview   [ Restart ]  ← orange
```

1. **Banner.** It is the first row of every restart or swap card, including each item's
   card in a batch.
   - **Glyph:** `exclamationmark.triangle.fill`, in orange.
   - **Headline:** semibold, with the session range in monospaced text on the right.
   - **Detail:** one secondary caption line.
   - **Close-first line:** shows `close_first` as a monospaced chip in the
     Pomodoro-session tint.
   - **Container:** a continuous rounded rectangle filled with orange at about 12%
     opacity, with a hairline orange stroke.
   - **Copy:**
     - Headline: `Overrides the running CAPTURE`. Overdue:
       `Overrides CAPTURE, still open past its end`. Unnamed R: "the running session"
       and "the session".
     - Detail for a restart: `Its clock restarts without closing it`.
     - Detail for a kept-ledger swap: `BUGS takes over its clock without closing it`.
     - Detail for a fresh swap: `It steps aside without closing it`.
     - Close-first line: `Type` + chip + `to close it first`. Omit the line when an
       older bob sends no `close_first`.
   - **VoiceOver:** the warning is read first: "Warning: overrides the running CAPTURE,
     0920 to 0945. Its clock restarts without closing it. Type x = to close it first."
2. **Orange Return.** The footer's primary button is tinted orange whenever the
   on-screen preview contains an override. Single items keep today's **Restart** /
   **Swap** titles; batches keep **Capture**.
3. **Guard: no unseen overrides.** Every real submit passes `--refuse-override` unless
   the preview on screen belongs to this exact draft, analysis generation, and bob
   client, and shows the override.
   - A refusal is not an error. The panel stays open, re-previews (so the banner
     appears), and says
     `A Pomodoro is running — review the override, then press Return again`.
   - A deliberate `=` that was previewed commits on the first Return. A fast `=` +
     Return that beat the preview costs one extra Return, with the warning in view.
4. **Copy.** Every `==` hint becomes `=`.
   - `Already running — = restarts it; pick another Pomodoro to swap in`
   - `Restarts it now`
   - `Already running. Restart it with =, or close it first with x.`
   - The swap picker status now appears for `=#` while a session runs.
5. **Notification.** A committed restart appends ` · was 0920–0945` to its body, so an
   unintended override is visible after the fact. Swaps already name the demoted
   session.

### Rejected alternatives and out of scope

- **A confirmation dialog, or a second Return every time.** That adds friction to the
  restart the user meant. The guard asks for a second Return only when the warning was
  never on screen.
- **Guessing intent,** for example treating `=` on an overdue session as `x =`. That
  makes `=` unpredictable. The overdue banner teaches `x =` instead.
- **Keeping `==` as a hidden alias.** That keeps two spellings of one gesture. The `=x`
  retirement set the precedent for teaching errors.
- **Out of scope:**
  - link-start overrides;
  - a shortcut that rewrites the draft to `close_first`;
  - underlining the `=` token in the editor.
- **bob-plugins:** no plugin source mentions the `==` capture grammar, so nothing
  changes there.
- **Rollout:** install bob before, or together with, the new app (`just install-all`).
  An older app on the new bob shows today's restart and swap cards without the banner or
  the guard.

## Phase: eq-overrides

Work in bob-cli `src/` only, so this phase cannot collide with bob-cli-9b's docs and Mac
phases. `==` keeps working throughout this phase, and the whole suite stays green.

1. **`pomodoro_start.rs`.**
   - When exactly one timed session is running, `plan_pomodoro_start_item` always calls
     `plan_override_restart_item`, and `plan_named_pomodoro_start_item` always calls
     `plan_override_swap_item`.
   - Delete the `spec.r#override` gates and the three still-running and already-running
     refusals they guarded, including their `==` hints.
   - In `plan_override_swap_item`, apply the `swap_clock` rules. Spell refusals and
     hints with the typed token (`parsed.body`), never a hard-coded sigil, so
     `==#capture` still echoes `==` until retire-double.
   - Keep the `==` idle `action: "start"` fallback for now.
2. **`overdue`.** Add `overdue: bool` to `PomodoroStartPreviousJson`, serialized only
   when true.
   - Compute it in one helper from R's range, `now`, and `today`. Reuse
     `previous_range_duration_minutes`.
   - Thread `now` into both override planners.
   - Unit-test these cases: mid-session; exactly at the end (overdue); past the end;
     `2350-0015` at 23:55; a start later than `now`; an unparsable range.
3. **`close_first`.** Add `close_first` to `PomodoroStartOverrideJson` as
   `format!("x {}", parsed.body)` for restarts and swaps. Make it optional and omit it
   on the idle `start` action.
4. **Human output (`start_output.rs`).** Add the yellow warning line exactly as in
   Design (dry and real runs, overdue, unnamed). Under the plain styler, the `⚠` stays
   and the color goes.
5. **`--refuse-override`.**
   - In `src/native/capture/cli.rs`, add a hidden `refuse_override_arg()` in
     alphabetical position, between `pull_in_arg()` and `require_pull_in_choice_arg()`.
   - Add `CaptureRequest.refuse_override` and wire it like `require_pull_in_choice`.
     Every other `CaptureRequest` constructor (for example the one in `task_blocks.rs`)
     sets it to `false`.
   - Enforce the flag inside the restart and swap planners, the one choke point every
     override passes. Add a `CaptureError::override_refused(message)` constructor (Io
     kind, code `pomodoro_override_refused`) next to `pull_in_choice_required` in
     `output.rs`.
6. **`capture-complete`** (`completion.rs` `pomodoro_start_override_at`, and
   `capture_complete/engine.rs` and `model.rs`).
   - Return `keeps_ledger` for every named-start name field, `=` or `==`.
   - Return `None` when an earlier item, or an earlier same-line chain token, is a
     whole-item session operator. Reuse `split_capture_draft` and
     `is_session_chain_token`.
   - Build `OverrideCompletion` only when `running_pomodoro_session` returns `Some`, and
     fix the struct's doc comment.
7. **Tests.**
   - Add CLI cases on the worked example in `tests/cli/capture/pomodoro_override.rs`
     (`BOB_NOW=2026-10-09 09:32:00`): `=`, `=3`, `=-2`, `=~2`, `=#bugs`, `=3#bugs`,
     `=#plan`, `=#capture`, `=3#capture`, and `=#bugs +2`.
   - Assert that each one leaves the same day file and JSON as its `==` twin, apart from
     `text` and `close_first`. Assert that an idle `=` carries no `override`.
   - Cover `overdue`, `close_first`, and the human warning lines (dry, real, overdue,
     unnamed, plain styler).
   - Cover `--refuse-override`:
     - it refuses restarts and swaps with the code, leaving the vault byte-identical;
     - it allows idle starts and `x =`;
     - a three-item batch whose last item overrides rolls back entirely;
     - dry-run with the flag refuses identically.
   - Cover `capture-complete`: `=#` with one session running, idle, `x =#`, and `=3#`.
   - Update every test that asserted the deleted refusals. Find them with
     `rg -n 'still running|already running' src tests`; they include
     `pomodoro_start.rs`, `pomodoro_start_named.rs`, `pomodoro_whole_item.rs`, and the
     chain tests.
8. **Verify.**
   - Run `just check`.
   - Smoke the worked example in a scratch vault (`BOB_DIR`, `BOB_NOW`):
     - `bob capture --dry-run '='`
     - `bob capture --dry-run '=#bugs'`
     - both again with `--format json`
     - one run with `--refuse-override`

## Phase: retire-double

1. **Lexer.** `session_equals_token` keeps lexing `==`, so the claim policy and the
   ranges stay put. Rename the `r#override` field to say what it now means (for example
   `doubled`), and update its docs.
2. **Execution** (`parse_pomodoro_equals_item` in `item.rs`).
   - Every claimed doubled token fails through one shared `markers.rs` helper that
     computes the one-`=` rewrite (`==` → `=`, `==3#bugs~2` → `=3#bugs~2`, `==#bugs@3` →
     `=#bugs@3`).
   - Respell the `==x`-family message as in Design.
3. **Editor** (`editor_pomodoro.rs`).
   - Claimed doubled tokens get the diagnostic described in Design.
   - Delete the `sigil_len`/`override_token` branches, plus the incomplete
     `pomodoro_name` and `pomodoro_start_task` paths for `==`.
4. **Completion.** `named_start_name_at` skips doubled tokens, so there is no start-name
   context. Simplify `pomodoro_start_override_at` to match.
5. **Model and execution cleanup.**
   - Delete `PomodoroStartSpec.r#override` and its JSON key.
   - Delete the idle `idle_override` fallback in both start planners, and the idle line
     in `start_output.rs`.
   - Make `close_first` non-optional.
6. **Respell every remaining `==` mention:**
   - `markers.rs`: `pomodoro_close_hash_error` now says "`=#bugs` swaps it in without
     closing the running session";
   - `start_selection.rs` and `item.rs` suggestions;
   - the help prose and examples in `capture/cli.rs`, `capture_parse.rs` (including its
     Modes/Needs text), `capture_complete/cli.rs`, and `capture_rewrite.rs`, including
     any sentence that still says `=` refuses while a session runs (add a
     `bob capture '=#bugs'` override example near the other `=` examples);
   - the did-you-mean warning;
   - comments.

   Afterwards, `rg -n '`==|"==|==#|==<|==~' src
   tests`should list only retired-form code and tests,`==x`-family code, and
   highlight-prose tests.

7. **Migrate the tests.**
   - Override behavior moves to the `=` spelling, mostly
     `tests/cli/capture/pomodoro_override.rs`. Delete cases that are now duplicates of
     the eq-overrides cases.
   - `==` parse, editor, chain, and completion tests become retired-form tests:
     - `capture_language/tests/grammar.rs`, `editor_modes.rs`, `chain.rs`,
       `completion.rs`;
     - `tests/cli/capture/parse_pomodoro.rs`, `pomodoro_whole_item.rs`,
       `complete_editor.rs`;
     - `capture_complete/tests/output.rs`;
     - the `capture_parse.rs` unit tests.
   - Add retired-form tests in both parsers and on the CLI: `==`, `==3`, `==#bugs`,
     `==3#bugs~2`, `==#bugs@3`, `==#`, `==~`, `==#bugs=3`, `x ==` (failing on its own
     range), the unchanged prose rows, and the new `==x` message.
8. **Verify.** Run `just check` and the `rg` audit from step 6, then do a dry-run smoke
   of each retired form and its rewrite.

## Phase: docs

**Precondition:** bob-cli-9b.3 is closed, because it rewrites these files. Check with
`sase bead read bob-cli-9b -r "<why>"`. If it is still open, report the blocker on your
phase bead and stop; do not edit around it. Quote every message and output block from a
bob built at this tree, never from memory.

1. **`docs/capture.md`.**
   - **Grammar at a glance.**
     - The `=`/`=<X>` row also restarts the running session.
     - The `=<X>#pomodoro` row also swaps per `swap_clock`.
     - Delete the three `==` rows and add one retired-`==` row with its exact error.
   - **The `#` and `=` meaning table.**
     - Replace the `==…` rows with `=` override rows or retired-form rows.
     - Keep `==important== thing` prose, with the reason "`==` is retired; a bare `==`
       run followed by prose stays prose".
   - **Starting the next Pomodoro.**
     - Lifecycle table: `=` starts or restarts; `=#name` starts or swaps; `x` stops.
     - Mnemonic: "`=` starts the next session — or restarts the running one; `=#name`
       starts that one — or swaps it in; `x` stops the running one."
     - Replace the still-running guard and its switch idiom with a pointer to the
       override section.
   - **Starting a named Pomodoro.** Rewrite the guards.
   - **Restarting or swapping the running Pomodoro.** Rename it to "Overriding the
     running Pomodoro" and keep the worked example.
     - Respell every row and error.
     - Delete the idle-fallback paragraph.
     - Document `overdue`, `close_first`, the warning lines, `--refuse-override` with
       `pomodoro_override_refused`, and the Mac guard in one sentence.
   - **Starting the session atomically.** The `==` sentence becomes "only a whole-item
     `=` start overrides it; link starts never do".
   - **Plan budget.** Session starts, including overrides, are never refused.
   - **Contract sections.**
     - `capture-parse`: drop the `override` spec flag.
     - `capture-complete`: describe the state-based `override` context under
       `pomodoro_start_name`.
     - Document `--refuse-override` next to `--require-pull-in-choice`.
2. **`README.md`.** Fold the three `==` grammar rows into the `=` rows and add the
   retired row.
3. **Memory.**

   > [!decision] decision_record Use `/sase_memory_write` to add the strand
   > `sase/memory/decisions/eq-overrides-the-running-pomodoro.md`, then run
   > `sase memory init`.
   >
   > - **Applies to:** bob-cli and Bob Mac Capture.
   > - **Claim:**
   >   - A whole-item `=` restarts or swaps the running Pomodoro; `==` is retired.
   >   - Link starts never override.
   >   - Bob Mac Capture commits an override only after its banner was on screen
   >     (`--refuse-override`).
   > - **Why:** `==` duplicated `=` in every state except a refusal, and the warning
   >   plus the guard replace that refusal.
   > - **Rejected alternatives:** keeping `==`; keeping the refusal; a confirmation
   >   every time; guessing intent on overdue sessions.
   > - **Cost:** a CLI `=` overrides without any preview.
   > - **Reopens when:** an unintended override happens with the banner on screen.
   > - **Evidence:** this epic's plan and commits.

4. **Verify.** `rg -n '==' docs/capture.md README.md` should show only retired-form
   documentation, highlight prose, and Dataview `===`. Every quoted message must match
   bob's real output.

## Phase: mac-warning

**Precondition:** bob-cli-9b.4 is closed; check it the same way as the docs phase.

1. **Setup.**
   - Open the repo with `/sase_repo` (`sase repo open bob-mac-capture`) and read its
     `AGENTS.md`.
   - Build bob from the bob-cli checkout at this tree, so fixtures come from the shipped
     grammar.
   - Keep the Swift simple: early returns, and explicit types in long expressions.
2. **`CaptureModels.swift`.**
   - Add `PomodoroStartPrevious.overdue` (`decodeIfPresent`, default false).
   - Add `PomodoroStartOverride.closeFirst: String?`.
   - Delete `PomodoroStartSpec.isOverride`.
3. **`CapturePomodoroStartPresentation.swift`.**
   - Delete `.idleStart` and `idleCaption`. Unknown actions and `"start"` decode as
     `.start`.
   - Add an `OverrideWarning` value for restarts and swaps, plus an `isOverride`
     convenience. It carries `headline`, `rangeText` (en dash), `detail`, `closeFirst`,
     and `accessibilityText`, with the copy from Design.
   - Prefix `accessibilitySummary` with the warning text.
   - Append ` · was 0920–0945` to the restart notification body.
4. **`CapturePanelView.swift` `startPreviewItem`.**
   - Render the banner as the first row, as specified in Design: a triangle glyph; a
     headline row with the range on the right; the detail caption; and the `Type` / chip
     / `to close it first` line.
   - Use a continuous `RoundedRectangle(cornerRadius: 8)`, filled with
     `Color.orange.opacity(0.12)` and stroked with `Color.orange.opacity(0.4)`.
   - Keep the `was …` caption, the demoted row, and every other row as they are.
5. **Footer.**
   - Add `CapturePanelModel.primaryActionIsOverride`: true when any capture in the
     on-screen preview has an override start presentation.
   - Apply `.tint(model.primaryActionIsOverride ? .orange : nil)` to the footer's
     primary `Button(model.primaryActionTitle)`.
6. **Picker copy** (`CompletionRowContent.swift`, `CapturePickerPresentation.swift`).
   - Use the strings from Design.
   - Doc comments now say the `override` object appears for `=#` while a session runs.
   - Respell every remaining `==` mention in `Sources`.
7. **Fixtures.** Regenerate from the new bob, using the worked-example vault and the
   recipe the override fixtures record.
   - Respell the drafts of the `pomodoro-start-override-restart`, `-swap-kept`, and
     `-swap-fresh-created` fixtures to `=`.
   - Add an overdue fixture.
   - Regenerate `pomodoro-start-running`, `pomodoro-start-named-still-running`, and
     `pomodoro-start-switch`, which now preview overrides. Rename any whose name now
     lies.
   - Replace the `-override-idle` and `-override-parse*` fixtures with one retired-`==`
     parse diagnostic fixture.
   - Regenerate the `-override-complete*` fixtures under `=#` and `=3#` (idle now
     carries no object).
   - Respell the matching `fake-bob` routes.
8. **Tests.**
   - Banner copy for a restart, a kept swap, a fresh swap, an overdue session, and an
     unnamed R.
   - No banner on a plain start.
   - An older bob without `overdue` or `close_first` still gets a banner, minus the
     close-first line.
   - A `"start"` action decodes as a plain start.
   - The restart notification shows `was`.
   - `primaryActionIsOverride` and the primary title.
   - Respelled row and picker tests.
   - Extend the start card's design or render test if one exists.
9. **README.** Update:
   - the start-card paragraph (banner, orange Return);
   - the editor grammar and start-name completion text;
   - the Runtime Contract (`override` now comes from `=`; `overdue`, `close_first`);
   - Notifications;
   - the "still-running error" sentence, which now describes the override card.
10. **Verify** per `AGENTS.md:`
    - Run `just portable-check` on athena.
    - On the Mac, best effort: compile a scratch copy, run `bundle.sh` and
      `just typecheck-tests`, then delete the copy.
    - Confirm CI's Lint, Build, and Test steps pass.
    - If the Mac is offline, say in the final report that the macOS targets are
      unverified.

## Phase: mac-guard

1. **`BobProcessClient.capture`.**
   - Add a `refuseOverride: Bool = false` parameter that appends `--refuse-override`.
   - Generalize the unsupported-guard-flag fallback (`isUnsupportedGuardFlagFailure` and
     its retry) so that an older bob that rejects the flag is retried without it, and
     that answer is remembered per client identity. This is safe because an older bob
     never overrides on `=`.
2. **`CapturePanelModel`.**
   - At `submit`, compute `allowsOverride`. It is true only when the on-screen preview
     matches this draft, `analysisGeneration`, and the client identity, and shows an
     override. Mirror `currentPullInPreview`.
   - Thread `allowsOverride` through `dispatchCapture` and the pull-in choice path, and
     pass `refuseOverride: !allowsOverride` on every real submit. Live and explicit
     previews never pass the flag.
   - On failure code `pomodoro_override_refused`:
     - keep the panel and the draft;
     - set no error and send no failure notification;
     - set the status to
       `A Pomodoro is running — review the override, then press Return again`;
     - request a fresh live preview, so the banner appears.
3. **`fake-bob`.** Add an env switch that refuses override drafts when
   `--refuse-override` is present, and a flag-unsupported mode.
4. **Tests.**
   - `=` + Return before any preview sends the flag.
   - After an override preview, no flag is sent and the first Return commits.
   - After a plain-start preview, the flag is sent, which is harmless.
   - A refusal leads to the calm status, no error, and a re-preview; the second Return
     commits.
   - A stale preview for another draft sends the flag.
   - An older bob falls back to retrying without the flag.
5. **README.** Add to the Runtime Contract and the start-card paragraph: "an override
   commits only after its warning was on screen".
6. **Verify** the same way as mac-warning.
