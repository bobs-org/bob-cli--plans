---
tier: tale
title: Fix Ctrl+N / Ctrl+P navigation in the successor offer picker
goal:
  Ctrl+N / Ctrl+P move the successor offer picker's highlight like ↓ / ↑, as in every
  other Bob picker.
size: small
decisions:
  footer_hint:
    ask: Should the successor picker footer also advertise the new ^N / ^P keys?
    choices:
      skip:
        Footer stays exactly as today; Ctrl+N/P work but stay unadvertised, like ↑/↓
        already are
      add:
        Every footer variant gains a `^N ^P move` chip after Tab; one more chip in a
        long footer
    default: skip
    why:
      Footer already has 6-7 chips and never showed ↑/↓; other Bob pickers already teach
      Ctrl+N/P
    answer: skip
proposed_by: bbugyi200.athena.1a9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 06:26:47
status: wip
---

# Plan: Fix Ctrl+N / Ctrl+P navigation in the successor offer picker

## Diagnosis (confirm / deny)

**Confirmed:** Ctrl+N / Ctrl+P do not move the highlight in the successor offer picker
("Pull in successors of …", "Keep ‹project› moving", relation successor groups, the
project-promotion card, and the relation-merge offer pick, which all open the same
modal).

**Denied, in part:** it is not intermittent. In this modal the keys _never_ work, on
every platform and in every state. It only feels unreliable because:

- `↓` / `↑` and `Tab` ("toggle and move") do move the highlight here, and a Ctrl+X /
  Ctrl+Shift+X verdict also auto-advances it, so the highlight often moves near a Ctrl+N
  press.
- Almost every other Bob picker supports Ctrl+N / Ctrl+P: the Task Card and all its
  stages, the relation label stage, the block-id-prompt pickers, and the cycler demotion
  picker. Muscle memory works in the card before and after this one.

**Root cause:** in the `bob-plugins` linked repo,
`plugins/bob-navigation-hotkeys/src/206-successor-picker-modal.js` defines
`SuccessorOfferPickerModal extends FilteredPickerModal` and overrides
`handleKeydown(event)` completely. It handles Ctrl/Cmd+Shift+Enter, Esc / Ctrl+[,
verdict keys (Ctrl+X), Enter, Ctrl+A, Tab, `ArrowDown`, and `ArrowUp`, and then returns
without ever calling `super.handleKeydown(event)`. Ctrl+N / Ctrl+P live only in the base
handler (`plugins/bob-navigation-hotkeys/src/200-filtered-picker-modals.js`,
`FilteredPickerModal.handleKeydown`:
`event.key === "ArrowDown" || isCtrlKey(event, "n")`, and the same for `"p"`), so the
override silently drops them.

The keydown is never `preventDefault`-ed, so it falls through to the native `<input>`.
On macOS that is Cocoa's moveDown / moveUp caret motion in a one-line field: nothing
visible happens. The modal has had this gap since it was introduced in `0d69a20`
("grouped successor offer picker"). Its footer also replaces the base `KEYBOARD_HINTS`
(which advertise `^N ^P Move`), so the gap never showed in the UI.

Evidence gathered while planning:

- Audit: about 20 `FilteredPickerModal` subclasses override `handleKeydown` (inbox
  route, parent note, pomodoro split, task split, themes manager and actions, relation
  label management, the Task Card keydown mixin, the move pickers). Every one except
  `SuccessorOfferPickerModal` ends by delegating unmatched keys to
  `super.handleKeydown(event)`. This is a single outlier, not a systemic bug.
- A throwaway reproduction used the existing `scripts/modal-harness.cjs` with the
  `scripts/test-navigation-successor-picker.cjs` setup and a 3-row model. `ArrowDown`
  and `Tab` moved `selectedIndex` 0 → 1 with `defaultPrevented: true`. `Ctrl+N` and
  `Ctrl+P` left it at 0 with `defaultPrevented: false`.
- No OS-level remap is involved: the chezmoi Hammerspoon config has no Ctrl+N/P or
  Obsidian bindings, the vault `hotkeys.json` has no plain Ctrl+N/P, and no third-party
  Obsidian plugin remaps them.

## Implementation

All code changes are in the `bob-plugins` linked repo. Open it with the `/sase_repo`
skill (`sase repo open bob-plugins -r "<reason>"`) and use only the path it prints. Read
that repo's `AGENTS.md` first. `bob-navigation-hotkeys` uses the committed source build:
edit `src/` fragments only, never `main.js` by hand, and keep each fragment at or below
1000 lines (206 is 975 lines today).

### 1. Delegate unmatched keys to the base picker (the fix)

In `plugins/bob-navigation-hotkeys/src/206-successor-picker-modal.js`,
`SuccessorOfferPickerModal.handleKeydown`:

- Replace the trailing `ArrowDown` and `ArrowUp` branches with one final
  `super.handleKeydown(event);` call. Add a short comment saying that unmatched keys go
  to the base picker for its navigation (↑/↓ and Ctrl+N/Ctrl+P), as every other
  `FilteredPickerModal` subclass does, and that Enter never reaches the base because the
  Enter branch above always returns.
- The base calls `this.moveSelection(±1)`, which resolves to the successor override.
  That override wraps, updates the status line for the new highlight, and re-renders, so
  behavior is identical to today's arrow branches.
- Check that each earlier branch (Ctrl+Shift+Enter, Esc / Ctrl+[, `handleVerdictKey`,
  Enter, Ctrl+A, Tab) still `return`s, so nothing it consumes reaches the base. In
  particular, the base `Enter` → `openSelectedItem` path must stay unreachable, because
  this modal has no `openItem`.
- Keep the existing `escArmed` disarm before the delegation unchanged. Ctrl+N/P disarm
  the two-step Esc, just as ↑/↓ do today.
- Net effect: Ctrl+N / Ctrl+P (and ↑/↓, unchanged) move the highlight, wrap at the ends,
  land on `+N more` expand rows like ↓ does, are `preventDefault`-ed, and never toggle
  marks or verdicts. Cmd+N, Ctrl+Alt+N, and so on are still not consumed, because
  `isCtrlKey` requires no Meta or Alt.

Do not touch `handleVerdictKey`, the reason editor (it has its own `<input>` and key
listener, so Ctrl+N/P typed there never reach this handler), or the
`successorInstallVerdictModalPatches` prototype patches.

> [!decision] footer_hint = add Add `{ keys: ["^N", "^P"], label: "move" }` right after
> the `Tab toggle and move` entry in `SUCCESSOR_VERDICT_FOOTER_BASE` and in the
> catch-path fallback of `successorVerdictFooterHints`
> (`src/206d-successor-picker-verdict-render.js`). Add it in the same position in the
> static `SUCCESSOR_PICKER_FOOTER_HINTS` and `SUCCESSOR_PICKER_VERDICT_HINTS` in 206.
> Add a test assertion that the rendered footer shows the chip in both the plain and the
> verdict-active picker. With `skip`, the footer code and its tests stay unchanged.

### 2. Regression tests

Write the tests first and confirm they fail before step 1. In the reproduction, `Ctrl+N`
left `selectedIndex` at 0 with `defaultPrevented: false`.

In `scripts/test-navigation-successor-picker.cjs` (already in `npm test`; use its
`makePlugin`, `modelFor`, `openPicker`, and `pressKey` helpers):

- **Ctrl+N / Ctrl+P move the highlight like ↓ / ↑.** Use `modelFor(3)`:
  - `pressKey(modal, "n", { ctrlKey: true })` moves `selectedIndex` 0 → 1. The event is
    `defaultPrevented`, the `is-highlighted` / `aria-selected="true"` row is the second
    key, and `markedKeys.size` stays 0.
  - A second Ctrl+N goes to 2. A third wraps to 0. Ctrl+P wraps back to 2, and another
    Ctrl+P goes to 1.
  - An uppercase `"N"` with `ctrlKey` (Caps Lock) also moves.
  - `metaKey` + `n` and `ctrlKey` + `altKey` + `n` do not move and are not prevented.
  - Finally, after advancing the injected clock past the 350 ms Enter guard, Enter
    resolves `{ picks: [{ key: <the highlighted row's key> }] }`. This proves the
    highlight Ctrl+N/P set is the one Enter pulls in.
- **Ctrl+P reaches the collapsed group's expand row.** Use
  `modelFor(20, { group: "project" })`. From the first row, Ctrl+P wraps to the last
  navigation item, which has `kind === "expand"`, exactly as `ArrowUp` does. Enter there
  (past the guard) expands the group.
- Keep the existing ArrowDown / ArrowUp / Tab assertions passing unchanged. They now
  exercise the delegated path.

In `scripts/test-navigation-successor-verdicts.cjs` (use its `verdictModel` and
`openPicker`):

- **Verdict mode keeps Ctrl+N / Ctrl+P.** Use `verdictModel(3)`. Ctrl+X stages Done on
  row 0 and auto-advances to row 1. Ctrl+P then returns the highlight to row 0 and
  Ctrl+N moves it to row 1 again. Both are prevented, and `modal.verdicts` and
  `modal.markedKeys` stay unchanged.

### 3. Build, version, docs

- Run `npm run build` to regenerate `plugins/bob-navigation-hotkeys/main.js`.
- Bump the patch version in `plugins/bob-navigation-hotkeys/manifest.json` (2.76.0 →
  2.76.1 at planning time; bump whatever is current). Do not edit the manifest
  `description`.
- Run `npm run docs:gen` to refresh the generated version column in `docs/index.md`.
- In `docs/bob-navigation-hotkeys.md` (the plugin reference, mounted into the Bob site),
  in the successor picker paragraph that begins "In the successor picker,
  `Ctrl+Shift+Enter` retargets …", add one sentence before it: "`↑` / `↓` and `Ctrl+N` /
  `Ctrl+P` move the successor picker's highlight without marking." No bob-cli contract
  change is needed: `docs/task-dependencies.md` §12.13 does not list the picker's
  navigation keys.

### 4. Validate and deploy

In the bob-plugins checkout, run:

- `node --test scripts/test-navigation-successor-picker.cjs scripts/test-navigation-successor-verdicts.cjs`
- `npm test` (includes `build:check` and the docs inventory check)
- `npm run docs:check`
- `npm run validate`

Then run `bob plugins sync` to deploy to the vault, as that repo's `AGENTS.md` requires.

## Acceptance

- In the successor offer picker, Ctrl+N / Ctrl+P move the highlight down / up with wrap,
  in plain and verdict-active pickers. Enter pulls in the highlighted row. Marks,
  verdicts, Tab, Ctrl+A, Ctrl+X, Esc, Enter, and Ctrl+Shift+Enter behave exactly as
  before.
- The new tests fail on the pre-fix source and pass after it. The full `npm test`,
  `npm run docs:check`, and `npm run validate` are green.
- `main.js` is regenerated, not hand-edited. The manifest patch version and
  `docs/index.md` agree. The plugin is synced to the vault.
