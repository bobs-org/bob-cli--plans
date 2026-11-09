---
tier: tale
title: Glued . toggles the block-ID separator
goal:
  Typing `.` against a complete `@route:id` or `@route^id` flips its separator, in the
  editor and the New ID composer.
size: medium
decisions:
  task_bullet_ids:
    ask: Should a glued `.` also toggle bullet task IDs (`- Task :id.` → `- Task ^id`)?
    default: true
    why:
      Same rule and lane as the marker toggle; a key working on one but not the other
      would surprise
    answer: true
  semicolon_toggle:
    ask:
      Should `;` (the `:` alias) also switch `@route^id;` to `@route:id`, like a typed
      `:`?
    default: false
    why: You asked for `.` only, and `.` already flips both ways
    answer: false
  glossary_capture_alias:
    ask:
      Should the Capture Alias glossary list the glued `.` toggle beside the `^`/`:`
      separator toggle?
    memory:
      - glossary:capture-alias
    default: false
    answer: true
proposed_by: bbugyi200.athena.1am
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 17:51:32
status: wip
---

# Glued `.` toggles the block-ID separator (`@route:id.` ↔ `@route^id`)

## Goal

Make `.`, the easy-to-reach alias key for `^`, toggle a complete block-ID marker's
separator between `:` and `^` when it is typed right against the ID:

- `Create sase-gallery … @sase:sase-gallery.` →
  `Create sase-gallery … @sase^sase-gallery`
- `@sase^sase-gallery.` → `@sase:sase-gallery`
- sealed project-note forms: `@route:id@.` → `@route^id@`, `@route^id@.` → `@route:id@`

Today the `.` stays literal (`bob capture-rewrite` answers `changed: false`). It ends up
inside the ID, and the preview fails with "Pomodoro capture block ID must be non-empty
and contain only A-Z, a-z, 0-9 or '-'". A literal glued `.` is never valid there, so the
toggle takes nothing away. Undo in Bob Mac Capture still restores the literal `.`.

## Findings that shape the design

1. **The toggle already exists for `^`, `:`, and `+`.** `switch_block_id_separator` in
   `src/native/capture_language/rewrite.rs` consumes a trigger glued to a complete plain
   `@route:id` / `@route^id` / `@route+id` (or a sealed `@route^id@` / `@route:id@`) and
   rewrites the separator. It reports rule `switch_block_id_separator` and no `alias`
   stamp. `switch_task_bullet_id_separator` does the same for ` :id` / ` ^id` task
   bullets. Both run before `expand_sigil_alias` in `rewrite_draft`.
2. **Precedent: `=` in the relation flip.** `toggle_relation_direction` accepts `=` (the
   `~` alias) as a second trigger for the same rule. It does not add a new sigil alias,
   and the Capture Alias glossary lists such marker-editing rewrites as "not aliases".
   The `.` toggle follows this pattern. It stays inside `switch_block_id_separator`,
   reports that rule, carries `alias: None`, and the CLI help still says "five rules".
   `sigil_alias_tip` only counts `expand_sigil_alias` rewrites, so typing tips don't
   change.
3. **Bob Mac Capture's editor path needs no code change.** The controller already
   forwards a typed `.` through the rewrite lane
   (`CapturePanelController.noteSeparatorToggleIntentIfEligible`). `applyCaptureRewrite`
   applies any `switch_block_id_separator` reply as one native, undoable edit.
4. **The screenshot's path does need a Mac change.** `@sase:` on an item with text opens
   the Block ID picker's **New ID composer**. Typing `sase-gallery` goes into its filter
   field. Then `.`, a character outside Bob's ID charset, triggers **type-through**:
   `CapturePanelModel.updatePickerFilter` → `commitBlockIDTypeThrough(id:remainder:)`
   inserts `id + remainder` into the editor and only schedules analysis. It never calls
   `bob capture-rewrite`. So even with the bob-cli rule, `@sase:sase-gallery.` stays
   literal there. The same gap already affects `^`, `:`, and `+` typed through the
   composer, even though `docs/task-pickers.md` promises type-through "types exactly the
   draft it would without the picker".

## Scope decisions made here (not reviewer choices)

- **`.` after `@route+id` stays literal.** The request is a `:`↔`^` toggle. `+id` names
  an existing task, so flipping it to `^id` would almost always produce a duplicate-ID
  error. Typing `^` there still switches it as before.
- **Transfer and merge-in slots behave exactly as a typed `^`/`:` would.** Those already
  stay literal after a transfer `+id`, and the existing gates decline them. A `.`
  resolves to its target separator first and then goes through the same gates.
- **Mac type-through forwards only the block-ID separator keys** (`.`, `^`, `:`, `+`,
  `;`), and bob decides what each one does. Other type-through characters (space, `#`,
  `@`, `=`, `/`, `\`, …) keep today's behavior. Routing those through the rewrite lane
  would be a broader change than this request.

## Part A — bob-cli (this repo)

### A1. Rewrite rule (`src/native/capture_language/rewrite.rs`)

- `switch_block_id_separator`: accept a trailing `b'.'` trigger. Lex `stripped` with
  `plain_block_id_parts` as today. Resolve the effective separator: `:` → `^`, `^` →
  `:`, `+` → no toggle (return `None`). Sealed markers resolve the same way, which is
  allowed because the existing `eligible` table already lets `^` follow `@route:id@` and
  `:` follow `@route^id@`. Then run the existing flow with that resolved character as
  `trigger_char`: the eligibility match, the `+` transfer branch (unreachable for `.`),
  `merge_in_source_token_after_removing_trigger`, and `candidate_selects_plain_marker`,
  whose `trigger` argument builds the switched marker and must get the resolved
  separator, never `.`. The replacement, cursor, and `Changed <old> to <new>` summary
  are unchanged, as are rule `SwitchBlockIdSeparator` and `alias: None`.
- Update the doc comments on `rewrite_draft`, `switch_block_id_separator`, and (if
  changed) `switch_task_bullet_id_separator` to name the `.` trigger.
- Existing `.` rules must not change. These all fail `plain_block_id_parts` or
  `TaskIdLex::Valid`, so they fall through to `expand_sigil_alias` as before:
  `@errands.` → `@errands^`, `Buy milk .` → `Buy milk @`, `@home .` → `@home &`, `- ^.`
  → `- ^^`, and the sealed `- .` Move alias.

> [!decision] task_bullet_ids `switch_task_bullet_id_separator`: accept `b'.'` and set
> `new_sigil` to the opposite of the lexed `TaskIdLex::Valid { sigil, .. }`. Leave
> `TaskIdLex::Lone` alone, so `- ^.` → `- ^^` (the Move choice alias) keeps working.
> `candidate_selects_task_bullet_id` still validates the switched candidate.

> [!decision] task_bullet_ids = no Leave `switch_task_bullet_id_separator` unchanged; a
> `.` after a bullet task ID stays literal.

> [!decision] semicolon_toggle Treat a trailing `;` as `:` in both toggle functions:
> after `@route^id`, `@route+id`, sealed `@route^id@`, and a bullet `^id` (the last only
> when `task_bullet_ids` is yes). Wherever `:` stays literal, `;` stays literal too and
> falls through to the existing `;` aliases.

> [!decision] semicolon_toggle = no Do not touch `;` handling; `;` after a complete
> marker stays literal.

### A2. Unit tests (`src/native/capture_language/tests/rewrite.rs`)

Use the existing `assert_switch` helper (it also checks idempotence) and the
`rewrite_draft_separator_toggle_*` tests as models:

- `Do work @file:id.` → `Do work @file^id` and `Do work @file^id.` → `Do work @file:id`,
  with summary, single covering edit, and cursor just past the ID.
- The request's example:
  `Create sase-gallery that gives agents access to all of my personal photos (Google Photos?)! @sase:sase-gallery.`
  → `… @sase^sase-gallery`.
- Sealed: `Do work @file:id@.` → `@file^id@` and `Do work @file^id@.` → `@file:id@`.
- Alternation: three successive `.` keystrokes flip `:` → `^` → `:` → `^`.
- Leading, solo, child-line, list-item, batch-item, CRLF and Unicode neighbours, and a
  neighbouring `#TYPE` marker: add a `.` case next to the existing `^` / `:` coverage
  wherever it fits (`rewrite_draft_switches_solo_and_leading_markers`,
  `…_child_line_trailing_marker`, `…_one_item_in_a_batch`,
  `…_beside_emoji_and_preserves_crlf`, `…_leaves_type_markers_intact`,
  `…_parses_as_the_switched_mode`).
- Run membership: `@work+a @dev:b.` gets whatever `@work+a @dev:b^` gets today. Assert
  the two rewrites are equal rather than hard-coding a result.
- Requires a cursor: `rewrite_draft("Do work @file:id.", None)` is unchanged.
- Exclusions, where `rule != Some(SwitchBlockIdSeparator)` and the text is unchanged
  unless an existing `.` alias legitimately fires: `@file+id.`, `Do work @file:i.d`
  (cursor after the `.`), `@file:id#name.`, `@file:id=3.`, `@@file:id.`, `@!file:id.`,
  `^file:id.`, `&note:id.`, `!note:id.`, `@file:.`, `@file^.`, `@file^id@#bugs.`,
  `see @file:id. here` (cursor after the `.`), `Do work @file:id\n.`, and a transfer
  slot `@inbox:bench. > @db`.
- A spaced `Do work @file:id .` is not a toggle. Assert only that the rule is not
  `SwitchBlockIdSeparator`, since the spaced `.` aliases may legitimately fire.
- Regression: the existing `.` alias tests (`…_expands_dot_route_alias`,
  `…_expands_dot_dependency_alias`, `…_expands_move_choice_dot_alias`, Block ID `.`
  alias) pass unchanged.
- `sigil_alias_tip`: a draft produced by the toggle (`Do work @file^id`, cursor at end)
  still proves no tip, and the toggle rewrite carries `alias: None`.
- If `task_bullet_ids` is yes: `@dev\n- Task :task-id.` → `@dev\n- Task ^task-id`, the
  reverse, the project-note variants (`Project @cash^project-id@\n- Task :task-id.`), a
  nested line, and `@cash^goog-exit@\n- ^.` still → `- ^^`.
- If `semicolon_toggle` is yes: `@file^id;` → `@file:id`, `@file+id;` → `@file:id`,
  `@file^id@;` → `@file:id@`, and `@file:id;` stays literal.

### A3. CLI tests (`tests/cli/capture/rewrite.rs`)

- Add a JSON test modelled on `capture_rewrite_json_switches_a_block_id_separator`.
  `bob capture-rewrite -c <len> -f json -- 'Create sase-gallery! @sase:sase-gallery.'`
  reports `changed: true`, `rule: "switch_block_id_separator"`, text
  `Create sase-gallery! @sase^sase-gallery`, cursor 39, one edit
  `{21..40 → "@sase^sase-gallery"}`, summary
  `Changed @sase:sase-gallery to @sase^sase-gallery`, and no `alias` and no
  `sigil_alias_tip` field.
- Extend `capture_parse_does_not_apply_the_separator_toggle` (or add a sibling) to show
  that `capture-parse` keeps `Do work @file:id.` literal and still reports the invalid
  block ID.
- If `task_bullet_ids` is yes: extend
  `capture_rewrite_toggles_task_bullet_ids_in_lists_and_project_notes` with `.` rows.
- `capture_rewrite_help_describes_only_the_remaining_rules` keeps "five rules". Add an
  assertion that the help mentions the glued `.` toggle.

### A4. Help and docs

- `src/native/capture_rewrite.rs` `long_about`, "Block-ID separator toggle" paragraph:
  add that a glued `.` toggles between `:` and `^` (`'@file:id.'` becomes `'@file^id'`,
  `'@file^id.'` becomes `'@file:id'`, sealed `'@file:id@.'` becomes `'@file^id@'`), that
  `.` after `'@route+id'` stays literal, and the task-bullet / `;` sentences if those
  decisions are yes. In the "Sigil alias" paragraph, note that a `.` glued to a complete
  `@route:id` / `@route^id` toggles the separator instead of aliasing. Add an
  `after_help` example: `bob capture-rewrite -c 17 -f json -- 'Do work @file:id.'`.
- Run `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md`. Never
  hand-edit generated reference pages.
- `docs/capture.md` → `## bob capture-rewrite` → **Block-ID separator toggle**: add the
  `.` toggle with the `@sase:sase-gallery.` example, the sealed forms, the `+`
  exclusion, and the Undo note (Mac). In the **Sigil alias** paragraph, add one clause:
  a `.` glued to a complete marker toggles the separator (see above), while `@errands.`
  still becomes `@errands^`.
- Follow the docs memory routing (behavior change → contract/reference). Run
  `just docs-check`.
- Do not edit `CHANGELOG.md` or the `Cargo.toml` version (the release train owns them).
  This is an additive `feat(capture)`, not a `!` break: the `.` was always an invalid ID
  character there.

### A5. Verify (bob-cli)

- `just check` (fmt, clippy, every test binary) and `just docs-check`.
- Spot-check with the freshly built binary (`cargo run -- capture-rewrite …` or
  `target/debug/bob`), not the installed `bob`:
  - `t='Create sase-gallery! @sase:sase-gallery.'; target/debug/bob capture-rewrite -c ${#t} -f json -- "$t"`
  - the `^` → `:` direction
  - `@errands.` (still `@errands^`)
  - `Buy milk .` (still `@`)

## Part B — Bob Mac Capture (linked repo `bob-mac-capture`)

Open it with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`) and read its
`AGENTS.md` first. It requires compiled Swift, `just portable-check`, a best-effort Mac
type-check of `Sources/BobMacCapture*` / `Tests/BobMacCapture*`,
`Scripts/macos-ci-check` before landing, and watching the exact-SHA macOS CI run to
green with `/sase_monitor` after a push. If the Mac is offline, say in the final report
that the macOS targets are unverified.

### B1. Forward separator keys typed through the New ID composer

In `Sources/BobMacCapture/CapturePanelModel.swift`,
`commitBlockIDTypeThrough(id:remainder:)` currently applies `id + remainder` (natively,
or through `setPlainDraft`) and calls
`scheduleAnalysis(cursorUTF8Offset:requestCompletion: true, trigger: .edit)`. When
`remainder` is exactly one of `.`, `^`, `:`, `+`, `;`, after the commit succeeds:

- start
  `startCaptureRewrite(draft: <post-commit draft>, cursorUTF8Offset: <caret>, triggeredCharacter: remainder)`,
  and
- pass the returned task as the `completionGate` of that `scheduleAnalysis` call.

Do this on every success branch (splice-native, splice fallback, ordinary-native,
ordinary fallback). Model it on the `,` route-accept path in
`commitRouteCompletionOnAcceptKey`, which already does `setPlainDraft` →
`startCaptureRewrite` → `scheduleAnalysis(…, completionGate:)`.

Bob decides everything. With `changed: true`, the existing `applyCaptureRewrite` →
`applyNativeSingleEditRewrite` path applies the toggle as its own native Undo step (Undo
restores the literal post-commit draft). It announces Bob's summary and analyzes without
requesting completion, so the Block ID picker must not reopen. With `changed: false` (an
older bob, `+id`, or an ineligible spot), the draft keeps the literal key and behaves
exactly as today. Remainders other than those five characters, and multi-character
remainders (paste), keep today's behavior. Keep the controller's editor key path
untouched; it already forwards `.`.

### B2. Tests and fixtures

- `Tests/Fixtures/fake-bob`: add `capture-rewrite` answers for the new drafts, following
  the existing `switch_block_id_separator` entries:
  - `Follow up @file:new-id.` (cursor 23) → `Follow up @file^new-id`, cursor 22
  - `Follow up @file^new-id.` → `Follow up @file:new-id`
  - an unchanged answer for a `+` / unchanged case
- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`, next to
  `testBlockIDNewIDSpaceTypeThroughKeepsTyping`:
  - `@file:` composer + `new-id` + `.` settles to `Follow up @file^new-id`, with caret
    22, picker closed, status `Changed @file:new-id to @file^new-id`
  - the `@file^` → `:` direction
  - a fake-bob `changed: false` answer leaves `Follow up @file:new-id.` literal
  - space, `#`, and `@` type-throughs are unchanged (the existing tests still pass)
- `Tests/BobMacCaptureTests/CaptureListPanelTests.swift`, near
  `testTypeThroughRemainderIsOneNativeUndoStep`: after a forwarded toggle, one Undo
  restores the literal `@file:new-id.` draft.
- Never await an untimed continuation in async tests (watchdog hangs).

### B3. Mac docs (mounted into Bob's site; run `just docs-check` in that repo)

- `docs/capture-panel.md`, separator-toggle paragraph ("Typing `^` immediately after a
  complete `@route:id` …"): add that a glued `.` toggles `:` ↔ `^` (sealed forms too),
  that `.` after `@route+block-id` stays literal, and that Undo restores the `.`.
- `docs/keyboard.md`, the `.` row: add "glued to a complete `@route:id` / `@route^id`,
  `.` toggles the separator".
- `docs/task-pickers.md`, New ID type-through sentence: a single `.`, `^`, `:`, `+`, or
  `;` typed through the composer goes through Bob's rewrite lane exactly as in the
  editor, so `@sase:sase-gallery.` becomes `@sase^sase-gallery`.
- `docs/protocol.md` capability line: `switch_block_id_separator` for `^`, `:`, `+`, and
  a glued `.`. An older bob leaves `.` literal; no new capability gate is needed.

## Memory

> [!decision] glossary_capture_alias Using `/sase_memory_write`, edit the
> `glossary:capture-alias` strand. In the sentence "Rewrites in the same lane that edit
> an existing marker are not aliases: the block-ID separator toggle (`@route:id^` →
> `@route^id`) …", add the glued `.` toggle (`@route:id.` ↔ `@route^id`). Then run
> `sase memory init`. Change nothing else.

> [!decision] glossary_capture_alias = no Make no memory edit. If no human reviewed this
> plan, record the skipped change as a `memory` task bead via `/sase_new_task`.

## Out of scope

- A typing tip that teaches `.` after a hard-typed `^`/`:` toggle (tips are proved only
  for unchanged drafts and `expand_sigil_alias`).
- Routing other type-through characters (`/`, `\`, `,`, `=`, …) through the rewrite
  lane.
- Any change to `bob capture`, `capture-parse`, paste, or stdin, which keep the literal
  `.`.

## Acceptance

- In the Mac panel, typing `@sase:` then `sase-gallery` in the New ID composer, then
  `.`, leaves `@sase^sase-gallery` with the caret after the ID. Another `.` gives
  `@sase:sase-gallery`, and Undo restores the literal `.`.
- Typing the same keys straight into the editor (picker dismissed) behaves the same.
- `@errands.`, `Buy milk .`, `@home .`, and `- ^.` aliases are unchanged.
- bob-cli `just check` and `just docs-check` pass. The Mac repo passes
  `just portable-check`, `just docs-check`, `Scripts/macos-ci-check`, and the exact-SHA
  macOS CI run.
