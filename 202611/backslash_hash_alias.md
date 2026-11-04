---
tier: tale
title: Backslash types hash where hash opens a capture menu
goal:
  In Mac Capture, typing a backslash where a real hash would open a Bob completion menu
  becomes that hash (never at a line start), and a second backslash right away leaves
  one literal backslash and closes the menu.
size: medium
proposed_by: bbugyi200.athena.185.f0.f2
proposed_with: claude/opus
create_time: 2026-11-04 17:35:59
status: wip
---

# `\` types `#` where `#` opens a menu

## Goal

In Bob Mac Capture, the `\` key should work as an unshifted `#` wherever a real `#`
would open a Bob completion menu, the same way `.` and `;` already stand in for `^` and
`:` (bob rule `expand_sigil_alias`, Mac Capture rewrite lane):

```text
Buy milk \                  →  Buy milk #                   (task-type menu)
Buy milk @cash\             →  Buy milk @cash#              (section menu)
Buy milk @cash+goog-exit\   →  Buy milk @cash+goog-exit#    (drawer menu)
@cash^goog-exit@ ⏎ - \      →  @cash^goog-exit@ ⏎ - #       (drawer-bullet menu)
Buy milk @cash:goog-exit\   →  Buy milk @cash:goog-exit#    (Pomodoro-name menu)
```

Two rules from the request:

1. **Line starts are excluded.** A `\` with only whitespace before it on its physical
   line never becomes `#`. In a blank draft, `\` keeps opening Go To as `/` (unchanged
   Mac behavior). At the start of any later line, `\` stays a literal `\`. A bullet
   start (`- \`) is not a line start, because the bullet marker comes first.
2. **A second `\` right away types one literal `\`.** Typing `\` `\` on `Buy milk `
   gives `Buy milk \`, and the type or drawer menu the alias opened closes. This mirrors
   the Go To double-tap (`fde8211` in bob-mac-capture).

As with the other aliases, Command-Z right after the alias restores the typed `\`.
`bob capture` and `bob capture-parse` keep a literal `\`; this is an editor assist, not
a grammar change.

## Current state (verified against today's bob)

- **Go To `\`** lives only in Mac Capture. `CaptureKeyCommandRouter` sends a typed `\`
  on a blank draft to `.openGotoFromBackslash`, which inserts `/` and sets the one-shot
  `backslashLiteralArmed`. A second `\` on the bare `/` draft gives
  `.typeLiteralBackslash`. On every other draft, `\` is typed natively and never reaches
  bob's rewrite lane.
- **The `.`/`;` aliases** run through `noteSeparatorToggleIntentIfEligible`
  (`CapturePanelController.swift`) and `noteSeparatorToggleIntent`
  (`CapturePanelModel.swift`), which accept only `:`, `^`, `+`, `|`, `.`, and `;`. The
  model applies an `expand_sigil_alias` reply through `applyNativeSingleEditRewrite` /
  `applyNativeRewrite` (one native, undoable NSTextView edit) and then requests
  completion.
- **bob side.** `expand_sigil_alias` in `src/native/capture_language/rewrite.rs` handles
  only `.` and `;`, so `capture-rewrite` leaves `\` unchanged.
- **Where `#` opens a menu.** Probed with the current debug build and `capture-complete`
  on a scratch vault holding `cash.md`. The caret sits right after the `#`, and
  "replacement" is the field's byte range:

| Draft with `#` at the caret                          | Context               | Replacement                |
| ---------------------------------------------------- | --------------------- | -------------------------- |
| `Buy milk #`, `foo #a #`, nested `  - #`             | `task_type`           | covers the `#`             |
| `Buy milk @cash+goog-exit#`, `@cash+goog-exit!#`     | `drawer`              | covers the `#`             |
| `@cash^goog-exit@\n- #`, `Buy milk\n- #`             | `drawer` (bullet)     | covers the `#`             |
| `@cash#`, `a > @cash#`                               | `section`             | empty, starts at the caret |
| `@cash:goog-exit#`, `@cash^id@#`, `^cash:goog-exit#` | `pomodoro_name`       | empty, starts at the caret |
| `=#`, `=25#`                                         | `pomodoro_start_name` | empty, starts at the caret |
| `x=25#`                                              | `pomodoro_past_name`  | empty, starts at the caret |
| `=/#`, `=/fix#a#`                                    | `pomodoro_split_type` | empty, starts at the caret |
| `=*fix#`                                             | `pomodoro_merge_dest` | empty, starts at the caret |
| `[[cash#`, `[[#`                                     | `wikilink_heading`    | empty, starts at the caret |

These stay non-fields: glued `dir#`/`C#`, mid-line `#`, `@@cash#`, Work Log text
(`x foo #`), relation targets (`~note:id#`), and an escaped `\#` (`foo \#`, `- \#`,
`@cash\#`). The last case means a `\` typed right after another `\` can never alias.

- **No `#` context opens a filter-field picker.** `CapturePickerSource` has no `#`
  source; these are inline completion menus. So the second `\` arrives in the editor
  while the menu is visible, through the router's main path rather than `pickerCommand`.
- **Wikilink fields come from a separate scanner.** `capture_links::completion_field_at`
  (`src/native/capture_links.rs`) produces `wikilink_heading`, and
  `capture_complete::engine::build_result_inner` checks it before the capture-marker
  grammar. `capture_language` already calls into `capture_links` (for example
  `is_protected_markup_range`).

## Part 1: bob-cli

### 1a. Accept `\` → `#` in `expand_sigil_alias`

In `src/native/capture_language/rewrite.rs`, add a `\` branch. It is cleanest as a
helper (for example `expand_hash_alias(raw_text, start, cursor)`) called from
`expand_sigil_alias` when the typed byte is `\`, before the existing `.`/`;` match, so
that code stays untouched. The helper:

1. **Line-start exception.** Find the physical line start
   (`raw_text[..start].rfind('\n')` + 1, or 0). If `raw_text[line_start..start]` is only
   spaces/tabs (empty included), return `None`.
2. **Token rule (same as today).** Some `tokenize_with_spans` token must end exactly at
   `cursor`, so a `\` with text glued after it (`foo \alpha` with the caret after `\`)
   never aliases.
3. **Candidate.** Replace the `\` byte at `start..cursor` with `#`.
4. **Wikilinks first.** If
   `crate::native::capture_links::completion_field_at(&candidate, cursor, String::new())`
   returns a field, allow only `WikilinkHeading` with `replacement.0 == cursor`. Do not
   fall through to the marker grammar; this matches the engine's precedence.
5. **Marker grammar.** Otherwise call
   `super::completion::completion_field_at(&candidate, cursor)`. Allow it when:
   - the context is `TaskType` or `Drawer` and `replacement == (start, cursor)` (the
     bare `#` is the whole field: edge type markers, `@route+id#`, `- #` drawer
     bullets); or
   - the context is `Section`, `PomodoroName`, `PomodoroStartName`, `PomodoroPastName`,
     `PomodoroSplitType`, or `PomodoroMergeDest` and `replacement.0 == cursor` (the `#`
     sits just before the empty component).
6. **Edit.** Reuse `finish_sigil_alias(raw_text, &candidate, start, cursor, '#', "\\")`:
   one-byte edit, caret unchanged, summary `Typed # for \`, rule `expand_sigil_alias`.

Templates are already excluded at the top of `rewrite_draft`. The rule never reads the
vault, so an unresolved route still aliases where the lexical field exists, just like
`.` does. Feeding the result back in is a no-op, because the byte before the caret is
now `#`.

### 1b. Tests

`src/native/capture_language/tests/rewrite.rs`, a new
`rewrite_draft_expands_backslash_hash_alias`. Mirror
`rewrite_draft_expands_sigil_aliases_in_route_picker_positions` (rule, text, cursor,
summary, edits, idempotence). The caret is at the end unless noted:

- **Becomes `#`:**
  - `Buy milk \`, `foo #a \`
  - `@cash\`, `Buy milk @cash\`, `a > @cash\`
  - `Buy milk @cash:goog-exit\`, `Buy milk @cash^newid@\`, `^cash:goog-exit\`
  - `Buy milk @cash+goog-exit\`, and `@cash+goog-exit\ Buy milk` with the caret right
    after the `\`
  - `@cash^goog-exit@\n- \`, `@cash:goog-exit@\n- \`, `Buy milk\n- \`,
    `@cash^goog-exit@\n  - \`
  - `=\`, `=25\`, `x=25\`, `=/\`, `=*fix\`
  - `[[cash\`, `[[\`
  - `café \` (check that `edits[0].start` is a char boundary)
- **Stays `\`** (a separate negative test):
  - line starts: `\`, `  \`, `Buy milk\n\`, `Buy milk\n\n\` (a new item's first line),
    `\ foo` with the caret at 1
  - glued: `dir\`, `C:\`, `C:\Users\`
  - mid-token and mid-line: `foo \alpha` (caret after `\`), `foo \ bar` (caret after
    `\`)
  - after another `\`: `foo \\`, `@cash\\`, `- \\`, `Buy milk\n- \\`
  - Work Log text: `x foo \`, `x\n- 1 \`
  - not a `#` menu: `@@cash\`, `foo |\`, `foo ~cash:goog-exit\`, `:\`,
    `Buy milk @cash:goog-exit#a\`
  - no cursor (`rewrite_draft(.., None)`), and a template draft ending in `\`

If a probe here disagrees with the table above, trust the completion grammar and fix the
test, not the allow-list, unless the context really is not a `#` menu.

`tests/cli/capture/rewrite.rs`: add one end-to-end JSON case,
`capture-rewrite -c 10 -f json -- 'Buy milk \'`. Expect `rule: expand_sigil_alias`,
`text: "Buy milk #"`, `cursor: 10`, one edit `{start: 9, end: 10, replacement: "#"}`,
and `summary: "Typed # for \"`. Add one line-start negative (`Buy milk\n\` →
`changed: false`).

### 1c. Help and docs

- `src/native/capture_rewrite.rs` `long_about`, "Sigil alias" paragraph: add that `\`
  becomes `#` wherever bob's completion grammar would open a `#` menu. Name the menus:
  task type, section, drawer and drawer bullet, Pomodoro name, named-start and
  past-session name, split type, merge destination, and wikilink heading. Add that a `\`
  at a line start, glued to a word, or right after another `\` stays literal. Keep "four
  rules". Add one `after_help` example after the existing alias examples:
  `bob capture-rewrite -c 10 -f json -- 'Buy milk \'` (escape it correctly in the Rust
  literal; the offset must equal the draft's byte length).
- `docs/capture.md`, `## bob capture-rewrite` "**Sigil alias.**" paragraph: the same
  facts, with `Buy milk \` → `Buy milk #` and `@cash+goog-exit\` → `@cash+goog-exit#` as
  examples. Point out that a `\` after `\` never aliases, so `\#` escapes stay typeable
  directly (Mac Capture's double-tap below gives the same text).
- `docs/reference/capture-syntax.md`, "Bare `#` {#hash-picker}": one short sentence in
  the style of the existing alias notes. In Mac Capture, `\` typed where `#` would open
  a menu becomes `#` (not at a line start), and `\` `\` right away leaves one literal
  `\`, so type `\` `\` `#` to write the `\#` escape there.
- Read the `docs.md` reference memory first. Run `just docs-gen` (regenerates
  `docs/reference/cli/capture-rewrite.md`), then `just docs-check`. Do not edit the
  mounted `docs/reference/mac-capture/*` copies.
- Commit as `feat(capture): ...` with no `!`: the change adds an editor-only trigger to
  an existing rule. Do not touch the `Cargo.toml` version or `CHANGELOG.md`
  (release-train decision).

## Part 2: bob-mac-capture

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
first. Bob still authors every decision about where `\` aliases. The app only forwards
the key and reverts bob's own edit on a double-tap, so it never parses grammar (decision
`mac-capture-is-a-thin-client`).

### 2a. Forward `\` through the rewrite lane

- `CapturePanelController.noteSeparatorToggleIntentIfEligible`: also accept `\`. Match
  it by characters as `CaptureKeyCommandRouter.isTypedBackslash` does: Option/Shift
  allowed for layouts that need them, Command/Control veto. Keep today's
  Command/Control/Option veto for the other keys.
- `CapturePanelModel.noteSeparatorToggleIntent`: accept `typed == "\\"`.
- Nothing else changes for the first `\`. The blank-draft and bare-`/` Go To branches
  still claim `\` before the editor sees it, and an `expand_sigil_alias` reply is
  already applied natively with completion requested, so the `#` menu opens through the
  existing handlers.

### 2b. Double-tap: the hash-alias literal arm

- **State.** Add a one-shot arm next to `backslashLiteralArmed`, for example
  `hashAliasLiteralArm: (draft: String, hashUTF8: Int, caretUTF8: Int)?`. Set it in
  `applyNativeRewrite` only after the native apply succeeds,
  `response.rule == "expand_sigil_alias"`, and Bob's covering edit replaced one byte
  with `#`. Use bob's edit and cursor, never a scan of the draft. `.`/`;` aliases never
  arm it.
- **Take and clear.** Take it on every key-down in `routeKeyDown`, the way
  `takeBackslashLiteralArm()` is taken. Clear it wherever `clearBackslashLiteralArm()`
  runs (panel dismissal/reset). The router context gets `hashAliasLiteralArmed: Bool`,
  true only when the taken arm's `draft` equals `plainDraft` and the collapsed caret
  equals `caretUTF8`, so a stale arm is inert.
- **Router.** In the main-path `isTypedBackslash` block of
  `CaptureKeyCommandRouter.command(for:context:)` (no marked text), when
  `hashAliasLiteralArmed`:
  - a non-repeat `\` returns a new `.typeLiteralBackslashForHashAlias`;
  - an auto-repeat `\` returns `.consumeKey`, and the controller puts the arm back, so
    holding the key never stacks `\` after the `#`.

  Command-`\` (today's Pomodoro), modal surfaces, and the Go To branches keep their
  current precedence and behavior.

- **Action.** The controller handles the command through a model method such as
  `replaceHashAliasWithLiteralBackslash()`:
  1. Guard: the arm still matches, the byte at `hashUTF8` is `#`, no marked text, not
     submitting, input not locked. Otherwise return `false` so the key falls through.
  2. Replace that `#` with `\` through the same NSTextView path as `applyNativeRewrite`
     (`applyNativeRewriteEdit` with a `CaptureNativeRewriteEdit`), so Undo stays native:
     Command-Z returns to `#`, and again to the typed `\`.
  3. Leave the caret just past the `\`.
  4. Then `dismissCompletion()`, set `suppressedCompletionAcceptanceDraft` to the new
     draft, and refresh analysis with `requestCompletion: false`. No menu or picker
     reopens until the next edit or caret move. This matters because some literal drafts
     are fields again: `@cash+goog-exit\` is `task`, `[[cash\` is `wikilink_note`.
- A third `\` is ordinary typing: it goes through the lane, bob sees `\` after `\` and
  leaves it, so the draft becomes `\\`.
- **Accepted race.** If the second `\` lands before bob answers the first (the rewrite
  is a few ms), the in-flight rewrite is dropped because the caret moved, and the draft
  keeps `\\` with no menu. Do not add keystroke buffering for this. Pin the outcome in a
  test so it stays deterministic.

### 2c. Fixtures and tests

- Generate **real bob output** from the Part 1 build into `Tests/Fixtures/`, with
  `bob capture-rewrite -c <N> -f json -- <draft>` and the offset equal to the draft's
  byte length:
  - `capture-rewrite-hash-alias-task-type.json` (`jot idea \`)
  - `capture-rewrite-hash-alias-section.json` (`jot idea @foo\`)
  - `capture-rewrite-hash-alias-drawer.json` (`Postgres 17 minimum @foo+bar\`)
  - `capture-rewrite-hash-alias-unchanged.json` (a line-start `\`, for example
    `jot idea\n\`)

  These drafts match the existing `fake-bob` `capture-complete` branches for
  `jot idea #`, `jot idea @foo#`, and `Postgres 17 minimum @foo+bar#`. Reuse them; add a
  drawer-bullet case only if an existing fixture or
  `Scripts/generate-drawer-fixtures.sh` already serves `- #`.

- `Tests/Fixtures/fake-bob`: wire the new rewrite branches beside the
  `capture-rewrite-sigil-alias-*` ones, plus an older-bob gate (for example
  `FAKE_BOB_REWRITE_HASH_UNCHANGED=1` → `changed: false`). Make sure `jot idea \\` and
  the other literal drafts get `changed: false`.
- `CaptureSeparatorToggleTests.swift` (live panel tests, next to the `.`/`;` alias
  tests):
  - `jot idea ` + `\` → `jot idea #`; the argv log shows `capture-rewrite` then
    `capture-complete` on `jot idea #`; the `task_type` menu is visible.
  - Then `\` → `jot idea \`, caret after it, completion dismissed, and no
    `capture-complete` reopens a menu for that draft.
  - The drawer case (`Postgres 17 minimum @foo+bar\`): alias, drawer menu, then the
    double-tap closes it.
  - The section case (`jot idea @foo\`) opens section completion.
  - Command-Z right after the alias restores `\` once, without retriggering.
  - An older bob (`changed: false`) leaves `\` literal with no menu, and a following `\`
    types natively (nothing armed).
  - A different key after the alias (for example `b`) clears the arm, so a later `\` is
    ordinary typing.
  - The delayed-reply race from 2b ends in `\\` with no menu.
- `CaptureBackslashGotoTests.swift` (router and model units):
  - armed + `\` → `.typeLiteralBackslashForHashAlias`
  - armed + repeat → `.consumeKey`
  - armed + marked text → `nil`
  - armed + Command-`\` → `.openTodayPomodoro`
  - unarmed non-blank `\` → `nil` (existing)
  - the arm is set only for a `#` alias edit; a stale arm (draft or caret changed) is
    inert; dismissal clears it
  - every existing Go To test still passes unchanged

### 2d. Docs (`just docs-check`)

- `docs/keyboard.md`, "Editor and completion keys" table: add a `\` row next to the `.`
  and `;` rows. In a blank draft it opens Go To (link the existing section). Elsewhere
  it asks Bob to rewrite `\` as `#` where `#` would open a menu, never at a line start.
  A second `\` right away leaves one literal `\` and closes the menu; Undo restores `\`.
  The completion-visible column: the same. The prompt columns: native. In "Going to a
  task from any app", add one sentence that the same `\` `\` double-tap works after the
  `#` alias.
- `docs/capture-panel.md`, the rewrite-lane alias paragraph: add `\` → `#`, the
  line-start exception, the client-side one-shot double-tap (cleared by any other key),
  Undo, and that an older bob leaves `\` literal.
- `docs/task-pickers.md`, "Sections and Pomodoro names": one sentence that `\` in these
  positions is turned into `#` through Bob's rewrite lane.
- `docs/protocol.md`, the `capture-rewrite` / `expand_sigil_alias` compatibility row:
  add `\` → `#` at `#`-menu positions. An older bob leaves `\` literal, and the
  double-tap then never arms.

## Verification

- **bob-cli.** Run `just lint`, `just test`, `rustfmt --check` on the changed files,
  `just docs-gen`, and `just docs-check`.
  - Earlier runs had baseline failures this work does not touch: eight Clippy errors in
    `src/native/capture_language/editor_pomodoro.rs`, `past.rs`, and
    `articles/inbound.rs`, and `np_move_groups_follow_requested_sources_in_cli_fixtures`
    (`bob-cli-h5`). If they, or other failures outside the changed files, still appear,
    confirm they also fail on unmodified `master` and report them as baseline. No new
    diagnostics or failures may appear in changed files.
- **Spot-check the binary.** No vault is needed for `capture-rewrite`:

  ```bash
  cargo run -q --bin bob -- capture-rewrite -c 10 -f json -- 'Buy milk \'
  cargo run -q --bin bob -- capture-rewrite -c 25 -f json -- 'Buy milk @cash+goog-exit\'
  printf '@cash^goog-exit@\n- \\' | cargo run -q --bin bob -- capture-rewrite -c 20 -f json
  printf 'Buy milk\n\\' | cargo run -q --bin bob -- capture-rewrite -c 10 -f json   # unchanged
  cargo run -q --bin bob -- capture-rewrite -c 11 -f json -- 'Buy milk \\'          # unchanged
  ```

- **bob-mac-capture.** Never land uncompiled Swift.
  - On athena, run `just portable-check` and `just docs-check`.
  - The macOS targets need the Mac. Read the `tailnet.md` reference memory, rsync a
    scratch copy without `.build`/`.git`, run `Scripts/bundle.sh` and
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`.
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified.
- **Manual check after bob ships, in Mac Capture:**
  - `Buy milk \` opens the type menu on `#`, and `\` again leaves `Buy milk \` with the
    menu closed;
  - `@cash+goog-exit` + body + `\` opens drawers;
  - `@cash^goog-exit@` ⏎ `- \` opens the drawer-bullet menu;
  - a blank draft `\` still opens Go To;
  - `foo` ⏎ `\` stays `\`.

## Non-goals and accepted costs

- **No grammar change.** `\` is not a new sigil (decision
  `capture-sigils-say-where-what-how`). `bob capture` and `capture-parse` are unchanged.
- **Escape habit.** Where the alias fires, a `\#`, `\%`, or `\>` escape now needs `\`
  `\` first (`\` `\` `#` → `\#`), exactly like the Go To blank draft. A `#` typed right
  after the alias just gives `##`. Do not add escape-aware follow-up keys; that would
  need grammar knowledge in the app.
- **No other `\` positions.** A line-start `\` never becomes `#`. A later line's leading
  `#type` needs a real `#`. Go To stays blank-draft-only.
- **No other clients.** No zsh, Obsidian plugin, or bob-plugins changes; nothing in
  bob-plugins calls `capture-rewrite`.
- **No memory changes.** No memory note describes the sigil alias positions, so none
  becomes false.
