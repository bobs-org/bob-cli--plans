---
tier: tale
title: Add x= past Pomodoro grammar and capture-parse output
goal: Complete bob-cli-h2.1 with grammar 12, shared parsing, editor output, stack
  metadata, and a safe execution refusal until recording lands.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-h2.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-h2.1
status: done
---

- **PARENT:**
  [202611/past_pomodoro_recording.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/past_pomodoro_recording.md)
- **BEAD:**
  [bob-cli-h2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h2/bob-cli-h2.1.md)

# Implement the x= grammar and capture-parse contract

Complete only phase **bob-cli-h2.1**, `grammar`, of **bob-cli-h2**. The approved design
is `plan:202611/past_pomodoro_recording.md`; read it with `sase artifact read` and read
the assigned bead before implementing. This tale expands that phase into one bounded
coding task. It does not change the epic's architecture or decisions.

## Starting evidence and scope

At planning time, the checkout was clean at `abf427c2`. `GRAMMAR_REVISION` was 11, and
no `PomodoroPast` model or `pomodoro_past` implementation existed. The phase was
`in_progress`, with no previous implementation or remaining-work notes.
`sase bead epic-symbols bob-cli-h2.1` reported no entries. Recheck the current tree and
phase notes before coding to avoid duplicating work that lands meanwhile.

One coder can finish this phase: it adds a text grammar using existing selection,
literal-name, and Work Log parsers, plus mechanical model propagation and protocol
tests. Ledger selection, time windows, overlap repair, actual recording, and name
completion candidates belong to later phases. Do not implement those here.

The final epic decision is `memory_decision_record = no`. **Do not edit any memory note
or regenerate AGENTS.md.** Keep Cargo.toml's version and CHANGELOG.md unchanged. Use a
`feat(capture)!: ...` implementation commit subject because previously valid prose
changes meaning. The required plan handoff is separate from that commit.

## Observable behavior

`bob capture-parse` fully understands this new whole-item session operator, in JSON and
human output. `bob capture` parses it correctly but refuses execution with the exact
usage error:

```text
recording past Pomodoros (x=) is not available yet
```

The refusal must leave every file unchanged, including when an earlier item in the same
batch would have written. Valid syntax must not acquire this execution refusal as an
editor diagnostic.

### Grammar

```text
past := "x=" E [ "!" ] [ "=" X ] [ SP inline-entry ]
E    := "" | D | name | "#" name | "#" literal
        | D "#" name | D "#" literal
        | name "@" D | "#" name "@" D | "#" literal "@" D
D    := positive ASCII integer, default 5 blocks of 5 minutes
X    := existing whole-item x selection, unchanged:
        [N][*P][!M][~K][@N], including its order rules and wildcards
```

Claim every lowercase `x=` prefix at an item start or at a session-chain token boundary,
including malformed and incomplete tokens. `X=3` and `Plan x=3` stay prose. Never
reparse E by prepending `=` and invoking `session_equals_token`: that would accidentally
allow start, merge, split, or retired-close semantics.

Bare names use `bare_start_name_len` and must start with an ASCII letter without being
close-shaped. A digit-leading name requires `#` (`x=#2fa`). Unquoted names end at `!`,
`=`, `@`, `~`, or whitespace. Introduce a past-specific extent helper; leave
`start_name_extent` and the existing selector alphabet unchanged. Reuse
`lex_exact_name_after_hash` and `canonicalize_exact_name` for straight/curly quoted
names. Consume the complete quoted region before looking for operators; characters such
as `!` and `=` inside quotes are name content for lexical purposes and remain subject to
existing name validation, never repair/close delimiters.

Only the `!` before the second `=` means repair. Pass everything after the second `=`
unchanged to `lex_close_selection`, with absolute ranges shifted to the raw input. An
absent or empty X produces a plain close; `x==0` keeps the explicit empty in-progress
selection for the recording engine to interpret later.

Required valid examples:

| Input                             | Blocks/name         | Repair | Close selection    |
| --------------------------------- | ------------------- | ------ | ------------------ |
| `x=`                              | 5 / unnamed         | false  | plain              |
| `x=3`                             | 3 / unnamed         | false  | plain              |
| `x=3!`                            | 3 / unnamed         | true   | plain              |
| `x=build`, `x=#build`, `x=build=` | 5 / build           | false  | plain              |
| `x=2#fix=*3`, `x=fix@2=*3`        | 2 / fix             | false  | park 3             |
| `x==1!2`                          | 5 / unnamed         | false  | keep 1, complete 2 |
| `x==!`                            | 5 / unnamed         | false  | complete all       |
| `x=!`                             | 5 / unnamed         | true   | plain              |
| `x=!=!`                           | 5 / unnamed         | true   | complete all       |
| `x=3!=1`                          | 3 / unnamed         | true   | keep 1             |
| `x==0`                            | 5 / unnamed         | false  | explicit none      |
| `x=#"DEEP WORK"@3`                | 3 / exact DEEP WORK | false  | plain              |

Keep raw spelling in `raw`; compare equivalent spellings by semantic fields, not raw
strings or source offsets. Preserve the named-start convention for typed names and
`section`; the lexer does not resolve a name against the vault.

### Errors and unfinished input

Malformed E syntax reports `invalid_pomodoro_past` in the editor and a usage error from
capture. Each teaching error names the typed token and preserves the epic's corrective
spelling:

| Inputs                                 | Required teaching correction                                                           |
| -------------------------------------- | -------------------------------------------------------------------------------------- |
| `x=3-1`, `x=-2`, `x=3-`, `x=build@3-1` | No offset: remove the dash part, yielding `x=3`, `x=`, `x=3`, `x=build@3` respectively |
| `x=3#fix@2`                            | Give blocks once: `x=3#fix` or `x=fix@3`                                               |
| `x=~2`, `x=3~2`, `x=build~2`           | Move drops after a second `=`: `x==~2`, `x=3=~2`, `x=build=~2`                         |
| `x=*3`                                 | Put parking after a second `=`: `x==*3`                                                |
| `x=0`, `x=build@0`                     | At least one block: `x=1`, preserving a name when present                              |
| `x=/...`, `x=*build`                   | Cannot split or merge a past-session token; explain `=/` and `=*`                      |
| `x=1!2`, `x=!2`                        | Put selection after a second `=`: `x==1!2`, `x==!2`                                    |
| `x=fix!2`                              | `x=fix=!2`; repair as well requires `x=fix!=!2`                                        |
| `x=3x`, `x=3build`                     | Insert `#` between count and name                                                      |
| Oversized D                            | Reuse the start family's "too large" wording                                           |

Close-part diagnostics and Work Log errors reuse the existing parsers and wording; do
not invent another selection grammar. Unfinished input uses existing needs only:

- `x=#`, `x=3#`, `x=#"DEEP`: `pomodoro_name`, with the same interactive `#` placeholder
  treatment as named starts and no premature error.
- `x=build@`: `pomodoro_start_timing`.
- `x==1,`, `x==~`: `pomodoro_close_task`.
- `x= 2`: `pomodoro_close_log_text`.

## Implementation

### 1. Add the shared token and typed model

In `src/native/capture_language/item.rs`, add `EqualsToken::Past` and a small dedicated
E lexer. Recognize it immediately after the bare-close/hash checks in
`session_equals_token`, before merge and retired-close handling. Preserve the selector,
duration text/typedness, repair flag, close text, byte offsets, token extent, and
unfinished states so both execution and editor adapters consume the same lexical result.
Use shared errors in `markers.rs` or a focused new past module rather than duplicating
diagnostic construction.

In `model.rs`, add `CaptureKind::PomodoroPast { spec: PomodoroPastSpec }` and:

- `PomodoroPastSpec { raw, name: Option<String>, name_is_literal, blocks: u32, blocks_typed: bool, repair, close: PomodoroCloseSpec, stack: PastStackPos }`.
- `PastStackPos { index, size, offset_blocks }`; index is 1-based.

Export these from `capture_language/mod.rs`. Parse counts without panics or wrapping.
Calculate minutes and summed offsets with a wide checked representation so accepted
large counts cannot overflow during text-only parsing. `blocks_typed` is internal, not
an extra JSON contract field.

Adapt `parse_pomodoro_equals_item` to construct this kind. Factor only the necessary
close/log plumbing so Past and ordinary Close share selection parsing,
positional/numbered/star bullet assignment, inline entries, nested details, and mixing
errors. Keep every existing `x`, start, split, merge, and retired spelling behavior
unchanged except for the three intentional compatibility changes below.

### 2. Preserve token boundaries, chain ownership, and stack positions

In `draft.rs`, teach `is_session_chain_token`, `is_chain_close_token`, trail-head
recognition, and every owner `rposition` that Past is a close for splitting. The last
close token may now be `x=` and owns child bullets. Inline text attaches to the last
close in the lead, with later session operators remaining separate items.

The current `session_chain_tokens` consumes `type_marker::masked_tokens`, while
`pomodoro_literal::merge_leading_literal_token` only merges `@`/`^` literals. Ensure
past tokens with spaced exact names stay whole at any chain position. A focused shared
tokenization adjustment may be needed in `type_marker.rs`, `pomodoro_literal.rs`, or the
draft tokenizer. Include glued `@D`, repair, and close suffixes after the closing quote;
do not swallow the following inline entry or next session token. Preserve raw byte
offsets and ordinary-prose behavior.

Add one pure `assign_past_stacks` algorithm that accepts the item sequence's optional
past specs (or a similarly small shared adapter). Call it from `parse_ordinary_draft`
after all items are parsed, and `parse_for_editor_with` after items are assembled but
before projecting the first item.

A stack is a maximal run of consecutive Past items, across same-line chains and
blank-line boundaries. Each member gets its 1-based index, total size, and the sum of
later members' blocks. Every non-Past item interrupts the run. Invalid or unfinished
items must not be skipped to join valid runs or fabricate durations. There is no clock,
vault read, window computation, or target selection here.

### 3. Thread the model through the editor and capture-parse

Add `EditorMode::PomodoroPast` and span labels `pomodoro_past` and
`pomodoro_past_repair` in `editor_model.rs`. Add optional `pomodoro_past` fields to
`EditorParse` and `EditorItemParse`, initialize every construction path, and preserve
the field in the first-item projection. Adapt `editor_pomodoro.rs`, `editor_parse.rs`,
and the close-outcome builder to use the shared past lexer and close/log parsing. Store
the typed name in `section`.

Spans cover exactly:

| Kind                               | Text                                             |
| ---------------------------------- | ------------------------------------------------ |
| `pomodoro_past`                    | `x=` and any leading count                       |
| another `pomodoro_past`            | `@D` timing suffix                               |
| `pomodoro_name`                    | name only, using the existing literal convention |
| `pomodoro_past_repair`             | repair `!` only                                  |
| `pomodoro_close`                   | second `=` only                                  |
| existing close selection/log spans | X groups and log indices                         |

Update `cursor_in_close_log_text` to suppress completion in past inline entries and
child logs, including quoted-name tokens. Name candidate generation and a new completion
context remain phase bob-cli-h2.2's work.

In `src/native/capture_parse.rs`, emit an optional `pomodoro_past` object both at top
level and on each item, with exactly `raw`, `name` (null when unnamed),
`name_is_literal`, `blocks`, `minutes`, `repair`,
`stack { index, size, offset_blocks }`, and `close`. `close` is the existing close-spec
shape, including logs. Do not expose it as the separate `pomodoro_close` field for a
Past item. Use a wire projection if necessary to add `minutes` and exclude
`blocks_typed`.

Add a human line such as:

```text
past  x=fix@3!=*3 (FIX · 15m · 2 of 2 · repair · close: parked 3 · defer the rest)
```

Reuse the existing close formatter. Extend `refusal_diagnostic` so templates cannot
drive past sessions. All text-level parse responses remain independent of BOB_NOW and
the vault.

### 4. Integrate kind restrictions and the temporary execution refusal

Audit all exhaustive `CaptureKind`, `EqualsToken`, and `EditorMode` matches and all
non-exhaustive lists of session kinds; compilation alone misses those lists. Known sites
include:

- `capture_language/type_marker.rs`: `TypeMarkerSubject::for_kind`.
- `capture_language/dependencies.rs`: ownership and operator dependency refusals.
- `capture_language/draft.rs`: `is_template_session_kind`.
- Clip incompatibility, forced route/section restrictions, and global-destination
  handling: match the established session policy, never turn Past into a task or absorb
  `@@` routing into it.
- `capture_language/rewrite.rs` and `capture_task_wikilinks.rs`: Past uses no task
  rewrite or task-wikilink interpretation.
- `capture/batch.rs`, `capture/relations_merge.rs`, and `capture/plan.rs`: session
  restrictions, line formatting, and `capture_kind_label = "pomodoro_past"`.

Put the temporary usage refusal in `plan_capture_item` before any task fallback or
forced-target transformation. A parsed valid Past remains a session kind but cannot
mutate the vault. Later phase bob-cli-h2.3 replaces this branch; do not introduce ledger
placeholders or partial recording behavior to silence matches.

### 5. Publish grammar 12 and generated help

Set `capture_language::GRAMMAR_REVISION` to 12; keep JSON `schema_version` at 1. Update
current grammar-11 assertions/fixtures and explicit help version strings in
`capture_parse.rs` and `capture_complete/cli.rs`. Search narrowly for current grammar
assertions rather than replacing unrelated 11s or historical prose.

Read `docs.md` through `/sase_memory_read` before editing help. Extend capture-parse
long help with the new mode, object, spans, and teaching errors, and add one recognition
sentence to `capture/cli.rs` stating that recording is not available yet. Run
`just docs-gen` and include resulting help goldens and `docs/reference/cli/*` changes.
Comprehensive user guides, JSON reference prose, compatibility history, and Mac UI
remain later epic phases.

## Verification

Add `capture_language/tests/past.rs` and register it. Extend existing `chain.rs`,
`bare_close.rs`, `editor_modes.rs`, and `editor_spans.rs` where those tests give the
clearest regression evidence. Cover:

- Every valid, malformed, and unfinished example above, with editor/execution semantic
  agreement and accurate diagnostic ranges.
- Default and explicit duration, zero/overflow counts, both FIX spellings,
  bare/hash/straight-quote/curly-quote names, and every name terminator. Quoted
  delimiters must not become operators even when the decoded name is invalid.
- Selection permutations, wildcard `!` versus repair `!`, `@N`, duplicate-list failures,
  and preservation of the close lexer's incomplete states.
- Prose guards, leading whitespace, absolute byte offsets in non-first items, quoted
  literals in first/middle/last chain positions, and unterminated literals.
- Compatibility breaks: `x=3` becomes Past; `x x=3` becomes two items; `=3 x=3` becomes
  a Start followed by Past.
- Ownership: `x x=3! entry`, `x=3 =`, `x=build@2 x=fix@3 entry`, child bullets on the
  last `x=`, and ordinary closes mixed with Past. Test positional, numbered, star,
  nested, inline, and mixed-log refusals; marker-looking Work Log content stays literal.
- Stacks on one line and across blank-line items. `x=build@2 x=fix@3` has
  `(index,size,offset_blocks)` values `(1,2,3)` and `(2,2,0)`. Starts, closes,
  adjustments, tasks, and invalid/incomplete items separate runs.
- Session restrictions for templates, clips, routing, type/dependency modifiers,
  rewrite, and suppression of completion inside logs.

Add `tests/cli/capture/parse_pomodoro_past.rs` and register it in the test module.
Assert exact JSON object keys/values, absence on older kinds, first-item projection,
`items[]`, name/repair/close spans, needs/diagnostics, human formatting, grammar 12, and
unchanged schema 1. With isolated fixtures, verify valid capture and dry-run refusal,
malformed usage errors, and byte-identical rollback of a mixed batch whose earlier item
would write. No live vault is needed.

Run focused language and CLI tests during development, then regenerate help and run the
required gates:

```sh
cargo test --lib capture_language
cargo test --test cli capture::parse_pomodoro_past
just docs-gen
just docs-check
just check
```

Use `/sase_monitor` if a required command needs a long-running handoff; never end the
provider turn with an unfinished subprocess. Fix regressions caused by this work. For a
failure reproduced identically on the clean base, append a `PROPOSED FOLLOW-UP:` note on
bob-cli-h2.1 with both commands/results and any existing tracking bead; the user's
instructions allow closing with that evidence. Do not create follow-up beads or close
the parent epic.

## Completion

Record verification on bob-cli-h2.1, including that the execution refusal is intentional
and removed by phase bob-cli-h2.3. Before closing, rerun
`sase bead epic-symbols bob-cli-h2.1`; resolve entries or re-key genuinely deferred
symbols to a still-open later phase or the parent. Close **only bob-cli-h2.1** via
`sase bead close bob-cli-h2.1 --note "<verified contract and check results>"`. Any
child-plan or ancestor closure belongs to its land agent. Finish through `/sase_final`
with the breaking-change commit subject after verification.
