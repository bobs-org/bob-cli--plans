---
tier: tale
title: Star Work Log bullets on Pomodoro closes
goal:
  A `- * <text>` child bullet under an `x` close logs the same Work Log entry to every
  worked task that no other entry in the capture logs to.
size: medium
decisions:
  all_covered:
    ask:
      Should a `- *` bullet with no task left (every worked task already has its own
      entry) fail the capture?
    choices:
      error:
        Fail with invalid_pomodoro_close and write nothing, like too many unnumbered
        bullets
      skip:
        Accept the close, write nothing for that bullet, and print a warning naming the
        dropped text
    default: error
    why:
      the close grammar never silently discards typed text, and positional overflow
      already fails
    answer: error
proposed_by: bbugyi200.apollo.bw
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 11:33:33
status: wip
---

# Plan: `- * <text>` logs one Work Log entry to every remaining worked task

## Goal

A whole-item `x` close already takes Work Log child bullets: `- <n> <text>` (numbered)
or `- <text>` (unnumbered, logged in order to the close's worked tasks `W`). Add a third
bullet form, the **star bullet** `- * <text>`. It logs the same entry (and its nested
details) to **every task in `W` that no other entry in this capture logs to**.

```text
x
- wired the lexer          # → task 1 (positional)
- * code review + cleanup  # → every other worked task (2, 3, …)
```

`*` is the same "every remaining" sigil the close token already uses (`x*` and `x!`
select every remaining Task Link; JSON `park_all`/`complete_all`). The star bullet
extends that meaning to Work Log entries.

## Grammar contract

1. **Recognition.** A first-level bullet (any `-`/`*`/`+` marker, column 0) whose first
   whitespace-separated body token is exactly `*` is a star bullet. Everything after
   that token is literal entry text, checked with the existing bullet text rules
   (`check_bullet_text`: no block links, no leading fence). Only the exact `*` token
   counts: `- *bold* text`, `- **x**`, and `- *2 foo` stay ordinary unnumbered text. A
   leading number after the star is text: `- * 2 bugs fixed` logs `2 bugs fixed`.
   Behavior change: `- * foo` used to log the literal text `* foo`. To log text starting
   with `*`, use `- * * foo` (star) or `- 1 * foo` (numbered).
2. **Mixes with either list kind.** Star bullets are neutral in the numbered all-or-none
   rule. They never fix the list's kind, a list may start with one, and they may sit
   before, between, or after numbered or unnumbered bullets. The first _non-star_
   first-level bullet still fixes the kind, and the two existing mixed-numbering errors
   apply among non-star bullets only.
3. **Several stars are allowed.** Each star bullet logs to the same remaining set, in
   typed order.
4. **Details.** Two-space nested bullets under a star bullet are its details, the same
   as under any entry. Every expanded copy writes them.
5. **Who counts as covered.** "Other entries" are every non-star entry in the same
   close: numbered bullets plus unnumbered bullets after positional assignment. Star
   bullets never cover each other.
6. **Positional assignment ignores stars.** Unnumbered bullets are assigned with the
   existing `assign_log_positions(W, count)`, where `count` now counts only unnumbered
   (non-star) bullets. Examples: `W=[1,2,3]` with `- a`, `- * b` gives a→1 and b→2,3;
   `- * b`, `- a` gives the same. With `W=[1,2,3]` and `- a`, `- b`, `- c`, `- d`,
   `- * e` the result is still the too-many error, and its bullet count is 4, never
   counting stars.
7. **Remaining set.** `remaining = W \ covered`, in ascending task order. `W` is exactly
   the set unnumbered bullets use. In selection mode (`<N>` typed, including `x0`, or a
   non-empty `*<P>`, without wildcard intent) that is the lexical
   `sorted(<N> ∪ *<P> ∪ !<M>)`. Otherwise it is the staged session's top-level worked
   links (`top_level_worked_links`). At execution, also drop any index whose Task Link
   target `(path_part, block_id)` equals a covered entry's target, and keep only the
   lowest index per target. One task linked twice in the session then gets the star
   entry at most once and never on top of its own entry.
8. **Errors.**
   - `W` empty (for example `x0` plus only `- * foo`) reuses the existing "works no
     task" errors (`close_log_positional_none_error` lexically, `LogPositionalNone` at
     execution).
   - Nothing left (`remaining` empty while `W` is not) follows decision `all_covered`.
     Under `error`, lexical (selection mode) wording is:
     ``every task `x2` works (1 task: 2) already has its own Work Log entry, so `- *` has no task left to log to; drop the `- *` bullet``.
     Execution wording adds the session owner, mirroring `LogPositionalTooMany`:
     ``every task `x` works (2 of CAPTURE's tasks: 1 and 3) already has its own Work Log entry, so `- *` has no task left to log to; drop the `- *` bullet``.
     Reuse the existing task-list joiners (`join_task_numbers` / `join_numbers`) and
     singular/plural forms, and keep the wording identical between `capture-parse` and
     `bob capture` apart from the owner part.
   - Expanded star entries that hit the existing per-entry checks (for example a listed
     task nested under another bullet → `LogNested`) fail with that check's message
     under a new origin wording:
     ``Work Log bullet {position} (`- *`) logs to task {index} because no other entry names it, but …``
     (the rest is unchanged).
9. **Dangling star.** A star bullet with no text (`- *`) is an editing state, like
   `- 1`. `capture-parse` reports mode `incomplete` needing `pomodoro_close_log_text`,
   with an `interactive_placeholder` span over the `*` and the partial spec of complete
   entries. `bob capture` refuses it:
   ``"`- *` is incomplete: type the Work Log text after `*`"``. Details under a dangling
   star are validated and dropped, as under a dangling numbered bullet.
10. **Inline entries are unchanged.** `x * foo` keeps its stray-marker error. An inline
    entry plus any bullet, star bullets included, keeps the existing mixing error. The
    suggested bullet stays numbered unless the first _non-star_ bullet is unnumbered.

> [!decision] all_covered = skip Instead of either "nothing left" error, the lexer
> reports nothing. Execution drops the star entry, adds one capture warning
> (```- * <text>` logged to no task: every task the close works already has its own Work Log entry``),
> and still closes. `capture-parse` stays diagnostic-free for that bullet. Document it
> as accepted-with-warning instead of listing it under diagnostics. Every other rule
> above is unchanged.

## Design

Keep the star entry **unexpanded** through parsing, since `capture-parse` and the spec
carry one entry per typed bullet. **Expand it in one place:** the runtime resolver in
`apply_close_selection_with_clock`, which already resolves positional entries against
the staged lineup. After expansion, every downstream writer (ledger insertion,
`inserted_lines` ordinals, `write_logs`, `typed_work_log`, warnings, JSON) sees ordinary
entries with real indices and needs no change.

### Model (`src/native/capture_language/model.rs`)

- `CloseLogEntry`: add a JSON-visible `all: bool`, serialized as `"all": true` and
  omitted when false (reuse the file's `is_false` helper). Update the struct literals in
  tests (`capture_parse.rs`, `selection_tests.rs`, `linked_task_tests.rs`). Adding
  `Default` to the struct or using a constructor helper are both fine.
- `CloseLogOrigin`: add `StarBullet { position: u32 }`, where `position` is the 1-based
  typed position among the close's entries (the same counting `PositionalBullet` uses).
  Origin stays `#[serde(skip)]`; `all` is the JSON signal. Keep the two consistent:
  `all == matches!(origin, StarBullet { .. })`.
- Update the `PomodoroCloseSpec`/`CloseLogEntry` doc comments ("numbered all or none"
  now excludes star bullets).

### Lexer (`src/native/capture_language/close_log.rs`)

- `CloseLogEntryLex`: add `all: bool`. For a star entry set `index: None`,
  `index_range: Some(<star token range>)` (so the existing span loop gives the `*` a
  `pomodoro_close_log_index` chip, which Bob Mac Capture already renders cyan), and
  `text_range` from the first text token.
- `CloseLogDanglingBullet`: represent a dangling star, for example `index: Option<u32>`
  with `None` for `*`. Update its two consumers (`item.rs` execution error,
  `editor_pomodoro.rs` placeholder span).
- In `lex_close_log_bullets_with_all`, check for the star token before the
  numbered/unnumbered classification. Set `seen_first_level` but leave `numbered_list`
  untouched. After the loop:
  - run positional assignment over the unnumbered non-star entries only. Use their
    count, and take the too-many `excess`/suggestion from the unnumbered entries, not
    from `entries[worked.len()]`;
  - in selection mode (`lexical_worked_tasks_with_all` is `Some(W)`) with any star
    entry, compute `remaining = W \ covered` lexically and raise the `W`-empty or
    nothing-left error (per `all_covered`), pointing at the first star bullet's `*`
    token. Do **not** expand lexically; star entries keep `index: None`.
- `close_first_bullet_is_unnumbered`: skip star bullets (complete or dangling) like
  placeholder rows. A list of only stars returns `false`.
- `log_entries_from_lex`: star entries become `origin: StarBullet { position }`,
  `all: true`, `index: None`.
- Update the module doc comment to describe the star bullet.

### Diagnostics text (`src/native/capture_language/markers.rs`)

Add `close_log_star_dangling_error()` and the lexical nothing-left message helper (under
`error`). Keep diagnostic text in `markers.rs`, as the module docs require.

### Execution parser (`src/native/capture_language/item.rs`)

On a dangling star, return the star dangling error instead of
`close_log_dangling_error("- N", N)`.

### Editor parser (`src/native/capture_language/editor_pomodoro.rs`)

No structural change. Star index ranges already become `PomodoroCloseLogIndex` spans,
and dangling ranges become `InteractivePlaceholder` with `Need::PomodoroCloseLogText`.
Confirm the inline-mixing suggestion still works through the updated
`close_first_bullet_is_unnumbered`. Work Log lines already suppress marker completion
(`cursor_in_close_log_text`), so a `- *` never opens the `*note:block-id` blocked-task
picker. Add a test that locks this in.

### Runtime resolver (`src/native/capture_pomodoro_close/selection.rs`)

Replace the "any index is `None` ⇒ assign every entry positionally" block with:

1. Collect the positional entries (`index.is_none() && !all`). If there are any, resolve
   them with `assign_log_positions(&top_level_worked, positional_count)`. Map errors
   exactly as today, but compute `LogPositionalTooMany.bullets` and `first_text` over
   the positional entries only.
2. `covered` = indices (and link targets) of every non-star entry after step 1.
3. If any star entries: `W_star = lexical_worked_tasks_with_all(…)` from the selection's
   sets, else `top_level_worked_links(…)`. Compute `remaining` per contract rule 7. If
   `W_star` is empty, return `LogPositionalNone`. If `remaining` is empty, follow
   `all_covered` (`error`: a new
   `CloseSelectionError::LogStarCovered { raw, worked, running_name }` with the
   execution wording above; `skip`: drop the entry and surface a warning through the
   plan's existing warning channel).
4. Build `resolved_log` in typed order, replacing each star entry in place by one clone
   per remaining index (ascending), with `index: Some(i)`, `all: true`, and origin
   `StarBullet { position }`.

Then add `StarBullet` arms everywhere the code matches on `CloseLogOrigin` (`Display`
for `LogOutOfRange`/`LogDeferred`/`LogDropped`/`LogNested`, using the rule-8 origin
wording, and the `worked` suggestion matches in `validate_close_log_entries`, which
return `None` for stars).

### Outputs

- `bob capture` JSON (`src/native/capture/pomodoro_close.rs`):
  `PomodoroCloseLogEntryJson` gains `all` (omitted when false). Expanded entries each
  report their real `index` plus `"all": true`, keeping the "`index` is always the task
  the entry logged to" contract. Per-row `typed_work_log`/`typed_work_log_details` and
  human output need no change.
- `capture-parse` JSON: the unexpanded star entry reports
  `{"text": …, "all": true, "details": …}` with `index` omitted.
- `capture-parse` human (`format_pomodoro_close` in `src/native/capture_parse.rs`): a
  star entry renders as `* "text"`, for example
  `x (log "wired the lexer", * "code review" (+1 detail))`.

## Tests

- **Lexer unit tests** (`close_log.rs` tests): recognition (`- * foo`, `* * foo`,
  `- *foo` / `- *bold* x` / `- **x**` stay text, `- * 2 bugs` text), star first then
  unnumbered, star first then numbered, star between numbered bullets, mixed-numbering
  errors still fire among non-star bullets, positional count and too-many excess ignore
  stars, star details, dangling `- *` (with and without details below), the
  selection-mode `W`-empty (`x0`) and nothing-left (`x2` + `- a` + `- * b`) errors, the
  `close_first_bullet_is_unnumbered` skip, and `log_entries_from_lex` origin/`all`.
- **Resolver unit tests** (`selection_tests.rs`): expansion in typed order; numbered
  plus star in non-selection mode against the session lineup; selection mode using
  lexical `W`; wildcard (`x*`, `x!`) using the top-level lineup; a duplicate-target
  lineup (same task on two numbers) getting the star entry once and never on top of its
  own entry; runtime nothing-left and `W`-empty errors; and a star entry hitting
  `LogNested` with the star origin wording.
- **CLI integration** (`tests/cli/capture/pomodoro_close_log.rs`, reusing
  `close_worked_vault` / `close_four_link_vault` / `run_close_json` /
  `run_close_expect_error`):
  - `x` + `- a` + `- * b` writes `a` under link 1 and `b` (with a detail) under every
    other top-level worked link, in both the ledger and each task's Work Log;
  - JSON `pomodoro_close.log` lists expanded entries with `"all": true`, and each task
    row's `typed_work_log` includes the star entry;
  - `--dry-run` writes nothing;
  - the nothing-left case writes nothing and rolls the batch back (or, under `skip`,
    closes and warns);
  - `bob capture` refuses a dangling `x\n- *`;
  - `capture-parse -f json` reports the unexpanded entry (`all: true`, no `index`), the
    `pomodoro_close_log_index` span over `*`, and the dangling `incomplete` shape with
    its `interactive_placeholder`;
  - `capture-parse` human output shows `* "text"`.
- **Completion:** a `capture-complete` cursor after `- *` under a close returns no
  blocked-task picker.

## Documentation

- `docs/capture.md`:
  - the syntax table row for `x` + `- [<n>] <text>` (around the "Several entries while
    closing" row) gains the `- * <text>` form;
  - the examples/diagnostics table gains a valid star example, the dangling `- *`
    incomplete row, and the nothing-left error row (under `error`);
  - the **Several entries on bullets** section gets a star-bullet paragraph: rules 1–9,
    the examples from this plan, and the `- * foo` behavior change with its escapes;
  - the JSON field notes for `pomodoro_close.log` describe `all`;
  - the diagnostics list at the end of that section adds the new messages.
- CLI help: `src/native/capture/cli.rs` (`bob capture` long help, the "Bullets are
  numbered all or none" passage) and `src/native/capture_parse.rs` (the `log` entries /
  span / dangling-bullet passages). Then run `just docs-gen` so
  `docs/reference/cli/capture.md` and `docs/reference/cli/capture-parse.md` regenerate,
  and `just docs-check`.
- Do not edit `docs/reference/mac-capture/*`. Those pages are mounted from the
  bob-mac-capture repo.

## Follow-up (file, don't implement)

After the bob change lands, use `/sase_new_task` to file one small task bead for
**bob-mac-capture**. A dangling `x` + `- *` reaches `CapturePanelModel`'s pending row
with separator `*`, so it reads `Type a task number after *`. It should read
`Type the Work Log entry for the remaining tasks`. Also mention `- *` in the Mac
`docs/protocol.md` close-log paragraph. Nothing else in the app needs to change: the `*`
chip reuses `pomodoro_close_log_index`, extra JSON keys decode harmlessly, and the close
card previews star entries through the dry run's per-row `typed_work_log`. bob-plugins
do not read close Work Log data.

## Non-goals

- Inline `x * <text>` (it keeps the stray-marker error).
- Allowing an inline entry together with star bullets.
- Lexically expanding star entries in `capture-parse` (they resolve only in
  `bob capture`).
- Any change to numbered/unnumbered semantics beyond excluding stars from the
  all-or-none rule and from positional counts.

## Verification

```bash
just fmt
just lint
cargo test close_log
cargo test --test cli pomodoro_close_log
just test
just docs-gen
just docs-check
```

Release hygiene: per the daily-train decision, do not edit the `Cargo.toml` version or
`CHANGELOG.md`. The change is additive to the JSON contracts (a new optional `all` key),
so it carries no `!` marker.
