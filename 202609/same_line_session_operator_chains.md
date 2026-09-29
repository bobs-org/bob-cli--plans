---
tier: tale
title: Same-line Pomodoro session operator chains
goal:
  Whitespace-separated session operators on one line (for example `+2 =x`) behave
  exactly like the same operators split across blank-line capture items, in bob capture,
  capture-parse, capture-complete, and therefore bob-mac-capture.
size: medium
proposed_by: bbugyi200.apollo.36
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.36](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.36.md)
- **COMMITS:**
  - [dcf7db2](https://github.com/bobs-org/bob-cli/commit/dcf7db203008eb6ff72283e05caab3f9403836dd)
    — feat(capture): support same-line Pomodoro session operator chains

# Plan: Same-line Pomodoro session operator chains (`+2 =x`)

## Goal

Let the six whole-item Pomodoro session operators — `+[N]`, `-[N]`, `++[N]`, `--[N]`,
`=`/`=<X>`, and `=x[<N>][!<M>]` — share one physical line when whitespace separates
them. `+2 =x` must behave exactly like the blank-line batch `+2⏎⏎=x` everywhere:

- `bob capture`: execution, `--dry-run`, JSON, human output, and rollback.
- `bob capture-parse`: modes, spans, diagnostics, and `items[]`.
- `bob capture-complete`.
- bob-mac-capture, which hands grammar, preview, and writes to those commands.

## Background (current behavior)

- `src/native/capture_language/draft.rs`: `split_capture_draft` →
  `split_items_from_item_lines` → `push_capture_item` splits a draft into `CaptureItem`s
  on blank lines. Every consumer uses this one splitter:
  - execution: `parse_capture_draft_with_clip_control` → `capture/plan.rs`
    `plan_capture_batch`
  - editor: `parse_for_editor`, `editor_item_at`
  - completion: `completion_field_at`, `project_task_block_id_detail`
  - rewrite: `rewrite_draft`
- `src/native/capture_language/item.rs` owns the whole-item session grammar:
  - lexers `session_operator_token` (`+ - ++ --` plus digits) and `session_equals_token`
    (`=`-family: `Close` or `Start{suffix,counted,len}`)
  - execution parsers `parse_pomodoro_equals_item` (runs first) and
    `parse_pomodoro_adjust_item`
- `src/native/capture_language/editor_pomodoro.rs` mirrors these for the editor in
  `parse_editor_close_item` and `parse_editor_adjust_item`. Spans are computed as
  `parent.raw.start + leading`, so they are absolute byte offsets.
- Today every one of these parsers requires the _whole item_ to be the token. So `+2 =x`
  is an adjustment shape error, `=x =` is the close shape error, and bare-first lines
  such as `- -` or `= =` stay prose.
- bob-mac-capture already renders multi-item operator batches. For example, fixture
  `pomodoro-close-batch-adjust.json` is `-2⏎⏎=x`. The app gates its single-item
  presentations on `previewResults.count == 1` and trims dangling close separators using
  per-item `range`s (`closePendingTrim`), so it needs **no Swift change** once Bob emits
  per-token items.

## Design

### 1. Split chains in the draft layer (single choke point)

When an item's parent line is a **session chain**, `push_capture_item` pushes one
`CaptureItem` per token instead of one for the whole item.

Each synthetic item holds a single `ItemLine`:

- `raw: RawLine { text: token.text, start: token.start, end: token.end }`, with absolute
  offsets taken from `tokenize_line_with_spans`
- `line_number` = the physical line number

Item fields:

- `index`: `items.len()`, so numbering stays sequential across the draft.
- `start`/`end`: the token's range.
- `line_start == line_end`: the line number.

Every existing whole-item parser, for both execution and the editor, then sees an exact
single-token item and works **unchanged**. Their spans already land on the correct
absolute bytes because they use `raw.start`.

These consequences need no further code:

- **`bob capture`**:
  - One `CaptureItemResult` per token, planned left to right through
    `CaptureBatchPlanner`, so later tokens see earlier staged edits.
  - The whole batch rolls back on any failure, and `--dry-run` behaves the same way.
  - JSON gets `captures[]` with legacy top-level fields taken from the first token.
    Human output numbers items `1/N`.
  - Errors keep the `capture item K starting on line L: …` prefix.
- **`capture-parse`**:
  - `items[]` has one entry per token. Each `range` is that token, and all entries share
    `line_start`/`line_end`.
  - Top-level legacy fields describe the first token. Modes, spans, specs, and
    diagnostics are per token, exactly as for standalone tokens.
- **Completion**:
  - `completion_field_at` already returns `None` for operator/close items.
  - A cursor in the whitespace between chain tokens matches no item line, so it also
    returns `None`.
  - `editor_item_at` picks the token under the cursor.
- **Rewrite / `@@`**: a chain line can't contain `@@` (it isn't a chain token). Chain
  items have session kinds and modes, which never inherit a declaration.

### 2. Chain recognition

Add `is_session_chain_token(token: &str) -> bool` to `item.rs`, next to the two lexers.
Give it a doc comment that ties it to the two whole-item parsers.

A token qualifies exactly when the whole-item session parsers would **claim** it as a
standalone one-line item. "Claim" means returning `Ok(Some(_))` or `Err(_)`, not
`Ok(None)`:

- `session_operator_token(token) == Some((_, digits, len))` and (`len == token.len()` or
  `!digits.is_empty()`):
  - valid: `+`, `-2`, `++3`, `--`
  - claimed near misses: `+0`, `+5more`, `++3@work`, overflow
- `session_equals_token(token) == Some(Start { counted, len, .. })` and
  (`len == token.len()` or `counted`):
  - valid: `=`, `=-`, `=3`, `=-2`, `=2-1`
  - claimed near misses: `=3x`, an oversized suffix
- `session_equals_token(token) == Some(Close)` and (`token.eq_ignore_ascii_case("=x")`
  or `whole_item_close_after_x(token).is_some()`):
  - valid: `=x`, `=X`, `=x2`, `=x1,3!2`, `=x0`, `=x!2`
  - claimed near misses and editing states: `=x1a`, `=x1,1`, `=x1,`, `=x!`
- Every other token is not a chain token. Examples: `foo`, `==`, `=xx`, `=xa`, `+++`,
  `+-`, `-foo`, `--aside`, `1,3`, `!2`, `^r:id=`, `@r:id=x`, `s:1`, `p:2`, `%`, `#`,
  `@@r`.

Why "claimed" rather than "valid":

- A broken token inside a chain gets its own precise family diagnostic on its own range.
  For example, `+2 +0` gives item 2 the zero-magnitude error on `+0`, instead of a vague
  shape error for the first token.
- It keeps the grammar's rule that "near misses never fall through as ordinary tasks".

Add `session_chain_tokens(line: &RawLine<'a>) -> Option<Vec<Token<'a>>>` to `draft.rs`.
It returns the line's `tokenize_line_with_spans` tokens only when **there are at least
two** and every one passes `is_session_chain_token`.

A single token is never a chain. The existing single-token path must stay
byte-identical.

### 3. Child lines

If a chain's parent line has authored child lines, attach the child lines to the
**last** token's item. Extend its `lines`, and take `end`/`line_end` from the last child
line.

The last token's family parser then reports its existing "exact token with child lines"
shape error:

| Surface   | Error                                                                                                                      |
| --------- | -------------------------------------------------------------------------------------------------------------------------- |
| Execution | `POMODORO_CLOSE_SHAPE_ERROR`, `pomodoro_start_shape_error`, `POMODORO_ADJUST_SHAPE_ERROR`, or `POMODORO_SHIFT_SHAPE_ERROR` |
| Editor    | The matching `invalid_pomodoro_*` diagnostic                                                                               |

This requires no new error constants or diagnostic codes, and a chain line with children
never turns into a prose task. That includes bare-first chains such as `+ =x⏎- child`.

### 4. Semantics to accept and document

- Tokens run left to right exactly like blank-line items:
  - `+2 =x` extends, then closes.
  - `=x =` closes, then starts the next future Pomodoro (a session switch in one line).
  - `=x2 =3` closes keeping task 2 in progress, then starts a 15-minute session.
  - `= +2` starts, then extends.
  - `--2 +` shifts earlier, then extends.
- Spaces are significant:
  - `= -2` starts a 25-minute session and then shortens it by 10 minutes. `=-2` is one
    start with a 10-minute offset.
  - `=x 1,3`, `=x1, 3`, and `=x1 !2` keep today's no-spaces hint, because `1,3`, `3`,
    and `!2` aren't chain tokens.
- A line with any non-chain token is not a chain. Today's behavior stays:
  - `+2 more`, `=x more`, `=3 more`, and `++3 plan` keep their shape errors.
  - `Plan +2 =x` and `- foo` stay prose.
  - `=x ^bob:ready=` keeps the `=x` shape error (see Non-goals).
- **Newly recognized:** lines made only of bare operators, which used to be prose,
  become chains: `- -`, `+ -`, `= =`, `-- --`, `- - -`. This is intentional and
  consistent; document it.
- Runtime guards are unchanged and apply per token, in order:
  - `=x -2` closes and then fails because nothing is running to adjust. The whole batch
    rolls back.
  - `= =` fails on the second start.
- Forced `--route/--section/--task/--task-ref/--task-section/--clip` fail on the first
  chain token with that family's forced error.
- zsh expands a leading `=word`, so docs and examples single-quote chains:
  `bob capture '+2 =x'`. Positional args are already joined with spaces, so
  `bob capture +2 '=x'` also works.

### Non-goals

- No chaining for task/link forms (`^route:id=…`, `@route:id=x…`,
  `<text> @route:id=x…`), markers, `@@`, or prose.
- No JSON contract change:
  - `schema_version` stays 1.
  - No new keys, modes, span kinds, needs, or diagnostic codes.
- No bob-mac-capture edits. Its README sentence "Type `=x`, a blank line, then `=` …"
  stays accurate.
- No SASE memory changes.

## Implementation steps

1. **`src/native/capture_language/item.rs`**:
   - Add `pub(super) fn is_session_chain_token` (claim rule above).
   - Update the doc comments on `parse_pomodoro_adjust_item` and
     `parse_pomodoro_equals_item` to say that a same-line chain has already been split
     into single-token items upstream in `draft.rs`.
   - Make the same one-line note on `parse_editor_adjust_item` and
     `parse_editor_close_item` in `editor_pomodoro.rs`.
2. **`src/native/capture_language/draft.rs`**:
   - Add `session_chain_tokens`.
   - Make `push_capture_item` chain-aware (per-token items, children on the last token).
   - Update the doc comments on `split_capture_draft` and `split_items_from_item_lines`.
   - Leave the test-only `parse_capture_text_with_clip_control` alone. New tests use
     `parse_capture_draft_with_clip_control`.
3. **Help text**:
   - `src/native/capture/cli.rs` `long_about`:
     - In the item-splitting paragraph and the "Pomodoro sessions form a lifecycle"
       paragraph, state the chain rule: `+2 =x` ≡ `+2`, blank line, `=x`, and `=x =`
       switches sessions.
     - Add the examples `bob capture '+2 =x'` and `bob capture '=x ='`.
   - `src/native/capture_parse.rs` `long_about`:
     - Add one sentence: chains yield one `items[]` entry per token, each `range` covers
       its token, and all entries share `line_start`/`line_end`.
     - Add the example `bob capture-parse -f json -- '+2 =x'`.
4. **`docs/capture.md`**:
   - "Grammar at a glance" marker table: add a chain row (`+2 =x`, `=x =`).
   - "Grammar at a glance" examples table: add rows for `+2 =x`, `=x =`, `= -2` vs
     `=-2`, and `- -`. Keep the existing error rows (`++3 more`, `=3 more`, `=x more`);
     they are still errors.
   - "Multi-item capture": add a short paragraph on same-line chains.
   - New subsection `### Chaining session operators on one line`:
     - Place it after "Closing the running Pomodoro" (and its "Choosing each Task Link's
       outcome" subsection), before "Project notes", and add it to Contents.
     - Cover the recognition rule: at least two whitespace-separated tokens, every token
       a session token, near misses included.
     - Cover equivalence to blank-line items, staging, rollback, and dry-run.
     - Cover output: one JSON/human result per token, and `capture-parse` items with
       per-token ranges.
     - Cover child lines: they attach to the last token and fail its shape rule.
     - Cover the spacing gotchas, the newly recognized bare chains, zsh quoting, and the
       non-goals.
   - "Starting the next Pomodoro", "Adjusting", "Shifting", and "Closing":
     - Where each says the item must contain only the token, add "(or share its line
       only with other session operators; see Chaining …)" with an anchor link.
     - Next to the existing blank-line idioms, add `=x =` and `+2 =x`.
     - In the start section's "Recognition" paragraph, note that `t` is the token after
       a chain split.
   - `bob capture-parse` section: in the multi-item `items` paragraph, describe chain
     items (per-token `range`, shared line numbers).
5. **`README.md`**:
   - Add a chain row to the marker table.
   - Mention `bob capture '+2 =x'` / `bob capture '=x ='` in the adjust/close
     paragraphs.
6. **Tests.** Put them in new files so no test file grows past ~1500 lines.
   - **Unit tests: `src/native/capture_language/tests/chain.rs`.** Register it in
     `tests/mod.rs`.
     - **Predicate table**:
       - Assert `is_session_chain_token` over the positives and negatives listed in
         Design §2.
       - **Drift guard:** for every whitespace-free token in the table, build the item
         with `split_capture_draft(t).items[0]`. A single token is never a chain.
       - Assert that the predicate equals "claimed": either
         `parse_pomodoro_equals_item(&item, &item.lines[0], None, None)` or
         `parse_pomodoro_adjust_item(…)` returns something other than `Ok(None)`.
     - **Draft splitting**:
       - `draft_items("+2 =x") == [(0, 1, "+2"), (1, 1, "=x")]`.
       - Tabs and runs of spaces: `"  +2\t =x  "` splits correctly.
       - A multi-item CRLF draft that mixes a chain with ordinary items keeps sequential
         indices and correct `line_start`s.
       - A `@@work` declaration line plus a chain works.
       - `"+2 =x\n- child"` puts the child on item 2 and sets its `line_end` to 2.
     - **Execution** (`parse_capture_draft_with_clip_control`):
       - `"+2 =x"` gives `PomodoroAdjust{plus:true, units:2}`, then `PomodoroClose` with
         the plain spec.
       - `"=x ="` gives a close, then a start.
       - `"= -2"` gives a start (`""`), then an adjust with minus and 2 units. Contrast
         it with `"=-2"`, which is one start with an offset.
       - `"=x1,3!2 +"` carries the selection spec on item 1.
       - `"+2 +0"` returns an `Err` containing `capture item 2 starting on line 1` and
         the zero message.
       - `"+2 =x\n- child"` and `"+ =x\n- child"` both return the close shape error.
       - `"+2 =x"` with `forced_route` returns the adjust forced error.
       - These stay unchanged:
         - `"+2 more"`: adjust shape error
         - `"=x 1,3"`: no-spaces hint
         - `"=x ^bob:ready="`: close shape error
         - `"Plan +2 =x"`, `"- foo"`: tasks
         - single tokens: unchanged
     - **Editor** (`parse_for_editor`):
       - `"+2 =x"`:
         - two items with modes `PomodoroAdjust` and `PomodoroClose`
         - item ranges `(0,2)` and `(3,5)`
         - spans `(0,2,PomodoroAdjust)` and `(3,5,PomodoroClose)`
         - top-level mode `PomodoroAdjust`
       - `"+2 =x1,3!2"`: in-progress/complete spans at absolute offsets.
       - `"+2 =x1,"`: item 2 is `Incomplete`, needs `PomodoroCloseTask`, and has an
         `InteractivePlaceholder` span over the `,`.
       - `"+2 +0"`: item 2 has an `invalid_pomodoro_adjustment` diagnostic with range
         `(3,5)`.
       - `"@@work\n+2 =x"`: the items keep their session modes and don't inherit.
     - **Completion**:
       - `completion_field_at("+2 =x", c)` is `None` for every `c` in `0..=5`.
       - `editor_item_at("+2 =x", 4)` returns the close item.
   - **Integration tests: `tests/cli/capture/pomodoro_chain.rs`.** Register it in
     `tests/cli/capture/mod.rs`, and reuse the existing running-session fixtures and
     helpers from `pomodoro_whole_item.rs` / `pomodoro_close.rs` / `pomodoro_adjust.rs`
     (`close_worked_vault`, `BOB_DAY_FILE`, `BOB_NOW`, `run_with_stdin`; promote a
     helper to `pub(super)` if needed).
     - **Equivalence:**
       - Build two identical temp vaults.
       - Run `bob capture -f json '+2 =x'` on one and stdin `"+2\n\n=x\n"` on the other.
       - Assert byte-identical day files, identical task-note files, and equal
         `captures[].kind`/`pomodoro_adjust`/`pomodoro_close` objects. Paths may differ,
         so compare the relevant sub-objects or normalize paths.
     - `=x =` switches sessions exactly like `"=x\n\n=\n"`.
     - The positional-args form `bob capture +2 '=x'` works.
     - `--dry-run` writes nothing.
     - A failing later token rolls back and writes nothing, with the
       `capture item 2 starting on line 1` prefix. Test both `+2 +0` and `=x -2`.
     - Human output shows `1/2` and `2/2`.
     - `bob capture-parse -f json -- '+2 =x'` reports the right `items` ranges, modes,
       and spans.
     - `bob capture-complete` on a chain returns no candidates.
7. **Verify:**
   - `just all` must pass (`cargo fmt --check`,
     `cargo clippy --all-targets --all-features`, `cargo test`).
   - Before finishing, grep the tests for any existing input that is an all-session
     multi-token line and would change meaning. None is known today.
   - Smoke-test a temp vault by hand with `bob capture --dry-run '+2 =x'` and
     `bob capture-parse -f json -- '+2 =x'`.

## Acceptance criteria

- On a vault with a running timed Pomodoro, `bob capture '+2 =x'` leaves the files
  byte-identical to `printf '+2\n\n=x\n' | bob capture`. Its JSON has two `captures`
  (`pomodoro_adjust`, `pomodoro_close`).
- Every pairwise combination of the six operators works on one line, in order, with the
  same staging and rollback as blank-line batches. So does any longer chain.
- `bob capture-parse` reports one item per token with correct absolute ranges and spans.
  Dangling close separators inside a chain report `incomplete` / `pomodoro_close_task`
  on that token's item, which keeps bob-mac-capture's pending-close trim working.
- Single-token behavior, prose cases, near-miss errors, and the `=x 1,3` no-spaces hint
  are unchanged except for the documented new chains.
- Docs (`docs/capture.md`, `README.md`) and both `--help` texts describe the feature,
  and `just all` passes.
