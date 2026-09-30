---
tier: epic
title: Work Log entries on the =x Pomodoro close
goal: '`bob capture ''=x2,3 2 wired the lexer''` closes the running Pomodoro with
  tasks 2 and 3 in progress and first adds `wired the lexer` as a sub-bullet under
  Task Link 2, so the unchanged close writes it to that task''s Work Log. Bob Mac
  Capture highlights, previews, and submits the same drafts, and every mistake is
  caught loudly before anything is written.

  '
phases:
- id: engine
  title: Close planner inserts typed Work Log entries and reports them
  depends_on: []
  size: medium
  description: 'engine: add the `log` entries to the close spec and CloseSelection.
    Validate each target, append the entry sub-bullets under their numbered links
    before the unchanged close runs, and report `pomodoro_close.log` plus `tasks[].typed_work_log`
    in JSON and human output. Grammar comes in the next phase, so the tests build
    specs directly.

    '
- id: grammar
  title: Lex, parse, chain, and document the =x Work Log tail
  depends_on:
  - engine
  size: medium
  description: 'grammar: add one shared tail lexer for `bob capture` and `capture-parse`
    (index spans, the `pomodoro_close_log_text` editing state, precise diagnostics,
    `\` escapes). Add chain splitting for leading operators and trailing starts, help
    text, docs/capture.md, and CLI integration tests.

    '
- id: mac
  title: Bob Mac Capture highlights, previews, and submits close Work Log entries
  depends_on:
  - grammar
  size: medium
  description: 'mac: decode `log` and `typed_work_log`, color index chips, and add
    a pending state for a dangling index. Show typed entries on the close card, extend
    the teaching hint, and add real-bob fixtures, tests, README updates, and green
    macOS CI.

    '
- id: rollout
  title: Install bob, verify end to end with dry runs, and hand Bryan the Mac steps
  depends_on:
  - mac
  size: small
  description: 'rollout: reinstall bob on this host and (best effort) on the MacBook.
    Verify the grammar against the live vault with dry runs only, never closing a
    real session, and give Bryan the checklist for installing the Mac app.'
proposed_by: bbugyi200.athena.0ui
create_time: 2026-09-30 18:46:55
status: wip
bead_id: bob-cli-2z
---

- **PROMPT:** [prompts/202609/close_work_log_entries.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/close_work_log_entries.md)
- **BEAD:** [bob-cli-2z](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2z/README.md)

# Plan: Work Log entries on the `=x` Pomodoro close

## Context

Today a Pomodoro close is `=x[<N>][!<M>][~<K>]` (docs/capture.md, "Closing the running
Pomodoro" and "Choosing each Task Link's outcome"):

- `<N>` keeps only those numbered Task Links in progress.
- `!<M>` completes links.
- `~<K>` drops links.

The close already turns the descendants of every direct-child Task Link in the running
session into dated Work Log entries on that link's task (`*YYYY-MM-DD* — text` under
`🛠️ **WORK LOG**`, newest close on top). Writing a Work Log therefore means opening the
daily note and typing sub-bullets under the right link by hand before closing.

Bryan's request: `=x<N> <n> <text> <m> <text> …`, where each `<n>` names a Task Link
that this close works. With `<N>` typed, `<n>` must come from `<N>`; with no `<N>`, any
numbered link is allowed. `=x2,3 2 foo bar baz` works links 2 and 3 as usual and first
adds the sub-bullet `foo bar baz` under link 2. The close then writes that sub-bullet to
task 2's Work Log.

Governing records (read them with `sase memory read decisions:<keyword>`):

- **`mac-capture-is-a-thin-client`.** All grammar, previews, and vault writes land in
  bob-cli first. Bob Mac Capture only decodes and presents, and new JSON fields are
  additive.
- **`task-lanes-are-sticky`.** An `=x` in-progress outcome sets `[/]`. This plan changes
  no lane rule.

Glossary: `Pomodoro`, `Task Link`, `Work Log` (`sase memory read glossary:…`).

**Guiding principle (inherited from the selection design).** A selection is nothing but
the marker edits the user would make by hand, followed by the unchanged close. A Work
Log entry works the same way: it is the sub-bullet the user would have typed under that
link by hand, followed by the unchanged close. No new Work Log writer exists; the
existing descendant-to-Work-Log path does all the writing. Plain `=x` and every existing
selection stay byte-identical.

## Design

### Grammar

```text
=x[<N>][!<M>][~<K>] <n> <text…> [<m> <text…>]…
```

- **The tail.** Everything after the whitespace that follows the close token is the Work
  Log tail. It runs to the end of the physical line, minus a trailing start run (see
  "Chains" below). `=X` works too.
- **Entry index.** An entry starts at a whitespace-separated token made only of ASCII
  digits that names a _loggable_ task:
  - With `<N>` typed (including `=x0`), the loggable tasks are the numbers in `<N>` or
    `!<M>`: the tasks this close works or finishes.
  - With no `<N>`, every number ≥ 1 is loggable except those in `~<K>`.

  Every other token is entry text, including numbers that are not loggable, `0`, and
  leading-zero numbers. So `=x1 1 fixed 3 bugs` logs `fixed 3 bugs`.

- **Loggability is lexical.** It is decided from the typed token alone, with no vault
  access. That keeps `capture-parse` (which is purely lexical) and `bob capture` in
  exact agreement. Whether a task number is in range, and whether the target is actually
  worked, is checked at execution; see "Execution".
- **Escape.** A tail token that starts with `\` followed by a digit or `=` loses that
  backslash and is always text: `\3` writes `3`, `\=` writes `=`. Every other backslash
  is literal.
- **Order and repetition.** The first tail token must be a loggable index. Each entry
  needs at least one text token. The same index may appear again; each occurrence is a
  separate sub-bullet, in typed order.
- **Literal text.** Entry text is literal and whitespace-normalized like every capture
  line: `@route`, `@@route`, `s:<N>`, `p:<N>`, `%`, `#`, and `:query` are not markers
  here. Plain wikilinks (`[[Design notes]]`, `[[note#Heading]]`) are allowed.
- **Rejected text.** Two things are rejected because the close would misread them in the
  ledger:
  - Any block link or embed (`[[path#^id]]`, `![[path#^id]]`). The close would number,
    start, carry, or complete it as a Task Link. Detect these with
    `capture_pomodoro_close::links::wikilink_tokens`; do not write a second recognizer.
  - Text that starts with a code fence (` ``` ` or `~~~`).
- **Chains** (docs/capture.md, "Chaining session operators on one line"):
  - **Leading operators.** Session operators before a close with a tail run first:
    `-2 =x 1 wired the lexer`.
  - **Trailing start run.** After the first entry's text, the longest trailing run of
    session tokens that begins with a start token (`=`, `=<X>`, `=#name`, `=~K`, and so
    on) runs after the close. `=x2 2 wired the lexer =` is the one-line "log it and
    switch sessions" idiom.
  - A trailing run that does not begin with a start token stays text:
    `=x 1 bumped the version +1` logs `bumped the version +1`.
  - A line whose tokens are all session tokens keeps today's chain rules unchanged.
  - A line whose first token is prose stays prose (`Plan =x 1 foo`).
- **Scope.** Only the whole-item `=x` takes a tail. Link-form closes (`@route:id=x…`,
  `^route:id=x…`, `<text> @route:id=x…`) and close items with child lines keep rejecting
  extra text. See "Deliberately not doing".

Worked table, using the docs fixture (CAPTURE session: link 1 = plain
`[[bob#^capture-stop]]` with two note children, link 2 = deferred
`[[bob#^web-capture]]#`):

| Draft                             | Result                                                                                                                            |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `=x 1 wired the lexer`            | Ledger outcomes. `- wired the lexer` is appended under link 1 after `Wrote the plan`; `^capture-stop` gets three Work Log entries |
| `=x1,2 2 sketched the URL parser` | Both links in progress; entry under link 2                                                                                        |
| `=x1!2 2 shipped it`              | 2 completes, and its Work Log entry is written into the completed task                                                            |
| `=x 1 wrote docs 1 opened the PR` | Two sub-bullets under link 1, in typed order                                                                                      |
| `=x1 1 fixed 3 bugs`              | Entry `fixed 3 bugs` (3 is not loggable under `=x1`)                                                                              |
| `=x 1 fixed 3 bugs`               | Error at execution: task 3 is out of range (2 links); the message suggests `\3`                                                   |
| `=x 1 fixed \3 bugs`              | Entry `fixed 3 bugs`                                                                                                              |
| `=x 2 looked at it`               | Error at execution: link 2 is deferred (`#`), so it takes no Work Log entry                                                       |
| `=x1 2 foo`                       | Lexical error: task 2 isn't worked by `=x1`                                                                                       |
| `=x~2 2 foo`                      | Lexical error: task 2 is dropped by `~2`                                                                                          |
| `=x 1`                            | Editing state (`capture-parse` `incomplete`); `bob capture` rejects it: type the Work Log text after task 1                       |
| `=x 1 wired the lexer =`          | Close with the entry, then start the next Pomodoro                                                                                |
| `-2 =x 1 wired the lexer`         | Shorten by 10 minutes, then close with the entry                                                                                  |
| `=x 1 foo \=`                     | Entry `foo =`                                                                                                                     |
| `=x 1 moved @@inbox`              | Entry `moved @@inbox`; no global destination                                                                                      |
| `=x 1 see [[bob#^web-capture]]`   | Lexical error: block links are not allowed in entries                                                                             |
| `=x more`, `=x ^bob:ready=`       | Error: the tail must start with a task number (new wording, below)                                                                |
| `=x 1,3`, `=x1 !2`                | Unchanged "no spaces" hint (the first tail token is not a bare number)                                                            |
| `Plan =x 1 foo`                   | Prose task, unchanged                                                                                                             |

Proposed diagnostic wording. Keep the substance; polish is fine. All use
`invalid_pomodoro_close`.

- **Tail does not start with a task number:** "after `=x`, write a task number and then
  its Work Log text (for example `=x 1 wrote the tests`); to link a task while closing,
  use `^route:block-id=x`". The no-spaces hint still wins when the spaceless join forms
  a selection and the first tail token is not a bare number.
- **Child lines:** "`=x` takes no child lines; write Work Log entries on its line (for
  example `=x 1 wrote the tests`)". Replace `POMODORO_CLOSE_SHAPE_ERROR`'s "must be the
  whole capture item", which is no longer true, including the defensive copy in
  `reject_pomodoro_close_conflicts`.
- **Not loggable:**
  - "task 3 isn't worked by `=x2`; list it (`=x2,3`) or complete it (`=x2!3`) to log to
    it".
  - "task 2 is dropped by `~2`, so it can't take a Work Log entry".
  - A first token of `0` or a leading-zero number: "task numbers start at 1".
  - Overflow: the existing "task number N is too large".
- **Empty entry mid-tail:** "type Work Log text after task 2 (write `\3` to log the
  number 3)".
- **Dangling index at execution:** "`=x 1` is incomplete: type the Work Log text after
  task 1".
- **Block link:** "a Work Log entry can't contain the block link `[[bob#^x]]`; the close
  would treat it as a Task Link".
- **Fence:** "a Work Log entry can't start with a code fence".

### Execution (the `engine` phase)

The typed entries travel through the stack like this:

- `PomodoroCloseSpec` (`src/native/capture_language/model.rs`) gains
  `log: Vec<CloseLogEntry { index: u32, text: String }>` in typed order. The text is
  unescaped and final; the field is omitted from JSON when empty.
- `selection_from_spec` (`src/native/capture/pomodoro_close.rs`) returns a
  `CloseSelection` when the spec has lists **or** entries, so plain `=x 1 foo` reaches
  the selection path.
- `CloseSelection` (`src/native/capture_pomodoro_close/selection.rs`) carries the
  entries.

`apply_close_selection` keeps its current steps, then:

1. **Validate every entry against the numbered lineup** (numbers refer to the lineup
   before any insertion):
   - **In range**, 1..=total. With `<N>` typed this is already guaranteed. Error
     message: "`=x 5 …` logs to task 5, but CAPTURE has 2 numbered Task Links (1–2);
     write `\5` to keep the number as text", reusing the owner/range wording of
     `OutOfRange`.
   - **Worked:** the resolved outcome is `InProgress` or `Complete`. A deferred result
     can only happen without `<N>`, on a `[[T]]#` line. Error message: "task 2
     `[[bob#^web-capture]]` is deferred, so it can't take a Work Log entry; list it in
     `<N>` or `!<M>` to log to it".
   - **A direct child of the running entry**
     (`nearest_shallower_list_item_parent(...) == Some(entry line)`). Nested links do
     not get their own Work Log group. Error message: "task 3 `[[…]]` is nested under
     another bullet, so the close can't write its Work Log; move it to the top level of
     CAPTURE".

   Add these as new `CloseSelectionError` variants. They surface exactly like today's
   selection errors: nothing is written, and a batch rolls back.

2. **Insert.** For each entry in typed order, insert `<child indent>- <text>` after the
   target link line's current child block. Use `child_block_end_line`, clamped to the
   running entry's `sub_bullet_range`. Consequences:
   - Hand-typed notes come first.
   - Repeated entries for one link keep typed order.
   - The child indent is the link's first existing direct child's indent. Without one,
     it is the link indent plus one unit, following
     `capture_work_log::child_indent_unit`: a non-empty spaces-only indent doubles,
     anything else adds `\t`.
   - Preserve CRLF and a missing final newline, as the marker rewrite does.
3. **Renumber.** Renumber the post-insertion contents with `number_task_links` and
   return that lineup, so `task_links[].ledger_line` matches the close's pre-image and
   agrees with `tasks[].ledger_line`. As a defensive invariant, the renumbered lineup
   must equal the pre-insertion one apart from line numbers (same count, indices, block
   links, and markers after rewrite). Otherwise fail with an internal "Work Log entry
   changed the Task Link lineup" error.
4. **Return the inserted line numbers** so `ClosePlanner::write_logs` can tell typed
   roots from hand-typed ones. Carry a source line on the ledger's Work Log roots
   (`WorkLogNode` / `collect_descendant_tree` in `ledger.rs`). Keep a per-root mapping
   from `capture_work_log::write_work_log_group`, which skips empty roots, so the right
   dated strings are marked.

The rest of the close is unchanged:

- The sub-bullet stays in the closed session under the 🍅 link as history, exactly like
  a hand-typed note.
- Only the link line is carried.
- An `!<M>` target gets its entry after `apply_embedded_tree`, as today's
  completed-embed path already does.

**Post-condition warning.** When a typed entry does not land in its target row's Work
Log (for example an unresolved link, or a target line without `#task`), warn at top
level, and on the row when it has no other warning: "task 2 `[[bob#^x]]` has no task
line, so its Work Log entry stays only in the Pomodoro". The close still succeeds.

**JSON** (schema version stays 1; additive fields only; plain closes stay
byte-identical):

- `pomodoro_close.log`: `[{ "index": 2, "text": "wired the lexer" }]` in typed order, on
  both `bob capture` and `capture-parse`. Omitted when empty.
- `pomodoro_close.tasks[].typed_work_log`: the dated entries (`*YYYY-MM-DD* — text`, a
  subset of `work_log`) that this capture's typed entries produced, in typed order.
  Omitted when empty.
- `pomodoro_blocks` needs no change. The inserted lines already diff as added lines of
  the closed session.

**Human output** (`print_human_pomodoro_close_success`): typed entries print first, all
of them, not dimmed. Up to two other entries follow, dimmed as today. The `+N Work Log`
count stays the total.

### `capture-parse`, `capture-complete`, and help (the `grammar` phase)

- **Span kind `pomodoro_close_log_index`** covers each entry index token. Entry text
  gets no span, so it renders as neutral prose, but wikilinks inside it keep their
  wikilink spans. Spans stay sorted and non-overlapping; the Mac drops the whole set
  otherwise.
- **Editing state.** A dangling index (the last index has no text yet) reports:
  - mode `incomplete`;
  - `needs: ["pomodoro_close_log_text"]`;
  - one `interactive_placeholder` span over that index token, _instead of_ its index
    span;
  - a spec holding the lists plus every complete entry.

  Check that `capture_block_ids::detect_colon_intent` does not treat the new need as
  link intent.

- **Chains** report one `items[]` entry per operator; the close item's range covers the
  close token and its tail. Split in `draft.rs` (`session_chain_tokens` /
  `push_capture_item`) using the existing `is_session_chain_token` and
  `session_equals_token`. The split has two cases:
  - **Close with a tail.** The close is the first close token that comes after only
    session tokens and is followed by a non-session token. Leading tokens become their
    own items.
  - **Trailing start run.** When present, it splits into per-token items.
- **One tail lexer.** Add a module such as `src/native/capture_language/close_log.rs`,
  shared by `parse_pomodoro_equals_item` (`item.rs`) and `parse_editor_close_item`
  (`editor_pomodoro.rs`). It takes the lexed selection lists and returns entries with
  index and text ranges, a dangling-index editing state, or a `CloseSelectionError` with
  a precise byte range. Diagnostic text lives in `markers.rs`, like the selection
  lexer's.
- **Human output.** `format_pomodoro_close` shows the entries, for example
  `log 2 “wired the lexer”`.
- **`capture-complete`.** No new context: the Mac close card already shows the numbered
  lineup. Inside a close tail, `wikilink_block` candidates are suppressed because
  entries reject block links. Note and heading wikilink completion keep working.
- **`bob capture --help`.** Add `bob capture '=x2,3 2 wired the lexer'` and
  `bob capture '=x 1 wired the lexer ='` to the examples, plus any grammar summary line.
  Keep the help sorted, clear, and consistent (`sase memory read cli_rules.md`). Check
  `capture-parse --help` for lists of needs or span kinds.

### Presentation in Bob Mac Capture (the `mac` phase)

- **Editor.** `pomodoro_close_log_index` gets a new `CaptureSemanticCategory` in a color
  not yet in the palette (suggest `.cyan`). It is not a completion span kind. The typed
  entry lines on the close card use the same tint, so the chip you typed visibly lands
  on its row.
- **Pending.** `closePendingTrim` also accepts `pomodoro_close_log_text`. Its single
  placeholder is a task number rather than `,`, `!`, or `~`, and it is trimmed to
  preview the rest. The pending text teaches the escape: "Type the Work Log entry for
  task 3 — or write \3 to keep the number". Submit stays disabled while pending, as for
  dangling separators.
- **Close card.**
  - Each row shows its `typed_work_log` entries first, all of them, never capped, with
    the date stripped, a `square.and.pencil` glyph in the log tint, and primary text.
    The remaining `work_log` entries follow in secondary text, capped at two as today.
  - The accessibility label includes the typed entries.
  - The teaching hint (`hintTokens`) gains an entry-logging example (`=x` pink, `1` in
    the log tint, then " wrote the tests logs work to 1").
- **Models.**
  - `PomodoroCloseSpec.log` and `PomodoroCloseSummary.log` become
    `[PomodoroCloseLogEntry]`.
  - `PomodoroCloseTask.typedWorkLog` becomes `[String]`.
  - All decode with `decodeIfPresent … ?? []`, so an older bob still works. Add the
    README sentence "An older Bob that omits … decodes as …" for each field.

## Rejected alternatives

- **`2:` or `2)` index separators.** Unambiguous, but not the requested syntax. Loggable
  numbers plus the `\3` escape keep `=x2 2 foo` as typed.
- **Vault-aware splitting without `<N>`**, splitting only on numbers that are in range.
  Friendlier for `PR 42`, but `capture-parse` would then disagree with execution, and
  highlighting would change with ledger state. Lexical splitting plus a loud
  out-of-range error that suggests `\42` is predictable.
- **Allowing deferred or dropped targets.** A deferred link's descendants would be
  logged but stranded in the closed session, with the link line removed. A dropped link
  is never logged. Either would be a silent surprise.
- **Restricting `<n>` strictly to `<N>`.** Also allowing `!<M>` is deliberate: finishing
  a task is working it, and `=x!2 2 shipped it` is the natural wrap-up.
- **A dedicated Work Log writer.** It would be a second path to keep in sync with the
  descendant-to-Work-Log writer. Inserting the hand-typed sub-bullet keeps one path.

## Shared rules for every phase

- **bob-cli:** `just all` (fmt, clippy, test) must pass. Commit style is
  `feat(capture): …`. Epic workers record `PROPOSED FOLLOW-UP:` notes on their own phase
  bead instead of creating beads.
- **bob-mac-capture:** open it with `sase repo open bob-mac-capture -r "<why>"`, falling
  back to `gh:bobs-org/bob-mac-capture`. It has no `AGENTS.md`; follow the README's
  Development section and the `justfile`. On Linux only `CaptureCore` and its tests
  build (Swift via swiftly: `PATH="$HOME/.local/share/swiftly/bin:$PATH"`); the app
  target needs macOS. macOS CI (`.github/workflows/ci.yml`) is the gate: push, and the
  phase is done when CI is green for the commit (`gh run list` / `gh run view --log`).
  Regenerate fixtures from a real `bob` built from bob-cli master after `grammar`.
- **No SASE memory changes.** No decision record changes; the lane rules are untouched.
- **Never** close, start, or edit a real Pomodoro in `~/bob` from a test or verification
  step. Use temp vaults, `--dry-run`, or `capture-parse`.

## Phase: engine — close planner inserts typed Work Log entries and reports them

Implements "Execution" above. There is no grammar yet, so build specs and
`CloseSelection` values directly in tests.

1. **Model.**
   - Add `CloseLogEntry` and `PomodoroCloseSpec.log`
     (`#[serde(default, skip_serializing_if = "Vec::is_empty")]`).
   - Add `has_log()`. Keep `has_selection()` meaning lists only.
   - Update `plain()` and the struct literals in `close_selection.rs`
     (`close_spec_from_lex` / `close_spec_from_incomplete`, filled with empty entries
     for now), `selection_tests.rs`, and `linked_task_tests.rs`.
2. **Bridge.** `selection_from_spec` returns `Some` for lists or entries.
   `CloseSelection` gains the entries; prefer a builder or `Default` so call sites stay
   readable.
3. **Planner.** Implement the "Execution" steps 1–4 in `apply_close_selection`:
   - Return a small struct (contents, renumbered lineup, inserted lines) instead of the
     tuple.
   - Add the new error variants and their `Display` text.
   - Thread the inserted lines into `write_logs` to fill a new
     `PomodoroCloseTask.typed_work_log`.
   - Add the post-condition warning.
4. **Output.**
   - `PomodoroCloseSummaryJson.log` and `PomodoroCloseTaskJson.typed_work_log`, both
     skipped when empty.
   - The human renderer shows typed entries first, not dimmed and uncapped, then others
     capped at two.
   - Document both JSON fields in docs/capture.md's field notes for `pomodoro_close`.
5. **Tests** (unit tests next to `selection_tests.rs` / `linked_task_tests.rs`, plus a
   CLI JSON test only if it is reachable without grammar):
   - placement after existing descendants, including grandchildren;
   - tab and space indentation, CRLF, and no final newline;
   - repeated index order;
   - a complete target gets its entry in the completed task;
   - errors: out of range, deferred target without `<N>`, and nested target;
   - `task_links[].ledger_line` is correct after insertion for links below an insert;
   - `typed_work_log` is only the typed subset;
   - the unresolved-target warning;
   - plain `=x` and existing selections stay byte-identical (existing tests stay green).

## Phase: grammar — lex, parse, chain, and document the `=x` Work Log tail

Implements "Grammar" and "`capture-parse`, `capture-complete`, and help" above, on top
of `engine`'s model.

1. **Tail lexer module.**
   - Tokens with absolute ranges and loggability from the lexed lists.
   - `\` unescaping for a digit or `=`.
   - Empty-entry, dangling, block-link, and fence checks.
   - Diagnostics in `markers.rs`.
   - Inline unit tests for every row of the worked table and every diagnostic, including
     exact ranges.
2. **Execution parser** (`item.rs`, `parse_pomodoro_equals_item`):
   - Replace the "extra text" branch with the tail lexer. Keep the `#name` error, the
     no-spaces hint (only when the first tail token is not a bare number), and the
     child-line error with its new wording.
   - Fill `spec.log`.
   - Reject a dangling index with the incomplete message.
3. **Editor parser** (`editor_pomodoro.rs`, `parse_editor_close_item`):
   - Emit spans, the incomplete mode and need, the spec with `log`, and diagnostic
     ranges.
   - Add `Need::PomodoroCloseLogText` and `SpanKind::PomodoroCloseLogIndex` in
     `editor_model.rs`.
   - Make sure the `@@` inheritance skip still covers these items, and that
     `capture-rewrite` never absorbs or declares an `@@` inside a tail.
4. **Chains** in `draft.rs`, as specified.
   - Update `tests/chain.rs`; its non-chain table changes for `=x 1 …` lines.
   - Add execution and editor tests for `-2 =x 1 foo`, `=x2 2 foo =`, `=x2 2 foo = +2`,
     `=x 1 foo +1` (text), and `=x 1 foo \=`.
5. **Completion.** Suppress `wikilink_block` candidates inside a close tail; add a test
   in `capture_complete.rs` next to `close_items_and_suffixes_request_no_completion`.
6. **Existing expectations that change.** Update each with a precise new expectation:
   - `=x 2`, `=x1 1`, and `=x1,3 3` move from the no-spaces hint to the editing state /
     incomplete error (`tests/cli/capture/parse_pomodoro_close.rs` ~250–288,
     `pomodoro_close_selection.rs` ~1128).
   - `=x more` and similar get the new message (`grammar.rs` ~1028, `editor_modes.rs`
     ~1138, `pomodoro_close.rs` ~479, chain tests).
7. **CLI integration tests.** Add a new `tests/cli/capture/pomodoro_close_log.rs` built
   on `close_worked_vault`, with `BOB_NOW="2026-09-28 09:37:00"`:
   - the worked-table executions, with byte-exact day-file and `bob.md` post-images;
   - dry run equals real run (`capture_json_dry_run_matches_real`);
   - `pomodoro_blocks` covers the added lines (`assert_pomodoro_blocks_cover_changes`);
   - a batch rollback on an entry error;
   - `capture-parse` JSON for valid, chain, incomplete, and invalid drafts.
8. **Docs** (docs/capture.md):
   - A new "#### Logging work while closing" subsection after "Choosing each Task Link's
     outcome": grammar, loggability, escape, chains, literal text, rejected text, the
     worked example with before/after ledger and Work Log, human output, and
     diagnostics.
   - Update:
     - the grammar-at-a-glance and `#` tables (`=x` rows, the `=x more` row, new rows);
     - "Closing the running Pomodoro" ("must contain only the close token");
     - the chain section;
     - `capture-parse` (span kind, need, `log`);
     - `capture-complete`;
     - the Contents list.
   - Update `bob capture --help` / `capture-parse --help`.

## Phase: mac — Bob Mac Capture highlights, previews, and submits close Work Log entries

Implements "Presentation in Bob Mac Capture" above.

1. **Models** (`CaptureModels.swift`): `PomodoroCloseLogEntry`, `log` on the parse spec
   and the close summary, and `typedWorkLog` on `PomodoroCloseTask`. Each is additive,
   with defaulted init parameters.
2. **Spans and palette:**
   - `pomodoro_close_log_index` in `captureSemanticCategory`
     (`CompletionRowContent.swift`), the new category, and its palette color
     (`CaptureEditorPalette.swift`);
   - keep it out of `completionSpanKinds`;
   - extend `testSpanKindCategoriesCoverExistingCaptureMarkerAndWikilinkKinds`.
3. **Pending state:**
   - `closePendingTrim` and the pending text and action for `pomodoro_close_log_text`;
   - `CapturePomodoroClosePresentation.pendingText` gains a log variant;
   - add tests beside the existing pending-trim tests.
4. **Close card:**
   - `TaskRow` gains the typed previews, excluded from the capped remainder;
   - `closePreviewItem` renders them in the log tint;
   - the accessibility label;
   - the `hintTokens` example.

   Status and notification counts stay totals; check that `NotificationService` still
   reads well.

5. **Fixtures:** regenerate from a real `bob` built from bob-cli master:
   - `pomodoro-close-log.json` (dry-run capture with a typed entry, one line of JSON);
   - `pomodoro-close-parse-log.json`, `pomodoro-close-parse-log-incomplete.json`,
     `pomodoro-close-parse-log-chain.json`;
   - `pomodoro-close-log-invalid.json`;
   - matching `fake-bob` branches.
6. **Tests:**
   - decoding, including an older-bob inline JSON without the fields;
   - presentation (typed entries first and uncapped);
   - panel-model pending trim with the fake bob;
   - highlighting of chips next to wikilinks in entry text.
7. **Finish.** Update the README's runtime-contract, close-card, and syntax sections,
   then run `just format-lint`. Commit and push, and get macOS CI green.

## Phase: rollout — install bob, verify end to end with dry runs, and hand Bryan the Mac steps

1. **Install on the host running this phase** from an up-to-date bob-cli master:
   `cargo install --path . --locked --force`.
2. **MacBook (best effort).** Read `tailnet.md` with `/sase_memory_read` first. Then
   `ssh -o ConnectTimeout=15 mac` and reinstall `bob` the way it was installed (check
   `~/.cargo/.crates2.json`):
   - A clean `path+file://` checkout on master: `git pull --ff-only` plus
     `cargo install --path . --locked --force`.
   - A `git+` install: reinstall from the same source.
   - Anything else, or the Mac is unreachable: record the exact steps and continue.
3. **Verify against the live vault without writing.** Paste the key output into the
   phase notes:
   - `bob capture-parse -f json -- '=x2,3 2 wired the lexer'`: spans, `log`, and mode.
   - `bob capture-parse -f json -- '=x 1'`: incomplete, with the need.
   - `bob capture-parse -f json -- '=x 1 wired it ='`: two items.
   - If a Pomodoro is running today, `bob capture --dry-run -- '=x 1 rollout check'`
     (human) and `-f json`: the row's `typed_work_log` and the closed block's added
     line. If nothing is running, say so and skip it.
4. **Bryan's checklist** (final response, not automated):
   - Rebuild and install Bob Mac Capture from master (`just install`) once `mac` CI is
     green.
   - Try `=x2 2 <what you did>` and `=x 1 <what you did> =` in the panel. Watch the
     index chip color match the card's new entry line.
   - A number in the text that names a loggable task needs `\`: `fixed \3 bugs`.

## Deliberately not doing

- **Entries on link-form closes.** `@route:id=x …` already means "create a task named
  …", and `^route:id=x` numbers the new link last. Link it first (`^route:id`), then
  `=x <n> …`, in a batch or chain.
- **Multi-line entries, child lines under a close, or nested entry children.** One line
  per entry.
- **Clipboard (`%`) as entry text.**
- **A task-number completion list.** The close card's numbered rows already guide the
  choice.
- **Obsidian (bob-plugins) changes.** Ctrl+Enter keeps reading hand-typed sub-bullets.
