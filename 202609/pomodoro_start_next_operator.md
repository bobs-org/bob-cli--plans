---
tier: epic
title: Start the next future Pomodoro from capture with =<X>
goal: "A whole capture item `=<X>` starts today's next future Pomodoro with the same
  `se<X>` timing as `@route:block-id=<X>`, only when a future Pomodoro exists and none
  is running, and Bob CLI and Bob Mac Capture preview (including the session's queued
  tasks) and apply it with identical, atomic results.

  "
phases:
  - id: start_core
    title: Parse and atomically apply whole-item Pomodoro starts
    depends_on: []
    size: medium
    description: 'start_core: add the shared `=`-family lexer,
      CaptureKind::PomodoroStart, the staged next-future-Pomodoro planner with its
      guards and error copy, JSON/human output, the shared selection helper, the "start
      it with `=`" hints, and CLI tests.

      '
  - id: start_lineup
    title: Report the started session's queued Task Links
    depends_on:
      - start_core
    size: medium
    description: "start_lineup: list and read-only resolve the started entry's
      direct-child Task Links through the close planner's vault view, and report them as
      `pomodoro_start.tasks` rows and human lineup lines.

      "
  - id: editor_contract
    title: Expose and document the Pomodoro start editor contract
    depends_on:
      - start_lineup
    size: medium
    description: "editor_contract: teach capture-parse, completion, and rewrite the
      whole-item start (mode, span, spec, diagnostics, @@ skip), update help,
      docs/capture.md, and README with the lifecycle table and zsh quoting, and add
      protocol tests.

      "
  - id: mac_start
    title: Preview and submit Pomodoro starts in Bob Mac Capture
    depends_on:
      - editor_contract
    size: medium
    description:
      "mac_start: tolerant start decoding, a play-glyph start card with queued tasks,
      the Start footer action, notifications, real-bob fixtures replacing the `=`
      incomplete fixtures, tests, README, and green macOS CI."
proposed_by: bbugyi200.apollo.2s
create_time: 2026-09-28 12:19:12
status: wip
---

- **PROMPT:**
  [prompts/202609/pomodoro_start_next_operator.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/pomodoro_start_next_operator.md)

# Problem and outcome

`bob capture` already has a small family of whole-item **Pomodoro session operators**
that act on today's running Pomodoro: `+[N]`/`-[N]` resize it, `++[N]`/`--[N]` shift it,
and `=x` closes it. What is missing is the first step of the lifecycle: starting the
next planned session. Today that requires `^route:block-id=<X>` (which also links a
task) or opening Obsidian, moving the cursor into the next `- [ ] ()` placeholder, and
typing `se<X>` + Tab.

A whole capture item `=<X>` must start today's **next future Pomodoro** — the first
open, untimed `- [ ] ()` placeholder in the daily note's `## Pomodoros` section — with
exactly the `se<X>` timing the `@route:block-id=<X>` suffix already accepts (empty is 25
minutes, `3`, `-`, `-2`, `3-`, `2-1`, …). It is valid only when the ledger has a future
Pomodoro and no running one. Bob CLI and Bob Mac Capture must preview and apply it with
identical, atomic results, and the preview must show what session is about to start and
which tasks are queued in it.

Bob stays the only implementation of grammar, clock and ledger math, link resolution,
and vault writes. The Mac app only decodes and presents Bob's JSON.

# Design: `=` starts, `=x` stops

The operator family becomes a complete session lifecycle, taught in this order:

| Item            | Operator | Effect                                                             | Obsidian                  |
| --------------- | -------- | ------------------------------------------------------------------ | ------------------------- |
| `=` / `=<X>`    | start    | Start the next future Pomodoro now, timed like `se<X>`             | `se<X>` + Tab inside `()` |
| `+[N]` / `-[N]` | resize   | Move the running session's end N × 5 min later / earlier           | `N\p` / `N\P`             |
| `++[N]`/`--[N]` | shift    | Move the running session's start and end N × 5 min later / earlier | `N\o` / `N\O`             |
| `=x`            | close    | Close the running session (unchanged)                              | Ctrl+Enter                |

Mnemonic for docs and help: **"`=` starts the next session, `=x` stops the running
one."** The `=` echoes the `=<X>` suffix and the `se<X>` snippet, `x` means done, and a
whole day reads naturally as a sequence of drafts: `=`, `+2`, `=x`, `=`, … In the Mac
panel the start card uses a play glyph as the visual sibling of the close card's stop
glyph.

Naming: the kind and editor mode are both `pomodoro_start`. They reuse the existing
`pomodoro_start` objects: the parse spec `{raw, duration_units, offset_units}` and the
capture summary
`{start, end, duration_minutes, offset_units, pomodoro_name, pomodoro_line, created_pomodoro, time_range}`.
The kind name therefore matches its object name exactly as `pomodoro_shift` and
`pomodoro_close` do, and consumers that already render `pomodoro_start` (the Mac start
row) keep working. Human verbs are `started` / `would start`; the capture `placement`
value is the new `"started"`.

"Next future Pomodoro" means exactly one thing everywhere: the first entry in document
order that `capture_pomodoros::scan` reports as
`Open && placeholder && time_range.is_none()`. That is the rule the unnamed link start
already uses and the entry the `=x` diagnostic already names as "next up", so `=` starts
precisely the session Bob told the user is next.

## Recognition rules (exact)

Let `t` be the item's first physical line trimmed of leading and trailing whitespace. A
**start token** is `=` followed by the longest run matching the `se<X>` suffix grammar
`[0-9]*(-[0-9]*)?`. A token is **bare** when its suffix is empty or contains no digit
(`=`, `=-`) and **counted** when its suffix contains at least one digit (`=3`, `=-2`,
`=3-`, `=2-1`, `=0`). Recognition lives in the same `=`-family check as the whole-item
close, which runs first (before session operators, caret links, and ordinary parsing).
`=x`/`=X` keep their close meaning because `x` is never a suffix character.

1. **Exact start:** `t` is exactly one start token and the item has exactly one physical
   line. The suffix goes through the existing `parse_pomodoro_start_suffix`, so `=` is
   25 minutes, `=3` is 15 minutes, `=-` is 25 minutes with a 5-minute offset, `=2-1` is
   10 minutes with a 5-minute offset, `0` and leading zeros are valid, and an oversized
   value fails with the existing start overflow error before any write.
2. **Claimed near miss (error, never a task):** `t` begins with a counted token followed
   by anything else (`=3 more`, `=-2 @work`, `=3s:1`, `=2-1-`, `=3x`), or `t` is exactly
   a start token (bare or counted) in an item that also has child lines (`=` plus a
   child bullet). Suggested copy, echoing the typed token:
   ``Pomodoro start `=3` must be the whole capture item; remove extra text, markers, or child lines (to start a task's session instead, use `^route:block-id=3`)``.
3. **Not a start (ordinary parsing, unchanged):** a bare token followed by more text
   (`= foo`, `=- foo`, `==`, `=-)`), every close shape (`=x`, `=x more`, `=xx`, `=x!`
   keep today's meaning), and mid-body tokens (`Plan =3`, `a=3`). This follows the
   established precedent that a bare sign run followed by prose (`- foo`, `-- aside`)
   stays prose while a counted token claims its item.

Intentional behavior changes: a bare `=` stops being an incomplete editing state and
becomes a complete start; `=3`, `=-2`, … stop being prose; `=3 more` becomes an error.
`POMODORO_CLOSE_INCOMPLETE_ERROR` and every path that produces the `=` incomplete state
are deleted.

## Execution semantics

- **Target and guards,** checked in this order on the staged daily note (from
  `BOB_DAY_FILE` or `<bob-dir>/YYYY/YYYYMMDD.md`, read through `CaptureBatchPlanner`):
  1. Missing day file:
     ``no future Pomodoro to start: today's daily note `2026/20260928.md` does not exist``.
  2. Missing section: the existing `Bob daily note has no Pomodoros section`.
  3. Exactly one open timed entry (including one past its nominal end) — name it, give
     its range and line, and teach the switch idiom with the user's own token:
     ``cannot start the next Pomodoro: CAPTURE 0920-0950 is still running at line 5; close it with `=x` first, or capture `=x`, a blank line, then `=3` to switch sessions``
     (an unnamed entry reads "the current session"). This is also exactly what a user
     who typed `=` on the way to `=x` sees in the live preview, so it replaces the old
     "type `=x`" incomplete hint with something more useful.
  4. More than one open timed entry:
     `cannot start the next Pomodoro: today's ledger has multiple open timed Pomodoros; finish all but one first`.
  5. No future Pomodoro:
     ``no future Pomodoro to start: today's ledger (`2026/20260928.md`) has no open `- [ ] ()` placeholder (to start a task's session instead, use `^route:block-id=`)``.
     `=` never creates an entry; that stays the job of `^route:block-id=` and
     `@route:block-id#name`. Use the same `CaptureError` classes the close's "no running
     Pomodoro" and the start's "active timed Pomodoro" errors use, so exit codes and
     JSON error shape stay consistent.
- **Selection:** extract one shared helper (for example
  `capture_pomodoros::next_future_pomodoro(&PomodoroScan) -> Option<&PomodoroEntry>`)
  and use it for the new operator, for the unnamed branch of `plan_pomodoro_start` in
  `src/native/capture.rs`, and for the "next up" lookup in
  `capture_pomodoro_close::find_running_pomodoro`, so the three can never drift.
- **Time:** `compute_pomodoro_start_range(now, spec)` (the existing `se<X>` math:
  `start = ceil((now − 5·offset)/5)·5`, end = start + 5·duration, both mod 1440,
  rendered `(**HHMM-HHMM** [t:: Nm])`). With `BOB_NOW=… 09:42`: `=` → `0945-1010`, `=3`
  → `0945-1000`, `=-` → `0940-1005`, `=-2` → `0935-1000`, `=2-1` → `0940-0950`.
- **Rewrite:** `start_existing_pomodoro_entry(contents, index, &time_range)` — replace
  the `()` with the canonical range, preserving checkbox, name, other bytes, children,
  CRLF, and a missing final newline, then move the entry with its child block to the
  current slot (after the last completed entry's block, else before the first other open
  entry; left in place when already there). For a canonical placeholder the started line
  is byte-identical to Obsidian's `se<X>` + Tab inside `()`: `- [ ] () — CAPTURE` with
  `=` at 09:37 becomes `- [ ] (**0940-1005** [t:: 25m]) — CAPTURE`. No entry is created,
  no link is added, and no task note is written (Obsidian's snippet does not touch tasks
  either; tasks go In Progress when `=x` closes the session).
- **Transaction:** stage through `CaptureBatchPlanner` like the other operators. Later
  items see earlier staged edits and dry-run computes without committing; any failure
  rolls back the whole batch. This makes the lifecycle composable in one draft:
  - `=x`, blank line, `=` — close the running session and start the next one. When the
    close carried links it inserted a carried placeholder right after the closed block,
    so the next session continues the same work; otherwise the next planned entry
    starts. (When nothing follows, the close always inserts a placeholder, so this idiom
    cannot hit the "no future Pomodoro" error.)
  - `=`, blank line, `+2` — start, then extend.
  - `=`, blank line, `^bob:ready` — start, then link a task into the now-running
    session.
- **Isolation:** a start item is item-local. A `@@` declaration never applies to it.
  Forced `--route`/`--section`/`--task`/`--task-ref`/`--task-section`/`--clip` fail with
  ``Pomodoro start `=<X>` cannot be combined with --route, --section, --task, --task-ref, --task-section, or --clip; capture the start alone``.
  `%`, `s:<N>`, and `p:<N>` after a counted token are near-miss shape errors by
  construction. `--dry-run`, `--no-clip`, `--bob-dir`, `BOB_DAY_FILE`, `BOB_NOW`, and
  output formats keep working.

## Queued-task lineup (read-only)

The start result lists the session's queued Task Links so both the CLI and the Mac
preview can say what work is about to begin.

- **Which lines:** the started entry's direct child bullets (the lines of
  `capture_pomodoro_close::sub_bullet_range` at the entry's first child indentation)
  whose body, after stripping 🍅 markers, is exactly one block link — plain
  `[[path#^id]]` or embedded `![[path#^id]]` — and is not struck through. This is the
  glossary definition of a Task Link on a Pomodoro. Notes, deeper descendants, mixed
  lines, struck links, and fenced lines are not listed. Order is ledger order.
- **Resolution:** resolve each link read-only through the same vault view and
  `note_tasks` lookup the close planner uses (`CloseVault` over the batch planner's
  staged contents, `lookup_task`, `close_task_text`), so a task created earlier in the
  same draft resolves and text matches the close's rows. Make the needed close helpers
  `pub(crate)` rather than duplicating them. Resolution never fails the start and never
  writes: an unresolvable link becomes a row with `resolved: false` and a `warning`
  (reuse the close's wording, e.g. `bob.md has no task with block ID ^missing`).
- **Where:** a new focused module `src/native/capture_pomodoro_start.rs` holding the
  pure lister and the resolver, with unit tests.

## Contracts

`bob capture --format json` for a start keeps schema version 1 and every existing key.
It reports kind `"pomodoro_start"`, `text` = the raw token (`"="`, `"=3"`), `task_line`
= the started ledger line, `placement: "started"`, `routed: false`, `route: null`,
`route_label: ""`, `relative_target` = the day file, `created` = the clock's date,
`pomodoro_name` when named, and the existing `pomodoro_start` object with
`created_pomodoro: false` plus an additive `tasks` array (present, possibly empty, only
for kind `pomodoro_start`; link and task starts omit it and stay byte-stable):

```json
"pomodoro_start": {
  "start": "0940", "end": "1005",
  "duration_minutes": 25, "offset_units": 0,
  "pomodoro_name": "CAPTURE", "pomodoro_line": 13,
  "created_pomodoro": false,
  "time_range": "(**0940-1005** [t:: 25m])",
  "tasks": [
    {"block_link": "[[bob#^capture-stop]]", "embedded": false, "ledger_line": 14,
     "resolved": true, "relative_target": "bob.md", "block_id": "capture-stop",
     "text": "Stop capture from the panel", "status_symbol": "/",
     "status_name": "In Progress", "warning": null},
    {"block_link": "[[bob#^gone]]", "embedded": false, "ledger_line": 15,
     "resolved": false, "relative_target": null, "block_id": "gone", "text": null,
     "status_symbol": null, "status_name": null,
     "warning": "bob.md has no task with block ID ^gone"}
  ]
}
```

Row nullability follows the close convention: explicitly `null`, not omitted. Line
numbers are 1-based post-image lines. Batches carry the object per item in `captures`,
and the first item's value is also top-level, as today.

Human output mirrors the other operators' layout (green `✓`, `[dry-run] ok would start`
for dry runs), then the lineup, one row per task in the close's row style
(`print_human_pomodoro_close_success`) with the unchanged status marker instead of a
transition, and the close's dim `warning: …` line for unresolved rows:

```text
✓ started  2026/20260928.md
  CAPTURE 0940-1005 (25m) at line 13
  - [ ] (**0940-1005** [t:: 25m]) — CAPTURE
  [/] Stop capture from the panel bob.md ^capture-stop
  warning: bob.md has no task with block ID ^gone
```

An unnamed entry reads "next session"; an empty lineup prints a dim `  nothing queued`.

`bob capture-parse` stays purely lexical: mode `pomodoro_start`; one `pomodoro_start`
span over the whole token including `=` (the span kind the suffix start already uses);
the existing `pomodoro_start` spec (`raw` excludes `=` exactly as for the suffix, so
`=3` reports `{"raw": "3", "duration_units": 3, "offset_units": 0}` and `=` reports
`{"raw": "", "duration_units": 5, "offset_units": 0}`), top-level for the first item and
per item; and an `invalid_pomodoro_start` diagnostic for near misses whose range policy
matches `invalid_pomodoro_close` (the extra text for trailing text, the child lines for
child-line misses, the token for overflow). The human parse line keeps its existing
format (`start     =3 (15m, offset 0u)`).

## Discoverability hints

Each "nothing is running" error points at `=` when a future Pomodoro exists:

- Whole-item `=x`: `…; next up is CAPTURE at line 13` gains ``(start it with `=`)``.
  Link and task close forms keep their existing
  ``(to start a session with this task instead, use `^route:block-id=`)`` hint instead.
- Whole-item resize and shift: `Bob daily note has no open timed Pomodoro to shift`
  gains ``; next up is CAPTURE at line 13 (start it with `=`)`` when a future Pomodoro
  exists (compute it with the shared selection helper inside `select_running_session`).

## Command-line spelling

zsh (Bryan's shell) performs `EQUALS` expansion: an unquoted word starting with `=` is
replaced by a command path, so `bob capture =3` fails with `zsh: 3 not found` before Bob
runs (bash passes it through; a lone `=` is not expanded). Every doc and help example
must quote start and close items — `bob capture '='`, `bob capture '=3'`,
`bob capture '=-2'`, `bob capture '=x'` — and the docs carry a one-line note: "Quote `=`
items in zsh, which expands a leading `=word` to a command path." Fix the existing
unquoted `bob capture =x` examples too (`README.md`, `docs/capture.md`, and the
`bob capture --help` examples). The Mac app is unaffected because it passes the draft as
one argv element after `--`.

# start_core

Work in bob-cli.

- `src/native/capture_language.rs`:
  - Add a small `=`-family lexer shared by the execution and editor grammars (for
    example `session_equals_token(text) -> Option<EqualsToken>` with `Close` and
    `Start { suffix, counted, len }` variants), mirroring `session_operator_token`.
  - Rework `parse_pomodoro_close_item` into a `=`-family item parser implementing the
    recognition rules: it emits the unchanged `CaptureKind::PomodoroClose` or a new
    `CaptureKind::PomodoroStart { spec: PomodoroStartSpec }`, reusing
    `parse_pomodoro_start_suffix` for units and overflow.
  - Add `POMODORO_START_ITEM_SHAPE_ERROR`-style copy (formatted with the typed token)
    and `POMODORO_START_FORCED_ERROR`; delete `POMODORO_CLOSE_INCOMPLETE_ERROR`.
  - Add `EditorMode::PomodoroStart` (label `pomodoro_start`) now so the parity test's
    `expected_mode` match maps `PomodoroStart` without a placeholder; leave the editor
    recognition, spans, and diagnostics to `editor_contract`.
  - Update the unit tests that pin `=` as incomplete and `=3` as prose on the execution
    side.
- `src/native/capture_pomodoros.rs`: add the shared `next_future_pomodoro` helper and
  use it in `plan_pomodoro_start` and `capture_pomodoro_close::find_running_pomodoro`.
- `src/native/capture.rs`:
  - Add `plan_pomodoro_start_item`, `reject_pomodoro_start_conflicts`, and the dispatch
    arm in `plan_capture_item` beside the close/adjust/shift arms (grep
    `PomodoroShift`/`pomodoro_shift` to find every exhaustive site).
  - Add `Placement::Started` (`"started"`), kind label `pomodoro_start`, and
    `print_human_pomodoro_start_success`.
  - Reuse `PomodoroStartSummary` for the result; add the `tasks` field now as an
    `Option` that `start_lineup` fills (skip when `None`; this phase always sets
    `None`).
  - Implement the guards and exact error copy from "Execution semantics", and the
    discoverability hints for whole-item `=x`, resize, and shift.
- `tests/cli.rs` (follow the `capture_pomodoro_shift_*` and `capture_pomodoro_close_*`
  structure and helpers such as `close_worked_vault` / `run_close_json`). Cover:
  - `=`, `=3`, `=-`, `=-2`, `=3-`, `=2-1`, `=0`, `=03` JSON and human output against
    `BOB_NOW`, including the byte-identical canonical line with name, children, CRLF,
    and no final newline preserved.
  - The first-placeholder rule with several future entries, and the move to the current
    slot when a completed entry follows the placeholder.
  - Every error: running (named and unnamed, including one past its end), multiple
    running, no future Pomodoro (empty ledger, only completed entries, only a
    non-placeholder open line), missing day file, missing section, overflow, extra text,
    child lines, each forced flag, `=3 more`, `=3x`.
  - Prose that stays prose: `= foo`, `=- foo`, `==`, `Plan =3`; close shapes unchanged.
    Replace the existing assertion that `bob capture =3` is a task.
  - Batches in order: `=x` / `=` switch on the close worked example (carried placeholder
    starts), `=` then `+2`, `=` then `^route:id`, a start plus an ordinary `@@`-routed
    task; dry-run writes nothing; a late failing item rolls back the start.
  - The new hints on `=x`, `+`, and `++` "nothing running" errors (update existing
    exact-message assertions).
  - argv spellings `bob capture =`, `bob capture =3`, and `bob capture -- =-2` reach the
    parser verbatim (tests exec the binary directly, so no shell expansion applies).
- Run the focused tests, `cargo clippy --all-targets --all-features`, and
  `cargo fmt --check`.

# start_lineup

Work in bob-cli, building on `start_core`.

- New `src/native/capture_pomodoro_start.rs` (register it in `src/native.rs`): the pure
  Task Link lister for a started entry and the read-only resolver described in
  "Queued-task lineup", with unit tests for direct-child selection, 🍅 stripping,
  embeds, struck and mixed lines, fenced lines, and ledger order.
- `src/native/capture_pomodoro_close.rs`: make the minimum set of helpers `pub(crate)`
  (`wikilink_tokens`, `strip_pomodoro_markers`, `bare_plain_link`, `lookup_task`,
  `close_task_text`, the strikethrough helpers, the `CloseVault` implementation used by
  the close planner) instead of copying them. Keep close behavior byte-stable.
- `src/native/capture.rs`: fill `pomodoro_start.tasks` for kind `pomodoro_start` only,
  using the post-image day file and the batch planner's staged vault view; print the
  lineup rows and `nothing queued` in `print_human_pomodoro_start_success`.
- `tests/cli.rs`: lineup JSON and human rows for resolved Ready/Next/In Progress/Blocked
  tasks, an embedded link, an unresolved link (missing ID and missing note), a task
  created earlier in the same draft, an empty placeholder (`tasks: []`,
  `nothing queued`), explicit `null` row fields, and that link/task starts
  (`^route:id=`, `Text @route:id=3`) still omit `tasks`. Confirm no task note changes on
  start.
- Run the focused tests, clippy, and fmt.

# editor_contract

Work in bob-cli, building on the `start_core` lexer and the `start_lineup` contract.

- `src/native/capture_language.rs`:
  - Rework `parse_editor_close_item` into the editor `=`-family parser on the shared
    lexer. It never fails: exact starts report `pomodoro_start` with the spec and one
    `SpanKind::PomodoroStart` span over the whole token; near misses report mode
    `pomodoro_start` plus `invalid_pomodoro_start` with the close's range policy; bare
    `=` is a complete start and no longer `incomplete`.
  - Skip `@@` inheritance for start items (as for adjust, shift, and close), add the
    `@@ cannot take a Pomodoro start …` notice, and the `NonAbsorbable` arm.
  - Extend completion suppression in `completion_field_at` to start items.
  - Extend the parity inputs with `=`, `=3`, `=-`, `=-2`, `=3-`, `=2-1`, `=0`, `=03`,
    `  =3  `, and near misses; update the editor-side unit tests that pinned `=` as
    incomplete and `=3` as prose.
- `src/native/capture_parse.rs`: JSON for the new mode, the Modes list (`pomodoro_start`
  beside the other session modes), long help describing the whole-item start and its
  spec, and help examples `'='`, `'=3'`, and `printf '=x\n\n=\n'`.
- `src/native/capture_complete.rs` and `src/native/capture_rewrite.rs`: help text drops
  "a lone `=` is incomplete"; start items request no completion and are never rewritten
  (tests).
- `bob capture --help` (`src/native/capture.rs`): present the lifecycle table (start,
  resize, shift, close) with the mnemonic, drop "a lone `=` is incomplete, and `=3` …
  stay ordinary prose", add quoted examples `bob capture '='`, `bob capture '=3'`,
  `printf '=x\n\n=\n' | bob capture`, quote the existing `=x` example, and add the zsh
  quoting note.
- `docs/capture.md`:
  - Grammar-at-a-glance: add `=[X]` (start the next future Pomodoro) and update the "You
    typed" rows: `=` (start the next session, 25 minutes), `=3`, `=-2`, `=3 more`
    (error), `= foo` and `==` (prose); remove the `=` incomplete row and rewrite the
    `=3, =xx, =x!, ==` row.
  - A new "Starting the next Pomodoro" section placed before "Adjusting the current
    Pomodoro" (lifecycle order), with the operator table, recognition rules, guards,
    selection rule, placement, lineup, JSON example, human example, batch idioms
    (`printf '=x\n\n=\n' | bob capture`), and the zsh note. Add the contents entry.
  - Close section: drop the incomplete sentence, document the `(start it with `=`)`
    hint, add the `=x`/`=` switch idiom beside the existing `=x`/`^route:id=` idiom, and
    quote the `=x` examples. Adjust and shift sections: document the new hint.
  - Parse contract: modes, span kinds, diagnostic codes, human lines, and the
    completion-suppression sentence.
- `README.md`: update the capture grammar summary and quote the `=x` examples.
- `tests/cli.rs`: follow `capture_parse_pomodoro_close_protocol`,
  `capture_parse_pomodoro_shift_protocol`, `capture_complete_pomodoro_close_protocol`,
  and `capture_rewrite_pomodoro_close_protocol`. Cover exact and bare starts,
  whitespace, mixed drafts, `@@` drafts (including the notice), near-miss diagnostics
  and ranges, prose non-matches, JSON compatibility (spec omitted for other modes),
  completion and rewrite ignoring start items, and help-text mentions. Update the parse
  tests that pinned `=` as incomplete.
- Finish with `just fmt`, `just lint`, and `just test`.

# mac_start

Work in bob-mac-capture. Open it with `/sase_repo`; if the linked name `bob-mac-capture`
does not resolve, open `gh:bobs-org/bob-mac-capture`. Follow the patterns of the shift
commit `794f06a` and the close preview commits `4351e1c` and `aa4e156`.

- `Sources/CaptureCore/CaptureModels.swift`:
  - Make `PomodoroStartSpec` and `PomodoroStartSummary` tolerant (custom `init(from:)`
    with `decodeIfPresent` defaults, like `PomodoroShiftSummary`) so a future additive
    change can never break decoding.
  - Add a tolerant `PomodoroStartTask` (`block_link`, `embedded`, `ledger_line`,
    `resolved`, `relative_target?`, `block_id`, `text?`, `status_symbol?`,
    `status_name?`, `warning?`) and `tasks: [PomodoroStartTask]?` on
    `PomodoroStartSummary` (nil for older Bob and for link/task starts).
- New `Sources/CaptureCore/CapturePomodoroSessionStartPresentation.swift` (pure wording,
  no Swift clock, ledger, or link math), gated on `kind == "pomodoro_start"` with a
  `pomodoroStart` object:
  - `title`: `Start CAPTURE` (dry run) / `Started CAPTURE`; unnamed reads
    `next session`.
  - `sessionText`: `0940-1005 (25m)`; `destinationText`: `2026/20260928.md · line 13`.
  - `taskRows` / `visibleTaskRows` / `overflowTaskCount` using the close card's visible
    cap, and `emptyText` `Nothing queued`.
  - `statusText` matching the CLI (`Would start CAPTURE 0940-1005 (25m) at line 13`).
  - `primaryActionTitle` `Start`, `notificationTitle` `Started CAPTURE`,
    `notificationBody` `0940-1005 (25m) · 2 queued tasks` (singular for one,
    `Nothing queued` for none), `batchSuffix` ` (started CAPTURE 0940-1005)`, and a
    spoken `accessibilitySummary`.
- `Sources/BobMacCapture/CapturePanelView.swift`: a `startPreviewItem` card as the
  visual sibling of `closePreviewItem`. Header: `play.circle.fill` tinted
  `CaptureEditorPalette.color(for: .pomodoroStart)`, the title, and the monospaced
  semibold session text. Second row: the secondary destination. Then task rows with a
  status glyph (`circle` Ready, `circle.inset.filled` Next, `circle.lefthalf.filled` In
  Progress, `questionmark.circle` Blocked/other, `exclamationmark.triangle` unresolved
  with the warning as help text), the task text, and a truncating `bob ^id` locator;
  then `+N more` or the empty text. Select it in `previewItem` right after the close
  card, and include its summary in `previewAccessibilityLabel`.
- `Sources/BobMacCapture/CapturePanelModel.swift`: a single-item
  `sessionStartPresentation`; `primaryActionTitle` becomes `Start` for it (after
  `Close`); the live preview sets `statusText` for a single start as it does for close;
  add the `sole…Presentation` status after explicit preview and submit; treat
  `pomodoro_start` as a day-file write in `captureWroteDayFile`.
- `Sources/BobMacCapture/NotificationService.swift`: friendly kind `Start`, single
  capture title and body from the presentation, the batch suffix, and `dayFileChanged`
  true for `pomodoro_start`.
- `Sources/CaptureCore/CompletionRowContent.swift`: confirm `pomodoro_start` spans keep
  the Pomodoro-session category (already mapped); add a test.
- Fixtures and tests:
  - Build bob from bob-cli master after `editor_contract` lands and generate real JSON
    fixtures against a sandbox vault (`BOB_DIR`/`BOB_DAY_FILE` + `BOB_NOW`):
    `pomodoro-start-next.json` (`=` dry run, named, resolved and unresolved rows),
    `pomodoro-start-timed.json` (`=-2`), `pomodoro-start-empty.json` (nothing queued),
    `pomodoro-start-running.json` and `pomodoro-start-none.json` (failures),
    `pomodoro-start-switch.json` (`=x`, blank line, `=`), and parse responses for `=`,
    `=3`, and `=3 more`.
  - Remove `pomodoro-close-incomplete.json` and `pomodoro-close-parse-incomplete.json`;
    repoint `testFailedCloseDryRunClearsStaleCard` and the close presentation tests that
    used them at the running-session failure. Regenerate
    `pomodoro-close-no-running.json` and `pomodoro-shift-no-running.json` for the new
    `(start it with `=`)` hint and update their assertions.
  - Extend `Tests/Fixtures/fake-bob` (parse, capture, and the `capture-complete`
    no-candidates list for `=`, `=3`, `=-2`).
  - Add or extend `CaptureModelTests` (tolerant decode, older-Bob omission of `tasks`),
    `CapturePomodoroSessionStartPresentationTests`, `CapturePanelModelTests` (preview,
    `Start` footer, status text, switch batch keeps `Capture`, running-session error
    surface), `NotificationServiceTests`, and `BobProcessClientTests`. Watch argument
    order in test helpers (bob-cli-27.4 fixed a `relativeTarget`/`target` mix-up).
- `README.md`: add `=`/`=<X>` to the grammar requirement and keyboard paragraphs (start
  card, `Start` footer, older-Bob behavior: an older Bob reports `=` as incomplete and
  `=3` as a task, and the panel simply previews that).
- Verification: this Linux host has no Apple Swift toolchain. After committing, confirm
  the macOS CI workflow run for that exact commit passes (`gh run list` / `gh run view`
  in bob-mac-capture, waiting through `/sase_monitor`). Fix forward until it is green.

# Non-goals

- No named or targeted start from the whole-item form (`#name=3`, `=3#bugs`); naming a
  session stays with `^route:block-id#name=<X>`.
- `=` never creates a Pomodoro entry, never links a task, and never changes task
  statuses.
- No changes to bob-plugins or Obsidian keymaps, and no change to the top-level `bob`
  dispatcher's argument forwarding.
- No change to the shared `pomodoro_start` summary for link and task starts beyond the
  tolerant Mac decoding.

# Acceptance

- `=`, `=3`, `=-`, `=-2`, `=3-`, and `=2-1` start the first future Pomodoro with the
  same line Obsidian's `se<X>` + Tab produces, move it to the current slot, and fail
  write-free when a session is running or no future Pomodoro exists, with messages that
  name the fix.
- Near misses never become tasks; prose non-matches and every close shape keep their
  meaning.
- `=x`, blank line, `=` switches sessions atomically; mixed drafts apply in order;
  dry-run never writes; any failure leaves every file unchanged.
- The CLI and the Mac start card show the same session, time, line, and queued tasks as
  Bob's dry run, and submission produces the same result.
- bob-cli `just fmt`, `just lint`, and `just test` pass. bob-mac-capture macOS CI is
  green for the landed revision.
