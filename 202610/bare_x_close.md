---
tier: epic
title: Bare x Pomodoro close replaces =x
goal: "Closing today's running Pomodoro takes a bare `x` token with the same selection
  grammar as today's `=x` (`=x2!1*3` becomes `x2!1*3`). The `=*`/`=!` shorthands are
  gone. bob, its docs, and Bob Mac Capture teach only the new spelling.

  "
decisions:
  x_case:
    ask: Which spellings of the bare close token should close the running Pomodoro?
    choices:
      lower:
        "`X post about the launch` and `X11 fix` stay tasks; capitalizing is the escape"
      any:
        "`X…` closes too, like today's `=X`; tasks starting with the word X must be
        reworded"
    default: lower
    why: Capital X starts real prose (X posts, X11, X86); `=X` was only a convenience
    answer: lower
  old_close:
    ask: What should a whole-item `=x…` do once the bare `x…` close lands?
    choices:
      retire:
        Fails with a teaching error that shows the `x…` rewrite; one spelling everywhere
      alias: Keeps working as a hidden alias that docs, help, and diagnostics never show
    default: retire
    why:
      You asked to replace `=x`; an error retrains muscle memory instead of hiding a
      twin
    answer: retire
  glossary_strands:
    ask:
      Update the Capture and Bulk Capture glossary strands to spell the close `x` (`+2
      x`, `x =`)?
    memory:
      - glossary:capture
      - glossary:bulk-capture
    default: true
    corrects:
      or start, adjust, shift, or close today's Pomodoro (`=`, `+N`, `++N`, `=x`).
    answer: true
  decision_record:
    ask:
      Add a decisions-web record for the bare `x` close and its rejected alternatives?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: bare-x
    title: Bare x close in both parsers
    depends_on: []
    size: medium
    description:
      "bare-x: teach the shared capture grammar a bare `x…` whole-item close (plain,
      selection-shaped, `x#…` near miss, chains, inline Work Log entries) next to
      today's `=x…`, with parity tests proving identical behavior."
  - id: retire
    title: Retire =x and the =*/=! shorthands
    depends_on:
      - bare-x
    size: medium
    description:
      "retire: turn the whole-item `=x…`/`=*…`/`=!…` shapes and the link `=*`/`=!`
      suffix aliases into teaching errors, respell every message, help text, and
      comment, and migrate the bob-cli test suite to `x…`."
  - id: docs
    title: Docs and memory teach x
    depends_on:
      - retire
    size: medium
    description:
      "docs: respell the whole-item close as `x` across docs/capture.md, README.md, and
      the other docs, document the retired forms and prose collisions, and apply the
      accepted glossary and decision-record memory edits."
  - id: mac
    title: Bob Mac Capture teaches x
    depends_on:
      - retire
    size: medium
    description:
      "mac: update Bob Mac Capture's hint text, teaching strings, fake-bob drafts,
      regenerated JSON fixtures, Swift tests, and README for the bare `x` close."
proposed_by: bbugyi200.athena.11i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-18 14:36:55
status: wip
---

- **PROMPT:**
  [prompts/202610/bare_x_close.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bare_x_close.md)

# Plan: Bare `x` Pomodoro close replaces `=x`

## Context

Today the running timed Pomodoro closes with the whole-item token
`=x[<N>][*<P>][!<M>][~<K>][@<N>]` (case-insensitive `=X`). The short aliases `=*…`/`=!…`
omit the `x`, and the same selection can follow a link suffix (`@route:id=x…`,
`^route:id=x…`, `<text> @route:id=x…`). Closes also chain on one line (`+2 =x`, `=x =`,
`=x =#bugs`, `=x wired the lexer =`) and take an inline Work Log entry (`=x 1 fixed it`)
or child bullets.

The user wants the whole-item close to drop its `=`. For example, `=x2!1*3` becomes
`x2!1*3`. Since `x!`/`x*` are as short as `=!`/`=*`, those shorthands go away too.

How the grammar is wired today:

- **One shared gate.** Both parsers reach a whole-item close through
  `session_equals_token` (`src/native/capture_language/item.rs`), which returns `None`
  unless the text starts with `=`.
  - The execution dispatcher calls it from `parse_capture_item` →
    `parse_pomodoro_equals_item`.
  - The editor dispatcher (`capture-parse`, completion, rewrite) calls it from
    `parse_editor_item_with` → `parse_editor_close_item`
    (`src/native/capture_language/editor_pomodoro.rs`).
- **Shared helpers.** Both sides use:
  - `src/native/capture_language/close_selection.rs`: `close_after_eq_parts`,
    `classify_whole_item_close`, `whole_item_close_after_x`, `classify_link_close`,
    `link_close_after_x`, `lex_close_selection`.
  - `src/native/capture_language/close_log.rs`: inline entries and bullets.
  - `src/native/capture_language/draft.rs`: same-line chain split through
    `is_session_chain_token`, `is_chain_close_token`, `is_trail_head_token`, and
    `split_inline_close_tokens`.
- **Diagnostic text.** Most of it lives in `src/native/capture_language/markers.rs`.
- **Mac app.** Bob Mac Capture is a thin client: its close logic follows bob's spans,
  needs, and summaries. Only its hint strings, fixtures, tests, and README spell the
  close.

## Design (applies to every phase)

### The new token

A whole-item close is a first parent-line token that is one of these:

- exactly `x`;
- `x` immediately followed by a digit, `,`, `!`, `~`, `*`, or `@` (selection-shaped);
- `x#…`, the claimed close-hash near miss.

Everything after the `x` is today's selection grammar, byte for byte. That covers
`x1,3!2`, `x0`, `x*`, `x!`, `x*!2`, `x1*`, `x~2`, `x@2`, `x!1@2`, plus dangling
separators as incomplete states (`x1,`, `x~`, `x@`).

Every other `x` shape stays ordinary prose: `xx`, `xa`, `x.`, `x-ray`, `xylophone`,
`x:`, and mid-body tokens such as `Plan x` and `a x1`.

> [!decision] x_case = lower Only lowercase `x` closes. Capitalized `X`, `X2`,
> `X post about the launch`, `X11 forwarding`, and `X *job:cleanup` stay ordinary task
> text. Capitalizing the first letter is the documented escape for a task that must
> start with the word `x`.

> [!decision] x_case = any `x` and `X` both close (case-insensitive, like today's `=X`).
> Every `X…` shape listed above is claimed exactly like its lowercase twin. Tests and
> docs that use `X` as task text (the `X *job:` blocked-task rows) switch to other
> prose. Tasks that must start with the word X must be reworded.

### Same behavior as today

- **Identical results.** A bare close behaves exactly like today's `=x` close with the
  same selection: same planner, writes, JSON kind `pomodoro_close`, editor mode, needs
  (`pomodoro_close_task`, `pomodoro_close_log_text`), and span kinds.
- **Inline entries.** `x wired the lexer` logs to task 1, and `x2,3 2 foo bar` logs
  `foo bar` to task 2. `x 1` is incomplete, `x @2 foo` reports the no-spaces error, and
  `x0 wired` reports the no-default error.
- **Bullets.** `x` plus `- 1 wired` works, and so do positional bullets.
- **Chains.** `+2 x`, `x =`, `x =#bugs`, `x =~2`, `x~2 =`, `x wired the lexer =`, and
  `- x` (shorten, then close). A bare `x` token counts as a chain token, a close owner,
  and a trail head everywhere `=x` does today.
- **What else changes meaning.** These mostly become refusals or closes instead of inbox
  tasks:
  - `x more` becomes a close with an inline entry.
  - `x86 build` claims `x86` as a selection, so it fails unless task 86 exists.
  - `x *job:cleanup` is refused by `reject_operator_dependencies`.
  - An inline entry ending in a standalone close-shaped token closes again (for example,
    `x fixed it x`), just as a trailing `=x` does today.
- **Not affected:** `!note:id` completion, `+`/`:`/`/` pickers, checkbox `[x]`, `@@`
  declarations, gkeep imports, and URL lists.

### What `=` still means

- **Starts are unchanged.** `=`, `=<X>`, `=#name`, `=~K`, `==`, `==#name`, and the rest
  of the start family stay as they are. Mixed chains keep working: `x =` closes, then
  starts.
- **`==` near misses.** `==x`, `==X1`, `==*`, and `==!2` stay claimed teaching errors.
  Their message now teaches the bare spelling: "close it with `x`", with the tail
  normalized (`==*2` → `x*2`, `==!` → `x!`).
- **Link and task suffixes keep `=x…`.** `@route:id=x…`, `^route:id=x…`, and
  `<text> @route:id=x…` are unchanged, because the `=` there separates the suffix from
  the block ID, exactly as in `^route:id=3`. Dropping it would make `^bob:stopx`
  ambiguous, and a space would save no keystroke. Bob Mac Capture's active-task teaching
  line "Then type #name, = to start, or =x to close" therefore stays correct.

### Retired forms

- **`=*` and `=!` shorthands are removed** everywhere, as requested.
  - Whole-item `=*…`/`=!…` (including malformed ones like `=*abc`) stay claimed. They
    fail with a teaching error that names the rewrite: `=*1` → `x*1`, `=!` → `x!`,
    `=*!2` → `x*!2`.
  - Link and task suffix aliases (`^bob:stop=*`, `@r:id=!2`) fail with a teaching error
    naming `=x*` / `=x!2`.
- **Error text and placement.** The phase settles the exact wording, but every
  retired-form error names the typed token and the exact rewrite. One message helper in
  `markers.rs` serves both parsers.
  - In the editor it is mode `pomodoro_close` with an `invalid_pomodoro_close`
    diagnostic over the token.
  - In `bob capture` it is an item error.
  - In a chain, each retired token fails on its own range.

> [!decision] old_close = retire Whole-item `=x`, `=X`, selection-shaped `=x…`/`=X…`,
> `=x… <entry>`, and `=x#…` are also claimed retired forms. They fail with the same
> teaching error and its exact rewrite:
>
> - `=x2!1` → `x2!1`
> - `=X` → `x`
> - `=x wired` → `x wired`
> - `=x#bugs` → `x =#bugs`
>
> `=xx`, `=xa`, and mid-body `Plan =x` stay prose, as today.

> [!decision] old_close = alias Whole-item `=x…`/`=X…` keep working exactly as the
> bare-x phase leaves them: a hidden alias, never shown in docs, help, completion, or
> diagnostics, with `raw` echoing what was typed. `=x#…` keeps its hash near miss, but
> the message teaches `x =#name`. Only the `=*`/`=!` shorthands retire.

### JSON and contract

- **No new fields.** `capture-parse` and `bob capture` keep schema version 1 and every
  key.
- **`pomodoro_close.raw`** stays "exactly as typed":
  - the bare token (`x`, `x1*2!3~4`) for whole-item closes;
  - the `=`-prefixed suffix for link and task closes (`=x!1`).
- **Spans.** The whole-item `pomodoro_close` span covers the 1-byte `x`, so the
  selection spans after it start one byte earlier than with `=x`.
- **Human output.** `format_pomodoro_close` (`src/native/capture_parse.rs`) prints `raw`
  as typed and no longer prepends `=`.

### Out of scope

- **bob-plugins:** two source comments (`bob-ledger-tools/src/060-today.js`,
  `task-status-cycler/src/085-successor-plan.js`) mention `=x` historically. No behavior
  depends on them, and editing them means rebuilding `main.js`, so leave them.
- **Decision records:** records under `sase/memory/decisions/` that say "an `=x` close"
  stay as written, because accepted records are immutable and the phrase still names the
  close gesture.
- **History:** the vault's `done/bob_done.md` archive is history.

## Phase: bare-x

Add the bare close next to `=x` without retiring anything, so the whole suite stays
green and the parity tests prove "works the same way".

1. **`close_selection.rs`.** Add one classifier for bare tokens: plain `x`, a
   selection-shaped `x…`, or `None`. It returns the text after `x` and the prefix length
   1, following the casing rule in Design.
   - Extend `classify_whole_item_close` and `whole_item_close_after_x` so they accept
     bare tokens alongside `=x…`/`=*…`/`=!…` and report the right prefix length (1 for
     bare, 2 for `=x`, 1 for the aliases).
   - Keep the link path (`classify_link_close`, `link_close_after_x`) untouched.
2. **`item.rs`.**
   - **The gate.** Make the shared gate (`session_equals_token`, or a renamed or
     wrapping helper with updated docs) return `EqualsToken::Close` when the first
     whitespace-delimited token is a bare close or `x#…`. It must do this without
     consuming starts or changing any `=`-family lexing.
   - **Other recognizers.** Update `is_session_chain_token` (the Close arm),
     `is_close_hash_token` (also `x#`), and `double_close_tail_is_claimed` (unchanged
     semantics).
   - **Close arm of `parse_pomodoro_equals_item`.** Replace the
     `first.eq_ignore_ascii_case("=x")` prose fallback and the `plain` check (the
     dangling-entry hint) with a plain-close predicate covering `=x`/`=X`/bare `x`.
   - **Offsets.** `close_after_x_offset` is already prefix-agnostic. Audit every other
     place that assumes a 2-byte prefix.
3. **`editor_pomodoro.rs`.**
   - In `parse_editor_close_item`: take `prefix_len` from the classifier (1 for bare)
     instead of falling back to 2, start the `x#…` diagnostic range at
     `token_start + prefix_len`, and generalize the prose fallback.
   - Check that `cursor_in_close_log_text` recognizes bare heads, so completion stays
     suppressed on close items and in their inline entry text.
4. **Other shared code.**
   - **`draft.rs`.** Chain split, inline owner, and trail head must pick up bare tokens
     through the gate. Add any missing case.
   - **`close_log.rs`.** The reorder and no-spaces hints work with a 1-byte token
     (`format!("{close_token}{first}")` must render `x@2`).
   - **`editor_classify.rs`, `tokens.rs`, `completion.rs`.** Audit for `=`-assumptions
     on the whole-item path; the link path stays as it is.
5. **Human output.** `src/native/capture_parse.rs` `format_pomodoro_close` prints `raw`
   verbatim.
6. **Tests that used `x` as prose.** These now become closes or refusals; switch them to
   other prose without changing what they test:
   - `tests/cli/capture/complete_editor.rs` (`"x @dev:some-id#…"`)
   - `tests/cli/capture/complete_block_id.rs` (`"x @sase:"`)
   - `src/native/capture_language/tests/completion.rs` (around the `"x …"` drafts)
   - under `x_case = any`, also `tests/cli/capture/complete_blocked_task.rs` and
     `tests/cli/capture/parse_blocked_task.rs` (`X *…`)

   Find any others with `cargo test` and `rg -n '"[xX][ ,!~*@0-9]' tests src`.

7. **New tests.**
   - **Parity table.** Unit tests under `src/native/capture_language/tests/` assert that
     the editor and execution parsers produce the same specs, needs, and diagnostics
     (shifted one byte) for each pair. Pairs: `x`/`=x`, `x1,3!2`, `x0`, `x*`, `x!`,
     `x*!2`, `x1*`, `x~2`, `x@2`, `x!1@2`, `x1,`, `x~`, `x@`, `x 1`, `x more`,
     `x2,3 2 foo`, `x @2 foo`, `x` + bullets, `x#bugs`. Only `raw` differs.
   - **Prose rows.** `xx`, `xa`, `x.`, `x-ray`, `xylophone`, and `Plan x` stay prose.
     Under `x_case = lower`, so do `X`, `X2`, `X post about the launch`, and
     `X *job:cleanup`.
   - **Chain rows.** `+2 x`, `x =`, `x =#bugs`, `x =~2`, `x~2 =`, `x wired the lexer =`,
     and `x =` + `- 1 wired it`.
   - **CLI tests.** Under `tests/cli/capture/` (`pomodoro_close.rs`,
     `pomodoro_close_selection.rs`, `pomodoro_close_log.rs`, `pomodoro_chain.rs`,
     `parse_pomodoro_close.rs`): one real write each for `bob capture x`, `x1!2`,
     `x wired the lexer`, and `x =`, asserting the same ledger bytes as the `=x` twins.
8. **Verify** with `just check` (fmt, clippy, tests). Then run a manual smoke in a
   scratch vault: set `BOB_DIR` and `BOB_NOW` as the existing CLI tests do, and run
   `bob capture --dry-run x1!2` and `bob capture-parse x2`.

## Phase: retire

Retire the old forms, respell everything bob prints, and migrate the test suite. The
decisions are final here; read them with `sase bead read`.

1. **Retired whole-item forms.** Implement them as specified in Design (per `old_close`)
   in both parsers and in chains. Use one shared message helper in `markers.rs` that
   computes the rewrite (`=*1` → `x*1`, `=X2` → `x2`, and under `retire`, `=x#bugs` →
   `x =#bugs`).
   - `close_after_eq_parts` is shared with the link path, so split the whole-item and
     link handling explicitly rather than deleting branches.
2. **`==` near misses.** Respell them to teach `x…` (`pomodoro_double_close_error`
   callers in `item.rs` and `editor_pomodoro.rs`).
3. **Link and task suffix `=*…`/`=!…`.** Make them a claimed teaching error naming
   `=x*…`/`=x!…`, on every parser path that classifies link suffixes today:
   - `tokens.rs`: `parse_task_block_id_route_token`, `parse_colon_link_tail`,
     `caret_tail_has_close_suffix`, `classify_caret_close_suffix`;
   - `editor_classify.rs`: `classify_pomodoro_token`.

   Link `=x…` stays valid.

4. **Respell user-facing text.** Every whole-item close reference becomes `x…`; text
   about the link suffix keeps `=x…`. Shared selection diagnostics describe the
   selection after `x` and use `x…` examples on every form. Known sites:
   - **`markers.rs`:**
     - `BANG_SUFFIX_HINT` ("use `x!N`")
     - `POMODORO_CLOSE_FORCED_ERROR`
     - `pomodoro_close_hash_error` ("write `x =#name`")
     - `close_selection_zero_alone_error`, `close_selection_bad_list_error`, the
       one-bang/one-tilde/one-star/one-at and keep-twice errors
     - `close_inline_reorder_error`, `close_inline_no_spaces_error`,
       `close_inline_dangling_error`
     - `start_drop_bang_error` ("`x!3`")
   - **`close_log.rs`:** `not_worked_suggestions` (`x{index}`, `x0…`, `x!{index}`).
   - **Template session error:** in `draft.rs` and `capture_parse.rs`.
   - **`src/native/capture/`:** `pomodoro_close.rs` forced-flag errors;
     `pomodoro_start.rs` ("close it with `x`", the `x =…` switch hint);
     `pomodoro_link.rs`.
   - **`src/native/capture_pomodoro_close/selection.rs`.**
   - **`src/native/task_status_hooks/sync.rs`:** "close all but one with
     `bob capture x`".
   - **Help text:** the `capture/cli.rs`, `capture_parse.rs`, `capture_complete/cli.rs`,
     and `capture_rewrite.rs` grammar prose and examples. Drop the `=*`/`=!` mentions
     and the "zsh expands `=word`" warning for closes. Keep the advice to single-quote
     tokens containing `!`, `*`, or `~`.
   - **`model.rs`:** the `raw` field docs (per the JSON section above).
   - **Gotcha:** `editor_parse.rs` matches the prefix of a "`=x` always closes" message
     on the caret-link path. Keep that match working against the link message, or
     replace it with a typed check.
5. **Comments and docstrings** in `src/` that name the whole-item spelling become `x`.
   Afterwards, `rg -n '=x|=X' src tests` should list only:
   - link-suffix forms;
   - URL literals (`?v=x`, `utm_source=x`);
   - the gkeep literal-preservation test in `src/native/gkeep/render.rs`;
   - the new retired-form tests.

   Check each remaining hit.

6. **Migrate the tests** (about 1,500 occurrences, roughly 60 files: the unit tests in
   `capture_language/tests/`, `close_selection.rs`, `close_log.rs`, `capture_parse.rs`,
   `capture_pomodoro_close/*tests.rs`, `capture_complete/tests/output.rs`, plus
   `tests/cli/capture/*.rs`, `tests/cli/help.rs`, and
   `tests/cli/task_status_hooks/structure.rs`).
   - **Scripted first pass.** Rewrite `=x`/`=X` to `x` only where the `=` follows the
     start of a string, a quote, whitespace, a newline, or `\n`. Never after an
     identifier character (that is a link suffix) or after another `=` (that is `==x`).
   - **Then review every hunk by hand:** span ranges shift by one, `raw` assertions,
     messages.
   - **Alias tests** (`=*…`, `=!…`) become `x*…`/`x!…` cases, deduplicated where the
     long form already covers them.
   - **Link alias tests** become `=x*…`/`=x!…`.
   - **Add retired-form tests** in both parsers and on the CLI: whole-item retired
     tokens alone, with an inline entry, inside a chain (`=x =`), the link suffix
     aliases, and the `==` messages.
   - Under `old_close = alias`, keep a small hidden-alias parity test instead of the
     `=x` retired-form tests.
7. **Verify** with `just check`, then `rg` (step 5). Do a dry-run smoke of each retired
   form and its rewrite in a scratch vault.

## Phase: docs

Bring every bob-cli doc and the accepted memory in line with the shipped grammar. Quote
the real messages the retire phase landed by running bob, never from memory.

1. **`docs/capture.md`** (about 270 occurrences). Respell every whole-item `=x` row,
   example, and table as `x…`, and remove the `=*`/`=!` rows. Then:
   - add a retired-forms table (whole-item `=x…` per `old_close`, `=*…`, `=!…`, link
     `=*`/`=!`, `==x`) with the exact errors;
   - document the casing rule and the prose rows (`xx`, `xylophone`, `Plan x`, and, per
     `x_case`, the `X…` rows) plus the collision notes (`x more`, `x86 build`,
     `x *job:cleanup`, the capitalize escape under `lower`);
   - update the lifecycle table and mnemonic ("`x` stops the running one");
   - rewrite the recognition paragraph (it currently says `=x`/`=X` keep their close
     meaning because `x` is never a start suffix), the shell-quoting advice, and the
     `pomodoro_close.raw` field notes;
   - keep the link `=x…` rows, explaining that the `=` is the suffix separator.
2. **Other docs.** Respell whole-item `=x` in `README.md` (grammar table, Pomodoro
   section, examples), `docs/getting-started.md`, `docs/completion.md`,
   `docs/task-status-hooks.md`, `docs/task-dependencies.md`, `docs/freshness.md`, and
   `docs/plan.md`. Leave link-suffix rows and URL `=x` (`docs/ref-jobs.md`,
   `docs/highlights-ref-sync.md`) alone.
3. **Memory.** Use `/sase_memory_write`.

> [!decision] glossary_strands In `sase/memory/glossary/capture.md`, change the close
> example from `=x` to `x`. In `sase/memory/glossary/bulk-capture.md`, change the chain
> examples from "(`+2 =x`, `=x =`)" to "(`+2 x`, `x =`)".

> [!decision] decision_record Add a decisions-web strand (for example
> `bare-x-closes-the-pomodoro`) that cites this epic's commits. It should record:
>
> - **Claim:** a whole-item close is a bare `x…` token (casing per `x_case`); link
>   suffixes keep `=x…`; `=*`/`=!` are gone.
> - **Rejected alternatives:** keeping `=x`; case-insensitive `X` (if `lower` won);
>   respelling link suffixes; keeping the shorthands.
>
> Ask Bryan for any rationale no source records.

After any memory edit, run `sase memory init`. 4. **Verify.**
`rg -n '=x|=X|=\*|=!' docs README.md sase/memory/glossary` should show only link
suffixes, URLs, and retired-form documentation. Every quoted message must match bob's
actual output.

## Phase: mac

Bob Mac Capture follows bob's spans, so no Swift logic recognizes the close. This phase
only fixes what the app _says_ and what its tests feed it.

1. **Open the repo.** Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`.
   Build bob from the bob-cli checkout (`cargo build`) so the fixtures come from the
   shipped grammar.
2. **Hint and teaching text.**
   - `Sources/CaptureCore/CapturePomodoroClosePresentation.swift` `hintTokens`: lead
     with `x*`/`x!` (drop the `=*`/`=!` lead), then `x*1`/`x!1`, `x1,2`, `x*2`, `x!2`,
     `x~N`, `x0`, `x 2 wrote the tests`. Keep the span-category colors: the close `x`
     takes the Pomodoro-session color that `=` and `=x` use today.
   - If these hints also render on link or task close cards, spell them with `=x…`
     there. Decide this from bob's reported `pomodoro_close.raw` (`=`-prefixed means a
     suffix), never by parsing the draft.
   - `Sources/CaptureCore/CompletionRowContent.swift`: "close it first with x."
   - `Sources/CaptureCore/CaptureAgendaPresentation.swift`: the multiple-sessions
     message.
   - Comments naming the whole-item `=x` or the aliases (`CaptureModels.swift`
     `PomodoroCloseSpec`, `CapturePanelModel.swift`, `CaptureAgendaLayout.swift`,
     `CapturePomodoroResetPresentation.swift`).
   - Leave `Sources/BobMacCapture/CapturePickerView.swift`'s "Then type #name, = to
     start, or =x to close" alone; it describes the link suffix.
3. **Fixtures and fake-bob.**
   - In `Tests/Fixtures/fake-bob`, respell the draft keys and inline JSON for whole-item
     closes.
   - Regenerate every affected `Tests/Fixtures/*.json` with the new bob (same drafts,
     respelled; same scratch-vault setup the fixtures record).
   - Replace the `pomodoro-close-*-alias-*` fixtures with `x*`/`x!` equivalents, plus
     one retired-form diagnostic fixture. Delete pure duplicates.
4. **Swift tests.** Respell drafts and expectations in the tests listed by
   `rg -l '=x|"=\*|"=!' Tests`, for example `CaptureAgendaModelTests.swift` (`=*abc`,
   `=x1,`), `CaptureCloseTaskCommaAssistTests.swift`,
   `CapturePomodoroClosePresentationTests.swift`, `CapturePanelModelTests.swift`, and
   `BobProcessClientTests.swift`.
5. **README.** Respell every whole-item `=x`, `=*`, and `=!` mention (the close grammar
   notes, the auto-comma table row, the Pomodoro section, the agenda numbers). Keep the
   link-suffix mentions.
6. **Verify** per `AGENTS.md`:
   - Run `just portable-check` on athena.
   - On the mac, best effort: compile a scratch copy and run `bundle.sh` and
     `just typecheck-tests`.
   - Confirm CI's Lint/Build/Test steps pass.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.
   - **Rollout:** install the new bob before or with the new app build. An older app on
     a newer bob still works, because only its hint text is stale.
