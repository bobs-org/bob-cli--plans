---
tier: tale
title:
  Tab marks and advances on the Mac Pull-In card; empty-filter Space in the Obsidian
  successor picker
goal:
  Mac Pull-In Tab toggles the highlighted offer then moves to the next; Obsidian Space
  toggles while the filter is empty.
size: medium
decisions:
  shift_tab:
    ask:
      Should Shift-Tab on the Mac Pull-In card move the highlight to the previous offer,
      like Obsidian?
    choices:
      previous:
        Shift-Tab moves up one visible offer without marking, keeping focus where Tab
        does
      native: Shift-Tab stays AppKit's reverse focus traversal; only Tab changes
    default: previous
    why: Once the card owns Tab, native Shift-Tab would jump focus out of the list
    answer: previous
proposed_by: bbugyi200.athena.1ee
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 09:12:35
status: wip
---

# Plan: Tab marks and advances in the Mac Pull-In card; empty-filter Space in the Obsidian successor picker

## Goal

Make the capture panel's successor choice card (the Mac "Pull-In" card) work like the
Obsidian successor picker:

1. **Mac:** add a `Tab` key. It toggles the mark on the highlighted offer, then moves
   the highlight to the next visible offer. If the highlighted offer is the last one,
   the highlight stays where it is. `Space` keeps its current behavior: the same toggle,
   but the highlight does not move.
2. **Obsidian:** add a `Space` key to the successor picker. It toggles the mark on the
   highlighted offer without moving, and works only while the filter input is completely
   empty. Once the filter holds any text, `Space` types into it as it does today.

"Select" means the existing **toggle** on both surfaces, so a second press unmarks.
Obsidian's `Tab` ("toggle and move") and the Mac's `Space` both toggle today.

This work touches two linked repositories and nothing in bob-cli itself. The two halves
do not depend on each other, and no `bob` JSON or CLI contract changes.

- **bob-mac-capture.** Open it with `/sase_repo` (`sase repo open bob-mac-capture`) and
  read its `AGENTS.md` before you edit anything.
- **bob-plugins.** Open it with `/sase_repo` (`sase repo open bob-plugins`) and read its
  `AGENTS.md`/`CLAUDE.md` before you edit anything.

## Current behavior (verified at plan time)

### Mac Pull-In card (bob-mac-capture)

- **Key routing.** `CaptureKeyCommandRouter.pullInChoiceCommand`
  (`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`) maps the card's keys:
  - Return, Escape, `Ctrl-[`, Up/Down, and `Ctrl-N`/`Ctrl-P`.
  - `Space`, which toggles only when it is unmodified and not a repeat (a repeat is
    consumed). It returns `.togglePullInOffer` when the filter is unfocused or empty,
    and `nil` (native typing) only when the filter is focused and holds text.
  - `Cmd-A`, `Ctrl-X`/`Ctrl-Shift-X`, `Ctrl-T`, and `Cmd-\`.
  - **Tab is unmapped.** It returns `nil` and falls through to AppKit focus traversal.
- **Ctrl-Tab.** `Ctrl-Tab`/`Ctrl-Shift-Tab` are matched at the top of
  `command(for:context:)`, before any modal branch, and always return `.switchPanel`.
  That must stay true.
- **Reason field.** While the Cancel reason field is focused (`reasonFocused`), the
  router handles only Return, Escape, and `Ctrl-[`. Every other key, Tab included,
  returns `nil`.
- **Controller dispatch.** `CapturePanelController.perform` handles the card's commands
  (around the `.confirmPullInChoice` … `.restorePullInReason` cases):
  - `.nextPullInOffer`/`.previousPullInOffer` first call
    `resignPullInFilterIfFocused()`, then `model.movePullInChoiceHighlight(by:)`.
  - `.togglePullInOffer` calls `model.toggleHighlightedPullInOffer()`.
- **Model and state.** `CapturePanelModel` (`movePullInChoiceHighlight`,
  `toggleHighlightedPullInOffer`) wraps `CapturePullInChoiceState`
  (`Sources/BobMacCapture/CapturePullInChoice.swift`):
  - `moveHighlight(by:)` walks `visibleSelectableKeys`. It clamps at both ends, and a
    missing or hidden highlight jumps to the first visible key (the last, when moving
    up). It also clears `verdictRefusalMessage`.
  - `toggleHighlighted()` toggles only a visible highlighted key, cancels any active
    reason editing, and calls `outcomes.toggleMark(for:)`. That call turns a staged Done
    or Cancel verdict into a Pull-In mark.
  - The model's move path also calls `disarmEscDiscard()` and
    `updateConsequenceForHighlight()`.
  - Both model methods call `requestFocus(.pullInOfferList)`.
- **Focus.** `model.focusPullInChoiceFilter()` (`requestFocus(.pullInFilter)`) is how
  the filter claims focus. The controller's focus repair re-claims first responder for
  the filter whenever `focusRequest.target == .pullInFilter`. Marks outside the current
  filter still submit (`effectiveSelectionKeys` uses every selectable key), so marking
  across several filters is a supported workflow.
- **Footer.** `CapturePullInChoiceFooter.hintItems`
  (`Sources/BobMacCapture/CapturePullInChoiceView.swift`) is
  `⌃N ⌃P Move · space Mark · ⌘A All · ⌃X Done · ⌃⇧X Cancel · esc Back · ⌃↩ Pull In & Open`.
  `accessibilityText` and `openIntentAccessibilityText` spell the keys out.
  `testFooterContractPinsHintsAndAccessibilityText` in
  `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift` pins both.
- **Docs.** The keys are documented in `docs/keyboard.md` ("Successor choices") and
  `docs/capture-previews.md` ("Successor offers").

### Obsidian successor picker (bob-plugins)

- **Where the keys live.** The picker is
  `plugins/bob-navigation-hotkeys/src/206-successor-picker-modal.js` (class extends
  `FilteredPickerModal` from `200-filtered-picker-modals.js`). Its `handleKeydown` is
  attached to the filter `inputEl`, which keeps focus. In order, it handles:
  1. `Ctrl/Cmd+Shift+Enter` retarget.
  2. Escape and `Ctrl-[`.
  3. Verdict keys (`handleVerdictKey`, `206c`).
  4. Disarming the Esc-discard arm.
  5. Enter.
  6. `Ctrl/Cmd+A`.
  7. **Tab**: Shift-Tab calls `moveSelection(-1)`. Plain Tab calls `toggleMark(key)` on
     a selectable offer row, then `moveSelection(1)`, which wraps.
  8. Everything else goes to `super.handleKeydown` (arrows and `Ctrl-N`/`Ctrl-P`).
- **Space is not handled.** It always types into the filter.
- **Toggling.** `toggleMark(key)` already turns a verdict row into a Pull-in mark.
- **Filter values.** `getQuery()` is the trimmed, lowercased filter value. Use the raw
  `this.inputEl.value` for the "has the user typed anything" test.
- **Reason input.** The Cancel reason input (`206d`) is a separate `<input>` with its
  own keydown listener, so its keys never reach `handleKeydown`.
- **Footer.** `SUCCESSOR_PICKER_FOOTER_HINTS` and `SUCCESSOR_PICKER_VERDICT_HINTS`
  advertise `Tab toggle and move`.
- **Tests and docs.** Tests live in `scripts/test-navigation-successor-picker.cjs` and
  `scripts/test-navigation-successor-verdicts.cjs`, which use `pressKey` from
  `scripts/modal-harness.cjs`. Modifier objects such as `{ repeat: true }` pass straight
  through to the event. Docs are in `docs/bob-navigation-hotkeys.md` (the paragraph
  starting "`↑`/`↓` and `Ctrl+N`/`Ctrl+P` move the successor picker's highlight"). The
  plugin manifest is at version `2.88.0`.

## Changes: bob-mac-capture

### 1. Router: Tab (and Shift-Tab) on the Pull-In card

In `pullInChoiceCommand`, add a `KeyCode.tab` case. It applies only when `reasonFocused`
is false; the reason-field branch keeps returning `nil` for Tab.

- **Plain Tab** uses the same unmodified test Space uses.
  - A repeat returns `.consumeKey`, the same as Space and `Ctrl-X` on this card, so
    holding Tab never sweeps marks down the list.
  - Otherwise it returns a new command, for example
    `.togglePullInOfferAndAdvance(keepFilterFocus: filterFocused)`.
  - Tab acts **whether or not the filter holds text**. Tab never types into the filter;
    this matches Obsidian, where Tab always toggles.

> [!decision] shift_tab = previous **Shift-Tab** (exactly `.shift`) returns a new
> command such as `.previousPullInOfferFromTab(keepFilterFocus: filterFocused)`. It
> moves the highlight up one visible offer without marking and clamps at the first row.
> Repeats are allowed, as they are for Up/`Ctrl-P`. Do not reuse `.previousPullInOffer`:
> that command resigns the filter, but Shift-Tab must keep focus where Tab keeps it.

> [!decision] shift_tab = native Shift-Tab returns `nil` from the Pull-In branch, so
> AppKit still handles it. Only plain Tab changes.

- **Every other modifier set on Tab** returns `nil`. `Ctrl-Tab` and `Ctrl-Shift-Tab`
  never reach this branch anyway, because the top-of-router panel switch matches them
  first. Keep it that way.

Add the new case or cases to `CaptureKeyCommand`. If the compiler finds any other
exhaustive `switch` over the enum, handle the new cases there too.

### 2. State and model: toggle, then advance

- **State.** In `CapturePullInChoiceState`, add
  `mutating func toggleHighlightedAndAdvance()`:
  1. Call `toggleHighlighted()`, which keeps its visibility guard and verdict-to-mark
     conversion.
  2. Call `moveHighlight(by: 1)`.
  3. Call `updateConsequenceForHighlight()`.

  This gives the following behavior:
  - On the last visible offer, the row toggles and the highlight stays.
  - With no visible highlight, nothing toggles and the highlight lands on the first
    visible offer.
  - With an empty filter result, nothing happens. The key is still consumed.

- **Model.** In `CapturePanelModel`, add
  `toggleHighlightedPullInOfferAndAdvance(keepFilterFocus: Bool)`:
  1. Guard on `pullInChoice`.
  2. Call `choice.disarmEscDiscard()`, the same as navigation.
  3. Call `choice.toggleHighlightedAndAdvance()` and store the choice.
  4. Request focus: `.pullInFilter` when `keepFilterFocus`, else `.pullInOfferList`.
- **Shift-Tab.** For the `previous` branch, add a matching move-up method that does what
  `movePullInChoiceHighlight(by: -1)` does but applies the same focus rule. For example,
  give `movePullInChoiceHighlight` a `keepFilterFocus: Bool = false` parameter.

The focus rule exists so you can type a filter, Tab through matches, change the filter,
Tab again, and press Return without touching the mouse, the same as in Obsidian.
Keyboard-only marking across filters depends on it. When the filter does not hold focus,
the offer list keeps focus, the same as Space.

### 3. Controller dispatch

In `CapturePanelController.perform`:

- **Tab command:** call `model.toggleHighlightedPullInOfferAndAdvance(keepFilterFocus:)`
  with the router's value and return `true`. Do **not** call
  `resignPullInFilterIfFocused()`.
- **Shift-Tab command (if `previous`):** the same pattern with the move-up method.

### 4. Footer hints and accessibility copy

- **Hints.** Insert `("tab", "Mark & Next")` into `CapturePullInChoiceFooter.hintItems`
  directly after `("space", "Mark")`. The `ViewThatFits` fallbacks need no change:
  `prefix(2)` still shows Move and Mark.
- **Accessibility text.** In `accessibilityText` and `openIntentAccessibilityText`,
  after "Space marks the highlighted offer.", add "Tab marks it and moves to the next
  offer." For `previous`, also add "Shift-Tab moves to the previous offer."
- **Format.** Format only the lines you change (`.swift-format`: 4-space indent, 100
  columns).

### 5. Docs

- **`docs/keyboard.md`, "Successor choices":** add Tab to the first sentence. For
  example: "Space marks, Tab marks and moves to the next offer (staying on the last)"
  and, for `previous`, ", Shift-Tab moves back". Keep the rest of the paragraph.
- **`docs/capture-previews.md`, "Successor offers":** after "Rows start unselected: …",
  add one sentence. For example: "Space marks or unmarks the highlighted row; Tab does
  the same and then moves to the next visible offer, keeping filter focus while you
  type." For `previous`, add "Shift-Tab moves back".
- **Check:** run `just docs-check`, which `AGENTS.md` requires for docs changes.

### 6. Tests (`Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift`; add `CapturePullInChoiceModelTests.swift` cases if state-level tests fit better there)

- **Router.** Use `keyEvent(48)` (Tab keyCode 48).
  - It returns the new command with `keepFilterFocus == false` from the plain card
    context.
  - It returns `keepFilterFocus == true` with `pullInFilterFocused: true`, for both an
    empty filter and a filter that holds text.
  - A repeat returns `.consumeKey`.
  - `.option` and `.command` return `nil`.
  - `pullInReasonFocused: true` returns `nil`.
  - `Ctrl-Tab` with `pullInChoiceVisible: true` still returns `.switchPanel`.
  - For `previous`: Shift-Tab returns the move-up command, and its repeat is not
    consumed. For `native`: Shift-Tab returns `nil`.
- **Controller/model.** Use the three-offer `installPullInChoiceForPreviews` fixture,
  where visible order is `opaque#2`, `opaque#1`, `opaque#3` and the highlight starts on
  `opaque#1`.
  - Tab marks `opaque#1` and moves the highlight to `opaque#3`, and the focus target is
    `.pullInOfferList`.
  - Tab again marks `opaque#3` and the highlight stays on `opaque#3`.
  - Tab on a marked row unmarks it and still advances.
  - Tab on a staged Done row turns it into a mark. Use a close-supported fixture, as the
    existing `Ctrl-X` tests do.
  - With a filter active, Tab only walks visible offers.
  - `keepFilterFocus: true` leaves `focusRequest.target == .pullInFilter`.
  - Space still marks without moving.
  - For `previous`: Shift-Tab moves up without marking and clamps at the first row.
- **Footer contract.** Update `testFooterContractPinsHintsAndAccessibilityText` for the
  new `tab` item in all three `hintItems` assertions, and add "Tab" to the phrase list.
  For `previous`, also add "Shift-Tab".

## Changes: bob-plugins (`plugins/bob-navigation-hotkeys`)

### 7. Empty-filter Space in `206-successor-picker-modal.js`

In `handleKeydown`, add a Space branch next to the Tab branch. It must come after the
retarget, Escape, verdict, and Esc-disarm handling, and before `super.handleKeydown`.

- **When it applies.** All of these must hold:
  - `event.key === " "`.
  - No `ctrlKey`, `metaKey`, `altKey`, or `shiftKey`.
  - The event is not composing (`event.isComposing !== true && event.keyCode !== 229`).
  - `this.inputEl` exists and `this.inputEl.value === ""`. This is the raw value, not
    the trimmed `getQuery()`.
- **What it does.**
  - Call `event.preventDefault()` and `event.stopPropagation()` so no space reaches the
    filter.
  - If `event.repeat === true`, return without toggling.
  - Otherwise, when the active visible item is a selectable offer row, call
    `this.toggleMark(active.row.key)`. That already converts verdicts and re-renders.
  - Do not move the selection. On an `expand` row or a muted row, the key is consumed
    and does nothing.
- **What it leaves alone.**
  - Any non-empty filter value: Space falls through untouched, so it types.
  - Modified Space and IME composition fall through.
  - The Cancel reason input is unaffected, because its keys never reach this handler.
  - Do **not** change Tab, Shift-Tab, or the footer hint arrays. Tab stays the
    advertised gesture, and Space only works while the filter is empty.
- **Size.** Keep the fragment at or below 1000 lines; it is 968 now. If the branch
  pushes it over, move a small helper such as an `isPlainSpace(event)` predicate into a
  smaller sibling fragment that is already listed in `src/fragments.json` before
  `206-…`. Fragments share one scope.

### 8. Version, build, docs

- **Version.** Bump `plugins/bob-navigation-hotkeys/manifest.json` one minor version for
  the new key: `2.88.0` → `2.89.0`, or the next minor above whatever is current when you
  land.
- **Docs.** In `docs/bob-navigation-hotkeys.md`, extend the successor-picker sentence
  after "`↑`/`↓` and `Ctrl+N`/`Ctrl+P` move the successor picker's highlight without
  marking." For example: "`Tab` marks or unmarks the highlighted offer and moves down;
  while the filter is empty, `Space` marks or unmarks it in place, and once the filter
  holds text `Space` types." Also change "`Tab` on a verdict makes it Pull in" to cover
  empty-filter `Space`.
- **Build and generated docs.** Run `npm run build`, which regenerates `main.js`; never
  hand-edit it. Then run `npm run docs:gen`, which refreshes the generated facts/version
  tables.

### 9. Tests

- **`scripts/test-navigation-successor-picker.cjs`.**
  - With an empty filter, `pressKey(modal, " ")` marks the highlighted offer, leaves
    `selectedIndex` unchanged, sets `defaultPrevented`, and leaves `inputEl.value` as
    `""`. A second Space unmarks it.
  - Enter then submits exactly the Space-marked picks.
  - `{ repeat: true }` is consumed without toggling.
  - Space on a muted row and on an `expand` row does nothing but is consumed.
  - With `inputEl.value = "Offer"` (fire `inputEl.listeners.input()`), Space is not
    prevented and the marks are unchanged.
  - Shift+Space, Ctrl+Space, and `{ isComposing: true }` are not prevented.
- **`scripts/test-navigation-successor-verdicts.cjs`.** Empty-filter Space on a Done or
  Cancel verdict row turns it into a Pull-in mark, the same as the existing Tab case.

## Verification and landing

### bob-mac-capture

Follow `AGENTS.md` exactly:

- **Compile.**
  - Run `just portable-check` on a host with Swift (athena).
  - `portable-check` cannot type-check `Sources/BobMacCapture*`/`Tests/BobMacCapture*`.
    So, best effort, compile a scratch rsync copy on the mac with `bundle.sh` and
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`.
  - Never land Swift you have not compiled. If the mac is offline, say in the final
    report that the macOS targets are unverified.
- **Before landing:** run `just docs-check` and `Scripts/macos-ci-check`.
- **After the push:** watch the exact-SHA macOS CI run to green with `/sase_monitor`.

### bob-plugins

- **Checks.** Run `npm run build:check`, the two successor test files (for example
  `node --test scripts/test-navigation-successor-picker.cjs scripts/test-navigation-successor-verdicts.cjs`),
  then `npm test` and `npm run docs:check`.
- **Deploy.** Run `bob plugins sync` to deploy to the vault. If sync refuses an
  overwrite, follow the `AGENTS.md` procedure; never pass `-F` on your own.

### Commits

Both linked repos become commit obligations of the implementing turn. Commit each one
through `/sase_final`.

## Out of scope

- The Mac Move choice card (`- ^^`) and the inbox route card keep their keys.
- Obsidian Tab keeps wrapping at the end of the list; only the Mac Tab stops on the last
  offer, per "next one (if any)".
- Typing a filter from the Mac offer list without focusing the filter first.
- bob-cli docs: the Obsidian successor-picker footer mockups in
  `docs/task-dependencies.md` stay accurate, because the Obsidian footer is unchanged.
- No memory note or decision record states these keys, so none needs a correction.
