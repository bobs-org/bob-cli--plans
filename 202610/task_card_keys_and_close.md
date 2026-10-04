---
tier: tale
title: Make Task Card keys work and close the card after a write
goal: Every Task Card key works from open, every writing gesture closes the card,
  Next/Pending P-level picks show the Work summary stage, and the test harness models
  Obsidian's real Modal lifecycle.
size: medium
proposed_by: bbugyi200.athena.0w4.f1
status: done
---

# Make Task Card keys work and close the card after a write

## Outcome

On the `Ctrl+Shift+P` Task Card (`bob-navigation-hotkeys` 2.1.0):

- Every documented card key works from the moment the card opens: `1`–`9`, `0`, `Enter`,
  `b`, `f`, `x`, `Alt+N`, `Ctrl+Enter` / `Cmd+Enter`, `Ctrl+R`, `Ctrl+D`, the arrows,
  `Escape`, `q`, and `Ctrl+]`. They keep working after a mouse click anywhere on the
  card.
- Any card gesture that writes closes the card, whether it came from a key or a click.
  For example, `1` or a click on **P1** on a Ready task writes once and closes.
- On a Next (`[*]`) or Pending (`[/]`) task, a P-level or recommendation gesture opens
  the visible **Schedule task** Work summary stage, with the cursor in the field.
  Pressing `↵` writes once and closes. `Esc` writes nothing.
- Every stage the card opens focuses its input. This covers Schedule, Depends on, Review
  every, the cancel reason, Work summary, More and the schedule review. `Backspace` on
  an empty field and the Back button repaint the card.
- Task Link cards leave "Resolving…" and become usable.
- The other filtered pickers focus their search input again. These are the child-note,
  task-move, Pomodoro-move, link-candidate and yank-path pickers.
- The shared test harness models Obsidian's real `Modal` lifecycle, so the test suite
  catches this class of bug.

## Root causes (verified)

The causes were checked in two ways:

- **Obsidian source.** `Modal` in Obsidian 1.13.7's `app.js`, the build running on
  Bryan's MacBook, plus the public `obsidian` typings.
- **jsdom reproduction.** A scratch repro drove the deployed 2.1.0 `main.js` through a
  `Modal` shim that copies 1.13.7's `open()`, `close()`, initial-focus and Escape
  behavior. The mac's vault runs a byte-identical `main.js`.

### 1. `isOpen` is not an Obsidian `Modal` property (primary cause)

Obsidian's `Modal` never sets `isOpen`:

- `open()` guards on `containerEl.parentNode`.
- `close()` sets no flag.
- Only `SuggestModal.onOpen` sets `this.isOpen`, and the public typings have no such
  member.

The harness `scripts/modal-harness.cjs` `ModalStub` invents `isOpen`. So the tests pass,
while in Obsidian `this.isOpen` is always `undefined`. The plugin first relied on it in
the Task Card work (c0ff974, b3269d9). The vault ran 1.71.0 until the 2.1.0 sync on
2026-10-04, so the regression went live that day. Seven readers break:

- **`FilteredPickerModal.onOpen` deferred input focus** (`if (this.isOpen && …)`) never
  fires.
  - Every stage the card opens starts with focus on `<body>`, so typing, `Enter`,
    `Backspace` and `Ctrl+]` in it do nothing until clicked.
  - Every other `FilteredPickerModal` picker lost input autofocus too.
- **`BulletPropertyPickerModal.dispatchTaskCardIntent`** closes after a successful write
  only `if (result === true && this.isOpen)`. So it never closes, which is the P1-click
  symptom.
- **`showTaskCard`** repaints only when `isOpen`. So Back, `Backspace` on an empty
  field, `Ctrl+R` and refused deletions update the model but leave stale UI on screen.
- **`openTaskCardStage`** never falls back to the card when no stage took over.
- **`returnHome`** never closes a refused direct stage.
- **`deleteTaskCardProperty`** never returns to the card after a refused delete.
- **The Task Link opener** treats a session as current only when
  `picker.isOpen === true`. That check always fails, so a Task Link card stays
  `linkResolving` forever and swallows every key and click.

### 2. Obsidian moves focus after `onOpen`

After calling `onOpen()`, `Modal.open()` focuses the first focusable element in
`modalEl` when a physical keyboard is present:

- The selector is
  `a[href], button, input, select, textarea, [contenteditable], [tabindex]`, minus
  disabled elements and `tabindex="-1"`.
- On the card, that element is the header **Close** `<button>`. It overrides the card's
  synchronous `listEl.focus()`.
- The Close button forwards most keys to the resolver but keeps `Enter` and `Space`
  native. So `Enter` closes the card instead of opening Schedule, and `Ctrl+Enter` /
  `Cmd+Enter` never reach the resolver.
- Obsidian's own header "x" (`.modal-header-button`) is not focusable.

The other modals in this plugin already defer focus with `window.setTimeout(…, 0)` for
this reason. The card does not.

### 3. Card priority and recommendation gestures never paint the Work Log stage

On a Next or Pending task:

- `commitTaskCardPriority` and `applyTaskCardRecommendation` reach
  `offerSchedulingWorkLogOrDispatch`, which calls `showSchedulingWorkLogStage`.
- That method paints only `if (this.resultsEl)`. The card sets `resultsEl` to null, and
  nothing on this path calls `ensureTaskCardStageChrome()`.
- So the stage switches to `schedule-work-log` invisibly. Nothing is written, the card
  looks dead, and repeated presses do the same.

This is the "pressing `1` does nothing" symptom. The existing test ("pending priority
action passes its card preview through the Work Log adapter") stubs out
`offerSchedulingWorkLogOrDispatch`, so this path never ran under test.

### 4. Hardening: card keys are bound per element

Today the card's keydown listeners sit on individual elements: the actions list, the
Close button, the priority radios, the banner and the More rows.

In a real browser, a click on non-focusable card content moves focus to `<body>`. That
content includes the title, chips, dates, timeline and footer. After such a click, no
card key reaches the resolver; only Escape still works, through Obsidian's modal scope.

### Repro evidence

These are jsdom runs of the deployed 2.1.0 `main.js` with an Obsidian 1.13.7-faithful
`Modal`.

| Case                                | 2.1.0 today                             | With fixes 1–3 prototyped                                         |
| ----------------------------------- | --------------------------------------- | ----------------------------------------------------------------- |
| Focus after open                    | Close button                            | actions list                                                      |
| Ready: `1` / P1 click               | writes 1, stays open                    | writes 1, closes                                                  |
| Next/Pending: `1` / P1 click        | invisible `schedule-work-log`, writes 0 | painted **Schedule task**, input focused; `↵` writes 1 and closes |
| Next overdue: `Ctrl+Enter` / banner | invisible `schedule-work-log`           | painted Work summary stage                                        |
| `Enter` on open                     | closes the card                         | Schedule stage, date input focused                                |
| `b` / `f` / `x`                     | stage painted, focus on `<body>`        | stage input focused                                               |
| `b` then Back                       | stage still on screen                   | card repainted, list focused                                      |

## Implementation

All plugin work happens in the linked `bob-plugins` repository; open it with
`/sase_repo` first. Read its `AGENTS.md`. The code lives in
`plugins/bob-navigation-hotkeys/main.js`, and the tests in `scripts/`.

### A. Plugin-owned open state

- In `FilteredPickerModal`, initialize `this.isOpen = false` in the constructor.
- Override `open()`:
  - No-op when already open.
  - Set `isOpen = true` before `super.open()`, because `onOpen` and its deferred focus
    callbacks read it.
  - Reset the flag if `super.open()` throws.
- Override `close()`:
  - No-op unless open.
  - Set `isOpen = false` before `super.close()`.
  - This makes close idempotent. Obsidian's is not: a second `close()` re-runs
    `onClose`.
- Add a short comment that Obsidian's `Modal` has no `isOpen`, so the plugin owns it.
- Keep the seven existing readers as they are; they become correct.
- Grep for any other `isOpen` read on a direct `Modal` subclass. None exists today.

### B. Deferred initial card focus

- In `BulletPropertyPickerModal.onOpen`'s card path, after `renderTaskCard()`, schedule
  a `window.setTimeout(…, 0)` that focuses the actions list.
- Guard the callback the same way `FilteredPickerModal` does: it runs only if
  `this.isOpen`, `this.stage === "task-card"`, and the list element is still the one
  just rendered.
- Re-renders while the modal is open keep their current synchronous focus.
- Do not set Obsidian internals such as `hasInitialInputFocus`; they are not public API.

### C. One card-level key router and a focusable card root

- Bind one keydown listener on `contentEl`, once per modal, the way `bindCloseChord` is
  bound.
  - It routes to `handleTaskCardKeydown` only while `this.stage === "task-card"`.
  - It ignores events already handled (`defaultPrevented`).
- While the card is painted, give its root `tabindex="-1"`. A click on non-focusable
  card content then focuses the root and keys keep routing. Obsidian's initial-focus
  selector skips `tabindex="-1"`, so B still decides initial focus. Remove the attribute
  when a stage takes over the chrome (`ensureTaskCardStageChrome` /
  `FilteredPickerModal` rendering).
- Element listeners keep only their own semantics and let every other key bubble to the
  router:
  - Priority radios: Arrow / Home / End and Enter / Space activation.
  - Banner and More rows: Enter / Space activation.
  - Close button: native Enter / Space.
- Remove the per-element `routeCardKeydown` forwarding and the `listEl` keydown binding,
  so each key reaches `handleTaskCardKeydown` exactly once.
- `renderTaskCardView` may keep an `onKeydown` option for standalone renders, but the
  modal must not route the same event twice.

### D. Paint every stage a card intent opens

- `showSchedulingWorkLogStage` calls `this.ensureTaskCardStageChrome()` before setting
  its stage. That method is already a no-op off the card.
- Then audit every other stage entry reachable from `dispatchTaskCardIntent` without
  `openTaskCardStage`, and apply the same rule:
  - `applyRecommendedRoll`, `applyCountedRecommendedRoll`, `applyLinkRecommendedRoll`
    and `maybeOfferPinnedRollWorkLog`.
  - The schedule-review stage.
  - `deletePropertyItem`, for `0` and `Ctrl+D`.
  - `plugin.applyLaneToggleFromPicker`.
- The rule: any `show*Stage` that can run while `this.stage === "task-card"` swaps the
  chrome first.
- Confirming the Work Log stage keeps its current flow:
  - `openItemAtIndex` closes on a truthy result.
  - `Esc` writes nothing.
  - Back / `Backspace` on an empty field returns to a repainted card.

### E. Make the harness model Obsidian

Edit `scripts/modal-harness.cjs`.

`ModalStub` mirrors Obsidian 1.13.7:

- It has no `isOpen`. Expose a test-only `attached` flag instead.
- `open()` no-ops while attached. Otherwise it attaches, runs `onOpen()`, then focuses
  the first focusable descendant of `modalEl` in document order, using the same selector
  and `tabindex="-1"` exclusion as Obsidian. Then it flushes deferred callbacks.
- `close()` detaches and runs `onClose()` on every call, so it is not idempotent. Its
  existing Escape handling stays.

`ElementStub` gains:

- Parent pointers.
- Several listeners per event type. Keep `listeners[type]` callable for existing direct
  calls, invoking every listener.
- A shared focus tracker: `focus()`, `blur()` and a harness `activeElement`.
- `removeAttribute`.

The harness also exports:

- A deferred `setTimeout` queue with `flushDeferred()`. It replaces the synchronous
  `global.window = { setTimeout: (cb) => cb() }` in the five navigation test files that
  use the harness, so deferred focus runs after `open()`'s initial-focus step, as in
  Obsidian.
- A `pressKey(modal, key, modifiers)` helper:
  - It dispatches keydown at the focused element and bubbles through ancestors, honoring
    `stopPropagation`.
  - It emulates native button activation for an unprevented `Enter` / `Space` on a
    focused `button`.
  - It then flushes deferred callbacks.
- A `click(element)` helper that focuses the element's nearest focusable ancestor, or
  the body, before firing `click`, as browsers do.

Then update the tests built on the wrong assumptions:

- "card focus is synchronous; …" becomes "after open the actions list, not the Close
  button, has focus".
- Tests that call `listEl.listeners.keydown` directly move to `pressKey`. Pure resolver
  tests in `test-navigation-task-card-model.cjs` stay as they are.
- Update the existing `isOpen` assertions so they still pass under the faithful stub.
  They now read the plugin-owned flag; other modal classes use `attached`.

### F. Regression tests

Put these in `scripts/test-navigation-task-card-view.cjs` unless noted. Drive each
through `modal.open()` and `pressKey` / `click`, and stub writers to return `true`.

1. **Initial focus.** After open, focus is on the actions list.
   - `Enter` opens the Schedule stage, not a close, and its date input has focus.
   - `Ctrl+Enter` and `Cmd+Enter` reach the recommendation.
2. **Ready-task writes close.**
   - `1` writes once and closes.
   - A click on the P1 radio writes once and closes.
   - A successful `0`, `Ctrl+D`, `Ctrl+Enter` / banner click and `Alt+N` also close.
3. **Next and Pending tasks.** Do not stub `offerSchedulingWorkLogOrDispatch`.
   - `1` and a P1 click paint **Schedule task** with its Work summary input focused, and
     nothing is written.
   - An empty `↵` writes once with the card's frozen date and closes.
   - `Esc` writes nothing.
   - `Ctrl+Enter` on an overdue Next task paints the same stage. Cover counted sessions
     too.
4. **Stage focus and Back.** Each stage the card opens (`Enter`, `b`, `f`, `x`, a More
   row) focuses its input.
   - `q` typed there stays text.
   - `Ctrl+]` closes from the focused input.
   - `Backspace` on an empty field and the Back button repaint the card and refocus the
     list.
5. **Ctrl+R.** `Ctrl+R` repaints the card with the new previews.
6. **Task Link cards.** After resolution, a Task Link card is interactive:
   `linkResolving` is false and `1` writes through the link writer. Run the existing
   Task Link resolution tests on the faithful stub.
7. **Click on card text.** A click on card content (title or a chip) followed by `1`
   still reaches the resolver exactly once.
8. **Close idempotence.** Two `close()` calls run `onClose` once.
9. **Other pickers.** One other `FilteredPickerModal` picker focuses its input after
   open, for example `ChildNotePickerModal` in `scripts/test-navigation-hotkeys.cjs`.

### G. Version, docs and deploy

- In `bob-plugins`:
  - Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.1.0 to 2.1.1.
  - In the README's Task Card section, note that a gesture that writes closes the card.
  - Run `npm test` and `npm run validate`.
- In this repo (`bob-cli`), edit `docs/projects.md` § Task Card with one or two
  sentences:
  - A card gesture that writes closes the card.
  - A P-level or recommendation on a Next/Pending task first opens the Work summary
    stage, linked to "Scheduling Work Log prompt".
- Deploy with
  `bob plugins sync -p bob-navigation-hotkeys --repo <opened bob-plugins checkout>`. Add
  `--no-pull` when that checkout has unpushed commits.
  - The default `--repo` is a different clone that lacks these changes.
  - Afterwards, confirm the vault's `main.js` and `manifest.json` match the source byte
    for byte.

## Acceptance

- `npm test` and `npm run validate` pass in `bob-plugins`.
- The new tests in F fail against 2.1.0 behavior and pass after the fix. The harness has
  no `isOpen`.
- The vault copy of `bob-navigation-hotkeys` is 2.1.1 and identical to the source.
- Manual check on the mac (Bryan), after reloading the plugin:
  - On a Ready task, `Ctrl+Shift+P` then `1` closes with the P1 notice.
  - On a Next task, `Ctrl+Shift+P` then `1` shows **Schedule task** with the cursor in
    Work summary, and `↵` closes.
  - `Enter` opens Schedule with the cursor in the date field.

## Out of scope and rejected alternatives

- **Routing card keys through Obsidian's modal `Scope`**
  (`scope.register(null, null, …)`). It sees keys typed into stage inputs, depends on
  keymap dispatch the harness cannot model, and the focusable root plus router covers
  the click case.
- **Adding jsdom as a test dependency.** `bob-plugins` tests are dependency-free plain
  `node --test`. The in-repo harness is extended instead. The jsdom repro was diagnostic
  only and is not committed.
- **Changing card key bindings, Work Log semantics or the decay card.**
