---
tier: tale
title:
  Ctrl+9 / Ctrl+0 move from an unpinned copy of the daily / dash note to its pinned tab
goal:
  Ctrl+9 and Ctrl+0 reach the pinned daily / dash tab even when the current tab is an
  unpinned copy of that note, including right after an Obsidian restart.
size: small
decisions:
  dup_daily_press:
    ask:
      Should one Ctrl+9 press from an unpinned daily copy also jump to the current
      Pomodoro on the pinned tab?
    choices:
      switch_only:
        Land on the pinned tab with its cursor kept; a second press jumps, as from any
        other note
      switch_and_jump:
        Land on the pinned tab and jump to the current Pomodoro line in the same press
    default: switch_only
    why:
      Keeps the existing rule that the first press opens the daily note and the second
      press jumps.
    answer: switch_only
  other_pomodoro_notes:
    ask:
      Should Ctrl+9 in another note with its own Pomodoros section (e.g. yesterday's
      daily) go to today's daily?
    choices:
      keep_in_place: Keep jumping inside that note; only copies of today's daily change
      go_to_today:
        Go to today's daily tab or open it; jump in place only on today's best daily tab
    default: keep_in_place
    why:
      The in-place jump is existing, tested behavior, and the bug report does not touch
      it.
    answer: keep_in_place
proposed_by: bbugyi200.apollo.9z.f0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-21 19:09:23
status: wip
---

# Plan: Ctrl+9 / Ctrl+0 move from an unpinned copy of the daily / dash note to its pinned tab

Fixes the gap left by bob-plugins commit ebfdc64, which taught both keys to find
deferred pinned tabs but kept their "act in place" shortcuts.

## Diagnosis

Bryan restarted Obsidian on the Mac and pressed Ctrl+9. It stayed in the current tab and
did not switch to the pinned daily tab. The screenshot shows this Mac tab bar: pinned
`20261021`, pinned `dash`, pinned `sase_blog_contents`, the **active, unpinned**
`20261021`, then `sase_pypi_releases`.

Evidence. It comes from read-only checks on the Mac over Tailscale SSH (files plus the
Obsidian CLI `eval`) and from reading Obsidian 1.14.4's `app.js`:

1. **Commit ebfdc64 was running.** The plugin files were deployed at 18:51, Obsidian
   started at 18:51:45, and the screenshot was taken at 18:52:14. The running app
   reports bob-ledger-tools 1.63.2 and has `openDailyFallbackFromCommand`.
2. **ebfdc64's deferred-tab lookup is correct for Obsidian 1.14.4.**
   - A deferred tab's `DeferredView` returns the saved type from `getViewType()` and the
     saved state from `getState()`.
   - So `leaf.getViewState()` returns `{ type: "markdown", state: { file }, pinned }`.
   - Today's daily path resolves to `2026/20261021.md` (daily-notes format
     `YYYY/YYYYMMDD`), which matches the pinned tab's saved state.
3. **Ctrl+9 did not navigate the copy tab.** That tab already showed today's daily
   before the press; it was a leftover copy, for example from the old bug before ebfdc64
   or from "Duplicate tab", and the restart restored it.
   - Obsidian's undo-close record for that tab has an empty back history. Navigating
     from another note would have added an entry.
   - When the active tab is pinned, Obsidian adds a new tab at the end of the tab group.
     This tab sits before `sase_pypi_releases`, so Obsidian did not just create it.
4. **Ctrl+9 jumped inside the copy.** The copy's cursor was at line 39, column 0, the
   open `(1800-1825)` Pomodoro, centered in the screenshot. That is exactly what
   `jumpToPomodoroTarget` plus `scheduleCenterOnLine` produce.
5. **The behavior reproduces on current master.** Use the repo's test harness. Make the
   active tab an unpinned, loaded copy of `Daily.md` and add a deferred, pinned
   `Daily.md` tab. The `jump-to-current-pomodoro` command callback then sets the cursor
   in the copy (`{ line: 3, ch: 0 }`) and never reveals the pinned tab.

**Root cause: the order of checks.** All paths below are in the **bob-plugins** linked
repo.

- The `jump-to-current-pomodoro` command calls `jumpToCurrentPomodoro(activeEditor)` in
  `plugins/bob-ledger-tools/src/300-plugin-vim-and-snippets.js`.
- That method first looks for a Pomodoro target in the **active** note. Every copy of
  today's daily has one, so it jumps in place and returns.
- The pinned-tab search that ebfdc64 fixed runs only when the active note has no target:
  `openDailyFallbackOnly` → `openTodayDailyNoteWithState` →
  `findOpenMarkdownLeafByPath`.
- Even then, `openDailyFallbackOnly` stops with a notice when the active note is today's
  daily.
- The previous plan deliberately kept both exceptions.

The restart matters only because it restored the stale copy; the bug also happens
without a restart.

**Ctrl+0 has the same gap.** `openDashTasks` in
`plugins/bob-navigation-hotkeys/src/620-plugin-dash-and-vim-jumps.js` returns early
whenever the active view is `dash.md`. That includes an unpinned copy while a pinned
`dash.md` tab exists. Bryan did not hit it only because no unpinned dash copy was open.

## Target behavior

**Ranking.** Rank every tab that shows the note as ebfdc64 does: pinned main-area tab
(0), other main-area tab (1), pinned sidebar tab (2), other sidebar tab (3). Add one new
tie-break: among matches of equal rank, the active leaf (`workspace.activeLeaf`) wins;
otherwise iteration order decides, as today. Call the best match the **home tab**.

**Ctrl+9 when the active note is today's daily:**

- If the active leaf is the home tab, nothing changes. Ctrl+9 jumps to the Pomodoro
  target in place, or shows the existing notice when there is none.
- Otherwise a better-ranked tab, such as the pinned daily, shows the note. Activate that
  home tab exactly as the from-another-note path does:
  - load it if deferred, reveal it, focus it;
  - keep its live cursor and viewport (`reusedOpenLeaf: true` semantics);
  - leave the copy's cursor alone;
  - do not run the daily-notes command or call `getLeaf`.

**Ctrl+9 when the active note is not today's daily** is governed by the
`other_pomodoro_notes` decision.

**Ctrl+0 when the active view is `dash.md`:**

- If the active leaf is the home tab, keep today's early return.
- Otherwise activate the home tab through the existing "already open in a tab" branch:
  no Tasks jump, no remembered-location restore, and the home tab's scroll is left
  untouched.

Never close, move, or edit the copy tab.

## Implementation (bob-plugins linked repo)

Open the repo with `sase repo open bob-plugins` and read its `AGENTS.md` first.

- Edit `src/` fragments, then run `npm run build`. Never hand-edit a generated
  `main.js`.
- Each hand-edited fragment must stay at or below 1000 lines.
  `bob-navigation-hotkeys/src/620-plugin-dash-and-vim-jumps.js` is at 949 now.

### bob-ledger-tools (Ctrl+9)

- `src/040-editor-and-daily.js`, `findOpenMarkdownLeafByPath(app, path)`:
  - read `workspace.activeLeaf` once, before iterating;
  - change the rank comparison to
    `rank < matchingRank || (rank === matchingRank && leaf === activeLeaf)`;
  - keep the name and signature, because the function is exported. The Today-mark jump
    (`275-plugin-today-marks.js`) and the `open-today-daily-note` command use it too;
    they only gain the tie-break.
- `src/300-plugin-vim-and-snippets.js`:
  - Add a small mixin method, for example `findDailyHomeLeafAwayFromActive()`.
    - It returns the home tab only when all of these hold: the active file is today's
      daily (`isTodayDailyFile(this.app, getActiveFile(this.app))`),
      `this.app.workspace.activeLeaf` exists, and
      `findOpenMarkdownLeafByPath(this.app, this.currentDailyPath())` returns a
      **different** leaf.
    - Otherwise it returns `null`. Because of the tie-break, a different leaf always
      means a strictly better rank.
  - In `jumpToCurrentPomodoro(cm)`, call it right after the `dailyNavigationActionId`
    increment and before `getEditorLines`. When it returns a leaf, return
    `this.openDailyFallbackOnly(null, actionId, { leavingDailyCopy: true })`. This
    covers the command callback and the Vim action alike.
  - Change the signature to `openDailyFallbackOnly(error, actionId, options = {})`. When
    `options.leavingDailyCopy` is true, skip the early `isTodayDailyFile` notice;
    everything else stays unchanged. `openTodayDailyNoteWithState` picks the same home
    tab and reports `reusedOpenLeaf: true`, so the existing code keeps the home tab's
    live location.
- `src/170-plugin-lifecycle.js` should need no change, because the command callback
  already goes through `jumpToCurrentPomodoro`.

> [!decision] dup_daily_press = switch_and_jump After `openDailyFallbackOnly` succeeds
> on the `leavingDailyCopy` path, and only when `actionId` is still current:
>
> - read the home tab's editor lines;
> - call `getJumpPomodoroTarget`;
> - if a target exists, run `this.jumpToPomodoroTarget(result.view.editor, target)`.
>
> If there is no target, skip the notice. Have `openDailyFallbackOnly` return the result
> view, or add a sibling method, whichever keeps the code simplest. With `switch_only`,
> add nothing; the second press jumps.

> [!decision] other_pomodoro_notes = go_to_today In `jumpToCurrentPomodoro`, when the
> active file is **not** today's daily, skip the active-note target lookup and return
> `this.openDailyFallbackOnly(null, actionId)` directly. Ctrl+9 then jumps in place only
> when the active leaf is today's daily home tab.
>
> - Update the existing test "an active-note Pomodoro target still jumps and centers
>   immediately": `Work.md` with a Pomodoros section now goes to the daily note instead.
> - Say so in the docs paragraph.
>
> With `keep_in_place`, add nothing.

### bob-navigation-hotkeys (Ctrl+0)

- `src/675-plugin-goto-task-uri.js`, `findLeafShowingPath`, **`preferPinned` branch
  only**: add the same active-leaf tie-break, reading `workspace.activeLeaf` once. Keep
  the non-`preferPinned` path byte-for-byte; Ctrl+Return, `bob-open-notes`, and
  `bob-goto-task` depend on it.
- `src/620-plugin-dash-and-vim-jumps.js`, `openDashTasks`, in the branch where
  `activeView.file.path === file.path`:
  - call `this.findMarkdownLeafByPath(file.path, { preferPinned: true })`;
  - keep the existing early return (refresh and capture without moving) when the result
    is `null`, when it is `workspace.activeLeaf`, or when there is no active leaf;
  - otherwise fall through to the existing `captureActiveFilePosition` and reuse path,
    which activates the home tab. Reuse the leaf you already found rather than looking
    it up twice if that stays simple.
  - If the fragment would exceed 1000 lines, put a small helper in `675` instead.

### Versions and docs

- Bump the patch versions:
  - `plugins/bob-ledger-tools/manifest.json`: 1.63.2 → 1.63.3;
  - `plugins/bob-navigation-hotkeys/manifest.json`: 2.50.3 → 2.50.4.

  Then run `npm run docs:gen` so `docs/index.md` and the generated facts follow.

- Extend the hand-written paragraphs that ebfdc64 added:
  - In `docs/bob-ledger-tools.md`, the `Ctrl+9` paragraph after "Session toggles".
  - In `docs/bob-navigation-hotkeys.md`, the `Ctrl+0` paragraph.

  Add one or two sentences to each: when the current tab is itself an unpinned (or
  lower-ranked) copy of the note, the key moves to the better tab instead of acting in
  place. Ctrl+9 jumps to the Pomodoro line only on the best-ranked daily tab; adjust the
  wording for the chosen decisions. Ctrl+0 leaves the home tab's scroll alone.

## Tests

### `scripts/test-ledger-tools-navigation.cjs`

Reuse `makeView`, `makeLeaf`, `TestWorkspace`, `makeApp`, `setupPlugin`, and
`POMODORO_NOTE`. Drive the key through the `jump-to-current-pomodoro` command callback
where noted, because that is the real Ctrl+9 path. Add these tests:

- **The regression.** The active leaf is an unpinned, loaded `Daily.md` copy with
  `POMODORO_NOTE`, and there is a **deferred pinned** `Daily.md` leaf. The command
  callback:
  - activates the pinned leaf, loads it once, and focuses its editor;
  - leaves `setCursorCalls` on the copy empty;
  - leaves `app.dailyCommandCalls === 0` and `workspace.getLeafCalls === 0`;
  - shows no notices.
  - With `switch_only`, the pinned editor has no `setCursorCalls`, and a second
    `jumpToCurrentPomodoro(pinnedView.editor)` jumps to `{ line: 3, ch: 0 }`. With
    `switch_and_jump`, the first press already sets `{ line: 3, ch: 0 }` on the pinned
    editor.
- **No Pomodoros section.** Same layout, but the copy has no `## Pomodoros` section. The
  press still moves to the pinned tab, and the "No ## Pomodoros section found" notice
  does not appear.
- **Pressing on the pinned tab.** The active leaf is the pinned daily, and an unpinned
  copy also exists. The press still jumps in place, and nothing else is activated.
- **Equal rank stays.** The active leaf is an unpinned daily, another unpinned daily
  comes **earlier** in iteration order, and there is no pinned daily. The press jumps in
  place in the active leaf; the tie-break wins.
- **Sidebar copy.** The active daily leaf is in the sidebar, and there is an unpinned
  main-area daily. The press moves to the main-area tab.
- **Any Ctrl+9 path, through `jumpToCurrentPomodoro(copy.editor)`.** The Vim action path
  also moves to the pinned tab.

Every existing test in the file must still pass. The one exception is the `go_to_today`
update described in that decision.

### `scripts/test-navigation-hotkeys-dash-open.cjs`

Use the existing `makeDashHarness`. Set `activeLeaf` to an unpinned `dash.md` leaf with
`harness.makeLeaf(harness.dashFile)`, or pass `activeFile` and use
`harness.workspace.activeLeaf`. Add these tests:

- **Unpinned dash copy.** The active leaf is an unpinned `dash.md` copy, and a
  **deferred pinned** `dash.md` tab exists. `openDashTasks()`:
  - activates the pinned tab, loads it once, reveals it, and calls `setActiveLeaf` with
    `{ focus: true }`;
  - makes no `openCalls`, no Tasks jump (`jumpCalls === 0`), and no restore.
- **Pinned dash active.** The active leaf is the pinned dash, and an unpinned copy
  exists. Nothing is activated or revealed.
- **Equal rank stays.** The active leaf is an unpinned dash, and another unpinned dash
  comes earlier in iteration order. Nothing is activated.
- The existing test "Ctrl+0 leaves an already active dash tab alone" must keep passing
  unchanged.

`scripts/test-navigation-open-notes-uri.cjs` and the goto-task tests must pass
unmodified. That proves the non-`preferPinned` lookups did not change.

## Verification

Run these in the bob-plugins checkout:

1. `npm run build`
2. `npm test`
3. `npm run validate`
4. `npm run docs:check`

`npm test` already had six failures on master during the ebfdc64 run. Compare failures
against `master` before and after the change, fix only failures this change introduces,
and report the rest.

Then deploy and commit:

- Run `bob plugins sync`. From a numbered linked checkout, a bare sync refuses; pass
  `--repo <the path printed by sase repo open bob-plugins>`.
- Commit (and push) the bob-plugins changes with the `sase_git_commit` workflow.

Obsidian does not run on this Linux host, and plugins reach the Mac only through
bob-plugins. So the final report must ask Bryan to:

1. Run `just install-all` from bob-cli on the Mac. It pulls bob-plugins, syncs the
   plugins, and restarts Obsidian.
2. Keep the pinned daily and pinned `dash.md` tabs. Open an unpinned copy of each, for
   example with the duplicate-tab command, then restart Obsidian.
3. Focus the unpinned daily copy and press Ctrl+9. It should land on the pinned daily
   tab with that tab's own cursor; a second press jumps to the current Pomodoro (or one
   press does both with `switch_and_jump`).
4. Focus the unpinned dash copy and press Ctrl+0. It should land on the pinned `dash`
   tab.
5. From an unrelated note, Ctrl+9 and Ctrl+0 should still reach the pinned tabs. On the
   pinned daily tab, Ctrl+9 should still jump in place.
