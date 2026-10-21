---
tier: tale
title: Ctrl+9 / Ctrl+0 reuse pinned daily and dash tabs even before they load
goal:
  Ctrl+9 and Ctrl+0 always land in the existing pinned daily / dash.md tab, even one
  never focused since startup.
size: medium
decisions:
  ctrl9_any_tab:
    ask:
      Should Ctrl+9 also open today's daily note when the active tab has no Markdown
      editor (empty tab, PDF, canvas)?
    choices:
      any_tab: Ctrl+9 finds the pinned daily tab from any tab, as Ctrl+0 already does
      editor_only: Ctrl+9 still shows "No active markdown editor" outside Markdown tabs
    default: any_tab
    why:
      Matches Ctrl+0, and a pinned daily tab helps most when the current tab is not a
      note.
    answer: any_tab
proposed_by: bbugyi200.apollo.9z
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-21 18:08:21
status: wip
---

# Plan: Ctrl+9 / Ctrl+0 always find pinned daily / dash tabs, even if Obsidian has not loaded them yet

## Problem

Ctrl+9 runs `bob-ledger-tools:jump-to-current-pomodoro`. Ctrl+0 runs
`bob-navigation-hotkeys:open-dash-tasks`. Both keys should bring Bryan back to the tab
he keeps pinned for today's daily note or `dash.md`. Today they only find that tab after
it has been focused at least once since Obsidian started. Until then they reuse the
current tab, or open a new one when the current tab is pinned.

Since Obsidian 1.7.2, every background tab restored at startup is **deferred**:
`leaf.isDeferred === true`, and `leaf.view` is a `DeferredView` placeholder rather than
a `MarkdownView`. So the placeholder has no `view.file`. The note path is only available
from `leaf.getViewState()` (`{ type: "markdown", state: { file }, pinned }`) until the
tab is revealed or `leaf.loadIfDeferred()` runs. The required Obsidian version is 1.8.7,
so these APIs always exist.

Root cause per key (all code is in the **bob-plugins** linked repo; open it with
`sase repo open bob-plugins`, and read its `AGENTS.md` first):

- **Ctrl+9 (definite bug).** `jumpToCurrentPomodoro`
  (`plugins/bob-ledger-tools/src/300-plugin-vim-and-snippets.js`) falls back to
  `openDailyFallbackOnly`, which calls `openTodayDailyNoteWithState`. That calls
  `findOpenMarkdownLeafByPath` (`plugins/bob-ledger-tools/src/040-editor-and-daily.js`),
  which only matches leaves for which `getMarkdownLeafFile(leaf)` returns a file, so it
  requires `leaf.view instanceof MarkdownView`. Deferred tabs never match. The code then
  runs the core `daily-notes` command or `workspace.getLeaf(false).openFile`, which uses
  the current tab or a new one. The Today-mark jump in
  `plugins/bob-ledger-tools/src/275-plugin-today-marks.js` (around line 884) uses the
  same lookup and has the same bug.
- **Ctrl+0 (fragile).** `openDashTasks`
  (`plugins/bob-navigation-hotkeys/src/620-plugin-dash-and-vim-jumps.js`) calls
  `findMarkdownLeafByPath`, which calls `findLeafShowingPath` / `leafShowsPath` in
  `plugins/bob-navigation-hotkeys/src/675-plugin-goto-task-uri.js`. Commit bca788d
  (2.44.2, Oct 17) taught that helper about deferred tabs as a side effect of the
  Ctrl+Return work, but:
  - no test covers Ctrl+0;
  - the view-state branch only runs when `leaf.isDeferred === true`;
  - a deferred tab in a sidebar is always rejected;
  - nothing prefers a pinned tab when the same note is open in more than one tab.

  Make Ctrl+0 use the same robust rules as Ctrl+9 and add tests for it.

## Target behavior (both keys)

When the key needs to show today's daily note (Ctrl+9 fallback) or `dash.md` (Ctrl+0),
search **every** workspace leaf in every window with `workspace.iterateAllLeaves`,
before the current tab or a new tab is considered.

1. **A leaf "shows" the path** in either of these cases:
   - its loaded view has `view.file.path` equal to the path (`sameVaultPath` in
     ledger-tools) and the view is a Markdown view;
   - its loaded view has no file, and `leaf.getViewState()` returns
     `type === "markdown"` and `state.file` equal to the path. Use this branch whether
     or not `isDeferred` reports `true`. Wrap `getViewState()` in `try`/`catch`.

   A loaded or deferred **non-Markdown** leaf never matches, even if its state names the
   path. Examples are backlink or outline sidebar views whose state carries `file`, and
   PDF views.

2. **Rank the matches.** Iteration order breaks ties.
   1. pinned main-area leaf
   2. other main-area leaf
   3. pinned sidebar leaf
   4. other sidebar leaf

   "Pinned" means `leaf.pinned` or `getViewState().pinned` is truthy. Ctrl+0 already has
   `isWorkspaceLeafPinned` for this. "Sidebar" means `leaf.getRoot()` is
   `workspace.leftSplit` or `workspace.rightSplit`; ledger-tools needs a local copy of
   nav's `leafIsInSidebar`.

3. **Activate the best match.**
   1. If `leaf.isDeferred === true` and `typeof leaf.loadIfDeferred === "function"`,
      `await leaf.loadIfDeferred()`. Catch errors; `revealLeaf` also loads the leaf.
   2. Reveal the leaf, then make it the active, focused leaf. Use the existing helpers,
      `activateMarkdownLeaf` and `activateWorkspaceLeaf`, which already
      `await workspace.revealLeaf(leaf)` and `setActiveLeaf(leaf, { focus: true })`.

   Treat this as an **already-open** tab. Ctrl+0 leaves its scroll alone and does not
   jump to Tasks. Ctrl+9 returns `reusedOpenLeaf: true`, so it records the location
   rather than restoring an older one. The second press still performs the Pomodoro
   jump.

4. **Fall back only when nothing matches.** The current path stays unchanged:
   - Ctrl+9 runs the daily-notes command, then calls `resolveOrCreateDailyFile` and
     `openVaultFile`.
   - Ctrl+0 calls `workspace.getLeaf(false).openFile(file)` and restores the remembered
     location or jumps to Tasks.

Unpinned tabs that show the note are still reused (rank 2/4); pinned tabs only win ties.
The Ctrl+0 early return when `dash.md` is already the active view stays. The Ctrl+9
behavior is also unchanged in two cases:

- when the active note itself has a `## Pomodoros` target, it jumps there;
- when the active note is today's daily note, it shows its existing notice.

## Implementation

### bob-ledger-tools (Ctrl+9, the `open-today-daily-note` command / `:bob_daily`, Today-mark jump)

Edit `src/` fragments only, then run `npm run build`. Never hand-edit `main.js`.

- `src/040-editor-and-daily.js`:
  - Add `leafIsInSidebar(workspace, leaf)`, the same logic as nav's.
  - Add `markdownLeafPathInfo(leaf)`, which returns `{ path, pinned }` or `null`. It
    implements rule 1, combining loaded `MarkdownView` file matching with the
    `getViewState()` fallback.
  - Rewrite `findOpenMarkdownLeafByPath(app, path)` to apply rules 1 and 2. It keeps its
    name and signature because it is exported via `350-exports.js` and used by
    `275-plugin-today-marks.js`.
  - In `activateMarkdownLeaf`, run the `loadIfDeferred` step from rule 3 before
    `revealLeaf`. The existing `waitForMarkdownLeafViewByPath` retry loop then gets the
    loaded `MarkdownView`. `getMarkdownLeafFile` and `getMarkdownLeafViewByPath` stay
    strict about `MarkdownView`, because callers need a live editor.
  - If the fragment would exceed 1000 lines, move the new leaf helpers into a new
    fragment listed right after `040-editor-and-daily.js` in `src/fragments.json`.
- Export any new helper from `src/350-exports.js` only if tests need it.

> [!decision] ctrl9_any_tab = any_tab In `src/170-plugin-lifecycle.js`, the
> `jump-to-current-pomodoro` callback currently shows "No active markdown editor" when
> `getActiveMarkdownView` returns null. Instead, add a small mixin method next to
> `openDailyFallbackOnly` in `src/300-plugin-vim-and-snippets.js` and call it from the
> callback.
>
> - The method increments `dailyNavigationActionId` exactly as `jumpToCurrentPomodoro`
>   does, then returns `this.openDailyFallbackOnly(null, actionId)`.
> - `openDailyFallbackOnly` already shows its own "Could not open daily note" notice on
>   failure.
> - The Vim action path (`bobLedgerJumpToCurrentPomodoro`) is unchanged.
>
> With `editor_only`, skip this callout entirely and keep the notice.

### bob-navigation-hotkeys (Ctrl+0)

- `src/675-plugin-goto-task-uri.js`, in `leafShowsPath`, when `markdownOnly` is true:
  - consult the view state whenever the loaded view has no file, without requiring
    `isDeferred === true`;
  - stop rejecting sidebar leaves in this branch, because the `type === "markdown"`
    check already excludes backlink/outline views.

  Keep the non-`markdownOnly` behavior byte-for-byte. Ctrl+Return / `bob-open-notes` and
  `bob-goto-task` depend on it, and the test "a deferred sidebar leaf whose saved state
  names the note is not reused" must still pass.

- Same file, `findLeafShowingPath`:
  - Add an opt-in `preferPinned` option that applies the rank from rule 2. Use
    `plugin.isWorkspaceLeafPinned(leaf)` when available; otherwise read `leaf.pinned` /
    `getViewState().pinned`.
  - Without the option, keep today's order: first main-area match, then first sidebar
    match.
- `src/620-plugin-dash-and-vim-jumps.js`:
  - Let `findMarkdownLeafByPath(filePath, options)` pass `options` through.
  - `openDashTasks` calls it with `{ preferPinned: true }`.
  - In `activateWorkspaceLeaf`, add the guarded `loadIfDeferred` step from rule 3 before
    `revealLeaf`. It is a no-op for loaded leaves, so the vim-jump, link, and review
    callers are unaffected. The open-notes test's `loadIfDeferredCalls` assertion stays
    exact because it is guarded by `isDeferred === true`.
  - Keep the fragment at or below 1000 lines; it is at 933 now.

### Versions, docs, deploy

- Bump the patch versions in `plugins/bob-ledger-tools/manifest.json` (1.63.1 → 1.63.2)
  and `plugins/bob-navigation-hotkeys/manifest.json` (2.50.2 → 2.50.3). Run
  `npm run docs:gen` so `docs/index.md` and the generated facts blocks follow.
- Add one short hand-written paragraph outside the GENERATED blocks in each of these
  docs:
  - `docs/bob-ledger-tools.md`, near the "Session toggles" mention of `Ctrl+9`;
  - `docs/bob-navigation-hotkeys.md`, next to the Ctrl+0 / `open-dash-tasks` discussion.

  Each paragraph says that the key reuses a tab already showing the note in any window
  before it uses the current tab or opens a new one, including a background tab Obsidian
  has not loaded yet. Pinned tabs win, then other main-area tabs, then sidebar tabs.
  Follow the wording style of `docs/open-notes-uri.md` ("An already open note is reused
  in any window, including a background tab Obsidian has not loaded yet…"). The bob-cli
  project's `docs` memory note covers bob-cli's own `docs/`; it does not need changes.

## Tests

The tests model Obsidian's deferred leaf exactly as
`scripts/test-navigation-open-notes-uri.cjs` already does:

- `view` is a plain object that is not a `MarkdownView` and has no `file`;
- `isDeferred: true`;
- `getViewState()` returns `{ type: "markdown", state: { file }, pinned }`;
- `loadIfDeferred()` swaps in the real view;
- `getRoot()` returns `leftSplit`, `rightSplit`, or the root split;
- the workspace's `revealLeaf` also loads a deferred leaf.

- `scripts/test-ledger-tools-navigation.cjs`: extend the existing `TestWorkspace` /
  `makeLeaf` helpers with the deferred, pinned, and sidebar fields above. Add tests:
  - Ctrl+9 from `Work.md` with a **deferred pinned** daily tab activates that tab, loads
    it, and focuses its editor. The daily-notes command count stays 0, nothing opens
    through `getLeaf`, and the second press performs the Pomodoro jump.
  - With the daily note open in both a pinned and an unpinned tab, the pinned one is
    activated, whichever comes first in iteration order.
  - A deferred **sidebar** daily tab is used only when no main-area tab shows the note.
    A main-area tab, deferred or loaded, wins over it.
  - A deferred non-Markdown leaf whose saved state names the daily path is ignored, and
    the existing fallback opens the note.
  - The standalone `open-today-daily-note` command reuses a deferred pinned daily tab.
  - Only for the `any_tab` decision: invoking the `jump-to-current-pomodoro` command
    while the active leaf is not a Markdown view activates the pinned daily tab instead
    of showing "No active markdown editor".
  - Every existing test in the file still passes unchanged.
- New `scripts/test-navigation-hotkeys-dash-open.cjs`, added to the `test` script list
  in `package.json`. Build the harness the same way as the open-notes test's `makeLeaf`,
  or reuse `scripts/navigation-hotkeys-harness.cjs` if that is simpler. Add tests:
  - Ctrl+0 from another note with a **deferred pinned** `dash.md` tab activates that tab
    and does not call `getLeaf(false).openFile`. It is treated as already open, so no
    Tasks jump and no remembered-location restore are scheduled.
  - A pinned `dash.md` tab beats an earlier unpinned one.
  - A deferred sidebar `dash.md` tab is used only when there is no main-area match.
  - With no matching tab, the old fallback opens `dash.md` in `getLeaf(false)`.
  - When `dash.md` is already active, the command returns without activating anything.
- `scripts/test-navigation-open-notes-uri.cjs` must pass unmodified, which proves
  non-`markdownOnly` lookups did not change.

## Verification

In the bob-plugins checkout:

1. `npm run build`
2. `npm test`, which includes `build:check` and the docs inventory check
3. `npm run validate`
4. `npm run docs:check`

`npm test` may already show unrelated failures, as noted in bead bob-cli-am.6. Compare
any failures against `master` before and after the change, and report them; do not fix
unrelated ones.

Then run `bob plugins sync`, as bob-plugins' `AGENTS.md` requires, and commit the
bob-plugins changes.

Obsidian does not run on this Linux host, so the final report must ask Bryan to check
the change on the Mac:

1. Restart Obsidian with the pinned daily and `dash.md` tabs left unfocused.
2. From another note, press Ctrl+9 and then Ctrl+0.
3. Confirm each key lands in the existing pinned tab and opens no new tab.
