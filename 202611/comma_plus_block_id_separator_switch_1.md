---
tier: tale
title: Glued , switches a block-ID marker to +
goal:
  Typing `,` against a complete `@route:id` or `@route^id` switches it to `@route+id`,
  the same way a typed `+` does, in the editor and in the New ID composer.
size: small
decisions:
  glossary_capture_alias:
    ask:
      Should the Capture Alias glossary list the glued `,` switch beside the glued `.`
      toggle?
    memory:
      - glossary:capture-alias
    default: false
    answer: true
proposed_by: bbugyi200.athena.1am.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 07:16:46
status: wip
---

# Glued `,` switches a block-ID marker to `+` (`@route:id,` → `@route+id`)

## Goal

`,` is the easy-to-reach alias key for `+`. The previous change made a glued `.` (the
`^` alias) toggle `@route:id` ↔ `@route^id`. This change makes a glued `,` act exactly
like a typed `+` when it lands right against a complete block-ID marker:

- `Do work @foo:bar,` → `Do work @foo+bar`
- `Do work @foo^bar,` → `Do work @foo+bar`
- `Create sase-gallery! @sase:sase-gallery,` → `Create sase-gallery! @sase+sase-gallery`

Today bob answers `changed: false` and the `,` stays literal. A glued `,` is never valid
in a block ID: `bob capture-parse` reports `invalid_pomodoro_block_id` for
`Talk to @work:a,`. So the switch takes nothing away, and Undo in Bob Mac Capture still
restores the literal `,`.

## Findings that shape the design

1. **`+` already does this in every context.** `switch_block_id_separator` in
   `src/native/capture_language/rewrite.rs` consumes a `+` glued to a complete plain
   `@route:id` / `@route^id` and switches it to `@route+id`. It reports rule
   `switch_block_id_separator`, summary `Changed @route:id to @route+id`, and no `alias`
   stamp. The `+` path covers all of these, verified against current master:
   - solo, leading, and trailing markers
   - list items: `- [ ] Task @work:a+` → `- [ ] Task @work+a`
   - child lines: `Parent\n- detail @work:a+` → `Parent\n- detail @work+a`
   - batch items and CRLF line endings
   - `>` transfer slots: `@inbox:bench+ > @db` → `@inbox+bench > @db`, and
     `@a+x > @c:tid+` → `@a+x > @c+tid`
   - `<` merge-in sources: `New task < @work:a+` → `New task < @work+a`
   - target-run members: `@work+a @dev:b+` → `@work+a @dev+b`, and
     `note @work:a+ @dev+b` → `note @work+a @dev+b`

   `+` stays literal after a sealed `@route:id@` / `@route^id@` and after `@route+id`
   (same separator).

2. **`,` already has neighbouring meanings, and all of them stay.** These run in other
   rules that a `@route:id,` / `@route^id,` token never reaches today:
   - `@cash,` → `@cash+`: Parent Task alias in `expand_parent_task_comma_alias`
   - `@@cash,` → `@cash+`: `collapse_global_destination`
   - `@cash+goog-exit,` → `@cash+goog-exit +`: next-parent continuation
   - `New task < @work+a,` → `New task < @work+a +`: next `<` source

   The separator switch declines `+` after `+id`, so `@route+id,` still falls through to
   the continuation.

3. **The behavior change hits two existing assertions.**
   `rewrite_draft_keeps_comma_literal_outside_parent_task_picker_positions` in
   `src/native/capture_language/tests/rewrite.rs` lists `"@cash:goog-exit,"` and
   `"@cash^goog-exit,"` as literal. Those two entries must move into the new switch
   tests. No other bob-cli or Mac test pins a comma glued to a `:`/`^` marker.
4. **Bob Mac Capture's editor path needs no code change.** The controller's key filter
   (`CapturePanelController.swift`) and `CapturePanelModel.noteSeparatorToggleIntent`
   already forward a typed `,` through `bob capture-rewrite --cursor`. The reply handler
   applies any `switch_block_id_separator` reply as one native, undoable edit.
5. **The New ID composer needs one Mac change.** Typing `@sase:` opens the Block ID
   picker's New ID composer, and the ID is typed into its filter field. A `,` there is
   outside Bob's ID charset, so `updatePickerFilter` → `commitBlockIDTypeThrough`
   commits the ID and inserts the `,` into the editor. Then
   `blockIDSeparatorTypeThroughRewrite` in
   `Sources/BobMacCapture/CapturePanelModel.swift` sends only `.`, `^`, `:`, `+`, and
   `;` to bob. `,` must join that list, or `@sase:sase-gallery,` stays literal in the
   composer flow.

## Scope decisions made here (not reviewer choices)

- **`,` only switches toward `+`.** `@route+id,` keeps opening the next parent pick
  (`@route+id +`), so there is no reverse toggle. To leave `+id`, type `:` or `^` as
  today.
- **Sealed markers stay literal.** `@route:id@,` and `@route^id@,` keep their `,`, just
  as `+` after a seal does.
- **Task-bullet IDs are untouched.** `- Task :id,` stays literal because bullet task IDs
  have no `+` form. `switch_task_bullet_id_separator` is not changed.
- **`,` reports the same rule as `+`.** That is `switch_block_id_separator` with
  `alias: None`, following the `.` toggle and the relation `=` flip precedent, so
  `sigil_alias_tip` and the typing tips do not change. The CLI help still says "five
  rules".
- **Mac:** only the New ID composer type-through list grows. The Link mode filter and
  the other pickers are unchanged.

## Part A — bob-cli (this repo)

### A1. Rewrite rule (`src/native/capture_language/rewrite.rs`)

- In `switch_block_id_separator`'s `trigger_char` match, map `b','` to `'+'`. Everything
  downstream keys off `trigger_char`, so a `,` then takes the `+` path:
  - the `eligible` table: `+` only after unsealed `:`/`^`
  - the `switch_transfer_block_id_separator` branch, which covers `>` transfer slots and
    `<` merge-in sources
  - the skipped `merge_in_source_token_after_removing_trigger` check
  - `candidate_selects_plain_marker(..., trigger_char, ...)`, which must receive `'+'`,
    never `','`

  The edit still covers `token.start..token.end`, which includes the `,`, so the trigger
  is consumed.

- Leave the `if trigger == b'.'` transfer-claim guard `.`-only. A `,` must reach the
  transfer and merge-in `+` handling exactly like `+`.
- Do not touch `switch_task_bullet_id_separator`, `collapse_global_destination`, or
  `expand_parent_task_comma_alias`.
- Update the doc comments:
  - `rewrite_draft`: the supported triggers become `^`, `:`, `+`, `.`, or `,`, and a
    glued `,` acts as `+`.
  - `switch_block_id_separator`: name the `,` trigger and say that after `@route+id` it
    falls through to the next-parent continuation.
  - `expand_sigil_alias` / `expand_parent_task_comma_alias`: note that a `,` glued to a
    complete `@route:id` / `@route^id` is claimed earlier by the separator switch.

### A2. Unit tests (`src/native/capture_language/tests/rewrite.rs`)

Use the existing `assert_switch` helper, which also checks idempotence, and
`rewrite_draft_dot_toggles_block_id_separators` as the model.

- New `rewrite_draft_comma_switches_block_id_separators_to_plus`:
  - `Do work @file:id,` → `Do work @file+id`
  - `Do work @file^id,` → `Do work @file+id`
  - the request's shape: `Create sase-gallery! @sase:sase-gallery,` →
    `Create sase-gallery! @sase+sase-gallery`
  - solo `@cash:goog-exit,` → `@cash+goog-exit` and `@cash^goog-exit,` →
    `@cash+goog-exit`, the two entries moved out of the literal list
  - list item `- [ ] Task @work:a,`, child line `Parent\n- detail @work:a,`, one item in
    a blank-line batch, a CRLF neighbour, and a Unicode prefix (`café 🚀 `)
- New `rewrite_draft_comma_matches_plus_in_transfer_merge_and_run_slots`: for each
  context below, build the `+` draft and the `,` draft with the cursor right after the
  trigger. Assert equal `rule`, `text`, `cursor`, `summary`, and `edits`, and pin the
  concrete text for each:
  - `@inbox:bench, > @db` (cursor 13) → `@inbox+bench > @db`
  - `@a+x > @c:tid,` → `@a+x > @c+tid`
  - `Create task < @cash:line,` → `Create task < @cash+line`
  - `@work+a @dev:b,` → `@work+a @dev+b`
  - `note @work:a, @dev+b` (cursor after the `,`) → `note @work+a @dev+b`
- Chaining: after `@cash:goog-exit,` switches to `@cash+goog-exit`, typing `,` again
  (`@cash+goog-exit,`) still gives `@cash+goog-exit +` with rule `ExpandSigilAlias` and
  alias id `NextParent`. That is the existing continuation, unchanged.
- Exclusions. For each, assert `rule != Some(SwitchBlockIdSeparator)` and, unless noted,
  unchanged text:
  - `@file:id@,` and `@file^id@,`
  - `@file:id#name,`, `@file:id=3,`, `@@file:id,`, `@!file:id,`, `^file:id,`,
    `&note:id,`
  - `@a+x > @c+tid,`
  - task bullets `@dev\n- Task :task-id,` and `@cash^p@\n- Task ^task-id,`
  - a draft with no cursor
  - a template draft
  - spaced `Do work @file:id ,`: assert only that the rule is not
    `SwitchBlockIdSeparator`, because the Parent Task alias may legitimately fire there.
- Keep every other entry of
  `rewrite_draft_keeps_comma_literal_outside_parent_task_picker_positions` and all
  existing `,` alias tests passing unchanged.
- The switch rewrite carries `alias: None`, and the switched draft (`Do work @file+id`,
  cursor at end) proves no `sigil_alias_tip`.

### A3. CLI tests (`tests/cli/capture/rewrite.rs`)

- Add `capture_rewrite_json_comma_switches_a_block_id_separator_to_plus`, modelled on
  `capture_rewrite_json_dot_switches_a_block_id_separator`. Running
  `bob capture-rewrite -c <len> -f json -- 'Create sase-gallery! @sase:sase-gallery,'`
  must report:
  - `changed: true` and `rule: "switch_block_id_separator"`
  - text `Create sase-gallery! @sase+sase-gallery`, with cursor at that text's length
  - one edit from the marker start to the input length, replaced with
    `@sase+sase-gallery`
  - summary `Changed @sase:sase-gallery to @sase+sase-gallery`
  - no `alias` and no `sigil_alias_tip`
- Extend `capture_parse_does_not_apply_the_separator_toggle`, or add a sibling test, to
  show that `capture-parse` keeps `Do work @file:id,` literal.
- `capture_rewrite_json_expands_glued_target_comma_with_one_edit` (`@cash+goog-exit,` →
  `@cash+goog-exit +`) must pass unchanged.

### A4. CLI help (`src/native/capture_rewrite.rs`)

In the `long_about` "Block-ID separator toggle" paragraph, after the glued-`.`
sentences, add one sentence:

> A glued ',' after '@route:id' or '@route^id' switches to '@route+id' exactly like '+',
> including transfer and '<' source slots; after '@route+id' it still inserts ' +' for
> the next parent pick, and after a seal it stays literal.

Keep "five rules". Run `just docs-gen` to regenerate
`docs/reference/cli/capture-rewrite.md`. Never hand-edit that file.

### A5. Docs (`docs/capture.md`, `bob capture-rewrite` section)

- **Block-ID separator toggle:** after the glued-`.` example
  (`Create sase-gallery! @sase:sase-gallery.`), add the `,` behavior: a glued `,` after
  `@route:id` or `@route^id` gives `@route+id`, as `+` does, including `>` transfer and
  `<` source slots. Give the example `Create sase-gallery! @sase:sase-gallery,` →
  `Create sase-gallery! @sase+sase-gallery`. Say that after `@route+id` the `,` keeps
  inserting ` +`, and that after a seal it stays literal. In the Undo sentence
  ("…`@route^id+`, or glued-dot rewrite…"), add glued-comma.
- **Sigil alias, the `,` paragraph:** add one clause saying that a `,` glued to a
  complete `@route:id` / `@route^id` switches the separator instead (see above).

### A6. Memory (only if `glossary_capture_alias` is yes)

> [!decision] glossary_capture_alias Using `/sase_memory_write`, edit the
> `glossary:capture-alias` strand. In the sentence "Rewrites in the same lane that edit
> an existing marker are not aliases: the block-ID separator toggle (`@route:id^` →
> `@route^id`, or a glued `.` toggle (`@route:id.` ↔ `@route^id`))…", add the glued `,`
> switch (`@route:id,` → `@route+id`). Change nothing else in the note. Then run
> `sase memory init` and `just docs-gen`, which regenerates
> `docs/reference/glossary.md`.

> [!decision] glossary_capture_alias = no Make no memory edit. If no human reviewed this
> plan, record the skipped glossary update as a `memory` task bead through
> `/sase_new_task`, as `/sase_memory_write` directs.

## Part B — Bob Mac Capture (linked repo `bob-mac-capture`)

Open it with `/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`) and work only in
the path that command prints.

### B1. Code (`Sources/BobMacCapture/CapturePanelModel.swift`)

- In `blockIDSeparatorTypeThroughRewrite`, add `","` to the forwarded keys:
  `[".", "^", ":", "+", ";", ","]`. Update its doc comment and the doc comment above
  `commitBlockIDTypeThrough`, which says "a single separator key also passes through
  Bob's rewrite lane", if it lists the keys. With an older bob the `,` simply stays
  literal.

### B2. Fixture (`Tests/Fixtures/fake-bob`)

Add `capture-rewrite` cases next to the existing `Follow up @file:new-id.` cases, in the
same `printf`/heredoc shape:

- `Follow up @file:new-id,` → text `Follow up @file+new-id`, cursor 22, edit
  `{10..23 → "@file+new-id"}`, summary `Changed @file:new-id to @file+new-id`
- `Follow up @file^new-id,` → the same text, cursor, and edit, with summary
  `Changed @file^new-id to @file+new-id`
- `Do work @file:id,` → text `Do work @file+id`, cursor 16, edit `{8..17 → "@file+id"}`,
  summary `Changed @file:id to @file+id`

### B3. Swift tests

Use only helpers already defined in the test class you edit. Each class has its own
`private func waitUntil`. Last time the macOS typecheck failed because a new test called
a helper its class did not define.

- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`: add
  `testBlockIDNewIDCommaTypeThroughSwitchesColonToPlus` and
  `testBlockIDNewIDCommaTypeThroughSwitchesCaretToPlus`, modelled on
  `testBlockIDNewIDDottedTypeThroughTogglesColonToCaret`. Each opens the composer with
  `@file:` or `@file^`, calls `updatePickerFilter("new-id")` and then
  `updatePickerFilter("new-id,")`, and expects `Follow up @file+new-id`, the summary
  status, caret 22, and the picker hidden.
- `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`: add
  `testCommaKeySwitchesColonMarkerThroughNativeInsert`, modelled on
  `testPlusKeySwitchesColonMarkerThroughNativeInsert`. It uses
  `typeSeparator(",", ...)`, expects `Do work @file+id` with caret 16, and checks that
  the record contains
  `argv=capture-rewrite --cursor 17 --format json -- Do work @file:id,`.

### B4. Docs

- `docs/task-pickers.md`, the New ID composer type-through paragraph: list `,` among the
  keys that go through Bob's rewrite lane, and add the example `@sase:sase-gallery,` →
  `@sase+sase-gallery`.
- `docs/capture-panel.md`, the separator-toggle bullet: a glued `,` after a complete
  `@route:id` / `@route^id` switches to `@route+id`, like `+`; after `@route+block-id`
  it keeps opening the next parent pick. In the Undo sentence, add `,` beside `.`.
- `docs/keyboard.md`, the `,` row: add "glued to a complete `@route:id` / `@route^id`,
  switches it to `@route+id`".
- `docs/protocol.md`: the capability line becomes "`switch_block_id_separator` for `^`,
  `:`, `+`, a glued `.`, and a glued `,`". In the compatibility row, change "Older Bob
  leaves `.` literal" to "Older Bob leaves `.`/`,` literal".

## Verification

bob-cli:

- `cargo test capture_language::tests::rewrite` and
  `cargo test --test cli capture_rewrite` while iterating.
- `just docs-gen`, then `just check` and `just docs-check`. All must pass.
- Spot-check the built binary: `bob capture-rewrite -c 12 -f json -- 'Do @foo:bar,'`
  gives `Do @foo+bar`, and `bob capture-rewrite -c 16 -f json -- '@cash+goog-exit,'`
  still gives `@cash+goog-exit +`.

Bob Mac Capture:

- `just portable-check`, which runs swift-format lint and the portable targets.
- `just docs-check`. Set `BOB_CLI_DIR` to this workspace's bob-cli checkout so the
  mounted docs build against the edited bob-cli.
- The `BobMacCapture*` targets only type-check on macOS. Gate them with
  `scripts/dev/macos-ci-check` from the Mac checkout, launched through `/sase_monitor`
  because it can take over an hour. Report the macOS step verdicts, and fix any failure
  this change introduced.

Do not edit `Cargo.toml`'s version or `CHANGELOG.md` in either repo (release train).
Commit messages: `feat(capture): switch block ID separators to plus with glued comma`
(bob-cli) and `feat(capture): forward comma separator type-through` (Mac). Neither is a
contract break, so neither takes `!`.
