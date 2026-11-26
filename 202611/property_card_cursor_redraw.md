---
tier: tale
title: Keep the Vim cursor on its text when a property card changes height
goal:
  After `-` or `_` resizes a Live Preview property card, the Vim block cursor repaints
  on the line it is on.
size: medium
decisions:
  foot_walkthrough:
    ask:
      Should the card-foot walkthrough also gain the cursor checks, proving a record
      card too?
    choices:
      home: Prove it on Area and Project cards only; that walkthrough needs no bob build
      both:
        Also prove it on a record card; needs a bob built from bob-cli and a longer run
    default: home
    why: Every card species mounts through the one title-slot primitive this fix changes
    answer: home
proposed_by: bbugyi200.athena.1gf
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 06:03:28
status: wip
---

# Redraw the Vim cursor when a title-slot property card changes height

## Problem

Screencast `20261126_053704.mp4` shows an Area card (`gtd_daily`) in Live Preview with
Vim on. The cursor sits on the "Reflect on yesterday" task. When `-` collapses the
card's Properties drawer, the card shrinks by about 330 px and the text moves up. The
purple block cursor stays at its old screen row, which now covers "Check weather". The
relative line numbers still place the real head on "Reflect on yesterday". `k` then
lands on "Morning review", the line above, and the cursor repaints in the right place.
So the selection is correct and only the painted cursor is stale. `_` (fold card
details) changes the card height the same way.

## Root cause

This was checked against Obsidian 1.15.1's bundled CodeMirror and codemirror-vim
(`/snap/obsidian/current/app/resources/obsidian.asar`, `app.js`).

1. In Live Preview, property cards (album, record, bank, area, project, day and period
   cards) mount in the title slot. The slot is a direct child of `.cm-sizer`, placed
   before `.cm-contentContainer`, so it sits outside `.cm-content`. See
   `plugins/bob-ledger-tools/src/240e-plugin-property-card-title-slot.js` and
   `240k-plugin-property-card-title-adopt.js`.
2. Obsidian draws the Vim block cursor in a `.cm-cursorLayer.cm-vimCursorLayer` div
   appended to `.cm-scroller`. The cursor is absolutely positioned in scroller
   coordinates. Its ViewPlugin redraws only when a ViewUpdate has `selectionSet`,
   `geometryChanged`, or `viewportChanged` set, when Obsidian sends its internal refresh
   effect, or after codemirror-vim's `vim-command-done` and `vim-mode-change` events.
3. `-` and `_` are consumed in the capture phase by `handleSchemaDoubleEqualsKeydown`
   (`248m-plugin-schema-double-equals.js`). Vim runs no command, so `vim-command-done`
   never fires.
4. The card repaints later, after the key arm has returned:
   - `-`: `propertyCardToggleProperties` → `propertyCardFootToggle` → adapter refresh.
     The host rebuild then patches the same mounted node in place (`homeCardPatchLive`).
   - `_`: `propertyCardToggleDetails` → fold-kind toggle → `homeCardSetCollapsed` →
     `scheduleHomeCardRefresh`, then the same in-place patch.
5. The title slot's ResizeObserver sees the new height and calls
   `propertyCardTitleSlotMeasure`, which calls `view.requestMeasure(rescue)`.
6. CodeMirror's `ViewState.measure` sets geometry flags only when content size, padding,
   width, scale, or editor height changes. A shift of `.cm-content` inside the scroller
   sets no flag. With zero flags the measure's ViewUpdate is empty, so
   `EditorView.measure` skips `updatePlugins`. The Vim cursor layer keeps its old pixel
   position until the next selection change.

Any title-slot height change in Live Preview has this bug: `-`, `_`, mouse clicks on the
same controls, and async repaints that add or remove rows. The fold and drawer state is
correct; only the editor's cursor layer is out of date.

## Fix

All code changes are in the linked `bob-plugins` repo. Open it with `/sase_repo`
(`sase repo open bob-plugins`) and read its `AGENTS.md` first. Edit `src/` fragments
only, and regenerate `main.js` with `npm run build`.

In `plugins/bob-ledger-tools/src/240e-plugin-property-card-title-slot.js`:

- Add `propertyCardTitleSlotRedrawCursor(view)`. It calls
  `view?.cm?.state?.vimPlugin?.blockCursor?.scheduleRedraw?.()` inside `try/catch` and
  returns whether it asked for a redraw.
  - `EditorView.cm` is the codemirror-vim adapter. bob-plugins already reaches it as
    `view.editor.cm.cm`.
  - `state.vimPlugin.blockCursor.scheduleRedraw()` is codemirror-vim's own redraw entry
    point. It only queues the cursor plugin's identity-deduplicated measure request: no
    transaction, no selection change, and no history entry.
  - When the hook is missing (Vim off, a future Obsidian), the call does nothing and
    never throws.
- Call it from the Live Preview branch of `propertyCardTitleSlotMeasure`, before the
  measure-budget gate. That branch already runs for fresh insertion, observer
  re-insertion, and every real height change (`propertyCardTitleSlotResized` returns
  early when the height is unchanged), so one call site covers `-`, `_`, clicks, and
  async repaints for every species.
  - It sits outside the budget so a fast burst of toggles cannot leave the cursor stale.
  - It cannot feed a loop: redrawing the cursor layer never changes the card's size.
  - Reading mode never calls it.
  - Leave the existing rescue `requestMeasure`, its budget, and the scroll anchoring as
    they are.
- Export the helper through `374a-exports-property-card-title-slot.js` if the unit tests
  call it directly.
- Update the fragment's header comment and the "Title slot" section of
  `docs/develop/architecture.md`. Say that Live Preview insertions and resizes also
  repaint the Vim cursor, and explain why: content moves without a CodeMirror geometry
  change.

The `-` and `_` key arms in `248m` stay unchanged. They return before the card repaints,
so a redraw there would run before the text moves.

Rejected approaches:

- **Re-dispatch the unchanged selection.** It uses only public API, but it would wake
  about 30 `selectionSet` decoration rebuilders across Bob plugins, plus
  codemirror-vim's external-selection handling, on every card resize.
- **Call `requestMeasure()` or the adapter's `cm.refresh()` alone.** Both run a measure
  that produces an empty update, which is the bug itself.
- **Add a fallback that re-dispatches the selection when the Vim hook is absent.** Out
  of scope. `-` and `_` act only in Vim normal mode. CodeMirror's own selection layer
  (Visual mode) and non-Vim cursors are not touched, and reaching them would need the
  selection re-dispatch above.

No accepted decision governs property-card drawing or the editor cursor layer. The
decisions roster was checked. bob-cli needs no change.

## Tests

### Unit (`scripts/test-ledger-tools-property-card-title-slot.cjs`)

- **Mount and resize.** Mount in Live Preview with a fake `EditorView` whose
  `cm.state.vimPlugin.blockCursor.scheduleRedraw` counts calls. Expect one redraw at
  insertion and one more for each ResizeObserver fire that changes the card height.
  Same-height fires cause no redraw.
- **Budget.** Make more height-changing resize fires than
  `PROPERTY_CARD_TITLE_SLOT_MEASURE_BUDGET` within one window. Every fire still redraws
  the cursor, while `requestMeasure` calls stay within the budget, as the existing loop
  test asserts.
- **Observer re-insertion.** When the observer re-inserts a displaced node, the cursor
  is redrawn.
- **Fail-safe.** A Reading mount never redraws. An editor with no `cm`, with no
  `vimPlugin`, or whose `scheduleRedraw` throws never throws, and the mount stays in
  place.
- Keep every existing title-slot test green.

### Real Obsidian (`scripts/obsidian-home-cards-e2e.cjs`, its `-lib.cjs`, and the offline wrapper)

- **Fixture.** Give the long-area fixture (or a new dedicated area fixture) several
  plain body lines, so the Vim cursor can sit on a text line below the card.
- **Cursor probe.**
  - Get the `EditorView` from `app.workspace.activeLeaf.view.editor.cm`.
  - Take `head = state.selection.main.head`.
  - The expected top is `coordsAtPos(head, 1).top`, the same coordinate codemirror-vim
    uses.
  - The drawn top is the rect top of
    `.cm-vimCursorLayer .cm-fat-cursor.cm-cursor-primary`.
  - Also record the card's height.
- **Sequence.** In Live Preview, put the cursor on a body line below the card with Vim
  keys and record a baseline. Then run:
  - Area card: `-` open, `-` close, `_` collapse, `_` expand.
  - Project card: `_` collapse and `_` expand, with the cursor on a body line below that
    card.
- **After each step.** Wait for the existing state probe, then two animation frames.
  Assert:
  - the head offset is unchanged;
  - the card height changed, so the check is not vacuous;
  - the drawn top is within 2 px of the expected top.
- **Check ids.** `minus-open-cursor-tracks-text`, `minus-close-cursor-tracks-text`,
  `underscore-collapse-cursor-tracks-text`, `underscore-expand-cursor-tracks-text`, and
  `underscore-project-cursor-tracks-text`. The offline wrapper
  (`scripts/test-obsidian-home-cards-e2e.cjs`) asserts the runner contains them.
- **Red, then green.** Run the real walkthrough once on the unfixed build and confirm
  the new checks fail, with a delta about the size of the card's height change. Then run
  it on the fixed build, where they pass. Report both results.

> [!decision] foot_walkthrough = both Also add the same cursor probe to
> `scripts/obsidian-card-foot-e2e.cjs` (and its offline wrapper), after its existing
> Live Preview `-` step on the record note and around a `_` fold on that card. Run it
> with `BOB_BIN` and `BOB_CLI_DIR` from a bob built from the current bob-cli checkout,
> as `docs/develop/validation.md` describes.

## Validation and ship

1. In bob-plugins, run `npm run build`.
2. Bump `plugins/bob-ledger-tools/manifest.json` from `1.166.4` to `1.166.5`, following
   the repo's patch-bump convention for fixes.
3. Run `npm run docs:gen`, `npm run build:check`, `npm test`, and `npm run validate`.
4. Update the "Area and Project card walkthrough" paragraph in
   `docs/develop/validation.md` to list the cursor-tracking checks. Read the `docs.md`
   reference memory before editing docs.
5. Run the real home-cards walkthrough on athena from a fresh `/tmp` copy under
   `xvfb-run`, using the command in `docs/develop/validation.md`. Review the report and
   every screenshot.
6. Run `bob plugins sync`. If it refuses, follow bob-plugins' `AGENTS.md`.
7. Manual check in the vault: open an area note in Live Preview with Vim, put the cursor
   on a task below the card, and press `-` and `_` repeatedly. The block cursor must
   stay on its text line every time.
