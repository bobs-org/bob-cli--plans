---
tier: tale
title: Counted Ctrl+Shift+] toggles for bulk task/note conversion
goal:
  N<Ctrl+Shift+]> converts the cursor item and the N items below it, prompting for a
  section at most once.
size: medium
decisions:
  count_unit:
    ask:
      Should the count in N<Ctrl+Shift+]> count sibling list items rather than physical
      lines?
    choices:
      items:
        N counts sibling items below the cursor item; children ride along without using
        up the count
      lines:
        N counts physical lines like counted Alt+[/] and N#, so child lines use up the
        count
    default: items
    why: Matches "the current task and the 3 below it" even when tasks carry child lines
    answer: items
proposed_by: bbugyi200.apollo.7i
decided_by: auto
create_time: 2026-10-14 06:51:58
status: wip
---

# Plan: Counted `Ctrl+Shift+]` (toggle Obsidian task) for bulk task ⇄ note conversion

## Goal

Make task-status-cycler's **Toggle Obsidian task** gesture (`<Ctrl+Shift+]>` /
`<Ctrl+}>`) accept a Vim count. `N<Ctrl+Shift+]>` converts the item under the cursor
**and the N items below it** in one gesture, in either direction (tasks → plain
bullets/notes, or plain bullets → tasks). When the conversion needs a destination
section, the section picker opens **at most once per gesture**, and every converted item
goes to the one section chosen. A bare `<Ctrl+Shift+]>` (no count) must behave exactly
as it does today.

## Where the work happens

All source changes land in the **`bob-plugins`** linked repository. `bob-cli` needs no
source or doc changes. Open the repository first with the `/sase_repo` skill:

```bash
sase repo open bob-plugins -r "Add Vim-count support to task-status-cycler Toggle Obsidian task"
```

Use the printed path for all reads, writes, builds, and tests. All paths below are
relative to that checkout root. Read its `AGENTS.md` first. Key rules from it:

- `plugins/task-status-cycler/` builds from `src/` fragments listed in
  `src/fragments.json`. Edit the fragments, run `npm run build`, and never hand-edit
  `main.js`.
- Each hand-edited fragment must stay at or below **1000 lines**. `040-sections.js`
  (~921), `050-picker-and-links.js` (~887), and `180-plugin-editor-edits.js` (~925) are
  close to that limit. Put new logic in a **new fragment** and keep edits to those three
  small.
- After the changes, run `bob plugins sync` to deploy to the vault.

## How the single toggle works today (read before changing)

- **Key routes.** `src/140-plugin-vim.js` maps `<C-S-]>` and `<C-}>` (normal mode) to
  the Vim action `taskStatusCyclerToggleObsidianTask`. Its handler,
  `handleVimToggleObsidianTask()`, ignores `(cm, actionArgs)` and calls
  `toggleActiveObsidianTask(view.editor, undefined, view)`. The Obsidian command
  `toggle-obsidian-task` (`src/110-plugin-lifecycle.js`) calls
  `handleToggleObsidianTaskCommand` (`src/160-plugin-completion.js`). It runs from the
  command palette, or from the hotkey if the vault's `hotkeys.json` binds one.
- **Planning.** `getActiveObsidianTaskToggle(editor)` (`src/180-plugin-editor-edits.js`)
  calls the pure
  `getObsidianTaskToggleDocumentPlan(lines, activeLine, cursorCh, createdDateString)`
  (`src/050-picker-and-links.js`). That returns one of:
  1. `mode: "replace"`: an in-place rewrite of one line. This covers nested or non-`-`
     items, promotion in a note with no exact `## Tasks`, and demotion outside Tasks
     when no later section exists.
  2. `mode: "prompt"`, `promptKind: "demotion"`: a top-level `- [ ] #task` in the direct
     body of `## Tasks` opens the section picker.
  3. `mode: "prompt"`, `promptKind: "promotion"`: a top-level plain `-` bullet in a note
     that has an exact `## Tasks` opens the picker. Enter defaults to Tasks.
  4. `mode: "move"`, `targetSection: "nextSection"`: a demotion outside the Tasks body
     moves the block into the next Markdown section.
- **Blocks.** Each top-level item moves with its descendants. `getListItemBlockRange`
  stops at blank lines, headings, and lines indented no deeper than the item.
- **Picker.** `openDemotionSectionPicker` snapshots the document.
  `DemotionSectionPickerModal` (`src/100-editor-and-modal.js`) shows `plan.previewText`.
  On accept, `submitDemotionSectionPicker` checks that the snapshot is still fresh,
  calls `resolveObsidianTaskPromptDestination(prompt, destination)`
  (`src/040-sections.js`), and applies the resulting `move` plan with
  `applyObsidianTaskMovePlan`. That makes one `applyDocumentLineReplacement`, places the
  cursor, and centers it.
- **Prompt resolution helpers** (`src/040-sections.js`):
  - `removeSourceBlockForTaskPrompt` removes `prompt.sourceBlock` through the removal
    state (`createLineRemovalState` / `removeSourceListItemBlockFromState` /
    `removeDoubleBlankCurrentSeam`). For promotions it then runs
    `removeEmptySourceSectionFromState`.
  - `resolveSameSourceTasksPromotion` rewrites `prompt.sourceLine` in place.
  - `getTaskSectionPromptMovedBlock` / `LineText` / `CursorCh`,
    `buildObsidianTaskSectionMovePlan`, and `getCreatedSectionMovePlan` consume
    `movedBlock` / `originalMovedBlock` and put the cursor on the first moved line.
- **Count precedents** in this plugin:
  - `getPomodoroMoveOnlyAdditionalLines(actionArgs)` (`src/010-core.js`) turns
    CodeMirror Vim's `actionArgs.repeat` / `repeatIsExplicit` into "N additional" (bare
    = 0).
  - `getPendingVimRepeat(cm)` and `resetPendingVimInputState(cm, reason)` read and clear
    a count still pending in Vim input state. The counted `Alt+[`/`Alt+]` route uses
    them.
  - Nav's `consumePendingOpenTaskJumpRepeat` (bob-navigation-hotkeys
    `src/610-plugin-jumps-and-counted-keys.js`) is the model for resolving a count on
    the Obsidian-command route.

## Behavior to implement

### Count semantics

- `N<Ctrl+Shift+]>` with an explicit count `N ≥ 1` means "the cursor item plus up to N
  more". A bare press, or `additional = 0`, keeps today's single-item code path
  untouched.
- The **cursor item decides the direction** for the whole gesture. If its line is a
  proper Obsidian task (`isProperObsidianTaskLine`), the gesture demotes. Otherwise, if
  the line is promotable (`getPromoteLineToObsidianTaskRewrite` returns non-null), the
  gesture promotes. Otherwise nothing happens, as today.
- Range items whose line is already in the target form are **left exactly where they
  are, unchanged**: plain bullets or `[ ]` lines without `#task` during a demotion,
  proper tasks during a promotion. They still count toward N. A mixed range therefore
  never needs two prompts.
- The range only extends **downward** from the cursor. The cursor item is always the
  first converted item.
- The range never leaves the cursor item's section. It stops at any Markdown heading
  (`parseMarkdownHeadingLine`). It also never leaves the cursor item's list level.

### Range collection (new pure helper)

Add `collectObsidianTaskToggleRangeItems(lines, activeLine, additionalItems)`. It
returns the ordered list of `{ line, block: { startLine, endLine } }` for the cursor
item and the items the count reaches. A **sibling** is a list-item line
(`LIST_ITEM_MARKER_RE`) whose leading prefix matches the cursor item's prefix exactly.
The prefix is the whitespace and blockquote part before the marker:
`[ \t]*(?:>[ \t]*)*`. Use `getListItemBlockRange` for each item's block.

> [!decision] count_unit = items Walk forward from the end of the cursor item's block.
> Skip blank lines, and skip lines indented deeper than the sibling level (loose
> continuations or children). Each sibling list item found is the next range item, and
> the walk resumes after that item's block. Stop after N additional siblings, or at the
> first heading, the first line indented shallower than the sibling level, the first
> non-list line at the sibling level (for example a paragraph or a code fence), or EOF.
> Fewer than N available siblings is not an error: convert what exists, as Vim clamps
> counts.

> [!decision] count_unit = lines The range is physical lines
> `activeLine … activeLine + N`, clamped to the document like `cycleTaskStatusRange`. It
> is also cut at the first heading or at the first line indented shallower than the
> sibling level. Every sibling list item whose **first line** falls inside the range is
> a range item, and its whole block moves with it even if the block extends past the
> range end. Child or continuation lines inside the range never convert on their own;
> they only use up the count.

### Range document plan (new pure planner)

Add
`getObsidianTaskToggleRangeDocumentPlan(lines, activeLine, cursorCh, createdDateString, additionalItems)`:

1. If `additionalItems` is not a positive integer, return
   `getObsidianTaskToggleDocumentPlan(lines, activeLine, cursorCh, createdDateString)`
   unchanged.
2. Compute `single = getObsidianTaskToggleDocumentPlan(...)` for the cursor item. If it
   is `null`, return `null`.
3. Collect the range items. Select the ones in the source form for the direction. For
   each selected item, compute its toggled first line with
   `getObsidianTaskToggle(lineText, createdDateString)`. In the move and prompt modes
   below, only top-level `-` items (`isTopLevelDashListToggleLine`) are selectable, as
   in the single path; other siblings stay put. If exactly one item is selected (the
   cursor item), return `single` unchanged.
4. Pick the mode from `single.mode`. The cursor item's routing applies to every selected
   item:
   - **`replace`**: return a new `mode: "replaceLines"` plan whose `nextLines` is
     `lines` with every selected line replaced in place. Set `cursorLine = activeLine`
     and `cursorCh = single.cursorCh`.
   - **`move` (next section)**: remove every selected block from a
     `createLineRemovalState(lines)`. Work bottom-up so earlier coordinates stay valid,
     and run `removeSourceListItemBlockFromState` for each block, which also splices
     double-blank seams. Then find
     `findNextMarkdownSection(remaining, firstBlock.startLine)`, take its
     `getNextSectionBulletInsertion`, and call
     `buildObsidianTaskSectionMovePlan(remaining, insertion, movedBlock, single.sourceLineText, single.lineText, single.cursorCh, "nextSection", documentHasFinalNewline(lines))`.
     `movedBlock` is the concatenation, in document order, of each selected item's
     toggled first line followed by its descendant lines. If no next section exists,
     fall back to the `replaceLines` plan.
   - **`prompt`**: return `single` extended for many items:
     - `sourceBlocks`: every selected block in document order.
     - `items`: `[{ line, sourceLineText, lineText }]` for every selected item.
     - `movedBlock`: the concatenation of each toggled block.
     - `originalMovedBlock`: the concatenation of each original block.
     - `itemCount`.
     - `previewLines`: each item's preview, which is the toggled line for demotion and
       the source line for promotion, matching the single `previewText` rule.
     - `previewText`: `previewLines.join("\n")`.

     Keep every existing single-prompt field describing the cursor item (`sourceLine`,
     `sourceLineText`, `lineText`, `cursorLine`, `cursorCh`, `originalCursorCh`,
     `sourceBlock`, `sourceHeading`, `headings`, `promptKind`).

Place both new helpers in a **new fragment**, `src/055-obsidian-task-toggle-ranges.js`.
List it in `src/fragments.json` right after `050-picker-and-links.js`. Fragments share
one script scope, so the helpers can call functions from 040 and 050. Export both
helpers from `src/210-exports.js` so tests can reach them.

### Multi-item prompt resolution (small edits in `src/040-sections.js`)

Generalize the two helpers that assume one block. A prompt without `sourceBlocks` /
`items` must behave exactly as it does today.

- `removeSourceBlockForTaskPrompt`: remove every block in
  `prompt.sourceBlocks || [prompt.sourceBlock]` from last to first, with
  `removeSourceListItemBlockFromState`. Then run the existing promotion
  empty-source-section cleanup once. `splicedSeam` becomes true if any removal spliced a
  seam.
- `resolveSameSourceTasksPromotion`: rewrite every `prompt.items[i].line` to its
  `lineText`. Fall back to the single `sourceLine` when `items` is absent.

The other helpers need no change. `resolveExistingTaskSectionPromptHeading`, the
create-section path, and the `getTaskSectionPrompt*` accessors already insert
`movedBlock` or `originalMovedBlock` as one contiguous group and put the cursor on its
first line, which is the cursor item. Demotions still reject `Tasks` as a destination.
Promotions to a non-Tasks destination still keep every selected bullet plain, using
`originalMovedBlock`. A promotion source section that ends up empty after **all** blocks
leave is still removed.

### Count plumbing (both key routes)

- Add a shared helper in `src/010-core.js`, `getExplicitVimAdditionalCount(actionArgs)`:
  the explicit repeat, or 0 for a bare key. Make `getPomodoroMoveOnlyAdditionalLines`
  delegate to it and keep that name exported.
- **Vim route** (`src/140-plugin-vim.js`): define the action as
  `(cm, actionArgs) => this.handleVimToggleObsidianTask(cm, actionArgs)`. The handler
  computes `additional = getExplicitVimAdditionalCount(actionArgs)` and calls
  `this.toggleActiveObsidianTask( view.editor, this.getActiveObsidianTaskToggle(view.editor, additional), view)`.
  CodeMirror Vim has already cleared its input state before calling the action.
- **Command route** (`src/160-plugin-completion.js`): the `checking` pass stays a pure
  single-item eligibility check and consumes nothing. In the executing pass, call a new
  method `consumePendingObsidianTaskToggleCount(editor, view)`. It resolves `cm` with
  `resolveNormalModeVimCm(editor, view)`, reads `getPendingVimRepeat(cm)`, and returns 0
  when there is no cm or the count is not explicit, without resetting anything.
  Otherwise it calls `resetPendingVimInputState(cm, "counted-toggle-obsidian-task")` and
  returns the repeat. The executing pass then re-plans with
  `getActiveObsidianTaskToggle(editor, additional)`. Add a comment explaining why: if
  `hotkeys.json` binds `Ctrl+Shift+]`, Obsidian's command route wins the keydown, and
  CodeMirror Vim never sees the chord. The typed count would then be dropped and would
  also leak into the next Vim command, the failure that nav fixed for `Ctrl+Shift+J/K`.
  Exactly one route fires per press today (otherwise the existing in-place toggle would
  undo itself), so no dispatch guard is needed.
- `getActiveObsidianTaskToggle(editor, additionalItems = 0)`
  (`src/180-plugin-editor-edits.js`) calls `getObsidianTaskToggleRangeDocumentPlan` with
  the extra argument.
- `toggleActiveObsidianTask` gains a `replaceLines` branch. Add a new method,
  `applyObsidianTaskReplaceLinesPlan(editor, plan)`. It makes one
  `applyDocumentLineReplacement(editor, this.getEditorLineArray(editor), plan.nextLines)`
  call, so the gesture is one undo step, then sets the cursor to `plan.cursorLine` /
  `plan.cursorCh` (clamped) and does **not** center, matching today's in-place toggle.
  Put this method in the new fragment's code path or in `160-plugin-completion.js`,
  whichever keeps every fragment under 1000 lines. Plugin methods must live on one of
  the existing mixin classes; do not add a new mixin unless it is also registered in
  `src/200-install-methods.js`.
- Every gesture writes the document exactly once: once on accept for prompts, and once
  immediately for `move` and `replaceLines`.

### Picker presentation for many items (`src/100-editor-and-modal.js`)

When `snapshot.plan.itemCount > 1`:

- Title: `Move ${n} bullets to section` instead of `DEMOTION_PICKER_TITLE`.
- Preview: render one `.tsc-sdp-preview-text` row per entry of `previewLines`, at most 5
  rows, followed by a `+K more` row when more remain. The existing nowrap/ellipsis style
  applies per row, so `styles.css` needs no change unless the extra row needs a muted
  class.

Single-item prompts render exactly as today. The stale-snapshot and missing-section
notices are unchanged. A second counted press while a picker is open still re-uses that
picker.

## Edge cases to cover

| Situation                                                                                                            | Expected                                                                                                                                                                                                         |
| -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bare press, or explicit count that reaches no extra selectable item                                                  | Byte-for-byte today's single-item behavior                                                                                                                                                                       |
| `3<Ctrl+Shift+]>` on the first of 4+ Tasks-body tasks with tab-indented children (Depends-On, Work Log, sub-bullets) | One picker. On accept, all 4 blocks are demoted and land contiguously, in original order, in the chosen section with their children. Remaining tasks stay in Tasks. Cursor and center go to the first moved line |
| Same in a Tasks-only note                                                                                            | One picker. Blank Enter creates `## Requirements` once and holds all 4                                                                                                                                           |
| Count larger than the remaining siblings in the section                                                              | Converts what exists. Never crosses the next heading                                                                                                                                                             |
| Range contains already-converted items                                                                               | Those stay in place, unchanged                                                                                                                                                                                   |
| Counted promotion from `## Ideas` with exact `## Tasks` present                                                      | One picker. Enter converts all to tasks (with `[created::]`) appended to Tasks in order. A non-Tasks pick keeps them plain. `## Ideas` is removed only if it is now empty                                        |
| Counted promotion of bullets already in the Tasks body, Enter on Tasks                                               | Every selected line is converted in place                                                                                                                                                                        |
| Counted demotion outside Tasks                                                                                       | The whole group moves to the next section. If there is no next section, every item is converted in place                                                                                                         |
| Nested or non-`-` items, or a note without exact Tasks (promotion)                                                   | Every selected sibling is converted in place in one replace, cursor stays put, no centering                                                                                                                      |
| Document edited while the picker is open                                                                             | The existing "note changed" notice, no write                                                                                                                                                                     |
| CRLF documents, final-newline preservation                                                                           | Same as the single path                                                                                                                                                                                          |
| Command-route press with a pending Vim count                                                                         | Count consumed once, input state reset, same result as the Vim route                                                                                                                                             |

## Tests

Add `scripts/test-task-status-cycler-counted-toggle.cjs`, register it in the
`package.json` `test` script next to the other `test-task-status-cycler-*` files, and
reuse `scripts/task-status-cycler-harness.cjs` (`helpers`, `noteLines`,
`openDemotionPicker`, `acceptPickerDestination`, `acceptSelectedPickerRow`, `notices`,
`createTextEditor`). Cover:

1. **No-count parity.** For representative documents covering every single mode, the
   range planner with `additionalItems` 0, `undefined`, or a count that reaches no extra
   item deep-equals `getObsidianTaskToggleDocumentPlan`.
2. **Range collection.** Clamp at a heading, at a shallower line, at a same-level
   paragraph or fence, and at EOF. Children never count as items. Blank-line behavior,
   blockquote prefixes, and nested sibling levels all behave as specified. Test only the
   chosen `count_unit` branch.
3. **Counted demotion to an existing section, to a typed new section, and in a
   Tasks-only note.** Assert exact `nextLines`, the order of moved blocks, carried
   children, the cursor, and that `resolveObsidianTaskDemotionDestination` still rejects
   `Tasks`.
4. **Counted promotion.** Enter → Tasks, a non-Tasks destination stays plain, in-place
   same-source Tasks promotion, and removal of an emptied source section.
5. **Next-section move and in-place `replaceLines`.** Use a spy editor to show that
   `replaceLines` makes exactly one `replaceRange` call and does not center.
6. **Mixed range.** Already-converted siblings stay where they were.
7. **Vim route end to end.** Register mappings with a fake
   `window.CodeMirrorAdapter.Vim` that records `defineAction` handlers, then fetch
   `taskStatusCyclerToggleObsidianTask`. Calling it with
   `{ repeat: 3, repeatIsExplicit: true }` opens exactly one picker, and accepting moves
   all four tasks. `{ repeat: 1, repeatIsExplicit: false }` behaves like the bare press.
8. **Command route.** Give the editor `cm: { cm }` with
   `cm.state.vim = { mode: "normal", inputState: { prefixRepeat: ["3"], getRepeat: () => 3, … } }`
   and the cm methods `resolveNormalModeVimCm` requires.
   `handleToggleObsidianTaskCommand(true, …)` consumes nothing.
   `handleToggleObsidianTaskCommand(false, …)` applies the counted plan and clears the
   pending count. With no Vim cm, behavior is single-item.
9. **Picker UI.** The many-item title and preview rows (including `+K more`). The
   single-item modal is unchanged. A stale snapshot gives the notice with no write.
10. **CRLF and final newline** round trips for one counted move.

All existing suites must pass unchanged, in particular
`test-task-status-cycler-demotion-picker.cjs`, `test-task-status-cycler-promotion.cjs`,
and `test-plugin-build.cjs`.

## Docs and versioning

- Bump `plugins/task-status-cycler/manifest.json` `version` to `1.41.0`. Add one clause
  to its `description` saying that a Vim count (`N<Ctrl+Shift+]>`) toggles the cursor
  item and the next N sibling items together, prompting at most once and sending the
  whole group to one section.
- Update the Task Status Cycler row in `README.md` (version `1.41.0`), adding the same
  clause next to the existing `<Ctrl+Shift+]>` / `<Ctrl+}>` text.

## Verification

From the bob-plugins checkout:

```bash
npm run build
npm test
npm run validate
bob plugins sync
```

`npm test` starts with `build:check`, so a stale `main.js` fails fast. Finish by running
`bob plugins sync`, as the repo's `AGENTS.md` requires, so the deployed vault copy picks
up the change.
