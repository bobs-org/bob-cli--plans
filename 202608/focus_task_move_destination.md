---
tier: tale
title: Focus the destination note after a Ctrl+Shift+M task move
goal:
  A successful Ctrl+Shift+M task move focuses the destination note and places the cursor on the moved task's line
  instead of staying in the source note.
proposed_by: bbugyi200.athena.tp
create_time: 2026-08-05 20:44:45
status: done
---

- **PROMPT:**
  [prompts/202608/focus_task_move_destination.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/focus_task_move_destination.md)
- **AGENTS:**
  - [bbugyi200.athena.tp](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.tp.md)
- **COMMITS:**
  - [2728043](https://github.com/bobs-org/bob-plugins/commit/272804381359a6f579180ee9ca172c96b71a8391) —
    feat(navigation-hotkeys): focus destination note after task move

# Plan: Focus the destination note after a `Ctrl+Shift+M` task move

## Problem

`Ctrl+Shift+M` (bare) and `N<Ctrl+Shift+M>` (counted) move the task under the cursor — plus any requested follow-on
tasks — from the current note into an area or open-project note chosen from a picker. When the move succeeds, the editor
stays in the **source** note: the plugin deliberately restores the source editor's scroll position and focus, and the
only feedback about the destination is a `Moved N tasks to <name>` toast.

Bryan wants the opposite: after a successful move, Obsidian should **focus the destination note** and put the cursor on
**the line the task was moved to**, so the moved task can be edited immediately without navigating there by hand.

## Repository

All code changes are in the **`bob-plugins`** linked repo (plugin `bob-navigation-hotkeys`). Open it with the
`/sase_repo` skill before reading or editing anything:

```bash
sase repo open bob-plugins -r "Focus the task-move destination note after Ctrl+Shift+M"
```

Every path below is relative to that checkout. **The `bob-cli` repo needs no changes** — its `docs/` and `README.md` do
not document the `Ctrl+Shift+M` move flow (`docs/projects.md` documents only the `Ctrl+Shift+P` bullet-property picker
and "Create project note from task").

## How the move works today

- `plugins/bob-navigation-hotkeys/main.js`
  - `move-tasks-to-note` command (~line 13355) → `openTaskMoveDestinationPicker(editor, view, options)` (~line 17586).
    Counted presses reach the same method from `registerCountedTaskMoveInputListeners` (~line 15757).
  - The picker builds a frozen `session`:
    `{ sourceFile, sourcePath, sourceView, editor, sourceContent, cursor, scroll, countExplicit, discovery, ranges }`,
    then opens `TaskMoveDestinationPickerModal` (~line 7737) whose `openItem` calls
    `plugin.commitTaskMoveSession(session, entry)`.
  - `commitTaskMoveSession` (~line 17836): re-validates the session, snapshots every Markdown file, calls
    `planTaskMoveAcrossFiles(...)` (~line 9244), then writes `plan.changes` in the order
    `[destination, ...auxiliaryPaths, source]` through `writeTaskMoveChange` (~line 17696).
  - The source cursor is placed on `plan.nextSourceLine` by passing `finalCursor` to `writeTaskMoveChange` for the
    source path only.
  - On success it calls `restoreTaskMoveSourceContext(session)` (~line 17811) — which restores the source editor's
    scroll and focus — then shows the `Moved N tasks to <name>` notice and returns `true`.
  - On failure it rolls back, calls `restoreTaskMoveSourceContext(session)`, notices, returns `false`.
- `planTaskMoveAcrossFiles` returns
  `{ valid, error, changes, ranges, movedBlocks, movedBlockIds, idReplacements, nextSourceLine, destinationKind }`.
  **Nothing in the plan says where in the destination the tasks landed** — `insertTaskMoveBlocks` (~line 8996) returns
  only `{ valid, error, content, createdSection }`.

So two things are missing: the destination insertion line is never computed, and nothing ever opens or focuses the
destination note.

## Design

### 1. `insertTaskMoveBlocks` reports the inserted line

Add an `insertedLine` field (0-based index, within the returned `content`, of the first moved line) to **all three**
success returns of `insertTaskMoveBlocks` (~line 8996). Keep the existing `{ valid, error, content, createdSection }`
fields and the `Object.freeze` shape unchanged.

- **Created-section branch** (area destination with no `## Tasks` header): the branch builds `next` by string
  concatenation. Capture the index immediately before appending `movedLines`, from the prefix built so far — the prefix
  always ends with `source.lineEnding`, so `next.split(source.lineEnding).length - 1` is exactly the 0-based index of
  the first moved line.
- **Placeholder branch**: `insertedLine = placeholderIndex`.
- **Default branch**: the code may unshift a `""` separator into `insertion` when `insertAt === headerIndex + 1`, so use
  `insertedLine = insertAt + (insertion.length - movedLines.length)` (equivalently `insertAt + 1` when the blank
  separator was added, `insertAt` otherwise). Do not hardcode; derive it so the two sub-cases stay correct.

The invalid returns keep their current shape.

### 2. `planTaskMoveAcrossFiles` exposes a destination anchor

Add three fields to the valid return of `planTaskMoveAcrossFiles` (~line 9358), alongside `nextSourceLine`:

- `destinationLine: insertion.insertedLine`
- `destinationAnchorText: movedBlocks[0][0]` — the first line of the first moved block **after** block-link/`[id::]`
  rewriting, i.e. byte-for-byte what gets written into the destination.
- `destinationBlockId: getTrailingBlockId(movedBlocks[0][0]) || null` — may legitimately be `null`;
  `prepareTaskMoveBlockIdentities` (~line 9091) skips lines without a trailing `^block-id`, so moved tasks are not
  guaranteed to have one.

`movedBlocks[0][0]` is always a real `#task` checkbox line: `discoverMovableObsidianTaskTargets` rejects a session whose
first target is not one ("Move tasks must start on a real #task checkbox", ~line 8558), and `rebaseTaskMoveBlock` only
dedents.

### 3. New pure helper: `resolveTaskMoveDestinationLine(content, anchor)`

**Why this is needed rather than trusting `plan.destinationLine` blindly:** the planned index is computed against the
destination content as of the write, but the cursor is placed _after_ the write lands, and other plugins react to that
write. In particular `plugins/bob-project-tasks/main.js` listens on `metadataCache.on("changed")` (~line 171) and, after
a debounce, writes `task_count` / `open_task_count` into project frontmatter via `processFrontMatter` (~line 269). When
those keys were absent, that insertion adds frontmatter lines and shifts every task line down — the cursor would land
above the moved task. Re-anchoring makes the jump correct regardless.

Add a module-level pure function next to the other task-move helpers:

```js
function resolveTaskMoveDestinationLine(content, anchor)
// anchor: { line, text, blockId }
// returns: { line, source: "planned" | "block-id" | "text" | "clamped" }
```

Resolution order:

1. `"planned"` — `lines[anchor.line]` is exactly `anchor.text`.
2. `"block-id"` — `anchor.blockId` is set and `findTaskLineByBlockId(lines, anchor.blockId)` (existing helper,
   ~line 6566) returns a line. That helper already requires the match to be a real Obsidian task line, so
   frontmatter/fence lookalikes cannot win.
3. `"text"` — the first line exactly equal to `anchor.text` that is neither in frontmatter nor in a fenced code block
   (use `getMarkdownLineContexts`, as `collectTaskMoveBlockIds` does at ~line 9073).
4. `"clamped"` — fall back to `anchor.line` clamped to `[0, lines.length - 1]` (and `0` for empty content).

Export it from the `module.exports.helpers` block (~line 19434+) next to `planTaskMoveAcrossFiles` and
`insertTaskMoveBlocks` so it is unit-testable.

### 4. Focus the destination on success

In `commitTaskMoveSession`, **only on the success path** (the `restoreTaskMoveSourceContext(session)` call _after_ the
write `try/catch`, not the one inside `catch`), replace the source-restore with a destination focus. Keep the
failure-path `restoreTaskMoveSourceContext(session)` call and keep `restoreTaskMoveSourceContext` itself — it stays the
correct behavior when nothing moved.

Keep the existing `Moved N tasks to <name>` notice and the `return true`. Sequence on success:

1. Build the anchor from the plan:
   `{ line: plan.destinationLine, text: plan.destinationAnchorText, blockId: plan.destinationBlockId }`.
2. `await this.focusTaskMoveDestination(destinationFile, anchor)`.
3. Show the existing notice, `return true`.

New method `async focusTaskMoveDestination(file, anchor)`:

- `this.captureActiveFilePosition()` first — mirrors `openChildNote` (~line 17494) and `openAlternateFile` (~line 18600)
  so the source note's cursor/scroll is remembered and alternate-file navigation still works after the jump.
- `const opened = await this.openMarkdownFileWithLeafReuse(file, "Moved tasks, but could not open <basename>")`. This is
  the same helper every other "open a related note" command uses (~line 13263): it activates an existing leaf for the
  file when one exists (`findMarkdownLeafByPath` + `activateWorkspaceLeaf`, which handle sidebar and pinned leaves),
  otherwise opens it in the active leaf.
- If `opened` is false, return `false` — the move already succeeded and must **not** be rolled back;
  `openMarkdownFileWithLeafReuse` already showed the failure notice.
- Otherwise `return this.jumpOrDeferTaskMoveDestination(file.path, anchor)`.

New method `jumpOrDeferTaskMoveDestination(path, anchor, retriesRemaining = TASK_MOVE_DESTINATION_JUMP_RETRIES)`,
modeled directly on `jumpOrDeferDashTasks` (~line 16722):

- Cancel any pending task-move jump, then try `jumpToActiveTaskMoveDestination(path, anchor)`.
- On failure and `retriesRemaining > 0`, re-arm via `deferToNextFrame` into `this.pendingTaskMoveJumpDeferred` and
  recurse with `retriesRemaining - 1`.
- When retries are exhausted, return `false` silently — do not turn a completed move into an error toast; the
  `Moved N tasks…` notice already told the user the move worked.
- Add `const TASK_MOVE_DESTINATION_JUMP_RETRIES = 8;` next to `DASH_TASKS_JUMP_RETRIES` (~line 112).

New method `jumpToActiveTaskMoveDestination(path, anchor)`:

- Return `false` unless the active Markdown view is `path` and exposes an editor with `getValue` (this is what makes the
  retry loop wait for the freshly opened leaf).
- `resolveTaskMoveDestinationLine(editor.getValue(), anchor)` → `setEditorCursor(editor, { line, ch: 0 })`.
- Then center the line the way the open-task jump does: `scheduleOpenTaskJumpCenter(this, editor, line, 0)` (~line
  5864). Reusing it is intentional — it defers one frame so Vim normal mode's trailing cursor-visibility scroll cannot
  override the centering, and it cancels its own stale frame. Centering is best-effort and must not flip the result to
  `false`.

New method `cancelPendingTaskMoveJump()` (`cancelDeferred` + null the field), initialize
`this.pendingTaskMoveJumpDeferred = null` alongside the other pending-deferred fields in `onload` (~line 13292+), and
call the cancel from the teardown `this.register(() => { ... })` block (~line 13543) next to
`cancelPendingDashTasksJump()`.

### Edge cases to respect

- Destination equal to the source is already rejected before any write (~line 17847), so the focus step never no-ops on
  the source note.
- A destination already open in another tab (including a pinned one or a sidebar leaf) is activated rather than
  duplicated — that is `openMarkdownFileWithLeafReuse`'s existing behavior.
- If the destination leaf is in reading/preview mode, cursor placement is a no-op. Accept it; do not force a mode switch
  and do not report failure.
- Counted moves (`N<Ctrl+Shift+M>`) jump to the **first** moved task, which is the task the cursor was on when the
  picker opened.
- The source cursor still ends up on `plan.nextSourceLine` (unchanged, applied during the source write), so returning to
  the source note lands where it does today.

## Files to change

| File                                           | Change                                                                                                                                                                                                                                                                                                                |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugins/bob-navigation-hotkeys/main.js`       | `insertedLine`; plan destination anchor fields; `resolveTaskMoveDestinationLine`; `focusTaskMoveDestination` / `jumpOrDeferTaskMoveDestination` / `jumpToActiveTaskMoveDestination` / `cancelPendingTaskMoveJump`; `TASK_MOVE_DESTINATION_JUMP_RETRIES`; success-path swap in `commitTaskMoveSession`; helper exports |
| `plugins/bob-navigation-hotkeys/manifest.json` | `1.16.0` → `1.17.0` (feature)                                                                                                                                                                                                                                                                                         |
| `README.md`                                    | Bob Navigation Hotkeys row: version `1.17.0`, and extend the description so `Ctrl+Shift+M` is described as moving the task **and focusing the destination note on the moved task**                                                                                                                                    |
| `scripts/test-navigation-hotkeys.cjs`          | New/extended coverage (below)                                                                                                                                                                                                                                                                                         |

Repo convention (see `git log -- plugins/bob-navigation-hotkeys/manifest.json`): the manifest bump and the README row
update ship in the **same commit** as the behavior change; a `feat` gets a minor bump.

## Tests

`scripts/test-navigation-hotkeys.cjs` is plain `node --test`. Extend the existing task-move sections rather than
starting a new file.

1. **`insertTaskMoveBlocks` `insertedLine`** — extend the existing assertions (~lines 4954–5065) so each case also
   checks that `content.split("\n")[insertedLine]` is the first moved line, for: a `## Tasks` section with existing
   tasks; a section whose body is only the placeholder task; an empty `## Tasks` section (blank separator inserted); a
   `## Tasks` section followed by trailing blank lines and a later `##` header; an area note with no `## Tasks` header,
   both with and without a terminal newline; and CRLF content.
2. **`planTaskMoveAcrossFiles` anchor fields** — extend the existing plan tests (~lines 5097–5190) with
   `assert.equal(plan.changes.get(destinationPath).after.split("\n")[plan.destinationLine], plan.destinationAnchorText)`
   for a project destination, an area destination, and a counted move (assert the anchor is the _first_ moved task).
   Also assert `destinationBlockId` matches the trailing `^id` and is `null` for a task with no block ID.
3. **`resolveTaskMoveDestinationLine`** — new unit tests: planned line hits (`source: "planned"`); content shifted down
   by two inserted frontmatter lines resolves via `"block-id"`; the same shift with no block ID resolves via `"text"`;
   an anchor whose text appears only inside a fenced code block or frontmatter is _not_ matched there (falls through to
   `"clamped"`); a planned line past the end of a shrunken document clamps; empty content returns line `0`.
4. **`focusTaskMoveDestination` behavior** — add a small harness (alongside `createTaskMovePickerHarness`, ~line 3586)
   that stubs `captureActiveFilePosition`, `openMarkdownFileWithLeafReuse`, and `getActiveMarkdownView`, and assert: the
   destination file is the one passed to `openMarkdownFileWithLeafReuse`; the cursor lands on the anchored line when the
   destination view is already active (synchronous path); a failed open returns `false` without throwing and without
   moving the cursor; a drifted destination still anchors by block ID.

**Known coverage gap, stated deliberately:** `commitTaskMoveSession` has no end-to-end test today —
`createTaskMovePickerHarness` stubs it (`plugin.commitTaskMoveSession = async () => true;`, test file ~line 3618)
because a real run needs vault snapshot/`process`/leaf mocks. This plan does not add one. The new behavior is covered at
the seams instead (pure anchor resolution + the focus method). If the implementing agent finds the vault mock cheap to
build, an integration test asserting "destination focused on success, source restored on failure" is a welcome bonus,
not a requirement.

## Validation

From the `bob-plugins` checkout:

```bash
npm test          # node --test across all plugin test files
npm run validate  # manifest + `node --check` syntax validation
```

Both must pass. Then deploy to the vault — `bob-plugins/AGENTS.md` requires a sync after any change to that repo, and in
a SASE workspace `bob plugins sync` must be pointed at the checkout because its default source path
(`~/projects/github/bbugyi200/bob-plugins`) does not exist there:

```bash
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run   # preview first
bob plugins sync -p bob-navigation-hotkeys -r "$PWD"
```

Run both from the `bob-plugins` checkout root so `$PWD` resolves to it. If sync reports the managed file as "dirty in
vault; use -F/--force", stop and report rather than forcing — that means the vault git repo has uncommitted state for
the file and needs a human look.

Note for the user in the final report: **Obsidian must reload the plugin** (disable/enable Bob Navigation Hotkeys, or
reload the vault) to pick up the new `main.js`.

## Manual smoke test (for Bryan, after reload)

1. Put the cursor on a `#task` in a note with several tasks; press `Ctrl+Shift+M`; pick an open project note that is
   **not** currently open in any tab → the project note opens in the current tab with the cursor on the moved task,
   centered.
2. Repeat with the destination already open in another tab → that tab is activated (not duplicated), cursor on the moved
   task.
3. Move into an **area** note that has no `## Tasks` section → the section is created and the cursor lands on the moved
   task under the new header.
4. `3<Ctrl+Shift+M>` → three tasks move; the cursor lands on the first of them.
5. Move into a project note whose frontmatter has no `task_count` yet (so `bob-project-tasks` adds frontmatter right
   after the write) → the cursor is still on the moved task, not two lines above.
6. Force a failure (e.g. pick a destination whose `## Tasks` section was removed) → the notice explains the failure and
   focus stays in the source note.

## Commit

Single commit, repo commit-message convention:

```
feat(navigation-hotkeys): focus destination note after task move
```

Body: describe the destination focus + moved-line jump, the block-ID re-anchoring rationale, and "Bump Bob Navigation
Hotkeys to 1.17.0." Commit through the `/sase_git_commit` skill.
