---
tier: epic
title: Work Log entries as bullets under the =x close
goal: "Work Log entries become child bullets of the `=x` close instead of an inline
  tail:

  `=x2,3` followed by `- 2 foo bar baz` on the next line closes the running Pomodoro

  with tasks 2 and 3 in progress and logs `foo bar baz` to task 2. A two-space

  `  - …` bullet under an entry is a detail that nests under that entry in the task's

  Work Log. The inline tail (`=x2,3 2 foo bar baz`) is retired; it now fails with a

  message that shows the bullet to write. Bob Mac Capture highlights, previews, and

  submits the new drafts, and the close card shows each entry with its details.

  "
phases:
  - id: engine
    title: Close planner writes Work Log details under typed entries and reports them
    depends_on: []
    size: small
    description: 'engine: give each typed close Work Log entry a `details` list. The
      planner

      inserts the details one level under the entry line, so the unchanged close

      nests them under the dated entry in the task''s Work Log. Report the details in

      `pomodoro_close.log[].details` and `tasks[].typed_work_log_details`, and print

      them in human output. Out-of-range errors stop suggesting the retired `\N`

      escape.

      '
  - id: grammar
    title: Parse Work Log bullets under =x, retire the inline tail, and document it
    depends_on:
      - engine
    size: medium
    description: "grammar: replace the inline tail lexer with one shared Work Log bullet
      lexer

      used by `bob capture` and `capture-parse`. It produces index spans, the

      dangling-bullet editing state, and diagnostics. Retire the tail with a message

      that shows the bullet to write. Revert the tail-only chain splitting, and attach

      a chain line's bullets to its `=x`. Silence marker and block-link completion on

      bullet lines, then update help, docs/capture.md, and tests.

      "
  - id: mac
    title: Bob Mac Capture previews Work Log bullets and their details
    depends_on:
      - engine
      - grammar
    size: medium
    description: "mac: decode `details` and `typed_work_log_details`, and show each
      typed entry's

      details under it on the close card. Reword the pending notice (the escape is

      gone), and teach the bullet form in the hint (`=x ⌃J 1 wrote the tests`).

      Regenerate the real-bob fixtures for bullet drafts, update tests and the README,

      and get macOS CI green.

      "
  - id: rollout
    title: Install bob, verify bullet drafts with dry runs, and hand Bryan the Mac steps
    depends_on:
      - grammar
      - mac
    size: small
    description: "rollout: reinstall bob on this host and, best effort, on the MacBook.
      Verify

      bullet drafts against the live vault with `capture-parse` and `--dry-run` only,

      never closing a real session. Give Bryan the checklist for installing the Mac

      app."
proposed_by: bbugyi200.athena.0uj
create_time: 2026-09-30 21:31:27
status: wip
---

- **PROMPT:**
  [prompts/202609/close_work_log_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/close_work_log_bullets.md)

# Plan: Work Log entries as bullets under the `=x` close

## Context

Epic `bob-cli-2z` (plan `202609/close_work_log_entries.md`) shipped an inline Work Log
tail on the whole-item close. `=x2,3 2 wired the lexer` closes the running Pomodoro and
first appends `- wired the lexer` under Task Link 2. The unchanged close then writes
that sub-bullet to task 2's Work Log. It works, but a whole sentence on the close's own
line is awkward:

- **Numbers collide with indexes.** Any number in the text that names a loggable task
  starts a new entry, so you write `fixed \3 bugs`.
- **Endings collide with operators.** A trailing `=` might be text or a chained start,
  and the "trailing start run" rule exists only to guess which.
- **Many entries read poorly.** `=x 1 wrote docs 1 opened the PR 2 sketched it` is one
  long line that is hard to read and to edit.
- **No detail.** An entry cannot carry nested notes, although the close already writes
  nested notes into Work Logs.
- **It breaks the item model.** Every other capture item puts its children on bullet
  lines below the parent line.

Bryan's request: use sub-bullets to specify Work Log entries instead. The draft

```text
=x2,3 2 foo bar baz
```

is now written as

```text
=x2,3
- 2 foo bar baz
```

**Guiding principle (inherited, unchanged).** A Work Log bullet is exactly the
sub-bullet you would have typed by hand under that Task Link in the daily note. The
unchanged close runs after it. No new Work Log writer exists, and plain `=x` plus every
existing selection stay byte-identical. The bullet form mirrors the ledger itself: the
daily note already holds work notes as bullets under each Task Link.

**Governing records** (read with `sase memory read decisions:<keyword>`):

- **`mac-capture-is-a-thin-client`.** All grammar, previews, and vault writes land in
  bob-cli first. Bob Mac Capture only decodes and presents. New JSON fields are
  additive, and the app decodes them with `decodeIfPresent`.
- **`task-lanes-are-sticky`.** This plan changes no lane rule; `=x` outcomes are
  untouched.

**Glossary:** `Pomodoro`, `Task Link`, `Work Log` (`sase memory read glossary:<term>`).

**What already exists and is reused:**

- The authored-bullet line rules (`classify_authored_line`, `strip_bullet_marker_at`,
  `is_marker_only_placeholder` in `src/native/capture_language/draft.rs`):
  - a first-level bullet sits at column 0, and a nested one has exactly two spaces;
  - the marker is `-`, `*`, or `+`, then a space or tab;
  - a marker-only row (`- `, `-`) is a harmless placeholder;
  - a nested bullet with no first-level owner fails with `orphaned_nested_bullet_error`;
  - any other shape fails with `invalid_child_line_error`.
- The engine path from `bob-cli-2z.1`:
  - `PomodoroCloseSpec.log` / `CloseLogEntry` (`capture_language/model.rs`);
  - `CloseSelection::with_log`;
  - `validate_close_log_entries` and `insert_close_log_entry`
    (`capture_pomodoro_close/selection.rs`);
  - typed-root tracking via `inserted_lines` → `ClosePlanner::write_logs` →
    `tasks[].typed_work_log` (`capture_pomodoro_close/linked_tasks.rs`);
  - `warn_missing_typed_logs`.
- The Work Log writer already nests descendants. In `capture_work_log.rs`,
  `append_descendants` writes a grandchild undated under its dated parent entry and does
  not count it as an entry. See
  `worked_example_updates_tasks_and_writes_dated_work_logs` in
  `capture_pomodoro_close/linked_task_tests.rs`.
- Bob Mac Capture's bullet editing:
  - **⌃J** inserts a new line plus `- `, copying a 0- or 2-space indent; from the
    `=x2,3` line it inserts `\n- `.
  - **Tab** and **⇧Tab** move a bullet between column 0 and two spaces.
  - **⌫** on a bare `- ` row deletes the row.
  - Spans are absolute UTF-8 byte offsets, so a chip on line 3 already highlights.

## Design

### The shape

```text
=x[<N>][!<M>][~<K>]
- <n> <entry text…>
  - <detail text…>
- <m> <entry text…>
```

- **Where bullets go.** A whole-item close (`=x`/`=X` plus an optional selection, alone
  on its parent line, or the `=x` of a chain line; see "Chains") takes Work Log bullets
  as its child lines. They use exactly the authored-bullet line rules above. Placeholder
  rows are skipped, which matters because ⌃J leaves a `- ` row while you type. Bad
  indentation and orphaned nested bullets fail with the existing messages.
- **Entry bullet.** A first-level bullet is a Work Log entry. Its first
  whitespace-separated token is the task number `<n>`, and everything after it is the
  entry text. The text is whitespace-normalized like every capture line.
- **Which numbers are loggable** (decided lexically, unchanged from the tail design):
  - with `<N>` typed (including `=x0`), the numbers in `<N>` or `!<M>`;
  - with no `<N>`, every number ≥ 1 except those in `~<K>`.

  Range, deferred, and nesting are still checked at execution.

- **Only the first token is an index.** That makes number handling trivial:
  `- 1 fixed 3 bugs` logs `fixed 3 bugs`, and `- 1 3 bugs fixed` logs `3 bugs fixed`.
  The `\N` / `\=` escape is retired, and every backslash is literal.
- **Detail bullet.** A two-space nested bullet is a detail of the nearest preceding
  entry. It has no task number, so its text is entirely literal; a leading number is
  just text. Details become undated sub-bullets under their dated entry in the task's
  Work Log. They are not separate entries and are not counted in `+N Work Log`.
- **Order and repetition.** Entries keep typed order. The same number may appear on
  several bullets, and each one is a separate entry.
- **Literal text.** Entry and detail text never contain capture markers:
  - `@route`, `@@route`, `s:<N>`, `p:<N>`, `%`, `#`, and `:query` are plain text there;
  - plain wikilinks (`[[Design notes]]`, `[[note#Heading]]`) are allowed;
  - the bullets never feed `@@` declarations, item-wide markers, or `capture-rewrite`
    absorption.
- **Rejected text** (unchanged reasons; the close would misread them in the ledger), in
  entry and detail text alike:
  - any block link or embed (`[[path#^id]]`, `![[path#^id]]`), detected with
    `capture_pomodoro_close::wikilink_tokens`;
  - text that starts with a code fence (` ``` ` or `~~~`).
- **Scope.** Only the whole-item `=x` takes Work Log bullets. Link-form closes keep
  their current child-line behavior:
  - `^route:id=x` and a solo `@route:id=x` reject children;
  - `<text> @route:id=x` writes them as the new task's sub-bullets.

### Retiring the inline tail

Any token after the close token on its parent line is now an error (code
`invalid_pomodoro_close`). The checks run in this order:

1. **`=x#name`** keeps its own error.
2. **A broken selection token** (`=x1a`, `=x1,1`, …) reports its own diagnostic; a
   dangling separator (`=x1,`) stays incomplete.
3. **The no-spaces hint** wins when the spaceless join of the line lexes as a selection:
   `=x 1,3`, `=x1 !2`, and `=x 1` (which now means "did you mean `=x1`?").
4. **Otherwise, the bullet hint.** If the tail starts with a positive number, the hint
   echoes the user's own words as the bullet to write:
   - `=x2,3 2 foo bar baz` →
     `` `=x` takes no text on its line; put each Work Log entry on its own line below it, as a bullet: `- 2 foo bar baz` ``
   - The same holds after a stray leading `-`/`*`/`+` token (`=x - 2 foo`, which is what
     positional shell args produce when they are joined).
   - Any other tail (`=x more`, `=x ^bob:ready=`) gets:
     `` `=x` takes no text on its line; log work as bullets below it (for example `- 1 wrote the tests`); to link a task while closing, use `^route:block-id=x` ``

When the parent line has extra text **and** child lines, the parent-line error wins.
`POMODORO_CLOSE_SHAPE_ERROR` ("takes no child lines…") is no longer reachable from
grammar, so delete it. Reword its defensive copy in `reject_pomodoro_close_conflicts` to
an internal "a close never carries authored sub-bullets" error: close bullets travel in
`spec.log`, never in `sub_bullets`.

### Chains

Revert the tail-only chain splitting added by `c7ce096` in `draft.rs`:
`is_close_tail_opener`, `is_trailing_start_opener`, `close_tail_split`, and the tail
branch of `push_capture_item`. Recognition is then back to "every token is a session
token".

**One new rule.** A chain line's child lines attach to the line's `=x` item, or to the
last `=x` if the line closes twice, instead of the last token's item. That `=x` item's
`end`/`line_end` extend to its last child. With no `=x` on the line, children still
attach to the last token and fail its shape rule.

```text
=x =
- 1 wired the lexer
```

This logs `wired the lexer` to task 1, closes, then starts the next Pomodoro. It
replaces the retired one-line `=x 1 wired the lexer =` idiom. `-2 =x` + bullets
shortens, then closes with the entries. Consequence for `capture-parse`: in a chain
whose `=x` is not last, the close item's range runs from its token through its last
bullet, so it **contains** the ranges of the later tokens on the parent line. Item
ranges therefore nest; they never partially overlap. Document this.

After the revert, a line that mixes session tokens with prose is not a chain. So
`-2 =x 1 foo` reports the `-2` adjust shape error ("remove extra text…"), and
`=x 1 wired the lexer =` reports the bullet hint (`- 1 wired the lexer =`). Both are
acceptable for a one-day-old form, and neither writes anything.

### Worked table

Fixture: the docs fixture. The running CAPTURE session has link 1 = plain
`[[bob#^capture-stop]]` with two note children and link 2 = deferred
`[[bob#^web-capture]]#`. `⏎` marks a line break.

| Draft                                                                                 | Result                                                                                                        |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `=x⏎- 1 wired the lexer`                                                              | `- wired the lexer` appended under link 1 after `Wrote the plan`; `^capture-stop` gets three Work Log entries |
| `=x2,3⏎- 2 foo bar baz` (on a 3-link session)                                         | 2 and 3 in progress; `foo bar baz` logged to task 2 (Bryan's example)                                         |
| `=x1,2⏎- 1 wired the lexer⏎  - chose a hand-rolled lexer⏎- 2 sketched the URL parser` | Both in progress; entry plus nested detail under link 1, entry under link 2                                   |
| `=x1!2⏎- 2 shipped it`                                                                | 2 completes; its entry is written into the completed task                                                     |
| `=x⏎- 1 wrote docs⏎- 1 opened the PR`                                                 | Two entries under link 1, in typed order                                                                      |
| `=x1⏎- 1 fixed 3 bugs`                                                                | Entry `fixed 3 bugs`; no escape needed                                                                        |
| `=x⏎- 1 moved @@inbox s:3`                                                            | Entry `moved @@inbox s:3`; no destination, no schedule                                                        |
| `=x⏎- 1 see [[Design notes]]`                                                         | Allowed; the wikilink keeps its span                                                                          |
| `=x⏎- `                                                                               | Plain `=x`; the placeholder row is ignored                                                                    |
| `=x⏎- 1`                                                                              | Editing state (`capture-parse` `incomplete`); `bob capture` rejects it                                        |
| `=x =⏎- 1 wired the lexer`                                                            | Close with the entry, then start the next Pomodoro                                                            |
| `-2 =x⏎- 1 wired the lexer`                                                           | Shorten by 10 minutes, then close with the entry                                                              |
| `=x⏎- 1 wired the lexer⏎⏎=`                                                           | Batch: close with the entry, then start (the blank line ends the close item)                                  |
| `=x2,3 2 foo bar baz`                                                                 | Error: the bullet hint, echoing `- 2 foo bar baz`                                                             |
| `=x 1`                                                                                | Error: the no-spaces hint (`=x1`)                                                                             |
| `=x more`                                                                             | Error: the generic bullet hint                                                                                |
| `=x⏎- wired the lexer`                                                                | Error: the bullet needs a task number                                                                         |
| `=x1⏎- 2 foo`                                                                         | Error: task 2 isn't worked by `=x1`                                                                           |
| `=x~2⏎- 2 foo`                                                                        | Error: task 2 is dropped by `~2`                                                                              |
| `=x⏎- 2 looked at it`                                                                 | Error at execution: link 2 is deferred                                                                        |
| `=x⏎- 5 foo`                                                                          | Error at execution: task 5 is out of range                                                                    |
| `=x⏎- 1 see [[bob#^web-capture]]`                                                     | Error: block link in a Work Log bullet                                                                        |
| `=x⏎- 1 ok⏎  - ![[bob#^x]]`                                                           | Error: block links are rejected in details too                                                                |
| `=x⏎  - orphan`                                                                       | Existing `orphaned_nested_bullet` error                                                                       |
| `^bob:ready=x⏎- 1 foo`                                                                | Unchanged: link forms reject child bullets                                                                    |

**Expected post-image** for the third row (`BOB_NOW=2026-09-28 09:37:00`, TAB
indentation; the implementer pins exact bytes in tests). The closed session reads:

```markdown
- [x] (**0920-0940** [t:: 20m]) — CAPTURE
  - 🍅 [[bob#^capture-stop]]
    - Designed the `=x` grammar
      - chose `x` for done
    - Wrote the plan
    - wired the lexer
      - chose a hand-rolled lexer
  - 🍅 [[bob#^web-capture]]
    - sketched the URL parser
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
```

`^capture-stop`'s Work Log in `bob.md` ends with:

```markdown
    	- *2026-09-28* — wired the lexer
    		- chose a hand-rolled lexer
```

### Diagnostics

All use code `invalid_pomodoro_close`, write nothing, and roll a batch back. Keep the
substance; polish is fine. Bullet diagnostics identify their bullet by its number or
text, so they need no line numbers. Diagnostic text lives in `markers.rs`.

- **No task number:**
  ``start each Work Log bullet with the number of the task it logs to: `- 2 wired the lexer` ``.
  The example echoes the bullet's own text with the smallest loggable number. That is
  the smallest of `<N>`∪`!<M>` when `<N>` is typed and the union is non-empty; otherwise
  it is `1`.
- **Not loggable:** reuse the current wording and suggestions,
  ``task 3 isn't worked by `=x2`; list it (`=x2,3`) or complete it (`=x2!3`) to log to it``
  and ``task 2 is dropped by `~2`, so it can't take a Work Log entry``.
- **Bad numbers:** `0` and leading-zero numbers give `task numbers start at 1`; overflow
  gives `task number N is too large`.
- **Dangling bullet** (execution only):
  `` `- 1` is incomplete: type the Work Log text after task 1 ``.
- **Block link:**
  ``a Work Log bullet can't contain the block link `[[bob#^x]]`; the close would treat it as a Task Link``.
- **Fence:** `a Work Log bullet can't start with a code fence`.
- **Execution-time planner errors** (`CloseSelectionError`), reworded to quote the
  bullet head instead of the retired tail:
  - **Out of range:**
    `` `- 5` logs to task 5, but CAPTURE has 2 numbered Task Links (1–2) ``. Drop the
    "write `\5`" advice and keep the owner/range wording of `OutOfRange`.
  - **Deferred** and **nested:** unchanged.
- **Precedence.**
  - Parent-line errors outrank bullet errors.
  - Bullets are checked top to bottom, and the first error wins.
  - Any error outranks a dangling bullet.
  - If the selection itself is incomplete (`=x1,`), the item reports that state, and
    bullets get no spans or diagnostics until the selection is complete.

### Editing states and spans (`capture-parse`)

- **Spans.**
  - `pomodoro_close_log_index` covers each entry's number token on its bullet line, at
    absolute byte offsets.
  - Entry and detail text get no capture span and render as neutral prose. Wikilinks
    inside them keep their wikilink spans.
  - Bullet markers get no span.
  - Spans stay sorted and non-overlapping; the Mac drops the whole set otherwise.
- **Dangling bullet.** A bullet whose body is only a number (`- 2`, `- 2 `) is an
  editing state, wherever it sits. A user may be adding a bullet between existing ones,
  so it never flashes red. The item reports:
  - mode `incomplete`;
  - `needs: ["pomodoro_close_log_text"]`;
  - an `interactive_placeholder` span over that number, _instead of_ its index span;
  - a spec that holds the lists plus every complete entry with its details.

  Several dangling bullets give one placeholder each; the Mac pending trim handles
  exactly one and otherwise falls back to a normal preview.

- **Spec.** `pomodoro_close.log` is
  `[{ "index": 2, "text": "wired the lexer", "details": ["chose a hand-rolled lexer"] }]`.
  `details` is omitted when empty. `sub_bullets` / `sub_bullet_depths` stay empty for
  closes: Work Log bullets are not authored task sub-bullets.
- **Human output.** `format_pomodoro_close` shows `log 2 “wired the lexer” (+1 detail)`,
  with `(+N details)` for several and nothing when there are none.

### Execution (`engine` phase)

- **Model.** `CloseLogEntry` gains `details: Vec<String>`
  (`#[serde(default, skip_serializing_if = "Vec::is_empty")]`). Each detail is
  unescaped, final, whitespace-normalized text.
- **Insertion** (`insert_close_log_entry`). After the entry line `<indent>- <text>`,
  insert one `<indent><unit>- <detail>` line per detail, in order:
  - `<unit>` is the entry indent with the link indent stripped as a prefix, so the
    details step one level deeper exactly as the entry did.
  - When the entry indent does not extend the link indent, fall back to
    `child_indent_unit(entry indent)`.
  - Tag detail lines with a new `LineTag::InsertedDetail(ordinal)`, so
    `inserted_lines[ordinal]` stays the **entry** line. Reusing `Inserted` would record
    the last detail line, break `typed_work_log`, and fire a false "no task line"
    warning.
  - Later entries for the same link still land after the details, because
    `child_block_end` includes deeper lines.
  - CRLF, a missing final newline, and the lineup invariant need no change. Details
    cannot hold block links, so the lineup cannot change; the defensive
    `LogLineupChanged` check stays.
- **Reporting.** `ClosePlanner::write_logs` already maps a landed typed root to its
  ordinal. Alongside each typed dated string, record that entry's
  `selection.log[ordinal].details`. Pass the details by ordinal next to `log_lines`. The
  descendants written under an inserted entry are exactly its typed details, because the
  line after the insertion point is never deeper than the link.
- **JSON.** Schema version stays 1, and only additive fields are added.
  `pomodoro_close.log[].details` appears on `bob capture` (and on `capture-parse` via
  the spec). `pomodoro_close.tasks[].typed_work_log_details` is aligned 1:1 with
  `typed_work_log`: element _i_ lists the detail lines written under typed entry _i_. It
  is omitted when no typed entry on the row has details. `typed_work_log` keeps its
  shape (dated root strings), so an older Mac keeps decoding.
- **Human output** (`print_human_pomodoro_close_success`). Typed entries still print
  first, all of them, not dimmed. Each is followed by its details, indented two more
  spaces and not dimmed. Up to two other entries follow, dimmed. `+N Work Log` still
  counts entries only:

  ```text
  ✓ closed CAPTURE 0920-0950 → 0920-0940 (20m, −10m) · 2026/20260928.md line 5
    1 [*] → [/] Add support for `=x` syntax! bob.md ^capture-stop +3 Work Log
        *2026-09-28* — wired the lexer
          chose a hand-rolled lexer
        *2026-09-28* — Designed the `=x` grammar
        *2026-09-28* — Wrote the plan
    2 [*] → [/] Add capture support for web URLs! bob.md ^web-capture +1 Work Log
        *2026-09-28* — sketched the URL parser
  ```

### `capture-complete`, help, and docs (`grammar` phase)

- **Completion on a Work Log bullet line.** This is any physical line after the parent
  line inside a close item's line range, chain closes included. On such a line:
  - Suppress `wikilink_block` candidates; entries reject block links.
  - Suppress every capture-marker completion: `@route`, `@@route`, and task / `:`
    pickers. Bullet text is literal, and today the global `@@` completion still fires on
    any line.
  - Note and heading wikilink completion keep working.

  Replace `cursor_in_close_tail` (`capture_complete.rs`) with a line-based check. Byte
  ranges are unreliable here because a chain's close range contains later tokens.

- **Help.**
  - `bob capture --help`: replace the tail paragraph and its JSON notes. Update the
    examples to `printf '=x2,3\n- 2 wired the lexer\n' | bob capture` and
    `printf '=x =\n- 1 wired the lexer\n' | bob capture`.
  - `capture-parse --help`: update the span, need, `log`/`details`, and chain-range
    text.
  - Read `cli_rules.md` (`/sase_memory_read`) first, and keep help sorted, clear, and
    consistent.

### Presentation in Bob Mac Capture (`mac` phase)

- **Editing needs no new machinery.** The flow is `=x2,3`, ⌃J, `2 wired the lexer`, ⌃J,
  Tab, `chose a hand-rolled lexer`, ⌃J, ⇧Tab, `3 …`. Index chips on bullet lines are
  already cyan through `pomodoro_close_log_index` → `.pomodoroCloseLog`.
- **Pending.** `closePendingTrim` already works for bullets:
  - It removes the number under the placeholder, leaving `- `, which bob treats as a
    harmless placeholder row.
  - Its trailing-whitespace strip at most turns a final `- ` into `-`, which is still a
    placeholder.
  - Update its doc comment and add tests for a final and a mid-draft dangling bullet.

  `pendingLogText(index:)` drops the escape: "Type the Work Log entry for task 3".

- **Close card.**
  - `TaskRow` gains `typedWorkLogDetails: [[String]]`, aligned with
    `typedWorkLogPreviews`; a missing element counts as empty.
  - Under each typed entry line (pencil glyph, log tint, primary text), render its
    details in secondary callout text, one line each and tail-truncated. Indent them to
    the entry text, with no glyph.
  - Details are typed, so they are never capped. The other `work_log` entries stay
    capped at two.
  - The accessibility label and `accessibilitySummary` include details after their
    entry.
- **Hint.** `hintTokens` changes its log example to teach the bullet form:
  - the tokens are `=x` (pink), `⌃J` (neutral), `1` (log tint), and
    ` wrote the tests logs work to 1` (neutral);
  - this also fixes today's misleading `=x1 wrote the tests`, which reads as an
    in-progress list.
- **Models** (`CaptureModels.swift`). Both new fields decode with
  `decodeIfPresent … ?? []`, so an older bob still works:
  - `PomodoroCloseLogEntry.details: [String]`;
  - `PomodoroCloseTask.typedWorkLogDetails: [[String]]`.

## Rejected alternatives

- **Keep the inline tail alongside bullets.** Two grammars for one thing, and the tail
  carries every ambiguity listed in Context. It shipped hours ago, so there is no muscle
  memory to protect, and the retirement error shows the bullet to write.
- **An implicit target when exactly one task is loggable** (`=x2` then `- wired it`). It
  saves two keystrokes but makes the first token ambiguous: is `- 3 bugs fixed` an index
  or text? The number chip would also appear and disappear depending on the selection.
  An explicit number on every entry is predictable, and the error shows the fix.
- **Numbers anywhere in a bullet start entries** (the tail's rule). The bullet line
  already delimits the entry; only its first token needs to be an index.
- **`2:` / `2)` separators.** The request is `- 2 foo bar baz`, and with first-token
  indexing a separator adds nothing.
- **Details as separate dated entries, or a third bullet level.** The close's Work Log
  writer already nests undated descendants, and authored bullets have two levels.
  Mirroring both keeps one mental model.
- **Requiring `=x` to be last on a chain line that has bullets.** That forces the "log
  and switch sessions" idiom into a blank-line batch. Attaching bullets to the line's
  `=x` is unambiguous, because no other session operator takes children, and it costs
  only nested item ranges.
- **Keeping tail-chain detection only for nicer errors** (`-2 =x 1 foo`). It keeps about
  200 lines of splitting logic alive for a one-day-old form. The shape error is already
  clear.
- **Changing `typed_work_log` to objects with details.** A type change makes an older
  Mac's `decodeIfPresent` throw. An aligned additive array keeps the contract additive.
- **Line-number prefixes on bullet diagnostics.** Every bullet diagnostic already names
  its task number or echoes its text, which reads better in the Mac's error row than
  "capture line 3".

## Shared rules for every phase

- **bob-cli:**
  - `just all` (fmt, clippy, test) must pass.
  - Commit style is `feat(capture): …`.
  - Epic phase workers record `PROPOSED FOLLOW-UP:` notes on their own phase bead
    instead of creating beads.
  - Known flake: the parallel `BOB_DAY_FILE` race is tracked as `bob-cli-2e`; don't
    re-file it.
- **bob-mac-capture:**
  - Open it with `sase repo open bob-mac-capture -r "<why>"`. It has no `AGENTS.md`;
    follow the README's Development section and the `justfile`.
  - On Linux only `CaptureCore` and its tests build. Use Swift via swiftly:
    `PATH="$HOME/.local/share/swiftly/bin:$PATH"`, and don't source its `env.sh` under
    `sh`. The app target needs macOS.
  - macOS CI (`.github/workflows/ci.yml`) is the gate. Push, and the phase is done when
    CI is green for the commit (`gh run list` / `gh run view --log`).
  - Regenerate fixtures from a real `bob` built from bob-cli master after `grammar`.
- **No SASE memory changes and no decision-record changes.** Lane rules are untouched.
- **Never** close, start, or edit a real Pomodoro in `~/bob` from a test or a
  verification step. Use temp vaults, `--dry-run`, or `capture-parse`.

## Phase: engine — close planner writes Work Log details under typed entries and reports them

Implements "Execution" above. There is no grammar for details yet, so tests build specs
and `CloseSelection::with_log` values directly.

1. **Model.**
   - Add `CloseLogEntry.details` (serde default, skipped when empty).
   - Update every struct literal: `close_selection.rs`, `selection_tests.rs`,
     `linked_task_tests.rs`, `close_log.rs` (`log_entries_from_lex` fills an empty list
     for now), and the capture output mapping.
2. **Planner** (`capture_pomodoro_close/selection.rs`).
   - Insert detail lines after each entry, with the indent rule and
     `LineTag::InsertedDetail`.
   - Keep `inserted_lines` pointing at entry lines.
   - Reword `LogOutOfRange`: quote `- <n>` and drop the `\N` advice. It no longer needs
     the tail `raw`; carry what the new wording needs.
3. **Reporting** (`linked_tasks.rs`, `capture/pomodoro_close.rs`, `capture/output.rs`).
   - Collect the details of each landed typed entry in typed order, and add
     `PomodoroCloseTask.typed_work_log_details`.
   - Add `PomodoroCloseLogEntryJson.details` and
     `PomodoroCloseTaskJson.typed_work_log_details`, both skipped when empty.
   - Print details under their typed entry in human output.
4. **Docs.** In docs/capture.md's `pomodoro_close` field notes, document `log[].details`
   and `tasks[].typed_work_log_details`.
5. **Tests.**
   - Details land one level under their entry with tab indentation, 2-space indentation,
     and 4-space indentation where the link's first child is at 8.
   - The CRLF and no-final-newline variants.
   - Repeated entries for one link with details keep typed order and nesting.
   - A complete target gets its details in the completed task.
   - `inserted_lines` and `task_links[].ledger_line` stay correct for links below an
     insert that has details.
   - `typed_work_log_details` is aligned, and is omitted when there are no details.
   - The Work Log nests details undated, and `work_log.len()` counts only entries.
   - No false "no task line" warning.
   - Human output.
   - The new out-of-range wording.
   - Existing tests stay green, and plain closes stay byte-identical.

## Phase: grammar — parse Work Log bullets under `=x`, retire the inline tail, and document it

Implements "The shape", "Retiring the inline tail", "Chains", "Diagnostics", "Editing
states and spans", and "`capture-complete`, help, and docs".

1. **Bullet lexer** (rewrite `src/native/capture_language/close_log.rs`; keep the
   module).
   - **Input:** the close item's child `ItemLine`s, the close display token, and the
     lexed lists.
   - **Line classification:** classify each line with `classify_authored_line`.
     Placeholders are skipped; invalid and orphaned lines get the existing messages.
   - **Output:** every entry with its index, normalized text and details, the index
     range, and the text and detail ranges. Also report any dangling bullets, or the
     first error with its precise byte range.
   - **Delete** the tail machinery: escapes, `close_log_tail_start_error`, and
     `close_log_empty_entry_error`.
   - **Diagnostics** live in `markers.rs`.
   - **Unit tests:** every worked-table row and every diagnostic, with exact ranges.
2. **Execution parser** (`item.rs`, `parse_pomodoro_equals_item`).
   - A single-token parent with child lines goes through the bullet lexer and fills
     `spec.log`.
   - A dangling bullet gives the incomplete error.
   - A parent with extra tokens follows "Retiring the inline tail" (step order and echo
     rule), even when child lines follow.
   - Remove the redundant double `lex_close_selection` call.
   - Bullet text must never reach `resolve_line`, so markers stay literal.
3. **Editor parser** (`editor_pomodoro.rs`, `parse_editor_close_item`).
   - Emit the spans, the incomplete mode and need, the spec with `log`/`details`, and
     diagnostic ranges on bullet lines.
   - Keep the selection spans when bullets follow; today the child-line branch drops
     them.
   - Check that `@@` inheritance still skips closes, and that `capture-rewrite` never
     absorbs or declares an `@@` found in a bullet.
4. **Chains** (`draft.rs`).
   - Revert the tail splitting as specified.
   - Attach chain children to the line's (last) `=x` item.
   - Update `capture_language/tests/chain.rs`:
     - `+2 =x⏎- 1 foo` becomes a valid close with an entry, and `+2 =x⏎- child` now
       reports the missing-number error, replacing the child-shape error;
     - `=x =⏎- 1 foo` attaches the bullet to the close;
     - `= +2⏎- child` still fails the last token's shape rule;
     - nested item ranges are covered;
     - the non-chain table drops its tail rows.
5. **Completion** (`capture_complete.rs`, `capture_language/completion.rs`).
   - Add the line-based Work Log bullet check.
   - Suppress block, marker, and `@@` candidates on bullet lines, and keep note and
     heading candidates.
   - Replace `close_tail_suppresses_wikilink_block_but_keeps_note` with bullet-line
     equivalents, including a chain close and an `@@` attempt.
6. **Existing expectations that change.** Give each a precise new expectation:
   - `tests/cli/capture/pomodoro_close.rs`: `=x more` and `=x⏎- detail`, which is now a
     missing-number error.
   - `tests/cli/capture/parse_pomodoro_close.rs`:
     - the no-spaces cases;
     - `=x1 1`, `=x1,3 3`, and `=x 2` are no longer dangling; give each its new hint;
     - `=x1 x1`;
     - the tail-start case;
     - `=x1⏎- detail`.
   - `tests/cli/capture/pomodoro_close_selection.rs`: `=x1 more` and the dangling-index
     block.
   - `capture_language/tests/grammar.rs`: `=x more`.
   - `capture_language/tests/editor_modes.rs`: `=x more` and `=x⏎- child`.
   - `capture_parse.rs`: the JSON test.
   - `capture_complete.rs`: `close_items_and_suffixes_request_no_completion`.
7. **CLI integration tests.** Rewrite `tests/cli/capture/pomodoro_close_log.rs` for
   bullets on `close_worked_vault` with `BOB_NOW="2026-09-28 09:37:00"`:
   - the worked-table executions, with byte-exact day-file and `bob.md` post-images,
     including details;
   - dry run equals the real run;
   - `pomodoro_blocks` covers the inserted entry and detail lines;
   - batch rollback on a bullet error;
   - chains (`-2 =x` and `=x =` with bullets);
   - each retired-tail message;
   - `capture-parse` JSON for valid, detail, chain, dangling (final and mid-draft), and
     invalid drafts.
8. **Docs** (`docs/capture.md`).
   - Rewrite "Logging work while closing": the shape, loggability, details, literal and
     rejected text, a worked example with before/after ledger and Work Log, human
     output, JSON, diagnostics, and the retired tail with its hint.
   - Update the grammar-at-a-glance rows for `=x` and the tail/chain rows.
   - Update the `#` table:
     - the `=x more`, `=x 1`, and `=x 1 wired the lexer` rows;
     - add rows for a bullet with no number and for `=x` + a placeholder.
   - In "Closing the running Pomodoro", replace "takes no child lines" with "its child
     lines are Work Log bullets". Fix the related "`=x more` (or a child line)" sentence
     in the worked-example paragraph.
   - "Chaining session operators on one line": remove the tail text and document the
     bullet attachment rule plus nested item ranges.
   - The `capture-parse` and `capture-complete` sections.
   - Update the help per the design.

## Phase: mac — Bob Mac Capture previews Work Log bullets and their details

Implements "Presentation in Bob Mac Capture".

1. **Models.**
   - Add `PomodoroCloseLogEntry.details` and `PomodoroCloseTask.typedWorkLogDetails`,
     additive with defaulted init parameters.
   - Fix the merged doc comment above `PomodoroCloseLogEntry` / `PomodoroCloseSpec`.
2. **Presentation** (`CapturePomodoroClosePresentation.swift`).
   - Add the aligned `typedWorkLogDetails` to `TaskRow`, built in `taskRow(for:)`.
   - Include details in the accessibility label and in `accessibilitySummary`.
   - Make `pendingLogText` escape-free.
   - Change the `hintTokens` log example.
3. **Close card** (`CapturePanelView.swift`, `closePreviewItem`). Render the details
   under each typed entry as specified.
4. **Pending trim** (`CapturePanelModel.swift`). Update the doc comment. Add tests for
   `=x⏎- 1`, `=x⏎- 2 a⏎- 1`, and `=x⏎- 1⏎- 2 a`: each trims to a placeholder row,
   previews with Close disabled, and shows the escape-free pending text.
5. **Fixtures.** Regenerate from a real `bob` built from bob-cli master, keeping the
   file names with new bullet drafts:
   - `pomodoro-close-parse-log.json` for
     `=x⏎- 1 wired the lexer⏎  - chose a hand-rolled lexer`;
   - `pomodoro-close-parse-log-incomplete.json` for `=x⏎- 1`;
   - `pomodoro-close-parse-log-chain.json` for `=x =⏎- 1 wired it`;
   - `pomodoro-close-log.json` (dry-run capture with an entry plus a detail);
   - `pomodoro-close-log-invalid.json` for `=x⏎- 1 see [[bob#^x]]`;
   - the matching multi-line `fake-bob` branches.
6. **Tests.**
   - Decoding of the new fields, including an older-bob JSON without them.
   - Details render under their entry, uncapped and aligned, with missing elements
     treated as empty.
   - Hint and pending text, updating the pinned strings.
   - Index-chip highlighting on a later line next to a wikilink in entry text.
   - Retire the tail-only test inputs.
7. **Finish.**
   - Update the README: the runtime contract, highlighting, the close card (details),
     the syntax section (bullets, the ⌃J/Tab flow, no escape), and the pending text.
     Also fix the stale "six visual lines" editor-height sentence you will pass.
   - Run `just format-lint` where available, commit, push, and get macOS CI green.

## Phase: rollout — install bob, verify bullet drafts with dry runs, and hand Bryan the Mac steps

1. **Install on the host running this phase.** From an up-to-date bob-cli master, run
   `cargo install --path . --locked --force`.
2. **MacBook (best effort).** Read `tailnet.md` with `/sase_memory_read`, then
   `ssh -o ConnectTimeout=15 mac` and reinstall `bob` the way it was installed (check
   `~/.cargo/.crates2.json`):
   - A clean `path+file://` checkout on master: `git pull --ff-only`, then
     `cargo install --path . --locked --force`.
   - A `git+` install: reinstall from the same source.
   - Anything else, or the Mac is unreachable: record the exact steps and continue.
3. **Verify against the live vault without writing.** Paste the key output into the
   phase notes:
   - `bob capture-parse -f json -- "$(printf '=x2,3\n- 2 wired the lexer\n  - chose a hand-rolled lexer')"`:
     spans, `log` with `details`, and mode.
   - `bob capture-parse -f json -- "$(printf '=x\n- 1')"`: incomplete, with the need and
     the placeholder.
   - `bob capture-parse -f json -- "$(printf '=x =\n- 1 wired it')"`: two items, with
     the bullet on the close.
   - `bob capture-parse -f json -- '=x2,3 2 foo bar baz'`: the retired-tail hint echoing
     `- 2 foo bar baz`.
   - If a Pomodoro is running today, run
     `printf '=x\n- 1 rollout check\n  - detail\n' | bob capture --dry-run` in human
     output and in `-f json`. Check the row's `typed_work_log`,
     `typed_work_log_details`, and the closed block's added lines. If nothing is
     running, say so and skip it.
4. **Bryan's checklist** (final response, not automated).
   - Rebuild and install Bob Mac Capture from master (`just install`) once `mac` CI is
     green.
   - Type `=x2,3`, then ⌃J, `2 wired the lexer`, ⌃J, Tab, `a detail`. Check that the
     cyan chip matches the card's entry line and that the detail sits under it.
   - Try `=x =` with a bullet to log and switch sessions in one draft.
   - The old `=x2 2 …` form now shows the bullet to write.

## Deliberately not doing

- **Work Log bullets on link-form closes.** `^route:id=x` and a solo `@route:id=x` keep
  rejecting child bullets. `<text> @route:id=x` keeps writing them as the new task's
  sub-bullets. To log on a just-linked task, link it first (`^route:id`), then close
  with bullets in a batch.
- **A task-number completion list on bullet lines.** The close card's numbered rows
  already guide the choice.
- **Clipboard (`%`) as entry text, or a third bullet level.**
- **Obsidian (bob-plugins) changes.** Ctrl+Enter keeps reading hand-typed sub-bullets.
- **A Mac auto-rewrite of the retired tail into bullets.** The error already shows the
  exact bullet, and the grammar stays in bob.
