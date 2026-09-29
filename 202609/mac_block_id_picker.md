---
tier: epic
title: Block ID Picker for `@file:` and `@file^` in Bob Mac Capture
goal:
  Typing `@route:` or `@route^` anywhere those markers are valid opens the same large,
  fuzzy, keyboard-first picker language as `^`. A marker-only `@route:` browses and
  links the note's tasks. Every new-ID position (`@route^`, and `@route:` on an item
  with text) becomes an ID composer with Bob-generated suggestions, live availability
  against every ID already in the note, and type-through commits. Bob stays the only
  authority for grammar, candidates, intent, used IDs, and suggestions.
phases:
  - id: bob-contract
    title:
      "bob-cli: block-ID completion contract (intent, used IDs, suggestions,
      `task_block_id`)"
    depends_on: []
    size: medium
    description:
      "bob-contract: extend `bob capture-complete` with the `task_block_id` context, the
      additive `block_id` object (intent, marker range, body, allowed-character rule,
      used IDs, suggestions), link-only candidates with Pomodoro annotations, and the
      project-note `+` replacement fix; update docs and tests."
  - id: picker-generalize
    title:
      "Mac: generalize the Active Task Picker into a source-agnostic capture picker"
    depends_on: []
    size: medium
    description:
      "picker-generalize: refactor the `^` picker's presentation types, model state
      machine, routing, controller focus repair, filter field, and card into
      source-agnostic `CapturePicker*` building blocks with zero user-visible change to
      `^`."
  - id: block-id-core
    title:
      "Mac CaptureCore: decode the block-ID contract and build the Block ID Picker
      engine"
    depends_on:
      - bob-contract
      - picker-generalize
    size: medium
    description:
      "block-id-core: decode the `block_id` object and `task_block_id` context, add
      Bob-driven ID rules, and build the link and new-ID presentation engine on the
      generic picker types, with real-bob fixtures and exhaustive tests."
  - id: block-id-flow
    title: "Mac app: Block ID Picker flow, type-through, quiet states, and routing"
    depends_on:
      - block-id-core
    size: medium
    description:
      "block-id-flow: route `pomodoro_block_id`/`task_block_id` completions into the
      generic picker with intent-aware opening rules, accept, type-through commits,
      trigger removal, chip, quiet incomplete states, fake-bob fixtures, model and
      router tests, and README behavior docs."
  - id: block-id-design
    title:
      "Mac app: Block ID Picker visuals, sizing, accessibility, docs, and macOS
      verification"
    depends_on:
      - block-id-flow
    size: medium
    description:
      "block-id-design: finish the link and new-ID visuals (scope token, availability
      badge, section headers, row kinds, detail strip, key hints, chip, marker
      highlight), sizing, accessibility, rendered-image review of both pickers, README
      visuals, and a real-panel smoke test."
proposed_by: bbugyi200.apollo.31
create_time: 2026-09-29 09:43:05
status: wip
---

- **PROMPT:**
  [prompts/202609/mac_block_id_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/mac_block_id_picker.md)

# Plan: Block ID Picker for `@file:` and `@file^` in Bob Mac Capture

## Context

Bob Mac Capture (the linked `bob-mac-capture` repo) is the macOS menu-bar front end for
`bob capture`. The Active Task Picker for a leading `^` landed on 2026-09-29 (bob-cli
epic `bob-cli-2g`; linked-repo commits `309431b`, `039a322`, `0a6cd0e`, `866e165`,
`4802d23`). It is a modal mode of the capture panel: Bob supplies one snapshot, the app
fuzzy-filters it locally, and the card shows a filter bar, grouped rows, a detail strip,
and key hints. The user loves it and wants the same experience for `@file:` and `@file^`
wherever those markers are valid: leading or trailing on an item's first line, trailing
on authored child lines, and in any item of a blank-line-separated batch draft.

What those markers mean (from `docs/capture.md`):

- `@route:block-id` **with item text** creates a new Next task with that authored ID
  plus a Pomodoro Task Link. The ID must be new. `#name`, `=<X>`, `=x`, and a
  project-note `+` may follow.
- `@route:block-id` **as the whole item** (no text, no children, no `s:`/`p:`/`%`) links
  an existing `^block-id` task into today's ledger (`pomodoro_link` mode). `#name`,
  `=<X>`, and `=x` may follow.
- `@route^block-id` always creates an ordinary task with a new authored ID.
  `@route^block-id+` instead creates the project note `<route>_<id>.md`.

Today's behavior and its problems (measured on the user's vault on 2026-09-29 with bob
built from this repo; the installed `~/.cargo/bin/bob` on the planning host predates
solo `@route:id` linking, so build from source before probing):

1. `@route:` uses the generic inline list: a 430pt, five-row popover of Bob's
   `pomodoro_block_id` candidates with prefix/substring matching on the block ID only.
   `sase.md` has 96 candidates, so the list is cramped and not fuzzy.
2. **The inline list is a trap on items with text.**
   `bob capture-complete -c 17 -- 'Write docs @sase:'` offers all 96 existing IDs.
   Accepting any of them fails:
   `block ID ^tool already exists in /home/bryan/bob/sase.md`.
3. `@route^` offers nothing (`context: null`). A collision only shows up afterwards as a
   red live-preview error.
4. While you are still typing, `@sase:` and `@sase^` show red dry-run errors: "Pomodoro
   capture block ID must be non-empty and contain only A-Z, a-z, 0-9, '_' or '-'" and
   "task block-ID capture block ID must be non-empty and contain only A-Z, a-z, 0-9 or
   '-'". This is the same ugliness the `^` picker fixed with its quiet incomplete state.
5. **Bob bug:** for `@sase:x+` with the cursor inside the ID,
   `bob capture-complete -c 7` returns `replacement {6, 8}`. That range includes the
   project-note `+`, so accepting a candidate erases the sigil.
6. Bob's duplicate check is exact and case-sensitive: `Write docs @sase^Tool` succeeds
   while `^tool` exists. The picker must mirror Bob exactly and never invent its own
   notion of "taken".

Real data profile (`bob capture-tasks`, 2026-09-29):

- `sase.md` has 1225 lines and 253 open tasks, 96 of them with block IDs.
  - By section: "Next & In Progress" 59, "Blocked" 28, "Tasks" 8, no heading 1.
  - By status: In Progress 39, Blocked `[?]` 28, Next 20, Todo 9.
  - All are depth 0.
  - 113 lines end in `^id`, so about 17 IDs belong to done tasks or non-task anchors.
- `bob.md` has 9 identified tasks and `cash.md` has 12 (all under "Blocked").
- Task text runs 16–112 characters (median about 54). It often contains `` `code` ``
  spans, `[[ref/chat/…]]` wikilinks, curly quotes, and a trailing `!`.
- The user's IDs are short, hand-authored kebab-case. They often come from a quoted or
  code phrase, or from 1–3 key words, sometimes keeping the verb: `tool` (from
  `` `sase tool` ``), `agent-data-panels` (from "agent data panels"), `card-blocks`,
  `hold` (`` `%hold` ``), `fix-v-key`, `harden-services`, `clean-core`, `release-v18`,
  `q-weight`.

Repository access:

- Mac phases: open the linked repo with `sase repo open bob-mac-capture -r "<reason>"`.
  If that fails because the linked checkout's primary workspace directory is missing (a
  known condition on Linux hosts), use
  `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`.
- Use the printed path for all reads and edits, and read its `AGENTS.md` if the open
  command names one.
- `bob-contract` works in this bob-cli repo.

Relevant code:

- bob-cli
  - `src/native/capture_language/completion.rs`: `completion_field_at`,
    `marker_field_at_cursor` (the `@route^` branch sets `right_context: None`; the
    pomodoro branch keeps a trailing `+` in the block part), and
    `completion_field_from_parts`.
  - `src/native/capture_complete.rs`: `build_result`, `task_candidates`, the
    `Candidates` enum, JSON and human output, and the help `Contexts:` list.
  - `src/native/capture_active_tasks.rs`: `read_ledger` (the Pomodoro owner of each
    `(route, block_id)` Task Link).
  - `src/native/collect_done/transform.rs`: `block_ids_in_markdown` (the exact scanner
    Bob's duplicate check uses) and `is_block_id_byte`.
  - `src/native/capture/pomodoro_adjust.rs`: `reject_duplicate_block_id`.
  - `src/native/note_tasks.rs`: `scan`, `NoteTask`, and `suggest_block_id`.
  - Tests: `src/native/capture_language/tests/completion.rs` and
    `tests/cli/capture/complete_editor.rs` / `complete_query.rs`.
  - Docs: `docs/capture.md` (§`bob capture-complete`).
- bob-mac-capture
  - `Sources/CaptureCore/ActiveTaskPickerPresentation.swift`, `FuzzyMatcher.swift`,
    `ActiveTaskDisplayText.swift`, and `CaptureModels.swift`
    (`CaptureCompletionResponse`, `CaptureCompletionCandidate`, `ActiveTaskPomodoro`).
  - `Sources/CaptureCore/CompletionRowContent.swift` (`CaptureCompletionContext`).
  - `Sources/BobMacCapture/ActiveTaskPickerState.swift` and `CapturePanelModel.swift`:
    `handleCompletionResponse`, `handleActiveTaskCompletion`, `presentActiveTaskPicker`,
    accept, cancel, chip, `removeActiveTaskTrigger`, `parseNeedsActiveTask`,
    `applyQuietIncompleteActiveTask`, `shouldRequestCompletion`, and
    `routeReplacementRange`.
  - `Sources/BobMacCapture/ActiveTaskPickerView.swift`, `ActiveTaskFilterField.swift`,
    `CaptureKeyCommandRouter.swift`, `CapturePanelController.swift`
    (`repairActiveTaskFilterFocusIfNeeded`), `CapturePanelView.swift` (editor dimming,
    `auxiliaryRegion`, `currentAuxiliaryHeight`, footer swap), and
    `CaptureEditorPalette.swift`.
  - `Tests/Fixtures/fake-bob` (existing `Follow up @file^new-id` and
    `Do work @Dev^new-id` fixtures), `Tests/CaptureCoreTests`, and
    `Tests/BobMacCaptureTests`.

## Design decisions (lead)

1. **One picker language, two intents.** The right-hand side of `@route:` and `@route^`
   is a block ID in `route.md`, and there are exactly two things a person can mean
   there:
   - **Link:** reference an existing task. This is only valid for a marker-only
     `@route:` item.
   - **New ID:** mint an ID the note does not use yet. This applies to every `@route^`,
     to `@route:` on an item with text, and to either marker followed by the
     project-note `+`.

   So `@file:` gets a Link picker that is a sibling of `^`: the note's linkable tasks,
   grouped by the note's own headings, fuzzy-filtered. `@file^` (and `@file:` with text)
   gets a **New ID** composer in the same card: suggestions derived from the task text,
   live availability against every ID already in the note, a one-key "next free"
   alternative when an ID is taken, and similar existing IDs for naming consistency.
   Offering existing tasks in a new-ID position (today's behavior) is a guaranteed Bob
   error, so the design never does it.

2. **Bob decides; the app presents.** Bob reports the intent, the candidates, every used
   ID (with what uses it), suggestions, the marker's token range, and the ID grammar as
   a one-character regex plus a human description. The app only filters locally (the `^`
   precedent), checks exact membership in Bob's used-ID list, applies Bob's character
   rule, and derives the `-2…-99` "next free" variant validated by that rule. Live
   preview after insert remains the final authority.
3. **Generalize, don't clone.** The `^` picker's state machine, routing, focus repair,
   filter field, sizing, and card become source-agnostic `CapturePicker*` building
   blocks. `^` and the Block ID Picker are two sources feeding one mechanism. Two
   parallel modal state machines with twin precedence rules are where reliability bugs
   live. The refactor ships alone first, with zero user-visible change, so regressions
   are isolated.
4. **Type-through keeps keystrokes faithful (New ID only).** In the New ID composer, the
   field only ever holds ID characters. Typing a character outside Bob's allowed set —
   space, `#`, `=`, `+`, `.`, … — at the end of the field commits the typed ID and
   continues typing that character in the editor. So `@sase^flaky-test Fix it` or
   `Fix it @sase:flaky#bugs=3` produce exactly the draft they would without the picker,
   and the next context (Pomodoro-name completion after `#`) opens naturally. The Link
   picker keeps `^`'s semantics: spaces separate fuzzy tokens.
5. **Incomplete is a state, not an error.** When `capture-parse` reports `pomodoro_id`
   or `block_id` in `needs` (top-level or any item), the app skips the doomed dry run
   and shows a calm status line, exactly like `active_task`.
6. **Show which marker you are editing.** Markers can now be anywhere in a multi-item
   draft. While any picker is open, the marker token being completed is highlighted in
   the dimmed editor, and the scope token names the marker (`@sase^`) plus `line N` on
   multi-line drafts.
7. **Compatibility.**
   - Older Bob sends `pomodoro_block_id` without a `block_id` object. That is treated as
     Link intent without the New ID row and without type-through.
   - Older Bob never sends `task_block_id`, so `@route^` simply shows no picker.
   - The app never falls back to the inline list for these two contexts.
8. **Out of scope** (record as proposed follow-ups, not in this epic):
   - The `@route+` / `@@route+` parent-task context in the same picker, including its
     Add block ID flow.
   - Adding an ID to an unidentified task from inside the Link picker.
   - Project-note filename availability.
   - Changing `^` behavior, `#name` completion, or wikilink completion.
   - Settings toggles.
   - A latent Bob inconsistency: the `:` validator accepts `_`, but `is_block_id_byte`
     and Obsidian do not treat `_` as a block-ID character. Report Bob's rule faithfully
     here; do not change validation in this epic.

### UX specification (final state after `block-id-design`)

Link picker (`@sase:` as the whole item):

```text
┌ capture panel (grows while picking) ─────────────────────────────────────────┐
│ @sase:▔▔▔▔▔▔                                     (editor dimmed, token lit)   │
│ ╭────────────────────────────────────────────────────────────────────────╮   │
│ │ [@sase: Tasks]  Filter sase.md tasks, or type a new ID…        96 tasks │   │ filter bar
│ ├────────────────────────────────────────────────────────────────────────┤   │
│ │ # Next & In Progress                                           59 tasks │   │ pinned note heading
│ │  ◐  Read and act on core_schema_skew_outage_recovery_ux  SASE  ^recovery-p…│ │
│ │  ◉  Bug bash and improve sase TUI command-mode panel           ^tui-cli │   │
│ │ # Tasks                                                         8 tasks │   │
│ │  ○  Add sase tool command to wrap common tool calls!              ^tool │   │
│ │ …                                                                      │   │
│ ├────────────────────────────────────────────────────────────────────────┤   │
│ │ ○ Add sase tool command to wrap common tool calls!                      │   │ detail strip
│ │ Todo · ▣ sase.md › Tasks · Not in a Pomodoro · 12 sub-items            │   │
│ │                                                  ↩ inserts @sase:tool │   │
│ ╰────────────────────────────────────────────────────────────────────────╯   │
│ ↑↓ Move   ↩ Insert   ⌘↩ Insert & Capture   esc Clear / Cancel                │
└──────────────────────────────────────────────────────────────────────────────┘
```

New ID composer (`Fix flaky gkeep test @sase^`, then typing `tool`):

```text
│ Fix flaky gkeep test @sase^tool▔                  (editor dimmed, token lit)  │
│ ╭────────────────────────────────────────────────────────────────────────╮   │
│ │ [@sase^ New ID]  tool▌                              ⚠ Used · line 36    │   │
│ ├────────────────────────────────────────────────────────────────────────┤   │
│ │  ⚠  ^tool is already used — Add sase tool command to wrap…     line 36 │   │ status row (not selectable)
│ │  ⊕  ^tool-2                                                 Next free │   │ selected
│ │ ✦ Suggested from your text                                             │   │
│ │  ✦  ^fix-flaky-gkeep                                         Available │   │
│ │ In use in sase.md                                              3 similar│  │
│ │    ○ Tool registry cleanup                              ^tool-registry │   │ info row (dim, 28pt)
│ ├────────────────────────────────────────────────────────────────────────┤   │
│ │ “Fix flaky gkeep test”                                                  │   │
│ │ New task in ▣ sase.md · ^tool-2 is available                            │   │
│ │                                              ↩ inserts @sase^tool-2    │   │
│ ╰────────────────────────────────────────────────────────────────────────╯   │
│ ↑↓ Move  ↩ Insert  ⌘↩ Insert & Capture  ␣ Insert & keep typing  esc Cancel  │
```

- **Opening rules.** These mirror `^` and are driven only by Bob's context, range, and
  intent:
  - Link intent:
    - A draft part that exactly equals a candidate `replacement` opens nothing.
    - Otherwise an `.edit` opens the picker, unless Escape suppressed auto-open for this
      token start. The seed is the text from the range start to the caret.
    - A caret-only move shows the chip.
  - New ID intent:
    - An `.edit` opens the picker only when the part is empty or the caret is at the
      part's end (typing forward). The seed is the whole part.
    - Any other edit, a caret-only move, or suppression shows the chip.
  - Leaving the token clears suppression and the chip, exactly as today.
- **Filter bar.**
  - Scope token capsule:
    - Link: `@sase:` with the route in the route color and the sigil secondary, then a
      "Tasks" caption.
    - New ID: `@sase^` (or `@sase:`) with a "New ID" caption.
    - Project-note intent: a "Project note" caption.
    - On multi-line drafts, append ` · line N` (the physical line of Bob's
      `marker_range.start`).
  - Placeholders:
    - Link: "Filter sase.md tasks, or type a new ID" (without ", or type a new ID" when
      Bob sent no `block_id`).
    - New ID: "Type a new ID for sase.md".
  - Trailing element:
    - Link: the count ("96 tasks" / "5 of 96").
    - New ID: an availability badge capsule:
      - `checkmark.circle.fill` "Available" (green).
      - `exclamationmark.triangle.fill` "Used · line 36" (orange).
      - `xmark.octagon.fill` with Bob's `allowed_description` (red).
      - A secondary "113 IDs in use" while the field is empty.
      - `questionmark.circle` "Checked on capture" for project-note intent.
- **Link rows (34pt).**
  - Status glyph:
    - In Progress `circle.lefthalf.filled` orange.
    - Next `circle.inset.filled` blue.
    - Todo `circle` secondary.
    - Blocked `pause.circle` secondary.
    - Other `circle.dashed` secondary.
  - Rich display text, the same as `^`.
  - A pink Pomodoro chip when the task's Task Link is queued ("BUGS", "Now · BUGS
    09:05–09:30", "Planned"), shown in both grouped and filtered modes.
  - A trailing `^block-id` locator in the block-ID color with match highlights.
  - Nested tasks indent 14pt per depth (max 2) in grouped mode.
  - Headers are the note's headings in document order, with a `#` glyph in the section
    color, the title, and a count. Tasks before the first heading go under "Top of
    note".
  - Filtered mode is one ranked flat list:
    - An exact block-ID match is pinned first.
    - When the filter is one token that satisfies Bob's rule and is not an exact
      candidate, a final **New ID** row appears (`plus.circle`, `^flaky`, badge
      "Available · add task text"). If that ID is in Bob's used list, a non-selectable
      status row appears instead ("Used by a Done task · line 80").
- **New ID rows.**
  - When the field is non-empty, the first row is the New ID row:
    - Available: `plus.circle.fill` green, "Available".
    - Taken: a non-selectable orange status row naming the user (line and text),
      followed by a selectable `plus.circle` "Next free" alternative
      (`BlockIDRules.nextFreeVariant`).
    - Invalid: a non-selectable red status row with Bob's description.
    - Project-note intent: selectable, badge "Checked on capture".
  - A "Suggested from your text" section follows:
    - Bob's suggestions with a `sparkles` glyph.
    - When the field is empty, all of them.
    - Otherwise those that start with or fuzzy-match the field, excluding one equal to
      the field.
  - With a non-empty field, an "In use in sase.md" section shows up to 5 dim 28pt **info
    rows**: used IDs that fuzzy-match the field, excluding the exact conflict already
    shown. Each info row has a status glyph (Done `checkmark.circle` green, Canceled
    `xmark.circle`, non-task anchor `paragraphsign`), text, and `^id` with highlights.
    Info rows are never selectable, and navigation skips them.
  - With an empty field and no suggestions, the empty state reads: "Type a new block ID"
    / "`<allowed_description>` · N IDs already used in sase.md". When the item has no
    text yet, add "Add the task text after the marker."
- **Detail strip (80pt).**
  - Link task row:
    - The full text (two lines).
    - Status · note icon + label › section · Pomodoro summary · "N sub-items" (when
      `child_count > 0`).
    - The right-aligned `↩ inserts @sase:tool` in palette colors.
    - After first use, a teaching line: "Then type #name, = to start, or =x to close".
  - Link New ID row: "New ID ^flaky" / "Available in sase.md — add task text after the
    marker to create a new Next task" / `↩ inserts @sase:flaky`.
  - New ID rows:
    - Line one is the item text in quotes (Bob's `body`), or "New task" when empty.
    - Line two is the outcome plus availability:
      - `^`: "New task in ▣ sase.md".
      - `:`: "New Next task in ▣ sase.md, linked into today's Pomodoro".
      - Project note: "Names the new project note".
    - Then `↩ inserts @sase^flaky-test`.
    - After first use, a teaching line: `^` "Then type + to make it a project note"; `:`
      "Then type #name, = to start, or + for a project note".
- **Empty states.**
  - Link with no identified linkable tasks: "No linkable tasks in sase.md" / "A task
    needs a ^block-id to be linked. Type a new ID, or use @sase+ to add an ID to a
    task."
  - Link where the note is missing: "sase.md doesn't exist yet" / "Type a new ID — Bob
    creates the note when you capture."
  - Link with no matches and no New ID row: "No sase.md tasks match “xyz”" / "Esc clears
    the filter."
  - Bob warnings collapse to one orange line, exactly like `^`.
- **Chip.**
  - Link: `list.bullet.rectangle.portrait` "Browse sase.md tasks ⇥".
  - New ID: `sparkles` "Suggest an ID for sase.md ⇥".
  - Tab, Down, and Ctrl-N open it; Escape hides it.
- **Quiet incomplete status.** Precedence is `active_task`, then `pomodoro_id`, then
  `block_id`:
  - `active_task`: "Pick an active task — press Tab to browse" (unchanged).
  - `pomodoro_id`: "Pick a task or type a new ID — press Tab to browse".
  - `block_id`: "Type a new block ID — press Tab for suggestions".
- **Sizing.**
  - Link uses the `^` budget: grouped rows plus headers, clamped to 4…11, fixed at open.
  - New ID uses a fixed budget of 6.
  - The panel grows once on open and never resizes while filtering. On short screens it
    keeps the three-row minimum and scrolls.

### Keyboard while a Block ID Picker is open

| Key                                                 | Link                                                                  | New ID                                                                                                                                                                                                                                                                   |
| --------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Printable keys, Cmd-A/C/V/X/Z, Ctrl-A/E, Left/Right | Native filter editing                                                 | Native editing. When the caret is at the end and newly typed or pasted text contains a character outside Bob's allowed set, commit the field text before that character as the ID, then insert the rest (from that character on) into the editor after it (type-through) |
| Return / keypad Enter, Tab                          | Insert the selected row                                               | Insert the selected row                                                                                                                                                                                                                                                  |
| Command-Return                                      | Insert, then capture                                                  | Insert, then capture                                                                                                                                                                                                                                                     |
| Shift/Option-Return, Shift-Tab                      | Consumed                                                              | Consumed                                                                                                                                                                                                                                                                 |
| Down / Ctrl-N / Ctrl-J, Up / Ctrl-P / Ctrl-K        | Move (wrap)                                                           | Move (wrap), skipping status and info rows                                                                                                                                                                                                                               |
| Page Up/Down, Home/End, Cmd-Up/Down                 | Page / first / last                                                   | Page / first / last                                                                                                                                                                                                                                                      |
| Escape / Ctrl-[                                     | Clear a non-empty filter, else cancel (suppress auto-open, show chip) | Same                                                                                                                                                                                                                                                                     |
| Backspace on an empty field                         | Delete the `:` separator and the part; route completion resumes       | Delete the `:`/`^` separator and the part                                                                                                                                                                                                                                |
| Ctrl-S                                              | Consumed                                                              | Consumed                                                                                                                                                                                                                                                                 |
| Ctrl-C                                              | Stash and close                                                       | Stash and close                                                                                                                                                                                                                                                          |

Return with no selectable row is a no-op that announces why ("No matches", "Type a new
ID", or "tool is already used on line 36").

## Phase bob-contract — bob-cli block-ID completion contract

All changes are additive to `capture-complete` schema version 1. Put new logic in a new
module (suggested: `src/native/capture_block_ids.rs`) with its own unit tests. Keep the
additions to `capture_complete.rs` to wiring, because that file is already large.

1. **`task_block_id` context.**
   - The right-hand side of `@route^block-id` becomes completable once the route
     resolves (`is_route_token`). This covers leading and trailing markers on the parent
     line and trailing markers on authored child lines, with the same line and item
     scoping as every other marker context.
   - `candidates` is always `[]`.
   - Add the context to `CompletionContext`, `context_label`, the help `Contexts:` list
     (keep examples readable; add
     `bob capture-complete -c 20 -f json -- 'Fix flaky test @sase^'`), and the JSON and
     human output.
2. **Replacement ranges exclude the project-note sigil.**
   - In both the `:` and `^` branches, a `+` directly after the ID part is the
     project-note sigil. Strip it from the block part before computing the replacement,
     so `@sase:x+` at cursor 7 returns `{6, 7}`.
   - A cursor after the sigil (and before `#` for `:`) returns an empty success.
   - A `+` after `#name` stays part of the Pomodoro name, unchanged.
   - Regression-test both markers, and `@cash:goog-exit+#bugs`.
3. **Additive top-level `block_id` object** is present exactly when the context is
   `pomodoro_block_id` or `task_block_id`:

   ```json
   "block_id": {
     "route": "sase",
     "relative_target": "sase.md",
     "note_exists": true,
     "marker": "^",
     "marker_range": { "start": 21, "end": 27 },
     "intent": "new",
     "body": "Fix flaky gkeep test",
     "allowed_character": "[A-Za-z0-9-]",
     "allowed_description": "A-Z, a-z, 0-9 or '-'",
     "suggestions": ["fix-flaky-gkeep", "flaky-gkeep-test"],
     "used": [
       { "id": "tool", "line": 36, "task": true, "status_symbol": " ", "status_name": "Todo", "text": "Add `sase tool` command to wrap common tool calls!" },
       { "id": "intro", "line": 3, "task": false, "status_symbol": null, "status_name": null, "text": "Some paragraph" }
     ]
   }
   ```

   - `marker` is `:` for `pomodoro_block_id` and `^` for `task_block_id`.
   - `marker_range` is the whole marker token's draft-global UTF-8 byte range (from `@`
     through any suffix), used for highlighting.
   - `intent` is one of:
     - `"project_note"` when the project-note `+` follows the ID part.
     - For `:` otherwise, `"link"` exactly when Bob's own whole-item parse would
       classify the item as `pomodoro_link` once a valid ID fills the part. Recommended:
       substitute a placeholder ID such as `x` into the replacement range, re-split the
       draft, re-run `parse_editor_item` on the item containing the cursor, and compare
       modes. That way intent can never drift from capture semantics. Any equivalent
       that reuses Bob's parser is acceptable.
     - `"new"` otherwise. `^` is always `"new"` or `"project_note"`.
   - `body` is the item's normalized parent-line body from `parse_editor_item`, or ""
     when there is none.
   - `allowed_character` is a regex matching exactly one allowed ID character, taken
     from the validator the marker uses (`[A-Za-z0-9_-]` for `:`, `[A-Za-z0-9-]` for
     `^`). `allowed_description` is the human wording those validators already use in
     their errors. Define both next to the validators so they cannot drift, and add a
     test that the regex agrees with the validator for every ASCII character.
   - `used` lists every ID `collect_done::block_ids_in_markdown` finds in the note (the
     exact scanner `reject_duplicate_block_id` uses), deduplicated by first occurrence,
     in document order, with 1-based `line`.
     - For task lines (use the same `note_tasks::scan` settings): `task: true`,
       `status_symbol`, `status_name`, and the task description as `text`.
     - For other lines: `task: false`, null status fields, and `text` set to the line
       with its list marker and trailing ` ^id` removed, trimmed, and bounded to 160
       characters.
     - `used` is `[]` when the note is missing and for `project_note` intent.
   - `note_exists` is false for a missing note, which is not an error.

4. **Link-only candidates.**
   - `pomodoro_block_id` keeps returning identified open tasks only for `link` intent.
     Restrict them to statuses the link path accepts (Ready, Blocked, Next, In Progress,
     via the same predicate the link resolver uses), so every candidate is linkable.
   - For `new` and `project_note` intents, `candidates` is `[]`: every existing ID would
     be a duplicate-ID error.
   - Link candidates gain an additive 1-based `line` and a nullable `pomodoro` object
     with the exact `active_task` shape (`line`, `name`, `time_range`, `is_current`).
     Fill it from `capture_active_tasks::read_ledger` (make it `pub(crate)`) using
     today's day file, and surface its bounded warnings (missing day file or section) in
     `warnings`.
   - Order stays document order when the query is empty. Query ranking is unchanged.
5. **Suggestions** (`new` and `project_note` intents, non-empty `body` only; at most 3,
   deterministic). Implement as a pure function with focused unit tests:
   - Phrases, in text order: each `` `code span` ``, quoted phrase, `(parenthetical)`,
     and `[[wikilink]]`. A quoted phrase runs from any opening `"` or `“` to the next
     `"` or `”`; the vault mixes them, as in `"agent data panels”`. For a wikilink, use
     its alias when present, else the target's last path segment with `_`/`-` as word
     breaks.
   - Words: ASCII alphanumeric runs, lowercased. Non-ASCII characters are separators.
   - Drop these stopwords: a, an, and, as, at, be, by, for, from, in, into, is, it, its,
     my, of, on, or, our, so, that, the, their, this, to, via, with, your.
   - Leading verbs (used only by candidate 3): add, build, check, clean, create, delete,
     document, enable, ensure, finish, fix, implement, improve, investigate, make,
     migrate, move, plan, read, refactor, remove, rename, research, review, run, start,
     stop, support, test, try, update, use, write.
   - Candidates, in order:
     1. The first phrase with at least one word, as its first 4 words.
     2. The first 3 prose words (markup stripped, so code and link words count).
     3. When the first prose word is a leading verb, the next 3 words.
   - Join words with `-`. Truncate at a word boundary to ≤ 32 bytes. Validate against
     the marker's grammar and deduplicate.
   - A candidate already in `used` takes the first free `-2`…`-9` suffix, or is dropped
     when none is free.
   - Examples to pin:

     | Body                                     | Suggestions                                                    |
     | ---------------------------------------- | -------------------------------------------------------------- |
     | "Fix flaky gkeep test"                   | `fix-flaky-gkeep`, `flaky-gkeep-test`                          |
     | `Add concept of "agent data panels”!`    | `agent-data-panels`, `add-concept-agent`, `concept-agent-data` |
     | "Add support for new `%hold` directive!" | `hold`, `add-support-new`, `support-new-hold`                  |
     | "Release v0.18.0!"                       | `release-v0-18`                                                |
     | "Tool" with `tool` used                  | `tool-2`                                                       |
     | non-ASCII-only body                      | none                                                           |

6. **Human output.** Block-ID contexts add one compact, plain-when-piped summary line.
   It names the note, intent, `N IDs in use`, and suggestions. Queued link rows append
   `· NAME` like `active_task` rows.
7. **Tests.**
   - Unit tests (grammar-level): context and replacement for leading, trailing, and
     child-line trailing `@route^`/`@route:`, including a second item of a batch; the
     `+` fix; and a cursor after the sigil or inside `=<X>`/`=x`.
   - Intent detection:
     - `@sase:` → link.
     - `x @sase:`, `@sase: x`, and `Parent\n- child @sase:` → new.
     - `@sase:x+` → project_note.
     - `@sase^` → new.
     - `@sase^x+` → project_note.
     - `@sase:` with an `s:1` or `%` marker → new.
   - Integration tests in `tests/cli/capture/`, against a temp vault:
     - `used` covers done tasks, non-task anchors, duplicates, and document order.
     - Missing note gives `note_exists: false`.
     - Link candidates are filtered by status and carry `pomodoro`/`line`.
     - New intents have empty candidates.
     - Suggestions follow the examples above.
     - Pin the JSON shape (including `block_id`'s absence for other contexts) and the
       human line.
   - Update `task_block_id_completion_offers_routes_but_not_authored_ids` to the new
     contract.
8. **Docs.** Rewrite the `docs/capture.md` `bob capture-complete` paragraphs that say
   `@route^` has no completion and that `pomodoro_block_id` lists identified tasks.
   Document the `task_block_id` context, the `block_id` object field by field, intent
   rules, link candidate filtering and `pomodoro`/`line`, suggestion rules and examples,
   the `+` range rule, and older-client compatibility. `just fmt lint test` must pass.

## Phase picker-generalize — source-agnostic capture picker (Mac, refactor only)

Refactor the `^` picker so a second source can plug in. **No user-visible change to
`^`:** same visuals, strings, keys, sizing, announcements, and the same UserDefaults key
(`org.bobs.bob-mac-capture.active-task-picker-used`). All existing picker tests keep
their assertions, adapted only to new names and types.

1. **Generic presentation (CaptureCore, new `CapturePickerPresentation.swift`).** Move
   the source-agnostic parts out of `ActiveTaskPickerPresentation.swift`. Suggested
   shape (adjust names if the code reads better, but keep one generic row, section, and
   presentation type that both sources emit):
   - `CapturePickerTaskStatus`: `inProgress`, `next`, `todo`, `blocked`, `done`,
     `canceled`, `other(String)`, with `init(symbol:name:)`. Map `/`, `*`, space, `?`,
     `x`/`X`, `-`, and otherwise the name (or symbol).
   - `CapturePickerGlyph`: `.task(status)`, `.anchor`, `.newID(availability)`,
     `.alternativeID`, `.suggestion`.
   - `CapturePickerAvailability`: `.available`, `.unchecked`, `.taken(line:text:)`,
     `.invalid(String)`.
   - `CapturePickerSection`: `id`, `kind` (`pomodoro`, `unqueuedInProgress`,
     `unqueuedNext`, `other`, `matches`, `noteHeading`, `suggestions`, `usedIDs`),
     `title`, `subtitle`, `timeRangeText`, `ordinal`, `isCurrent`, `countText`, `rows`.
   - `CapturePickerRow`: `id`, `bobIndex`, `glyph`, `displayText`, `textSegments`,
     `textMatchRanges`, locator (`route`/`blockID` plus match ranges, and a style of
     `route:block` for `^` or `^block` for block IDs), `chipText`, `badgeText`, `depth`,
     `isSelectable`, `insertion` (the exact string an accept inserts; nil when not
     selectable), `detail` (status text, route, section, summary line, child count,
     insertion preview parts), and `accessibilityLabel`.
   - `CapturePickerPresentation`: `mode`, `sections`, `orderedRowIDs` (selectable rows
     only), `rowsByID`, `totalCount`, `matchCount`, `countText`, `emptyState` (title and
     message), and `visibleRowBudget`.
   - `CapturePickerNavigation`: the renamed `ActiveTaskPickerNavigation`.
   - `ActiveTaskPickerIndex` stays source-specific and now returns
     `CapturePickerPresentation`. `ActiveTaskDisplayText` becomes `TaskDisplayText`,
     since both sources use it.
2. **Model state.**
   - `ActiveTaskPickerState` / `ActiveTaskChipState` become `CapturePickerState` /
     `CapturePickerChipState`, each with `source: CapturePickerSource` (only
     `.activeTask` for now).
   - Model properties: `picker`, `pickerPresentation`, `pickerChip`, a private
     `pickerIndex` (an enum wrapping the per-source index), and
     `pickerAutoOpenSuppressedStart`.
   - Computed: `pickerVisible`, `pickerChipVisible`, `pickerFilterIsEmpty`, and
     `selectedPickerRow`.
   - Generic operations: `updatePickerFilter`, select/next/previous/page/first/last,
     `acceptPickerRow(id:submitAfterInsert:)` (inserts `row.insertion`), `escapePicker`,
     `cancelPicker`, `removePickerTrigger` (the source supplies the trigger byte, `^`
     for active tasks), `openPickerFromChip`, and `dismissPickerChip`.
   - `parseNeedsActiveTask` becomes a generic "picker need" check with per-need status
     text. The source-specific `handleActiveTaskCompletion` stays and calls the generic
     `presentPicker`.
3. **Focus, routing, and controller.**
   - Renames: `CapturePanelFocusTarget.activeTaskFilter` → `.pickerFilter`;
     `ActiveTaskFilterField` → `CapturePickerFilterField` (accessibility identifier
     `org.bobs.bob-mac-capture.picker-filter`); router context
     `pickerVisible`/`pickerFilterIsEmpty`/`pickerChipVisible`; commands
     `acceptPickerRow`, `acceptPickerRowAndSubmit`, `nextPickerRow`,
     `previousPickerRow`, `pagePickerRowsDown`, `pagePickerRowsUp`, `firstPickerRow`,
     `lastPickerRow`, `escapePicker`, `removePickerTrigger`, `openPickerFromChip`;
     controller `repairPickerFilterFocusIfNeeded`.
   - The filter field's placeholder and accessibility label come from the presentation
     or source.
   - When editing begins, the filter field disables automatic dash, quote, and text
     replacement and spelling correction on its field editor, so `--` never becomes an
     em dash.
4. **View.**
   - Renames: `ActiveTaskPickerCard` → `CapturePickerCard` (rendering any
     `CapturePickerPresentation`); `ActiveTaskKeyHints` → `CapturePickerKeyHints` (items
     supplied per source); `ActiveTaskChip` → `CapturePickerChip` (label and icon per
     source); `ActiveTaskPickerHeightPolicy` → `CapturePickerHeightPolicy`;
     `CapturePanelLayout.activeTask*` → `picker*`; `ActiveTaskRichText` →
     `CapturePickerRichText`; `CaptureEditorPalette.activeTaskStatus` →
     `taskStatus(_:)`, extended to the new statuses per the UX spec.
   - The scope token, header, row, detail strip, and empty state read from generic
     fields.
5. **Tests and verification.**
   - Rename or adapt `ActiveTaskPickerPresentationTests`, `ActiveTaskPickerDesignTests`,
     `ActiveTaskDisplayTextTests`, router tests, and `CapturePanelModelTests` picker
     tests. Expected values must not change.
   - Add `CapturePickerTaskStatus` mapping tests.
   - Before and after the refactor, run the `BOB_MAC_CAPTURE_RENDER_DIR` rendered-image
     test for the four `^` states in both appearances, and confirm the images are
     visually identical.
   - Update README identifiers or names only where they are mentioned.

## Phase block-id-core — decode the contract and build the Block ID Picker engine (CaptureCore)

1. **Decoding (`CaptureModels.swift`).**
   - `CaptureCompletionResponse.blockID: CaptureBlockIDField?` from the top-level
     `block_id` object (`decodeIfPresent`). The existing candidate `block_id` key is a
     different level.
   - `CaptureBlockIDField` has `route`, `relativeTarget`, `noteExists`, `marker`,
     `markerRange: CaptureRange?`, `intent` (enum `link`/`new`/`projectNote`, with
     unknown values decoding to `link`), `body`, `allowedCharacter`,
     `allowedDescription`, `suggestions`, and `used: [CaptureUsedBlockID]` (`id`,
     `line`, `isTask`, `statusSymbol`, `statusName`, `text`).
   - Every field is tolerant: missing arrays decode as empty, strings as "", and
     booleans as false.
   - Add `CaptureCompletionContext.taskBlockID` (`"task_block_id"`).
   - Candidate `line` and `pomodoro` already decode; add tests.
2. **`BlockIDRules` (new).**
   - `init?(allowedCharacter:description:)` compiles Bob's regex and returns nil when it
     is missing or invalid; nil disables type-through and New ID rows.
   - `isAllowed(Character)`.
   - `isValid(String)`: non-empty and every character allowed.
   - `typeThroughSplit(old:new:)`: returns `(id, remainder)` only when `new` has `old`
     as a prefix and the appended text contains a disallowed character. `id` is `old`
     plus the appended text before that character, and `remainder` is the rest.
     Otherwise it returns nil.
   - `nextFreeVariant(of:used:)`: `-2…-99`, validated by the rule; nil when `-` is
     disallowed or no variant is free.
   - Membership is exact and case-sensitive, like Bob.
3. **`BlockIDPickerIndex` (new, emits `CapturePickerPresentation`).** It is built once
   per snapshot from `(field, candidates, route)` and implements the UX spec:
   - **Link** (intent `link`, or no field):
     - Empty filter: sections by `section` in first-appearance order, with nil → "Top of
       note". Rows carry status glyph, depth, Pomodoro chip text, the `^id` locator, and
       detail text (status, section, Pomodoro summary as in `^`: "Now · NAME
       HH:MM–HH:MM", "Queued in NAME", "Not in a Pomodoro"; child count). Count text is
       "N tasks".
     - Filter: fuzzy weights are display text 100, block ID 95, Pomodoro name 50,
       section 40, status name 30. Multi-token AND, like `^`; an exact block-ID match is
       pinned first; stable Bob-order ties.
     - The New ID row or used-status row applies only when a field and rules exist.
       Count text is "M of N".
     - Budget: rows plus headers, clamped 4…11.
   - **New ID** (intent `new` or `projectNote`):
     - Status rows, the alternative row, suggestions, and info rows per the UX spec.
     - Info rows fuzzy-match on the ID only; take the top 5 by score, then document
       order.
     - Availability is taken from `used`. `projectNote` is `.unchecked` and has no info
       rows.
     - Presentation-level `availability`, `usedCount`, and `body` feed the badge and
       detail strip.
     - Budget is fixed at 6.
   - `insertion` is the candidate's Bob `replacement` for link tasks and the ID string
     for New ID, alternative, and suggestion rows. The insertion preview is `@route` +
     marker + ID.
   - Accessibility labels:
     - Link: "Todo. Add sase tool command…. Block tool. Not in a Pomodoro."
     - New ID: "New ID tool-2, available."
     - Status row: "tool is already used on line 36 by Add sase tool command…".
4. **Real-bob fixtures.**
   - Build bob from the bob-cli repo (after `bob-contract`) and run it against a small
     temp vault: a note with two headings, identified Todo, Next, In Progress, and
     Blocked tasks, a done task with an ID, a non-task `^anchor`, a nested task, and a
     day file with a Pomodoro holding one Task Link.
   - Capture `capture-complete --format json` outputs as
     `Tests/Fixtures/block-id-*.json` for:
     - link `@notes:` (full) and `@notes:rea` (caret);
     - new `Fix flaky test @notes^`;
     - new via `Fix flaky test @notes:`;
     - `project_note` `@notes^x+`;
     - a missing note.
   - Document the generating command in a fixture README comment or test header.
5. **Tests (`Tests/CaptureCoreTests`).**
   - Decoding of every fixture and of the tolerant defaults.
   - `BlockIDRules` for both grammars: type-through splits for space, `#`, `=`, `+`, a
     pasted "x y"; no split on mid-field edits; invalid regex → nil; variants.
   - Link grouping, headers, "Top of note", chips, depth, exact-ID pinning, fuzzy
     ranking, count text, the New ID row, the used-status row for a done-task ID, and
     empty states (none identified, missing note, no matches).
   - New ID: an empty field with suggestions; an available typed ID; a taken ID (status
     row plus selected alternative); an invalid seed; suggestion filtering; info-row
     filtering (top 5, exact conflict excluded); `projectNote` unchecked;
     `orderedRowIDs` excludes non-selectable rows; budgets.
   - `^` presentation tests stay green.

## Phase block-id-flow — Block ID Picker flow, type-through, quiet states, and routing (Mac app)

1. **Source.** Add `CapturePickerSource.blockID(BlockIDPickerContext)` with the decoded
   field (optional), the route, the marker, the intent, and the rules. The index enum
   gains the block-ID case.
2. **Completion requests.**
   - Add `block_id` to `shouldRequestCompletion`'s needs set and `task_block_id` to its
     span kinds, so completion is requested while the caret is on an authored `^` ID.
   - Confirm that `routeReplacementRange` and the cached route completion never
     intercept a caret on the right-hand side of `:` or `^`.
3. **Interception (`handleCompletionResponse`).**
   - `pomodoro_block_id` and `task_block_id` never populate the inline list, even when
     empty. Route them to `handleBlockIDCompletion`, which applies the opening rules
     from the UX spec, generation-guarded like `^`.
   - A `task_block_id` response without a `block_id` object is treated as no completion.
   - Link intent with the caret after the range start refetches
     `captureComplete(draft, cursor: r.start)` for the full snapshot, as `^` does,
     falling back to the caret response with `snapshotIsPartial`.
   - New ID intent needs no refetch.
   - Present through the generic `presentPicker` (seed, restore caret, dismiss
     completion, clear chip, lock the editor, focus `.pickerFilter`, apply warnings).
4. **Accept.**
   - Generic `acceptPickerRow` inserts `row.insertion` into Bob's range with the
     stale-draft guard. The caret goes to `r.start + insertion.utf8.count`.
   - Then close the picker, unlock and refocus the editor, and set
     `suppressedCompletionAcceptanceDraft`.
   - Run an `.edit` analysis without completion and announce "Inserted @sase:tool".
   - Command-accept also submits.
   - A non-selectable or absent selection is a no-op with a spoken reason.
5. **Type-through (New ID source only, rules present).**
   - In `updatePickerFilter`, when `BlockIDRules.typeThroughSplit(old:new:)` returns a
     split, commit it. With the stale-draft guard, replace the range with `id`, insert
     `remainder` right after it, and put the caret after the remainder.
   - Then close the picker, unlock, and focus the editor.
   - Run an `.edit` analysis **with** completion at the new caret, so `#` opens
     Pomodoro-name completion, without reopening the ID picker for the committed token.
     The caret is past the token; keep suppression semantics correct.
   - The commit is literal: it happens even when the ID is taken or invalid, because
     Bob's preview reports that, exactly as without the picker.
6. **Trigger removal.** For a block-ID source, Backspace on an empty field deletes
   `[r.start − 1, r.end)` when the byte at `r.start − 1` is the source's marker (`:` or
   `^`). Then run an `.edit` analysis with completion, so route completion for `@sase`
   resumes. Otherwise it behaves like cancel.
7. **Chip, suppression, and lifecycle.**
   - Generic chip state carries the source. Labels and icons follow the UX spec.
   - The existing `^` lifecycle rules (hide/re-show, discard, submit, stash, and
     `setProcessClient(nil)`) apply to every source unchanged.
   - Submission and any keystroke abandon a pending open (generation guard).
8. **Quiet incomplete.** Extend the generic need check to `pomodoro_id` and `block_id`
   with the status texts and precedence in the UX spec. Suffix-only states such as
   `@sase:x#` (`pomodoro_name`) keep today's behavior.
9. **Router and controller.** The picker branch already routes generically. Confirm Tab,
   Down, and Ctrl-N open a block-ID chip, and that printable keys and Cmd-V still pass
   through natively. No new key commands are needed; type-through lives in the field's
   text-change path.
10. **Fake-bob fixtures (`Tests/Fixtures/fake-bob`).**
    - Add `capture-parse`, `capture-complete`, and dry-run responses shaped like the
      real-bob fixtures, for these drafts:
      - `@file:` (link, cursor 6);
      - `@file:rea` at cursor 9 and 6;
      - `@file:ready` (exact → nothing opens);
      - `Follow up @file^` (new);
      - `Follow up @file:` (new via `:`);
      - `Parent` plus a child `- child @file^` (child-line trailing marker);
      - a batch draft whose second item ends in `@file^`;
      - the type-through results `Follow up @file^new-id ` and
        `Follow up @file:new-id#`.
    - Reuse the existing `Follow up @file^new-id` parse and dry-run fixtures for
      accepts.
11. **Model tests (`CapturePanelModelTests`).**
    - Link:
      - Typing `@file:` opens the Link picker (no inline list, focus `.pickerFilter`,
        editor locked, no dry run recorded, `errorMessage == nil`, quiet status).
      - `@file:rea` refetches at the range start and seeds `rea`.
      - The exact `@file:ready` opens nothing.
      - A caret-only move shows the chip.
      - Accept inserts and live preview becomes `pomodoro_link`.
      - Command-accept submits.
    - New ID:
      - `Follow up @file^` opens the New ID composer with suggestions.
      - Typing `new-id` then Return gives the draft `Follow up @file^new-id`, with the
        caret at its end and the editor unlocked.
      - Typing a taken ID selects the alternative.
      - Space type-through gives `Follow up @file^new-id ` with the editor focused.
      - `#` type-through after `@file:` requests `pomodoro_name` completion.
      - A mid-part edit shows the chip instead of opening.
    - Anywhere: the child-line trailing marker and the second batch item both open the
      picker at Bob's global range.
    - Shared behavior:
      - Two-stage Escape, suppression, and chip reopen.
      - Backspace on an empty field deletes the separator and part (`@file:` → `@file`,
        `Follow up @file^` → `Follow up @file`) and resumes route completion.
      - The stale-draft guard for accept and type-through.
      - Hide closes, and re-show reopens.
      - `pomodoro_block_id` from an older Bob without `block_id` opens a Link picker
        without a New ID row.
    - Update `testCaret…`/`Do work @Dev^new-id` expectations where the picker now
      applies, keeping their intent.
12. **Router tests.** Chip routing for the block-ID source, printable pass-through while
    the New ID picker is open, and precedence against the prompts and the stash picker.
13. **Functional view.** The generic card already renders the block-ID presentation.
    Ensure the New ID status, alternative, suggestion, and info rows, the badge text,
    and the empty states render legibly, even if they are plain until `block-id-design`.
14. **README (behavior).** In the Runtime Contract:
    - Document the Block ID Picker: the intents, opening rules, snapshot plus local
      filtering (the same presentation-only precedent), accept and type-through, the
      chip, the quiet incomplete states, and older-Bob compatibility.
    - Add the keyboard table.
    - Update the Requirements list with the `task_block_id` context and the `block_id`
      object.
    - Remove the claims that the `^` family's authored ID "has no picker" and that
      `@route:` uses the inline list.

## Phase block-id-design — visuals, sizing, accessibility, docs, and macOS verification

1. **Visuals per the UX spec** in the generic card:
   - Scope-token variants with the ` · line N` suffix.
   - Link and New ID placeholders.
   - The availability badge capsule (fixed width range, no layout jump while typing).
   - Note-heading headers with the `#` glyph in the section color.
   - Link rows with Pomodoro chips and depth indent.
   - New ID status, alternative, and suggestion rows (`sparkles` in the accent color).
   - Dim 28pt info rows with highlights.
   - Detail-strip variants, with a separate UserDefaults key
     `org.bobs.bob-mac-capture.block-id-picker-used` for the teaching line.
   - Empty states, and the chip variants.
   - New ID key hints add `␣ Insert & keep typing`. All hints must fit at the 620pt
     minimum panel width; shorten labels, never wrap, if needed.
   - Keep `^` visuals unchanged.
2. **Marker highlight.**
   - While any picker is open, give the marker token a rounded accent background (about
     0.22 opacity, 0.35 under Increase Contrast) in the dimmed editor: `marker_range`
     for block-ID sources, and `[r.start − 1, r.end)` for `^`.
   - Apply it through the model's attributed-draft transform path (the
     `applyHighlighting` pattern, guarded by `isApplyingProgrammaticDraft`) and remove
     it on every close path.
   - It must never change `plainDraft`, the selection, or undo, or trigger
     `editorTextDidChange`. Add a model test.
3. **Sizing.**
   - `CapturePickerHeightPolicy` takes the source's `visibleRowBudget`: Link 4…11, New
     ID 6.
   - Info rows are 28pt inside the fixed viewport. The panel grows once on open, never
     resizes while typing or filtering, and shrinks on close.
   - Add controller metrics tests mirroring the `^` ones for both block-ID modes.
4. **Accessibility.**
   - Card labels: "Task picker for sase.md" / "New block ID for sase.md".
   - Open announcements: "sase.md tasks, 96 tasks" / "New ID for sase.md, 3
     suggestions".
   - Announce availability only on category change ("tool is already used on line 36;
     tool-2 is available"), and announce keyboard moves.
   - Status and info rows are static text without the button trait.
   - Fills strengthen under Increase Contrast, and color is never the only signal:
     badges carry text.
5. **Rendered-image review.**
   - Extend the `BOB_MAC_CAPTURE_RENDER_DIR` test to render, at 760pt and 620pt in light
     and dark appearance:
     - Link grouped and filtered (with a New ID row).
     - New ID: empty with suggestions, available, taken, and project note.
     - The link empty state.
     - The four `^` states, as a regression check.
   - Run it on macOS, copy the PNGs back, inspect them with the image reader, and
     iterate on spacing, contrast, truncation, and alignment until both modes look
     polished and consistent with `^`.
6. **README (visuals).** Card anatomy for both modes: headers, row kinds, badge, detail
   strip, info rows, marker highlight, key hints, sizing, and accessibility. Keep it
   consistent with the behavior text from `block-id-flow`.
7. **Real-panel smoke** (when a logged-in GUI session on `mac` is available):
   - Install a bundle together with a bob built from bob-cli master.
   - Against a real vault using dry runs only, walk through:
     - `@sase:` alone (browse, filter, Return, ⌘↩ with a dry-run preview);
     - `Fix it @sase^` (suggestion Return, typed ID, taken ID → alternative, space and
       `+` type-through);
     - `Fix it @sase:` then `#` type-through into Pomodoro names;
     - a child-line trailing marker;
     - Escape twice, Backspace, and the chip.
   - Record anything that could not be exercised.

## Verification (every phase)

- `bob-contract`: `just fmt lint test` in bob-cli. Paste one real
  `bob capture-complete -f json` output per intent from a temp vault into the phase
  notes.
- Mac phases: there is no Swift toolchain on Linux hosts, so verify on macOS.
  - Sync the opened checkout to the tailnet host `mac`, for example
    `rsync -a --delete --exclude .build <checkout>/ mac:/tmp/bob-mac-capture-<phase>/`,
    then run `just format-lint build test` there over `ssh mac`. `mac` is best-effort
    and often offline.
  - If it is unreachable, confirm the `macOS 26 SwiftPM` GitHub Actions run passes for
    the pushed commit (format lint, build, full `swift test`) and record the run ID.
  - If neither is possible, record the precise limitation on the phase bead and keep
    every automated assertion in place.
- Cross-repo: Mac phases that consume the contract must use fixtures generated by the
  `bob-contract` build, never hand-invented shapes that disagree with it.

## Done when

- Typing `@route:` as a whole item opens a large Link picker of that note's linkable
  tasks, grouped by the note's headings, fuzzy-filtered locally, with Pomodoro chips.
  Accept inserts Bob's ID; ⌘↩ inserts and captures.
- Typing `@route^`, or `@route:` on an item with text, opens the New ID composer.
  - Bob suggestions from the task text appear, and live availability is checked against
    every ID Bob's duplicate check sees.
  - A taken ID offers a one-key next-free alternative.
  - Similar used IDs are shown.
  - Type-through makes `space`, `#`, `=`, and `+` commit and keep typing, with draft
    output identical to typing without the picker.
- All of this works for leading, trailing, and child-line markers in any item of a batch
  draft. The marker being edited is highlighted.
- No red live-preview errors appear for incomplete `@route:`/`@route^`. Existing IDs are
  never offered where they would be duplicate-ID errors. Accepting never erases a
  project-note `+`.
- `^` is visually and behaviorally unchanged. It runs on the same generic picker, so
  keyboard, VoiceOver, Increase Contrast, and fixed-height sizing behave identically
  across both sources.
- bob-cli docs and the Mac README document the contract, behavior, and visuals.
  `just fmt lint test` passes in bob-cli, and `just format-lint build test` passes on
  macOS (or CI is green) with the new tests.
