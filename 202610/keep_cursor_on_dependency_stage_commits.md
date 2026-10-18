---
tier: tale
title:
  Keep the cursor and viewport on the task after Ctrl+Shift+B / Ctrl+Shift+Alt+B commits
goal:
  After a Depends on or Blocks stage commit, the cursor stays on the task line the
  gesture started on, the viewport does not jump to the top of the note, and the
  freshness stamp always lands on the dependent task.
size: medium
proposed_by: bbugyi200.apollo.8u
proposed_with: claude/opus
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.8u](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.8u.md)
- **COMMITS:**
  - [13b9e0f](https://github.com/bobs-org/bob-plugins/commit/13b9e0fce9b2a05b1cbe8928491146c81cb374b2)
    — fix(nav): keep the cursor and viewport on dependency stage commits

# Keep the cursor and viewport on the task after Ctrl+Shift+B / Ctrl+Shift+Alt+B commits

## Diagnosis (confirmed during planning)

**The suspicion is right.** After a Depends on (Ctrl+Shift+B) or Blocks
(Ctrl+Shift+Alt+B) stage commit, the cursor goes to line 0 and the viewport scrolls to
the top of the note. It happens only when the write changes the note's line count.

### Root cause

All paths below are in the `bob-plugins` linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md` first.

`applyEditorContentTransaction` in
`plugins/bob-navigation-hotkeys/src/180-editor-transactions.js` has two branches:

- **Same line count:** `applyEditorLineChanges` writes one change per changed line.
  CodeMirror keeps the cursor on its line.
- **Different line count:** it writes **one change that replaces the whole document**,
  from `{line: 0, ch: 0}` to the end of the last line, with the full new text.

The Depends on and Blocks writers call this helper with `finalCursor = null`. In that
case the transaction carries no `selection`, so CodeMirror 6 maps the existing cursor
through the change with `assoc = -1`. Any position inside a replaced range maps to the
range start, which is offset 0, the top of the file.

The viewport has the same problem. In `@codemirror/view` `ViewState.update`,
`scrollAnchorPos = update.changes.mapPos(scrollAnchor.from, -1)` also maps to 0. So even
writers that pass a cursor, or set one afterwards, scroll the view to the top.

**Why only "sometimes":** the line count changes only in these cases:

- adding a task's first prerequisite (a new `⛓️ DEPENDS ON` line);
- removing its last prerequisite;
- a same-note prerequisite gaining or losing a `🧱 BLOCKS` line (on by default:
  `dependencies.blocks_lines`);
- a Blocks-stage source gaining its first or losing its last BLOCKS line;
- a same-note Pomodoro cleanup deleting lines.

Adding to an existing Depends-On or BLOCKS line keeps the line count, so the cursor
stays put.

**Evidence:**

- **Real CodeMirror state:** with `@codemirror/state` 6, the cursor was at line 40 ch 5.
  A whole-document replacement with no selection moved it to line 0 ch 0. Per-line
  changes kept it on line 40.
- **End to end:** the plugin's own `planDependencyEdit` added a first same-note
  prerequisite (62 → 64 lines) and `applyEditorContentTransaction` committed it into a
  CM6-backed editor adapter. The cursor moved from line 30 to line 0.
- **Why the tests missed it:** every fake `TransactionEditor` in `scripts/` keeps its
  cursor when a transaction has no `selection`. Real CM6 does not.

Writers that call the helper with a null cursor on these gestures:

| Writer                                                                                         | Location                                                    |
| ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| vault/cross-note single add and every remove: `applyDependencyEdit` → `commitDependentContent` | `src/580-plugin-properties-and-dependency-edit.js` (~l.608) |
| same-note single add: `setLocalTaskDependencyLink`                                             | `src/600-plugin-dependency-mirror.js` (~l.155)              |
| same-note marked batch: `executeDependencyBatchWithoutInboxRoute`                              | `src/430-picker-vault-batch.js` (~l.729)                    |
| counted `N<Ctrl+Shift+B>` (two places)                                                         | `src/420-picker-counted-dependency.js` (~l.362, ~l.642)     |
| Blocks stage: `writeBlocksEditNote`                                                            | `src/592-plugin-blocks-edit.js` (~l.41)                     |
| cross-note BLOCKS follow-ups                                                                   | `src/591-blocks-follow-ups.js` (~l.106)                     |
| sub-project pass                                                                               | `src/218-plugin-project-subtree.js` (~l.736)                |

### Second, related bug: stale line index in same-note writers

`setLocalTaskDependencyLink` (600) and `executeDependencyBatchWithoutInboxRoute` (430)
do two things at the **pre-write** `cursor.line`:

- stamp `[fresh:: …]` through `stampDependencyParentLine`;
- move the cursor back afterwards with `setEditorCursorSafely`.

When a same-note prerequisite **above** the dependent gains a BLOCKS line, the dependent
moves down one line. Both the stamp and the cursor then land on the generated BLOCKS
line. Reproduced with the stage harness on `Other / Target / Parent`, adding Target to
Parent from line 2:

```
1 - [ ] #task Target [id:: Tasks__target] ^target
2 	- 🧱 **BLOCKS:** [[#^parent]] [fresh:: X]      <- stamp and cursor land here
3 - [?] #task Parent [dependsOn:: Tasks__target] ^parent
4 	- ⛓️ **DEPENDS ON:** [[#^target]]
```

`applyDependencyEdit` (580) already fixed this in commit `07c446b`: it stamps
`plan.parentLineNext` before the Pomodoro fold. The 600 and 430 writers never got that
fix.

## Changes (bob-plugins, `plugins/bob-navigation-hotkeys`)

The plugin uses the fragment build. Edit `src/`, never `main.js`. Each hand-edited
fragment must stay ≤ 1000 lines (`180-editor-transactions.js` is already at 917), so the
diff code goes in a new fragment.

### 1. New fragment `src/181-editor-line-diff.js` (listed after `180-editor-transactions.js` in `src/fragments.json`)

Plain function declarations, matching the style of fragment 180:

- **`editorLineAlignmentKey(line)`**: a line's identity for alignment purposes only. It
  drops the edits the dependency writers make to a task line:
  - Dataview inline fields `[key:: value]`, including the no-space form
    `[created::2026-10-18]`;
  - a trailing ` ^block-id`;
  - the checkbox status character, normalised so `[ ]`, `[?]`, `[/]` and the rest
    compare equal;
  - trailing whitespace.

  The key is never written anywhere.

- **`alignEditorLines(oldLines, newLines)`**: returns strictly increasing
  `[oldIndex, newIndex]` pairs of key-equal lines.
  - First trim the common prefix and suffix of exact raw-equal lines.
  - Then run Myers' greedy O((N+M)·D) diff on the keys of the middle, with a backtrack
    trace.
  - If D exceeds a cap (a named constant, e.g. `EDITOR_LINE_DIFF_MAX_EDITS = 2000`),
    stop and treat the whole middle as one unaligned gap. That is still far smaller than
    a whole-document replacement.
- **`planAlignedEditorChanges(oldLines, newLines, pairs, lineEnding)`**: returns
  ascending, non-overlapping Obsidian `EditorChange`s, all in **original-document**
  coordinates. This matches how `applyInlinePropertyAndScheduleLogTransaction` already
  batches changes.
  - An aligned pair whose raw text differs gets a full-line replacement
    `{line,0}..{line,len}`, the same shape `applyEditorLineChanges` writes.
  - A gap holding both old and new lines: replace old lines `[a, b)` with the new lines
    joined by `lineEnding`.
  - Insertion only:
    - if `a > 0`, insert `lineEnding + text` at the end of old line `a-1`, so CM6's
      scroll anchor (a line start) stays on its old line;
    - otherwise insert `text + lineEnding` at `{0, 0}`.
  - Deletion only:
    - if `b < oldLines.length`, delete `{a,0}..{b,0}`;
    - otherwise delete from the end of line `a-1` to the end of line `b-1`.
  - Invariant: applying the changes to the original text yields exactly the new text.
- **`mapEditorPositionThroughLineAlignment(position, oldLines, newLines, pairs)`**: maps
  a `{line, ch}` from the old document to the new one.
  - **Aligned line:** map to its partner line.
    - Unchanged line: keep `ch`.
    - Changed line: keep `ch` if it lies inside the common prefix; shift it by the
      length delta if it lies inside the common suffix; otherwise clamp it to the new
      line length.
  - **Line inside a gap that has new lines:** map to
    `newStart + min(offset, newCount - 1)` and clamp `ch`.
  - **Line inside a pure-deletion gap:** map to the first line after the gap, clamped to
    the last line, with `ch: 0`.

Export the three non-key helpers through `src/700-exports.js` next to
`applyEditorContentTransaction`.

### 2. Rewire `applyEditorContentTransaction` (`src/180-editor-transactions.js`)

1. If the texts are equal, return `true` (unchanged).
2. Compute `pairs = alignEditorLines(original.lines, next.lines)`.
3. **In-place case:** the line counts are equal and every pair is `[i, i]`. Keep the
   current `applyEditorLineChanges` call unchanged (same change shapes, no added
   selection), so existing tests and behaviour hold.
4. **Otherwise:** build the changes with `planAlignedEditorChanges`.
   - Selection is `finalCursor` when the caller passed one (explicit cursors always
     win).
   - Otherwise it is `mapEditorPositionThroughLineAlignment(getEditorCursor(cm), …)`.
   - With `cm.transaction`, send `{ changes, selection: {from, to}, annotation }`. Keep
     the `annotation` pass-through; `BLOCKS_MACHINE_WRITE_ANNOTATION` readers depend on
     it.
   - Fallback without `transaction`: apply the same changes bottom-up with
     `replaceRange`, then set the cursor, then restore scroll with the existing
     `getScrollInfo`/`scrollTo` code.
5. Delete the whole-document `{line: 0, ch: 0}` change. Leave `replaceEditorContent`
   alone: it is exported but has no `src/` callers.

This one change fixes every null-cursor caller in the table above. It also stops the
viewport jump for every whole-document caller that already passes a cursor (Alt+N, task
move, roll, cancel, and others), because CM6 now maps its scroll anchor through small
hunks.

### 3. Fix the stale-index same-note writers

- **`setLocalTaskDependencyLink`** (`src/600-plugin-dependency-mirror.js`):
  - Mirror 580: when `plan.changed`, stamp `plan.nextContent` at `plan.parentLineNext`
    **before** `foldLocalDependencyPomodoroCleanup`, then fold the stamped content.
    Stamping changes no line counts, so the fold's coordinates stay valid. Confirm the
    fold does not depend on the parent line's unstamped text.
  - Delete the post-write `setEditorCursorSafely(cm, cursor.line, …)`. The helper now
    keeps the cursor on the dependent's line.
- **`executeDependencyBatchWithoutInboxRoute`** (`src/430-picker-vault-batch.js`):
  - Stamp `batchPlan.nextContent` at `plan.parentLineNext` **before**
    `prepareProjectPomodoroCleanup`, passing the stamped plan as 580 does.
  - Remove the later
    `stampDependencyParentLine(batchCommitContent, this.cursor.line, …)`.
  - Delete the post-write `setEditorCursorSafely(this.editor, this.cursor.line, …)`
    block (the `finalCursorLine` lines).
- **Leave alone:**
  - any explicit `finalCursor` a caller already computes;
  - the intended review-walk advance (`decisions:answering-advances-the-walk`). On a
    `]s` landing, a resolving Task Card commit still moves the cursor to the next review
    item.

### 4. Tests (`scripts/`, Node `node:test`, no new npm dependencies)

New file `scripts/test-navigation-editor-line-diff.cjs`, added to the `test` script list
in the root `package.json`. It loads helpers through `navigation-hotkeys-harness.cjs`
and covers the following.

- **Alignment and change shapes:**
  - insertion after a modified line;
  - adjacent prerequisite and parent (both modified, BLOCKS and Depends-On inserted);
  - deletion at EOF;
  - insertion at the top;
  - empty document;
  - CRLF input;
  - equal-count shifted edits (+1 above, −1 below) taking the aligned path;
  - pure in-place edits still producing `applyEditorLineChanges` shapes (reuse
    `assertLineBoundedTransaction`).
- **Seeded property test** (deterministic PRNG, a few hundred cases): random documents
  and random insert, delete, modify and key-preserving-modify scripts.
  - Applying `planAlignedEditorChanges` in original coordinates, as the harness
    `TransactionEditor` does, yields exactly the new text.
  - Every unchanged aligned line maps to its partner with the same `ch`.
- **CM6-semantics double** (`Cm6MappingEditor`):
  - When a transaction has no `selection`, it maps the cursor through the changes using
    CM6 simple-mode `assoc = -1` rules, with positions in original offsets:
    - a pure insertion at `pos` keeps `pos` before the insert;
    - a position at or inside a non-empty replaced range `[from, to)` maps to `from`;
    - `pos == to` maps after the insert;
    - later positions shift by the length delta.
  - It always maps a "scroll anchor" (the start of a line above the edit) by the same
    rules.
  - Assert, for each case below, that the cursor ends on the dependent or source line
    and the anchor line maps to itself (before the fix both went to line 0):
    - first cross-note prerequisite (Depends-On inserted);
    - same-note prerequisite directly above (BLOCKS inserted, both task lines modified);
    - last prerequisite removed;
    - Blocks source gaining its first BLOCKS line with an adjacent same-note T above.
- **Gesture regressions:**
  - In `scripts/test-navigation-dependencies-stage-writes.cjs`, cover the same-note
    **single** add (`chooseTaskDependency`) and the **marked batch**
    (`executeDependencyBatch`) with Target above Parent and a stub stamper. Assert:
    - `[fresh:: …]` is on the Parent line;
    - the BLOCKS line is exactly `\t- 🧱 **BLOCKS:** [[#^parent]]`;
    - the final cursor is on Parent's new line.
  - `applyDependencyEdit` adding a cross-note first prerequisite: the dependent editor's
    transaction selection is on the parent line. Put it in the existing writer or stage
    test file that already stubs the plugin.
  - `applyBlocksEdit` (`scripts/test-navigation-blocks-edit.cjs`) where the source gains
    its first BLOCKS line: the source editor's transaction selection stays on the source
    line.

### 5. Build, version, docs, deploy

- In bob-plugins:
  - Run `npm run build`, then `npm test`.
  - **Baseline at planning time:** 3013 tests, 7 failing on `master`, unrelated to this
    change:
    - "Live Preview places one glyph after each unblocked link";
    - "widget mousedown places the cursor past the link";
    - "rebuild triggers skip the cursor but take doc, viewport, and refresh";
    - "Reading view glyphs unblocked rows and dedupes";
    - "Keep counts once and saturates at 999 without reset";
    - "stage ranker filters 1,000 synthetic tasks under 16 ms per keystroke";
    - "forward conversion carries a Depends-On line and reverse restores it".
  - Add no new failures. Do not fix those seven here. If no bead tracks them, file one
    through `/sase_new_task`.
  - Bump `bob-navigation-hotkeys` to `2.45.2` in `manifest.json` and in the root
    `README.md` plugin table. Run `npm run validate`.
  - Run `bob plugins sync` to deploy to the vault.
- In bob-cli `docs/task-dependencies.md`, add one sentence each to §6.6 (One gesture
  writes) and §15.6 (One gesture writes and undo). It should say: nav applies editor
  writes as line-level edits, never a whole-note replacement, so the cursor stays on the
  task line the gesture started on and the viewport does not move. The review-walk
  advance on a `]s` landing is the one intended exception.

## Follow-up to file (not in scope)

Use `/sase_new_task` to file this as its own bead.
`BulletPropertyPickerModal.settleReviewOrigin` (`src/350-picker-task-card.js`) reads the
after-line at the pre-write `reviewLineIndex`. When a same-note prerequisite above the
reviewed task gains a BLOCKS line, that index points at the BLOCKS line. The review walk
may then misjudge whether the answered row left the walk.

## Acceptance

- In Obsidian, on a note longer than one screen, scroll to a task well below the top.
  Each of the following leaves the cursor on that task's line and the viewport where it
  was:
  - Ctrl+Shift+B adding its first prerequisite (same-note above, same-note below, and
    cross-note);
  - Ctrl+Shift+B removing its last prerequisite;
  - Ctrl+Shift+Alt+B blocking a task when the source has no BLOCKS line yet;
  - a marked batch;
  - counted `N<Ctrl+Shift+B>`;
  - Task Link mode, where the cursor stays on the linked task in the linked note.
- `[fresh:: …]` is always written on the dependent task, never on a generated BLOCKS
  line.
- One Ctrl+Z still undoes a same-note gesture.
- `npm run build:check`, `npm test` (no new failures beyond the seven listed) and
  `npm run validate` pass.
- `bob plugins sync` has run.
