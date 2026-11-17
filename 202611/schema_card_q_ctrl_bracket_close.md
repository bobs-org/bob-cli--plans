---
tier: tale
title: q and Ctrl+[ close the album Schema and bank Setup cards like Esc
goal:
  q and Ctrl+[ back out of and close the album Schema card and bank Setup card exactly
  as Esc does, from any focus, and a decision record makes that the rule for every Bob
  panel.
size: medium
decisions:
  hint_q:
    ask:
      Should q leave Move to… hint mode the way Esc does, by dropping q from the
      gap-badge alphabet?
    default: true
    why:
      The Copy card and bob's bank hotkeys already reserve q for closing, so a key never
      changes meaning
    answer: true
  memory_close_keys_decision:
    ask:
      Should the coder add a decisions-web record saying every panel Esc closes also
      closes with q and Ctrl+[?
    memory:
      - decisions
    default: true
    requested:
      Also, add a new decision memory that states that all panels that support the
      `<esc>` keymap to close that panel should also support the `q` and `<ctrl+[>`
      keymaps.
    answer: true
proposed_by: bbugyi200.apollo.el
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 11:12:26
status: wip
---

# q and Ctrl+[ close the album Schema card and the bank Setup card like Esc

## Goal

The album **Schema** card (opened from an album note's header, `==`, the palette, or
Command Mode) and the bank **Setup** card on `~/bob/bank.md` must accept `q` and
`Ctrl+[` wherever they accept `Esc`. The two new keys do exactly what `Esc` does at that
moment: back out one layer, then close. Also record that rule as an accepted decision
for every Bob panel.

## Root cause (verified while planning)

Both cards live in the `bob-plugins` linked repo (open it with `/sase_repo`), plugin
`bob-ledger-tools`, in `src/` fragments built into `main.js`.

- **Esc works everywhere** because Obsidian binds `Escape` on the modal's keymap scope
  to `onEscapeKey`. That scope sees every keydown while the modal is on top, wherever
  focus is. The comments in `241y-plugin-album-schema-card.js` (`onEscapeKey`) and in
  block-id-prompt's `087-task-wikilink-picker-modal.js` record this. The scope runs
  before any `contentEl` listener.
- **Album Schema card** (`241y-plugin-album-schema-card.js`, `schemaOnKeyDown`): it has
  no `q` or `Ctrl+[` handling at all.
- **Bank Setup card** (`248k-plugin-bank-setup-card.js`, `onKeyDown`): it handles `q`
  (outside inputs) only from a `contentEl` capture listener. It has no `Ctrl+[`
  handling. If focus is not inside the card's content (the editor behind the modal, the
  modal frame, or the body after a click on chrome), `q` never arrives. In the editor
  case it reaches CodeMirror Vim, which starts macro recording.
- `BankSetupCardModal` extends Obsidian's `Modal` directly and does **not** override
  `onEscapeKey`. In real Obsidian, Esc therefore closes the whole card at once, even
  from a stage or during a lock wait. That contradicts its own hint strip (`esc cancel`,
  `esc stop waiting`, `esc revert`), its footer (`Esc/q cancel stage`), and
  `docs/bob-ledger-tools.md` ("Esc stops unwritten"). Giving all three keys one dispatch
  fixes this as a side effect.
- Precedent for the fix: the bank Category card (`247-plugin-bank-category-card-ui.js`,
  commit `f72bd4e` "route Category card keys through the modal scope") registers a
  catch-all `this.scope.register(null, null, handler)` in `onOpen` and unregisters it in
  `onClose`, so its keys work with editor focus. The Copy card (commit `c9d4572`) and
  the Task Card (commit `31e5c0e`) already close on `q` and `Ctrl+[`.

## The rule to implement (both cards)

- **Keys.** `Esc` (unchanged). Bare `q`/`Q`: Shift allowed; never with Ctrl, Meta, or
  Alt. `Ctrl+[`: Ctrl alone, no Alt/Meta/Shift, matched by
  `event.code === "BracketLeft"` **or** `event.key === "["`. Ignore composition
  (`isComposing` or `keyCode === 229`).
- **Same ladder as Esc.** `q` and `Ctrl+[` call the card's existing Esc back-out path.
  They never get a parallel implementation that could drift.
- **Text exception (q only).** `q` stays literal text when the focused element is
  **inside the panel** and accepts typed characters: `input`, `select`, `textarea`,
  `[contenteditable]` (not `"false"`), the Emoji Slot search, and the Emoji Slot well
  `[data-emoji-well]` (type-to-search). Focus outside the panel, for example the editor
  behind the modal, is **not** a text context: `q` closes and is consumed, so Vim never
  sees it. `Ctrl+[` has no text exception and backs out from inside fields too.
- **Hint mode.** While Move to… gap badges show (arrange hint mode, or the add-ghost
  hint mode from Alt+M), letters are badge keys, not text. Under `hint_q = true`, `q`
  leaves hint mode exactly like Esc, even with focus in a stage input.

> [!decision] hint_q = no Keep `SCHEMA_ARRANGE_GAP_ALPHABET` unchanged. While hints
> show, `q` goes to the badge handler as today. `Esc` and `Ctrl+[` leave hint mode.
> Outside hint mode, `q` follows the rule above. Skip the alphabet, test, and doc
> changes in step 4.

- **Focus-independent.** `q` and `Ctrl+[` reach the card through the modal's keymap
  scope, like Esc, as well as through the existing `contentEl` path. The second path
  keeps the Node tests (whose Modal fallback has no scope) and any non-scope host
  working. Whichever path handles the event first consumes it (`preventDefault` +
  `stopPropagation`, plus `__schemaEscapeHandled` on the album card), and the other path
  sees `defaultPrevented` and does nothing.
- **Pass-through.** The new scope handler returns a non-`false` value without touching
  the event for every key except `q` and `Ctrl+[`. Obsidian's own `Escape` binding is
  registered at Modal construction, so it keeps matching first, as it does for the
  Category card's catch-all.

## Steps

### 1. Shared close-key helper (new fragment)

Add `plugins/bob-ledger-tools/src/241xc-plugin-card-close-keys.js` and list it in
`src/fragments.json` right after `241xb-plugin-schema-arrange-dom.js`. That slot comes
before both cards (`248k*` and `241y*`). It holds DOM-free helpers only:

- `bobCardCloseKey(event)` returns `"escape" | "q" | "ctrl-bracket" | null` per the Keys
  bullet above.
- `bobCardTypesText(event, panelEl)` is true only for a focused text control inside
  `panelEl` (use `event.target`, falling back to `panelEl.ownerDocument.activeElement`),
  per the text exception.
- `bobCardWireCloseKeys(modal, { backOut, hintActive })` registers the catch-all on
  `modal.scope` when `modal.scope?.register` exists and stores the ref.
  `bobCardUnwireCloseKeys(modal)` unregisters it. The handler:
  1. Pass through unless the kind is `q` or `ctrl-bracket`.
  2. For `q`, pass through when `!hintActive()` and
     `bobCardTypesText(event, modal.modalEl)`.
  3. Otherwise consume the event, call `backOut(event)`, and return `false`. Wrap
     everything in `try`, as the Category card does.

Export these through a small exports fragment (for example
`370-exports-card-close-keys.js`, appended to `fragments.json`) or an existing exports
fragment, so tests can reach them via `helpers`. Keep every hand-edited fragment at or
below 1000 lines (`241y` is ~913 and `241yb` ~939 today, so put new bulk in the new
fragment).

### 2. Album Schema card

- In `241y-plugin-album-schema-card.js`, factor one `schemaBackOut(event)` from
  `onEscapeKey`: run `schemaHandleEscape()`, mark the event handled, and `close()` on
  `"close"`. `"ignore"` and `"stay"` keep the card open. `onEscapeKey` keeps its
  `return false` contract and calls it. Replace the three `event.key === "Escape"`
  branches in `schemaOnKeyDown` (in-input, stage, browse) with the close-key predicate
  plus the q text exception, all calling `schemaBackOut`.
- In `onOpen`, call
  `bobCardWireCloseKeys(this, { backOut: (e) => this.schemaBackOut(e), hintActive: () => Boolean(this.schemaHintMode) })`.
  Unwire in `onClose`, including the arrange wrapper's early-return close-during-save
  path in `241yb` (`onCloseArrange`).
- `241yb-plugin-album-schema-arrange.js` (`schemaOnKeyDownArrange`): the hint-mode
  `event.key === "Escape"` branch becomes the close-key predicate, with `q` included
  under `hint_q = true`. Its badge-letter branch must not claim `Ctrl+[`.
- `241ye-plugin-album-schema-place.js` (`schemaOnKeyDownPlace`): the ghost hint-mode
  letter branch excludes close keys, so `q` (under `hint_q = true`) and `Ctrl+[` fall
  through to the back-out. Drag cancel already lives in `schemaHandleEscapePlace`, so it
  comes for free.
- The Emoji Slot (`242g-plugin-emoji-slot-dom.js`): `albumEmojiSlotHandleKeydown` treats
  `Ctrl+[` like `Escape` (close only the panel, focus the well). `q` in the search types
  as before. The `schemaHandleEscapeEmojiSlot` wrapper already closes the panel first.
- Every `schemaHandleEscape` wrapper (arrange, place, emoji slot) stays the single
  ladder: All actions list → drag → ghost hints → hints → emoji panel → pending moves →
  stage → draft → close, with `"ignore"` while applying. Do not reorder it.

### 3. Bank Setup card

- In `248k-plugin-bank-setup-card.js`, factor the current Esc/q branch of `onKeyDown`
  into `bankSetupBackOut()`. Lock wait → stop waiting and repaint. `busy` → ignore.
  Otherwise `cancelStage()`, or `close()` when there is no stage. Add
  `onEscapeKey(event)`: consume, call `bankSetupBackOut()`, and `return false`. In
  `onKeyDown`, route Esc, `Ctrl+[`, and `q` (outside text and hint mode) to
  `bankSetupBackOut()`, keeping the existing `defaultPrevented` guard. Wire the scope
  handler in `onOpen` with `hintActive: () => Boolean(this.bankHintMode)`, and unwire in
  `onClose`.
- `248ka-plugin-bank-setup-card-arrange.js`: move the hint-mode leave and the
  dirty-arrangement revert out of `bankArrangeOnKeyDown` and into a `bankSetupBackOut`
  prototype wrapper (hint mode → leave; dirty, no stage, not applying →
  `bankArrangeRevert`; else the original). The keydown wrapper then only routes close
  keys to `this.bankSetupBackOut()`. The hint-mode letter branch must not claim close
  keys.
- `242h-plugin-bank-emoji-slot.js`: wrap `bankSetupBackOut` so an open slot closes first
  (`albumEmojiSlotClose(slot, { focusWell: true })`). `bankEmojiSlotInterceptKey` treats
  `Ctrl+[` like `Escape`. While the slot is open, `q` in its search still types.
- Leave the Category card (`245`–`248h`) alone; it already handles all three keys.

### 4. Gap-badge alphabet (`hint_q = true` only)

In `241xa-plugin-schema-arrange-model.js`, drop `q` from `SCHEMA_ARRANGE_GAP_ALPHABET` →
`"asdfghjklwertyuiopzxcvbnm"` (25 letters, home-row first). Both cards share it through
`arrangeGapHintLabels`. Update the `gap labels are home-row-first…` test in
`scripts/test-ledger-tools-schema-arrange.cjs`: 25 one-letter labels, two-character
labels past 25, still prefix-free, and no label contains `q`.

### 5. Hint strips and footers

- In `241yc-plugin-album-schema-card-regions.js` (`albumSchemaCardHintItems`) and
  `248kb-plugin-bank-setup-card-regions.js` (`bankSetupCardHintItems`), the
  plain-browsing **close** chip reads `esc/q`. Leave every other `esc` chip (cancel,
  revert, leave hint mode, stop waiting) as `esc`, since `q` types in stage inputs.
- The bank footer in `248k` (`footerText`) already says `Esc/q`. Keep it. Update any
  test that pins the chip text.

### 6. Tests (extend existing files; they are already in `test:unit`)

Give the modal a fake scope before `open()`
(`{ register(m, k, fn) { … return ref; }, unregister(ref) { … } }`) and call the
captured handler directly. Also drive the `contentEl` / `onKeyDown` path with no scope.
Cover:

- Predicate: `q`, `Q` (Shift), and `Ctrl+[` by `key` and by `code` are accepted.
  Ctrl/Alt/Meta+`q`, `Ctrl+Shift+[`, `Ctrl+Alt+[`, `Meta+[`, and composing events are
  rejected. `bobCardTypesText` is false for a target outside the panel.
- Album (`scripts/test-ledger-tools-album-schema-card-shell.cjs` and/or
  `…-album-schema-arrange-card.cjs`, `…-album-schema-place-drag.cjs`):
  - From browsing, with the target outside the card (the editor), `q` closes and
    `Ctrl+[` closes. Both are consumed.
  - In a stage with focus in an input, `q` passes through (no `preventDefault`; the
    stage stays). `Ctrl+[` cancels the stage and the card stays; a second `Ctrl+[`
    closes.
  - With the Emoji Slot open, `q` in the search types, and `Ctrl+[` closes only the
    panel.
  - In hint mode, `q` (`hint_q = true`) and `Ctrl+[` leave hint mode without picking a
    gap.
  - Pending moves are reverted, an open All actions list closes, a drag cancels, and
    `"applying"` ignores the keys exactly like Esc.
  - Unrelated keys (`j`, `a`, `Enter`) pass through the scope handler untouched.
  - `onClose` unregisters the handler.
- Bank (`scripts/test-ledger-tools-bank-setup.cjs`, `…-bank-setup-arrange.cjs`,
  `…-bank-emoji-slot.cjs`): the same matrix, plus lock wait (`q`/`Ctrl+[`/`onEscapeKey`
  stop waiting and the card stays open), a stage cancelled by `onEscapeKey` (not
  closed), and close from browsing. Keep the existing "q in an input types instead of
  cancelling the stage" assertion.
- Optional real-Obsidian check: if Obsidian and `xvfb-run` are available on the host
  (see `docs/develop/validation.md`; athena has them), extend
  `scripts/obsidian-album-ui-e2e.cjs` and `scripts/obsidian-bank-ui-e2e-setup.cjs` with
  a `q` / `Ctrl+[` close probe dispatched with focus outside the card. If you cannot run
  it, say so in the final report. It is not a gate.

### 7. Docs (bob-plugins)

- `docs/albums.md`: the Esc ladder paragraph (the one starting "The card is
  keyboard-first…") and the arrange paragraph ("`esc` leaves hint mode, then reverts…")
  say **Esc, `q`, or `Ctrl+[`**, with the caveat that `q` types inside fields and the
  Emoji Slot well while `Ctrl+[` always backs out. The Emoji Slot sentence becomes "the
  first Esc or Ctrl+[ closes only the picker". Under `hint_q = true`, the gap-badge
  sentence becomes "letters except `q` … two characters past 25 gaps; `q` leaves hint
  mode".
- `docs/bob-ledger-tools.md` (the bank Setup paragraph): add that Esc, `q`, and `Ctrl+[`
  back out one layer (stop waiting, close the Emoji Slot, leave hints, revert moves,
  cancel the stage) and then close, and work with editor focus.
- Run `npm run docs:gen`, then `npm run docs:inventory:check`. If bob-cli is reachable,
  also run `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`.

### 8. Decision record (bob-cli; `memory_close_keys_decision = true`)

Use `/sase_memory_write` first. Add
`sase/memory/decisions/q-and-ctrl-bracket-mirror-esc.md`, copying the frontmatter shape
of a recent sibling strand (keyword title, `aliases`, one-sentence `description` roster
summary, `origin: requested`, project scope).

- **Title:** "q And Ctrl+[ Mirror Esc In Every Panel".
- **Aliases:** `q closes`, `ctrl+[ closes`, `close key parity`, `esc q ctrl-bracket`.
- **Roster summary:** "Every Bob panel that Esc closes or backs out of also does so with
  q and Ctrl+[, through Esc's own dispatch and regardless of focus; q types only in the
  panel's text controls, Ctrl+[ never does, and no panel binds q to anything else."
- **Body** (same sections as the other records):
  - **Applies to.** Every panel Esc dismisses: Obsidian modals, cards, pickers, and
    in-card panels across bob-plugins, and Bob Mac Capture panels.
  - **Claim.** The rule above: aliases of Esc's current layer, the q text exception,
    Ctrl+[ meaning Ctrl alone, focus independence, and q reserved (no hint badge,
    hotkey, or row key may claim it).
  - **Why.** Quote Bryan's request. Vim habits: `q` closes help, quickfix, and pager
    windows, and Ctrl+[ is Esc. Repeated per-panel fixes show opt-in drifts. Rejected
    alternatives:
    - per-panel opt-in;
    - `q` closing outright even from a stage, which loses Esc's layering;
    - `q` closing inside text fields;
    - `q` as a hint badge, which changes meaning by mode. Evidence: bob-plugins
      `c9d4572` (Copy card), `31e5c0e` (Task Card Ctrl+[), `f72bd4e` (Category card keys
      through the modal scope), bead `bob-cli-a1` (image zoom closes on Esc, q, Ctrl+[),
      bob's reserved bank hotkey `q` (`src/native/bank/config.rs` `RESERVED_HOTKEY`),
      and this plan.
  - **Cost.** `q` can't be a panel hotkey or hint badge, every panel needs a
    keymap-scope route, and text-control detection must be kept current.
  - **Reopens when.** A panel needs `q` as a content key, Ctrl+[ should type in fields,
    or a panel's Esc must do something other than back out.

Then run `sase memory init` and include the regenerated `AGENTS.md`, shims, and memory
README in the commit.

> [!decision] memory_close_keys_decision = no Skip step 8. Because Bryan requested the
> record, a tale coder files a `memory` task bead with `/sase_new_task` only when no
> human reviewed the plan.

### 9. Follow-up bead for other panels

Once the rule exists, grep `bob-plugins` for other panels that handle `Escape` or
override `onEscapeKey` but lack `q` or `Ctrl+[` parity. Examples: the block-id-prompt
task wikilink picker, the delete-confirm modal, and the Pomodoro/priority pickers. Check
each candidate before listing it; image zoom, the Copy card, the Task Card, the inbox
route picker, and the successor picker already conform. Use `/sase_new_task` to file
**one** task bead (type `bug`) listing the non-conforming panels with file paths. Note
that Bob Mac Capture was not audited. Do not fix them in this change.

### 10. Build, version, validate, deploy

- Bump `plugins/bob-ledger-tools/manifest.json` by one patch version.
- In bob-plugins, run `npm run build`, `npm run build:check`, `npm test`, and
  `npm run validate`, and fix all failures. Never hand-edit `main.js`.
- Run `bob plugins sync`, following the repo's `AGENTS.md` refusal procedure if sync
  refuses. Commit both repos (bob-plugins: code, tests, docs, and the generated
  `main.js`; bob-cli: the memory strand and regenerated files).

## Acceptance criteria

- On both cards, `q` (outside text controls) and `Ctrl+[` (anywhere) do exactly what Esc
  does at that moment, including when focus is in the editor behind the modal. Neither
  key ever reaches the note or Vim while the card is open.
- `q` still types in every text control and the Emoji Slot well/search.
- Esc on the bank Setup card now backs out layer by layer, as its hint strip says,
  instead of closing from a stage or during a lock wait.
- Under `hint_q = true`, no gap badge is `q`, and `q` leaves hint mode.
- The scope handler leaves every other key untouched, and it is unregistered on close.
- `npm test` and `npm run validate` pass. Every hand-edited fragment is ≤1000 lines. The
  plugin is synced to the vault.
- Under `memory_close_keys_decision = true`, the decisions roster lists the new record
  after `sase memory init`.
- One follow-up bead lists the other non-conforming panels.

## Out of scope

- Fixing other panels (tracked by the step 9 bead).
- Changing Esc on the album Schema card (already layered) or on the bank Category card.
- Bob Mac Capture code.
