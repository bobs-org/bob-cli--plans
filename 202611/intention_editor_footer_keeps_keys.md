---
tier: tale
title: Keep the intention editor's keymap footer on j/k
goal:
  Browsing the intention card editor with j/k (and other selection moves) never hides
  the keymap footer.
size: small
decisions:
  summary_explainer:
    ask:
      Should `s` (select Summary) also leave the keymap footer in place instead of
      showing its explainer?
    choices:
      explainer:
        "`s` keeps its 'Summary selected…' line until the next key; j/k then restores
        the keys"
      footer:
        "`s` keeps the keymap footer like other moves; ownership still shows inside the
        Summary stage"
    default: explainer
    why: The report is about j/k; `s` is a documented hint that the next move now clears
    answer: explainer
proposed_by: bbugyi200.athena.1ev
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 10:11:30
status: wip
---

# Plan: Keep the intention card editor's keymap footer on j/k

## Problem

In the intention card editor (`==` on a daily or period note → "EDIT INTENTIONS"), the
line under the rows starts as the keymap list:

`j/k move · Enter edit · o add · r reflect · Alt+J/K reorder · Ctrl+D remove · u undo · [ ] = travel · y YAML`

After one `j` or `k`, that line shows only the newly selected row's label ("Intention
2", "Reflection", …), and the keymap list does not come back while you keep browsing.
The screencast `~/tmp/screencasts/20261121_100019.mp4` shows this (keys `j`, `j k k`,
`j`, then `Ctrl+[`).

## Root cause (confirmed)

All of this lives in the **bob-plugins** linked repo, plugin `bob-ledger-tools`.

- The status line is painted in
  `plugins/bob-ledger-tools/src/297cc-plugin-intention-card-editor-paint.js`
  (`intentionCardEditorPaintAll`, the `bob-intention-editor-status` element) as
  `[announce, refusalMessage, refusalHint].filter(Boolean).join(" · ") || intentionCardEditorStatusFallback(model)`.
  The keymap list (`INTENTION_CARD_EDITOR_BROWSE_KEYS` in
  `297ca-plugin-intention-card-editor-input.js`) is only the fallback, shown when
  nothing is announced or refused.
- `IntentionCardModal.intentionSelect(key)` in
  `plugins/bob-ledger-tools/src/297c-plugin-intention-card-editor.js` (around line 291)
  sets `this.announce = target.label;`. Every browse move goes through it: j/k,
  ArrowUp/Down, Tab/Shift+Tab (`intentionMove`), digits 1–9 (`select-digit:`), row
  clicks (`intentionOnClick`), and `s` (`select-summary`). Once `announce` holds a
  label, the fallback never paints, so the keymap list is gone.
- This line has been there since the editor was first added
  (`feat(bob-ledger-tools): add guarded intention card editor`). It looks like a
  screen-reader announcement, because the status element has `role="status"` and
  `aria-live="polite"`. It adds nothing, though: `intentionCardEditorFocus` focuses the
  selected row's `<button>` (which also carries `aria-current="true"`) after every
  paint, so the focused row's text is already announced.
- I reproduced it with the existing node harness
  (`scripts/intention-card-test-helpers.cjs`, set up as in
  `scripts/test-ledger-tools-intention-card-editor-dom.cjs`). The status text is the
  keymap list when the editor opens, `Intention 2` after `j`, and `Intention 1` after
  `k`.

## Change

Open the linked repo with your `/sase_repo` skill
(`sase repo open bob-plugins -r "<reason>"`), use only the path it prints, and read its
`AGENTS.md` before editing. Its rules apply: edit `src/` fragments, never hand-edit the
generated `main.js`, run `npm run build`, then `bob plugins sync`.

### 1. Selection stops announcing (the fix)

In `plugins/bob-ledger-tools/src/297c-plugin-intention-card-editor.js`, change
`intentionSelect(key)` so a selection clears the transient announcement instead of
setting it to the label:

```js
intentionSelect(key) {
  const target = intentionCardEditorTargetByKey(this.intentionTargets(), key);
  if (!target) return;
  this.selectedKey = key;
  // A move is not an outcome: the focused row already says where you are,
  // so the status line returns to the browse keys.
  this.announce = "";
  this.paintIntentionCard();
}
```

What this means for the user:

- j/k, arrows, Tab/Shift+Tab, 1–9, and row clicks keep the keymap footer visible.
- An outcome line such as `✓ Saved · u undo`, `↶ Undone · u redoes`,
  `No change — the note already says that.`, `Nothing to undo.`, or
  `Retry discarded · nothing was written` stays until the next move. The move then
  brings the keymap list back, where before it was replaced by a row label.
- Refusal, read-only, and needs-update text is not touched: `refusalMessage` and
  `refusalHint` stay. Before, a move put a row label in front of them; now it does not.
  The read-only fallback (`Read-only · [ ] = travel · y YAML`) still paints when there
  is no refusal text.
- Stage keys, `intentionConsumeApplied` (which sets `selectedKey` directly and announces
  Saved), travel (`297e-plugin-intention-card-navigation.js`, which already clears
  `announce`), and the write and undo paths stay as they are.

### 2. `s` (select Summary) follows the decision

`intentionHandleAction("select-summary")` calls `this.intentionSelect("summary")` and
then sets its own explainer `announce`.

> [!decision] summary_explainer = explainer No code change for `s`. After step 1,
> `intentionSelect` clears `announce` and the `select-summary` branch then sets the
> explainer, so `s` shows "Summary selected · Enter explains ownership before opening
> its edit stage." until the next key. A later j/k brings the keymap list back.

> [!decision] summary_explainer = footer In the `select-summary` branch, remove the line
> that sets the explainer `announce` and the second `paintIntentionCard()` that only
> repaints for it, keeping just `this.intentionSelect("summary")`. Leave the Summary
> stage's ownership note (`INTENTION_CARD_EDITOR_SUMMARY_EXPLAINER` via `stageNote` in
> `intentionOpenStage`) as it is. The `s` row in the `docs/intention-card.md` gesture
> table stays accurate, because Enter still explains ownership inside the stage.

### 3. Regression tests

Add tests to `scripts/test-ledger-tools-intention-card-editor-dom.cjs`. It is already in
`npm run test:unit` and its `setup()`, `card()`, `rows()`, and `key()` helpers fit.
Update its header comment so it mentions the footer. Read the status text from
`card(modal).querySelector(".bob-intention-editor-status").textContent`, and compare it
against the keymap list, either by exact text or by matching
`/^j\/k move · Enter edit · /`.

1. **Browse moves keep the keymap footer.** Seed `{ intentions: ["One", "Two"] }` and
   `await flush()`. Assert the status is the keymap list. Then, one at a time, send `j`,
   `k`, `ArrowDown`, `Tab`, `Tab` with `shiftKey: true`, and digit `2`, and select a row
   through the click path
   (`modal.intentionOnClick({ target: <row button>, preventDefault() {} })` if the fake
   DOM's `closest` handles it; otherwise call `modal.intentionSelect("reflection")`).
   After each one, assert that the selection moved (`aria-current="true"` on the
   expected `data-intention-key` row) and that the status is still the full keymap list,
   not a row label.
2. **A move after an outcome brings the keymap list back.** Produce an outcome
   announcement. The simplest is `u` with nothing to undo, then `await flush()`, since
   undo is async (`Nothing to undo.`). A saved edit (`i`, type, Enter →
   `✓ Saved · u undo`) also works, using the simulator as the other tests do. Assert the
   outcome text shows, press `j`, and assert the status is the keymap list again.
3. **`s` follows the decision.** For `explainer`: after `s`, the status contains
   `Summary selected`, and after a following `j` it is the keymap list again. For
   `footer`: after `s`, the status is the keymap list and the Summary row is selected.

The fragment stays far below the 1000-line limit (757 lines now).

### 4. Docs

`docs/intention-card.md` (Card editor section) is the plugin reference for this behavior
and is mounted into the Bob site. After the gesture table, add one sentence to the
paragraph that starts "Stage keys stay text…", or put it just before that paragraph:

> The line under the rows always lists the browse keys; moving the selection never
> replaces it, and an outcome such as Saved or Undone stays only until the next move.

If the decision is `explainer`, add "(`s` briefly explains Summary ownership)". Keep the
gesture table as it is.

### 5. Version, build, verify, deploy

1. Bump the `bob-ledger-tools` manifest patch version
   (`plugins/bob-ledger-tools/manifest.json`, currently `1.149.0` → `1.149.1`; if master
   has moved, bump the patch of the current value).
2. Run `npm run build` to regenerate `plugins/bob-ledger-tools/main.js`, then
   `npm run build:check`.
3. Run the focused intention suites:
   `node --test scripts/test-ledger-tools-intention-card.cjs scripts/test-ledger-tools-intention-card-editor.cjs scripts/test-ledger-tools-intention-card-editor-dom.cjs scripts/test-ledger-tools-intention-card-writes.cjs scripts/test-ledger-tools-intention-card-overlay.cjs scripts/test-ledger-tools-intention-card-navigation.cjs scripts/test-ledger-tools-intention-card-acceptance.cjs scripts/test-ledger-tools-intention-card-title-block.cjs`.
4. Run the full `npm test`. If it runs longer than a foreground timeout, hand it to
   `/sase_monitor` instead of waiting on it.
5. Run `npm run docs:check` (bob-cli's documentation gate).
6. Run `bob plugins sync` to deploy into the vault. If it refuses an overwrite, follow
   the bob-plugins `AGENTS.md` procedure exactly: never pass `-F` on your own.

## Acceptance

- In the editor, pressing j/k (and arrows, Tab, 1–9, or clicking a row) moves the
  selection, and the footer still reads
  `j/k move · Enter edit · o add · r reflect · Alt+J/K reorder · Ctrl+D remove · u undo · [ ] = travel · y YAML`.
- After a save or undo, the outcome line shows until the next move, and then the keymap
  list returns.
- Refusal, read-only, stale-retry, and stage behavior are unchanged, and no row label is
  put in front of refusal text.
- `s` behaves as the `summary_explainer` decision says.
- New DOM tests fail before the fix and pass after it. `npm run build:check`, the
  intention suites, the full `npm test`, and `npm run docs:check` pass.
  `bob plugins sync` deploys the new `bob-ledger-tools` version.

## Out of scope

- No bob-cli, Bob Mac Capture, or vault data changes.
- No change to other panels' footers or to the shared `bobCard` close-key wiring.
- No visually hidden live region. Row focus already gives screen readers the selection.
