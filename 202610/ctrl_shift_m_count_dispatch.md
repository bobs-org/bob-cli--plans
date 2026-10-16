---
tier: tale
title: Fix N<Ctrl+Shift+M> dropping its count
goal: A counted Ctrl+Shift+M always moves the current task plus the next N tasks,
  on and off a review-walk landing and with a cold or warm cache, because one physical
  key now means one dispatch that reads the Vim count on the command route.
size: small
proposed_by: bbugyi200.apollo.7z
status: done
---

# Plan: Fix `N<Ctrl+Shift+M>` dropping its count

## Verdict

Bryan suspected that `N<Ctrl+Shift+M>` (move the current task plus the next N tasks)
does not always honour the count. **Confirmed.** It fails in two ways:

- **On a `]s` review-walk landing**, a counted move never works. Depending on timing,
  either no picker opens at all, or the picker opens for one task.
- **Off a landing**, it fails intermittently. The picker opens with "1 selected task"
  whenever any open project note's content is not already in Obsidian's read cache. Cold
  caches after a restart make this common.

In every failing case the typed count is still consumed: the Vim pending count is
cleared, then the count is thrown away.

## Root cause

All code lives in the **bob-plugins** linked repo, plugin `bob-navigation-hotkeys`.

### How one counted keypress is dispatched today

One physical `2<C-S-M>` in Vim normal mode is dispatched **twice**:

1. **Obsidian's own hotkey runs first.**
   - Obsidian ≥ 1.14's keymap is a `window` capture-phase keydown listener, registered
     before any plugin loads. This was verified on the live Mac in
     `plan:202610/refresh_prompt_double_dispatch.md`.
   - It matches the plugin's default hotkey for command `move-tasks-to-note`
     (`src/490-plugin-lifecycle.js`; `~/bob/.obsidian/hotkeys.json` does not override
     it).
   - It runs the `editorCallback`, which calls
     `openTaskMoveOrPomodoroBulletPicker(editor, view)` **with no count**. This command
     route never reads the pending Vim count.
   - It then calls `preventDefault()` and `stopPropagation()`, but not
     `stopImmediatePropagation()`.
2. **nav's own capture fallback runs second.**
   - `handleCountedTaskMovePhysicalKeydown` (`src/610-plugin-jumps-and-counted-keys.js`)
     listens on the same `window` capture phase, so it still receives the event.
   - It reads the pending Vim count, resets the Vim input state, and calls the
     dispatcher again with `{ countExplicit: true, additionalTaskCount: N }`.
3. **The plugin relies on the second call upgrading the first.**
   - Since 8f0276e (2026-07-17), `openTaskMoveDestinationPicker`
     (`src/640-plugin-notes-and-task-move.js`) closes an open _bare_ picker when a
     _counted_ request arrives, then reopens with the count.
   - Both later features break this upgrade.

### Failure 1: the async opening guard drops the counted call

This has been happening since a42f45d (2026-10-12, "schedule projects from inline ^prj
fields").

- That commit made the destination list asynchronous whenever any `wip`/`waiting`
  project note is a destination, which is always true in the real vault.
  `hydrateTaskMoveDestinationSchedules` (`src/240-cancel-and-lanes.js`) runs
  `vault.cachedRead` on every project note.
- The bare call from step 1 sets `this.taskMoveDestinationPickerOpening = true` and
  returns before any picker exists.
- The counted call from step 2 hits
  `if (this.taskMoveDestinationPickerOpening) return true;` at the top of
  `openTaskMoveDestinationPicker` and is silently dropped. The fallback has already
  reset the Vim count by then.
- When hydration resolves, the bare (one-task) picker opens.
- **Why it only sometimes fails:** browsers flush microtasks between the two listeners
  of a native event. If every `cachedRead` resolves within that flush (all project notes
  cached), the bare picker is already open when the counted call arrives, and the old
  upgrade still works. A single disk read loses the count.

### Failure 2: the review-walk gesture lock swallows the upgrade on a landing

This has been happening since f100300 (2026-10-06, review-walk capture on Ctrl+Shift+M).

- The bare picker's `openPicker` calls `captureReviewGesture`. On a landing, that takes
  the 3 s review-walk lock.
- The counted upgrade then closes the bare picker. That picker's `onClose` settles the
  origin only in a later `setTimeout`.
- The counted reopen therefore calls `captureReviewGesture` while the lock is still
  held. It gets `REVIEW_WALK_BUSY` and returns without opening anything. The result is
  no picker, and the count is consumed.
- With a cold cache, Failure 1 wins first instead, and the result is a one-task picker.

### Reproduction (done during planning, against the real plugin code)

A throwaway Node script drove the real `NavigationHotkeysPlugin` through the live order:

1. command route with no count;
2. a microtask flush;
3. the fallback on the now-`defaultPrevented` event with a pending Vim count of `2`.

The test vault had one area destination and one `wip` project destination.

| Scenario                                                       | Picker result       |
| -------------------------------------------------------------- | ------------------- |
| Off landing, `cachedRead` resolves in microtasks               | counted, 3 tasks ✅ |
| Off landing, `cachedRead` resolves on a later task (disk read) | **bare, 1 task** ❌ |
| On a landing (real lock primitives), cache hit                 | **no picker** ❌    |
| On a landing, disk read                                        | **bare, 1 task** ❌ |

In every row the Vim count ended up cleared.

Existing tests missed both failures for two reasons:

- They call `openTaskMoveDestinationPicker` directly with counts, for example "a counted
  move from a landing resumes past every moved target".
- Their harness has no `wip` project destination and no `cachedRead`, so they only take
  the synchronous path.

## Fix

Open the repo with `sase repo open bob-plugins -r "<why>"` and read its `AGENTS.md`.

- Edit only `plugins/bob-navigation-hotkeys/src/` fragments, plus tests, the manifest,
  and the README.
- Never hand-edit `main.js`; regenerate it with `npm run build`.
- Each hand-edited fragment must stay at or below 1000 lines. `610` is at 983 lines and
  `640` at 977, so do not grow either one meaningfully.

The fix makes one physical key mean **one dispatch**, which is the pattern Alt+N and
Alt+F already use (c837be9).

### 1. New fragment for counted-move dispatch

Create `src/615-plugin-counted-task-move.js`, holding a new mixin class
`BobNavigationHotkeysCountedTaskMoveMixin`.

- Add the fragment to `src/fragments.json` right after
  `"610-plugin-jumps-and-counted-keys.js"`.
- Add the mixin to the `installBobNavigationHotkeysMixins` list in
  `src/690-install-plugin-mixins.js`, right after `BobNavigationHotkeysJumpKeyMixin`.
- **Move**, unchanged apart from step 3, these three methods out of `610` into the new
  mixin: `registerCountedTaskMoveInputListeners`,
  `handleCountedTaskMovePhysicalKeydown`, and `isCountedTaskMoveKeydown`. The installer
  throws on duplicate method names, so remove them from `610`.

### 2. The command route consumes the pending Vim count

Add this method to the new mixin:

```js
// Obsidian >= 1.14 runs the bound Ctrl+Shift+M command from its window
// capture keymap before CodeMirror Vim (or nav's fallback) sees the key, so
// the command route reads and consumes a pending Vim count itself, as
// toggleTaskLane and refreshTaskFreshness do. An explicit count wins.
withPendingTaskMoveCount(editor, view, options = {}) {
  if (options.countExplicit === true) {
    return options;
  }
  const vimView = view || this.getActiveMarkdownView();
  if (!this.isVimNormalModeEditor(editor, vimView)) {
    return options;
  }
  const cm = this.resolveVimCodeMirror(editor, vimView);
  const pending = getPendingVimRepeat(cm);
  if (!pending.explicit) {
    return options;
  }
  resetPendingVimInputState(cm, "counted-task-move");
  return {
    ...options,
    countExplicit: true,
    additionalTaskCount: pending.repeat,
  };
}
```

Then make its result the first statement of `openTaskMoveOrPomodoroBulletPicker`
(`src/640`):

```js
options = this.withPendingTaskMoveCount(editor, view, options);
```

This covers all three Ctrl+Shift+M contexts at once: tasks, Pomodoro sub-bullets, and
Pomodoro entries (the entry picker accepts and ignores a count). The count semantics
stay the same: `N` means the current task plus N more, matching the fallback and the
README.

### 3. The fallback becomes a true fallback

In `handleCountedTaskMovePhysicalKeydown`, add an early return right after the
`isCountedTaskMoveKeydown` check and before the WeakSet, view, or Vim reads:

```js
if (event && event.defaultPrevented === true) {
  return false;
}
```

- On that path, do not call `preventDefault`/`stop*`, do not reset Vim state, and do not
  open anything.
- Above `registerCountedTaskMoveInputListeners`, add a block comment in the same shape
  as the one above `registerCountedLaneToggleInputListeners`. It should say:
  - Obsidian ≥ 1.14 runs a bound hotkey first and only calls `stopPropagation`.
  - The command route consumes the Vim count itself (`withPendingTaskMoveCount`).
  - So this listener acts only when no Obsidian binding handled the chord, for example
    after the hotkey is removed or on an older build.
- Keep the existing `event.repeat` and WeakSet de-duplication.

**Why this is robust:** if a future Obsidian build stops marking handled keydowns
`defaultPrevented`, the fallback runs after the counted command route and finds no
count, because the route already reset it. Its bare request then loses to the existing
guards: the in-flight `taskMoveDestinationPickerOpening` guard, or the rule that an
active counted picker ignores a bare request. The counted session survives either way.

Leave the bare→counted reconcile blocks in `openTaskMoveDestinationPicker`,
`openPomodoroBulletMovePicker`, and `openPomodoroEntryMovePicker` in place as defensive
guards. Leave the `taskMoveDestinationPickerOpening` guard unchanged.

### 4. Regression tests

Model the **live order** in every new test:

1. Call the command route the way the `editorCallback` does:
   `plugin.openTaskMoveOrPomodoroBulletPicker(editor, view)`, with the editor carrying a
   Vim `cm` stub in normal mode with a pending count.
2. Pass a Ctrl+Shift+M event with `defaultPrevented: true` to
   `handleCountedTaskMovePhysicalKeydown`.

Stub `isVimNormalModeEditor` / `resolveVimCodeMirror` as
`scripts/test-navigation-hotkeys-lane-toggle.cjs` does, or give the editor `cm.cm`.

Put the tests in `scripts/test-navigation-hotkeys-task-move.cjs`, or in a new sibling
file `scripts/test-navigation-hotkeys-task-move-count-dispatch.cjs`. A new file must be
added to the `test` script list in `package.json`.

1. **Async destinations, cold cache.**
   - Set up an area destination plus a `[[project]]` destination with `status: wip`. Its
     `vault.cachedRead` resolves on a later macrotask (`setTimeout`).
   - Run the live order with a pending count of 2. The fallback returns `false`, calls
     no `preventDefault`, and opens nothing.
   - After hydration settles, exactly one `TaskMoveDestinationPickerModal` is open, with
     `session.countExplicit === true` and `session.discovery.actualCount === 3`.
   - The Vim input state is reset.
2. **Async destinations, warm cache.** Same as test 1, with `cachedRead` returning
   `Promise.resolve("")`. Expect the same counted 3-task picker, and assert that only
   one picker was ever constructed or opened (no bare-then-close flicker).
3. **Bare command route.**
   - With no pending count in Vim normal mode, the picker is uncounted with one target.
   - With the editor in insert mode, a stale `prefixRepeat` is ignored and left
     untouched.
4. **Already-handled event leaves Vim state alone.** A `defaultPrevented` Ctrl+Shift+M
   with a pending `prefixRepeat: ["2"]` returns `false`, opens nothing, and leaves
   `prefixRepeat` as `["2"]`.
5. **True fallback still works.** The existing test "physical task move chord declines
   auto-repeat and consumes Vim normal input once" must still pass unchanged. Its events
   are not `defaultPrevented`.
6. **Pomodoro sub-bullet.**
   - Use `createPomodoroMovePickerHarness` with a Vim count stub.
   - The live order opens one `PomodoroBulletMovePickerModal` whose session is counted,
     with `discovery.actualCount` equal to N+1 (clamped as the fixture allows).
7. **Walk landing.** Add this to `scripts/test-navigation-review-advance-gestures.cjs`,
   next to "a counted move from a landing resumes past every moved target".
   - `land(fixture, "first")`, then drive the live order with a pending count of 1.
   - Assert that exactly one picker opened, with `session.countExplicit === true`, two
     targets, and a non-null `session.reviewOrigin`.
   - Assert that after `openItemAtIndex(0)` and `flushAll`, the walk parks with
     `reviewAnchor.resumeNext.text === "- [*] #task Three"`, as in the existing direct
     test.
   - Before the fix, this test fails with no picker. Confirm that by running it against
     the unfixed code.

Keep the existing reconcile tests in
`scripts/test-navigation-hotkeys-picker-lifecycle.cjs` as they are.

### 5. Version, build, verify, deploy

1. Bump `bob-navigation-hotkeys` from 2.40.0 to 2.40.1 in
   `plugins/bob-navigation-hotkeys/manifest.json`, and in its version cell in the root
   `README.md` table. The README behaviour text already promises counted moves from
   landings, so no prose changes are needed.
2. From the bob-plugins root, run:
   - `npm run build`
   - `npm test`
   - `npm run validate`

   All three must pass. `build:check` must report that `main.js` is current.

3. Run `bob plugins sync` to deploy to the vault, as `AGENTS.md` requires.
4. In the final response, give Bryan this manual check on the Mac:
   - In Vim normal mode on a task in an area note, `2<Ctrl+Shift+M>` shows "3 selected
     tasks" in the picker subtitle.
   - The same holds right after a `]s` landing, and right after restarting Obsidian
     (cold cache).
   - Bare `Ctrl+Shift+M` still shows "1 selected task".

## Out of scope

- **Counted Task Card (`N<Ctrl+Shift+P>` / `N<Ctrl+Shift+B>`).** It uses the same
  two-dispatch upgrade.
  - Its count works, because that path is synchronous with no opening guard.
  - But the upgraded counted card deliberately skips review capture
    (`!hadActivePicker`). A counted Task Card opened on a landing therefore never
    advances the walk.
  - Whether that is intended is a separate question for Bryan. Do not change it here.
- No bob-cli changes, no memory or decision-record changes, and no change to move
  semantics, focus, walk parking, or the opening and reconcile guards.

## Acceptance

- Every new test above passes, and the landing test fails against the pre-fix code.
- `npm test` and `npm run validate` pass in bob-plugins.
- `bob-navigation-hotkeys` is at 2.40.1 and deployed with `bob plugins sync`.
- No fragment exceeds 1000 lines.
