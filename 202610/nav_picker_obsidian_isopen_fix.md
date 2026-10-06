---
tier: tale
title: Fix dead Ctrl+Shift+M/P on Obsidian 1.14 and land bob-cli-4l
goal:
  Ctrl+Shift+M and Ctrl+Shift+P open their pickers again on Obsidian 1.14+, and epic
  bob-cli-4l is closed with its plan marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-4l.land
bead: bob-cli-4l
status: done
---

- **BEAD:**
  [bob-cli-4l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4l/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-4l.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-4l.land.md)
- **COMMITS:**
  - [14fbfe5](https://github.com/bobs-org/bob-plugins/commit/14fbfe5274dc37b731b6cf14539f998fa9600b33)
    — fix(nav): own pickerOpen flag instead of native Modal.isOpen (nav 2.6.2)

# Fix dead Ctrl+Shift+M/P on Obsidian 1.14 (FilteredPickerModal `isOpen` collision), then land bob-cli-4l

## Context

This tale is the remaining work of epic **bob-cli-4l** ("Answer once, advance once:
review-walk auto-advance"). All five phases are closed, and their work is verified in
bob-plugins (824ad2a, 7ff2459, 0e0fb98, f100300, 5d0a200) and bob-cli (756c4fc). Nothing
has drifted since the epic started, and it has no `--epic-symbol` entries. Bryan
reported that **Ctrl+Shift+M and Ctrl+Shift+P do nothing at all, everywhere** (no
picker, no Task Card, no toast), and chose to fix that inside this epic. The landing
agent's notes on bob-cli-4l hold the full findings.

### Root cause (confirmed live on the Mac with read-only `obsidian-cli eval`)

- The Mac runs **Obsidian 1.14.4**. Its native `Modal` now owns an instance field
  `isOpen` (set to `false` in the constructor). The decompiled `Modal.prototype.open` is
  `if (!this.isOpen) { this.isOpen = true; this.win = activeWindow; …attach…; this.onOpen() }`
  and `close` is `if (this.isOpen) { this.isOpen = false; …; detach; this.onClose() }`.
- bob-navigation-hotkeys' `FilteredPickerModal` (bob-plugins
  `plugins/bob-navigation-hotkeys/src/200-filtered-picker-modals.js`, lines 1-34) keeps
  its own guard under the **same name**. Its comment says "Obsidian's Modal does not
  expose isOpen; the picker owns this state". Its `open()` sets `this.isOpen = true`
  **before** `super.open()`, so Obsidian's `open()` sees `isOpen === true` and does
  nothing: no DOM, no `onOpen`, and `onClose` never runs later.
- The openers register the picker before calling `open()`
  (`this.activeTaskMoveDestinationPicker = picker` in
  `src/640-plugin-notes-and-task-move.js` and `this.activeBulletPropertyPicker = picker`
  in `src/550-plugin-cancel-and-counted-property.js`). Nothing ever clears that
  registration. From then on, every Ctrl+Shift+M / Ctrl+Shift+P hits the "picker already
  active" guard at the top of `openTaskMoveDestinationPicker` /
  `openTaskMoveOrPomodoroBulletPicker` / `openBulletPropertyPicker` and returns `true`
  silently.
- The live Mac state matched exactly: both active pickers had `isOpen: true`,
  `win: null`, and a disconnected `containerEl`, with 0 `.modal-container` elements
  present. `dev:errors` was empty, and nav 2.6.1 was loaded, enabled, and bound to ⌃⇧M /
  ⌃⇧P.
- Every `FilteredPickerModal` subclass is affected: the Task Card /
  `BulletPropertyPickerModal`, `TaskMoveDestinationPickerModal`,
  `PomodoroBulletMovePickerModal`, `PomodoroEntryMovePickerModal`,
  `ChildNotePickerModal`, `LinkCandidatePickerModal`, and `YankPathPickerModal`. No
  other `Modal` subclass in bob-plugins writes `isOpen` (audited).
- Obsidian 1.14.4's native instance fields are:
  `shouldRestoreSelection, selection, win, shouldAnimateOpen, isOpen, hasInitialInputFocus, dimBackground, bgOpacity, app, scope, onWindowClose, containerEl, bgEl, modalEl, headerEl, titleEl, contentEl`.
  Its prototype methods are:
  `getRootEl, canAnimate, doc, open, animateOpen, animateClose, onHistoryBack, close, onOpen, onClose, onWindowClose, onClickOutside, canDismiss, onEscapeKey, setTitle, setContent, setBackgroundOpacity, setCloseCallback, setDimBackground`.
  `FilteredPickerModal` also overwrites `this.headerEl` / `this.titleEl` (200:91/94) and
  the Task Card nulls them (350:128/130). Native `open`/`close` do not read those
  fields, so leave them as they are unless the audit in step 2 shows a real break.
- The epic's own diffs did not touch modal open/close. The node test harness could not
  reproduce the bug because `scripts/modal-harness.cjs` `ModalStub` tracks `attached`,
  not a native `isOpen`.

## Work (all code changes are in the bob-plugins linked repo; open it with `sase repo open bob-plugins -r "<why>"` and read its AGENTS.md)

1. **Reproduce in the harness first.** Make `ModalStub` in `scripts/modal-harness.cjs`
   behave like Obsidian ≥ 1.14:
   - Set `this.isOpen = false` in the constructor.
   - `open()` returns early when `this.isOpen` is already true; otherwise it sets
     `isOpen = true` and then does the existing attach/`onOpen` work.
   - `close()` returns early when `isOpen` is false; otherwise it sets `isOpen = false`
     and then does the existing detach/`onClose` work.

   Keep the existing `attached` bookkeeping only if other tests still need it. Then run
   the nav picker suites, for example
   `node scripts/test-navigation-hotkeys-picker-lifecycle.cjs` and the Task Card / move
   picker tests, and confirm they fail on the current tree. Also add one focused
   regression test (for example in `test-navigation-hotkeys-picker-lifecycle.cjs`) that:
   - opens a Task Card and a task-move picker through the real plugin openers;
   - asserts the modal actually attached and `onOpen` ran;
   - closes each one and asserts the plugin's active-picker slot is cleared;
   - opens each picker a second time and asserts that it works.

   If any other harness defines its own `TestModal` for nav pickers (for example
   `scripts/navigation-hotkeys-harness.cjs`, which extends `ModalStub`), make sure it
   inherits the same native-`isOpen` semantics.

2. **Fix `FilteredPickerModal`** (`src/200-filtered-picker-modals.js`):
   - Rename the picker-owned open flag from `isOpen` to a name Obsidian does not own,
     such as `pickerOpen`, in the constructor, `open()`, and `close()`.
   - Leave `isOpen` entirely to Obsidian, so the picker never writes it.
   - Update the comment to say that Obsidian ≥ 1.14 `Modal` owns `isOpen`, and that
     writing it before `super.open()` turns `open()` into a no-op.
   - Update every reader of the picker flag to the new name. Known sites:
     - `src/200-filtered-picker-modals.js:124` (initial input focus);
     - `src/350-picker-task-card.js:70, 233, 244, 353, 399, 513, 589, 720`;
     - `src/500-plugin-transclusion-and-link-picker.js:527` (`picker.isOpen === true`).

     Re-grep `src/` for `\.isOpen\b` on picker instances to catch any others. Do not
     touch the unrelated `isOpen` fields in block-id-prompt's Pomodoro source context.

   - Quickly audit the other native names listed above against `FilteredPickerModal` and
     `BulletPropertyPicker*` own fields and methods. Fix only collisions that actually
     break open/close.

3. **Make the active-picker guards fail open instead of silently swallowing the key.**
   In each "already active" guard, drop the registered picker and continue opening a
   fresh one when either of these holds:
   - the registered picker's `pickerOpen` is false;
   - its `containerEl` exposes `isConnected === false`. Check this strictly, so test
     stubs without `isConnected` behave as before.

   The guards are:
   - `openBulletPropertyPicker` in `src/550-plugin-cancel-and-counted-property.js`;
   - `openTaskMoveDestinationPicker`, the Pomodoro bullet/entry openers, and
     `openTaskMoveOrPomodoroBulletPicker` in `src/640-plugin-notes-and-task-move.js`;
   - any sibling guard on `activeTaskMoveDestinationPicker` /
     `activeBulletPropertyPicker` / link-picker slots.

   Cover this with a test that plants a never-opened (detached) picker in the slot and
   asserts the next Ctrl+Shift+P / Ctrl+Shift+M opens a working picker.

   Keep the review-walk capture logic (`captureReviewGesture` / `hadActivePicker`)
   unchanged, except that a dropped stale picker must count as "no active picker".

4. **Version, build, verify, deploy.**
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.6.1 to **2.6.2**.
   - Update the README plugin table row and the "ahead of the others at `2.6.1`"
     sentence to 2.6.2.
   - Run `npm run build`, then `npm run build:check`, then the full `npm test`. The only
     known unrelated flake is bob-cli-3w ("stage ranker filters 1,000 synthetic tasks
     under 16 ms", load-dependent). If it fails, rerun that test alone to confirm, and
     do not count it against this tale.
   - Keep every hand-edited fragment at or below 1000 lines.
   - Run `bob plugins sync`.
   - If the Mac is reachable (`ssh mac`; best effort), confirm deployment read-only with
     `/Applications/Obsidian.app/Contents/MacOS/obsidian-cli plugin id=bob-navigation-hotkeys`,
     which should report version 2.6.2 once the vault has synced. Do **not** reload
     plugins, run commands, or drive Obsidian's UI on the Mac. Tell Bryan to reload the
     plugin or restart Obsidian, then press Ctrl+Shift+M and Ctrl+Shift+P on an ordinary
     task line, and run the bob-cli-4l.5 manual smoke checklist (that bead's note #1).
   - Leave bob-cli-4e out of this tale. It is the pre-existing shadowed Task Card
     `onClose` cleanup (class removal and `linkResolving` / `taskCardDispatching`
     resets), a separate ready task. If your edits touch the same `onClose`, keep that
     defect as it is.

5. **Close out epic bob-cli-4l (final step; do it in this same turn, after steps 1-4
   pass).**
   - Run `sase bead epic-symbols bob-cli-4l`. It currently lists none. For any entry
     that has appeared, resolve the symbol (wire it up, privatize it, add a non-test
     pragma, or delete it per the Symvision epic-whitelist policy), or re-key its
     Justfile line to a still-open bead that needs the exemption.
   - Close the epic with `sase bead close bob-cli-4l --note "<verification>"`. The note
     must cover:
     - phases .1-.5 verified in bob-plugins 824ad2a/7ff2459/0e0fb98/f100300/5d0a200 and
       bob-cli 756c4fc;
     - no drift to integrate and no epic symbols;
     - the Ctrl+Shift+M/P regression root cause (the Obsidian 1.14 native `Modal.isOpen`
       collision in `FilteredPickerModal`, not caused by the epic's diffs, but fixed
       here at Bryan's request) and the fix commit and nav 2.6.2;
     - the test results;
     - follow-up triage: no PROPOSED FOLLOW-UP notes on any child; the perf flake
       corroborated as +1 on bob-cli-3w; bob-cli-4e left as its own ready task.

     Never use `--force` merely to make the close succeed.

   - Run `just symvision` from the bob-cli workspace and confirm the whitelist is clean.
   - Set `status: done` in the YAML frontmatter of the epic's plan file. That is the
     PLAN path printed by `sase bead read bob-cli-4l -r "Need the plan path"`, i.e.
     `plan:202610/review_walk_answer_auto_advance.md` in the plans sidecar.
   - bob-cli-4l has no `parent_bead`. Confirm this in the same `sase bead read` output;
     with no parent, the landing finishes after the plan-file update.
