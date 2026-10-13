---
tier: tale
title: Trailing @ spellings for close keep lists and start timings
goal:
  "`=x…@<N>` closes exactly like `=x<N>…`, and `=#name…@<X>` / `==#name…@<X>` start
  exactly like `=<X>#name…` / `==<X>#name…`, in bob capture, capture-parse, and
  capture-complete."
size: medium
decisions:
  unnamed_drop_starts:
    ask:
      Should `@<X>` also follow an unnamed start's drop list, so `=~2@3` works like
      `=3~2`?
    choices:
      accept:
        "`=~2@3` and `==~2@3` start exactly like `=3~2` and `==3~2`; a bare `=@3` stays
        prose"
      reject:
        Only named starts take `@<X>`; `=~2@3` stays an error, now one that teaches
        `=3~2`
    default: accept
    why:
      Same cursor-saving win as the named form; the drop-part lexer change is shared
      anyway.
    answer: accept
proposed_by: bbugyi200.athena.0zy
decided_by: auto
create_time: 2026-10-13 10:45:33
status: wip
---

# Trailing `@` spellings for close keep lists and start timings

## Goal

Add a second spelling for the leading number of a Pomodoro session token. The number
moves to the end of the token, after an `@`, so a user who has already typed the rest of
the token can append it without moving the cursor back:

| New spelling                         | Means exactly    | Note                                                                |
| ------------------------------------ | ---------------- | ------------------------------------------------------------------- |
| `=x!1@2`                             | `=x2!1`          | the motivating example                                              |
| `=x@2`                               | `=x2`            |                                                                     |
| `=x*@1`                              | `=x1*`           |                                                                     |
| `=x!1@2~3`, `=x@2!1~3`               | `=x2!1~3`        | `@` sits anywhere among `*`, `!`, `~`                               |
| `=x@0`, `=x!2@0`                     | `=x0`, `=x0!2`   | keeps the `=x0` reset semantics                                     |
| `=!1@2`, `=*1@2`                     | `=x2!1`, `=x2*1` | short aliases                                                       |
| `@route:id=x!1@2`, `^route:id=x!1@2` | `…=x2!1`         | link close suffixes                                                 |
| `=#bugs@3`                           | `=3#bugs`        | named start, 15 minutes                                             |
| `=#bugs@3~2`, `=#bugs~2@3`           | `=3#bugs~2`      | `@<X>` and `~<K>` in either order                                   |
| `==#bugs@3`                          | `==3#bugs`       | override swap with fresh timing (does **not** take over the ledger) |
| `==#bugs~2@3`                        | `==3#bugs~2`     |                                                                     |
| `=~2@3`, `==~2@3`                    | `=3~2`, `==3~2`  | only if DECISION `unnamed_drop_starts = accept`                     |

The new spelling must be **semantically identical** to the canonical one. That covers
the vault writes from `bob capture`, the `pomodoro_close` / `pomodoro_start` spec
objects in `bob capture`, `bob capture-parse`, and `bob capture-complete` JSON, and
every diagnostic that applies to the canonical form. Only three things may differ: text
that echoes what the user typed (item `body`, `pomodoro_close.raw`, ranges), and the
span layout described below.

## Background (current state, verified with `bob capture-parse`)

- **Close lexer.** All close forms go through `close_after_eq_parts` in
  `src/native/capture_language/close_selection.rs`, which only claims `=x` when the
  first byte after `x` is a digit, `,`, `!`, `~`, or `*`.
  - That is why `=x@2` is prose today.
  - `=x!1@2` is a claimed error today: "`=x!1@2` is not a task list …".
- **Close list parsing.** `lex_close_selection` → `parse_selection_body` reads the keep
  list `<N>` only from `body[..first_marker]`.
  - The trailing lists are hard-coded as 0–3 match arms, ending in `unreachable!`.
  - `validate_selection` checks zeros, duplicates, and overlaps.
  - The execution parser (`item.rs`), the editor (`editor_pomodoro.rs`,
    `editor_classify.rs`, `editor_parse.rs`), link suffixes (`tokens.rs`
    `parse_colon_link_tail`, `classify_caret_close_suffix`), chains (`draft.rs`,
    `item.rs` `is_session_chain_token`), and the close Work Log hint (`close_log.rs`)
    all share these functions. So the close change is mostly confined to
    `close_selection.rs`.
- **Start lexer.** Start and override tokens go through `session_equals_token` in
  `src/native/capture_language/item.rs`.
  - It returns `EqualsToken::Start { suffix, counted, len, name, drop, override }`.
  - The `#name` part runs to whitespace or `~`. So `=#bugs@3` is a name-charset error
    today ("Pomodoro name `bugs@3` …"), and `=~2@3` is a drop-list error.
- **Start consumers.**
  - Execution named-start branch: `item.rs` `parse_pomodoro_equals_item`, using the
    helpers `named_token_error`, `named_shape_error_with_drop`,
    `spaceless_start_drop_is_valid`, `lex_start_drop_for_execution`, plus the unnamed
    branch right after it.
  - Editor: `editor_pomodoro.rs` `parse_editor_named_start_item`,
    `named_start_token_diagnostic`, `parse_editor_start_item`.
  - Completion: `completion.rs` `named_start_name_at`, `pomodoro_start_name_field`,
    `pomodoro_start_override_at` (which drives `override.keeps_ledger` in
    `capture_complete/engine.rs`).
  - Today `==#bugs@3` reports `keeps_ledger: true`, and accepting a name candidate
    replaces `bugs@3`. Both are wrong under the new grammar.
- **Execution of starts.** Override behaviour keys on `spec.raw.is_empty()`
  (`capture/pomodoro_start.rs`, `keeps_ledger`). If the parser puts the `@<X>` text into
  `spec.raw`, execution needs **no change**.
- **No `@`-route side effects.** Every route, marker, and completion `@` check is a
  token-prefix check. An inner `@` (`=x!1@2`, `@r:id=x!1@2`) never triggers route
  parsing or route completion, and Bob Mac Capture only falls back to a _leading_ `@`.
- **Bob Mac Capture is a thin client.** It reuses Bob's span kinds.
  - Its close-list auto-comma assist (`CaptureCloseTaskCommaAssist`) arms on
    `pomodoro_close_in_progress` spans. Reusing that span kind for `@<N>` keeps the
    assist working with no Mac change.
  - `needs` decodes as `[String]`, so a new need value is additive.
- **No other grammar consumers.** No grammar golden or digest fixtures exist
  (`tests/fixtures/line_digest` is unrelated), and `bob-plugins` does not parse these
  tokens.

## Grammar contract

### Close: `@<N>` is the keep list typed last

1. **Recognition.**
   - `close_after_eq_parts` also accepts `@` as the first byte after `x`/`X`. So `=x@2`,
     `=x@`, and `=x@foo` become claimed closes. `=x@foo` reports the existing bad-list
     error, exactly like `=x!foo`.
   - `=@…` is **not** a close alias. `=@2` and `==@3` stay prose, unchanged.
   - Every caller inherits the new recognition through the shared helpers:
     - link suffixes;
     - project-note `=x` rejection;
     - chains (`=x@2 =`, `+2 =x@2`, `=x!1@2 =#bugs@3`);
     - `double_close_tail_is_claimed`, so `==x@2` becomes the existing "`==x@2` is not a
       close" teaching error, spelled with `=x@2`.
2. **Lexing.**
   - `@` becomes a fourth list marker in `lex_close_selection` / `parse_selection_body`,
     alongside `*`, `!`, `~`, in any order.
   - Generalize the hard-coded marker match arms into a loop over the sorted marker
     positions, so four markers never hit `unreachable!`.
   - The `@` list **is** the `<N>` list. Apply every rule `<N>` has today to its text:
     - comma lists;
     - the `=x1,!2` "expected a task number after `,`" special case (`=x@1,!2`);
     - `0` alone means none (`=x@0` ≡ `=x0`, `=x!2@0` ≡ `=x0!2`), and `00` / `0,1` give
       the zero-alone error;
     - duplicate and overlap checks against `*`, `!`, `~`;
     - the `None` vs `Some` rule for `in_progress`. An `@` list is always explicit, so
       it gives `Some`.
3. **Ranges.**
   - `in_progress_range` for an `@` list covers the `@` plus the list (like the
     `*`/`!`/`~` ranges include their sigil). The editor's `pomodoro_close_in_progress`
     span therefore covers `@2`.
   - For overlaps that involve the keep list, the diagnostic range points at the
     later-typed occurrence. Behaviour is unchanged for a leading `<N>`, which always
     comes first. For example, `=x!1@1` points at the `1` after `@`.
4. **Diagnostics.** New texts live in `markers.rs`.
   - **Keep list typed both ways.** This applies when the leading `<N>` is non-empty and
     an `@` group is present: `=x2@3`, `=x0@2`, `=x2@`.
     - Message:
       ``"`<token>` lists the in-progress tasks twice; type them once, right after `=x` or after `@` (for example `=x2,3` or `=x@2,3`)"``.
     - Range: the whole `@` group, from the sigil through the end of its list (just the
       `@` when it dangles).
     - This error wins over the dangling-`@` incomplete state.
   - **Second `@`.** `"use one `@`list:`=x!1@2,3`"`, on the second `@`. Add it next to
     the existing `!`/`~`/`*` count checks.
   - **Dangling `@` at token end** (`=x@`, `=x!1@`, `=*1@`). This is an `Incomplete`
     outcome with `separator: '@'` and `separator_range` on the `@`, mirroring the
     trailing-`~` branch.
     - The editor reports `incomplete` needing `pomodoro_close_task`, with one
       `interactive_placeholder` over `@`. Bob Mac Capture's existing pending-trim logic
       then previews the close typed so far.
     - Execution reports the existing `close_selection_incomplete_error`: "`=x!1@` is
       incomplete: type a task number after `@`".
   - **Empty `@` directly before another marker** (`=x@!1`). This is an error, unlike
     today's lenient empty `~`.
     - Message: `"expected a task number after `@`"`.
     - Range: the `@`.
   - **Comma right after `@`** (`=x@,`, `=x!1@,2`). Add `@` to the
     `rfind(['!', '~', '*'])` set in the trailing-comma branch. The result is "expected
     a task number before `,`" on the comma, matching `=x1!,`.
   - Leave `close_selection_bad_list_error` and the zero-alone text unchanged. Their
     canonical examples stay correct.
5. **Work Log entries.** No code change is needed. The inline-entry no-spaces hint
   (`close_log.rs`) will now fire for `=x @2 …` and `=x!1 @2`, because the glued `=x@2`
   / `=x!1@2` lexes. That is intended: the hint already offers the `=x 1 @2 …` escape.
   Add a test that pins it.

### Start: `@<X>` is the timing typed last

1. **Lexing in `session_equals_token`.**
   - The `#name` part now ends at ASCII whitespace, `~`, or `@`.
   - After the name, two optional parts may follow, each at most once and in either
     order: an `@<X>` timing part and a `~<K>` drop part.
   - Per DECISION `unnamed_drop_starts`, an unnamed token whose `~<K>` part is present
     may also take an `@<X>` part after it.
   - Whichever optional part comes first ends at whitespace or at the other part's
     sigil. The second part runs to whitespace, so trailing junk lands inside it and
     gets that part's own diagnostic.
   - A token with no `#name` and no `~<K>` never reads an `@`. So `=@3` and `==@3` stay
     prose, and `=3@2` keeps its shape error.
   - Extend `EqualsToken::Start` with `timing: Option<(usize, String)>`: the
     token-relative `@` offset plus the text after `@`, mirroring `drop`. Add one small
     helper that returns the effective `<X>`: the leading suffix, or the `@` text.
   - `len` covers every part.
   - Update the doc comments on `EqualsToken` and `session_equals_token`.
2. **Semantics.**
   - The effective `<X>` is validated by `parse_pomodoro_start_suffix`, with the same
     messages, so `@-` and `@3-2` behave like `=-` and `=3-2`.
   - The spec is built from it, so `PomodoroStartSpec.raw` equals the canonical
     spelling's `raw`.
   - `counted` reflects the effective `<X>`.
   - Execution stays untouched: `==#bugs@3` swaps BUGS in with fresh 15-minute timing,
     and only an `==` token with no timing at all (`==#bugs`) takes over the ledger.
3. **Diagnostics.** Execution and editor share the same text and ranges.
   - **Both timings** (`=3#bugs@5`, `==3#bugs@`).
     - Message:
       ``"`<token>` gives the timing twice; write `<sigil><X>#<name>` or `<sigil>#<name>@<X>`"``.
       Fill both suggestions from the `@` text when it is non-empty, otherwise from the
       leading text.
     - Range: the `@` part.
     - This error wins over the dangling-`@` incomplete state.
   - **Dangling `@`** (`=#bugs@`, `==#bugs~2@`, and `=~2@` under `accept`).
     - Execution error:
       ``"`<token>` is incomplete: type the timing after `@` (for example `<token>3`)"``.
     - Editor: mode `incomplete` with a new `Need::PomodoroStartTiming` (serialized
       `pomodoro_start_timing`, added in `editor_model.rs` next to `PomodoroStartTask`)
       and one `interactive_placeholder` span over the `@`. Keep `section` as the typed
       name. Report **no** `pomodoro_start` spec object, because the timing is unknown
       and an empty `raw` would read as a ledger takeover. It is never a takeover.
   - **Empty name with a timing** (`=#@3`). Report the existing empty-name error, the
     same way `=#~2` does.
   - **`=#bugs=3` teaching error** (inline in `item.rs` `named_token_error` and
     `editor_pomodoro.rs` `named_start_token_diagnostic`).
     - Change it to
       ``"write the duration after `@` or before the name: `=#bugs@3` or `=3#bugs` instead of `=#bugs=3`"``.
     - Keep the typed sigil and the `~<K>` suffix in both suggestions.
     - Update the 6 existing assertions and docs rows that quote the old text.
   - **No-space hint.**
     - `=#bugs @3` (and `=#bugs~2 @3`) gets
       ``"write the timing right after the name, with no space: `=#bugs@3`"`` (echo the
       spaceless join), not the generic shape error.
     - Generalize `spaceless_start_drop_is_valid` (and its editor twin) to accept a
       valid timing part as well. Pick the timing hint when the first extra word starts
       with `@`; keep the drop hint otherwise.
   - **Unnamed drop forms.**

     > [!decision] unnamed_drop_starts = accept `session_equals_token` reads an `@<X>`
     > part after an unnamed `~<K>` part, so `=~2@3` / `==~2@3` are valid starts
     > identical to `=3~2` / `==3~2`; `=~2@` is the dangling-`@` incomplete state;
     > `=3~2@4` is the timing-twice error. The unnamed execution and editor branches use
     > the effective `<X>` and emit the trailing `pomodoro_start` span. Add these forms
     > to the chain, editor, and CLI equivalence tests.

     > [!decision] unnamed_drop_starts = reject Unnamed tokens never read an `@` part:
     > the drop list keeps running to whitespace, and `lex_start_drop` sees the `@`.
     > Before the generic "not a drop list" error, detect a drop text of the form
     > `<K>@<X>` with a valid `<X>` and report
     > ``"write the timing before the drop list: `=3~2` instead of `=~2@3`"``
     > (sigil-aware, keeping the typed `<K>`). Drop the unnamed rows from the tests and
     > docs, and add the teaching error instead.

4. **Editor spans.**
   - Keep today's spans: `pomodoro_start` over the sigil plus the leading `<X>`,
     `pomodoro_name` over the name, and `pomodoro_start_drop` over `~<K>`.
   - Add a second `pomodoro_start` span over `@<X>` including the `@`. The link forms
     already put a `pomodoro_start` span after `pomodoro_name`.
   - Example: `==#bugs~2@3` → `pomodoro_start [0,2)`, `pomodoro_name [3,7)`,
     `pomodoro_start_drop [7,9)`, `pomodoro_start [9,11)`.
   - Spans stay sorted and non-overlapping.
   - Any code that derives name or drop offsets from `sigil_len + suffix.len()` must
     keep using the **leading** suffix only.
5. **Completion.**
   - `named_start_name_at` must use the leading suffix for `name_start`. With the lexer
     change the name field already ends before `@`, so accepting a candidate in
     `=#bu|gs@3` replaces only `bugs`.
   - A cursor inside `@<X>` yields no completion.
   - `pomodoro_start_override_at` returns `keeps_ledger = true` only when the leading
     suffix is empty **and** no `@` part was typed.
6. **Human output.** `capture_parse.rs` `format_pomodoro_start` and the
   `pomodoro_start.rs` did-you-mean warnings render the canonical `{sigil}{raw}#{name}`.
   That stays as is, because it is a valid spelling of the same start. Add one test so
   this is deliberate.

### Non-goals

- Link start suffixes (`^route:id#name=<X>`, `@route:id#name=<X>`) already put the
  timing last. They do not take `@<X>`.
- `=@…` / `==@…` keep today's behaviour (prose), and `=@` is not a close alias.
- No `bob-mac-capture` or `bob-plugins` code change. The JSON contract changes only
  additively: one new `needs` value, `pomodoro_start_timing`. See the follow-up in
  step 9.
- No schema-version bump. Additive needs and spans have never bumped `SCHEMA_VERSION`.

## Implementation steps

1. **`close_selection.rs`.**
   - Recognition: add `@`.
   - Count check: second `@`.
   - Trailing-`@` incomplete branch and keep-list-twice check.
   - Trailing-comma marker set.
   - Generalized marker splitting.
   - `@`-list handling in `parse_selection_body` / `validate_selection`: feed it in as
     the `<N>` list with its own base and range; the empty-`@` error; overlap ranges on
     the later occurrence.
   - Update the module and struct doc comments (`=x[<N>][*<P>][!<M>][~<K>][@<N>]`).
2. **`markers.rs`.** Add the new message functions: keep-list-twice, one-`@`-list,
   expected-number-after-`@`, timing-twice, timing-incomplete, timing no-space, and
   (`reject` only) timing-before-drop.
3. **`item.rs`.**
   - `session_equals_token`: name terminator, the two order-free optional parts, the new
     `timing` field, and the effective-`<X>` helper.
   - The named and unnamed execution branches of `parse_pomodoro_equals_item`: effective
     `<X>`, the new diagnostics, and the incomplete-`@` error.
   - `named_token_error` message, `spaceless_start_drop_is_valid` generalization, and
     `is_session_chain_token`. A token with a `timing` part is claimed, so it must
     chain; keep the "chain token ⇔ claimed" invariant.
   - Fix every exhaustive destructuring of `EqualsToken::Start`.
4. **`editor_pomodoro.rs`.**
   - `parse_editor_named_start_item`, `named_start_token_diagnostic`, and
     `parse_editor_start_item` (drop branch, under `accept`): the timing lex, the second
     `pomodoro_start` span, the `pomodoro_start_timing` incomplete outcome, and the same
     diagnostics and ranges as execution.
   - Confirm the close paths need nothing beyond the shared lexer: the
     `in_progress_range` spans and the `@` incomplete separator via
     `Need::PomodoroCloseTask`. Do the same for the link-close paths in
     `editor_classify.rs` / `editor_parse.rs` / `tokens.rs`.
5. **`editor_model.rs`.** Add `Need::PomodoroStartTiming` → `"pomodoro_start_timing"`.
6. **`completion.rs`.** Make the `named_start_name_at` / `pomodoro_start_override_at`
   changes above.
7. **Help text.** Update it in `src/native/capture/cli.rs` (start, override, and close
   grammar paragraphs; add a `bob capture '=#bugs@3'` example and a `=x!1@2` example),
   `src/native/capture_parse.rs` (the long `capture-parse` help), and
   `src/native/capture_complete/cli.rs`. Keep them consistent with the existing style.
   Extend `tests/cli/help.rs` `capture_named_start_help_mentions_start_forms` with the
   new example.
8. **Docs.**
   - `docs/capture.md`:
     - quick-reference rows (the start, override, and close rows near the top, plus the
       examples table with new valid, incomplete, and error rows for `=x!1@2`, `=x@2`,
       `=x@`, `=x2@3`, `=#bugs@3`, `==#bugs@3`, `==#bugs@`, `=3#bugs@5`, and
       `=x @2 foo`);
     - the session-operator table and mnemonic;
     - "Starting a named Pomodoro" (replace "the `<X>` always goes **before** the `#`"
       with the new two-spelling rule);
     - the override section (`==#name@<X>` is a fresh-timing swap, never a takeover);
     - the close section and grammar block (`=x[<N>][*<P>][!<M>][~<K>]` plus "`<N>` may
       instead be typed last as `@<N>`");
     - the capture-parse section (the new `pomodoro_start_timing` need, the second
       `pomodoro_start` span, the `@` close separator);
     - the completion section.
   - `README.md` rows and paragraphs for `=<X>#pomodoro` and the close grammar.
   - Update the `=#bugs=3` docs row to the new teaching text.
9. **Follow-up bead.** After the change lands, use `/sase_new_task` to file one
   `feature` task bead for `bob-mac-capture`. It should treat the new
   `pomodoro_start_timing` need like `pomodoro_start_task` in
   `CapturePanelModel.closePendingTrim`, so a dangling `=#bugs@` previews calmly instead
   of flashing Bob's execution error for one keystroke. Do not edit `bob-mac-capture` in
   this tale.

## Tests

Extend the existing suites. Reuse their helpers and fixtures; do not invent parallel
ones.

- **`close_selection.rs` unit tests**
  - `lex_reports_valid_selections`: `=x@2`, `=x!1@2`, `=x@2!1~3`, `=x*@1`, `=x@0`,
    `=x!2@0`, `=!1@2`, with the lists and `in_progress_range` covering `@…`.
  - `lex_reports_dangling_separators_as_incomplete`: `=x@`, `=x!1@`, `=x@2,`.
  - `lex_reports_precise_diagnostics`: `=x2@3`, `=x2@`, `=x@2@3`, `=x@!1`, `=x@,`,
    `=x@1,!2`, `=x!1@1` (range on the `@` occurrence), `=x@0,1`, `=x@foo`.
  - `selection_shape_detection`: `=x@2` and `=x@` become shaped, and `=@2` stays
    unshaped.
- **`start_selection.rs` / `item.rs` / `tests/grammar.rs`**
  - `execution_parses_equals_family_starts_alongside_close` and
    `execution_parses_override_family_with_identical_claim_rules`: every new start
    spelling yields the same `CaptureKind::PomodoroStart` as its canonical twin.
  - Every new diagnostic.
  - `=@3` / `==@3` are still prose, and `=3@2` keeps its shape error.
- **`tests/chain.rs`**
  - `chain_token_predicate_covers_the_documented_table` and
    `chain_token_predicate_equals_claimed_for_single_tokens` cover `=x@2`, `=x@`,
    `=x!1@2`, `=#bugs@3`, `==#bugs~2@3`, `=~2@3`.
  - A chain `=x!1@2 =#bugs@3` with absolute spans.
- **`tests/editor_modes.rs` / `tests/editor_spans.rs`**
  - The span layouts above.
  - The `pomodoro_start_timing` incomplete outcome with no spec.
  - `pomodoro_close_task` incomplete for `=x!1@`.
  - Execution and editor agree on every new diagnostic's message and range.
- **`tests/completion.rs` and `src/native/capture_complete/tests/output.rs`**
  - The name replacement range stops before `@`.
  - No completion inside `@<X>`.
  - `keeps_ledger` is false for `==#bugs@3` and `==#bugs@`, and still true for
    `==#bugs`.
- **`close_log.rs`**
  - `=x @2 foo` and `=x!1 @2` now report the no-spaces hint with the `=x 1 @2 foo`
    escape.
- **`capture_parse.rs` JSON tests**
  - `capture-parse -f json` for each new spelling equals the canonical spelling's JSON,
    except `body`/`raw` echo fields and spans.
  - `format_pomodoro_start` renders the canonical form.
- **CLI integration (`tests/cli/capture/`)**
  - Equivalence tests modelled on `capture_pomodoro_close_short_alias_equivalence` in
    `pomodoro_close_selection.rs`. Run both spellings against twin fixture vaults and
    assert equal `pomodoro_close` / `pomodoro_start` result fields **and**
    byte-identical day files.
  - Close pairs: `=x!1@2`/`=x2!1`, `=x@2`/`=x2`, `=x*@1`/`=x1*`, `=x@0`/`=x0`, and a
    link form `^route:id=x!1@2`.
  - Start pairs (`pomodoro_start_named.rs`, `pomodoro_override.rs`):
    `=#bugs@3`/`=3#bugs`, `=#bugs~2@3`/`=3#bugs~2`, `==#bugs@3`/`==3#bugs` (fresh
    timing, old session intact), and `=~2@3`/`=3~2` under `accept`.
  - Error-text cases in `pomodoro_close_selection.rs` diagnostics and
    `pomodoro_start_named.rs`.

## Acceptance

- `just check` passes: fmt, clippy with the rustc gate, and every test binary.
- `bob capture-parse -f json -- '=x!1@2'` reports mode `pomodoro_close` with
  `in_progress: [2]`, `complete: [1]`, and spans `pomodoro_close [0,2)`,
  `pomodoro_close_complete [2,4)`, `pomodoro_close_in_progress [4,6)`.
- `bob capture-parse -f json -- '==#bugs@3'` reports `pomodoro_start` with `raw: "3"`,
  `duration_units: 3`, and `override: true`, and `section` is `bugs`.
- `'=#bugs@'` reports `incomplete` / `pomodoro_start_timing`.
- `'=x@'` reports `incomplete` / `pomodoro_close_task`.
- `'=@3'` is still mode `task`.
- Docs, README, and help text describe both spellings, and no doc still claims the `<X>`
  must precede `#`.
