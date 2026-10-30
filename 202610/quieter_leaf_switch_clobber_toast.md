---
tier: tale
title: Quieter leaf-switch clobber toast
goal:
  The note-switch clobber toast fires only on a real clobber and disappears on its own
  after a few seconds.
size: small
decisions:
  duration:
    ask:
      How long should the clobber warning stay on screen before it disappears on its
      own?
    choices:
      eight:
        8 seconds, the same as the plugin's other long notice (file rename), so the long
        message can be read
      five:
        5 seconds; gone sooner, but the ~170-character message may vanish before it is
        read
    default: eight
    why:
      the message is long, and 8000 ms is this plugin's existing duration for long
      notices
    answer: eight
  repeat_window:
    ask:
      Should the plugin also hide repeats of the same warning for one target/source pair
      for 60 seconds?
    choices:
      none:
        Every warning shows; after the landing fix, only a real clobber signature warns
      collapse:
        Each pair warns at most once a minute; a repeat clobber in that minute relies on
        vault-sync
    default: none
    why: after the fix, every warning signals real risk, so each one should be seen
    answer: none
proposed_by: bbugyi200.apollo.bf
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 14:30:36
status: wip
---

# Plan: Stop false clobber toasts and auto-dismiss the real ones

## Goal

The Obsidian toast
`bob: <target> opened with <source>'s text — it may have been overwritten while switching notes. Walk stopped; see vault-git-sync docs, Clobbered notes`
should appear only when a pane really swapped to the target note while still holding the
previous note's text. It should also disappear on its own after a few seconds instead of
staying until it is clicked.

## Background (verified)

The toast comes from `bob-plugins`, in
`plugins/bob-navigation-hotkeys/src/475-leaf-switch-guard.js`. It is the landing
tripwire added by `a182635` ("guard same-pane note switches", epic `bob-cli-do`, phase
`walk-save-guard`). `openWithLeafSwitchGuard(owner, targetFile, leaf, openAction)` does
three things:

1. Saves and verifies the note in `leaf`.
2. Runs `openAction`.
3. Calls `checkLeafSwitchLanding`. It warns with a **sticky** Notice
   (`new Notice(msg, 0)`) and returns `false` when
   `leaf.view.editor.getValue() === sourceText` and `targetDiskText !== sourceText`. It
   also schedules a 2.5 s disk re-check.

The toast in Bryan's 2026-10-30 13:47 screenshot was a **false positive**:

- It said `sase.md opened with 2026/20261030.md's text`.
- The vault's git history has no `sase.md` change between 13:03 and well after 13:47.
- `_conflicts/sync_conflicts.md` has no entry since 2026-10-23, and vault-sync reports
  no `clobbers` / `recent_clobbers`.
- The screenshot shows every tab pinned. Bryan pins tabs on purpose with the plugin's
  Vim tab-pin action, and the pinned daily note was still the visible note.

**Root cause.** The immediate check reads the guarded `leaf`'s editor without first
checking that this leaf actually switched to the target. Obsidian 1.14.5's bundled
`app.js` (inspected from the Mac's `obsidian-1.14.5.asar`) shows two ways the guarded
leaf can keep showing the source note after `openAction` resolves:

- **Link opens go to another tab.**
  - `Workspace.openLinkText(link, src, newLeaf, state)` is
    `this.getLeaf(newLeaf).openLinkText(...)`.
  - `getLeaf(false)` is `getUnpinnedLeaf()`. That skips an active leaf whose
    `canNavigate()` (`view.navigation && !pinned`) is false. It picks another navigable
    leaf or creates a new tab, and makes it active.
  - Three callers guard `workspace.activeLeaf` but navigate with
    `workspace.openLinkText(...)`:
    - `680-plugin-link-parse.js` `openResolvedLink` (Vim link follow).
    - `285-unblocked-notice.js` `openUnblockedRowTarget`.
    - `427-relation-stage-plugin.js` (relation mention open).
  - From a pinned tab, the note opens in a different leaf. The guarded pinned leaf still
    holds the source text, so `landedText === sourceText` warns every time. It also
    returns `false`, so callers skip their follow-up steps.
- **Dropped opens.** `WorkspaceLeaf.setViewState` begins with
  `if (this.working) return;`. An `openFile` / `setViewState` issued while that leaf is
  already mid-switch resolves without navigating. The guarded leaf still shows the
  source note, and the same false warning fires.

A real clobber always looks different: the guarded leaf's `view.file` **is** the target
while its text is the source's. That is how the original `sase_agent_recovery.md`
incident looked. So requiring `leaf.view.file.path === targetPath` removes both false
paths and keeps every true positive. The 2.5 s disk re-check compares target disk bytes
to the saved source text, so it only fires on a real overwrite. Keep it as is, except
for the duration.

Vault-sync's clobber quarantine (bob-cli `docs/vault-git-sync.md`, "Clobbered notes")
stays the backstop. It restores the note, keeps a conflict copy, and fires a desktop
notification. That is why the plugin warning does not need to be sticky.

## Changes

All work is in the **bob-plugins** linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<reason>"`) and read its `AGENTS.md`.

- `bob-navigation-hotkeys` uses the fragment build. Edit `src/` fragments only, run
  `npm run build`, and never hand-edit `main.js`.
- Fragments share one script scope, so grep for name collisions before adding a helper
  or constant.

### 1. Landing check only for a leaf that actually switched (`475-leaf-switch-guard.js`)

- Add a small helper, for example `leafSwitchViewPath(leaf)`. It returns
  `leaf.view.file.path` when that is a string, else `null`, and never throws.
- In `checkLeafSwitchLanding`, before reading `landedText`, check whether the guarded
  `leaf` now shows the target: `leafSwitchViewPath(leaf) === targetPath`, where
  `targetPath` is `targetFile.path`.
  - If it does not, skip the immediate editor comparison. This covers a link opened in
    another tab and a dropped `setViewState`.
  - Do not look up or create other leaves. In particular, never call
    `workspace.getLeaf(...)` here, because `getLeaf(false)` creates a tab and changes
    the active leaf.
- Keep the existing comparison
  (`targetDiskText !== sourceText && landedText === sourceText`) for a leaf that did
  switch.
- In every non-warning case, keep scheduling the 2.5 s disk re-check and return `true`.
  That matches the pre-guard behavior of these callers when the open happened elsewhere
  or was dropped.

### 2. Auto-dismiss the warning

- Replace the sticky flag in `leafSwitchNotice(message, sticky = false)` with an
  optional duration, for example `leafSwitchNotice(message, durationMs)`, which passes
  it to `new Notice(message, durationMs)`.
  - Leave `undefined` (Obsidian's default) for the existing "Could not save/read …
    before switching notes; staying here" notices.
- Add a named constant, for example `LEAF_SWITCH_CLOBBER_NOTICE_MS`, and use it for both
  the immediate and the delayed clobber warning. Nothing may pass `0` any more.

> [!decision] duration = eight
>
> Set the constant to `8000`, matching `670-plugin-project-files.js`'s rename notice.

> [!decision] duration = five
>
> Set the constant to `5000`.

> [!decision] repeat_window = collapse
>
> - Add a module-level `Map` keyed by `` `${sourcePath}\u0000${targetPath}` `` that
>   stores the last warning time.
> - Before showing a clobber warning (immediate or delayed), skip the Notice when the
>   same key warned less than 60 s ago, using `Date.now()`.
> - Still return `false` from the immediate check either way. Suppression only hides the
>   toast; it never changes navigation.
> - Prune expired keys on insert so the map stays small.

> [!decision] repeat_window = none
>
> No extra state: every detected clobber signature shows its own timed warning.

Keep the message text unchanged.

### 3. Tests (`scripts/test-navigation-leaf-switch-guard.cjs`)

Use the existing `navigation-hotkeys-harness.cjs` (`helpers`, `notices`,
`noticeDurations`). The fixture's `openFile` already swaps `leaf.view` to the target.

- Change the two `assert.equal(noticeDurations[0], 0, …)` assertions (immediate and
  delayed warning) to the chosen duration. Drop the "the warning is sticky" message.
- **Pinned/other-tab open, new test.** Call
  `helpers.openWithLeafSwitchGuard(plugin, targetFile, leaf, openAction)`. Its
  `openAction` "opens" the target in a _different_ leaf and makes it
  `workspace.activeLeaf`, while the guarded leaf keeps `view.file === sourceFile` and
  the source editor text. Expect:
  - `true`, and no notice.
  - The source still saved first.
  - A delayed re-check scheduled.
- **Dropped open, new test.** `openAction` resolves without changing anything, so the
  guarded leaf still shows the source note and text. Expect `true` and no notice.
- **True positive, still passing.** The existing "landing with the source text" test,
  where `view.file` is the target and the editor holds the source text, must still warn
  and return `false`.
- **Delayed true positive, still passing.** The existing delayed-tripwire test must
  still warn when target disk text later equals the source text.
- If `repeat_window = collapse`, add one test: two clobber landings for the same pair
  within 60 s produce one notice, and both calls return `false`.
  - Isolate the dedupe map between tests, for example by exporting a reset helper
    through `700-exports.js`, or by stubbing `Date.now`.

Run the review-advance and unblocked-notice suites too. They drive guarded landings
(`scripts/test-navigation-review-advance.cjs` with `trackLeafSwitch: true`, and
`scripts/test-navigation-hotkeys-unblocked-notice.cjs`), and both must still pass
unchanged.

### 4. Docs, version, build

- `docs/bob-navigation-hotkeys.md`, "Review walk": rewrite the paragraph that starts
  "Before a same-pane note switch". It should say:
  - The landing warning fires only when the pane switched to the new note but shows the
    previous note's text.
  - It stops the walk and disappears after the chosen duration (not "sticky").
  - The disk re-check after Obsidian's save debounce still runs.
  - A switch Obsidian opens in another tab is not checked on landing, for example
    following a link from a pinned tab.
  - Keep the "Clobbered notes" pointer.
- Bump `plugins/bob-navigation-hotkeys/manifest.json` by one patch version from its
  current value (2.62.0 → 2.62.1 at plan time). Then run `npm run docs:gen` so the
  generated inventory in `docs/index.md` matches.
- Run `npm run build`, `npm test` (includes `build:check` and the inventory check), and
  `npm run docs:check` (against the resolved bob-cli checkout).

### 5. Deploy

- Run `bob plugins sync` on this host, as the bob-plugins `AGENTS.md` requires.
- The MacBook deploys from its own checkout once the commit is on origin. Best effort:
  run `ssh mac 'bob plugins sync'` after the bob-plugins commit is pushed. The Mac is
  offline unless its lid is open; if it is unreachable, report it as Bryan's remaining
  step.

## Non-goals

- Changing which leaf the three `openLinkText` callers guard. For example, guarding the
  leaf `getLeaf(false)` would pick would protect that leaf's own unsaved text. But
  `getLeaf(false)` has side effects (it activates or creates a tab before the save check
  could abort), and that is a separate behavior change. The landing fix above is enough
  to stop the false toasts.
- The "Could not save … before switching notes; staying here" notices, which already
  auto-dismiss, and the vault-sync quarantine notification in bob-cli.
- Any SASE memory or decision-record change. No memory note describes this toast.

## Acceptance

- Following a Task Link from a pinned tab (Vim link follow, Unblocked-card row, relation
  mention) opens the note with no clobber toast.
- A real clobber signature still warns and stops the walk. The warning disappears on its
  own after the chosen duration.
- `npm test` and `npm run docs:check` pass, the manifest version is bumped, the
  inventory is regenerated, and `bob plugins sync` succeeded on this host (Mac deploy
  done or reported).
