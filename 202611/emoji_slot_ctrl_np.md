---
tier: tale
title: Fix Ctrl+N / Ctrl+P in the Schema Card Emoji Slot
goal:
  Ctrl+N / Ctrl+P in the Emoji Slot picker step one emoji at a time in the order shown,
  wrap at the ends, and keep the highlight in view.
size: small
decisions:
  ends:
    ask: Should Ctrl+N / Ctrl+P wrap around at the ends of the emoji results?
    choices:
      wrap:
        Ctrl+N on the last emoji lands on the first, and Ctrl+P on the first lands on
        the last
      stop: Ctrl+N and Ctrl+P stop on the first and last emoji, like the arrow keys
    default: wrap
    why: Command Mode, the copy card, and the bank Category card all wrap Ctrl+N/P
    answer: wrap
proposed_by: bbugyi200.athena.1dp
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 17:00:28
status: wip
---

# Fix Ctrl+N / Ctrl+P in the Schema Card's Emoji Slot picker

## Problem

The album Schema Card's Emoji Slot picker (the "Search emoji" panel that opens from the
dashed ☻ well in an Add/Edit choice stage) shows its results as an 8-column grid.
Bryan's screencast shows a search for `paper` with 11 results in two rows. The highlight
starts on the last result (🖇 paperclips, index 10). One Ctrl+P jumps it to index 2 (🗞
newspaper, a whole row up). A second Ctrl+P jumps it to index 0. After that, further
presses do nothing. Ctrl+N / Ctrl+P never step one emoji at a time, and they get stuck
at the ends.

Root cause, in the bob-plugins linked repo (open it with `/sase_repo`:
`sase repo open bob-plugins -r "..."`):

- `plugins/bob-ledger-tools/src/242f-plugin-emoji-slot-model.js`,
  `albumEmojiSlotMoveHighlight(index, key, count, columns)`, treats `p`/`P` as `ArrowUp`
  (`-columns`) and `n`/`N` as `ArrowDown` (`+columns`), then clamps to `[0, count-1]`.
  Ctrl+N/P are therefore row jumps that clamp at the ends. That breaks the Bob-wide
  convention: Ctrl+N/P "move through completions in the order shown" and wrap
  (`docs/bob-command-mode.md`, `docs/copy-card.md`, and `bankCategoryMoveHighlight` in
  `245-plugin-bank-category-card-model.js`, the closest grid analogue).
- `plugins/bob-ledger-tools/src/242g-plugin-emoji-slot-dom.js`,
  `albumEmojiSlotHandleKeydown`, detects the chord loosely
  (`event.ctrlKey && ["n","N","p","P"].includes(event.key)`), so Ctrl+Shift/Alt/Meta
  variants also move. It passes the raw `event.key` to the model, so `"n"` means "down a
  row". After a move, nothing scrolls the active option into view, even though
  `.bob-emoji-slot-panel` scrolls (`max-height: min(17em, 35vh); overflow: auto`) and
  can hold up to 64 results (`ALBUM_EMOJI_SLOT_RESULT_LIMIT`). Once Ctrl+N walks or
  wraps through a long result list, the highlight would leave the visible area.

The schema card's own `schemaOnKeyDown` (`241y-plugin-album-schema-card.js`) returns
early for focused inputs, and the bank setup card's `bankEmojiSlotInterceptKey`
(`242h-plugin-bank-emoji-slot.js`) does not consume Ctrl+N/P. So neither card intercepts
these keys. The fix belongs only in the shared Emoji Slot. The bank setup card's Emoji
Slot gets the same fix automatically.

## Target behavior

- **Ctrl+N** moves the highlight to the next emoji in the order shown (reading order
  across sections and grid rows, the same order as `data-emoji-option-index`).
  **Ctrl+P** moves it to the previous emoji.

> [!decision] ends = wrap Ctrl+N on the last result goes to index 0, and Ctrl+P on index
> 0 goes to the last result.

> [!decision] ends = stop Ctrl+N on the last result and Ctrl+P on index 0 leave the
> highlight where it is, but the event is still consumed (no caret movement in the
> search field).

- Only the exact chord counts: Ctrl with no Alt, Meta, or Shift. Match on `event.key`
  (`n`/`N`/`p`/`P`) or `event.code` (`KeyN`/`KeyP`), so Caps Lock and non-QWERTY layouts
  work, like `cmCommandModeKeyAlias` in
  `plugins/bob-command-mode/src/176-sheet-keys.js`. A matched chord always calls
  `preventDefault` + `stopPropagation`, as today. That keeps the macOS emacs text
  bindings (Ctrl+N/P move the caret) from firing in the search input.
- The arrow keys keep their current two-dimensional, clamped behavior. Left and Right
  step ±1; Up and Down step ±8 columns (the docs say "Arrows move the highlight in two
  dimensions").
- After any keyboard move (arrows or Ctrl+N/P), the active option is scrolled into view
  inside the panel with `scrollIntoView({ block: "nearest" })`. Guard the call with
  optional chaining in a `try`/`catch`, like the existing `241yd`/`241xb` call sites. Do
  not scroll on plain repaints, such as typing in the search field or switching category
  tabs.
- Enter, Tab, Escape, `Ctrl+[`, Backspace-on-empty, `q`-types, and the footer and
  `aria-activedescendant` updates stay exactly as they are.

## Implementation

1. **Model (`242f-plugin-emoji-slot-model.js`).** Have `albumEmojiSlotMoveHighlight`
   take a normalized move token instead of overloading `n`/`p`:
   - `"ArrowLeft"`/`"ArrowRight"` → ±1, `"ArrowUp"`/`"ArrowDown"` → ±columns, all
     clamped (unchanged).
   - New `"next"`/`"previous"` → ±1, wrapping per the `ends` decision (with
     `ends = stop`, clamp).
   - Drop the `n`/`N`/`p`/`P` mappings. Keep the empty-list `-1` return and the
     out-of-range `index` normalization.
   - Add a small pure helper, `albumEmojiSlotMoveKey(event)`. It returns one of the four
     arrow keys, `"next"`, `"previous"`, or `null`, and implements the exact-Ctrl
     `key`/`code` matching described above. Export both from
     `plugins/bob-ledger-tools/src/352-exports-album.js`, next to the existing
     `albumEmojiSlotMoveHighlight` export.
2. **DOM (`242g-plugin-emoji-slot-dom.js`).** In `albumEmojiSlotHandleKeydown`, replace
   the `isMove` expression with `albumEmojiSlotMoveKey(event)`. When it returns a token
   and `slot.optionRows.length > 0`: call `stop()`, compute `slot.activeIndex` with the
   token, call `albumEmojiSlotRenderResults(slot)`, then scroll the active option
   (`#bob-emoji-slot-${slot.id}-option-${slot.activeIndex}`, found inside
   `slot.results`) into view. Put the scroll in a tiny
   `albumEmojiSlotRevealActive(slot)` helper so the render path stays scroll-free. Keep
   each hand-edited fragment at or below 1000 lines (both are far below).
3. **Build and version.** Run `npm run build` to regenerate
   `plugins/bob-ledger-tools/main.js`. Never hand-edit it. Bump
   `plugins/bob-ledger-tools/manifest.json` by one patch version from whatever is
   current at implementation time (it is `1.130.0` while this plan is written),
   following the repo's `fix(...)` convention.
4. **Docs (`docs/albums.md`, Emoji Slot paragraph).** Replace "Arrows move the highlight
   in two dimensions;" with wording that also covers the new keys, for example: "Arrows
   move the highlight in two dimensions, `Ctrl+N` / `Ctrl+P` step to the next or
   previous emoji in the order shown and wrap at the ends, and the highlight stays in
   view;". With `ends = stop`, say "stop at the ends" instead. No other docs page
   describes Emoji Slot movement keys. The bank line in `docs/bob-ledger-tools.md` only
   mentions closing the slot, so leave it.

## Tests

In `scripts/test-ledger-tools-album-schema-card-shell.cjs`:

- **Model assertions.** Add these to the Emoji Slot tests, or as a new focused test (use
  `helpers.albumEmojiSlotMoveHighlight`):
  - `"next"` from 7 of 11 → 8, crossing a row.
  - `"previous"` from 10 of 11 → 9. This is the screencast case: it used to jump to 2.
  - `"next"` from 10 of 11 → 0 and `"previous"` from 0 of 11 → 10 (with `ends = stop`:
    10 and 0).
  - Arrows unchanged: `"ArrowDown"` from 0 of 40 → 8 (the existing assertion stays),
    `"ArrowUp"` from 2 of 11 → 0, `"ArrowRight"` from 10 of 11 → 10.
- **`albumEmojiSlotMoveKey` cases.** `{ key: "n", ctrlKey: true }` → `"next"`,
  `{ key: "P", ctrlKey: true }` → `"previous"`, and
  `{ key: "т", code: "KeyN", ctrlKey: true }` → `"next"`. Each of
  `{ key: "n", ctrlKey: true, altKey: true }`, `metaKey`, `shiftKey`, and a plain
  `{ key: "n" }` → `null`.
- **DOM regression.** Open a choice stage with `emojiApiFixture()` and type a query, or
  use the empty-query browse, so there are at least three options. Dispatch
  `{ type: "keydown", key: "n", ctrlKey: true, code: "KeyN", ... }` on
  `.bob-emoji-slot-search`. Assert all of the following:
  - The event is prevented.
  - `aria-activedescendant` / `aria-selected` move from option 0 to option 1, and the
    footer names the new emoji.
  - Ctrl+P from option 0 lands on the last option (per `ends`).
  - The stage and modal stay open, with nothing staged or applied.
  - The revealed option had `scrollIntoView` called with `{ block: "nearest" }`. The
    harness `FakeNode` has no `scrollIntoView`, so stub it on `FakeNode.prototype` for
    this test and restore it afterward.
  - Typing into the search (an `input` event) does not call it.

In `scripts/test-ledger-tools-bank-emoji-slot.cjs`, add one assertion that the shared
slot also steps by one there. Ctrl+N inside an open bank slot's search moves
`activeIndex` from 0 to 1, and `bankEmojiSlotInterceptKey` still reports the key handled
without filing or saving. Add it only if the existing fake harness can build an open
slot cheaply. Otherwise the album shell test already covers the shared code.

## Validation and deploy

All commands run inside the bob-plugins checkout:

```bash
npm run build
npm run build:check
node --test scripts/test-ledger-tools-album-schema-card-shell.cjs scripts/test-ledger-tools-bank-emoji-slot.cjs
npm test
npm run validate
npm run docs:inventory:check
BOB_CLI_DIR=<path to this agent's bob-cli checkout> npm run docs:check
bob plugins sync
```

`bob plugins sync` is mandatory after any bob-plugins change (repo AGENTS.md). If it
refuses to overwrite another checkout's deploy, follow AGENTS.md: never pass `-F` on
your own; check `git log --oneline HEAD..@{upstream} -- plugins/bob-ledger-tools` and
follow "When sync refuses an overwrite" in `docs/develop/deployment.md`.

If an unrelated pre-existing `npm test` failure shows up, confirm it also fails on an
unmodified tree before attributing it elsewhere, and report it rather than fixing it
here.

Optional manual check in Obsidian on the Mac after sync:

1. Open a Schema Card, start an Add choice stage, and open the Emoji Slot.
2. Search `paper` and press Ctrl+N repeatedly. The highlight should walk 🧻 → 📰 → 🗞 →
   … one emoji at a time, wrap from 🖇 back to 🧻, and stay visible.
3. Ctrl+P should walk backward. Arrows should still move by row and column.

## Out of scope

- Changing arrow-key semantics, Enter/Tab/Escape behavior, result ranking, or the
  8-column layout.
- Other pickers' Ctrl+N/P handling (Command Mode, relation card, copy card, and the bank
  Category card already step and wrap).
