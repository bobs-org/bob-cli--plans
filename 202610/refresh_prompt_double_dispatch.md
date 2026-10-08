---
tier: tale
title: Stop Ctrl+Alt+F from leaving a duplicate Work Log prompt open
goal:
  One Alt+F / Ctrl+Alt+F press in Vim normal mode opens exactly one Refresh Work Log
  prompt, and answering it leaves no prompt behind after the walk jumps.
size: small
decisions:
  fix_alt_n:
    ask: Also guard the Alt+N lane-toggle fallback against the same double dispatch?
    default: true
    why:
      Same root cause; off-landing In Progress releases can stack two Release task
      prompts.
    answer: true
proposed_by: bbugyi200.apollo.5u
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.5u](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.5u.md)
- **COMMITS:**
  - [c837be9](https://github.com/bobs-org/bob-plugins/commit/c837be9ce9fdd58fa0a43adf2302473a724580fd)
    — fix(nav-hotkeys): skip already-handled refresh and lane-toggle keydowns

# Fix the duplicate Work Log prompt left open after Ctrl+Alt+F (double-dispatched refresh)

## Problem

On a Pending (`[/]`) task in the GTD morning review walk, Bryan presses `Ctrl+Alt+F`
(refresh and advance). He answers the "Refresh task" Work Log prompt, and the walk jumps
to the next task. But a "Refresh task" prompt (`FreshnessRefreshSummaryModal`) is still
open on top of the new landing. The README already promises the opposite: "Alt+F and
Ctrl+Alt+F ask **once** for an optional Work Log summary".

## Root cause (diagnosed; verified against the live Mac read-only)

**One physical `Ctrl+Alt+F` in Vim normal mode runs `refreshTaskFreshness` twice. Each
run opens its own prompt, so two prompts end up stacked.**

1. **Obsidian's hotkey runs first.** The Mac runs Obsidian 1.14.4. Its `Keymap` adds
   `window.addEventListener("keydown", onKeyEvent, true)` at app init, before any plugin
   loads.
   - `onKeyEvent` walks workspace scope → view scope → root scope. The root scope's
     catch-all is `hotkeyManager.onTrigger`.
   - `onTrigger` matches the baked hotkey
     `bob-navigation-hotkeys:refresh-task-freshness-and-advance` (`Alt,Ctrl` + `F`) by
     `vkey`, whatever the Vim mode, and runs the command. That calls
     `refreshTaskFreshness(editor, { advance: true })` (`src/490-plugin-lifecycle.js`).
   - It then calls `event.preventDefault()` and `event.stopPropagation()`. It does
     **not** call `stopImmediatePropagation()`.
2. **The plugin's fallback then runs the same gesture again.** nav's Vim-normal
   capture-phase fallback is `registerReviewRefreshInputListeners` /
   `handleReviewRefreshPhysicalKeydown` (`src/540-plugin-decay-picker-and-cancel.js`).
   - It listens on the same `window` in the same capture phase but was registered later,
     so it still receives the event.
   - Its `handledReviewRefreshEvents` WeakSet only dedupes its own window and document
     registrations.
   - It sees Vim normal mode and calls `refreshTaskFreshness` a second time.
   - Its comments (540:240-245, and `isReviewRefreshKeydown` at 480:824-828) still say
     "CodeMirror Vim swallows Alt chords before Obsidian's hotkey dispatcher runs". That
     is not true on 1.14.4: Obsidian's window-capture listener runs before CodeMirror
     ever sees the key.
3. **Why only Pending tasks show it.**
   - **Non-Pending row:** the first dispatch stamps synchronously and reaches
     `advanceReviewWalkAfterAnswer` (`src/536-plugin-review-advance.js:496-498`). That
     takes the review-walk lock synchronously, so the duplicate dispatch is swallowed by
     the `reviewWalkBusy()` check at the top of `refreshTaskFreshness`.
   - **Pending row:** the first dispatch parks on
     `await requestFreshnessRefreshSummary(...)`
     (`src/520-plugin-lane-links-and-review.js:648-664`; the Task Link route is
     `src/530-plugin-freshness-refresh-and-decay.js:127-160`). The prompt opens
     synchronously inside the keydown, and no lock is taken. The duplicate passes
     `reviewWalkBusy()` and opens a second prompt on top.
   - Bryan answers the top prompt. Its dispatch stamps, logs, and jumps. The bottom
     prompt is left open over the next task. If he then answers it, its stale guard
     refuses ("Current note changed; no tasks were updated").
   - Plain `Alt+F` on a Pending task stacks two prompts the same way; there is just no
     jump to make it obvious.
4. **Ruled out: the modal failing to close.** Native 1.14.4 `Modal.close()` detaches
   synchronously on desktop (`canAnimate()` is `Platform.isPhone`). Also,
   `FreshnessRefreshSummaryModal.submit()` always calls `close()` after `onDone`. A
   single prompt therefore cannot survive its own answer.
5. **Precedent in this repo.** `jumpToOpenObsidianTask`
   (`src/610-plugin-jumps-and-counted-keys.js:239-250`) already documents the same race
   for `Ctrl+Shift+J/K`: "can reach this method twice in the same dispatch turn … the
   Obsidian hotkeys.json command … in the live app wins this race". It dedupes that
   race. The refresh route never got an equivalent guard.
6. **Same latent bug in Alt+N.** `handleCountedLaneTogglePhysicalKeydown`
   (`src/610-plugin-jumps-and-counted-keys.js`, ~615-649) has the same unguarded
   duplicate against the bound `bob-navigation-hotkeys:toggle-task-lane` (`Alt`+`N`).
   - On a walk landing it is masked, because `captureReviewGesture` takes the lock
     synchronously.
   - Off a landing, releasing an In Progress task can stack two "Release task" prompts
     the same way.

Both command paths already read and consume a pending Vim count themselves:
`refreshTaskFreshness` (520:300-318) and `toggleTaskLane` (510:314-330). So when
Obsidian's hotkey has handled the keydown, the fallback has nothing left to do.

## Fix

All code changes are in the **bob-plugins** linked repo. Open it with
`sase repo open bob-plugins -r "<why>"` and read its `AGENTS.md`. Edit only
`plugins/bob-navigation-hotkeys/src/` fragments; never hand-edit `main.js`.

1. **Make the refresh fallback a true fallback.** This is the root-cause fix, in
   `handleReviewRefreshPhysicalKeydown` (`src/540-plugin-decay-picker-and-cancel.js`).
   - Right after the `isReviewRefreshKeydown` matcher check, and before the WeakSet,
     view, or Vim reads, return `false` when `event.defaultPrevented === true`.
   - In that case, do not call `preventDefault`/`stop*`, do not reset Vim input state,
     and do not call `refreshTaskFreshness`.
   - Rewrite the comments on `registerReviewRefreshInputListeners` (540:240-245) and
     `isReviewRefreshKeydown` (480:824-828) to say what actually happens:
     - Obsidian ≥ 1.14's keymap is a `window` capture listener registered before
       plugins.
     - A bound hotkey runs the command first, whatever the Vim mode, and only calls
       `stopPropagation()`. That command path consumes a pending Vim count itself.
     - So the fallback skips keydowns that are already `defaultPrevented`. It acts only
       when no Obsidian binding handled the chord, for example after the hotkey is
       removed or on an older build.
     - Keep the existing "Alt+Shift+F is retired" sentence.
2. **Apply the same guard to the Alt+N fallback.**

   > [!decision] fix_alt_n

   If `fix_alt_n` is declined, skip this step and the Alt+N tests and checks below, and
   leave `handleCountedLaneTogglePhysicalKeydown` untouched.
   - Add the same `event.defaultPrevented` early return to
     `handleCountedLaneTogglePhysicalKeydown`, placed right after
     `isCountedLaneToggleKeydown`.
   - Update its block comment above `registerCountedLaneToggleInputListeners` the same
     way.
   - Do **not** change the other capture fallbacks:
     - `Ctrl+Shift+J/K` already has its dispatch dedupe.
     - `Ctrl+Shift+M` and counted `Ctrl+Shift+P` rely on the fallback for Vim-count
       handling and are protected by their active-picker guards.
     - `!` transclusion and Esc/Ctrl+[ clear-search have no prompt.

3. **Regression tests.** Extend `scripts/test-navigation-keep-counting.cjs`, next to the
   existing physical-key tests around lines 1240-1400 that use `reviewRefreshKeyEvent` /
   `attachReviewRefreshCapture`. Add the Alt+N tests in
   `scripts/test-navigation-hotkeys-lane-toggle.cjs` or a sibling. Model the live order:
   the command route runs first, then the event is marked `defaultPrevented`, then the
   fallback runs.
   - **Pending refresh, live order.**
     - Set up a Pending `[/]` task line at the cursor. Stub
       `requestFreshnessRefreshSummary` to record calls and resolve on demand, and stub
       `jumpToDueTask` to count advances.
     - Call `plugin.refreshTaskFreshness(editor, { advance: true })` as Obsidian's
       `editorCallback` would. Then set `defaultPrevented = true` on a Ctrl+Alt+F event
       and pass it to `handleReviewRefreshPhysicalKeydown`.
     - Assert the handler returns `false` and exactly one summary prompt was requested.
     - Resolve the prompt with a summary. Assert one stamp, one Work Log entry, and
       exactly one advance.
   - **Same for Alt+F.** No advance, and still one prompt.
   - **True fallback still works.** With `defaultPrevented` false (no Obsidian binding),
     the existing behavior is unchanged: the handler returns `true`, the count prefix is
     consumed, and there is one prompt. The existing "Ctrl+Alt+F capture advances once
     and consumes a Vim prefix" and "one physical key through two routes stamps once"
     tests must still pass as they are.
   - **Already-handled event leaves Vim state alone.** A `defaultPrevented` event leaves
     a pending Vim `prefixRepeat` untouched and calls nothing.
   - **Alt+N.**
     - A `defaultPrevented` Alt+N keydown in Vim normal mode returns `false` and does
       not call `toggleTaskLane`.
     - A non-prevented one still dispatches once, with the count.
4. **Version, build, verify, deploy.**
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.12.0 to **2.12.1**, and
     update the README plugin-table version cell (README.md line 16). The README
     behavior text already says "ask once", so it needs no wording change.
   - Run `npm run build`, `npm run build:check`, then `npm test`. Keep every hand-edited
     fragment at or below 1000 lines.
     - The only known unrelated flake is bob-cli-3w (the 16 ms stage-ranker timing
       test). If it fails, rerun it alone; do not count it against this tale.
   - Run `bob plugins sync`.
   - Optionally, if `ssh mac` is reachable, confirm the deploy read-only with
     `/Applications/Obsidian.app/Contents/MacOS/obsidian-cli plugin id=bob-navigation-hotkeys`.
     It should report 2.12.1 once the vault syncs. Do **not** reload plugins, run
     commands, or drive Obsidian's UI on the Mac.

## Manual check for Bryan (after plugin reload)

In Vim normal mode on the review walk:

- On a Pending landing, `Ctrl+Alt+F` shows exactly one "Refresh task" prompt. Enter
  stamps, writes the Work Log if one was typed, jumps once, and leaves no prompt behind.
- `Alt+F` on a Pending task shows one prompt and stays put.
- `3<Ctrl+Alt+F>` shows "Refresh 4 tasks" once.
- Off a landing, Alt+N on an In Progress task shows one "Release task" prompt.
