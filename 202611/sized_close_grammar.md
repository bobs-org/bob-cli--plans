---
tier: tale
title: Claim sized Nx closes in both capture parsers
goal:
  Parse sized closes consistently, expose their size and spans, and refuse execution
  until the timing phase lands.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ix.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-ix.1
create_time: 2026-11-09 05:32:54
status: wip
---

- **PARENT:**
  [202611/sized_pomodoro_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/sized_pomodoro_close.md)
- **BEAD:**
  [bob-cli-ix.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ix/bob-cli-ix.1.md)

# Claim sized Pomodoro closes in both capture parsers

Implement assigned phase `bob-cli-ix.1` (grammar) of epic `bob-cli-ix`, following
`plan:202611/sized_pomodoro_close.md`. This is one bounded coding-agent task: extend the
existing close lexer and its consumers, preserve the existing selection grammar, and
refuse execution until the dependent execution phase supplies timing semantics.

## Current state and binding decisions

The phase was read with `sase bead read bob-cli-ix.1`; it is in progress, assigned, and
has no earlier implementation or remaining-work notes. At inspected HEAD `96196052`,
`PomodoroCloseSpec` has no size, the close classifiers only recognize an unsized `x`,
and no sized-close execution guard exists. The requested work has not landed. Recheck
the phase notes and current code before implementation to avoid duplicating intervening
work; never change the assigned status by hand or reopen it.

The parent decisions are final:

- `chain_scope = front`: a sized whole-item token can be an operator only first on its
  physical line, after zero or more adjust/shift operators. Once a close or start
  precedes it, it keeps its previous text interpretation. A sized token never begins an
  inline Work Log trail.
- `decision_record = no`: edit no memory notes and create no decision record.

This phase does not implement sized timing, close execution JSON, human execution
cards/tips, Mac changes, help, or documentation. Those belong to later phases. Do not
change `Cargo.toml` versions or `CHANGELOG.md`. Land implementation with
`feat(capture)!: claim the sized Nx close in the capture grammar`, since claiming
previous prose is a grammar break.

Preserve the current `GRAMMAR_REVISION`. The epic was written against 19, but
`src/native/capture_language/mod.rs` is already 21 after record-update work
(`12641768`). This phase must neither bump nor lower it. The execution phase must
reconcile its stale instruction to set 20 with the then-current revision; leave that
coordination on this phase as a `PROPOSED FOLLOW-UP:` note for the epic's land agent.

## Required behavior

The whole-item spelling is `[N]x[K][*P][!M][~D][@K]`, retaining every existing
selection, wildcard, ordering, log, and completion rule. `N` is one or two ASCII digits,
a value from 1 through 99, glued to lowercase `x`. Leading zeros are valid. For link
suffixes, preserve the existing `x`/`X` case rule: `=2X` is also a sized close. A number
before `x` is never a task selection and does not change `has_selection` or
`is_explicit` semantics.

Only claim a normal sized close when the body after `x` is empty or begins with an ASCII
digit, comma, `!`, `~`, `*`, or `@`. Recognize the explicitly reserved `#` and `=` near
misses separately through the same prefix lexer. Do not broaden claiming to arbitrary
words containing digits and `x`.

| Inputs                                                                          | Required interpretation                                                                         |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `2x`, `02x`, `12x`, `99x`                                                       | Sized close; blocks 2, 2, 12, 99; raw digits retained                                           |
| `2x!2`, `2x*!2`, `2x1,3!2`, `2x!2@1`, `2x0`                                     | Existing selections following a sized close; `0` after `x` is still an empty task selection     |
| `2x wired the lexer`, `2x` with numbered, positional, or star child log bullets | Existing Work Log grammar with a sized owner                                                    |
| `2x =`, `2x =bugs`, `--1 2x!2`, `+1 --1 2x`, `2x x=fix~`                        | Existing chain splitting with a front-position sized close                                      |
| `@r:id=2x!1`, `^r:id=2x`, `Text @r:id=2x`, `^r:id=02X!1`                        | Existing link/task close mode with a sized suffix                                               |
| `2x~`, `2x1,`, sized incomplete link suffixes                                   | Partial spec and existing incomplete selection needs; execution parser refuses incomplete input |
| `2X`, `2X4 lumber`, `2 x!2`, `2xx`, `2x-ray`, `2.5x`, `２x`                     | Ordinary prose in the grammar                                                                   |
| `x made it 2x`, `x 2x faster`                                                   | One unsized close whose log includes the sized-looking text                                     |
| `= 2x`, `=bugs 2x`, a sized token after a previous close/start                  | Preserve prior text parsing; do not split off a sized operator                                  |

Shared teaching errors, used by both parsers, must include these corrections:

- `0x` and `00x`:
  `` `0x` needs at least 1 block: write `1x` to close at 5 minutes, or `x0` to reset the session ``
  (display the actual offending token).
- Three or more digits such as `100x` or `0910x`:
  `` `100x` is too large: a close size is 1–99 five-minute blocks (`12x` is an hour) ``.
  Reserve the shape even when leading zeros would make its numeric value small; avoid
  integer overflow on long digit runs.
- Whole-item `=2x`:
  `` `=2x` is not a start: write `2x` to close at 10 minutes, or `=2#x` to start a Pomodoro named X ``.
  Preserve the legitimate link suffix `@r:id=2x` and other named starts; do not route
  this error through the retired `=x` migration rewrite and silently drop the size.
- `2x=3`:
  `` `2x=3` mixes a close size with a past record: `x=` takes its size after `=`; write `x=3` to record 15 minutes, or `2x =3` to close at 10 minutes and then start ``.
- `2x#bugs`: the existing `x#bugs` near-miss message, teaching `2x =bugs` and preserving
  the size plus existing name/literal handling.

Parameterize the messages for the typed size/token. Editor failures use
`invalid_pomodoro_close` and valid UTF-8 byte ranges covering the offending bytes.

## Implementation

1. **Give the prefix one owner and add the additive model.** In
   `src/native/capture_language/model.rs`, add a serializable, comparable
   `CloseSize { raw: String, blocks: u32, minutes: u32 }` and
   `PomodoroCloseSpec.size: Option<CloseSize>` with
   `#[serde(skip_serializing_if = "Option::is_none")]`. Unsized constructors,
   past-record closes, and all struct literals keep `None`; sized constructors propagate
   the lexed value without re-parsing the raw token independently. Keep
   `PomodoroCloseSpec.raw` exactly typed (`2x!2`, `=02X!1` on a suffix).

   Put the shared prefix lexer in `close_selection.rs`. Its result must distinguish
   non-close prose, claimed size errors, and a valid prefix; return the size, prefix
   byte extent, and selection body/range. Keep zero and 3+-digit failures claimed even
   when there is no selection. Use explicit whole-item/link case context. Reuse the
   existing selection lexer for everything after the prefix. Update
   `classify_whole_item_close`, `whole_item_close_after_x`, `close_after_eq_parts`,
   `classify_link_close`, and `link_close_after_x` so a valid empty sized suffix cannot
   fall through to the start parser. Update `plain`, `close_spec_from_lex`, and
   `close_spec_from_incomplete` (and callers) to carry size without affecting old
   serialized payloads. Centralize diagnostic copy in `markers.rs`.

2. **Integrate both parsers and shared line splitting together.** Adapt `item.rs`
   whole-item exact, inline-log, and child-bullet paths; `tokens.rs` at-link and
   caret-link suffix classifiers; and `editor_pomodoro.rs` / `editor_classify.rs`. Route
   `is_plain_whole_item_close`, `is_close_hash_token`, `session_equals_token`,
   `is_session_chain_token`, and the `2x=` near-miss handling around `past_token_len`
   through the prefix owner. Do not convert a sized-past near miss into an actual past
   record. Audit every close-related one-byte `x` assumption, especially
   `token_start + 1`, `x_len == 1`, and literal `raw.eq_ignore_ascii_case("x")` fast
   paths; preserve unrelated byte arithmetic.

   In `draft.rs`, share a position-aware chain predicate with rewrite eligibility: a
   sized whole-item token is eligible at index zero or after only adjust/shift tokens.
   Apply it to the all-chain test, lead and suffix scans in `split_inline_close_tokens`,
   and `session_chain_tokens`. A sized token is never a trail head. Apply position
   awareness in `close_log.rs` as well: its current leading-session reorder check calls
   `session_equals_token` without context, which would wrongly reject the required
   `x 2x faster` log after adding sizes. Preserve actual start/unsized-close trail
   behavior and attachment of child logs.

3. **Expose spans, partial states, and preview text.** Grow only the operator span
   through the digits and `x`: `2x!2` yields `0..2 pomodoro_close` and
   `2..4 pomodoro_close_complete`; `12x` has one operator span over `0..3`. For link
   suffixes, include `=` plus digits in the span that currently owns `=x`. Selection
   spans, diagnostic ranges, `needs`, and completion replacement ranges must start at
   the correct shifted byte offsets. Exercise both a UTF-8 line before the item and a
   UTF-8 task body before a link suffix. Use the existing completion machinery in
   `completion.rs`; size digits must never become a selection/comma-assist field.

   The JSON spec adds only `size: {"raw":"2","blocks":2,"minutes":10}` when present.
   Preserve all unsized JSON, including `pomodoro_past.close`. In
   `capture_parse.rs::format_pomodoro_close`, prepend the minutes to the existing
   parenthetical summary: `2x (10m)` and `2x!2 (10m · complete 2 · defer the rest)`.
   Update both formatter branches so selections, wildcard summaries, and logs retain
   their current wording. Do not edit the help strings in that file.

4. **Extend editor aliases without changing grammar prose.** In `rewrite.rs`, use the
   shared prefix lexer and line-position predicate in the close-outcome alias locator.
   With a cursor, accept `2xx -> 2x!`, `2x1s -> 2x1*`, `^r:id=2xs -> ^r:id=2x*`, and
   `--1 2xs -> --1 2x*`. Preserve existing lowercase alias eligibility even though link
   execution accepts uppercase `X`. `x made 2xx`, `x 2xx`, `= 2xx`, `Plan 2xx`, `2Xx`,
   quoted/code text, child bullets, invalid sizes, and calls without a cursor stay
   literal. Each rewrite changes only the final alias byte; size digits, raw leading
   zeros, cursor coordinates, and all surrounding content survive. Do not implement a
   draft-wide digits-plus-x replacement.

5. **Refuse every sized close in capture execution.** Use one shared size guard
   producing `CaptureError::usage_code` with `pomodoro_close_size_unavailable` and the
   exact template
   `` `<spec.raw>` closes at a size, which this bob cannot execute yet; nothing was written ``.
   Wire it before any close or reset behavior for all three paths:
   `plan_pomodoro_close_item`, `plan_pomodoro_close_link_item`, and
   `plan_pomodoro_close_task_item` in `capture/pomodoro_close.rs`. Place it before
   link/task staging, or centralize equivalent preflight over the parsed draft in
   `capture/plan.rs` if that provides complete coverage of all three kinds. Keep the
   guard easy for phase `execute` to replace. Both normal capture and `--dry-run`
   refuse; earlier items in the same batch must not be committed. `2x0` is refused too
   and cannot reach reset logic. No timing implementation or grammar revision change
   belongs in this phase.

## Verification

Use the existing helpers in `capture_language/tests/mod.rs` (`execute_draft`, `editor`,
`field`) and the close coverage in `tests/bare_close.rs`, `tests/chain.rs`,
`tests/completion.rs`, and `tests/rewrite.rs`. A dedicated `tests/sized_close.rs` module
is suitable for the shared table and should be registered in `tests/mod.rs`.

- Add a table-driven parity test covering every row in Required behavior. For valid
  closes, compare the full `PomodoroCloseSpec`, including size, selections, and logs,
  between both parsers, across all items in chains and across whole-item, link, and task
  kinds. For prose compare modes/body and absence of size. For claimed errors compare
  execution error text to editor diagnostics; for incomplete rows assert editor partial
  size/spec/needs while execution refuses.
- Include `02x`, `12x`, `99x`, zero, all 3+-digit shapes including `001x`, and a very
  long digit run; lowercase whole-item versus uppercase link cases; wildcard, typed-last
  `@`, `2x0`, inline and child logs; front-only chains and negative trailing/after-start
  cases. Ensure existing `x=3`, `=3`, `=2#x`, retired close errors, and unsized aliases
  remain unchanged.
- Extend `tests/cli/capture/parse_pomodoro_close.rs` for JSON omission/presence, raw
  preservation, exact size/operator and selection spans, incomplete needs, UTF-8
  offsets, diagnostics, and human minute summaries. Retain unsized expected JSON
  assertions and add a past-close omission assertion.
- Add completion assertions for `2x1,`, `2x~`, and corresponding link forms; accepting a
  suggestion must preserve the size and reparse correctly. Assert `12x` never exposes a
  selection range over its prefix.
- Extend the unit alias matrix and `tests/cli/capture/rewrite.rs` with all positive and
  negative sized aliases, exact edit ranges, no-cursor behavior, UTF-8/CRLF offsets, and
  idempotence.
- Add CLI execution tests in `tests/cli/capture/pomodoro_close_size.rs` (register in
  `tests/cli/capture/mod.rs`) or the existing close module. Use the temporary vault
  helpers from `pomodoro_close.rs` and pin `BOB_NOW` / `BOB_DAY_FILE`. Exercise `2x!2`,
  `2x0`, `@bob:ready=2x`, `^bob:ready=2x`, and `New task @bob:new-size=2x`, both
  normally and in dry-run. Assert usage failure, the exact code/message, and
  byte-identical daily/task notes with no new task or file. Include a multi-item draft
  with a valid earlier task or adjust followed by a sized close to prove batch rollback.
  Unsized close tests still pass.

Run focused language/CLI close, chain, completion, and rewrite tests during work; then
`cargo fmt --all` and the canonical `just check` (format, all-target Clippy, all
workspace tests). If a check fails identically on the clean base, document the baseline
evidence as `PROPOSED FOLLOW-UP:` on this phase and follow the user's instruction to
close after verifying the phase's own work. Use `/sase_monitor` for checks requiring a
long-command handoff; wait for the staging command to exit.

Before completion, run `sase bead epic-symbols bob-cli-ix.1` and resolve every entry or
re-key its Justfile line to a still-open parent/later phase. Close only `bob-cli-ix.1`
using `sase bead close bob-cli-ix.1 --note "<verified outcomes>"`; never close the
parent epic or any ancestor plan. Do not create follow-up beads: record discoveries with
`sase bead note bob-cli-ix.1 'PROPOSED FOLLOW-UP: ...'`. End through `/sase_final` so
the host commits the verified implementation.
