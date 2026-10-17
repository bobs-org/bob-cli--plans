---
tier: tale
title: Control-Return reuses any open tab, else appends at the far right
goal:
  Control-Return in Bob Mac Capture reuses a tab already showing each note, including
  unloaded background tabs, and otherwise appends new tabs at the far right of the
  active tab group instead of right after the focused tab.
size: small
decisions:
  empty_tab:
    ask:
      If the first note isn't open but the active tab is an empty New tab, fill that tab
      instead of appending one?
    choices:
      reuse:
        The note fills the blank tab in place; no stray blank tab, but it can land
        mid-strip
      end: The note always opens at the far right; the blank tab stays where it was
    default: reuse
    why:
      A blank tab holds nothing, and the usual one (all tabs closed) is already the far
      right
    answer: reuse
proposed_by: bbugyi200.apollo.8s
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 18:55:44
status: wip
---

# Plan: Control-Return reuses any open tab, else appends at the far right

Control-Return in Bob Mac Capture opens bob's `open_notes` through one
`obsidian://bob-open-notes` URL, handled by bob-navigation-hotkeys. The rule Bryan
wants: **if a tab already shows the note, use it; otherwise open a new tab at the end of
Obsidian's tab strip (far right).** Today's handler gets the first half mostly right and
the second half wrong.

## What is wrong today

Verified on bob-plugins `master` at `1b74d6c` (bob-navigation-hotkeys 2.44.1, the
bob-cli-8w landing).

1. **New tabs land right after the focused tab.**
   - In `plugins/bob-navigation-hotkeys/src/676-plugin-open-notes-uri.js`,
     `openBobOpenNotesEntry` creates the tab for a note that is not open with
     `this.createNewTabLeaf(workspace, workspace.activeLeaf)`, which is
     `workspace.getLeaf("tab")` (`630-plugin-vim-link-and-leaves.js`). Obsidian inserts
     that tab immediately after the active tab.
   - Each later new tab is then moved after the previous entry's leaf with
     `placeDuplicateTabAfterSource(previousEntryLeaf, leaf)`.
   - So the whole batch is inserted just after the tab you were on, not at the end.
   - The test harness already models Obsidian's after-active placement: its
     `getLeaf("tab")` splices the new leaf in after `workspace.activeLeaf`. But "three
     new tabs follow entry order in the tab strip" starts from a single open tab, so
     after-active and at-end look the same and the bug never shows.
2. **Reuse misses background tabs that Obsidian has not loaded yet.**
   - Reuse goes through `findLeafShowingPath` (`675-plugin-goto-task-uri.js`). It
     matches only `leaf.view.file.path`.
   - Since Obsidian 1.7.2 (the plugin's `minAppVersion` is 1.8.7), a background tab not
     shown since launch is _deferred_: `leaf.isDeferred` is true and, per the API docs,
     `leaf.view` is a `DeferredView` "instead of the View that it should normally have
     for its type (like MarkdownView)". A `DeferredView` is not a file view, so it has
     no `file`. The tab's file is only in `leaf.getViewState().state.file`.
   - Result: after an Obsidian restart, Control-Return very likely opens a **duplicate**
     tab for a note that is already open in a tab you have not clicked yet.
   - This can't be exercised off the Mac. The fix below is harmless if a deferred view
     turns out to expose `file`.
   - The same helper backs `findMarkdownLeafByPath`, which Go To, `bob-goto-task`, the
     dash jump, and link opening use, so they share the gap.

These already match the rule and stay unchanged:

- the first matching leaf in any window is reused;
- secondary notes load in the background at their line;
- the primary (entry 1) ends active and focused;
- landing, highlight, notices, freshness wait, and URL serialization.

## Target behavior

For each `bob-open-notes` entry, in entry order:

1. **Existing tab.** Reuse a leaf that already shows the note, in any window, including
   a deferred (not yet loaded) tab.
   - Main-area leaves (the root split or a popout window) win over sidebar leaves.
   - Within each of those two tiers, the first match wins, as today.
   - A reused tab never moves.
2. **Empty tab, entry 1 only.** Follow the `empty_tab` decision.

   > [!decision] empty_tab = reuse Keep today's rule: if entry 1 is not open anywhere
   > and the active leaf's view type is `empty`, open it in that empty tab where it
   > sits.

   > [!decision] empty_tab = end Delete the entry-1 empty-tab branch. A note that is not
   > open always gets a new tab appended at the far right, and the empty tab is left
   > untouched.

3. **Otherwise, append a new tab at the end of the active tab group.**
   - The active tab group is the parent of `workspace.getMostRecentLeaf()`. That is the
     main-area group you were last in (root split or popout window), never a sidebar,
     even when a sidebar leaf has focus.
   - Several new tabs therefore line up at the far right in entry order.
   - With side-by-side splits, "far right" means the end of the group you were in, not
     the rightmost split.

## Changes (bob-plugins only)

Open bob-plugins with your `/sase_repo` skill and read its `AGENTS.md`. Edit only the
`src/` fragments, never `main.js`, and keep every fragment at or below 1000 lines. Today
`676` is 615 lines, `675` is 453, and `630` is 974, so add nothing to `630`.

bob-cli and bob-mac-capture do not change. The URL contract stays the same, and the Mac
README's "needs bob-navigation-hotkeys ≥ 2.44.1" stays true.

### 1. Make `findLeafShowingPath` deferred-aware and main-area-first (`675-plugin-goto-task-uri.js`)

Rewrite `findLeafShowingPath(plugin, path, options)`, keeping its signature, the
`try`/`catch`, and its `null` results.

- **A loaded leaf** shows `path` when `leaf.view.file.path === path`. With
  `markdownOnly`, keep today's `plugin.isMarkdownFile(leaf.view.file)` test.
- **A deferred leaf** shows `path` when all of these hold:
  - `leaf.view.file` is absent;
  - `leaf.isDeferred === true`;
  - the leaf is in the main area;
  - `leaf.getViewState()` (guarded) has `state.file === path`.

  With `markdownOnly`, it also needs `viewState.type === "markdown"`. Requiring the main
  area keeps out sidebar info panes (backlinks, outline) whose saved state can name a
  file.

- **Sidebar leaves.** A leaf is in a sidebar when `leaf.getRoot()` returns
  `workspace.leftSplit` or `workspace.rightSplit`. Treat a leaf without `getRoot` as
  main area.
- **Order.** Scan every leaf with `iterateAllLeaves`. Return the first main-area match,
  else the first sidebar match.

Every caller already handles a deferred leaf:

- `openMarkdownFileWithLeafReuse`, the dash jump, the vim jump, and link opening all
  activate the found leaf through `activateWorkspaceLeaf`, which awaits
  `workspace.revealLeaf`, and that loads a deferred leaf.
- `ensureGoToTaskEditingLeaf` reads `getViewState()`.

So Go To and the other callers gain the same reuse. That is intended.

### 2. Append new tabs at the end (`676-plugin-open-notes-uri.js`)

1. **Add a mixin method `createBobOpenNotesTabAtEnd(workspace)`.**
   - **Anchor and group.** The anchor is `workspace.getMostRecentLeaf()` when that is a
     function and returns a leaf, else `workspace.activeLeaf`. The group is
     `anchor.parent` when that has an array `children`.
   - **Create.** When `workspace.createLeafInParent` is a function and a group exists,
     call `workspace.createLeafInParent(group, group.children.length)` inside a
     `try`/`catch`. Keep the leaf only if `group.children` now contains it. Otherwise
     `detach()` it (guarded) and fall back.
   - **Fallback.** Return
     `this.createNewTabLeaf(workspace, workspace.activeLeaf || null)`. That is
     Obsidian's default placement, so a runtime without the API still opens the note,
     just next to the active tab as before.
   - **Why this API.** Inserting through Obsidian's own `createLeafInParent` (public
     since 0.9.11) lets the tab group build the tab header at the right index. The
     alternative, moving the tab after `getLeaf("tab")` with
     `placeDuplicateTabAfterSource`, splices `parent.children` by hand.
   - The `.d.ts` types `parent` as `WorkspaceSplit`. On desktop a leaf's parent is a
     `WorkspaceTabs`, and that is what we pass. The containment check and the fallback
     cover a runtime that treats it differently.
2. **Update `openBobOpenNotesEntry`.**
   - Call the new method where it called `createNewTabLeaf`.
   - Delete the `previousEntryLeaf` parameter and the `placeDuplicateTabAfterSource`
     call.
   - In `handleBobOpenNotesUri`, delete the `previousEntryLeaf` bookkeeping. Appending
     each new tab in turn already gives entry order.
   - Leave `placeDuplicateTabAfterSource` in `630`, because `duplicateCurrentTab` still
     uses it.
3. **Load a reused deferred leaf first.** When the reused leaf has `isDeferred === true`
   and a `loadIfDeferred` function, `await leaf.loadIfDeferred()` (guarded) before
   reading `leaf.view`. Then the `currentPath` check sees the real file and the tab is
   not reopened from scratch. Primary landing still goes through `focusTaskAnchor`
   unchanged.
4. **Apply the `empty_tab` branch.** Under `reuse`, keep the entry-1 empty-tab block as
   is. Under `end`, delete it.

### 3. Tests (`scripts/test-navigation-open-notes-uri.cjs`)

**Extend `createBobOpenNotesHarness`.** Keep its `getLeaf("tab")` after-active model,
because the fallback test relies on it. Add:

- `workspace.getMostRecentLeaf()`: the active leaf when it is in the main area, else the
  main-area leaf that was active most recently;
- `workspace.createLeafInParent(parent, index)`: inserts a new harness leaf at `index`
  of `parent.children`;
- `leaf.getRoot()`, plus `workspace.leftSplit` and `workspace.rightSplit`, so a test can
  put leaves in a sidebar;
- deferred leaves: `isDeferred: true`, a view with no `file`, `getViewState()` returning
  `{ type, state: { file } }`, and a recorded `loadIfDeferred()` that turns the view
  into a loaded one;
- a way to place the active leaf in the middle of a tab strip.

**Add tests:**

1. **Middle-tab regression.** Start with strip `[a, b, c]` and `b` active. Entries
   `new1` and `new2` give strip `[a, b, c, new1, new2]` with `new1` active. This fails
   on 2.44.1, which gives `[a, b, new1, new2, c]`.
2. **A reused tab stays put while a new one is appended.** Start with `[a, x, c]` and
   `a` active. Entries `x` and `new` give `[a, x, c, new]`. `x` is reused with no
   `openFile` and ends active.
3. **A deferred tab is reused.** A deferred main-area leaf for `first.md` gets no new
   tab, `loadIfDeferred` was awaited, and the strip has no duplicate. Also call the real
   `NavigationHotkeysPlugin.prototype.findMarkdownLeafByPath`, not the harness override,
   and check it finds a deferred `markdown` leaf. Check it skips a deferred leaf whose
   saved type is not `markdown`.
4. **Sidebar rules.**
   - A deferred sidebar leaf whose saved state names the note is not reused.
   - A loaded main-area match beats an earlier loaded sidebar match.
5. **Active leaf in a sidebar.** The new tab goes to the end of the main group from
   `getMostRecentLeaf()`, never into the sidebar. Adapt "a non-file sidebar view is not
   reused as an empty tab" to put that leaf in a real sidebar root.
6. **Fallback.** Without `createLeafInParent`, or when it throws, the note still opens
   in a new tab through `getLeaf("tab")`.
7. **`empty_tab`.**
   - Under `reuse`, the existing empty-tab tests stay as they are.
   - Under `end`, rewrite "a true empty tab is reused for the first entry" and the
     empty-tab assertion in "new-tab policy reuses existing tabs…". They should expect
     the empty tab untouched and the note appended at the end.

Keep every other existing test passing. The "two tabs of the same note reuse the first
matching leaf" test must still pass.

### 4. Docs, release, and deploy

1. **README.** In "Open notes URI (`obsidian://bob-open-notes`)", replace "An existing
   tab showing a note is reused; otherwise the handler opens a new tab." with the target
   rules:
   - a tab already showing the note is reused, in any window, including a background tab
     Obsidian has not loaded yet, with the main area before sidebars;
   - the `empty_tab` sentence;
   - otherwise a new tab is appended at the far right of the active tab group, in entry
     order.
2. **Version.** Bump bob-navigation-hotkeys to **2.44.2** in its `manifest.json` and in
   the README plugin-table row.
3. **Build and test.** Run `npm run build`, `npm test`, and `npm run validate`.
   - The bob-cli-8w landing recorded seven `npm test` failures already on the clean
     base: bob-cli-6v (4), 6w, 7i, and 3w (a timing flake).
   - Compare against base before blaming this change, and don't fix those failures here.
   - `node --test scripts/test-navigation-open-notes-uri.cjs` and the Go To URI tests
     must pass in full.
4. **Deploy.** From the linked checkout, run
   `bob plugins sync --repo <checkout> -p bob-navigation-hotkeys --no-pull` (bob-cli
   `docs/plugins.md`).

## Rollout and manual check

Custom `bob-*` plugins are gitignored in the vault, so vault git sync never carries
them. The Mac gets 2.44.2 only when Bryan runs `just install-all` (which runs
`bob plugins sync`) or `bob plugins sync` there, then reloads Obsidian. Agents never run
`just install` on the Mac.

Manual checklist for Bryan:

- **Middle tab.** Open three tabs and focus the middle one. Control-Return a new inbox
  task. The inbox opens as a new tab at the far right and is focused.
- **Already open.** Same as above, but with the inbox already open in a tab. That tab is
  focused, and no tab is added.
- **Unloaded tab.** Restart Obsidian and don't click the inbox tab, then Control-Return.
  The existing inbox tab is reused.
- **Two notes.** Do a capture linked into a Pomodoro, with neither the inbox nor Today
  open. Two tabs are appended at the far right, inbox then Today, and the inbox is
  active.
- **Sidebar focus.** Focus the file explorer, then Control-Return. The tab still lands
  at the end of the main tab strip.

## Out of scope

- Go To and `bob-goto-task` placement when the note is not open: they still replace the
  active tab. They only gain deferred-tab reuse through the shared helper.
- Where Obsidian places tabs for its own commands.
