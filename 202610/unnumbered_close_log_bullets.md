---
tier: epic
title: Unnumbered =x Work Log bullets
goal: 'Work Log bullets under a whole-item `=x` close need a leading task number only
  when bullet order cannot say which task each bullet logs to. `=x3,4` plus `- foo
  bar` and `- baz bam` writes exactly what `- 3 foo bar` and `- 4 baz bam` write,
  and `bob capture`, `bob capture-parse`, dry-run previews, and Bob Mac Capture all
  agree on that.

  '
phases:
- id: bob-cli
  title: Positional Work Log bullets in bob-cli
  depends_on: []
  size: medium
  description: 'bob-cli: lex unnumbered first-level bullets (all or none), assign
    them in order to the close''s worked tasks (lexically when `<N>`/`*<P>` is typed,
    against the running session otherwise), make `CloseLogEntry.index` optional with
    a positional origin, report resolved indices in `bob capture` JSON, add the new
    diagnostics, and update help, docs, and tests.'
- id: mac
  title: Bob Mac Capture decoding, fixtures, and docs
  depends_on:
  - bob-cli
  size: small
  description: 'mac: decode an absent `log[].index` as nil, add real-bob parse and
    preview fixtures plus tests for unnumbered bullets, and rewrite the README''s
    Work Log bullet contract and syntax text; no grammar logic moves into Swift.'
proposed_by: bbugyi200.apollo.47
create_time: 2026-10-02 15:17:47
status: done
bead_id: bob-cli-3l
---

- **PROMPT:** [prompts/202610/unnumbered_close_log_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/unnumbered_close_log_bullets.md)
- **BEAD:** [bob-cli-3l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3l/README.md)

# Plan: Unnumbered `=x` Work Log bullets

## Context

A whole-item close (`=x[<N>][*<P>][!<M>][~<K>]`, case-insensitive, plus the `=*`/`=!`
shorthands) takes Work Log bullets as child lines. Today every first-level bullet must
start with a task number:

```text
=x3,4
- 3 foo bar
- 4 baz bam
```

`- foo bar` fails with
``start each Work Log bullet with the number of the task it logs to: `- 3 foo bar` ``.
The user wants the number only when it is needed to disambiguate. The draft above should
be writable as:

```text
=x3,4
- foo bar
- baz bam
```

and produce byte-identical vault writes and the same `bob capture` JSON.

Where things live today:

- **Shared lexer.** `src/native/capture_language/close_log.rs`. `lex_close_log_bullets`
  lexes bullets for both execution (`capture_language/item.rs`,
  `parse_pomodoro_equals_item`) and the editor (`capture_language/editor_pomodoro.rs`,
  `parse_editor_close_item`). The missing-number error is raised at lines ~747–765.
  Nearby helpers:
  - `is_loggable` and `default_log_index` encode the selection-mode rule: `<N>` typed
    (including `=x0`) or a non-empty `*<P>` means only `<N> ∪ *<P> ∪ !<M>` can take
    entries.
  - `log_entries_from_lex` builds `CloseLogEntry { origin: Bullet }`.
- **Model.** `src/native/capture_language/model.rs`.
  - `PomodoroCloseSpec.log: Vec<CloseLogEntry>`.
  - `CloseLogEntry { index: u32, text, details, origin }`, with `origin` marked
    `#[serde(skip)]`.
  - `CloseLogOrigin::{Bullet, Inline{..}}`.
- **Runtime.** `src/native/capture_pomodoro_close/selection.rs`.
  - `apply_close_selection` numbers the session's Task Links, assigns outcomes, runs
    `validate_close_log_entries` (`LogOutOfRange` / `LogDeferred` / `LogDropped` /
    `LogNested`, worded per `CloseLogOrigin`), then inserts each entry under its link by
    `entry.index`.
  - `first_worked_link` already defines "worked": outcome in progress, parked, or
    complete, and sitting at the session's top level.
  - `linked_tasks.rs::plan_pomodoro_close` takes `log_entries` from `sel.log`.
  - `capture/pomodoro_close.rs` (~436) builds JSON `pomodoro_close.log` from the _parsed
    spec_.
- **Selection lists.** `close_selection.rs::validate_selection` sorts all four lists
  ascending and rejects any number that appears in two lists.
- **Bob Mac Capture.** It is a thin client (decision `mac-capture-is-a-thin-client`): it
  spawns `bob capture-parse`, `bob capture-complete`, and `bob capture --dry-run`, and
  renders what comes back. No Swift code parses `- <n>`. The only digit-dependent code
  is `closePendingTrim`, and only for dangling numbered bullets (`=x\n- 1`), which stay
  as they are. `PomodoroCloseLogEntry.index` is decoded with `decodeIfPresent ?? 0` and
  never read by UI code. Presentation uses Bob's per-row `tasks[].typed_work_log`.

## Semantics (the contract both phases implement)

### Numbered and unnumbered bullets

- A first-level bullet is **numbered** when its first whitespace-separated token is all
  ASCII digits. This is exactly today's test, so `0`, `007`, and overflow still report
  today's index errors. Every other first-level bullet is **unnumbered**, and its whole
  body is entry text.
- A leading number is always a task number: `- 2 bugs fixed` names task 2. To log text
  that starts with a number, number every bullet (`- 3 2 bugs fixed`).
- **All or none.** The first first-level bullet (placeholder rows such as `- ` are
  skipped) fixes the list's kind.
  - **Numbered list:** behavior is byte-for-byte today's. A later unnumbered bullet gets
    the missing-number error, `- 1` alone is dangling/incomplete, and loggability errors
    are unchanged.
  - **Unnumbered list:** a later numbered bullet fails with a new mixed-numbering error
    (see Diagnostics). Dangling bullets cannot occur in an unnumbered list.
- Two-space nested detail bullets are unchanged. They attach to the nearest preceding
  entry, numbered or not.
- Inline entries (`=x [<n>] <text>`) and their default (`default_log_index`) are
  unchanged.

### Worked tasks `W`, in ascending number (ledger) order

- **Selection mode** (`<N>` typed, including `=x0`, or a non-empty `*<P>`):
  `W = sorted(<N> ∪ *<P> ∪ !<M>)`. This is lexical, so `capture-parse` and `bob capture`
  resolve identically.
- **Otherwise** (`=x`, `=x!2`, `=x~2`, `=x!2~3`): `W` is the running session's numbered
  Task Links whose outcome after the selection is in progress, parked, or complete _and_
  that sit at the session's top level. This is the `first_worked_link` predicate,
  generalized to return the full list. Only `bob capture` (including `--dry-run`) can
  resolve it, because it depends on the ledger (`[[T]]#` defers, `![[T]]` completes,
  nested links cannot take entries).

### Assignment rule

For `B` unnumbered entries in typed order:

1. `|W| = 0` → error: the close works no task.
2. `|W| = 1` → every bullet logs to `W[0]`.
3. `|B| ≤ |W|` → bullet `i` logs to `W[i]`. A single bullet therefore logs to the first
   worked task, matching the inline default.
4. `|B| > |W| ≥ 2` → error: which task takes the extra entry is ambiguous.

Implement the rule once, as
`pub(crate) fn assign_log_positions(worked: &[u32], count: usize) -> Result<Vec<u32>, PositionalLogError>`
in `close_log.rs`, with
`PositionalLogError::{NoWorkedTask, TooManyBullets { worked: usize }}`. The lexer and
the runtime each call it and word their own diagnostic.

| Draft                                                            | Result                                                                      |
| ---------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `=x3,4` + `- foo bar` + `- baz bam`                              | `foo bar` → 3, `baz bam` → 4; identical files and JSON to the numbered form |
| `=x2` + `- a` + `- b`                                            | both → 2                                                                    |
| `=x1!3` + `- a` + `- b`                                          | `a` → 1, `b` → 3                                                            |
| `=x1*2` + `- a` + `- b`                                          | `a` → 1, `b` → 2                                                            |
| `=x3,4` + `- a`                                                  | `a` → 3                                                                     |
| `=x3,4` + `- a` + `- b` + `- c`                                  | lexical error on `- c`                                                      |
| `=x0` + `- a`                                                    | lexical error: works no task                                                |
| `=x` on [1 `[[A]]`, 2 `[[B]]#`, 3 `[[C]]`] + `- a` + `- b`       | `a` → 1, `b` → 3 (2 is deferred)                                            |
| `=x` on a session with one worked top-level link + three bullets | all three → that link                                                       |
| `=x~1` on [1, 2, 3 plain] + `- a` + `- b`                        | `a` → 2, `b` → 3                                                            |
| `=x` + `- 1 a` + `- b`                                           | today's missing-number error on `b`                                         |
| `=x` + `- a` + `- 2 b`                                           | new mixed-numbering error on `2`                                            |

### Decisions and rejected alternatives

- **Order-based prefix rule, not strict count equality.**
  - Requiring `|B| = |W|` would make every intermediate draft an error. While typing the
    first of two bullets, the Mac live preview would flash red on every keystroke.
  - It would also make `=x\n- foo` disagree with `=x foo`, which already defaults to the
    first worked task.
  - The prefix rule only fails when order truly cannot decide (more bullets than worked
    tasks). The close preview shows each typed entry under the row it maps to before
    anything is written.
  - If the user wants strict equality, flip rule 3 to "`|B| = |W|`". Nothing else
    changes.
- **All or none, not mixing.** Unnumbered bullets interleaved with numbered ones have no
  clear position semantics. "Continue the previous bullet's task" conflicts with
  positional mapping.
- **Ascending number order.** Selection lists are already sorted, so typed order is
  gone. Ledger order is also the order the close card shows.
- **Run-time resolution without a selection.** A lexical guess (`1, 2, 3, …` minus
  `~<K>`) would map onto deferred or nested links and could not apply rule 2. Runtime
  already validates log entries against the lineup (`LogOutOfRange` / `LogDeferred` /
  `LogNested`), so this follows precedent. The consequence is that `capture-parse` omits
  `index` for those entries.
- **Non-goal:** the inline default stays lexical (task 1 for plain `=x`). `=x foo` and
  `=x` + `- foo` differ only when plain `=x`'s task 1 is not a top-level worked link.
  There, the inline form keeps its existing "name a worked task" error. Note this in the
  phase's final report as a possible follow-up; do not change inline behavior.
- **No new `needs`, span kinds, or schema bumps.** JSON grows additively: the only shape
  change is that `capture-parse` may omit `log[].index`. `bob capture` JSON always
  carries the resolved index.

### Diagnostics (all `invalid_pomodoro_close`; nothing written; batches roll back)

Wording may be polished, but each message must keep the quoted key phrases, because
tests assert them. Message builders live in `capture_language/markers.rs` (lexical) and
in the `CloseSelectionError` `Display` impl (runtime). Join number lists with the
existing `join_numbers` style (`3 and 4`, `1, 2 and 4`).

- **Numbered list, later unnumbered bullet** (unchanged range: the bullet's first
  token). Keep the existing prefix and add the rule:
  ``start each Work Log bullet with the number of the task it logs to; bullets are numbered all or none: `- 1 wired the lexer` ``.
- **Unnumbered list, later numbered bullet.** The range is that bullet's index token.
  This check runs before that bullet's index-validity and loggability checks:
  ``Work Log bullets are numbered all or none, and the first one has no task number; to log text that starts with `2`, number every bullet: `- 3 2 bugs fixed` ``.
  The example uses `default_log_index(..).unwrap_or(1)` plus the bullet's whole body.
- **Lexical too many.** The range is the first excess bullet's text, from its first
  token's start to its last token's end:
  ``\`=x3,4\` works 2 tasks (3 and 4) but has 3 unnumbered Work Log bullets; start each bullet with the number of the task it logs to: \`- 3 foo bar\` ``.
- **Lexical none** (`=x0`, `=x0~2`). The range is the first bullet's text. Use
  `not_worked_suggestions(1, …)` as `close_inline_no_default_error` does:
  ``\`=x0\` works no task, so its Work Log bullets have none to log to; list one (\`=x1\`) or complete one (\`=x0!1\`)``.
- **Runtime too many.** The owner is the session name, or "the running Pomodoro" via
  `owner_name`:
  ``\`=x\` works 2 of CAPTURE's tasks (1 and 3) but has 3 unnumbered Work Log bullets; start each bullet with the number of the task it logs to: \`- 1 wired the lexer\` ``.
- **Runtime none.**
  - With zero numbered links:
    ``\`=x\` has Work Log bullets, but CAPTURE has no numbered Task Links to log them to``.
  - Otherwise:
    ``\`=x\` works none of CAPTURE's top-level Task Links, so its Work Log bullets have none to log to; list one in \`<N>\` or \`!<M>\` to log to it``.
- **Runtime validation of a lexically resolved positional entry.** In practice only
  `LogNested` is reachable, for a listed task nested under another bullet. Give
  `CloseLogOrigin::PositionalBullet` its own head in every `Log*` arm,
  `Work Log bullet 2 logs to task 4 by its position, but …`, ending with the existing
  remedy text. Its `worked` suggestion is `None`, as for `Bullet`.

## Phase `bob-cli`: Positional Work Log bullets in bob-cli

### Lexer: `src/native/capture_language/close_log.rs`

- Change `CloseLogEntryLex.index` to `Option<u32>` (`None` = positional, resolved at
  execution) and `index_range` to `Option<(usize, usize)>` (`None` = unnumbered bullet).
- Rework `lex_close_log_bullets` as a single top-to-bottom pass:
  - The first first-level bullet fixes the kind.
  - Per-bullet errors still win in source order, including the two mixed-numbering
    errors.
  - After the loop, if the list is unnumbered and no error occurred, resolve positions:
    - In selection mode, compute `W` lexically (add a small
      `lexical_worked_tasks(in_progress, park, complete) -> Option<Vec<u32>>`) and apply
      `assign_log_positions`. Map failures to the lexical messages and ranges above.
    - Otherwise leave every index `None`.
  - Dangling bullets keep today's behavior, and any error still outranks a dangling
    bullet.
- `log_entries_from_lex`: numbered entries use `CloseLogOrigin::Bullet`, and unnumbered
  entries use `CloseLogOrigin::PositionalBullet { position }` (1-based typed position).
- Add `assign_log_positions` and `PositionalLogError` (shared with the runtime), plus a
  small helper that reports whether a close item's first first-level child bullet is
  unnumbered. The inline-mixing suggestion below uses it.
- Update the module doc comment: the bullet rules, all or none, `W`, and the assignment
  rule.

### Model: `src/native/capture_language/model.rs` (and the `mod.rs` re-exports)

- Change to `CloseLogEntry.index: Option<u32>` with
  `#[serde(skip_serializing_if = "Option::is_none")]`. Document that `None` means an
  unnumbered bullet under a close without `<N>`/`*<P>`, which `bob capture` resolves
  against the running session.
- Add `CloseLogOrigin::PositionalBullet { position: u32 }`.
- Refresh the `PomodoroCloseSpec` doc comment (`- [<n>] <text>`).
- Every constructor of `CloseLogEntry` in code and tests now passes `Some(n)` for
  numbered and inline entries.

### Execution parse (`item.rs`) and editor parse (`editor_pomodoro.rs`)

- Both paths keep calling the shared lexer. Lexer errors already flow through, as an
  execution error string and as an editor `invalid_pomodoro_close` diagnostic with the
  lexer's range.
- Editor: push `PomodoroCloseLogIndex` spans only for entries with an `index_range`.
  Positional entries get no index span, like a defaulted inline entry. A valid
  unnumbered list reports mode `pomodoro_close` with no diagnostics.
- Inline entry plus bullets (`close_inline_mixing_error`, built in both files): when the
  inline entry used the default number _and_ the close's first first-level bullet is
  unnumbered, suggest `- {text}` instead of `- {index} {text}`. Otherwise the suggested
  fix would trigger the mixed-numbering error.
- `src/native/capture_parse.rs::format_pomodoro_close`: print `log "text" (+N detail)`
  (no number) when `index` is `None`. Keep `log 2 "text"` otherwise.

### Runtime: `capture_pomodoro_close/selection.rs`, `linked_tasks.rs`, `capture/pomodoro_close.rs`

- Generalize `first_worked_link` into
  `top_level_worked_links(lineup, spans, entry_index) -> Vec<u32>`. `first_worked_link`
  becomes `.first()`.
- In `apply_close_selection`, after outcomes and the conflicting-duplicate check and
  before `validate_close_log_entries`: if any log entry has `index == None` (they are
  then all positional), resolve them with
  `assign_log_positions(top_level_worked_links(..), count)`. Map failures to new
  `CloseSelectionError` variants:
  - `LogPositionalNone { raw, total, running_name }`
  - `LogPositionalTooMany { raw, worked: Vec<u32>, bullets: usize, running_name, first_text }`
- Validate and insert the _resolved_ entries. Return them as a new
  `AppliedCloseSelection.log: Vec<CloseLogEntry>` (every index `Some`).
- `validate_close_log_entries`, `insert_close_log_entry`, and the `Display` arms read
  the resolved index. Add the `PositionalBullet` wording. Treat it like `Bullet` for
  `worked` suggestions.
- `linked_tasks.rs::plan_pomodoro_close`: take `log_entries` from the applied
  selection's resolved log, not `sel.log`, so `write_logs` and `warn_missing_typed_logs`
  see real indices.
- Add `log: Vec<CloseLogEntry>` (resolved, typed order) to `PomodoroCloseSummary`.
- `capture/pomodoro_close.rs`: build JSON `pomodoro_close.log` from the summary's
  resolved log instead of `spec.log`, so `bob capture` and `--dry-run` JSON always
  report the index each entry actually logged to.
- `selection_from_spec` needs no change: `has_log()` already forces a selection for
  plain `=x` with bullets.

### Help and docs

Follow `sase/memory/cli_rules.md` (read it with `/sase_memory_read`).

- **`src/native/capture/cli.rs`** (`bob capture` help, ~241–262) and
  **`src/native/capture_parse.rs`** (capture-parse help, ~160–195): describe
  `- [<n>] <text>`, all or none, in-order assignment to worked tasks (rules 2–4 in one
  sentence), and that a leading number is always a task number. For capture-parse, also
  say that `log[].index` is omitted when `bob capture` resolves it.
- **`docs/capture.md`:**
  - **Grammar at a glance.**
    - Update the `=x` + `- <n> <text>` row to `- [<n>] <text>`.
    - Replace the `=x` + `- wired the lexer` error row with the user's example
      (`=x3,4` + `- foo bar` + `- baz bam`, equivalent to the numbered form).
    - Add rows for the too-many error and the all-or-none error.
  - **"Several entries on bullets" (~1725–1804).**
    - Update the syntax template.
    - Add an "Unnumbered bullets" passage covering: all or none, `W` per mode, the
      assignment rule, the examples table above, the leading-number rule with its
      escape, and the new diagnostics.
    - Replace "Loggability is lexical with no vault access, so `capture-parse` and
      `bob capture` agree" with an accurate statement: numbered bullets and
      selection-mode positions are lexical; positions under a close without `<N>`/`*<P>`
      resolve against the running session at execution.
  - **`pomodoro_close.log` field notes (~1340).** In `bob capture` JSON, `index` is
    always the task the entry logged to, including for unnumbered bullets.
  - **`bob capture-parse` section (~2814–2845).** Positional entries get no
    `pomodoro_close_log_index` span. `index` is omitted for entries resolved at
    execution. List the new diagnostics.
- **`README.md`** grammar table, `=x` row (~242): mention that unnumbered bullets log in
  order.
- Refresh stale doc comments: the `tests/cli/capture/pomodoro_close_log.rs` header (it
  still says entries are "never text on the close line").

### Tests

**New unit tests in `close_log.rs`:**

- The user's example resolves to `(Some(3), "foo bar")`, `(Some(4), "baz bam")` with no
  index ranges.
- `=x2` with two bullets → both 2. `=x1!3` → 1, 3. `=x1*2` → 1, 2. `=x3,4` with one
  bullet → 3.
- Too many: message phrases and the range of the excess bullet.
- `=x0`: none.
- Plain `=x`, `=x!2`, and `=x~2` → indices `None`.
- Details under positional entries attach correctly.
- Both mixed-numbering orders: messages and ranges.
- `- 2 bugs fixed` alone is numbered task 2.
- A direct `assign_log_positions` table test.

**New runtime tests in `capture_pomodoro_close/selection_tests.rs`:**

- Plain `=x` with positional entries on `[plain, deferred, plain]` → 1, 3.
- A single worked link absorbs three bullets.
- `~2` and `!2` change `W`.
- A nested link is excluded.
- Too-many and none messages, including an unnamed session.
- A lexically resolved positional entry hitting a nested listed link gets the
  `PositionalBullet` `LogNested` wording.
- `AppliedCloseSelection.log` is fully resolved.

**New integration tests in `tests/cli/capture/pomodoro_close_log.rs`:**

- **Acceptance.** Build a fresh vault whose running session has at least 4 numbered Task
  Links. Run the user's numbered and unnumbered drafts on two identical copies. Assert
  identical day and task files, and an identical `pomodoro_close.log`
  (`[{index:3,…},{index:4,…}]`) and `tasks[]`.
- Plain `=x\n- wired the lexer` on `close_worked_vault` (link 1 plain, link 2 deferred)
  logs to task 1 and reports `log[0].index == 1`.
- `--dry-run` JSON reports resolved indices.
- Positional errors write nothing and roll back a batch.
- `=x =` plus unnumbered bullets attaches them to the close.
- `capture-parse`:
  - `=x3,4` plus two unnumbered bullets → `log` with indices 3 and 4, and no
    `pomodoro_close_log_index` spans.
  - `=x\n- foo` → mode `pomodoro_close`, and `log[0]` has no `index` key.
  - The error drafts → `invalid_pomodoro_close` with precise ranges.
- Extend
  `capture_language/tests/completion.rs::work_log_bullet_lines_request_no_marker_completion`
  with an unnumbered bullet. Completion stays suppressed.

**Existing tests to update**

These assert the old missing-number error on drafts that are now valid. Turn each into a
positive positional assertion, or into a mixed-numbering case where the test is about
error attribution:

- `tests/cli/capture/pomodoro_close.rs`, ~490–497 and ~760–781 (`=x\n- detail` now logs
  `detail` to task 1).
- `tests/cli/capture/pomodoro_close_log.rs`, ~275 (execution errors table) and ~654
  (parse diagnostics table).
- `tests/cli/capture/parse_pomodoro_close.rs`, ~395–407 (`=x1\n- detail` now logs to
  task 1).
- `src/native/capture_language/tests/editor_modes.rs`, ~1147–1155 (`=x\n- child` is now
  a valid close with a positional entry).
- `src/native/capture_language/tests/chain.rs`, ~334–346. To keep testing "capture item
  2 starting on line 1" attribution, switch to a mixed draft such as
  `+2 =x\n- 1 a\n- child`.
- `close_log.rs` `rejects_bad_bullets` and `default_log_index_covers_every_close_shape`
  (the `=x~1` + `- wired` example). Make them mixed-numbering cases that keep the
  example assertions.
- Every `CloseLogEntry { index: n, … }` literal becomes `Some(n)`: `selection_tests.rs`,
  `linked_task_tests.rs`, `capture_parse.rs` tests, and `capture/output.rs` tests.

### Validation

- `just all` (`cargo fmt --check`, clippy with all targets and features, and
  `cargo test`) passes.
- Smoke-test with the built binary:
  - `bob capture-parse -f json -- $'=x3,4\n- foo bar\n- baz bam'`
  - `bob capture-parse -f json -- $'=x\n- foo'`
  - A `BOB_NOW`/`BOB_DAY_FILE` dry run on a temp vault, for both the numbered and
    unnumbered drafts.
- Run `just install` at the end, so the installed `bob` that the `mac` phase captures
  fixtures from matches this phase.

## Phase `mac`: Bob Mac Capture decoding, fixtures, and docs

**Open the repo.** Use `/sase_repo`: `sase repo open bob-mac-capture -r "…"`. If the
linked primary checkout is missing on the host, run
`sase repo open gh:bobs-org/bob-mac-capture -r "…"` instead. Use only the printed path.
The repo has no AGENTS.md; its README's Development section is the guidance
(`just format-lint|build|test|bundle`).

**No grammar logic in Swift.** Bob does all mapping. No `needs` or span kinds are added.
`closePendingTrim` stays as is: its digit check only ever sees dangling _numbered_
bullets.

**Model.** In `Sources/CaptureCore/CaptureModels.swift`, change
`PomodoroCloseLogEntry.index` to `Int?`. It is `nil` when `bob capture-parse` leaves an
unnumbered bullet for execution to resolve; today a missing value decodes as a fake `0`.
Update the doc comment, the memberwise init (`index: Int?`), and every use or test that
compares `index`.

**Fixtures.** Capture them from the real `bob` built by the `bob-cli` phase, following
the existing convention ("Real `bob capture-parse` output"). Add a
`Tests/Fixtures/fake-bob` dispatch branch per exact draft.

- **Parse fixtures:**
  - `=x3,4\n- foo bar\n- baz bam`: indices 3 and 4, no `log_index` spans.
  - `=x\n- wired the lexer\n- sketched the URL parser`: `log` entries with no `index`.
  - A mixed-numbering draft: an `invalid_pomodoro_close` diagnostic.
- **Preview fixture:** `bob capture --dry-run --no-clip --format json` for the plain
  unnumbered draft on a vault whose running session has two worked links, showing
  `typed_work_log` on both rows.

**Tests:**

- **Decode** (`CaptureCoreTests`): an entry without `index` decodes as `nil`, and
  existing numbered fixtures still decode as `1`.
- **Presentation:** the positional preview shows each typed entry on its own row.
- **Panel model** (`BobMacCaptureTests`): unnumbered drafts produce no close pending
  trim, and **Close** stays enabled. The mixed draft surfaces Bob's diagnostic.

**README.**

- Rewrite the Work Log bullet parts of the grammar-contract paragraph (~118–142) and the
  user-syntax paragraph (~925–955):
  - Bullets may omit the number. Unnumbered bullets log in order to the close's worked
    tasks, and a single worked task takes them all.
  - Bullets are numbered all or none. A leading number is always a task number.
  - `capture-parse` omits `log[].index` when `bob capture` resolves it.
  - Positional entries get no cyan index chip.
- In that same paragraph, fix the stale sentence "Text on the `=x` line itself is
  retired", which contradicts the inline-entry support at ~122–128. Check that the Work
  Log hint description (~636–638) matches the code's inline `=x 2 …` hint, and fix it if
  not.

**Validation.** Agent hosts have no Swift toolchain, and the `BobMacCapture` target is
macOS-only. Run `just format-lint` and `just test` if `swift` is available. Otherwise:

- Keep edits compile-safe by grepping every `PomodoroCloseLogEntry` and `.index` use.
- Mirror the existing test patterns exactly.
- State in the final report that macOS CI (`.github/workflows/ci.yml`) is the gate.
