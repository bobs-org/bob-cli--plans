---
tier: tale
title: Sigil aliases on sealed project-note move bullets
goal:
  In Bob Mac Capture, `- .` / `- ;` on a first-level bullet under `@foo^bar@` or
  `@foo:bar@` become `- ^` / `- :` and open the Move picker.
size: medium
decisions:
  choice_alias:
    ask:
      Should `.` also type the second caret of the `- ^^` Move choice marker, so `- ..`
      makes `- ^^`?
    default: true
    why:
      "`- ^^` is the other start-of-bullet Move gesture, so `.` should type both of its
      carets"
    answer: true
proposed_by: bbugyi200.athena.185.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 15:53:57
status: wip
---

# Sigil aliases on sealed project-note move bullets

## Goal

The `.` → `^` and `;` → `:` capture aliases (bob rule `expand_sigil_alias`, Mac Capture
rewrite lane) should also work at the start of a first-level bullet in a sealed
project-note capture. Both seals count: `@foo^bar@` (create) and `@foo:bar@` (offer). In
Bob Mac Capture:

```text
@cash^goog-exit@          @cash:goog-exit@
- .   →  - ^  (Move)      - .   →  - ^  (Move)
- ;   →  - :  (Move&link) - ;   →  - :  (Move&link)
```

Each swap is one native, undoable edit. The Move picker (`project_task_move`) then opens
exactly as if `^` or `:` had been typed, and Command-Z restores the literal key.

## Current state (verified against today's bob)

Probes with the current debug build, using a scratch vault that holds `cash.md` with two
open tasks:

| Draft (caret at end)        | `capture-rewrite` | `capture-complete`                       |
| --------------------------- | ----------------- | ---------------------------------------- |
| `@cash^goog-exit@\n- ^`     | n/a               | `project_task_move`, replacement (20,20) |
| `@cash^goog-exit@\n- :`     | n/a               | `project_task_move`                      |
| `@cash:goog-exit@\n- ^`     | n/a               | **no context** (bug)                     |
| `@cash:goog-exit@\n- :`     | n/a               | **no context** (bug)                     |
| `@cash^goog-exit@\n- .`/`;` | `changed: false`  | none                                     |
| `@cash:goog-exit@\n- .`/`;` | `changed: false`  | none                                     |

Two gaps cause this:

1. **The alias rule never accepts the Move context.** `expand_sigil_alias` in
   `src/native/capture_language/rewrite.rs` allows only `TaskBlockId` for `.`, and
   `PomodoroBlockId` or a whole-item `TaskLink` for `;`. This was the reviewer's
   `scope = route` choice in the previous plan, `plan:202611/capture_sigil_aliases.md`.
   The user now explicitly asks for the move-bullet position.
2. **The Move picker never opens under an offer seal.** This bug predates the aliases.
   In `src/native/capture_language/completion.rs`, `has_project_note_create_marker`
   (called from `project_task_move_completion_field_at` and from both checks in
   `project_task_move_detail`) only accepts `SpanKind::ProjectNoteMarker` with
   `ProjectNoteForm::Create`. A `@route:id@` seal carries a
   `SpanKind::ProjectNoteOfferMarker` span with `ProjectNoteForm::Offer`, so it never
   matches.
   - Every other layer already treats offer seals as sealed project notes.
     `capture-parse` reports `- ^` under `@cash:goog-exit@` as `incomplete` with
     `needs: ["project_task_move"]` and parses `- ^sell` as a move.
   - The executor plans moves for `ProjectNoteForm::Offer` (`use_draft_ordered_links` in
     `src/native/capture/project_note.rs`).
   - `docs/capture.md` says `- ^^` works with both seals. Mac Capture's
     `docs/task-pickers.md` "Move Picker" says typing `^`/`:` opens it under
     `@cash^goog-exit@` _or_ `@cash:goog-exit@`.
   - The move completion landed (`b8688d9e`, 2026-11-03) after the offer form
     (`e9b9cd1d`, 2026-10-31), and the Create-only check was never widened.
   - Today, under an offer seal, Mac Capture shows the "Pick a task to move" need with
     no picker behind it.

Because the alias rule checks eligibility by asking the completion grammar
(`completion_field_at` on the swapped candidate), fixing gap 2 first means the alias
works under both seals with no seal-specific code.

Mac Capture already sends every unmodified, collapsed `.`/`;` keystroke in the main
editor through the rewrite lane (`noteSeparatorToggleIntentIfEligible` in
`CapturePanelController.swift`). It applies `expand_sigil_alias` replies natively and
requests completion afterwards (`applyNativeRewrite`). So for the single-caret case, bob
is the only gate, and Mac Capture needs no source change.

## Part 1: bob-cli

### 1a. Open the Move picker under offer seals (bug fix)

- In `completion.rs`, replace `has_project_note_create_marker` with a helper such as
  `has_sealed_project_note_marker`. It is true when the item has a
  `ProjectNoteMarker`/`Create` span **or** a `ProjectNoteOfferMarker`/`Offer` span. Use
  it at all three call sites, including the `existing_moves` scan, so moves already
  written under either seal count against the picker's rows.
- Check that the picker's Bob-authored copy is right for offer seals:
  - row `consequence` via `project_move_consequence` → `move_consequence::predict`;
  - the `project_task_move` header (`pomodoro_name` comes from `parsed.section`);
  - `action_continuation_keys`.

  Compare against what `bob capture` actually does for an authored `- ^id` / `- :id`
  under `@route:id@`. The `- ^^` Move card already serves both seals; see the form
  handling in `src/native/capture/move_choice.rs`. If an offer seal changes the outcome,
  for example a moved task that is later offered as a successor, carry the form into the
  prediction as the Move card does. Never let the copy promise something the write won't
  do.

- Tests in `tests/cli/capture/complete_project_task_move.rs`:
  - offer-seal twins of the existing MC1 and MC4 cases (`@cash:exit@\n- ^` and
    `@cash:exit@#admin\n- :`): context `project_task_move`, the same rows, and the same
    picker descriptor;
  - a lone `- ^` under an offer seal still returns `action_continuation_keys: ["^"]`.
  - In-crate completion tests: `- ^` under `@r:id@` gives a `ProjectTaskMove` field with
    an ID-only replacement.
- Docs:
  - `docs/capture.md` "Moving parent tasks in": state that the `- ^` / `- :` picker
    works with both the `@<route>^<id>@` and `@<route>:<id>@` seals, matching the
    existing `- ^^` sentence.
  - The `capture-complete` paragraph ("On a first-level `- ^` or `- :` bullet in a
    sealed project-note creation …") in `src/native/capture_complete/cli.rs`'s help:
    name both seals.

### 1b. Accept the Move context in `expand_sigil_alias`

In `src/native/capture_language/rewrite.rs`, extend the allowed table. The token rule,
candidate rule, one-byte edit, summary, and idempotence are unchanged.

| Key | Candidate field context | Replacement constraint                                | Example (caret at end)                            |
| --- | ----------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| `.` | `ProjectTaskMove`       | `replacement == (cursor, cursor)` (lone sigil, no ID) | `@cash^goog-exit@\n- .` → `@cash^goog-exit@\n- ^` |
| `;` | `ProjectTaskMove`       | `replacement == (cursor, cursor)`                     | `@cash:goog-exit@\n- ;` → `@cash:goog-exit@\n- :` |

All of these come from `scan_move_bullet` and the completion grammar:

- `*` and `+` bullet markers work too.
- A sealing child line works (the seal on a later line of the item).
- These stay literal: nested `  - .`, a checkbox `- [ ] .`, `- \.`, a `.` with text
  after it in the same token or on the line (`- .bashrc`, `- . foo`), task-list
  (`@route` header) bullets, `@route+id` items, and unsealed items.

Do **not** add trailing ` .` / ` ;` on project-note bullets (`ProjectTaskBlockId`).
Those are end-of-bullet positions outside this request.

> [!decision] choice_alias Also alias the second caret of the Move choice marker.
>
> - **Rewrite (picker closed, for example after Escape):** when the key is `.`, the
>   candidate's line is a first-level bullet in a sealed project-note item, and
>   `scan_move_bullet` on the candidate returns `MoveBulletHit::Choice` whose span ends
>   at `cursor`, rewrite `- ^.` → `- ^^`.
>   - This is lexical: use the same draft split and `classify_authored_line` path that
>     `project_task_move_completion_field_at` uses, and gate on the shared
>     sealed-project-note helper from 1a.
>   - `;` never aliases here: `- ::` is the `move_choice_colon` teaching error.
>   - The summary stays `Typed ^ for .`.
> - **Picker open:** the lone `- ^` Move picker already returns
>   `action_continuation_keys: ["^"]` (`engine.rs`, around the
>   `PickerKind::ProjectTaskMove` descriptor). Add an additive, optional picker field
>   `action_continuation_aliases: {".": "^"}`, emitted only where `"^"` is offered.
>   - Add it to `PickerDescriptor` in `src/native/capture_complete/model.rs`, skipped
>     when `None`, under `schema_version: 1`.
>   - Every other picker omits it.
>   - Document it next to `action_continuation_keys` in `docs/capture.md` (the MQ4 / "At
>     a lone `- ^`" paragraph and the JSON field list) and in
>     `docs/reference/json-outputs.md` if that page lists picker fields.
> - Tests: the rewrite cases `- ^.` → `- ^^` under both seals; `- ^;`, `- :.`, nested
>   `  - ^.`, and `- ^^.` stay literal. Completion asserts the alias map at a lone `- ^`
>   and its absence at `- :` and at a non-empty query.

### 1c. Help, docs, and tests for the alias

- `src/native/capture_rewrite.rs` `long_about`, "Sigil alias" paragraph: add the Move
  bullet position, plus the `- ^.` → `- ^^` case if `choice_alias` is on. Add one
  `after_help` example, for example
  `printf '@cash^goog-exit@\n- .' | bob capture-rewrite -c 20 -f json`. Count the byte
  offset; it must equal the draft length. Keep the examples' existing order.
- `docs/capture.md`, `## bob capture-rewrite` "Sigil alias." paragraph and the Move
  section: mention that `- .` / `- ;` on a first-level bullet of a sealed project-note
  item type `^` / `:`. Also say that a sealed project-note bullet starting with `.`
  becomes `^`, and Undo restores the `.`.
- `docs/reference/capture-syntax.md`: one short note at the move-bullet entries, in the
  style of the existing alias notes.
- `src/native/capture_language/tests/rewrite.rs`: positive cases for `- .` and `- ;`
  under both seals, a `*` bullet, a sealing child line, and multibyte text earlier in
  the draft. Negative cases from the list above. Replace any existing assertion that
  `- .` / `- ;` stay unchanged (the earlier `scope = route` tests).
- `tests/cli/capture/rewrite.rs`: one end-to-end JSON case for `@cash:goog-exit@\n- ;`.
- Run `just docs-gen` (regenerates `docs/reference/cli/capture-rewrite.md` and
  `capture-complete.md`), then `just docs-check`. Read the `docs.md` reference memory
  first. Do not edit the mounted `docs/reference/mac-capture/*` copies.
- Commit as `feat(capture): ...` with no `!`: the new rule positions, the bug fix, and
  the optional picker field are all additive. Do not touch the `Cargo.toml` version or
  `CHANGELOG.md` (release-train decision).

## Part 2: bob-mac-capture

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
first.

### Fixtures and tests (both branches)

- Generate **real bob output** from the Part 1 build into `Tests/Fixtures/`:
  - `capture-rewrite-sigil-alias-move-caret.json` (`@cash^goog-exit@\n- .`);
  - `capture-rewrite-sigil-alias-move-link.json` (`@cash:goog-exit@\n- ;`);
  - the matching `capture-complete` replies for `- ^` / `- :` under both seals. Reuse
    the existing `move-complete-mc*.json` fixtures where they already match.
- Wire these into `Tests/Fixtures/fake-bob`. Follow the existing
  `capture-rewrite-sigil-alias-*` branches and the move-picker branches that
  `CaptureProjectTaskMovePanelTests.swift` uses.
- Add tests to `CaptureSeparatorToggleTests.swift` or
  `CaptureProjectTaskMovePanelTests`, whichever already hosts the closest helpers:
  - Typing `.` at `@cash^goog-exit@\n- ` gives `- ^` and opens the Move picker. The argv
    log shows `capture-rewrite` then `capture-complete` on the swapped draft.
  - Typing `;` under `@cash:goog-exit@` gives `- :` and opens the Move picker with the
    `Move & link` header.
  - Command-Z right after restores the literal key once, with no retrigger.
  - A `changed: false` reply (older bob) leaves `.` literal and opens no picker.

> [!decision] choice_alias Mac source change, kept thin (bob authors the mapping):
>
> - Decode the optional `action_continuation_aliases` (`[String: String]`) on the picker
>   descriptor model and carry it into the move picker context next to
>   `actionContinuationKeys` (`CapturePickerState.swift` / `CapturePanelModel.swift`).
> - In `CaptureKeyCommandRouter.swift` (both continuation-key branches: the
>   `leftBracket` case and `default`), when the filter is empty and the typed character
>   is an alias whose mapped key is in `pickerOperatorContinuationKeys`, return
>   `.continuePickerOperator(mapped)`. The router context needs the alias map passed
>   alongside `pickerOperatorContinuationKeys` (see `CapturePanelController.swift`,
>   where that context is built).
> - `continuePickerOperator` then inserts `^` exactly as it does today.
> - Tests:
>   - With the lone `- ^` Move picker open, `.` gives `- ^^` and the picker closes.
>   - `.` with a non-empty filter types into the filter.
>   - An older-bob reply without the field leaves `.` in the filter.
>   - Router unit tests cover both branches.
> - Docs: `docs/task-pickers.md` Move Picker keys (`^` or `.` on an empty filter makes
>   `- ^^`).

### Docs (`just docs-check`)

- `docs/task-pickers.md` "Move Picker" intro: typing `.` or `;` after `- ` is turned
  into `^` / `:` through Bob's rewrite lane under either seal.
- `docs/keyboard.md` `.` / `;` rows: add "or after `- ` on a first-level bullet of a
  sealed project-note item".
- `docs/capture-panel.md` rewrite-lane alias paragraph: add the move-bullet position.
- `docs/protocol.md` `capture-rewrite` compatibility row: older bob leaves `- .` / `- ;`
  literal. If `choice_alias` is on, add `action_continuation_aliases` to the
  `capture-complete` picker-descriptor notes: optional, older bob omits it, and `.` then
  stays a filter key.

## Verification

- bob-cli: run `just lint` and `just test`.
  - Known baseline failures, not caused by this work: eight Clippy errors in
    `src/native/capture_language/editor_pomodoro.rs` and `past.rs`, and
    `np_move_groups_follow_requested_sources_in_cli_fixtures` (tracked in `bob-cli-h5`).
    If they are still present, report them as baseline. Make sure no new diagnostics or
    failures appear in changed files.
  - Run `rustfmt --check` on the changed files, then `just docs-gen` and
    `just docs-check`.
- Spot-check the binary with a scratch vault holding `cash.md` with open tasks:

  ```bash
  cargo run -q --bin bob -- capture-rewrite -c <N> -f json -- "$(printf '@cash:goog-exit@\n- ;')"
  cargo run -q --bin bob -- capture-complete -b <vault> -c <N> -f json -- "$(printf '@cash:goog-exit@\n- ^')"
  ```

  Also cover `- .` under `@cash^goog-exit@`, and `- ^.` if `choice_alias` is on.

- bob-mac-capture: never land uncompiled Swift.
  - On athena, run `just portable-check` and `just docs-check`.
  - The macOS targets need the Mac. Read the `tailnet.md` reference memory, rsync a
    scratch copy without `.build`/`.git`, run `Scripts/bundle.sh` and
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`. If the Mac is offline, say in the final report that the macOS
    targets are unverified.
- Manual check after bob ships, in Mac Capture:
  - `@cash^goog-exit@` ⏎ `- .` opens the Move picker on `- ^`;
  - `@cash:goog-exit@` ⏎ `- ;` opens Move & link on `- :`;
  - Command-Z after each restores the literal key;
  - with `choice_alias` on, `- ..` makes `- ^^`.

## Non-goals

- No grammar change. `bob capture` and `capture-parse` keep `- .` / `- ;` literal (a
  `.`/`;` task body), and `.`/`;` are not new sigils (decision
  `capture-sigils-say-where-what-how`).
- No trailing ` .` / ` ;` alias for project-note or task-list bullet task IDs, no
  whole-item `.` for the Active Task picker, and no `- ;;` → `- ::` (a teaching error).
- No zsh or Obsidian plugin changes.
- No memory changes. No memory note describes the Move picker's seal coverage or the
  alias positions, so none becomes false.
