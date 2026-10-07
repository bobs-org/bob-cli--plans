---
tier: tale
title: Keep the cursor out of folded date-mark whitespace
goal:
  Editing next to a Live Preview date mark (vim cw, Backspace, a cursor between Tasks'
  double spaces) types text in order, before the mark, because no date-mark replace
  range ever hides a selection endpoint.
size: small
proposed_by: bbugyi200.athena.0xt
create_time: 2026-10-07 10:52:44
status: wip
---

# Plan: Keep the cursor out of folded date-mark whitespace

## Problem

In Live Preview (vim mode), take the task line `- [ ] #task foo [created:: 2026-10-07]`,
which renders as `#task foo + today`. Typing `cwbar` to replace `foo` with `bar` shows
`#task + today rab`. The text lands after the date mark, with its letters reversed.

## Root cause (confirmed)

The cause is the whitespace-run fold that the date marks added (bob-cli-53), working
against the per-field reveal check. Both live in the linked `bob-plugins` repo, under
`plugins/bob-ledger-tools/src/`:

- `136-date-marks.js` `dateMarkFoldLength` folds the **whole** run of spaces before a
  field. `266-plugin-date-marks.js` `buildDateMarkDecorations` then emits
  `Decoration.replace` over `[fieldStart - foldLength, fieldEnd)`.
- The reveal check only hides the mark while a selection overlaps
  `[fieldStart, fieldEnd]`. Folded spaces are excluded by design.
- So when `foldLength >= 2`, every position strictly between the range start and
  `fieldStart` is hidden inside the replaced range, and putting the cursor there never
  reveals the field.

Vim `cw` deletes `foo` but keeps the trailing space. The line becomes
`- [ ] #task  [created:: 2026-10-07]` (two spaces), with the cursor at offset 12,
between the two spaces. The fold run is 2, so the decoration is `[11, 35)` and the
cursor sits strictly inside it. Reproduced with the existing date-mark test harness:
`buildDateMarkDecorations` returns `replace 11 35`, foldLength 2, cursor 12 inside.

CodeMirror cannot draw a caret inside a replaced range. A widget maps any interior
position to the DOM point after the widget, so the browser inserts each typed character
at the field's end, while the editor's selection stays at the hidden offset behind it.
Every keystroke therefore lands in front of the previous one, which produces `rab` after
`+ today`.

The same hidden-cursor state also happens:

- without vim: Backspace over the last word before a field (`#task foo| [created…]`
  becomes `#task | [created…]`);
- on Tasks-written double-spaced `  [completion:: …]` and `  [cancelled:: …]` fields,
  whenever the cursor stops between the two spaces (vim `h`/`l`, clicks, and so on).

Freshness and priority marks are not affected. They fold exactly one space, so every
interior position of their range is inside the field span and reveals. Rendered views
never fold.

## Fix (design)

New invariant: **a date-mark replace range never has a selection endpoint strictly
inside it.**

- Per-field reveal stays exactly as it is: folded spaces still never reveal the field.
- When any selection range's `from` or `to` lies strictly inside
  `(fieldStart - foldLength, fieldStart)`, that field's mark is emitted with fold 0. The
  range covers just the field span, so the spaces render as real text, the cursor sits
  in real text, and the mark stays visible.
- The fold returns as soon as the cursor leaves the run. After typing `b`, the run is
  one space and the cursor sits at its start, which is the range boundary, so typing
  continues in order before the mark.
- `dateMarkShouldRebuild` already rebuilds on `selectionSet`, so no new rebuild trigger
  is needed.

Rejected alternatives:

- **Reveal the raw field while the cursor is in the run.** It would flash
  `[created:: …]` for one keystroke on every `cw`/Backspace, and it contradicts the
  contract's rule that folded spaces never reveal.
- **Fold at most one space,** like the fresh and priority marks. That reverts the
  decided uniform gap for Tasks' double-spaced fields.
- **Shrink the fold so it starts at the furthest interior endpoint.** It is more
  complex, still hides some spaces, and gives no visual gain over fold 0.

## Changes

### bob-plugins

Open the repo with `sase repo open bob-plugins` and read its `AGENTS.md`. Edit `src/`
fragments only; never hand-edit the generated `main.js`.

1. `plugins/bob-ledger-tools/src/136-date-marks.js`: add a pure, synchronous,
   never-throwing helper,
   `dateMarkSelectionFold(fieldStart, fieldEnd, foldLength, selectionRanges)`, which
   takes absolute document offsets and works like this:
   - Returns `null` (reveal) when any range overlaps the field span inclusively
     (`from <= fieldEnd && to >= fieldStart`). This is the current rule, moved
     unchanged.
   - Otherwise returns `0` when any range's `from` or `to` lies strictly inside
     `(fieldStart - foldLength, fieldStart)`.
   - Otherwise returns `foldLength`, coerced to a non-negative integer the same way
     `DateMarkWidget` coerces it.
   - Skips ranges without numeric `from`/`to`, and treats a non-array `selectionRanges`
     as empty.
   - Returns `0` on an unexpected error, because no fold can ever hide a cursor.
   - Gets a comment above it, matching the fragment's style, that explains why: the
     interior of a replace decoration cannot hold a caret.
2. `plugins/bob-ledger-tools/src/266-plugin-date-marks.js` `buildDateMarkDecorations`:
   - Replace the inline reveal loop with one call to the helper, using `absFrom`,
     `absTo`, `source.foldLength`, and `selectionRanges`.
   - `continue` when the result is `null`.
   - Use the returned fold for both the range start (`absFrom - fold`) and
     `new DateMarkWidget(model, fold)`. That keeps the mousedown anchor
     (`posAtDOM + foldLength`) and `data-fold-space` consistent with the range that was
     actually emitted.
   - Keep the in-code and model checks in their current order.
   - The fragment is at 999 lines and must stay at or below 1000. This replacement
     shrinks it.
3. `plugins/bob-ledger-tools/src/350-exports.js`: export `dateMarkSelectionFold` in
   `helpers`, next to `dateMarkSources`.
4. Update the `DateMarkWidget`/`dateMarkSources` comments in `136` wherever they
   describe folding, so they mention the cursor rule.
5. Run `npm run build`.
6. Bump the bob-ledger-tools patch version: `manifest.json` 1.34.0 → 1.34.1, or the next
   patch above whatever is current. Make the same version change in the `bob-plugins`
   README plugin table row. The description is unchanged.
7. Tests, in `scripts/test-ledger-tools-date-marks.cjs` (reuse its `makeView` /
   `pluginWithToday` harness; today is `2026-10-07`):
   - **DM24, the reported bug:** use the line `- [ ] #task  [created:: 2026-10-07]`.
     - A cursor at 12 (between the two spaces) gives one decoration `[13, 35)` whose
       widget has `foldLength` 0.
     - A cursor at 11 (right after `#task`, the range boundary) keeps fold 2,
       `[11, 35)`.
     - After typing `b` (`- [ ] #task b [created:: 2026-10-07]`, cursor 13), the
       decoration is fold 1, `[13, 36)`.
   - **DM25, Tasks double space:** use the line
     `- [x] #task Review skill [created:: 2026-08-29]  [completion:: 2026-09-03] ^review`.
     A cursor at 48 (between the two spaces before `completion`) emits completion at
     `[49, 74)` with fold 0. Created stays `[24, 47)` with fold 1 (independence).
   - **Visual selections:** a non-empty selection whose `to` lies strictly inside a run
     gives fold 0 for that field. One that ends exactly at the run start keeps the fold.
   - **Invariant sweep:** for the DM1, DM13, DM14, DM18, and DM24 lines, plus a
     triple-spaced `…foo   [scheduled:: 2026-10-09]` line, check every cursor offset
     from 0 to the line length:
     - no emitted decoration has `from < cursor < to`;
     - a field is still revealed exactly when the cursor is inside
       `[fieldStart, fieldEnd]`.
   - **Helper unit test:** cover `null`, `0`, and `foldLength` results, inclusive
     boundaries, several ranges, fold 0/1 inputs, and garbage input (non-array,
     non-numeric, `null` ranges) that never throws.
   - **Mousedown on a fold-0 widget:** anchors at the field start (`posAtDOM` 42 gives
     anchor 42).
   - Keep the existing DM1–DM23, reveal, and mousedown tests passing unchanged.
8. Gates:
   - `npm test` (it includes `build:check`), `npm run validate`, and the focused
     `node --test scripts/test-ledger-tools-date-marks.cjs`.
   - Show that the new DM24, DM25, and sweep tests fail on the old code: temporarily
     revert only the `src` change, rebuild, run the focused file, then restore and
     rebuild.
9. Run `bob plugins sync`, then `bob plugins list --no-pull` to confirm that
   bob-ledger-tools 1.34.1 is synced and enabled with no drift.

### bob-cli (primary repo)

10. `docs/date-marks.md`:
    - **Eligibility and folding:** extend the "Whitespace-run folding" paragraph with
      the cursor rule: "A selection endpoint never sits inside a fold. While a cursor or
      selection end lies strictly inside the space run, that field's mark folds nothing
      and the spaces show as text; the fold returns once the cursor leaves the run.
      Folded spaces still never reveal the field."
    - **Surfaces and interaction:** add a short sentence noting the same rule next to
      the reveal sentence.
    - **Conformance vectors:** add DM24 and DM25 rows matching the tests above. Use the
      table's `fold N` style, and include the cursor offset in the Input column.
    - **Live verification:** add the item "Vim `cw` on the word before a mark (and
      Backspacing that word without vim) types in order, before the mark."

## Out of scope

- The already-corrupted line in Bryan's `sase` note (the reversed `rab` after the
  created field) is vault content. Do not edit the vault; Bryan fixes that line by hand.
- No changes to freshness, priority, or progress marks; to rendered-view or Tasks-result
  date marks; or to Rust, storage, or the `api.dateMarks` namespace.

## Done when

- The new tests pass on the fix and fail on the old code. `npm test`,
  `npm run validate`, and `build:check` pass, and `266-plugin-date-marks.js` is ≤ 1000
  lines.
- bob-ledger-tools 1.34.1 (or the next patch) is built, committed in `bob-plugins`, and
  synced to the vault with no drift.
- `docs/date-marks.md` in bob-cli carries the cursor rule, the DM24 and DM25 vectors,
  and the live-check item.
- Live check, pending for Bryan: in Obsidian vim mode, `cwbar` on `foo` in
  `#task foo + today` yields `#task bar + today`.
