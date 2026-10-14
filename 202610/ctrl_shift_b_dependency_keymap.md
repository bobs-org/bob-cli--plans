---
tier: tale
title: Ctrl+Shift+B opens the Task Card's Depends on stage
goal:
  Ctrl+Shift+B adds or removes task dependency links exactly as Ctrl+Shift+P then `b`
  does, including Task Links and counts.
size: medium
decisions:
  command:
    ask:
      Should Ctrl+Shift+B get its own new command instead of rebinding
      `edit-task-dependencies`?
    choices:
      new:
        New `task-card-depends-on` command; `edit-task-dependencies` stays an unbound
        direct stage
      reuse:
        Rebind `edit-task-dependencies` to the Task Card `b` route; its palette behavior
        changes
    default: new
    why: Additive; leaves the documented, tested palette direct-entry contract alone
    answer: new
  refusal:
    ask:
      If `b` is unavailable (e.g. a non-#task bullet), keep the Task Card open after the
      notice, as `b` does?
    choices:
      keep: Show the notice and leave the Task Card open, exactly as pressing `b` does
      close:
        Show the same notice and close the card, so a refused chord leaves no modal open
    default: keep
    why: Bryan asked for Ctrl+Shift+B to behave just like Ctrl+Shift+P then `b`
    answer: close
proposed_by: bbugyi200.athena.10e
decided_by: reviewer
decided_via: tui
create_time: 2026-10-14 10:43:58
status: wip
---

# Ctrl+Shift+B: open the Task Card's Depends on stage in one chord

## Goal

Add a `Ctrl+Shift+B` keymap to Obsidian (bob-navigation-hotkeys) that behaves exactly as
if the user pressed `Ctrl+Shift+P` (open the Task Card) and then `b` (Blocked by /
Depends on). It adds or removes task dependency links on the task under the cursor, or
on the linked task when the cursor is on a Task Link bullet.

"Exactly" is the design rule. Do not build a second dependency entry point. Route the
chord through the same `openBulletPropertyPicker` entry that Ctrl+Shift+P uses, then
dispatch the same card intent that `b` resolves to. Everything Ctrl+Shift+P then `b`
does today then comes for free:

| Cursor                                                             | Ctrl+Shift+P then `b` today (and therefore Ctrl+Shift+B)                                                                                                                                                                             |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `#task` line                                                       | Task Card → Depends on value stage for this task; Back / empty Backspace returns to the card                                                                                                                                         |
| `^prj` task line                                                   | Same stage with the project side effects (§13 of `docs/task-dependencies.md`)                                                                                                                                                        |
| Anywhere on a Depends-On line                                      | Ctrl+Shift+P already skips the card (`initialProperty: "dependsOn"` direct stage) for the owning task                                                                                                                                |
| Dedicated Task Link bullet                                         | Link picker shell → resolves → `b` closes the card and calls `openLinkedDependencyStage(target)` on the linked task in its own note; with several linked targets it opens the first one and shows the "Opened the first of N" notice |
| Counted `N<chord>` (Vim normal mode)                               | Counted session (this task plus the next N) → counted Depends on stage                                                                                                                                                               |
| Inbox note `#task`                                                 | The card is armed with `inboxRoute`, so a commit asks where the task goes first                                                                                                                                                      |
| Row the review walk just landed on                                 | Review origin is captured, so a resolving commit advances the walk once (decision `answering-advances-the-walk`: "a resolving Task Card commit")                                                                                     |
| Non-`#task` bullet                                                 | `b` shows "Depends on requires a #task checkbox" (see decision `refusal`)                                                                                                                                                            |
| Not a bullet / prose with several links / selection spanning tasks | The same refusals Ctrl+Shift+P gives; no card opens                                                                                                                                                                                  |

## Where the work lives

- **bob-plugins** (linked repo): code, tests, README, and the manifest. Open it with
  your `/sase_repo` skill (`sase repo open bob-plugins -r "<why>"`) and read its
  `AGENTS.md` first. `plugins/bob-navigation-hotkeys` uses the fragment build. Edit
  `src/*.js`, run `npm run build`, and never hand-edit `main.js`. Each hand-edited
  fragment must stay at or below 1000 lines. `590-plugin-dependency-stage.js` (992
  lines), `610-plugin-jumps-and-counted-keys.js` (966), and
  `500-plugin-transclusion-and-link-picker.js` (911) are close to the cap. Keep
  additions there to a few lines. If a fragment would cross 1000, move code into a new
  fragment registered in `src/fragments.json` (and, for a plugin mixin, in
  `690-install-plugin-mixins.js`).
- **bob-cli** (this repo): user docs under `docs/`.

## Implementation (bob-plugins, `plugins/bob-navigation-hotkeys/src/`)

### 1. Plumb a one-shot `initialTaskCardRow` option

- `550-plugin-cancel-and-counted-property.js`, `openBulletPropertyPicker(cm, options)`:
  pass `initialTaskCardRow: options.initialTaskCardRow || null` into the
  `BulletPropertyPickerModal` context. After `picker.open()` succeeds, and only when
  `!options.initialProperty`, call the new picker method from step 2. `initialProperty`
  always wins. The Depends-On-line redirect recurses with
  `{ ...options, initialProperty: "dependsOn" }`, so that path must stay a direct stage
  and never auto-dispatch. Leave review-origin capture alone: it is skipped only for
  `initialProperty`, and the Ctrl+Shift+B route must capture it just like Ctrl+Shift+P
  does.
- `500-plugin-transclusion-and-link-picker.js`, `openLinkPicker(cm, options)`: the modal
  context there is built explicitly (it does not spread `options`). Add
  `initialTaskCardRow: options.initialTaskCardRow || null`. After the resolved session's
  `picker.showTaskCard({ rebuild: true })`, call the same picker method. Keys pressed
  while the link resolves are still swallowed by the shell, as today.
- `340-picker-core.js`: store
  `this.initialTaskCardRow = context.initialTaskCardRow || null`.

### 2. Dispatch the row intent exactly as the key does

Add a picker method, e.g. `dispatchInitialTaskCardRow()`, in `350-picker-task-card.js`
(774 lines, so there is room). It should:

- Do nothing unless `this.hasTaskCard`, `this.pickerOpen`, `this.stage === "task-card"`,
  a `taskCardModel` exists, `!this.linkResolving`, and `initialTaskCardRow` is set.
- Clear `this.initialTaskCardRow` before dispatching. It is one-shot, so Back /
  `returnToTaskCard()` and later rebuilds never re-trigger it.
- Find the model row whose `id` is the requested row id (`"depends-on"`). Run
  `this.dispatchTaskCardIntent(taskCardIntentForRow(row))` with the same
  `.catch(() => new Notice("Could not apply Task Card action"))` that
  `handleTaskCardKeydown` uses. That is precisely the intent `resolveTaskCardKey`
  returns for an unmodified `b`, so availability notices, counted sessions, link
  sessions (`openLinkedDependencyStage`), `^prj` side effects, and inbox routing all
  behave identically.

> [!decision] refusal = close When the dispatched intent is `type: "unavailable"`, show
> its notice the same way and then close the picker. Do not leave a card open that the
> user never asked to see. Under `refusal = keep` (default), leave the card open after
> the notice, exactly as `b` does.

### 3. Register the command and its default hotkey (`490-plugin-lifecycle.js`)

> [!decision] command = new Add, next to `set-bullet-property`:
>
> ```js
> this.addCommand({
>   id: "task-card-depends-on",
>   name: "Task card: Blocked by (Depends on)",
>   hotkeys: [{ modifiers: ["Ctrl", "Shift"], key: "B" }],
>   editorCallback: (editor) =>
>     this.openBulletPropertyPicker(editor, { initialTaskCardRow: "depends-on" }),
> });
> ```
>
> `edit-task-dependencies` and `openDependencyStageAtCursor` stay unchanged and stay
> unbound. The existing "registers with no default hotkey" test still holds.

> [!decision] command = reuse Point `edit-task-dependencies` at
> `this.openBulletPropertyPicker(editor, { initialTaskCardRow: "depends-on" })`, give it
> the Ctrl+Shift+B `hotkeys` entry, update its comment (drop "no default hotkey, Q6"),
> and delete `openDependencyStageAtCursor` once nothing calls it. Rewrite the test
> "stage edit-task-dependencies registers with no default hotkey" in
> `scripts/test-navigation-dependencies-stage-writes.cjs` to assert the Ctrl+Shift+B
> binding.

Use a plugin default hotkey, not a vault `hotkeys.json` edit. It follows the
Ctrl+Shift+M / Alt+N pattern, ships through `bob plugins sync`, and the vault's
`.obsidian/hotkeys.json` has no entry for either command id (an entry would override the
default). Neither that file, `obsidian_vimrc.md`, nor any installed plugin currently
binds Ctrl+Shift+B.

### 4. Counted `N<Ctrl+Shift+B>` in Vim normal mode (`610-plugin-jumps-and-counted-keys.js`)

Ctrl+Shift+P's counted form relies on the capture-phase fallback
`handleCountedBulletPropertyPhysicalKeydown`, which acts only on an explicit Vim count
and upgrades an already-open uncounted picker. Extend that same listener. Do not
register a second one.

- Let it also accept Ctrl+Shift+B (`event.code === "KeyB"` or `key` `b`/`B`) with the
  same modifier rules. A small helper can return which chord matched. Keep
  `isCountedBulletPropertyKeydown` true for both chords, or rename it, and update
  callers and tests.
- For the B chord, call
  `openBulletPropertyPicker(view.editor, { countExplicit: true, additionalTaskCount: N, initialTaskCardRow: "depends-on" })`.
  The P chord must keep passing exactly
  `{ countExplicit: true, additionalTaskCount: N }`.
- Everything else stays shared: the repeat guard, the WeakSet dedupe, the
  normal-mode-only check, and `resetPendingVimInputState`. A bare chord with no count
  still returns `false` and is left to the Obsidian command.

### 5. Version and build

- Bump `plugins/bob-navigation-hotkeys/manifest.json` `version` from 2.38.1 to
  **2.39.0** (feature). Add one sentence to its `description`, e.g. "Ctrl+Shift+B is
  Ctrl+Shift+P then `b`: it opens the Task Card's Depends on stage for the task, a Task
  Link's linked task, or a counted `N<Ctrl+Shift+B>` batch."
- Run `npm run build`.

## Tests (bob-plugins `scripts/`)

Add the tests to the closest existing files. If you create a new `scripts/test-*.cjs`,
add it to the `test` script in `package.json`.

1. **Registration**: the Ctrl+Shift+B command is registered with
   `hotkeys: [{ modifiers: ["Ctrl", "Shift"], key: "B" }]`. Its `editorCallback` calls
   `openBulletPropertyPicker(editor, { initialTaskCardRow: "depends-on" })`. Follow the
   `onload()` stubbing in `test-navigation-dependencies-stage-writes.cjs`.
2. **Parity on a `#task` line** (`test-navigation-task-card-view.cjs` style): the route
   lands in `stage === "value"` with
   `selectedPropertyItem.property.values === "local_task_id"`, `hasTaskCard === true`,
   and the input focused. Its result equals what `dispatchCardKey(modal, "b")` produces
   on a card opened by plain Ctrl+Shift+P. Also assert the dispatched intent deep-equals
   `resolveTaskCardKey(model, { key: "b" })`.
3. **Back returns to the card**: Backspace on the empty filter (or `returnToTaskCard()`)
   lands on `stage === "task-card"` without re-dispatching.
4. **Refusal**: on a plain (non-`#task`) bullet, the notice "Depends on requires a #task
   checkbox" appears. The card stays open (`refusal = keep`) or the picker closes
   (`refusal = close`).
5. **Depends-On line**: the chord opens the direct stage (`directStage === true`, no
   card), and the auto-dispatch never runs.
6. **Task Link**: after link resolution, `openLinkedDependencyStage` is called once with
   the resolved target and the card closes. Stub `resolveLinkPickerTargets` /
   `openLinkedDependencyStage` as
   `test-navigation-dependencies-stage-entry-and-view.cjs` does.
7. **Counted chord** (`test-navigation-hotkeys-vim-counts.cjs`): an explicit count with
   KeyB opens with
   `{ countExplicit: true, additionalTaskCount: 2, initialTaskCardRow: "depends-on" }`.
   KeyP's options are unchanged. A bare KeyB, a wrong Vim mode, wrong modifiers, and a
   repeated event all return `false` without preventing default.
8. **Review walk**: the Ctrl+Shift+B route calls `captureReviewGesture` exactly as
   Ctrl+Shift+P does. The direct-stage `initialProperty` route still does not.

Run `npm test` and `npm run validate` until both pass.

## Docs

bob-plugins `README.md`, Task Card section: add a `Ctrl+Shift+B` paragraph naming the
command id and palette name, stating that it is Ctrl+Shift+P then `b` (Task Links and
counted `N<Ctrl+Shift+B>` included). Make sure the "Direct entries" paragraph does not
imply that Ctrl+Shift+B skips the card.

bob-cli:

- `docs/task-dependencies.md` §6.1 entry-point table: add Ctrl+Shift+B rows for a
  `#task` line, a dedicated Task Link, and counted `N<Ctrl+Shift+B>` (or fold
  "Ctrl+Shift+P then `b`, or Ctrl+Shift+B" into the existing rows). Under
  `command = reuse`, update the palette row as well: it now opens through the Task Card
  and is bound to Ctrl+Shift+B. Also add Ctrl+Shift+B wherever that doc enumerates
  Depends on stage entry points (the BLOCKS writer table "nav Depends on stage (Task
  Card `b`, …)", PDG28).
- `docs/projects.md` Task Card section: mention Ctrl+Shift+B next to the `b` row. Under
  `command = reuse`, fix the "palette dependency" direct-entry wording.
- `docs/freshness.md` stamping table (the "Ctrl+Shift+P Task Card (… `b` …)" cell): note
  that Ctrl+Shift+B is the same card route.

No SASE memory note becomes false. The glossary entry for Task Dependency Link still
correctly says to use "Ctrl+Shift+P → Depends on", and decision records are immutable
and already cover "a resolving Task Card commit". So no memory edit is planned.

## Deploy and verify

1. In bob-plugins: run `npm test`, `npm run validate`, then `bob plugins sync` (required
   after any bob-plugins change).
2. Manual Obsidian check after reloading the plugin. In Settings → Hotkeys, confirm
   Ctrl+Shift+B shows on the new (or reused) command and conflicts with nothing,
   including Obsidian core defaults. Then try:
   - Ctrl+Shift+B on a `#task` line → Depends on stage; Backspace → card.
   - On a Pomodoro Task Link → linked note focused, stage on the linked task.
   - `2<Ctrl+Shift+B>` in Vim normal mode → counted stage.
   - On a Depends-On line → direct stage.
   - Insert mode and normal mode both work.
3. Commit each repo separately: bob-plugins
   (`feat(nav): add Ctrl+Shift+B Task Card Depends on keymap`) and bob-cli
   (`docs(task-dependencies): document Ctrl+Shift+B`).
