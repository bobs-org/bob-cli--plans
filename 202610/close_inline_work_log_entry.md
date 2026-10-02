---
tier: tale
title: One Work Log entry on the =x close line
goal:
  A single Work Log entry can be typed on the close line itself (`=x foo bar baz`,
  `=x1,3,4 3 boom`), executing byte for byte like its bullet form, in both bob capture
  and Bob Mac Capture, while several entries keep using sub-bullets.
size: medium
proposed_by: bbugyi200.athena.0vc
create_time: 2026-10-02 10:05:31
status: wip
---

# Plan: One Work Log entry on the `=x` close line

## Context

Today a Pomodoro close takes Work Log entries only as child bullets:

```text
=x
- 1 foo bar baz

=x1,3,4
- 3 boom
```

Bryan wants the common single-entry case to be one line:

```text
=x foo bar baz

=x1,3,4 3 boom
```

When the task number is omitted, the entry logs to the first task the close works. That
is task 1 for a plain `=x`. Several entries still need bullets.

**Token note.** Bryan's request wrote `=1,3,4` and mentioned `=*`. The shipped close
token is `=x[<N>][*<P>][!<M>][~<K>]`, and `*<P>` is its park list. `=1,3,4` is a
malformed start, because `=<X>` gives a start's duration, and there is no `=*` token.
This plan therefore covers every whole-item close token (`=x`, `=x1,3,4`, `=x*2`,
`=x1!2~3`, …). Starts (`=`, `=<X>`, `=#name`) take no Work Log, as today.

**History; read it before changing anything.** Epic `bob-cli-2z`
(`plan:202609/close_work_log_entries.md`) shipped a multi-entry inline tail. Plan
`plan:202609/close_work_log_bullets.md` (bead `bob-cli-32`) retired it in favor of
bullets and named five problems. This design keeps bullets as the multi-entry form and
answers each problem for the single-entry case:

| Retired-tail problem           | Answer here                                                                                                                                                                 |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Numbers collide with indexes   | Only the entry's first token can be a task number, which is the bullet rule. With one entry per line, a later number never starts a new entry.                              |
| Endings collide with operators | One fixed rule, with no guessing: a trailing run of session operators that begins with a start or close token runs after the close. Trailing adjust/shift tokens stay text. |
| Many entries read poorly       | The line holds exactly one entry. Several entries use bullets, and mixing the two forms is an error that teaches the bullet form.                                           |
| No detail lines                | Details need bullets; the same mixing error teaches the move.                                                                                                               |
| Breaks the item model          | The entry is the close line's own text, the same way every task item carries its text on its parent line.                                                                   |

**Guiding principle.** An inline entry is exactly the bullet `- <n> <text>` written
below its close. `=x<sel> [<n>] <text>` must execute byte for byte like `=x<sel>` +
`- <n'> <text>`, where `<n'>` is the typed or default task number. No new Work Log
writer is added. The engine path is reused unchanged: `PomodoroCloseSpec.log` →
`insert_close_log_entry` → `ClosePlanner::write_logs` → `tasks[].typed_work_log`. Plain
`=x`, every selection, every bullet draft, and every all-operator chain stay
byte-identical.

**Governing records** (read with `/sase_memory_read`, for example
`sase memory read decisions:mac-capture-is-a-thin-client -r "<why>"`):

- `mac-capture-is-a-thin-client`: grammar, previews, and vault writes land in bob-cli
  first; Bob Mac Capture only decodes and presents. JSON grows additively, and this plan
  adds no JSON fields.
- `task-lanes-are-sticky`: no lane rule changes; close outcomes are untouched.

**Glossary:** `Pomodoro`, `Task Link`, `Work Log`
(`sase memory read glossary:Pomodoro "Task Link" "Work Log" -r "<why>"`). Read
`cli_rules.md` before editing any `--help` text.

## Design

### Shape

```text
[<operators…>] =x[<N>][*<P>][!<M>][~<K>] [<n>] <entry text…> [<start-or-close> [<operators…>]]
```

### The entry

- **What it is.** The text after the close token on its line is one Work Log entry,
  whitespace-normalized like every capture line.
- **Task number.** If the entry's first token is a plain positive integer (ASCII digits,
  no leading zero), it names the task and the rest of the entry is the text. Only that
  one token counts:
  - `=x 3 bugs fixed` logs `bugs fixed` to task 3;
  - `=x 1 3 bugs fixed` logs `3 bugs fixed` to task 1, which is the escape for text that
    starts with a number;
  - `=x 1 wrote docs 2 opened the PR` logs one entry to task 1.

  `0` and leading-zero tokens report the existing `task numbers start at 1` error, and
  overflow reports `task number N is too large`.

- **Default task number.** With no number, the entry logs to the **first task the close
  works**: the smallest number that may take an entry, under today's lexical loggability
  rule (`close_log.rs::is_loggable`):
  - `<N>` typed (including `=x0`) or `*<P>` present → the smallest of `<N>` ∪ `*<P>` ∪
    `!<M>`;
  - otherwise → the smallest `n ≥ 1` not in `~<K>`.

  | Close          | Default | Why                                                    |
  | -------------- | ------- | ------------------------------------------------------ |
  | `=x`           | 1       | Bryan's "assume 1"                                     |
  | `=x1,3,4`      | 1       | the first listed task                                  |
  | `=x2`          | 2       | 1 is deferred by the selection, so 1 would always fail |
  | `=x3,4`        | 3       |                                                        |
  | `=x*2`         | 2       | parked tasks are worked                                |
  | `=x0!2`        | 2       |                                                        |
  | `=x~1`         | 2       | 1 is dropped                                           |
  | `=x!2`         | 1       | link 1 keeps its ledger default (in progress)          |
  | `=x0`, `=x0~2` | none    | error: the close works no task                         |

  Whenever task 1 can take an entry, the default is 1. In every case where a default of
  1 would always fail, the default moves to the first task that can. Implement this as
  one helper (for example `default_log_index(...) -> Option<u32>` in `close_log.rs`).
  Also use it for the bullet missing-number example, which today wrongly suggests
  `- 1 …` under `=x~1`.

  **Rejected:** resolving the default at execution against the ledger, as "the first
  link whose outcome is worked". That would avoid a hand-deferred `[[…]]#` link 1, but
  `capture-parse` has no vault access and could no longer show the number. It would also
  break the documented invariant that loggability is lexical, so `capture-parse` and
  `bob capture` agree. The rare execution failure names a worked task instead (see
  "Execution messages").

- **Literal text.** Entry text follows the bullet rules:
  - `@route`, `@@route`, `s:<N>`, `p:<N>`, `%`, `#`, and `:query` are plain text;
  - plain wikilinks (`[[Design notes]]`, `[[note#Heading]]`) are allowed;
  - the entry never feeds `@@` declarations, item-wide markers, or `capture-rewrite`
    absorption.
- **Rejected text** (shared with bullets): any block link or embed (`[[path#^id]]`,
  `![[path#^id]]`), and text starting with a code fence.
- **Scope.** Only the whole-item `=x` takes an inline entry. Link-form closes
  (`^route:id=x…`, `@route:id=x…`, `<text> @route:id=x…`) keep their current behavior.

### Operators around the entry (same-line chains)

- **Before the close.** Any session operators may precede the close, exactly as in
  today's chains: `-2 =x wired the lexer` shortens, then closes with the entry.
- **After the entry.** A trailing run of session tokens that **begins with a start or
  close token** (`=`, `=<X>`, `=#name`, `=~K`, `=x…`) runs after the close:
  - `=x wired the lexer =` closes with the entry, then starts the next Pomodoro;
  - `=x wired it =#bugs` closes with the entry, then switches to `BUGS`;
  - adjust/shift tokens may follow inside that run (`=x wired it = +2`).

  A trailing `+1`, `-`, or `--` with no start or close before it stays entry text
  (`=x got a +1`). Resizing right after a close has nothing to resize, so those tokens
  are never a plausible chain.

- **Placement.** The entry must directly follow its close. `=x = wired the lexer` is an
  error that teaches `=x wired the lexer =`.
  - **Why this order:** the entry sits next to the close it belongs to, and the line
    reads in action order ("close — wired the lexer — then start").
  - **Rejected:** operators first (`=x = wired the lexer`). It reads like a label for
    the new session, and both orders reserve trailing start and close tokens, so it is
    no more expressive.
- **One entry per line.** A line holds at most one inline entry, and it belongs to the
  close it follows.

**Recognition** (`draft.rs`). For a parent line with whitespace tokens `t`:

1. If every token is a session token and there are at least two tokens, it is today's
   chain, unchanged.
2. Otherwise, let the _lead_ be the maximal prefix of session tokens
   (`is_session_chain_token`).
   - **No whole-item close in the lead** (`is_chain_close_token`): unchanged; the whole
     line is one item, so `-2 wired` and `= foo` behave as today.
   - **Otherwise:**
     - The _owner_ is the last close in the lead.
     - The _trail_ is the maximal suffix of session tokens, trimmed from its front until
       it begins with a start or close token; it is empty if there is none.
     - The _entry_ is every token after the owner and before the trail. It is never
       empty: the token after the lead is not a session token.
     - Emit one item per lead token before the owner, then the owner item, then one item
       per trail token, numbered sequentially as chains are today.
     - The owner item's single `ItemLine` is the contiguous slice from the owner token's
       start through the entry's last token. No sibling token sits inside it, so the
       close parser sees exactly `=x… <entry>`. Line-based lookups (named-start
       completion on `=#bu`, `editor_item_at`) still find each sibling's own token line.
     - When the owner is the only lead token and the trail is empty, the result is one
       item over the whole line, as today.
3. **Children.** When the line has an inline entry, child lines attach to the owner, so
   its parser reports the mixing error below. Otherwise today's rule applies: children
   attach to the line's last `=x`.
4. **Ranges.** The owner's range runs from its token through the entry's end (or through
   its last child). With no children it nests nothing.

Because the entry begins right after the owner, any session tokens between the owner and
the first prose token are the entry's leading tokens. The close lexer judges them by the
first-token rules below:

| Draft               | Diagnosis                          |
| ------------------- | ---------------------------------- |
| `=x = wired`        | reorder error                      |
| `=x - wired`        | stray-marker error                 |
| `=x -2 tests fixed` | valid: entry text `-2 tests fixed` |

Update `session_chain_tokens` / `push_capture_item` and their doc comments. Extract the
lead/owner/entry/trail computation as one helper so the execution and editor paths
cannot disagree.

### Several entries: bullets, never mixed

An inline entry (complete or dangling) plus any non-placeholder child line is an
`invalid_pomodoro_close` error that echoes the resolved number:

`` an inline Work Log entry can't be combined with Work Log bullets; for several entries, put this one on its own bullet too: `- 1 wired the lexer` ``

Placeholder rows stay harmless. The `- ` row that ⌃J leaves is skipped, so the Mac draft
does not flash red until the new bullet has text. Bullet-only drafts are unchanged.

### Lexing order and diagnostics

Add one shared lexer in `close_log.rs`, for example
`lex_close_inline_entry(entry_tokens, close_token, lists) -> Result<CloseInlineLex, CloseLogError>`.
It returns `Entry { index, index_range: Option<_>, text, text_range }` or
`Dangling { index, index_range }`. Both `item.rs` (execution) and `editor_pomodoro.rs`
(`capture-parse`, completion) call it, so the message and byte range are identical
everywhere. Diagnostic text lives in `markers.rs`. Every error uses code
`invalid_pomodoro_close`, writes nothing, and rolls the batch back. Keep the substance
below; polishing the wording is fine.

Check in this order; the first failure wins:

1. **`=x#name`**: existing error, unchanged.
2. **Broken selection token**: its own diagnostic. A dangling separator (`=x1, foo`)
   stays incomplete (`pomodoro_close_task`); the entry gets no spans or diagnostics
   until the selection is complete, matching bullets.
3. **First entry token is a start or close token** (`=x = wired`, `=x =#bugs 2 wired`):
   reorder error that echoes the fixed line, moving the leading operator run to the end:
   `` write the Work Log entry right after `=x`, then the session operators: `=x wired =` ``.
   The range covers the misplaced tokens.
4. **First entry token is a lone `-`, `+`, `*`, `--`, or `++`** (`=x - wired the lexer`;
   joined shell args such as `bob capture '=x' - 2 foo`): stray-marker error:
   `` drop the stray `-` before the Work Log entry: `=x wired the lexer` ``.
5. **No-spaces hint.** This fires when the first entry token is not a plain positive
   integer and the spaceless join `<close token><first token>` lexes as a valid or
   incomplete selection (`=x !2 shipped`, `=x 1,3 foo`, `=x 0`, `=x1 !2`):
   `` write the task numbers right after `=x`, with no spaces (`=x!2`); to log text that starts with `!2`, put the task number first: `=x 1 !2 shipped` ``.
   The escape uses the default number; omit that clause when the entry has no further
   text. A plain-integer first token is always a task number and never triggers this
   hint (`=x 1 !2` logs `!2`), so the explicit-number form is always the escape.
6. **All-digit first token that is not a valid index**: `task numbers start at 1` (`0`,
   leading zeros) or `task number N is too large`.
7. **Explicit number not loggable**: the existing `close_log_not_worked_error` /
   `close_log_dropped_error`, plus an escape clause, because a mistyped leading number
   is the likely cause:
   `` …; to log text that starts with `3`, put the task number first: `=x1 1 3 bugs fixed` ``.
8. **No default** (`=x0 wired it`, `=x0~2 wired it`):
   `` `=x0` works no task, so the Work Log entry has none to log to; list one (`=x1`) or complete one (`=x0!1`) ``.
   Reuse `not_worked_suggestions(1, …)`.
9. **Entry ends with a Task Link form** (`^route:id…` / `@route:id…`, recognized with
   the existing caret and `@route:` link token parsers; for example `=x ^bob:ready=`,
   `=x wired @bob:draft=x`):
   ``a Work Log entry can't end with the Task Link `^bob:ready=`; capture it as its own item after a blank line``.
   Without this check the old switch attempt would silently become Work Log text. Plain
   `@route` stays literal (`=x pinged @alice`).
10. **Block link or fence in the text**: shared with bullets via `check_bullet_text`,
    reworded from "a Work Log bullet …" to "a Work Log entry …" for the inline form.
11. **Mixing** with child bullets (above).
12. **Dangling number** (`=x 2`, `=x1,3 3`, `=x 2 =`): an editing state, never red.
    - `capture-parse` reports mode `incomplete`, `needs: ["pomodoro_close_log_text"]`,
      an `interactive_placeholder` span over the number instead of its index span, and
      the spec with the lists and an empty `log`.
    - `bob capture` rejects it:
      `` `=x 2` is incomplete: type the Work Log text after task 2 ``. When the close
      token is a plain `=x`/`=X`, append
      `, or write `=x2` (no space) to keep only task 2 in progress`, which keeps today's
      spacing hint.

Loggability (7) is still checked before dangling (12), as for bullets. This changes
today's behavior: `=x 1` used to fail with the no-spaces hint and is now this dangling
state with the `=x1` hint in its message.

### Execution messages (origin-aware)

`CloseLogEntry` gains an internal origin, for example
`#[serde(skip)] origin: CloseLogOrigin { Bullet, Inline { head, default_index } }`. JSON
is unchanged. `CloseSelectionError::{LogOutOfRange, LogDeferred, LogNested}` then word
their messages by origin:

- **Bullet:** unchanged (`` `- 5` logs to task 5, but … ``).
- **Inline, explicit number:**
  - out of range quotes the head and adds the escape:
    `` `=x 5` logs to task 5, but CAPTURE has 2 numbered Task Links (1–2); to log text that starts with `5`, put the task number first: `=x 1 5 bugs fixed` ``;
  - deferred and nested keep their current wording.
- **Inline, default number:** say it was the default and name a worked task when the
  lineup has one (the first link with a worked outcome that sits at the session's top
  level):
  - `` the Work Log entry logs to task 1 by default, but task 1 `[[bob#^web-capture]]` is deferred; name a worked task: `=x 2 wired the lexer` ``;
  - `the Work Log entry logs to task 1 by default, but CAPTURE has no numbered Task Links`.

  With no worked link, fall back to the current advice.

### JSON, human output, and `capture-parse`

- **JSON.** `schema_version` stays 1, and no fields are added.
  - `pomodoro_close.raw` stays the close token (`=x1,3,4`), never the entry.
  - `pomodoro_close.log` holds the inline entry with its resolved `index`, exactly as
    the equivalent bullet would.
  - The item `body` is the close token, matching the bullet and chain fixtures
    (`"body":"=x"`).
  - `tasks[].typed_work_log` is unchanged.
- **Human output** is unchanged: typed entries print first, not dimmed. `capture-parse`
  human output keeps `log 1 “wired the lexer”`.
- **Spans.**
  - The close and selection spans are as today.
  - An explicit number gets `pomodoro_close_log_index` on the close line; a default gets
    no index span.
  - Entry text gets no span (neutral prose, as on bullets); wikilinks inside it keep
    their wikilink spans.
  - Chain sibling spans are unchanged.
  - Spans stay sorted and non-overlapping.

### `capture-complete` and `capture-rewrite`

- **Completion.** Inside the inline entry (from its first byte through the line end),
  completion behaves exactly as on a bullet line:
  - suppress every marker completion, including the global `@@` path that runs before
    the item lookup in `completion.rs::completion_field_at`;
  - suppress the `:` and `^` pickers and `wikilink_block` candidates;
  - keep note and heading wikilink completion.

  Generalize `cursor_on_close_bullet_line` (in both `editor_pomodoro.rs` and
  `capture_complete.rs`), for example as `cursor_in_close_log_text`. Named-start
  completion on trail tokens (`=x wired it =#bu|`) keeps working because the owner's
  line ends at the entry.

- **Rewrite.** A bare `@@` inside an inline entry is literal: it is never absorbed and
  never treated as a declaration. Add a test, mirroring bullets.

### Bob Mac Capture (thin client: presentation only)

Editing needs no new machinery:

- the close-line index chip is a `pomodoro_close_log_index` span, which is already cyan;
- `closePendingTrim` already removes a numeric `interactive_placeholder`, so `=x 2`
  trims to `=x` and `=x 2 =` trims to `=x  =`, which bob reads as the chain `=x =`.

Changes:

- `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`:
  - `hintTokens` teaches the inline form in place of the `=x ⌃J 1 wrote the tests`
    example. With one row: `=x` + ` wrote the tests logs work to it`. With two or more
    rows: `=x` + ` ` + `2` (log tint) + ` wrote the tests logs work to 2`. Keep it
    short; the README keeps the bullet flow.
  - Update the doc comment of `pendingLogText(index:)`; it currently says "The inline
    tail is retired". The text stays `Type the Work Log entry for task N`.
- `Sources/BobMacCapture/CapturePanelModel.swift`: update the `closePendingTrim` doc
  comment to cover the inline dangling number; behavior is unchanged.
- `README.md`: lead the Work Log paragraph with `=x wired the lexer` and
  `=x1,3,4 3 fixed the flake`; keep ⌃J bullets for several entries and details; update
  the span and pending notes; state the minimum bob that understands inline entries.
- **Fixtures**, generated from the implemented `bob capture-parse --format json`:
  - `=x wired the lexer`;
  - `=x1,3 3 fixed the flake`;
  - `=x 2` (incomplete);
  - `=x wired it =` (chain).

  Record the generating inputs, as the existing fixtures do.

- **Tests:**
  - decoding, span categories, `log`, and needs;
  - `closePendingTrim` for `=x 2`, `=x1,3 3`, and `=x 2 =`;
  - the updated hint tokens;
  - the unchanged close card fed by an inline-entry dry run.

## Implementation steps

Land bob-cli first, then Bob Mac Capture, in this one tale.

1. **Shared lexer** (`src/native/capture_language/close_log.rs`):
   - `default_log_index`;
   - `lex_close_inline_entry` with the ordered checks above;
   - the escape and reorder suggestion builders;
   - fix the missing-number example to use the default helper;
   - unit tests.
2. **Recognition** (`draft.rs`): the lead/owner/entry/trail split, owner line slicing,
   and children attaching to the owner. Update the doc comments in `draft.rs` and on
   `parse_pomodoro_equals_item` ("only ever sees an exact single-token item").
3. **Execution parser** (`item.rs::parse_pomodoro_equals_item`): replace the "extra text
   → `close_parent_text_error`" path with the shared lexer. Build the spec with
   `log = [entry]` and the origin. Add the mixing check, and the dangling rejection with
   the `=x2` hint. Retire `close_parent_text_error` and `close_tail_bullet_hint` once
   nothing uses them.
4. **Editor parser** (`editor_pomodoro.rs::parse_editor_close_item`): the same
   replacement, with spans, needs, mode, and diagnostic ranges.
5. **Model and engine messages:**
   - `CloseLogEntry.origin` (`model.rs`, `#[serde(skip)]`, defaulting to `Bullet`);
   - origin-aware `Display` for the `Log*` variants in
     `capture_pomodoro_close/selection.rs`;
   - the first-worked-link suggestion;
   - keep `validate_close_log_entries` and the insertion untouched.
6. **Completion and rewrite** (`completion.rs`, `capture_complete.rs`, `rewrite.rs`
   tests).
7. **Help and docs:**
   - **`bob capture --help`** (`src/native/capture/cli.rs`): replace "`=x` takes no text
     on its line…" with the inline rule (default number, leading number, operator
     placement, bullets for several entries). Add examples
     `bob capture '=x wired the lexer'`, `bob capture '=x1,3 3 fixed the flaky test'`,
     and `bob capture '=x wired the lexer ='`. Update the chain sentence.
   - **`bob capture-parse --help`** (`src/native/capture_parse.rs`): the close-line
     index span, the dangling inline number, `body`, and the examples
     `bob capture-parse -f json -- '=x wired it'` and `-- '=x 2'`.
   - **`bob capture-complete --help`** (`src/native/capture_complete.rs`): inline entry
     text completes like a bullet line.
   - **`docs/capture.md`:**
     - Grammar at a glance: the `=x` row; a new inline row; the chain row with
       `=x wired the lexer =`.
     - The `#`/example table: `=x more`, `=x2,3 2 foo bar baz`, and `=x 1` now succeed
       or are incomplete, and the new error rows are added.
     - "Closing the running Pomodoro": the extra-text sentence.
     - "Logging work while closing": lead with a "One entry on the close line"
       subsection, keep bullets for several entries and details, and replace "The inline
       tail is retired" with the rules, the worked table below, and the diagnostics.
     - "Chaining session operators on one line": recognition, the new examples, and the
       replacement for "`=x 1 wired the lexer =` reports the bullet hint" and
       "`-2 =x 1 foo` reports the `-2` shape error".
     - JSON field notes: "`log` is the typed Work Log entries" (bullets or the inline
       entry).
     - The `capture-parse` and `capture-complete` sections.
   - **`README.md`** grammar table (rows near the `=x[<N>]…` entry, which still says
     "the item must contain only the token").
8. **Bob Mac Capture**: open it with `/sase_repo`
   (`sase repo open bob-mac-capture -r "<why>"`), read its `AGENTS.md`, and make the
   changes above.

### Worked table (docs fixture)

The running `CAPTURE` session has link 1 = plain `[[bob#^capture-stop]]` with two note
children and link 2 = deferred `[[bob#^web-capture]]#`. "≡" means byte-identical files
and identical `pomodoro_close` JSON.

| Draft                                                       | Result                                                                             |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `=x wired the lexer`                                        | ≡ `=x⏎- 1 wired the lexer`                                                         |
| `=x2 sketched the URL parser`                               | ≡ `=x2⏎- 2 sketched the URL parser`: 1 deferred, 2 in progress, entry under link 2 |
| `=x1,2 2 sketched the URL parser`                           | ≡ `=x1,2⏎- 2 sketched the URL parser`                                              |
| `=x1!2 2 shipped it`                                        | ≡ `=x1!2⏎- 2 shipped it`: 2 completes with its entry                               |
| `=x 1 3 bugs fixed`                                         | ≡ `=x⏎- 1 3 bugs fixed`                                                            |
| `=x moved @@inbox s:3`, `=x see [[Design notes]]`           | Literal entries; no declaration, no schedule; the wikilink keeps its span          |
| `=x got a +1`                                               | Entry `got a +1`; nothing chained                                                  |
| `=x wired the lexer =`                                      | ≡ `=x =⏎- 1 wired the lexer`: close with the entry, then start the next Pomodoro   |
| `-2 =x wired the lexer`                                     | ≡ `-2 =x⏎- 1 wired the lexer`                                                      |
| `=x wired it = +2`                                          | Close with the entry, start, extend                                                |
| `=x wired the lexer` + `- ` placeholder                     | ≡ `=x wired the lexer`                                                             |
| `=x 2`                                                      | Incomplete (`pomodoro_close_log_text`); `bob capture` rejects it, offering `=x2`   |
| `=x 2 looked at it`                                         | Execution error: link 2 is deferred (existing wording)                             |
| `=x 5 bugs fixed`                                           | Execution error: out of range, plus the `=x 1 5 bugs fixed` escape                 |
| `=x1 2 foo`                                                 | Lexical error: task 2 isn't worked by `=x1`, plus the escape                       |
| `=x~2 2 foo`                                                | Lexical error: task 2 is dropped                                                   |
| `=x0 wired it`                                              | Lexical error: `=x0` works no task                                                 |
| `=x = wired the lexer`                                      | Reorder error, teaching `=x wired the lexer =`                                     |
| `=x - wired the lexer`                                      | Stray `-` error, teaching `=x wired the lexer`                                     |
| `=x !2 shipped it`, `=x 1,2 foo`                            | No-spaces hint, plus the escape                                                    |
| `=x ^bob:ready=`                                            | Task Link error (capture it as its own item)                                       |
| `=x see [[bob#^web-capture]]`                               | Block-link error                                                                   |
| `=x wired the lexer` + `- 2 more`                           | Mixing error, echoing `- 1 wired the lexer`                                        |
| `=x`, `=x2`, `=x =`, bullet drafts, `^bob:ready=x wired it` | Unchanged                                                                          |

## Tests

Use temporary fixture vaults and a fixed `BOB_NOW`; never the real vault.

1. **Lexer** (`close_log.rs` unit tests):
   - default numbers for every close shape in the table above;
   - leading-number precedence and the `=x 1 3 bugs fixed` escape;
   - literal markers and wikilinks;
   - all twelve diagnostics with exact byte ranges, including Unicode text and tabs;
   - dangling numbers.
2. **Grammar and editor parity**
   (`capture_language/tests/{grammar,editor_modes,chain,completion,rewrite}.rs`):
   - modes, spans, needs, `body`, and specs for each worked-table row;
   - chain items and ranges for `-2 =x wired`, `=x wired =`, `=x wired it =#bugs`,
     `= =x wired`, `=x = =x wired`, and `=x wired =x`;
   - children attaching to the owner;
   - named-start completion on a trail token;
   - completion suppression inside the entry (`@`, `@@`, `:`, `^`, block links) while
     note and heading wikilinks still complete;
   - a bare `@@` in an entry is not absorbed and does not route a sibling item.
3. **Execution**
   (`tests/cli/capture/{pomodoro_close_log,pomodoro_chain,parse_pomodoro_close,pomodoro_close}.rs`):
   - the **equivalence invariant**: each "≡" row writes byte-identical daily-note and
     task-note files and the same `pomodoro_close` JSON as its bullet form, on writes
     and on `--dry-run`, with tabs or spaces, CRLF, and no final newline;
   - origin-aware execution errors (default deferred, which names link 2 only when it is
     worked; out of range with no links; explicit out of range with the escape);
   - batch rollback when a later item fails;
   - mixing.
4. **Update every existing assertion of the retired "takes no text on its line" errors**
   (grep `takes no text`, `close_parent_text_error`, `close_tail_bullet_hint` in `src/`
   and `tests/`) to the new behavior.
5. **Mac:** the fixtures and tests listed above.

## Verification

- Run focused Rust tests while developing, then `just all` in bob-cli (fmt, clippy for
  all targets, tests). It must pass.
- Smoke-test the CLI against a temporary vault. Use `bob capture -d` and
  `bob capture-parse` for `=x wired the lexer`, `=x1,3 3 fixed it`, and
  `=x wired the lexer =`, and paste the output into the commit or final report.
- **Mac:** run `just all` on macOS when available. Agent hosts usually lack a Swift
  toolchain; in that case push and watch the repo's `.github/workflows/ci.yml` run to
  completion through `/sase_monitor`. Never report Linux-only inspection as passing
  Swift validation, or claim an unobserved CI result. The close card's visuals are
  unchanged, so no new render-harness images are required.

## Non-goals

- Multiple inline entries, or inline details.
- Inline entries on starts or on link-form closes.
- A ⌃J auto-promotion that rewrites `=x wired it` into `=x⏎- 1 wired it⏎- `. ⌃J is a
  native, grammar-free text edit in the Mac app, and promoting needs bob's default
  number, so it would need a new `capture-rewrite` rule. Leave it as a possible
  follow-up; the mixing error already teaches the exact bullet.
- Any JSON schema change, memory or decision-record edit, deployment, or live-vault
  write.

## Completion criteria

- Bryan's examples work in the CLI and the app:
  - `=x foo bar baz` logs to task 1;
  - `=x1,3,4 3 boom` logs to task 3;
  - each is byte-identical to its bullet form.
- `=x wired the lexer =` closes with the entry and starts the next session in one line.
- Every near miss in the worked table fails or pends with the diagnostics above, writes
  nothing, and agrees between `bob capture` and `capture-parse`.
- Plain `=x`, every selection, every bullet draft, and every all-operator chain behave
  exactly as before.
- Help, `docs/capture.md`, `README.md`, and the Mac README describe the inline form
  first and bullets for several entries.
- `just all` passes in bob-cli, and the Mac validation result (macOS `just all` or an
  observed CI run) is recorded honestly.
