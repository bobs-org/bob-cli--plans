---
tier: tale
title: Make the bank Category card's hotkeys work
goal:
  Pressing a category hotkey, Space, Enter, or an arrow on the open bank Category card
  acts on the card even while the editor keeps focus, and no keystroke leaks into the
  note behind it.
size: medium
proposed_by: bbugyi200.apollo.e6
proposed_with: claude/opus
create_time: 2026-11-15 07:13:36
status: wip
---

# Plan: Make the bank Category card's hotkeys work

## Problem

On a waiting bank row, Ctrl+Enter opens the bank **Category card** (bob-plugins,
`bob-ledger-tools`). Clicking a category tile files the row. Pressing a tile's hotkey
(`c` Coffee, `g` Groceries, `1` Housing, …), Space, Enter, or the arrows does nothing.
Escape and a second Ctrl+Enter still work.

## Root cause (confirmed by reading the code)

`plugins/bob-ledger-tools/src/247-plugin-bank-category-card-ui.js`,
`BankCategoryCardModal.onOpen()`, listens for keys only with
`this.contentEl.addEventListener("keydown", this.keyListener, true)`. A DOM keydown
listener only fires when the focused element is `contentEl` or inside it. Nothing ever
puts focus there:

- The card opens from the editor (a Vim `<C-CR>` mapping → task-status-cycler →
  `api.bank.claimRowClose`), so focus stays in the CodeMirror editor, or ends up on
  `body` / `modalEl`. None of these is inside `contentEl`.
- `contentEl` has no `tabindex`, so `modal.contentEl.focus()` in
  `bankCategoryClaimRowClose` (`248-plugin-bank-category-card-actions.js`) does nothing.
- `paint()` empties and rebuilds `contentEl` on every change, so a tile button focused
  by a click is destroyed straight away.

So `onKeyDown` never runs for real keystrokes. The keys that still work go around it:

- **Escape** uses Obsidian's modal-scope binding (`onEscapeKey`).
- **Ctrl+Enter** goes from the editor's Vim mapping to `claimRowClose`, which calls
  `modal.onKeyDown` with a synthetic event.
- **Clicks** use a `click` listener whose target is the tile itself.

Worse, the letters go to the Vim editor behind the card (`d`, `s`, `x`, `u`, … are Vim
commands), so they can silently edit the bank note.

Tests never caught this. The unit tests
(`scripts/test-ledger-tools-bank-category-card-ui.cjs`) and the real-Obsidian
walkthrough (`scripts/obsidian-bank-ui-e2e.cjs`, the in-page `key()` helpers in the walk
and learn steps) all call `modal.onKeyDown(...)` directly. None of them sends a real
keystroke.

Other key-driven modals in the repo already avoid this. They register a catch-all on the
modal's own Obsidian `Scope`, which Obsidian's global key handler checks in the capture
phase whatever element has focus:

- `bob-navigation-hotkeys/src/103-copy-card-modal.js`
- `672-delete-confirm-modal.js`
- `207-image-zoom-modal.js`
- `bob-command-mode/src/176-sheet-keys.js`

The Category card should do the same.

## Fix (bob-plugins only; no bob-cli code changes)

Read the bob-plugins `AGENTS.md` first. Open the repo with `/sase_repo`
(`sase repo open bob-plugins`) and use the path it prints. Edit the `src/` fragments,
run `npm run build`, and never hand-edit `main.js`.

### 1. Route keys through the modal scope (`247-plugin-bank-category-card-ui.js`)

- In `onOpen()`, when `this.scope && typeof this.scope.register === "function"`,
  register one catch-all handler: `this.scope.register(null, null, handler)`. Keep the
  returned ref, for example as `this.keyScopeRef`. The handler:
  - passes the event to the card's existing key routing (`onKeyDown`). Its synchronous
    part already consumes the keys it acts on before any `await`.
  - then always consumes the event with `bankCategoryUiConsume(event)` and returns
    `false`, so **every** key is swallowed while the card is open. Keys the card does
    not bind (Tab, `j`, `x`, `u`, `z`, …) must never reach the editor behind it. This
    matches the Copy card and the delete-confirm modal.
- Escape keeps working through Obsidian's own modal-scope Escape binding, which runs
  before any later-registered handler, and the existing `onEscapeKey` override. The
  catch-all must not change that.
- Register the `contentEl` keydown listener **only** when no scope is available: the
  Node test harness has no Obsidian `Modal`, so the fallback base class has no `scope`.
  With exactly one route per environment, no key can be handled twice. In the scope
  path, make sure the `event.defaultPrevented` early-return in `onKeyDown` cannot drop a
  key the scope delivered. If needed, split the routing body into a helper that both
  paths call, and keep the `onKeyDown` name and behaviour for the tests and
  `claimRowClose`.
- In `onClose()`, unregister the scope handler
  (`this.scope.unregister(this.keyScopeRef)`, best-effort in try/catch). Remove the
  `contentEl` listeners as today. Keep the document `keyup` listener that the
  opening-chord logic uses.
- Do **not** steal focus into the card. After the card closes, the editor must still
  hold focus so the Vim `]b` walk and Ctrl+Enter keep working. Leave the `claimRowClose`
  API path alone; it stays as a fallback.

### 2. Guards that matter now that keys are live (same fragment)

- **Modifier guard.** In `bankCategoryUiEventKey`, keep the existing Ctrl+[, Ctrl+Enter,
  Ctrl+N, and Ctrl+P mappings. Then make any other one-character key pressed with Ctrl,
  Meta (Cmd), or Alt resolve to a non-matching token, for example
  `` `Mod+${event.key}` ``. `bankCategoryResolveCardKey` then returns `ignore`, and the
  scope handler swallows it. This way Cmd+C, Ctrl+C, or a Mac Alt-character never files
  a row. Shift+letter still files, as the resolver already lowercases it. Enter with
  modifiers is unchanged.
- **Auto-repeat guard.** A keydown with `event.repeat === true` must never file
  anything: no letter or digit `file`, no Space `no-category`, no Enter or Ctrl+Enter
  `accept`. A held key would otherwise file the next row after the walk advances.
  Swallow those repeats without acting. Arrow and Ctrl+N/P repeats still move the
  highlight. The existing opening-chord and landing-lock logic stays as is.

### 3. Unit tests (`scripts/test-ledger-tools-bank-category-card-ui.cjs`)

Build the modal with `makeModal`, give it a fake
`scope = { register(mods, key, fn) {…record…; return ref}, unregister(ref) {…} }` before
calling `modal.onOpen()`, and drive the recorded catch-all handler with events whose
`target` lies outside `contentEl`, as an editor element would. Cover:

1. One catch-all is registered as `(null, null, fn)`. With a scope present, no
   `contentEl` keydown listener is attached; the fallback `contentEl` exposes a
   `listeners` Map you can check.
2. A real-path `g` files `groceries` (`pick`). Digit hotkeys work too: add a
   digit-hotkey category to the fixture. Space files with no category, Enter accepts the
   highlight, and ArrowDown moves it. Each handled event is `preventDefault`ed and
   `stopPropagation`ed, and the handler returns `false`.
3. Unbound keys (`z`, `Tab`, `Shift`) are swallowed: prevented, stopped, handler returns
   `false`. Nothing is filed and the card stays open.
4. `c` with `ctrlKey`, `metaKey`, or `altKey` files nothing and is swallowed. Shift+`G`
   (`key: "G"`, `shiftKey`) still files `groceries`.
5. A `repeat: true` `g`, Space, or Enter files nothing. A `repeat: true` ArrowDown still
   moves.
6. While `busy` / `waitingRefresh` / `summary`, keys are swallowed with no action.
7. `onClose()` unregisters the scope ref.
8. Without a scope, the fallback `contentEl` keydown listener is attached, and sending
   an event through it still files. The existing `onKeyDown` tests keep passing
   unchanged.

### 4. Real-Obsidian walkthrough (`scripts/obsidian-bank-ui-e2e.cjs`)

The learn step (after `walk-card.png`) presses `c` and then `Enter` through the in-page
`key()` helper. Change it to use real CDP keystrokes:

- Explicitly focus the note editor (`view.editor.focus()`). Record that
  `document.activeElement` is outside the modal's `contentEl`.
- Send `c` with `client.call("Input.dispatchKeyEvent", …)` (keyDown + keyUp). Model it
  on `keyDispatchEvents` / `dispatchKey` in `scripts/obsidian-copy-card-e2e.cjs`, and
  set `text` for printable keys so a leaked key would really type. Then use an evaluate
  to poll for the learned `coffee` suggestion, as today, send a real `Enter`, and read
  both lines. Keep the `walk-learn-coffee` and `walk-learn-accept` record ids and
  assertions.
- Add a probe with a new record id, for example `walk-real-key-swallowed`. On a fresh
  card with the editor focused, send a real unbound printable key such as `z`. Assert
  the note text is byte-identical, nothing was filed, and the card is still open. Then
  close the card with Escape.
- Add the new record id and `Input.dispatchKeyEvent` to the offline `--self-test` token
  list, around the `for (const token of [...])` loop, so `npm test` checks that the
  real-key path exists.

The real CDP run needs the Obsidian Electron binary plus `xvfb-run`.
`docs/develop/validation.md` names athena. If the implementing host has neither, run
only the offline wrapper and say plainly that the real walkthrough was not run. A
pre-existing, unrelated `chip-today` failure in this e2e is tracked as bead
`bob-cli-le`; do not chase it.

### 5. Docs (`docs/bob-ledger-tools.md`, "Bank cards and the bank badge")

In the Category card paragraph, add one short sentence on the new contract: card keys
work while the editor keeps focus, keys the card does not bind (including
Ctrl/Cmd/Alt+letter) are swallowed and never edit the note, and a held key never files
twice. Run `npm run docs:check` with `BOB_CLI_DIR` set to your bob-cli checkout. No
manifest version bump is needed; fix commits in this repo don't bump it.

## Validation

From the bob-plugins checkout:

```bash
npm run build
node --test scripts/test-ledger-tools-bank-category-card-ui.cjs scripts/test-ledger-tools-bank-category-card.cjs scripts/test-obsidian-bank-ui-e2e.cjs
npm test
BOB_CLI_DIR=<bob-cli checkout> npm run docs:check
# only where Obsidian + xvfb exist:
xvfb-run -a node scripts/obsidian-bank-ui-e2e.cjs
```

Then deploy to this host's vault with `bob plugins sync -p bob-ledger-tools`. Follow the
AGENTS.md refusal procedure, and never pass `-F` on your own. Commit the bob-plugins
change through the normal finalization flow, with a message like
`fix(bank): route Category card keys through the modal scope`.

Bryan runs Obsidian on the Mac, and each host deploys its own plugins. The final report
must tell him that, once the commit lands, running
`bob plugins sync -p bob-ledger-tools` on the Mac (it `git pull`s first) and reloading
the plugin or Obsidian brings the fix there. It should also say that hotkeys pressed
before the fix may have reached Vim in the bank note behind the card, so a quick check
of the month note for stray edits is worth doing.

## Out of scope

- Moving focus into the card, or redesigning the card's key map.
- Changes to bob-cli, task-status-cycler, or bob-navigation-hotkeys.
- Changes to the Category card's write, advance, or refresh logic in
  `248-plugin-bank-category-card-actions.js`, except removing nothing that
  `claimRowClose` relies on.
