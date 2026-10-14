---
tier: tale
title: Focus the linked note before the Task Link Depends on stage
goal:
  Ctrl+Shift+P then Depends on from a Pomodoro Task Link always opens the linked task's
  Depends on stage, even when that note is already open in a background tab, instead of
  failing with "No active markdown note".
size: small
proposed_by: bbugyi200.athena.10d
create_time: 2026-10-14 10:05:55
status: wip
---

# Plan: Fix "No active markdown note" from the Task Link Depends on handoff

## Problem

On a dedicated Pomodoro Task Link bullet in today's daily note (for example
`- [[job_2026_apply_20#^ref-second-wave-job-applications]]`), Ctrl+Shift+P opens the
Task Card in Task Link mode. Picking **Depends on** (`b`) shows the Obsidian notice
**"No active markdown note"**, and no Depends on stage opens.

It fails only when the linked note is already open in a tab that is not the active one.
In Bryan's screenshot, a `job_2026_apply_20` tab sits behind the pinned `20261014` tab.
The first attempt on a note that is not open works. Once that note is left in a
background tab, every later attempt fails.

## Root cause

All paths below are in the `bob-plugins` linked repo, under
`plugins/bob-navigation-hotkeys/src/`. Open it with your `/sase_repo` skill
(`sase repo open bob-plugins -r "<why>"`).

1. `360-picker-schedule-review.js` (around lines 219-246): in a link session, the
   Depends on row (`propertyItem.linkDependency`) closes the card and calls
   `this.plugin.openLinkedDependencyStage(target)`.
2. `590-plugin-dependency-stage.js` `openLinkedDependencyStage` (lines 801-852) first
   calls `this.getOpenMarkdownEditorForPath(path)`, defined in
   `500-plugin-transclusion-and-link-picker.js` (lines 45-60). That helper returns the
   editor of any markdown leaf showing the note, including a background tab, and does
   not activate the leaf. Only when no leaf shows the note does the function go through
   `openMarkdownFileWithLeafReuse`, which focuses or opens a leaf.
3. The function then moves the cursor in that background editor and calls
   `this.openBulletPropertyPicker(editor, { initialProperty: "dependsOn" })`.
4. `550-plugin-cancel-and-counted-property.js` `openBulletPropertyPicker` (around lines
   391-395) checks
   `activeView = this.getActiveMarkdownView(); if (!activeView || activeView.editor !== cm || !activeView.file)`
   and shows `new Notice("No active markdown note")`. The active view is still the daily
   note and `cm` is the background note's editor, so the check fails.

This bug has been there since the Task Link Depends on handoff shipped (1.54.0). The
installed vault build (`~/bob/.obsidian/plugins/bob-navigation-hotkeys/main.js`) has the
same code.

## Fix

Rewrite `openLinkedDependencyStage` in `590-plugin-dependency-stage.js` so it always
focuses the linked note's leaf before it opens the stage:

- Keep the early refusals as they are: an empty path gives `Linked task has no note`,
  and a missing file from `vault.getAbstractFileByPath(path)` gives
  `Linked note not found: ${path}`.
- Always call `await this.openMarkdownFileWithLeafReuse(file, \`Could not open
  ${path}\`)`and return`false`if it does. That is the plugin method in`620-plugin-dash-and-vim-jumps.js`(line 110), which wraps the shared helper in`480-review-jump-and-nav-api.js`
  (line 862). The helper does one of three things:
  - returns at once if the active view already shows the note, which covers a Task Link
    into the same note;
  - activates an existing leaf through `findMarkdownLeafByPath` and
    `activateWorkspaceLeaf` (`revealLeaf` plus `setActiveLeaf` with focus);
  - otherwise opens the note with `getLeaf(false).openFile`.

  Call the method rather than the free function. Other callers in `650`, `670`, and
  `680` already do, and tests can stub the method.

- Then read `this.getActiveMarkdownView()`. Require a view with `editor` and `file`
  whose `normalizeVaultRelativePath(view.file.path) === path`. If that fails, show
  `Could not open ${path}` and return `false`, so the misleading guard notice can no
  longer appear.
- Put the cursor at `target.line` in `view.editor`, keeping the current
  `Number.isFinite` handling and `setEditorCursorSafely`. Then
  `return this.openBulletPropertyPicker(view.editor, { initialProperty: "dependsOn" })`.
- Remove the `getOpenMarkdownEditorForPath` shortcut from this function only. Leave
  `getOpenMarkdownEditorForPath` itself unchanged: its other callers (`218`, `500`)
  write through background buffers on purpose.
- Keep the outer `try/catch` and its `Could not open the linked task` notice.
- Change the leading comment to say the note's tab is focused first, even when it is
  already open in a background tab, because the stage needs the active view's editor.

Leave the active-view check in `openBulletPropertyPicker` alone. Loosening it was
considered and rejected: the stage and its writers (`filePath`,
`reopenDependencyStageFresh`, review-walk capture, `applyDependencyEdit`) all assume the
editor belongs to the active view, so they would act on a hidden buffer. Focusing the
note also matches what already happens when the note is not open.

**Fragment cap:** `590-plugin-dependency-stage.js` is at 997 lines, and
`npm run build:check` refuses any fragment over 1000 (`MAX_FRAGMENT_LINES`). Removing
the branch should leave the rewrite about the same length. If it grows, tighten the
function. Do not move unrelated code.

## Tests

Add two tests to `scripts/test-navigation-dependencies-stage-entry-and-view.cjs`. That
file already runs in `npm test`, so `package.json` does not change. Build the plugin
with `stubPlugin(...)`, and use `TransactionEditor` and `notices` from
`navigation-dependencies-stage-harness.cjs`. Check notices against a `before` index
(`notices.slice(before)`), as the existing tests do.

1. **"Task Link Depends on focuses a linked note open in a background tab"**
   - Set up a daily editor and a `job.md` editor (task on line 1).
   - Stub `plugin.getActiveMarkdownView` to return a mutable active view that starts on
     `daily.md`.
   - Stub `plugin.getOpenMarkdownEditorForPath` to return the job editor for `job.md`,
     as the real helper does for a background tab.
   - Stub `plugin.openMarkdownFileWithLeafReuse` to record the call and switch the
     active view to `job.md`.
   - Stub `plugin.openBulletPropertyPicker` to record `(editor, options)` and whether
     `plugin.getActiveMarkdownView().editor === editor` at the time of the call.
   - Run `await plugin.openLinkedDependencyStage({ path: "job.md", line: 1 })`.
   - Assert that the open was called once for `job.md`, and that the picker got the job
     editor with `{ initialProperty: "dependsOn" }` and the active-editor match `true`.
   - Assert that the job editor's cursor is on line 1 and that no new notice is
     `No active markdown note`.
   - Before applying the fix, confirm this test fails against the current code: the
     picker receives the background editor while the daily note is still active.
2. **"Task Link Depends on refuses clearly when the linked note cannot be focused"**
   - (a) `openMarkdownFileWithLeafReuse` resolves `true`, but the active view stays on
     `daily.md`. Assert the notice is `Could not open job.md`, the picker is not called,
     and the result is `false`.
   - (b) `openMarkdownFileWithLeafReuse` resolves `false`. Assert the picker is not
     called and the result is `false`.

## Release bookkeeping

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `2.38.0` to `2.38.1`. Fixes
  get a patch bump, as `task-status-cycler` did going from `1.40.0` to `1.40.1`. Leave
  the description as it is.
- Change the Bob Navigation Hotkeys version in the plugin table in `bob-plugins`
  `README.md` from `2.38.0` to `2.38.1`.
- Run `npm run build` to regenerate `main.js`. Never hand-edit `main.js`.
- In this repo (bob-cli), edit the "A dedicated Task Link" row of the §6.1 entry-points
  table in `docs/task-dependencies.md`. It should say the linked note's tab is focused,
  and opened if needed, before the stage opens on the linked task, which is named in the
  title. Change only that row.

## Validation

- In bob-plugins, run `npm run build` and `npm run build:check`, then
  `node --test scripts/test-navigation-dependencies-stage-entry-and-view.cjs scripts/test-navigation-hotkeys-link-picker.cjs`,
  then the full `npm test` and `npm run validate`.
- Run `bob plugins sync` to deploy to the vault, as the bob-plugins `AGENTS.md`
  requires.
- Manual check for Bryan: make today's daily note the active tab with the linked note
  open in another tab. Press Ctrl+Shift+P on a Pomodoro Task Link, then `b`. The linked
  note's tab should come forward and the Depends on stage should open on the linked
  task.

## Out of scope

- Background tabs that Obsidian has not loaded yet after a restart.
  `findMarkdownLeafByPath` matches on `leaf.view.file`, so it may miss such a tab and
  open a second tab for the note instead. That works without an error, and this fix does
  not change it.
- Moving focus back to the daily note after the stage closes.
