---
tier: epic
title: Named Pomodoro starts with `=<X>#pomodoro` in `bob capture` and Bob Mac Capture
goal: 'A whole capture item `=<X>#<pomodoro>` (for example `=#deep-work`, `=3#bugs`,
  `=-2#bugs`) starts the named Pomodoro now with `se<X>` timing, atomically. It starts
  the open placeholder whose name matches (whole slug, else prefix), or a new session
  named like a completed match, or a brand-new named session. The token composes in
  same-line chains (`=x =#bugs` switches sessions in one line). `bob capture-parse`
  and `bob capture-complete` expose it, and Bob Mac Capture completes the name with
  a start-aware list, highlights it, previews the session live, and teaches the syntax.

  '
phases:
- id: execution
  title: '`=<X>#pomodoro` grammar, chains, and named session start in `bob capture`'
  depends_on: []
  size: medium
  description: 'execution: extend the shared `=`-family lexer with an optional `#name`
    part, claim `=<X>#name` (and the `=x#name` near miss) as whole-item session tokens
    and chain tokens, plan the named start (resolve, guard, start or create, move
    to the current slot, report queued links), and update JSON, human output, and
    `bob capture --help`.

    '
- id: editor
  title: Named starts in `bob capture-parse`
  depends_on:
  - execution
  size: medium
  description: 'editor: mirror the named start in the live-editor parser. Report mode,
    `section`, spans, the `=<X>#` incomplete state, and every near-miss diagnostic
    with precise ranges. Keep `@@` away from named starts and extend the editor/execution
    parity tests.

    '
- id: completion
  title: '`pomodoro_start_name` completion context in `bob capture-complete`'
  depends_on:
  - editor
  size: medium
  description: 'completion: add the `pomodoro_start_name` context for the name part
    of `=<X>#name`, and return start-aware candidates from today''s ledger: planned
    placeholders with `next_up`, a create row, "again" rows for completed sessions,
    nameable rows, and the running entry last. Include help, human output, and tests.

    '
- id: docs
  title: Capture docs and README for named starts
  depends_on:
  - execution
  - editor
  - completion
  size: small
  description: 'docs: document `=<X>#pomodoro` in `docs/capture.md` (grammar tables,
    lifecycle, a new "Starting a named Pomodoro" section, chains, capture-parse, and
    capture-complete) and in `README.md`, all consistent with the shipped behavior.

    '
- id: mac
  title: Bob Mac Capture support for named starts
  depends_on:
  - execution
  - editor
  - completion
  size: medium
  description: 'mac: add the `pomodoro_start_name` completion rows (planned, next
    up, new, again, name it, running) with accept announcements. Add a calm incomplete
    state for `=#`, a New badge and a `#name` teaching hint on the start card, and
    real-bob fixtures, tests, and README. Verify with a green macOS CI run.

    '
proposed_by: bbugyi200.apollo.36.w0
create_time: 2026-09-29 19:17:56
status: done
bead_id: bob-cli-2p
---

- **PROMPT:** [prompts/202609/named_pomodoro_start.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/named_pomodoro_start.md)
- **BEAD:** [bob-cli-2p](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2p/README.md)

# Plan: Named Pomodoro starts with `=<X>#pomodoro`

## Background

Today a whole capture item `=`/`=<X>` starts **today's next future Pomodoro**. That is
the first open, untimed `- [ ] ()` placeholder in document order. It uses Obsidian's
`se<X>` timing: empty is 25 min, `3` is 15 min, and `-2` is 25 min with a 10-minute
offset. There is no way to say _which_ session to start without a task
(`^route:id#name=`). Bryan wants `=#foo` to start the Pomodoro named `FOO`, and
`=<X>#foo` to do the same with `se<X>` timing.

Relevant code (bob-cli, all paths repo-relative):

- **Lexer and whole-item parser**, `src/native/capture_language/item.rs`:
  - `session_equals_token` returns
    `EqualsToken::{Close, Start { suffix, counted, len }}`.
  - `parse_pomodoro_equals_item` claims exact starts. It returns
    `pomodoro_start_shape_error` for counted near misses and `Ok(None)` (prose) for a
    bare token followed by text.
  - `is_session_chain_token` is the chain predicate. It must equal "the whole-item
    parsers claim this token".
    `tests/chain.rs::chain_token_predicate_equals_claimed_for_single_tokens` enforces
    that.
- **Other grammar files**, same directory:
  - `tokens.rs`: `parse_pomodoro_start_suffix` rejects `=`, `#`, and `:`, so split the
    name off before calling it. `is_pomodoro_selector_component` is the name charset.
  - `markers.rs`: error constants.
  - `model.rs`: `CaptureKind::PomodoroStart { spec }` and
    `PomodoroStartSpec { raw, duration_units, offset_units }`.
  - `draft.rs`: `session_chain_tokens` and `push_capture_item` split a chain line into
    single-token items.
- **Execution**, `src/native/capture/pomodoro_start.rs`:
  - `plan_pomodoro_start_item` is the whole-item start. Its guards are: missing day
    file, no section, one running timed entry, several running timed entries, and no
    future placeholder.
  - Helpers: `compute_pomodoro_start_range`, `start_existing_pomodoro_entry` (rewrites
    `()` to the canonical range, then `move_started_pomodoro_to_current_slot`), and
    `create_started_pomodoro_entry` (always appends a child link, so it does not fit
    here).
  - The queued-link lineup comes from
    `capture_pomodoro_start::{list_queued_links, resolve_queued_links}`.
  - Dispatch is in `src/native/capture/plan.rs` (around line 95). It does not pass
    `warnings` to the start planner today.
- **Named-Pomodoro helpers**:
  - In `src/native/capture_pomodoros.rs`:
    - `scan` produces
      `PomodoroEntry { line, state, name, slug, selectable, time_range, placeholder, is_current, child_count, pomodoro_ref }`.
    - `select_named` returns `Found`, `CompletedOnly`, or `Missing { suggestion }`. It
      tries open entries first, then completed ones, each by whole slug and then by
      first prefix. `suggestion` is a unique Levenshtein ≤ 2 match among open entries.
    - Also used: `next_future_pomodoro`, `named_creation_name`,
      `canonicalize_pomodoro_name`, and `POMODORO_NAME_USAGE`.
  - `src/native/capture_task_toggle.rs::insert_named_placeholder` inserts
    `- [ ] () — NAME` at the current slot and returns `(contents, created_line, name)`.
- **Output**:
  - `src/native/capture/output.rs::print_human_pomodoro_start_success` never prints
    `(created)`.
  - `PomodoroStartSummary` lives in `src/native/capture/batch.rs`. Its fields are
    `start, end, duration_minutes, offset_units, pomodoro_name, pomodoro_line, created_pomodoro, time_range, tasks`.
  - Batch-level `warnings` print to stderr in human mode and appear as top-level JSON
    `warnings`.
- **Editor**, `src/native/capture_language/`:
  - `editor_pomodoro.rs::{parse_editor_close_item, parse_editor_start_item, editor_start_invalid_outcome}`.
  - `editor_classify.rs`: the `=x` with `#name` message is around line 657, and the
    `@route:id#` incomplete pattern (`interactive_placeholder` over `#` plus
    `needs: ["pomodoro_name"]`) is in `marker_parse`.
  - `editor_parse.rs` (around 246) skips `@@` inheritance for session modes and
    `pomodoro_close.is_some()`.
- **Completion**:
  - `completion.rs::completion_field_at` returns `None` early for any item that
    `parse_editor_adjust_item` or `parse_editor_close_item` claims. That covers every
    start today.
  - In `src/native/capture_complete.rs`:
    - `pomodoro_name_candidates_at` → `pomodoro_name_candidates_from_scan` builds open
      named rows, then nameable rows, plus a create row via `named_creation_name`.
    - `PomodoroNameCandidate` is the JSON row.
    - `SCHEMA_VERSION` is 1.
- **Help**: `src/native/capture/cli.rs` (`long_about` lifecycle and start blocks,
  `after_help` examples), `src/native/capture_parse.rs`, and
  `src/native/capture_complete.rs`.
- **Docs**: `docs/capture.md` and `README.md`.

Current behavior of the new inputs, from HEAD:

| Input      | Today                                                          |
| ---------- | -------------------------------------------------------------- |
| `=#foo`    | Prose. Writes the task `=#foo` to `mac_inbox.md`.              |
| `=3#foo`   | Error: ``Pomodoro start `=3` must be the whole capture item…`` |
| `=#`       | Prose task `=#`.                                               |
| `=x#foo`   | Prose task.                                                    |
| `=x =#foo` | Not a chain. The `=x` shape error applies.                     |
| `=#foo +2` | Prose task.                                                    |

Bob Mac Capture (separate repo) never parses grammar. It colors Bob's `capture-parse`
spans and requests `capture-complete` only when the caret is in a completion span kind
(`pomodoro_name` and `interactive_placeholder` are included; `pomodoro_start` is not) or
when `needs` includes `pomodoro_name`. It routes context `pomodoro_name` to the inline
completion list (Create row, Name-it prompt). It renders whole-item starts with a start
card (`CapturePomodoroStartPresentation`, `startPreviewItem`) that shows no "created"
indicator.

## Design

### Grammar

A **named start token** is `=` + an `se<X>` suffix (`[0-9]*(-[0-9]*)?`, possibly empty)

- `#` + a **name part**. The name part is every byte after that `#` up to the next ASCII
  whitespace or the end of the line. The `#` must come immediately after the suffix.

* A named start claims its item. It is never prose.
  - An exact token (the item's only text, one physical line) is a named start.
  - Anything else is a precise error.
* The name part is a Pomodoro selector: it is slugged (`selector_slug`) and matched
  exactly as `#name` works elsewhere. Its charset is `is_pomodoro_selector_component`
  (`A-Z a-z 0-9 & ' ( ) + , . / -`). Spaces in names are written as `-`.
* `<X>` always comes **before** `#`. The link forms' `#name=<X>` order is not accepted
  here, and it gets a teaching error.
* The token is a same-line **chain token**, so `=x =#bugs`, `=#bugs +2`, and
  `=3#bugs =x` chain. Spacing stays significant: `=#bugs+2` is a start of the Pomodoro
  named `BUGS+2`, because `+` is name charset.
* `=x#…` (the `=x` close immediately followed by `#`) becomes a claimed close near miss
  with a teaching error. It used to be prose.
* These are unchanged:
  - `=`/`=<X>` without `#`, and every close form.
  - `= #foo`, which stays whatever it is today because a bare `=` with prose remains
    prose.
  - Mid-body `Plan =#foo`, which stays prose.
  - Link forms such as `^route:id#name=<X>`.
* A `@@` declaration never applies to a named start. Forced
  `--route`/`--section`/`--task`/`--task-ref`/`--task-section`/`--clip` fail with the
  existing `POMODORO_START_FORCED_ERROR`.

### Semantics

The planner runs on the staged day file, so chains and batches see earlier edits. Let
`token` be the typed token, for example `=3#bugs`. Let `selector` be the name part, and
let `NAME` be the display name:

- For `Found`: `entry.name`.
- For `CompletedOnly`: that entry's canonical name.
- Otherwise: `canonicalize_pomodoro_name(selector)`, falling back to `` `#selector` ``
  when it cannot be canonicalized.

The checks run in this order:

1. **Resolve** `select_named(&scan, selector)` first, so that every message can name
   `NAME`.
2. **Day file.** A missing day file fails with R1. A missing `## Pomodoros` section
   fails with the existing `Bob daily note has no Pomodoros section`.
3. **Running guard.** Consider the open timed entries.
   - More than one fails with R5.
   - Exactly one that _is_ the `Found` entry fails with R3.
   - Exactly one other entry fails with R4, which teaches the one-line switch
     `=x <token>`.
4. **Start.**
   - **`Found(entry)`** and entry is an open untimed placeholder: call
     `start_existing_pomodoro_entry`, which rewrites `()` and moves the entry, with its
     child block, to the current slot. This gives `created_pomodoro: false`. A `Found`
     entry that is not an untimed placeholder fails with the existing "selected Pomodoro
     `#slug` is not an untimed open placeholder…" wording.
   - **`CompletedOnly(entry)`**, the "again" case: create a new session named
     `canonicalize_pomodoro_name(entry.name)`. Fall back to canonicalizing the selector
     when that fails. The completed history is never modified. This gives
     `created_pomodoro: true`.
   - **`Missing { suggestion }`**: create a new session named
     `canonicalize_pomodoro_name(selector)`. An invalid name fails with R6. This gives
     `created_pomodoro: true`. When `suggestion` is `Some(open entry)`, also push
     warning W1.
   - **Creating** means calling `insert_named_placeholder(staged, NAME-or-selector)` and
     then `start_existing_pomodoro_entry` on the created line. The placeholder is
     inserted after the last completed entry's block, otherwise before the first entry,
     otherwise at the section start. The move is then a no-op. This is exactly where the
     existing named link start places a created entry. Map `LinkRelocationError` to
     `CaptureError` the way `pomodoro_link.rs::link_plan_error` does.
5. **Report** the started entry's direct-child Task Links through the existing
   `list_queued_links` / `resolve_queued_links` lineup. A created entry reports
   `tasks: []`.

The existing bytes contract holds: the checkbox, name, children, CRLF, and a missing
final newline are all preserved. Dry-run computes the same result without writing, and
any failure rolls the whole batch back.

#### Decision flagged for review: "again" reuses the completed session's name

`=#deep-work` after a completed `DEEP WORK` creates `DEEP WORK`, not `DEEP-WORK`. `=#pl`
after a completed `PLAN` creates a new `PLAN`, just as `=#pl` starts an open `PLAN`.
Link forms (`@r:id#name`) keep today's rule and create the selector's canonical name.
Aligning them is listed under follow-ups. If Bryan prefers the link-form rule, only step
4 `CompletedOnly` and the completion "again" rows change.

### Worked example

This example uses `BOB_NOW=2026-07-10 09:02:00`, day file `2026/20260710.md`, and TAB
indentation:

```markdown
# 2026-07-10

## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[sase#^plan-day]]
- [ ] () — BUGS
  - [[sase#^deep-fix]]
- [ ] () — DEEP WORK
  - [[bob#^outline]]
  - [[bob#^draft]]
- [ ] ()
  - [[bob#^inbox-zero]]
```

`PLAN` is on line 5, `BUGS` on line 7, `DEEP WORK` on line 9, and the unnamed
placeholder on line 12.

| Item          | Result                                                                                                                            |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `=#deep-work` | `- [ ] (**0905-0930** [t:: 25m]) — DEEP WORK` and its two links move to line 7, before BUGS. Queued: `^outline`, `^draft`         |
| `=#deep`      | Identical (prefix match)                                                                                                          |
| `=3#bugs`     | BUGS becomes `(**0905-0920** [t:: 15m])` and stays on line 7                                                                      |
| `=-2#bugs`    | BUGS becomes `(**0855-0920** [t:: 25m])`                                                                                          |
| `=#plan`      | New `- [ ] (**0905-0930** [t:: 25m]) — PLAN` at line 7, `created_pomodoro: true`, nothing queued. The completed PLAN is untouched |
| `=#review`    | New `REVIEW` at line 7, created                                                                                                   |
| `=#bgus`      | New `BGUS` at line 7, created, plus warning W1 naming `BUGS`                                                                      |
| `=`           | Unchanged: starts BUGS                                                                                                            |
| `=#`          | E1 incomplete                                                                                                                     |
| `=#bugs=3`    | E3: put the duration first                                                                                                        |
| `=x#bugs`     | E5                                                                                                                                |
| `=#deep work` | E4 with the join-with-`-` hint                                                                                                    |

Variant: BUGS is instead running as `- [ ] (**0840-0905** [t:: 25m]) — BUGS`.

- `=#deep-work` fails with R4 and teaches `=x =#deep-work`.
- `=#bugs` fails with R3.
- `=x =#deep-work` closes BUGS, then starts DEEP WORK, atomically.
- `=#deep-work +2` starts DEEP WORK and then extends it, but only after an earlier `=x`.

Human output (`NO_COLOR`) for a start of an existing entry and for a created one:

```text
✓ started  2026/20260710.md
  BUGS 0905-0920 (15m) at line 7
  - [ ] (**0905-0920** [t:: 15m]) — BUGS
  [ ] Deep fix sase.md ^deep-fix
```

```text
✓ started  2026/20260710.md
  PLAN 0905-0930 (25m) (created) at line 7
  - [ ] (**0905-0930** [t:: 25m]) — PLAN
  nothing queued
```

`(created)` matches the wording the link-form start already uses
(`would start NEWTHING 0905-0920 (15m) (created) at line 7`). Dry-run uses
`would start`, as today.

### Error and warning texts

These texts are fixed across phases, and the editor diagnostics reuse them. `{token}` is
the typed token, `{suffix}` is `<X>`, and `{name}` is the typed name part.

- **E1** (incomplete, `=#` / `=3#`):
  `` `{token}` is incomplete: type a Pomodoro name after `#` (for example `{token}deep-work`) ``
- **E2** (invalid name characters):
  `` Pomodoro name `{name}` in `{token}` may contain only A-Z, a-z, 0-9 or `& ' ( ) + , . / -`; write spaces as `-` ``
- **E3** (link-form order): the name contains `=`, `{suffix}` is empty, and the text
  after the first `=` in the name is a valid `se<X>` suffix:
  `` write the duration before the name: `={after}#{before}` instead of `{token}` ``
- **E4** (extra text or child lines):
  ``Pomodoro start `{token}` must be the whole capture item; remove extra text, markers, or child lines (to start a task's session in a named Pomodoro instead, use `^route:block-id#{name}={suffix}`)``
  - When the extra text is one or more words that are all valid name components, add
    `` ; to name a multi-word Pomodoro, join the words with `-`: `={suffix}#{name}-{w1}-…` ``.
  - When the name is empty and extra text follows (`=# bugs`), use this instead:
    `` write the Pomodoro name right after `#`, with no space: `={suffix}#{w1}` ``.
  - As with the close, a broken token reports its own error first: E1 through E3, or the
    existing overflow error.
- **E5** (`=x#…`):
  `` `=x` always closes the running Pomodoro; remove `#{name}`, or write `=x =#{name}` to close it and then start that Pomodoro ``
  - When the name is empty, use `` remove `#` `` and drop the second clause.
- **R1**: ``cannot start {NAME}: today's daily note `{rel}` does not exist``
- **R3**:
  ``{NAME} is already running ({HHMM-HHMM} at line {N}); use `+N`/`-N` to resize it, `++N`/`--N` to shift it, or `=x` to close it``
- **R4**:
  ``cannot start {NAME}: {RUNNING HHMM-HHMM | the current session HHMM-HHMM} is still running at line {N}; close it with `=x` first, or capture `=x {token}` to switch sessions``
- **R5**:
  `cannot start {NAME}: today's ledger has multiple open timed Pomodoros; finish all but one first`
- **R6**: ``cannot create Pomodoro `#{selector}`:`` followed by `POMODORO_NAME_USAGE`.
- **W1** (warning, not an error):
  ``no open Pomodoro matches `#{selector}`; created {NAME} (did you mean {SUGGESTION}? use `={suffix}#{suggestion-slug}`)``
- **Bare-start discovery.** The existing bare `=` "no future Pomodoro to start" error
  gains a pointer. The parenthetical becomes
  ``(start a new named session with `=#<name>`, or a task's session with `^route:block-id=`)``.
  Update the tests that assert the old text.

Execution errors keep the `capture item K starting on line L: …` prefix. Editor
diagnostics use code `invalid_pomodoro_start` (E2–E4) or `invalid_pomodoro_close` (E5).
E1 is not a diagnostic in the editor; it is the incomplete state.

### Shared contracts

These names and shapes are fixed across phases, and bob-mac-capture depends on them.
Every change is additive: `schema_version` stays 1 everywhere.

- **Lexer.** `session_equals_token` stays the single lexer. `EqualsToken::Start` gains
  the optional name part: the byte offset of `#` within the token and the name text.
  Execution, editor, completion, and the chain predicate all consume that one lexing. An
  internal helper for the full token length, including the name part, is fine.
- **Execution kind.**
  `CaptureKind::PomodoroStart { spec, pomodoro_name: Option<String> }` carries the typed
  selector, or `None` for `=`/`=<X>`. `PomodoroStartSpec` is unchanged, and `raw` stays
  the `<X>` suffix only.
- **`bob capture` JSON.**
  - kind `pomodoro_start`, `placement: "started"`
  - `text` is the whole token, for example `=3#bugs`
  - `task_line` is the started ledger line
  - top-level `pomodoro_name` is the resolved visible name
  - `pomodoro_start` is the existing object with `pomodoro_name`, `pomodoro_line`,
    `created_pomodoro` (true for new sessions and "again" sessions), and `tasks` (always
    present)
  - top-level `warnings` carries W1
  - No new keys.
- **`bob capture-parse`**:
  - Exact named start:
    - mode `pomodoro_start`
    - `body` = the token
    - `section` = the typed selector (the existing convention for Pomodoro names on
      `pomodoro_task`/`pomodoro_link`)
    - `pomodoro_start` = the spec (for example
      `{"raw":"3","duration_units":3,"offset_units":0}`)
    - spans: `pomodoro_start` over `=<X>`, and `pomodoro_name` over the name bytes only.
      The `#` is in no span, exactly as on `@route:id#name`.
  - `=<X>#` with an empty name:
    - mode `incomplete`, `needs: ["pomodoro_name"]`
    - spans: `pomodoro_start` over `=<X>`, and `interactive_placeholder` over `#`
    - the partial `pomodoro_start` spec
    - no diagnostic
  - Near misses: mode `pomodoro_start`, the same spans as far as typed, and one
    `invalid_pomodoro_start` diagnostic:
    - E2 and E3: ranged over the name.
    - E4: ranged over the extra text, or over the child line.
    - Overflow: ranged over `=<X>`.
    - No spec.
  - `=x#name`: mode `pomodoro_close`, a `pomodoro_close` span over `=x`, and an
    `invalid_pomodoro_close` diagnostic over `#name`.
  - No new modes, needs, span kinds, or diagnostic codes.
- **`bob capture-complete`**:
  - A new context, `pomodoro_start_name`. A cursor from just after `#` to the end of the
    name part on a named start item, or on the `=<X>#` incomplete item, completes the
    name.
    - `replacement` is the name range. It is empty at the cursor for `=#`, and `#` is
      never inside it.
    - A cursor on `=<X>` or at the `#` byte itself returns an empty success. This keeps
      the typed suffix safe.
    - This works per token inside chains.
  - Candidates reuse the `PomodoroNameCandidate` JSON shape plus one additive field,
    `next_up: true` (omitted when false). It marks the row whose entry is
    `next_future_pomodoro(&scan)`, the entry a bare `=` would start.
  - Row kinds are identified by existing fields:

    | Row     | `requires_name` | `creates_pomodoro` | `state`       | `time_range` | Meaning                                                 |
    | ------- | --------------- | ------------------ | ------------- | ------------ | ------------------------------------------------------- |
    | start   | false           | false              | `"open"`      | null         | Start this planned placeholder now                      |
    | new     | false           | true               | `"open"`      | null         | Create and start this name (no `ref`/`line`)            |
    | again   | false           | true               | `"completed"` | the range    | Start a new session named like this completed one       |
    | name it | true            | false              | `"open"`      | null         | Unnamed or untypeable placeholder; prompt for a name    |
    | running | false           | false              | `"open"`      | the range    | Already running; starting it fails unless `=x` precedes |

  - The existing `pomodoro_name` context and its candidates are **unchanged**.

### Completion ordering (`pomodoro_start_name`)

Candidates come from today's day file only, read-only, through
`capture_pomodoros::scan`. Missing-note, missing-section, and scan warnings behave
exactly as in `pomodoro_name_candidates_at`. Rows appear in this order:

1. **Start rows.** Include open, untimed, placeholder, selectable entries.
   - Deduplicate by slug; the first in document order wins, and `match_count` counts
     same-slug open entries.
   - Rank with the existing prefix-then-substring `rank` on the slug. An empty query
     keeps document order, so the next-up row leads.
2. **New row.** Add it only when the query is non-empty, `select_named` returns
   `Missing`, the name canonicalizes, the section exists, and at most one open timed
   entry exists.
   - Fields: `replacement` = canonical slug, `name` = canonical name.
   - Insert it with the existing `insert_pomodoro_creation_candidate` rule over the
     combined list.
3. **Again rows.** Include completed selectable entries whose slug is not an open
   entry's slug.
   - Deduplicate by slug, keeping the **latest** in document order.
   - Rank prefix-then-substring. An empty query puts the most recent first.
   - Fields: `replacement` = slug, `name` = canonical name, plus that entry's `ref`,
     `line`, and `time_range`.
4. **Name-it rows.** Include open untimed placeholders that are not selectable, with
   `requires_name: true` and an empty replacement. The query never filters them, as
   today. One can carry `next_up: true`.
5. **Running row.** Include the open timed entry, if selectable, last. It is never the
   default selection.

On the worked example with an empty query, the rows are: BUGS (start, `next_up`, 1
link), DEEP WORK (start, 2 links), PLAN (again, `0830-0855`), and the unnamed entry
(name it, 1 link). More queries:

- `de`: DEEP WORK, then the name-it row.
- `pl`: PLAN (again), then the name-it row. There is no new row, because the match is
  `CompletedOnly`.
- `rev`: REV (new), then the name-it row.

When BUGS is running, the rows are: DEEP WORK (`next_up`), PLAN (again), name it, BUGS
(running).

The human output of `bob capture-complete` labels rows as follows:

- `next up · 1 link` (singular for one) or `planned · 2 links`
- `new session`
- `again · last 0830-0855`
- `name it · 1 link`
- `running 0840-0905`

### Bob Mac Capture design

The Mac phase follows these principles:

- The app stays a thin client: no grammar and no ledger logic, only wording and visuals
  over Bob's data.
- Typing `=#` opens a completion list of today's sessions immediately.
- Typing a prefix narrows the list while the live preview already shows "Start DEEP WORK
  0905–0930 · 2 queued".
- Accepting a row inserts the slug, and Return then starts the session.
- The syntax colors read as two tones: the pink `=3` start and the purple `bugs` name.

The phase `mac` section has the details.

## Phase `execution`: grammar, chains, and named start in `bob capture`

1. **Lexer** (`item.rs`):
   - Extend `session_equals_token` / `EqualsToken::Start` with the optional name part
     (see Shared contracts).
   - `Close` detection is unchanged. `x` is never suffix or `#`, so `=x#…` still lexes
     as `Close`.
2. **Whole-item parser** (`parse_pomodoro_equals_item`):
   - Start arm, when the name part is present:
     - Exact token and single line: check E1, E3, E2 in that order, then
       `parse_pomodoro_start_suffix(suffix)?` (overflow), then the forced-flag error.
       Then return `CaptureKind::PomodoroStart { spec, pomodoro_name: Some(selector) }`
       with `body` = the token.
     - Otherwise, report the token's own error first (E1–E3 or overflow), then E4 with
       its hints.
   - When the name is absent, behavior is byte-identical to today.
   - Close arm: a first token starting with `=x#`/`=X#` returns E5, whether or not the
     item has extra text or child lines.
   - Add the error builders to `markers.rs` next to `pomodoro_start_shape_error`.
3. **Chain predicate** (`is_session_chain_token`):
   - A start token with a name part is always claimed. So is a close token starting with
     `=x#`.
   - Update the doc comment.
   - Extend both tables in `tests/chain.rs` so the predicate/claim-equivalence test
     covers:
     - claimed: `=#bugs`, `=3#bugs`, `=-2#bugs`, `=#`, `=#bugs=3`, `=#b!`, `=x#bugs`
     - not claimed: `#bugs`, `bugs`
4. **Model and dispatch**:
   - Add `pomodoro_name` to `CaptureKind::PomodoroStart`, and update every match site
     (`plan.rs`, `capture_kind_label`, and any editor/model code that constructs the
     variant).
   - Pass `warnings` into `plan_pomodoro_start_item`.
5. **Planner** (`capture/pomodoro_start.rs`):
   - Implement Semantics steps 1–5 in a pure, unit-testable helper, for example
     `plan_named_session_start(staged_day, selector, time_range) -> Result<NamedSessionStart { updated, moved_index, name, created, suggestion }, CaptureError>`.
   - Call it from `plan_pomodoro_start_item` when `pomodoro_name` is `Some`. The unnamed
     path stays as is, apart from the bare-start discovery text.
   - Build the `CaptureItemResult` exactly as the unnamed path does, with
     `pomodoro_name` = `NAME` and `created_pomodoro` from the helper, and push W1 when
     applicable.
6. **Human output**: `print_human_pomodoro_start_success` prints ` (created)` after the
   duration when `created_pomodoro`. There is no other change. The line matches the
   link-form wording.
7. **Help** (`capture/cli.rs`):
   - In `long_about`, mention `=<X>#<pomodoro>` in the lifecycle paragraph. The mnemonic
     becomes: "`=` starts the next session, `=#name` starts that one, `=x` stops the
     running one".
   - In the whole-item start paragraph, cover: resolution (whole slug, else prefix;
     open, then completed "again"), creation, the switch idiom `=x =#name`, and "`<X>`
     goes before `#`".
   - In `after_help`, add the examples `bob capture '=#deep-work'`,
     `bob capture '=3#bugs'`, and `bob capture '=x =#bugs'` next to the existing `=`
     examples.
   - Update the help smoke test in `tests/cli/help.rs`.
8. **Tests**:
   - Grammar unit tests in `src/native/capture_language/tests/`: claims, E1–E5 exact
     texts, chain splits (`=x =#bugs`, `=#bugs +2`, `=3#bugs =x`, `=# bugs` is not a
     chain and gives the no-space E4), and the unchanged `=`/`=3`/`= foo`/`Plan =#foo`.
   - Planner unit tests (`src/native/capture/tests/`), covering every worked-example
     row:
     - bytes for start, move, created, and "again"
     - CRLF and missing final newline preserved
     - no section, missing day file
     - R3, R4, R5, R6, and W1
   - A new `tests/cli/capture/pomodoro_start_named.rs`, registered in `mod.rs`:
     - JSON and human output for existing, created, and "again" starts
     - `--dry-run` JSON equals the real JSON except `dry_run`, and dry-run writes
       nothing
     - the running variant: R3, R4, and `=x =#deep-work` in one line
     - a batch rollback (`=#bugs`, blank line, an invalid item leaves the file
       untouched)
     - forced-flag rejection
     - W1 on stderr (human) and in `warnings` (JSON)
9. `cargo fmt --check`, `cargo test`, and `cargo clippy --all-targets --all-features`
   are clean, apart from pre-existing warnings. Run `just all` if it is available.

## Phase `editor`: named starts in `bob capture-parse`

1. **`parse_editor_start_item` / `parse_editor_close_item`** (`editor_pomodoro.rs`):
   - Implement the capture-parse contract exactly: spans, `section`, spec, the
     incomplete state with `interactive_placeholder` and `needs: ["pomodoro_name"]`, and
     the near-miss diagnostics with E2–E5 texts and ranges.
   - Reuse the phase `execution` lexer and error builders. Never re-lex.
   - The editor never fails.
2. **`@@`** (`editor_parse.rs`): an incomplete named start never inherits a `@@`
   declaration.
   - Extend the skip condition, for example with the whole-item start's partial
     `pomodoro_start` on an item with no local route.
   - Test `@@work` plus `=#` in one draft.
3. **Chains**: the editor already sees per-token items. Add `tests/chain.rs` editor
   cases:
   - `=x =#bugs`: close span, start span, name span, and items[] ranges
   - `=x =#`: the second item is incomplete
   - `=#bugs +2`
4. **Parity**: extend `editor_agrees_with_execution_for_resolved_captures`
   (`tests/editor_modes.rs`) so named starts agree. Execution's `pomodoro_name` equals
   the editor's `section`, and the spec and mode match.
5. **Human output** of `capture-parse`: the start line shows the whole token, for
   example `=3#bugs (15m, offset 0u)`. `section` keeps its own line.
6. **Help** (`capture_parse.rs` `long_about`): add one sentence on named starts:
   `section`, the name span, the `=<X>#` incomplete state, and the `=x#name` close near
   miss. The Modes and Needs lists are unchanged.
7. **Tests**:
   - `tests/editor_modes.rs`: every contract row.
   - `capture_parse.rs` JSON unit tests.
   - `tests/cli/capture/parse_pomodoro.rs` protocol cases for `=#bugs`, `=3#bugs`, `=#`,
     `=#bugs=3`, `=#deep work`, `=x#bugs`, and `=x =#bugs`.
8. The same gates as phase `execution`.

## Phase `completion`: `pomodoro_start_name` in `bob capture-complete`

1. **Context** (`completion.rs`):
   - Add `CompletionContext::PomodoroStartName`, serialized `pomodoro_start_name`.
   - In `completion_field_at`, before the early `return None` for claimed session items,
     return a field when the cursor's item is a named start (valid, near miss, or
     `=<X>#` incomplete) and the cursor lies in `[name_start, name_end]`:
     - `route: None`, `block_id: None`
     - `query` = the name bytes before the cursor
     - `replacement` = `(name_start, name_end)`
   - Everything else on session items keeps returning `None`.
2. **Candidates** (`capture_complete.rs`):
   - Add `pomodoro_start_name_candidates(bob_dir, query)`. It shares the day-file read
     and warnings with `pomodoro_name_candidates_at`, and implements the ordering above.
   - Add `next_up: bool` to `PomodoroNameCandidate` with
     `skip_serializing_if = "is_false"`. It is always false in the `pomodoro_name`
     context, so that output is byte-identical.
   - Implement the "Missing only" create rule in a small start-specific helper. Do not
     change `named_creation_name`, which link-form completion still uses.
3. **Human output**: add the row labels from the ordering section.
4. **Help** (`capture_complete.rs`): list `pomodoro_start_name` in the Contexts list,
   and describe it in `long_about` next to the Pomodoro-name paragraph. Keep the list's
   existing order convention.
5. **Tests**:
   - `tests/completion.rs` field tests:
     - cursor after `#` on `=#` and `=3#` (empty replacement at the cursor)
     - cursor mid-name on `=#de|ep`: the query is `de` and the replacement is the whole
       name
     - cursor on `=3` or at the `#` byte: `None`
     - chain `=x =#de|`
     - `@r:id#` still returns `pomodoro_name`
   - `capture_complete.rs` candidate tests on the worked-example ledger, covering every
     query listed above, the running variant, a missing note, a missing section, and
     several open timed entries (no new row).
   - `tests/cli/capture/complete_editor.rs` CLI JSON for `=#` and `=#rev`.
   - A regression test that `pomodoro_name` context JSON is unchanged.
6. The same gates as phase `execution`.

## Phase `docs`: capture docs and README

Update `docs/capture.md`, and keep the Contents list in sync:

1. **Grammar at a glance**: add the rows below.
   - Main table: `=<X>#pomodoro`, "Start the named Pomodoro now: an open match (whole
     slug, else prefix) starts in place; a completed match starts a new session with
     that name; otherwise a new named session is created and started. The item must
     contain only the token."
   - `#` table: `=#bugs`, `=3#bugs`, `=#` (incomplete), `=#bugs=3` (E3), `=x#bugs` (E5),
     `=x =#bugs` (a switch in one line), `=#bugs +2` vs `=#bugs+2`, and `=#deep work`
     (join with `-`).
2. **Starting the next Pomodoro**: add a lifecycle-table row for `=<X>#name` and the new
   mnemonic.
3. **New section "Starting a named Pomodoro"** (after "Starting the next Pomodoro"):
   - recognition, resolution, "again", creation and placement
   - guards (R1–R6), W1, JSON/human output, and the zsh quoting note
   - the worked example and its table, including the running variant
4. **Chaining session operators on one line**: add `=x =#bugs` and `=#bugs +2` to the
   token list and examples.
5. **`bob capture-parse`**: named-start spans, `section`, the incomplete state, and the
   `=x#name` near miss.
6. **`bob capture-complete`**: the `pomodoro_start_name` context, the five row kinds,
   `next_up`, the ordering, and the unchanged `pomodoro_name` context.
7. **"Interactive editor markers"**: list `=#` / `=3#` as incomplete states.
8. **`README.md`**: add the grammar rows and a one-line example next to the existing `=`
   rows (around lines 208–216 and 275–297).
9. **Verify every documented example against a debug build** in a temp vault (`BOB_DIR`
   and `BOB_NOW`), and paste the real human output. Grep `docs/`, `README.md`, and
   `src/` help text for claims that `=` can never create or target a named session, and
   fix them.

## Phase `mac`: Bob Mac Capture support

**Repository.** Open bob-mac-capture with `/sase_repo`:
`sase repo open bob-mac-capture`. On Linux hosts the linked checkout is missing, so fall
back to `sase repo open gh:bobs-org/bob-mac-capture`. Use only the printed path, and
read its `AGENTS.md` if one exists. The app never parses grammar or reads the ledger; it
follows Bob's spans, needs, contexts, and JSON.

1. **Decoding** (`Sources/CaptureCore/CaptureModels.swift`,
   `CompletionRowContent.swift`):
   - Add `CaptureCompletionContext.pomodoroStartName` (`"pomodoro_start_name"`).
   - Add `CaptureCompletionCandidate.nextUp`, decoded from `next_up` with
     `decodeIfPresent ?? false`.
2. **Completion gating** (`CapturePanelModel.swift`, `shouldRequestCompletion`):
   - `=#`, `=3#`, `=#de`, and chain `=x =#de` already request completion through the
     `pomodoro_name` / `interactive_placeholder` spans and `needs`.
   - Add tests for each, and one proving that a caret on `=3` requests nothing.
3. **Rows** (`completionRowContent`, case `.pomodoroStartName`, context label
   **"Start"**):

   | Row     | Icon                     | Category          | Primary                          | Secondary                             | Badges                                                       | A11y hint                                                      |
   | ------- | ------------------------ | ----------------- | -------------------------------- | ------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------- |
   | start   | `play.circle`            | `.pomodoroStart`  | name                             | "Next up" if `nextUp`, else "Planned" | `Next` (if next up), `1 link`/`N links`/`Empty`, `N matches` | "Starts this Pomodoro now."                                    |
   | new     | `timer.badge.plus`       | `.priority`       | name                             | "New session"                         | `New`                                                        | "Creates this Pomodoro and starts it now."                     |
   | again   | `arrow.clockwise.circle` | `.pomodoroStart`  | name                             | "Last ran 0830–0855"                  | `Again`                                                      | "Starts a new NAME session now."                               |
   | name it | `square.and.pencil`      | `.priority`       | time range or "Unnamed Pomodoro" | "Next up"/"Planned"                   | link count, `Name it`                                        | "Names this Pomodoro, then selects it."                        |
   | running | `timer`                  | secondary/neutral | name                             | "Running 0840–0905"                   | `Running`                                                    | "Already running. Close it first with =x, or write =x =#name." |
   - Use the singular "1 link" in this context.
   - Leave `.pomodoroName` rows unchanged.

4. **Accept** (`acceptSelectedCompletion`):
   - For `pomodoro_start_name` rows, a new row announces "NAME will be created and
     started when captured". An again row announces "Starts a new NAME session when
     captured".
   - A `requiresName` row opens the existing Name Pomodoro prompt. Extend the context
     checks so `pomodoro_start_name` shares the `pomodoro_name` prompt flow and the slug
     splice.
   - Every other row inserts the slug. The caret lands after the name, and the live
     preview runs.
5. **Calm incomplete state**: a draft whose item (or top level) is mode `incomplete`,
   with `needs` containing `pomodoro_name` and a `pomodoro_start` object, gets special
   handling:
   - It skips the doomed live dry run.
   - It shows the status "Pick a Pomodoro to start, or type a new name". Mirror
     `applyQuietIncompletePicker`, and keep the completion list open.
   - Existing `@route:id#` behavior is unchanged.
6. **Start card** (`CapturePomodoroStartPresentation`,
   `CapturePanelView.startPreviewItem`):
   - When `createdPomodoro`, show a small pink capsule next to the title: **New** on a
     dry run, **Created** once committed. Add it to the accessibility summary. The
     notification body reads "0905–0930 (25m) · New session".
   - When the capture `text` has no `#` (a bare `=`/`=<X>` start), show a quiet caption
     teaching hint in the close card's `TeachingHint` style: "Type #name to start a
     specific Pomodoro".
   - Named starts show no hint.
   - Error previews (R3/R4) keep the existing red error block. Bob's message already
     teaches `=x =#name`.
7. **Colors**: `=3#bugs` renders the `pomodoro_start` span pink and the `pomodoro_name`
   span purple. No mapping change is needed; add a span-category test.
8. **README**:
   - Grammar list: `=<X>#name`, `=x =#name`.
   - Completion section: the start rows and their meaning.
   - Live-preview section: the New badge and the teaching hint.
   - Bump the required Bob features.
9. **Fixtures and tests**:
   - Build bob from the bob-cli checkout (`cargo build`). Generate real-bob fixtures for
     the worked-example ledger:
     - `capture-parse` for `=#`, `=3#bugs`, `=#deep work`, and `=x =#bugs`
     - `capture-complete` for `=#`, `=#rev`, and the running variant
     - `capture --dry-run --format json` for an existing start, a created start, an
       again start, and the R4 error
   - Add matching `Tests/Fixtures/fake-bob` cases.
   - Tests:
     - decoding the context and `next_up`
     - every row kind's content
     - accept announcements and the name prompt
     - the calm incomplete state
     - start-card badge and hint presence and absence
     - notification body
     - completion gating
   - Keep `testCaptureCompleteOffersNoCandidatesForStart` green for `=`/`=3`/`=-2`.
   - If the picker/design test suite covers the start card, add a named-start state.
10. **Verify on macOS.** Swift is not available on Linux:
    - Try `ssh -o ConnectTimeout=8 mac true` first. If it works, rsync the checkout
      (excluding `.build`) to `mac:/tmp/bob-mac-capture-named-start/` and run
      `just format-lint build test` there.
    - Otherwise iterate on GitHub Actions. This plan explicitly instructs you to use
      `/sase_git_commit` for each CI iteration in bob-mac-capture, with conventional
      `feat(capture): …` or `fix(capture): …` subjects.
    - Find the run with `gh run list -L 3`. Wait with `gh run watch <id> --exit-status`
      in the foreground with a tool timeout of about 30 minutes, or hand the wait to
      `/sase_monitor`.
    - Read failures with
      `gh run view <id> --log-failed | grep -E " error: |error: -\[|failed \("`.
    - Done means one `macOS 26 SwiftPM` run with every step green. Record the run ID in
      the final response.
    - Never weaken or skip an assertion to go green.

## Out of scope and follow-ups

- **Aligning link forms with the "again" name rule.** `@r:id#deep-work` after a
  completed `DEEP WORK` still creates `DEEP-WORK`. Propose a follow-up task if Bryan
  approves the flagged decision.
- **`#name=<X>` spelling for whole-item starts.** It stays an error (E3) that teaches
  `=<X>#name`, so one order exists per form.
- **Mentioning `=#name` in the `=x` "next up" diagnostic.** It keeps
  `(start it with `=`)`.
- **Obsidian plugin parity** (an `se<X>` variant that targets a named entry). Not
  requested.
