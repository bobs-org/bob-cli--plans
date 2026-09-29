---
tier: epic
title: Choose each Task Link's outcome while closing a Pomodoro with =x<N>!<M>
goal: '`=x<N>`, `=x!<M>`, and `=x<N>!<M>` close the running Pomodoro and decide, by
  number,

  which of its Task Links stay in progress, which are deferred, and which are

  completed and struck. This does in one capture what the user does by hand before

  Ctrl+Enter: append `#` to links that should not start, and transclude links whose

  tasks are finished. Every close stays atomic. A mistyped list gets a precise

  diagnostic. Plain `=x` keeps working byte for byte as it does today. `bob capture`

  output and the Bob Mac Capture close card show a number badge on every Task Link

  and the outcome each one will get, so choosing the numbers is easy.

  '
phases:
- id: selection-planner
  title: Numbered Task Links and outcome selection in the pure close planner
  depends_on: []
  size: medium
  description: 'selection-planner: in `src/native/capture_pomodoro_close/`, number
    the running

    session''s Task Links. Apply a `CloseSelection` by rewriting only the `#`/`![[…]]`

    markers of the numbered lines, then run the unchanged close. Validate numbers

    against the lineup, warn when a listed task cannot change status, and expose the

    numbered lineup plus a per-row `index`. Every caller passes `None` for now. Pinned

    by unit tests on the worked example.

    '
- id: selection-grammar
  title: =x<N>!<M> grammar, capture-parse contract, and editor states
  depends_on: []
  size: medium
  description: 'selection-grammar: lex `=x[<N>][!<M>]` once and share the lexer between
    the

    whole-item close and the `@`/`^`/body-bearing `=x` suffix, in both the execution

    and editor parsers. Report the additive `in_progress`/`complete` spec, new span

    kinds, precise `invalid_pomodoro_close` diagnostics, and an incomplete

    `pomodoro_close_task` need for a dangling `,`/`!`. Update capture-complete,

    capture-rewrite, and capture-parse help. The executor refuses a selection-bearing

    close until selection-capture wires it in.

    '
- id: selection-capture
  title: Wire the selection into all close forms, JSON, and human output
  depends_on:
  - selection-planner
  - selection-grammar
  size: medium
  description: 'selection-capture: pass the parsed selection into the planner for
    whole-item,

    link, and new-task closes, and remove the temporary refusal. Emit

    `in_progress`, `complete`, `task_links`, and `tasks[].index` in the

    `pomodoro_close` JSON. Print a numbered index column in human output, and fix
    the

    `carries 1 links` plural. Pin all of it with CLI integration tests on the worked

    example, including batches, link forms, dry-run parity, and every diagnostic.

    '
- id: selection-docs
  title: Help and docs for =x<N>!<M>
  depends_on:
  - selection-capture
  size: small
  description: 'selection-docs: document the selection grammar, numbering, outcomes,

    diagnostics, and the JSON and human output. Update `bob capture --help`,

    `docs/capture.md` (grammar tables, the close section with worked examples, and

    the capture-parse, capture, and capture-complete contracts), and `README.md`.

    '
- id: mac-selection-preview
  title: Bob Mac Capture numbered close card, span colors, and pending list state
  depends_on:
  - selection-capture
  size: medium
  description: 'mac-selection-preview: in the bob-mac-capture linked repo, decode
    the new close

    fields. Render SF Symbol number badges tinted by outcome on the close card, with

    a teaching hint before a selection is typed and an outcome summary after. Color

    the in-progress and complete list spans in the editor. Keep a live, dimmed card

    with Close disabled while a list ends in `,` or `!`. Update status and

    notifications, regenerate real-bob fixtures, add tests and README notes, and get

    macOS CI green.'
proposed_by: bbugyi200.apollo.34
create_time: 2026-09-29 13:45:01
status: wip
bead_id: bob-cli-2k
---

- **PROMPT:** [prompts/202609/close_task_selection.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/close_task_selection.md)
- **BEAD:** [bob-cli-2k](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2k/README.md)

# Plan: Choose each Task Link's outcome while closing with `=x<N>!<M>`

## Why this exists

`=x` closes today's running Pomodoro exactly the way Obsidian's Ctrl+Enter Pomodoro
completion does. That completion reads each Task Link's own markers:

- a bare `[[note#^id]]` is **worked on**: it gets one `🍅 `, is carried to the next
  session, and its task starts (`[ ]`/`[*]` → `[/]`);
- a bare `[[note#^id]]#` is **deferred**: it is removed from the session and carried
  without the `#`, and its task is not started;
- a bare `![[note#^id]]` transclusion is **completed**: its task closes recursively with
  a completion date, and the line retires to `~~[[note#^id]]~~`.

Before pressing Ctrl+Enter in Obsidian, the user edits those markers by hand. They
append `#` to every link whose task should not be marked In Progress, and they add `!`
to every link whose task they finished. The new syntax does those edits for them, by
number, from `bob capture` and Bob Mac Capture:

- `=x<N>`: only the numbered links in `<N>` stay in progress; every other numbered link
  is treated as if it had `#` appended.
- `=x!<M>`: the links in `<M>` are completed and struck; the rest keep today's behavior.
- `=x<N>!<M>`: both at once.

`<N>` and `<M>` are comma-separated task numbers. The number of each Task Link must be
obvious and pleasant to read in the preview that appears when `=x` is typed.

The user asked the implementer to lead the design and to make it intuitive, reliable,
and beautiful. This plan is that design. Its core idea: **a selection is nothing but the
marker edits the user would make by hand, applied to the numbered lines, followed by the
unchanged `=x` close.** That keeps the result identical to the manual workflow, keeps
plain `=x` untouched, and keeps the change small.

## The contract (shared by every phase)

### Terms

- **Running Pomodoro _R_, ledger, clock:** exactly as for `=x` today
  (`capture_pomodoro_close::find_running_pomodoro`, `BOB_DAY_FILE`/`BOB_NOW`).
- **Numbered Task Link:** a line of _R_'s sub-bullet range
  (`capture_pomodoro_close::sub_bullet_range`) that is not fenced
  (`markdown::fenced_lines`) and whose list-item body, after `strip_pomodoro_markers`
  and trimming trailing spaces and tabs, is exactly one block link in one of these four
  shapes:

  | Body shape | Marker                                | Outcome if nothing is listed |
  | ---------- | ------------------------------------- | ---------------------------- |
  | `[[T]]`    | plain                                 | in progress                  |
  | `[[T]]#`   | deferred                              | deferred                     |
  | `![[T]]`   | embedded                              | complete                     |
  | `![[T]]#`  | embedded (the `#` is inert, as today) | complete                     |

  `T` is any target that `wikilink_tokens` parses as a block link (`path#^id`, with an
  optional `|alias`). Depth does not matter: a nested bare link is numbered too, because
  the close starts, defers, or completes it just like a direct child. The following are
  **never numbered**: struck links (`~~[[T]]~~`, already done), lines that mix a link
  with other text (today's "mentioned" role, which is never started), notes, and fenced
  lines. Numbers start at 1 and follow ledger order, top to bottom. This follows the
  glossary's Task Link definition: a block link that is the only content of a Pomodoro
  sub-bullet.

- **Outcome:** `in_progress`, `deferred`, or `complete`, the fate the close gives a
  numbered line:
  - `in_progress` means today's worked role: one `🍅`, carried, and the task started
    when it is Ready or Next;
  - `deferred` means removed from the session, carried, and not started;
  - `complete` means today's embedded role: the task closes recursively and the line is
    struck.
- **Source**, which says why a line got its outcome:
  - `listed`: its number appears in `<N>` or `<M>`;
  - `unlisted`: `<N>` was typed and does not contain it, and neither does `<M>`;
  - `ledger`: no list mentions it, `<N>` was not typed, and the outcome comes from the
    line's own marker.

### Grammar

```
=x[<N>][!<M>]
<N> := 0 | <n>(,<n>)*        in-progress list; a lone 0 means "none"
<M> := <n>(,<n>)*            complete list
<n> := ASCII digits whose value is ≥ 1 and fits in u32
```

- `x` is case-insensitive (`=X1!2` is accepted). `raw` preserves what was typed. Docs
  and help show lowercase.
- Nothing may contain whitespace. `<N>`, when present, comes before `!<M>`. Only one `!`
  is allowed. Order inside a list does not matter; JSON reports each list sorted
  ascending.
- `<N>` omitted leaves unlisted links at their ledger outcome. `<N>` present, even as
  `0`, turns every unlisted link that would have been in progress into deferred.
- **Whole item.** Today's rules stay as they are, with one addition. When the first
  whitespace-delimited token of an item's first line starts with `=x`/`=X` and the
  character right after the `x` is a digit, `,`, or `!`, that token is a
  **selection-shaped close** and claims the item:
  - an exact item (single physical line, trimmed text equal to the token) with a valid
    selection closes _R_ with that selection;
  - anything else is an `invalid_pomodoro_close` error and never becomes task text. That
    covers a malformed list, extra text, markers, and child lines.

  `=x` followed by any other character (`=xx`, `=xa`, `=x.`) and a mid-body `=x…`
  (`Plan =x1`) stay ordinary prose, exactly as today. **Change:** `=x!` used to be prose
  and is now an incomplete close (see below). Update the docs row that lists it.

- **Link suffix.** Wherever `=x` is legal today (`@route:block-id=x…`,
  `^route:block-id=x…`, `<text> @route:block-id=x…`), the suffix `x`/`X` may be followed
  by the same selection. `parse_colon_link_tail` recognizes a suffix of `x`, or `x`
  followed by a digit, `,`, or `!`, as a close. Every other suffix keeps today's `=<X>`
  start handling and messages. All existing close conflicts keep their text: `#name=x…`,
  `s:<N>`, `p:<N>`, project-note `+`, forced flags, and `@@` never applying. On link
  forms the numbering is taken **after** the link step. A newly linked or created task
  lands last in _R_ and gets the highest number; a task that is already current keeps
  its place. The preview shows exactly those numbers.
- Make sure `!` and `,` inside an `@…:…=x…` token reach the suffix parser. The
  explicit-toggle `!` logic (`exact_explicit_toggle_prefix`,
  `explicit_toggle_unsupported_message`) only concerns `+` forms, and must stay that
  way.

### Lexical diagnostics

These are the same in `bob capture` (an error, nothing written) and `capture-parse` (an
`invalid_pomodoro_close` error with the byte range shown). Tests assert the stable
leading phrase. Keep the message text in one place in `capture_language/markers.rs` so
both parsers share it.

| Input                               | Message (leading phrase)                                                                                                                                | Range                                           |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `=x1,1`                             | ``task 1 is listed twice in `=x1,1` ``                                                                                                                  | the second `1`                                  |
| `=x1!1`, `=x1,2!2`                  | ``task 1 cannot both stay in progress and complete in `=x1!1` ``                                                                                        | the number inside `!<M>`                        |
| `=x0,2`, `=x2,0`                    | `` `0` means no task stays in progress; use it alone, as `=x0` or `=x0!2` ``                                                                            | the `0`                                         |
| `=x!0`, `=x1!0`                     | `task numbers start at 1`                                                                                                                               | the `0`                                         |
| `=x,1`, `=x1,,2`, `=x!,1`           | `` expected a task number before `,` ``                                                                                                                 | the `,`                                         |
| `=x1!2!3`                           | `` use one `!` list: `=x1!2,3` ``                                                                                                                       | the second `!`                                  |
| `=x1a`, `=x1;2`, `=x!2x`            | `` `=x1a` is not a task list: write `=x`, then comma-separated task numbers, then optionally `!` and the numbers to complete (for example `=x1,3!2`) `` | the first bad byte through the end of the token |
| `=x99999999999`                     | `task number 99999999999 is too large`                                                                                                                  | that number                                     |
| `=x 1,3`, `=x1, 3`, `=x1 !2`        | ``write the task numbers right after `=x`, with no spaces (for example `=x1,3!2`)``                                                                     | the extra text                                  |
| `=x1 more`, `=x1` plus a child line | today's `POMODORO_CLOSE_SHAPE_ERROR`                                                                                                                    | the extra text or child line                    |

**Incomplete lists.** A token ending in a dangling separator (`=x1,`, `=x!`, `=x1!`,
`=x!2,`, `=x0!`, and the same on link suffixes such as `^bob:ready=x1,`) is an editing
state, not a mistake:

- `capture-parse` reports mode `incomplete` and `needs: ["pomodoro_close_task"]`. It
  reports the partial `pomodoro_close` spec typed so far (`=x1,` gives
  `in_progress: [1]`; `=x!` gives `in_progress: null, complete: []`). It emits the spans
  typed so far plus one `interactive_placeholder` span over the dangling `,` or `!`. It
  adds no error diagnostic. A lexical error elsewhere in the token wins over the
  incomplete state.
- `bob capture` rejects it: `` `=x1,` is incomplete: type a task number after `,` `` (or
  `` after `!` ``).

### Applying a selection (pure planner)

For numbered line _i_ with marker _m_:

| Condition (first match wins)   | Outcome                                | Source   |
| ------------------------------ | -------------------------------------- | -------- |
| _i_ ∈ `<M>`                    | complete                               | listed   |
| _i_ ∈ `<N>`                    | in progress                            | listed   |
| `<N>` typed and _m_ = embedded | complete (a hand transclusion is kept) | unlisted |
| `<N>` typed                    | deferred                               | unlisted |
| otherwise                      | the ledger outcome of _m_              | ledger   |

The third row is the literal "as if `#` were appended" rule: `#` on a transclusion
changes nothing.

Then **rewrite, in place, only the numbered lines whose outcome differs from their
marker's ledger outcome.** Keep the indentation, list marker, and following whitespace.
Replace the body with the canonical shape for the outcome:

- in progress → `[[T]]`;
- deferred → `[[T]]#`;
- complete → `![[T]]`.

`[[T]]` is the original token text, copied verbatim with its alias. Drop 🍅 markers from
rewritten lines; the unchanged close adds back exactly one on worked lines. Line count
never changes, so every ledger line number stays valid. Lines whose outcome already
matches stay byte-identical. Then run today's close on the rewritten contents, with
nothing else changed.

Consequences, all asserted by tests:

- `=x` with no selection is byte-identical to today, in the ledger and in every task
  note.
- Each selection produces exactly the files that the matching hand edits followed by
  `=x` produce. The worked examples below were produced that way with the current
  binary.
- Work Log collection still sees notes under a deferred link, because
  `work_log_task_link_target` already strips a trailing `#`. A deferred task with notes
  keeps its Work Log entries. The orphaned-notes quirk of a deferred line is the same
  one the manual flow has.

**Vault-aware validation** runs after the lineup is numbered and before anything is
written or staged. In a batch it rolls the whole batch back.

- **Out of range:**
  `` `=x4` names task 4, but CAPTURE has 2 numbered Task Links (1–2) ``. For one link,
  say `(1)`. For none:
  `` `=x1` names task 1, but CAPTURE has no numbered Task Links; close it with `=x` ``.
  An unnamed _R_ reads "the running Pomodoro" in place of `CAPTURE`. `=x0` is always in
  range. List every out-of-range number in one message (`names tasks 4 and 6`).
- **Conflicting duplicates:** when two numbered lines carry the same `path#^id` and end
  up with different outcomes, the close fails with
  ``tasks 1 and 3 both link `[[bob#^a]]` but get different outcomes; give them the same one``.
  Same-outcome duplicates are fine.

**Warnings**, which do not block the close and are shown on the row and in the top-level
`warnings`:

- A **listed** in-progress line whose resolved task did not end In Progress (Blocked,
  Done, Canceled, and so on): ``task 2 `[[bob#^x]]` is Blocked, so it was not started``.
- A **listed** complete line whose resolved task did not end Done:
  ``task 3 `[[bob#^y]]` is Blocked, so it was not completed``.
- Unresolved rows already warn today. Do not warn twice.

### Worked examples (shared test fixture)

Use the existing close fixture, `close_worked_vault` in
`tests/cli/capture/pomodoro_close.rs`, with `BOB_NOW=2026-09-28 09:37:00`. _R_ is
CAPTURE on line 5. Its numbered Task Links are:

1. line 6, `[[bob#^capture-stop]]` (plain, with nested notes);
2. line 10, `[[bob#^web-capture]]#` (deferred).

The struck `~~[[sase#^axe-restart]]~~` line and `quick note` are unnumbered. Every case
below also closes the entry as `- [x] (**0920-0940** [t:: 20m]) — CAPTURE`, leaves PLAN
and SASE unchanged, and gives `sase.md` the same Work Log entry as plain `=x`.

**`=x`**: 1 in progress (ledger), 2 deferred (ledger). The ledger, `bob.md`, and
`sase.md` are byte-identical to today's documented worked example.

The other cases need exact bytes, so each is spelled out below. Each block shows the
ledger from the closed CAPTURE entry through the new placeholder, with TAB indentation.
These post-images were produced with the current binary by making the equivalent hand
edits and then running `=x`.

**`=x2`**: 1 deferred (unlisted), 2 in progress (listed).

```markdown
- [x] (**0920-0940** [t:: 20m]) — CAPTURE - Designed the `=x` grammar - chose `x` for
      done - Wrote the plan
  - 🍅 [[bob#^web-capture]]
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — CAPTURE
  - [[bob#^web-capture]]
  - [[bob#^capture-stop]]
```

`bob.md`: `^capture-stop` stays `[*]` but still gets its two-entry `🛠️ **WORK LOG**`.
`^web-capture` becomes `[/]`. Human rows: `[*] deferred … ^capture-stop +2 Work Log`,
`[*] → [/] … ^web-capture`. Next: CAPTURE, created at line 13, carries 2 links.

**`=x!1`**: 1 complete (listed), 2 deferred (ledger).

```markdown
- [x] (**0920-0940** [t:: 20m]) — CAPTURE
  - ~~[[bob#^capture-stop]]~~
    - Designed the `=x` grammar
      - chose `x` for done
    - Wrote the plan
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — CAPTURE
  - [[bob#^web-capture]]
```

`bob.md`:
`- [x] #task Add support for `=x` syntax! [created::2026-09-26]  [completion:: 2026-09-28] ^capture-stop`
(two spaces before `[completion::`, as today's embedded close writes it), followed by
the same two-entry Work Log. `^web-capture` stays `[*]`. Next: created at line 13,
carries 1 link.

**`=x1!2`**: 1 in progress (listed), 2 complete (listed).

```markdown
- [x] (**0920-0940** [t:: 20m]) — CAPTURE
  - 🍅 [[bob#^capture-stop]]
    - Designed the `=x` grammar
      - chose `x` for done
    - Wrote the plan
  - ~~[[bob#^web-capture]]~~
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
```

`bob.md`: `^capture-stop` becomes `[/]` with its Work Log. `^web-capture` becomes
`- [x] #task Add capture support for web URLs! [created::2026-09-21]  [completion:: 2026-09-28] ^web-capture`.
Next: created at line 14, carries 1 link.

**`=x0`**: 1 deferred (unlisted), 2 deferred (unlisted).

```markdown
- [x] (**0920-0940** [t:: 20m]) — CAPTURE - Designed the `=x` grammar - chose `x` for
      done - Wrote the plan
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
```

`bob.md`: both tasks stay `[*]`, and `^capture-stop` still gets its Work Log. Next:
created at line 12, carries 2 links.

**`=x1` and `=x1,2`**: `=x1` equals plain `=x` on this fixture (2 was already deferred).
`=x1,2` un-defers 2: both links get `🍅`, both tasks become `[/]`, and the placeholder
carries `[[bob#^capture-stop]]` and then `[[bob#^web-capture]]`.

**`^bob:ready=x3`**: the link step makes Ready `^ready` Next and appends
`[[bob#^ready]]` after `quick note` on line 13 as number 3. Then 1 is deferred
(unlisted), 2 is deferred (unlisted), and 3 is in progress (listed).

```markdown
- [x] (**0920-0940** [t:: 20m]) — CAPTURE - Designed the `=x` grammar - chose `x` for
      done - Wrote the plan
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
  - 🍅 [[bob#^ready]]
- [ ] () — CAPTURE
  - [[bob#^ready]]
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
```

`^ready` becomes `[/]`, `^capture-stop` stays `[*]` with its Work Log, and
`^web-capture` stays `[*]`.

**Failures on the same fixture, all of which write nothing:**

- `=x3` gives `names task 3, but CAPTURE has 2 numbered Task Links (1–2)`.
- `=x1,1`, `=x1!1`, `=x0,1`, and `=x1,` give the lexical messages above.
- `printf -- '-2\n\n=x5\n'` rolls back the `-2`.

### Output contract

All additions are additive, and schema version 1 is unchanged. Inside `pomodoro_close`,
follow the existing explicit-`null` convention: say `null`, never omit the field.

**`capture-parse`**:

- `pomodoro_close` becomes
  `{ "raw": "=x1,3!2", "in_progress": [1, 3], "complete": [2] }`.
  - Plain `=x` gives `in_progress: null, complete: []`.
  - `=x0` gives `in_progress: []`.
  - Link forms keep today's `raw` convention: the suffix including `=`, so `=x1!2`.
  - A lexically invalid token reports mode `pomodoro_close` (or the link item's mode)
    with an `invalid_pomodoro_close` error and **no** `pomodoro_close` spec, as today's
    near misses do.
- Spans:
  - `pomodoro_close` covers only the `=x` bytes, as today;
  - new `pomodoro_close_in_progress` covers `<N>` including its commas;
  - new `pomodoro_close_complete` covers `!<M>` including the `!`.

  For example, `=x1,3!2` gives `[0,2) pomodoro_close`,
  `[2,5) pomodoro_close_in_progress`, and `[5,7) pomodoro_close_complete`. Link forms
  put the same three spans after the route and block-ID spans.

- Add `pomodoro_close_task` to the `needs` vocabulary, last in the documented order.
- Human output: the `close` line reads
  `=x1,3!2 (in progress 1, 3 · complete 2 · defer the rest)`. Build it from these parts:
  - `in progress none` for `=x0`;
  - `complete …` only when `<M>` is non-empty;
  - `defer the rest` whenever `<N>` was typed.

  Plain `=x` prints `=x` alone, as today.

- The multi-item `items[]` entries carry the same spec.

**`bob capture` JSON (`pomodoro_close` object)**:

- `raw` now includes the selection.
- New `in_progress` (array or `null`) and `complete` (array), echoing the parse.
- New `task_links`: the numbered lineup, always present and possibly empty, in number
  order. Each entry is:

  ```json
  {
    "index": 1,
    "ledger_line": 6,
    "block_link": "[[bob#^capture-stop]]",
    "block_id": "capture-stop",
    "marker": "plain",
    "outcome": "in_progress",
    "source": "ledger"
  }
  ```

  - `marker` is one of `plain`, `deferred`, `embedded`: the line's marker before the
    selection.
  - `outcome` is one of `in_progress`, `deferred`, `complete`.
  - `source` is one of `ledger`, `listed`, `unlisted`.
  - `ledger_line` uses the same basis as `tasks[].ledger_line`: the close's pre-image,
    after any link step.
  - `block_link` never includes the `!`.

- New `tasks[].index`: the number of the numbered line that produced the row, or `null`
  for unnumbered rows (struck, mentioned, subtask, and Work-Log-only). When one task is
  numbered on several lines, its single row carries the lowest number.
- `tasks[].role` keeps its current meaning; it is the role after the selection. A
  completed row is `embedded` and a deferred row is `deferred`.
- A client joins `tasks[].index` to `task_links[index-1]` for `outcome`/`source`.

**`bob capture` human output**:

- When at least one row is numbered, prefix every task row with a right-aligned index
  column (width = digits of the highest number) plus one space. Unnumbered rows get
  blanks of the same width, and Work Log preview lines indent so they stay under the
  text.
- Style the index bold, colored by outcome: in progress uses
  `style_task_status_marker`'s `/` color, complete uses its `x` color, and deferred is
  dim. Unlisted indices are also dim.
- With zero numbered rows the output is byte-identical to today.
- Fix the existing `carries 1 links` so it reads `carries 1 link`.

Example (`=x1!2`, `NO_COLOR`):

```
✓ closed CAPTURE 0920-0950 → 0920-0940 (20m, −10m) · 2026/20260928.md line 5
  1 [*] → [/] Add support for `=x` syntax! bob.md ^capture-stop +2 Work Log
      *2026-09-28* — Designed the `=x` grammar
      *2026-09-28* — Wrote the plan
  2 [*] → [x] Add capture support for web URLs! bob.md ^web-capture
    [x] Restart axe sase.md ^axe-restart +1 Work Log
      *2026-09-28* — Restarted axe
  next: CAPTURE (created) at line 14 · carries 1 link
```

**`capture-complete` / `capture-rewrite` / `@@`**:

- A cursor anywhere inside `=x…`, including the lists and a dangling separator, returns
  an empty success.
- Selection-bearing close items are never rewritten or absorbed.
- A `@@` declaration never applies to them.

### Bob Mac Capture design (`mac-selection-preview`)

The close card is where the user picks numbers. It must teach the syntax before it is
used, reflect every keystroke, and never hide a number.

- **Number badges.** Every numbered row starts with a badge in a fixed-width leading
  column:
  - use the SF Symbol `"\(n).circle.fill"` when the row is **listed**, and
    `"\(n).circle"` otherwise;
  - for `n > 50`, fall back to a monospaced-digit `Text` in a `Capsule`;
  - tint by outcome with the palette's existing status hues: in progress `.orange` (the
    picker's In Progress color), complete `.green`, deferred `.secondary`;
  - **unlisted** rows render at reduced opacity (about 0.5), so the chosen rows stand
    out;
  - unnumbered rows get an equal-width clear spacer whenever any row is numbered, so
    text stays aligned. With no numbered rows the card looks exactly like today.
- **Outcome glyph and text.** In-progress and deferred rows keep today's transition text
  and glyphs. Complete rows keep today's embedded glyph, `checkmark.circle.fill`, but
  tint it green, and strike their task text through, because the ledger line will be
  struck. Their transition text, used for accessibility and status, becomes `[*] → [x]`
  when the status changed (previous marker → current marker) and stays `[x] closed`
  otherwise.
- **Teaching hint.** It appears when no selection is typed and at least one row is
  numbered. It is one caption row under the task rows with a `number.circle` glyph. Its
  example tokens use the same colors as the editor spans:
  - with 2+ numbered rows:
    `=x1,2 keeps only these in progress · =x!2 completes 2 · =x0 defers all`;
  - with exactly 1: `=x!1 completes it · =x0 defers it`.
- **Selection summary.** It replaces the hint once a selection is typed, for example
  `In progress 1, 3 · Complete 2 · Deferred 4`. It is built from `task_links` in
  ascending order. Empty groups are omitted, except that `=x0` shows `In progress none`.
- **Never hide a number.** Numbered rows always render. The `+N more` overflow (cap 6)
  only hides unnumbered rows beyond `max(0, 6 − numbered)`.
- **Editor colors.** `=x` keeps the Pomodoro-session pink. Map the new span kind
  `pomodoro_close_in_progress` to a new `CaptureSemanticCategory` rendered `.orange`,
  and `pomodoro_close_complete` to one rendered `.green`. The typed digits then share
  the colors of the badges they select.
- **Pending list (an item's `needs` contains `pomodoro_close_task`).** The existing
  picker needs (`active_task`, `pomodoro_id`, `block_id`) keep precedence. Otherwise, do
  not run the doomed dry run on the real draft. Instead:
  - for every item that needs `pomodoro_close_task`, remove the one
    `interactive_placeholder` span Bob reported inside that item's range (the dangling
    `,`/`!`), and run the normal live preview on the trimmed draft, so the card always
    reflects exactly what has been typed so far;
  - render that card in a pending style: the summary row reads
    `Type a task number after ,` (or `after !`), and the card is slightly dimmed;
  - disable the primary action, so Return does not submit and the status line explains
    why. Nothing new is submitted;
  - once the draft becomes valid again, render the normal card.

  No Swift-side ledger logic is added.

- **Status, notifications, accessibility.**
  - Status gains `· N completed` when N > 0, for example
    `Would close CAPTURE · 1 started · 1 completed · 2 Work Log entries`. Existing
    strings are unchanged when N is 0.
  - The notification's task line appends the same count.
  - Each numbered row's accessibility label reads
    `Task 1, <text>, stays in progress | deferred | completes`, adding `, chosen` when
    the row is listed. The hint and summary are read too.
- **Older Bob:** missing `in_progress`/`complete`/`task_links`/`index` decode as none,
  and the card is exactly today's.

### Deliberate choices

1. **A selection is marker edits plus the unchanged close.** This gives byte parity with
   the user's manual workflow, zero regression risk for plain `=x`, and no second close
   engine.
2. **Numbers cover every bare Task Link line, at any depth.** Those are exactly the
   lines the close would start, defer, or complete, so `=x<N>` is a complete and strict
   statement ("only these start"). Struck, mentioned, and note lines are never touched
   by a selection and are never numbered.
3. **Listing beats ledger markers.** An unlisted hand-transcluded link stays complete
   (the literal `#` rule), while an explicitly listed number always wins, including
   un-deferring `[[T]]#` or un-transcluding `![[T]]`.
4. **`0` means none.** Without it, "defer everything" or "complete 2 and defer the rest"
   cannot be expressed. `=x0` and `=x0!2` fill that gap with a digit users already read
   as "zero tasks".
5. **Errors versus warnings.** Anything that makes the request ambiguous is an error and
   writes nothing: malformed lists, duplicates, overlaps, out-of-range numbers, and
   conflicting duplicates. A listed task whose status the close cannot change is a
   visible warning, shown in the preview before submit, not a blocked close.
6. **A dangling `,`/`!` is an editing state.** capture-parse reports a need rather than
   an error, and the Mac card stays live, but strict capture still rejects it.
7. **No ranges** (`=x1-3`). Sessions hold a handful of links. The malformed-list message
   teaches commas.
8. **Link forms accept the selection.** `=x` means one thing everywhere. Numbers refer
   to the post-link lineup the preview shows.
9. **Vocabulary:** "in progress", "complete", "deferred", matching the user's words and
   the task status names.

### Out of scope

- Numbers on the `=` start card.
- Completion candidates for numbers.
- Range syntax.
- Obsidian plugin changes.
- Changing how a deferred line's notes are placed (that is manual-flow parity).
- New CLI subcommands or options.

## Phase: selection-planner

Work in `src/native/capture_pomodoro_close/`. Add a `selection.rs` child module,
register it in `mod.rs`, and re-export only the names callers use.

- **Types:**
  - `CloseSelection { in_progress: Option<BTreeSet<u32>>, complete: BTreeSet<u32>, raw: String }`;
  - `TaskLinkMarker { Plain, Deferred, Embedded }`;
  - `TaskLinkOutcome { InProgress, Deferred, Complete }`;
  - `TaskLinkSource { Ledger, Listed, Unlisted }`;
  - `NumberedTaskLink { index: u32, line: usize /* 1-based */, block_link, path_part, block_id, marker, outcome, source }`;
  - `CloseSelectionError { OutOfRange { .. }, ConflictingDuplicate { .. } }` with
    `Display` producing the contract's messages.
- **`number_task_links(contents, running) -> Vec<NumberedTaskLink>`** implements the
  numbering rule. Reuse `sub_bullet_range`, `markdown::fenced_lines`,
  `strip_pomodoro_markers`, `wikilink_tokens`, `bare_plain_link`, `bare_embedded_link`,
  and the `move_only_destination` shape check. Do not write a new wikilink parser. Look
  at `capture_pomodoro_start::list_queued_links` for the sibling pattern, but note that
  the close numbers all depths and includes `[[T]]#`.
- **`apply_close_selection(contents, running, selection) -> Result<(String, Vec<NumberedTaskLink>), CloseSelectionError>`**
  applies the outcome table, validates range and duplicates, and rewrites only the
  changed lines. It must preserve CRLF and a missing final newline.
- **`plan_pomodoro_close(day_path, day_contents, now, vault, selection: Option<&CloseSelection>)`**:
  - always number the lineup, so plain `=x` still reports numbers;
  - apply the selection when one is given, then run the existing pipeline on the
    rewritten contents;
  - after the task effects, emit the listed-status warnings;
  - add `task_links` to `PomodoroCloseSummary` and `index: Option<u32>` to
    `PomodoroCloseTask`. A row gets an index when it was registered from a numbered
    line's `ledger_line`/`block_id` and its role is not `Subtask`;
  - add `PomodoroClosePlanError::Selection(CloseSelectionError)`.

  Update the three callers in `src/native/capture/pomodoro_close.rs` to pass `None`, and
  make `map_close_plan_error` handle the new variant. No user-visible behavior changes
  in this phase.

**Unit tests** (in `capture_pomodoro_close/tests.rs` or a new `selection_tests.rs`) must
assert, not comment:

- Numbering on the worked example (1 = line 6 plain, 2 = line 10 deferred; struck and
  note unnumbered).
- Nested bare links numbered.
- `![[T]]#`, aliases, 🍅-prefixed links, and fenced lines.
- Mixed lines unnumbered.
- Every outcome-table row.
- The ledger post-images for `=x2`, `=x!1`, `=x1!2`, `=x0`, and `=x1,2`, byte for byte
  as above.
- `None` byte-identical to today's `plan_ledger_close` output.
- Out of range (none, one, many, and multiple bad numbers) and conflicting versus
  same-outcome duplicates.
- Listed Blocked/Done warnings.
- CRLF and no final newline.

In `linked_task_tests.rs`, assert the `bob.md` post-images for `=x!1` and `=x1!2`.

Run `cargo clippy --all-targets --all-features` (no new warnings in touched code) and
`cargo test`. Format only what you touch.

## Phase: selection-grammar

- **Model:** extend `PomodoroCloseSpec` (`capture_language/model.rs`) with
  `in_progress: Option<Vec<u32>>` and `complete: Vec<u32>`, both sorted ascending, and
  serialize them in the capture-parse JSON as specified. Add one pure lexer, for example
  `lex_close_selection(after_x: &str, base_offset) -> Result<CloseSelectionLex, CloseSelectionLexError>`.
  It returns the lists, the byte ranges for the `<N>` and `!<M>` spans, and either a
  typed error with its range or `Incomplete { separator, range }`. Share it between
  every caller below, and keep the diagnostic text in `markers.rs`.
- **Execution grammar:**
  - `session_equals_token`/`parse_pomodoro_equals_item` (`item.rs`): the
    selection-shaped close claims the item, including the no-spaces hint for
    `=x 1,3`-style near misses;
  - `parse_colon_link_tail` (`tokens.rs`): the `x…` suffix;
  - all existing conflicts are unchanged;
  - `@@` keeps skipping close items;
  - every exhaustive match and the parity test
    `editor_agrees_with_execution_for_resolved_captures` (in
    `capture_language/tests/editor_modes.rs`) must cover the new inputs.
- **Editor grammar:** `parse_editor_close_item` (`editor_pomodoro.rs`), the link close
  suffix in `editor_classify.rs`, and `editor_parse.rs` need the new span kinds
  (`SpanKind::PomodoroCloseInProgress`, `SpanKind::PomodoroCloseComplete`), the
  incomplete state (mode `incomplete`, need `pomodoro_close_task`, partial spec,
  placeholder span), and precise diagnostic ranges.
- **Temporary executor guard:** in `src/native/capture/pomodoro_close.rs`, fail a close
  whose spec has `in_progress.is_some() || !complete.is_empty()` with
  ``task numbers after `=x` are not supported by this build yet``, so a parsed selection
  is never silently ignored. selection-capture removes this guard.
- **`capture-parse`** (`capture_parse.rs`): JSON fields, human `close` line, and help
  text (modes/needs lists, the prose around line 139, and examples `=x1,3!2`, `=x0!2`,
  `^r:id=x1`).
- **`capture-complete` / `capture-rewrite`:** a cursor inside the whole close token
  returns an empty success; close items are never rewritten.

**Tests.** Unit tests plus protocol tests in `tests/cli/capture/parse_pomodoro.rs`,
`complete_editor.rs`, and `rewrite.rs`, pinning at least these inputs:

- valid: `=x`, `=X`, `=x1`, `=x1,3`, `=x3,1` (sorted), `=x!2`, `=x1,3!2`, `=X1!2`,
  `=x0`, `=x0!2`;
- every lexical-diagnostic row, with its range;
- incomplete: `=x1,`, `=x!`, `=x1!`, `=x!2,`;
- prose: `=xx`, `=xa`, `Plan =x1`;
- link forms: `@r:id=x1!2`, `^r:id=x1`, `Text @r:id=x!1`, `^r:id#n=x1` (the `#name`
  conflict), and `^r:id=x1,` (incomplete);
- `Text @r:id=x1 s:2` (schedule conflict);
- a multi-item draft mixing `+5`, `=x1!2`, and a task;
- a `@@` draft with `=x1`;
- the executor guard's error.

Run `cargo clippy --all-targets --all-features` and `cargo test`.

## Phase: selection-capture

- Remove the guard. Convert the spec into a `CloseSelection` and pass it to
  `plan_pomodoro_close` from `plan_pomodoro_close_item`,
  `plan_pomodoro_close_link_item`, and `plan_pomodoro_close_task_item`. On link forms,
  pass the post-link staged contents, as today.
- **JSON** (`capture/output.rs`, `build_close_summary_json`): `in_progress`, `complete`,
  `task_links`, and `tasks[].index`, with explicit nulls.
- **Human output** (`print_human_pomodoro_close_success`): the index column and styling,
  byte-identical output when nothing is numbered, and the `carries 1 link` plural fix.
  Also fix existing human-output assertions that include `carries 1 links`.
- **Integration tests**: add a new `tests/cli/capture/pomodoro_close_selection.rs`,
  registered in `tests/cli/capture/mod.rs`, that reuses `close_worked_vault`,
  `run_close_json`, and `run_close_expect_error`. Every case must be asserted. None may
  be left as a comment or with `|| true`.
  - Every worked-example row: day-file and `bob.md`/`sase.md` post-images plus JSON
    (`raw`, `in_progress`, `complete`, `task_links` entries, `tasks[].index`/`role`/
    status fields, `carried`, `next_pomodoro`).
  - Plain `=x`: files byte-identical to before this epic; JSON now carries `task_links`
    with `source: "ledger"`, `in_progress: null`, and `complete: []`.
  - The human output for `=x1!2`, plain and `--dry-run`.
  - `--dry-run` JSON identical to a real run, with nothing written.
  - Link forms: `^bob:ready=x3` (post-image above), `^bob:capture-stop=x!1`
    (`already_current`, completing it), and `Draft docs @bob:draft-docs=x0` (the new
    task deferred).
  - Batches: `-2`, blank, `=x1!2` (decrements once); `=x0`, blank,
    `^sase:recovery-panel=` (switch); `-2`, blank, `=x5` (rolls back the `-2`).
  - Diagnostics: out of range (`=x3`, and `=x1` with no numbered links), a conflicting
    duplicate (build a fixture with the same link on two lines), listed Blocked warnings
    (in progress and complete), every lexical row at least once through `bob capture`,
    and the incomplete `=x1,` error.
  - CRLF day file.

Run `cargo clippy --all-targets --all-features` and `cargo test`.

## Phase: selection-docs

- **`bob capture --help`** (`capture/cli.rs`):
  - the close paragraph gains the selection grammar, the `0` rule, and the outcomes;
  - add examples `bob capture '=x2'`, `bob capture '=x1!2'`, `bob capture '=x0'`, and
    `bob capture '^bob:capture-stop=x!1'`;
  - note that the argument must be single-quoted, because both zsh `=` expansion and `!`
    history expansion apply.
- **`docs/capture.md`:**
  - grammar table rows for `=x<N>`, `=x!<M>`, and `=x<N>!<M>`;
  - the examples table: add `=x1,3!2` and `=x0`, move `=x!` from prose to incomplete,
    and keep `=xx`;
  - in "Closing the running Pomodoro", add a subsection "Choosing each Task Link's
    outcome" with the numbering rule, the outcome table, the worked examples (at least
    `=x2`, `=x1!2`, and `=x0` post-images), the diagnostics, and the manual-flow
    equivalence;
  - update the capture-parse contract (spec fields, spans, the `pomodoro_close_task`
    need, diagnostics, human `close` line), the capture JSON field notes, the human
    output description, and capture-complete;
  - keep the Contents list accurate.
- **`README.md`:** the grammar rows near the existing `=x` rows and the close paragraph.

Verify every documented example against the built binary.

## Phase: mac-selection-preview

Open the linked repo with `/sase_repo`: `sase repo open bob-mac-capture -r "<reason>"`.
If that checkout is unavailable on the host, use
`sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`. Use only the path it prints,
and read its `AGENTS.md` if present.

- **Decode** (`Sources/CaptureCore/CaptureModels.swift`):
  - `PomodoroCloseSpec.inProgress: [Int]?` and `.complete: [Int]`;
  - `PomodoroCloseSummary.inProgress`, `.complete`, and
    `.taskLinks: [PomodoroCloseTaskLink]` (`index`, `ledgerLine`, `blockLink`,
    `blockID`, `marker`, `outcome`, `source`);
  - `PomodoroCloseTask.index: Int?`;
  - every one of these tolerates absence.
- **Presentation** (`CapturePomodoroClosePresentation.swift`):
  - `TaskRow.index`, `.outcome`, `.source`, `.badgeSymbolName`, and `.isDimmed`;
  - struck text for complete rows;
  - the hint as structured tokens (text + color role) plus a joined string;
  - the summary;
  - the pending text;
  - the visible-row rule;
  - status, notification, and accessibility strings exactly as in the design section.
- **View** (`CapturePanelView.swift` `closePreviewItem`/`closeTaskGlyph`): the badge
  column, dimming, the hint/summary row, and the pending style. **Palette**
  (`CompletionRowContent.swift`, `CaptureEditorPalette.swift`): the two new span
  categories.
- **Model** (`CapturePanelModel.swift`):
  - the pending flow: detect the `pomodoro_close_task` need, preview the draft with the
    placeholder range removed, mark the preview pending, and disable Close;
  - make sure a stale pending card can never be submitted;
  - older Bob that never reports the need behaves as today.
- **Fixtures**: build bob-cli at the selection-capture commit and regenerate them from
  real `bob` against the worked-example vault. Name the vault path as in the existing
  close fixtures. Add:
  - `pomodoro-close-select-worked.json` (`=x2`);
  - `pomodoro-close-select-complete.json` (`=x1!2`);
  - `pomodoro-close-select-none.json` (`=x0`);
  - `pomodoro-close-select-out-of-range.json` (the `=x3` failure);
  - parse fixtures for `=x1,3!2`, `=x1,` (incomplete), and `=x1,1` (invalid);
  - regenerate `pomodoro-close-worked.json`, which now carries `task_links`.

  Extend `Tests/Fixtures/fake-bob` for the panel-model tests.

- **Tests** (`CapturePomodoroClosePresentationTests`, `CaptureModelTests`,
  `CompletionRowContentTests`, `CapturePanelModelTests`):
  - badges, symbol names, and the fallback above 50;
  - outcome tints and roles, and dimming of unlisted rows;
  - hint strings for 1 and 2+ rows; summary strings, including `=x0`;
  - numbered rows never overflow;
  - status, notification, and accessibility strings;
  - decoding with fields absent (older Bob) and present;
  - span-category mapping;
  - the pending flow: the trimmed-draft preview runs, Close is disabled, the real draft
    is never submitted, and a valid draft restores the normal card.
- **README**: the grammar support list, the close-card description, and the pending-list
  behavior.
- **Validation:** Swift is not available on Linux hosts. If a macOS toolchain is
  available, run `just all`. Otherwise record that in the phase notes. After the commit
  reaches the repo, confirm the macOS CI run for that commit
  (`gh run list -R bobs-org/bob-mac-capture`) and fix any failure. The epic's land step
  must see that CI run green.
