---
tier: epic
title: 'Start lean: `=~<K>` drops queued Task Links as the next Pomodoro starts'
goal: '`bob capture ''=~2''` starts today''s next Pomodoro without queued Task Link
  2. It also works as `=<X>~<K>` and `=<X>#name~<K>`, in blank-line batches, and in
  same-line chains such as `=x =~2`. It is atomic and dry-runnable. Every start shows
  its numbered lineup, so you can see which number to drop. `capture-parse` and `capture-complete`
  support the new token, and Bob Mac Capture shows a numbered, drop-aware start card.

  '
phases:
- id: start-lineup
  title: 'bob-cli: numbered start lineup and the drop engine'
  depends_on: []
  size: medium
  description: 'start-lineup: number every whole-item start''s queued Task Links (`tasks[].index`,
    `tasks[].now`, a numbered human index column). Build the pure drop engine: validate
    numbers against the lineup, remove each dropped Task Link subtree byte-exactly,
    and report `drop`/`dropped` plus duplicate warnings. Plumb a `drop` list through
    both start planners, with no grammar yet.

    '
- id: start-drop-grammar
  title: 'bob-cli: `=[<X>][#name]~<K>` grammar, editor support, and docs'
  depends_on:
  - start-lineup
  size: medium
  description: 'start-drop-grammar: lex the trailing `~<K>` drop list on whole-item
    starts. Wire it through `bob capture` (execution, batches, chains) into the start-lineup
    engine. Mirror it in `capture-parse` (the `pomodoro_start_drop` span, the `pomodoro_start_task`
    need, incomplete states, precise diagnostics) and in `capture-complete`. Document
    the gesture everywhere.

    '
- id: mac-start-drop
  title: 'Bob Mac Capture: numbered, drop-aware start card'
  depends_on:
  - start-drop-grammar
  size: medium
  description: 'mac-start-drop: decode the new start fields tolerantly. Render numbered
    queued rows with NOW badges, dropped rows in place, a `Type ~2 to drop task 2`
    hint, and a `Dropped 2` summary. Color the new span. Preview a dangling `~`/`,`
    as a dimmed pending start with Start disabled. Add real-bob fixtures, tests, and
    README updates, gated by green macOS CI.'
proposed_by: bbugyi200.apollo.3f
create_time: 2026-09-30 08:28:31
status: wip
bead_id: bob-cli-2s
---

- **PROMPT:** [prompts/202609/start_drop_queued_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/start_drop_queued_links.md)
- **BEAD:** [bob-cli-2s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2s/README.md)

# Plan: Start lean — `=~<K>` drops queued Task Links as the next Pomodoro starts

## Context

`bob capture` has a whole-item Pomodoro session grammar:

- `=`/`=<X>` starts today's next future Pomodoro.
- `=<X>#name` starts a named Pomodoro.
- `=x[<N>][!<M>][~<K>]` closes the running Pomodoro. `~<K>` drops those numbered Task
  Links: they are removed, not carried, and not started.

Closes can already drop links. Starts cannot: to start the next session without one of
its queued links, you must edit the ledger in Obsidian first. Today `bob capture '=~2'`
silently captures a task whose text is `=~2` into `mac_inbox.md`.

This epic adds the mirror gesture: `=~<K>` starts the next session and drops queued Task
Links `<K>` from it. It is a whole item: the token must be the item's only text. It
works in blank-line batches and in same-line session chains.

Surfaces:

| Surface         | Where                                                                                  | How to open                                                                                                                                                                                                                                     |
| --------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob` CLI       | this repo (`bob-cli`)                                                                  | your workspace                                                                                                                                                                                                                                  |
| Mac capture app | `bob-mac-capture` (native macOS frontend; delegates grammar and preview to `bob` JSON) | `sase repo open bob-mac-capture -r "<why>"`. If that fails because the linked primary checkout is missing on this host, use `sase repo open gh:bobs-org/bob-mac-capture -r "<why>"`. Read its `AGENTS.md`/`README.md` "Runtime Contract" first. |

The closest prior art is commit `754d1f3` (`~<K>` drop outcome for `=x` closes) in
bob-cli and `b020df7` (drop outcome, `#now`, NOW badges) in bob-mac-capture. Read both
diffs before starting. This epic deliberately reuses their vocabulary, messages, span
colors, and visual language.

## Design

### The gesture (one index card)

> **`~` drops.** `=x~2` drops task 2 from the session you **stop**. `=~2` drops task 2
> from the session you **start**. The numbers are the ones `bob capture` shows for that
> session.

Examples:

| Typed                   | Meaning                                                                                                    |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| `=~2`                   | Start the next future Pomodoro (25m) without queued Task Link 2                                            |
| `=3~2,4`                | Start it for 15 minutes without links 2 and 4                                                              |
| `=-2~1`                 | Start it for 25 minutes with a 10-minute offset, without link 1                                            |
| `=#bugs~2`              | Start the open `BUGS` Pomodoro without its link 2                                                          |
| `=3#bugs~1,3`           | Start `BUGS` for 15 minutes without links 1 and 3                                                          |
| `=x =~2`                | Close the running session, then start the next one without link 2 (numbers refer to the post-close lineup) |
| `=x~2 =`                | Different: drop link 2 of the **running** session, so it is not carried; then start the next session       |
| `=~2 +2`                | Start without link 2, then extend by 10 minutes                                                            |
| `=x`, blank line, `=~2` | Same as `=x =~2`, as a batch                                                                               |

Why `~` is the only list a start can take: the digits after `=` are already the `se<X>`
timing, so a start has no room for a "keep" list like `=x<N>`. Also, starting never
changes a task's status, so a "complete" list like `!<M>` belongs to closes.

### Grammar

A whole-item start token is `=`, then the `se<X>` suffix (`[0-9]*(-[0-9]*)?`, possibly
empty), then an optional `#name` part, then an optional **drop part** `~<K>`:

- `<K>` is comma-separated task numbers, each at least 1, with no whitespace, in any
  order. JSON reports the list sorted ascending.
- The drop part always comes last: after `<X>`, and after `#name` when present.
- A named start's name part now ends at the first ASCII whitespace or `~`. `~` is not in
  the Pomodoro-name charset, so this cannot change any valid name.
- Any start token with a drop part **claims its item**, the same way a counted token
  does. It is never prose, near misses included. A bare `=` followed by separate text
  (`= ~2`, `= foo`) stays prose, unchanged.
- The `=x…` close grammar is unchanged. `=x~2` still closes. `x` is never a suffix
  character.
- Link and task starts (`^route:id=<X>`, `@route:id=<X>`, `<text> @route:id=<X>`) do
  **not** take a drop part. Their existing suffix errors stay as they are.

Lexical outcomes. These are shared by `bob capture` (execution) and `capture-parse`
(editor), with identical messages and precise byte ranges:

| Typed                                                        | Outcome                                                                                                                                                                                                                                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `=~2`, `=3~2,4`, `=-~1`, `=2-1~3`, `=#bugs~2`, `=3#bugs~1,3` | Valid                                                                                                                                                                                                                                                                    |
| `=~`, `=3~`, `=#bugs~`, `=~2,`                               | Editing state. `bob capture` rejects it with the existing wording, e.g. `` `=~` is incomplete: type a task number after `~` ``. `capture-parse` reports mode `incomplete`, needs `pomodoro_start_task`, and one `interactive_placeholder` span over the dangling `~`/`,` |
| `=~0`                                                        | `task numbers start at 1`                                                                                                                                                                                                                                                |
| `=~2,2`                                                      | ``task 2 is listed twice in `=~2,2` ``                                                                                                                                                                                                                                   |
| `=~,2`, `=~2,,3`, `=~,`                                      | ``expected a task number before `,` `` (on the offending comma)                                                                                                                                                                                                          |
| `=~2~3`                                                      | ``use one `~` list: `=~2,3` `` (on the second `~`)                                                                                                                                                                                                                       |
| `=~2!3`                                                      | ``a start can only drop Task Links; `!` completes them when you close (`=x!3`) `` (on `!3`)                                                                                                                                                                              |
| `=~2#bugs`                                                   | ``write the drop list after the name: `=#bugs~2` instead of `=~2#bugs` `` (keep any typed `<X>`)                                                                                                                                                                         |
| `=~2a`, `=~x`                                                | `` `=~2a` is not a drop list: write `~`, then comma-separated task numbers (for example `=~2,3`) `` (from the bad byte to the end of the token)                                                                                                                          |
| `=~99999999999`                                              | `task number 99999999999 is too large`                                                                                                                                                                                                                                   |
| `=~2 more`, or `=~2` with a child line                       | The existing start shape error, ``Pomodoro start `=~2` must be the whole capture item; …`` (on the extra text or child line)                                                                                                                                             |
| `=~ 2`, `=~2, 3`                                             | ``write the task numbers right after `~`, with no spaces (for example `=~2,3`) ``. This fires when the spaceless join of the line is a valid token; otherwise the shape error applies                                                                                    |
| `=#~2`                                                       | The existing empty-name error, interpolating the token **without** its `~<K>` part                                                                                                                                                                                       |

Precedence runs left to right: suffix errors and name errors first, then the drop list.
Named-start diagnostics keep their wording. They interpolate the token without the drop
part, and any corrected example they print keeps the typed `~<K>`: for example,
`=#bugs=3~2` teaches `=3#bugs~2`.

### Numbering: the queued lineup

- A start's lineup is exactly what `capture_pomodoro_start::list_queued_links` lists
  today: the started entry's direct-child bullets whose body, after stripping 🍅, is
  exactly one plain or embedded block link. Struck links, deeper descendants, mixed
  lines, notes, and fenced lines are excluded.
- The lineup is numbered 1..N in ledger order. The numbers are computed on the staged
  **pre-image** of the entry, after earlier batch items and chain tokens have run.
- Every whole-item start now reports these numbers, including a plain `=`: in human
  output, in `--dry-run`, and in JSON `tasks[].index`. This lets users discover the
  number to type.
- A named start that **creates** its session (no match, or "again") has an empty lineup.

### What a drop does

Order of operations for one start item:

1. Run the existing guards, in their existing order.
2. Resolve the entry (next future placeholder, or the named match).
3. Number the lineup and validate `<K>` against it.
4. Remove each dropped Task Link bullet **together with its nested child lines** from
   the entry's child block.
5. Run the existing start: rewrite the range and move the entry to the current slot.

Rules:

- **Subtree removal** matches what Obsidian's `planPomodoroLinkCleanupForRanges` and
  `@route+block-id!` already do when a Task Link leaves an open Pomodoro. It differs
  from close drops on purpose: a closed session keeps its notes as history, but in a
  session about to start, a queued link's children belong to that link. The removed
  child-line count is reported, so the preview shows it before anything is written.
- **Bytes contract**, same as the unnamed start: checkbox, name, untouched children,
  CRLF, and a missing final newline are preserved. Removing a last line that has no
  terminator must not leave a new final newline.
- **No task note is written.** `bob task-status-hooks` reconciles status as it does for
  any removed Pomodoro link. Its next run usually demotes the task from Next to Ready,
  unless another ledger link still plans it. `#now` keeps a dropped task in view this
  week. This is exactly the close-drop behavior.
- **Atomicity:** dry-run computes without writing. In a batch or chain, any failure
  rolls the whole batch back. `plan_budget` before→after picks up the smaller link count
  automatically.

Runtime diagnostics (nothing is written). The wording mirrors close selections:

- Out of range, one bad number:
  `` `=~4` names task 4, but CAPTURE has 3 queued Task Links (1–3) ``.
- One queued link: `… has 1 queued Task Link (1)`.
- No queued links:
  `` `=~1` names task 1, but CAPTURE has no queued Task Links; start it with `=` ``. The
  suggested token is the typed token without its `~<K>`.
- Several bad numbers: `names tasks 4 and 5`.
- An unnamed session reads `the next Pomodoro`.
- Named create or again:
  `` `=#plan~1` names task 1, but PLAN starts as a new session with no queued Task Links; start it with `=#plan` ``.
- Guard errors win over range errors. For example, a running session still reports the
  existing "still running" error. That error now echoes the **full typed token**
  (`… then `=~2` to switch sessions`) instead of the `={spec.raw}` it builds today.

Warning (non-blocking, on the dropped row and top-level): when a dropped block link is
still queued under another number, report
``task 2 `[[bob#^a]]` is still queued as task 4``.

### Output contracts (schema version 1, additive only)

`bob capture` JSON, `kind: "pomodoro_start"`, whole-item starts only:

- `text` is the raw token (`=3#bugs~1,3`).
- `pomodoro_start.drop`: the typed `~<K>` list, ascending, omitted when empty (mirrors
  `pomodoro_close.drop`).
- `pomodoro_start.tasks[]`: the rows **still queued**, in lineup order. Each row gains
  `index` (its lineup number, so a drop leaves gaps like 1, 3) and `now: true` when the
  task line carries `#now` (use `plan_budget::has_now_tag`, as close rows do). `now` is
  omitted when false.
- `pomodoro_start.dropped[]`: rows removed by `~<K>`, omitted when empty. They have the
  same row shape plus `index`, `now`, `ledger_line` (the **pre-image** line), and
  `nested_lines` (non-blank descendant lines removed with the link, omitted when 0).
  Unresolved rows keep the explicit-null convention.
- Dropped rows stay out of `tasks`, so an older client's "N queued tasks" stays
  truthful.
- Link and task starts are byte-stable: they still carry no `tasks`.

`bob capture` human output. Example: `NO_COLOR`, `BOB_NOW=2026-09-30 09:42`, the fixture
from the worked example, `bob capture --dry-run '=~2'`. The header lines are unchanged.

```text
[dry-run] ok would start  2026/20260930.md
  CAPTURE 0945-1010 (25m) at line 5
  - [ ] (**0945-1010** [t:: 25m]) — CAPTURE
  1 [*] Add support for `=x` syntax! bob.md ^capture-stop
  dropped 2 [[bob#^web-capture]] · stays in NOW · +1 nested line
  3 [*] Restart axe sase.md ^axe-restart
  Dropped 2
plan 2/3 themes · 2/10 links
```

- Queued rows get the close's index column: right-aligned to the width of the highest
  number, bold, no outcome color.
- Unresolved rows print `warning: …` after their index.
- Dropped rows appear inline in index order, in the close's exact wording, with the NOW
  caption and then the nested-line caption (`+1 nested line`, `+2 nested lines`). A dim
  `Dropped <K>` summary follows the rows.
- If nothing is left queued, `nothing queued` prints after the summary.
- A plain `=` differs from today only by the index column.

`capture-parse`:

- `pomodoro_start` (the spec) gains `drop`, omitted when empty.
- New span kind `pomodoro_start_drop` covers `~<K>` including the `~`. When a drop part
  is present, the `pomodoro_start` span covers only `=<X>`; otherwise it is unchanged.
  Named starts keep their `pomodoro_name` span over the name bytes.
- New need `pomodoro_start_task` (dangling `~`/`,`). The partial spec and spans typed so
  far are included, plus one `interactive_placeholder` span over the dangling separator.
- Diagnostics use code `invalid_pomodoro_start`.
- The human `start` line reads `=3#bugs~1,3 (15m, offset 0u · drop 1, 3)`.
- Chain items report per-token spans, as today.

`capture-complete`:

- A cursor anywhere inside a drop part, including a dangling separator, returns an empty
  success.
- The `pomodoro_start_name` replacement range ends before `~`, so accepting a name keeps
  a typed `~<K>`.

### Visual language (shared across every surface)

| Element         | CLI                              | Editor span / Mac                                                   |
| --------------- | -------------------------------- | ------------------------------------------------------------------- |
| Lineup number   | bold index column                | `n.circle` badge (`n.circle.fill` when listed), capsule above 50    |
| Dropped row     | `dropped 2 [[T]] · stays in NOW` | `minus.circle` gray glyph, struck and dimmed row, caption           |
| Drop list token | —                                | `pomodoro_start_drop`, the same muted gray as `pomodoro_close_drop` |
| Summary         | dim `Dropped 2`                  | `Dropped 2` caption row                                             |
| NOW             | `· stays in NOW` on dropped rows | mint `NOW` capsule on every `now` row                               |

### Worked example (use it in docs and tests)

Setup: `BOB_NOW=2026-09-30 09:42:00`, day file `2026/20260930.md`, TAB indentation. In
`bob.md`: `^capture-stop` and `^web-capture` are `[*]`, and `^web-capture` carries
`#now`. In `sase.md`: `^axe-restart` is `[*]`.

```markdown
## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[bob#^capture-stop]]
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
    - remember the URL parser
  - [[sase#^axe-restart]]
- [ ] () — SASE
```

`bob capture '='` numbers the lineup: 1 `^capture-stop`, 2 `^web-capture`, 3
`^axe-restart`. `bob capture '=~2'` writes:

```markdown
## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[bob#^capture-stop]]
- [ ] (**0945-1010** [t:: 25m]) — CAPTURE
  - [[bob#^capture-stop]]
  - [[sase#^axe-restart]]
- [ ] () — SASE
```

Other rows on this fixture:

- `=~2,3` leaves only `^capture-stop`.
- `=~1,2,3` starts an empty session: `Dropped 1, 2, 3`, then `nothing queued`.
- `=~4` fails with `… CAPTURE has 3 queued Task Links (1–3)`.
- `=#sase~1` fails with ``… SASE has no queued Task Links; start it with `=#sase` ``.
- `=~` is incomplete.
- Once CAPTURE is running, `=x =~2` closes it and then starts the next session without
  link 2 of that session's own lineup (the links `=x` carried into it), as `--dry-run`
  shows.
- In each case, `bob.md` and `sase.md` are untouched.

## Cross-cutting rules for every phase

- **Contracts are additive only.** Existing JSON keys, `schema_version: 1`, exit codes,
  and byte output for inputs that do not use `~` stay unchanged. The only intended
  exception is the new index column on start rows. New fields are omitted when
  empty/false/zero. Bob Mac Capture must decode every new field tolerantly and keep
  working against an older `bob`, and an older Mac app must keep working against the new
  `bob`.
- **One lexer.** Execution, `capture-parse`, completion, and chains all use the same
  drop-list lexer and the same `markers.rs` messages. There is no second copy of the
  grammar.
- **bob-cli checks.** `just all` (fmt, clippy, tests) must pass. Keep new or touched
  Rust files under about 1500 lines. Split into directory modules or new sibling files
  rather than growing `capture/output.rs` (≈1415 lines), `capture/pomodoro_start.rs`
  (≈1004), or `tests/cli/capture/pomodoro_whole_item.rs` (≈1147). Put new CLI tests in
  new files such as `tests/cli/capture/pomodoro_start_drop.rs` and
  `tests/cli/capture/parse_pomodoro_start_drop.rs`. Update `README.md` and
  `docs/capture.md` in the same phase that changes behavior. No new subcommands or
  options are added.
- **bob-mac-capture checks.** Linux hosts have no Swift toolchain, so green macOS CI on
  the pushed commit is the gate. Follow the repo's established commit and push flow for
  opened repos. Regenerate every new JSON fixture from a real `bob` built from bob-cli
  master (`cargo build`), following existing fixture conventions and
  `Tests/Fixtures/fake-bob`.
- **Epic workers** record `PROPOSED FOLLOW-UP:` notes on their own phase bead instead of
  creating beads.

## Phase: start-lineup — numbered start lineup and the drop engine

This is bob-cli. The goal is the ledger engine and the reporting. No grammar changes:
after this phase, `=~2` is still prose.

1. **Lineup rows.**
   - In `src/native/capture_pomodoro_start.rs`, give `StartLink`/`StartTaskRow` a
     1-based `index` in ledger order and a `now` flag. For `now`, read the resolved
     task's line through the same `line_text_at(..).is_some_and(has_now_tag)` pattern
     that `capture_pomodoro_close/linked_tasks.rs` uses.
   - If the module would pass about 1500 lines, convert it to a directory module
     (`capture_pomodoro_start/{mod.rs, lineup.rs, drop.rs, tests…}`).
2. **Drop engine.** Add a pure function, roughly
   `plan_start_drop(contents, entry_index, drop: &[u32], owner: &StartOwner, token: &str) -> Result<StartDropPlan, StartDropError>`.
   - Number the lineup on the pre-image.
   - Validate every number. Collect all out-of-range numbers into one error, with the
     exact wording from Design; `StartOwner` supplies the name, "the next Pomodoro", or
     the created-session variant.
   - Remove each dropped bullet plus its descendants. Reuse or promote
     `child_block_end_line`: a private copy exists in `capture_task_toggle.rs`, so make
     one `pub(crate)` rather than adding a third copy.
   - The subtrees are disjoint siblings, so remove the ranges back to front. Preserve
     CRLF, and keep a missing final newline missing.
   - Return the updated contents, the dropped rows (index, pre-image `ledger_line`,
     `nested_lines`, block link), the kept indices, and the duplicate warnings.
   - `entry_index` is unchanged because removals come after the entry line.
3. **Plumbing.**
   - Add `drop: Vec<u32>` to `PomodoroStartSpec`
     (`#[serde(default, skip_serializing_if = "Vec::is_empty")]`). Every parser still
     leaves it empty in this phase.
   - In `src/native/capture/pomodoro_start.rs`, both `plan_pomodoro_start_item` and
     `plan_named_pomodoro_start_item` apply the drop after their guards and before
     `start_existing_pomodoro_entry`.
   - A created or "again" named session with a non-empty drop fails before insertion.
   - Factor the duplicated `tasks → PomodoroStartTaskJson` mapping into one helper. It
     zips the post-image lineup with the kept indices and resolves the dropped rows
     against the same `SnapshotCloseVault`.
   - Make the running-session error echo `parsed.body`.
4. **JSON.**
   - `PomodoroStartTaskJson` gains `index` and `now` (omitted unless true). The dropped
     variant also carries `nested_lines` (omitted when 0).
   - `PomodoroStartSummary` gains `drop` and `dropped`, both omitted when empty.
   - Put warnings on the dropped row's `warning` and push them into the top-level
     `warnings`.
5. **Human output.** Implement the Design rendering in
   `print_human_pomodoro_start_success`. Move start rendering into its own file if
   `output.rs` would pass about 1500 lines. Keep `NO_COLOR` and piped output ANSI-free.
6. **Docs.**
   - In `docs/capture.md` "Starting the next Pomodoro" and "Starting a named Pomodoro",
     describe the numbered lineup and `tasks[].index`/`tasks[].now`.
   - In `bob capture --help` (`src/native/capture/cli.rs`), add one sentence saying
     start rows are numbered.
   - The drop fields are documented in start-drop-grammar, once they are reachable.
7. **Tests.**
   - Engine unit tests:
     - in range;
     - out of range: none, one, many, several bad numbers, and the created-session
       variant;
     - nested notes and nested links removed with their parent;
     - embedded rows;
     - unresolved rows;
     - fenced lines not numbered;
     - duplicate-still-queued warning;
     - CRLF;
     - a dropped last line with no final newline.
   - CLI tests: the numbered human output and JSON `index`/`now` for `=`, `=3`, and a
     named open start. Link and task starts stay byte-stable.
   - Existing start tests still pass, updated only for the index column.

## Phase: start-drop-grammar — `=[<X>][#name]~<K>` grammar, editor support, and docs

This is bob-cli. It makes `=~<K>` real end to end, for `bob capture` and for editors.

1. **Lexer.**
   - Extend `EqualsToken::Start` in `capture_language/item.rs` with the drop part (its
     offset inside the token and its text). The name part stops at `~`, and `len` covers
     the whole token.
   - Add `capture_language/start_selection.rs` with
     `lex_start_drop(after_tilde, base_offset, token)`. It returns
     `Valid { drop, drop_range }`,
     `Incomplete { drop, drop_range, separator_range, separator }`, or an error with a
     message and range, as in Design.
   - Reuse `close_selection.rs`'s number-list parser and error builders (promote them to
     `pub(super)`) instead of duplicating them. New messages go in `markers.rs`.
   - Lexer unit tests cover every row of the Design table, including ranges.
2. **Execution parser.** Update `parse_pomodoro_equals_item`:
   - A drop part claims the item.
   - An exact single-line token yields `CaptureKind::PomodoroStart` with `spec.drop`
     filled (named or not).
   - A dangling separator fails with `close_selection_incomplete_error`.
   - With extra text or child lines, a broken list reports its own error first, then the
     no-spaces hint when the spaceless join lexes, else the shape error.
   - `is_session_chain_token` accepts any start token with a drop part, near misses
     included, so `=x =~2` and `=x =~` split into chain items.
3. **Wiring.** Pass `spec.drop` into the start-lineup engine. Batches and chains then
   work through `CaptureBatchPlanner` with no further changes. Verify rollback when a
   later item fails.
4. **Editor parser.** Update `parse_editor_start_item`/`parse_editor_named_start_item`
   (`editor_pomodoro.rs`):
   - Add spans per the Design.
   - Add `SpanKind::PomodoroStartDrop` (`"pomodoro_start_drop"`) and
     `Need::PomodoroStartTask` (`"pomodoro_start_task"`).
   - Incomplete outcomes carry the partial spec and an `interactive_placeholder` span.
   - Diagnostics must match execution byte for byte in message and range.
   - The `capture-parse` human `start` line gains `· drop …`.
5. **Completion.** `pomodoro_start_name_field` stops at `~`. A cursor inside the drop
   part returns an empty success. Update the `capture-complete` help paragraph that
   lists non-completable suffixes.
6. **Docs.**
   - `docs/capture.md`:
     - grammar-at-a-glance rows for `=[<X>][#<name>]~<K>`, plus the examples table rows
       from Design (valid, incomplete, errors, and `= ~2` staying prose);
     - a new "Dropping queued Task Links as a session starts" subsection under "Starting
       the next Pomodoro", with the one-card mnemonic, the worked example, the
       lineup/numbering rules, subtree removal and why it differs from close drops, the
       hooks and `#now` note, and the diagnostics and warnings;
     - JSON field notes (`drop`, `dropped[]`, `nested_lines`);
     - the named-start section (`=3#bugs~1`);
     - the chaining section: the session-token list gains the drop form, plus the
       `=x =~2` vs `=x~2 =` contrast;
     - the `capture-parse` and `capture-complete` sections.
   - `README.md` grammar rows.
   - `bob capture --help`, and `capture-parse --help` (examples `'=~2'`, `'=3#bugs~1'`,
     `'=x =~2'`, plus `pomodoro_start_task` in the Needs list).
7. **Tests.**
   - CLI tests on the worked-example vault:
     - bare, counted, offset, and named forms;
     - named-created errors;
     - out-of-range errors;
     - every lexical error through `bob capture`;
     - `--dry-run` writes nothing;
     - the batch `=x`, blank line, `=~2`;
     - the chains `=x =~2` and `=~2 +2`;
     - batch rollback;
     - the CRLF day file;
     - the running-session error echoing `=~2`;
     - `=~2` no longer becoming a task.
   - `capture-parse` JSON tests: modes, specs, spans, needs, placeholder, diagnostics
     and ranges, and chain items.
   - Completion tests: the named replacement stops before `~`, and a cursor in the list
     returns an empty success.

## Phase: mac-start-drop — numbered, drop-aware start card

This is bob-mac-capture. Build and test only against real `bob` output from bob-cli
master with both earlier phases landed.

1. **Models** (`Sources/CaptureCore/CaptureModels.swift`). Decode every new field
   tolerantly:
   - `PomodoroStartSpec.drop` (default `[]`);
   - `PomodoroStartTask.index` (`Int?`), `now` (default false), and `nestedLines`
     (`nested_lines`, default 0);
   - `PomodoroStartSummary.drop` (default `[]`) and `dropped` (default `[]`).
2. **Editor.**
   - `pomodoro_start_drop` maps to a new `.pomodoroStartDrop` semantic category. It
     renders in exactly the gray `.pomodoroCloseDrop` uses, and the palette switch stays
     exhaustive.
   - Add `pomodoro_start_drop` to `completionSpanKinds`, as `pomodoro_close_drop` is.
3. **Pending state.** Generalize `closePendingTrim` so an item needing
   `pomodoro_start_task` is trimmed the same way.
   - The live preview runs on the trimmed draft (`=~` previews `=`, `=~2,` previews
     `=~2`).
   - The start card renders dimmed, with `Type a task number after ~` (or `,`), reusing
     `pendingText`.
   - Status reads `… — Start is disabled`, **Start** is disabled, and Return cannot
     submit.
   - Chains trim per item, as closes do.
4. **Start card** (`CapturePomodoroStartPresentation.swift` and `startPreviewItem` in
   `CapturePanelView.swift`).
   - The rows are the merge of `tasks` and `dropped`, sorted by `index`.
   - Each row gets the close card's numbered badge: `n.circle`, or `n.circle.fill` for
     dropped (listed) rows, and the capsule fallback above 50. Rows without `index`
     (older `bob`) render exactly like today's card.
   - Numbered rows never hide under `+N more`.
   - Dropped rows use the `minus.circle` gray glyph, struck and dimmed text (0.5), and a
     caption joining `stays in NOW` (for `now` rows) and `with N nested line(s)`.
   - Every `now` row gets the mint `NOW` capsule, identical to the close card's.
   - **Teaching hint.** Tint `~N` like the drop span and `#name` as today:
     - with no drop typed and a non-empty lineup, a bare or counted start reads
       `Type ~2 to drop task 2 · #name to start a specific Pomodoro`
       (`Type ~1 to drop it · …` for one row);
     - a named start reads `Type ~2 to drop task 2` or `Type ~1 to drop it`;
     - an empty lineup keeps today's `Type #name to start a specific Pomodoro`, and
       named starts with an empty lineup show no hint.
   - **Summary.** Once a drop is typed, show `Dropped 2, 4`, plus
     ` · nothing left queued` when no kept rows remain.
   - **Status and notification.** Status text gains ` · drops 2` (dry run) or
     ` · dropped 2` (committed). The notification body gains ` · dropped 2`, and its
     queued count counts `tasks` only.
   - **Accessibility.** Dropped rows read `Task 2, <text>, drops from today`. Queued
     rows read `Task 1, <text>, queued`.
   - A failed dry run (for example, out of range) keeps today's red error block with
     Bob's message unchanged.
5. **Fixtures and tests.**
   - Add real-bob fixtures and `fake-bob` branches:
     - capture dry-run for `=` (numbered, `now`), `=~2` (with `nested_lines`),
       `=~1,2,3`, and `=#name~1`;
     - `capture-parse` for `=~2`, `=3#bugs~1`, `=~` (incomplete), `=~2,`, `=~0`
       (invalid), and `=x =~2` (chain).
   - Presentation tests: numbering, merge order, dropped styling data, captions, hint
     variants, summary, and old-bob compatibility. Also cover the panel-model tests for
     the pending trim and span mapping.
6. **README.** Update the start-card paragraph, the pending-list paragraph, and the
   runtime-contract field list.
7. **Gate.** Push following the repo's flow. The phase is done only when macOS CI is
   green for the commit.

## Deliberately not doing

- **No drop on link or task starts** (`^route:id=~2`). The drop list is whole-item only,
  and those suffixes keep their current errors.
- **No `!<M>` or keep list on starts.** Starting never changes task status, and the
  digits after `=` are timing.
- **No task-note writes.** Next→Ready demotion stays with `bob task-status-hooks`, as
  for close drops.
- **No carrying or moving dropped links elsewhere.** To move a link to another session,
  use `@route+block-id#pomodoro`.
- **No task-number picker popup in the Mac app.** The numbered start card is the picker,
  as it is for closes.
- **No SASE memory changes.**
