---
tier: tale
title: Redesign the Mac successor pull-in choice card
goal:
  The Bob Mac Capture successor choice card becomes a full-width, picker-style card with
  two-line offer rows, live "what Return pulls in" cues, a keycap and action-button
  footer, and reviewable render fixtures, with keys and Bob contract unchanged.
size: medium
proposed_by: bbugyi200.apollo.7g.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.7g.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.7g.f0.md)
- **COMMITS:**
  - [b377146](https://github.com/bobs-org/bob-mac-capture/commit/b377146f9845ab214d3891a2b6cb739ab184be6f)
    — feat(pull-in): redesign successor pull-in choice card as picker sibling

# Plan: Redesign the Mac successor pull-in choice card

## Context

Bob Mac Capture (linked repo **bob-mac-capture**) shows an in-panel successor choice
card. It opens when Return is pressed on a `!` or `=x` draft whose current preview has
successor offers (epic bob-cli-7c, phase bob-cli-7c.7; key fixes in `76faeb6`). Open the
repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md` first.

Bryan asked for this card to look much nicer while staying intuitive, and delegated the
design. This plan is that design. It changes **presentation only**. Keys, routing,
selection semantics, and the `bob capture --pull-in` contract stay exactly as they are.
The app remains a thin client (decision `mac-capture-is-a-thin-client`): every number,
destination, and caption comes from Bob's payload. Client-side arithmetic stays limited
to what already exists (counts and the projected link total).

### What is wrong today

The card is `CapturePullInChoiceCard` / `CapturePullInChoiceRowView`, both
`private struct`s in `Sources/BobMacCapture/CapturePanelView.swift`, around lines
808–1000. Its state is in `Sources/BobMacCapture/CapturePullInChoice.swift`.

- **It looks like a different app from the capture picker.** It uses a fixed
  `.frame(width: 520).padding(.leading, 14)` in a 760 pt panel, a 9 pt radius, a
  0.22-opacity stroke, and a `.headline` title. Its rows use stacked `.caption` lines of
  uneven height. The capture picker (`CapturePickerCard` in `CapturePickerView.swift`)
  that Bryan uses all day is a full-width elevated card with a 46 pt filter bar, pinned
  26 pt section headers, fixed-height rows, and footer keycap hints. The two should read
  as siblings.
- **There are nested scroll views.** The card sits inside `auxiliaryScrollRegion`, an
  outer `ScrollView`, and it has its own inner `ScrollView` capped at
  `pullInChoiceViewportHeight = 190`. Panel sizing therefore relies on overflow
  measurement rather than an explicit height policy.
- **There is misleading copy.** For offer rows, `detailText` runs
  `SuccessorRows.unblockedRows`. Its recovered caption renders **"Ready · not pulled
  in"** under every offer, which describes the dry-run default, not the choice in front
  of the user. Separately, "needs ^id" in orange reads like a warning, but it only means
  Bob mints a block ID when the offer is picked.
- **What Return will do is invisible.** Return pulls in the marked offers, or the
  highlighted offer when none are marked. Nothing on screen shows that rule.
- **The panel footer is dead weight.** While the card is open, the usual
  `CapturePanelFooter` (status text plus Stash, Discard, Preview, and a disabled
  Capture) stays on screen, and the only key help is one caption line inside the card.
- **The render fixtures are blank.** `CapturePullInChoiceDesignTests` renders the whole
  `CapturePanelView`. CI's `render-fixtures` artifact (run `38464978866`) shows an empty
  panel, because `ImageRenderer` draws `ScrollView` content blank and draws the AppKit
  filter field as a placeholder. The Refs panel already solves this with a `previewMode`
  flag (`RefsListView`, `RefsSearchBar`, `RefsPanelView`). Reuse that pattern.

### Constraints carried forward

- The keys from `76faeb6` stay unchanged:
  - Up/Down and Ctrl-N/Ctrl-P move, including from the filter.
  - Space marks whenever the filter is empty or unfocused.
  - Cmd-A marks all.
  - Return chooses; Shift-Return chooses none.
  - Esc and Ctrl-[ cancel.
  - Do not touch `CaptureKeyCommandRouter` or `CaptureKeyRoutingContext`.
- Selection semantics stay unchanged: nothing is pre-marked, and
  `effectiveSelectionKeys` is computed as it is today.
- The copy rules in bob-cli `docs/task-dependencies.md` §12.6 still apply:
  - Group headers stay `Hand-offs`, `Project ‹note›`, and `Project finished`; do not
    uppercase them, because that would mangle note names.
  - Each row reads successor text first, then the destination, then the cause.
  - No row leads with a block ID.
  - Use only glyphs and colors the app already uses: system colors, no new hex values.
    The unblocked glyph is `lock.open.fill`, the still-blocked glyph is `lock.fill`, and
    the done glyph is `checkmark.circle(.fill)`.
- Leave room for bob-cli-7j (a per-row destination retarget key). Keep the destination
  capsule as its own small view so 7j can show `→ NAME` overrides there. bob-cli-76
  (project pull-in preview copy) does not touch this card.

## The design

The choice card becomes a sibling of the capture picker. It is one full-width elevated
card with a filter bar, a context line, a fixed-height list of two-line rows under
pinned group headers, and a footer that swaps the panel's buttons for keycap hints plus
two action buttons. The card's guiding idea is that **you can see what Return will do**.

```text
╭──────────────────────────────────────────────────────────────────────────╮
│ (🔓 Successors)  Filter offers                                  3 offers │ 46 bar
│ ✓ Unblocked by Fix apollo machine                    (Links 6/10) (+1)  │ 26 context
├──────────────────────────────────────────────────────────────────────────┤
│ ⌄ ↳ Hand-offs 2                                                 1 marked │ 28 header
│ ▣  Re-launch all failed agents                       (→ FIX · next up)  │ 44 row
│    Offer 1 · sase.md ^relaunch-failed-agents · adds ^id                  │
│ ☐  Book flights                                      (→ FIX · next up)  │ 44 row
│    Offer 2 · travel.md ^book-flights                                     │
│ ⌄ 📁 Project sase_foo 1                                                  │ 28 header
│    5 hidden · 3 already planned · 2 sub-projects stay blocked until …    │ 22 fold
│ ☐  Ship the outcome                                  (→ FIX)            │ 44 row
│    Offer 3 · projects/sase_foo.md ^ship · removed #hide                  │
│ 🔒 Project resident                                                      │ 44 muted
│    stays Blocked · blocked by project sase_foo · projects/sase_foo.md …  │
╰──────────────────────────────────────────────────────────────────────────╯
 [⌃N ⌃P] Move  [space] Mark  [⌘A] All  [esc] Back       (Skip ⇧↩) (Pull In 1 ↩)
```

### Card chrome

Use the picker's chrome exactly:

- `.background(.regularMaterial)` and a `RoundedRectangle` clip with
  `CapturePanelLayout.pickerCornerRadius` (12).
- A `Color.primary.opacity(0.08)` 0.5 pt stroke and `.shadow(radius: 18, y: 8)`.
- Full auxiliary width, with no fixed 520 frame and no leading inset.
- Keep `.focusable()` and `.focused($focusedControl, equals: .pullInOfferList)`, and add
  `.focusEffectDisabled()`. The highlighted row already shows focus, and a system focus
  ring around the whole card looks broken.

### Filter bar (46 pt)

This bar mirrors `CapturePickerFilterBar`.

- **Leading scope token capsule.** Show `lock.open.fill` in `.green` and "Successors" in
  `.caption.weight(.semibold)`, padded 8/4, on `.background.opacity(0.45)` in a
  `Capsule`. This is the same construction as the picker's `@sase: Tasks` token.
- **Filter field.** Use the existing `CapturePickerFilterField` wiring unchanged: the
  same `fieldIdentifier`, `focusTarget`, `onTextChange`, and `onFocusChange`. Use the
  placeholder "Filter offers" and keep the existing accessibility label.
- **Trailing count.** Use `.callout.monospacedDigit()` in `.secondary`. It reads "3
  offers" with no filter and "1 of 3" while filtering. Count visible selectable offers
  over all selectable offers.

### Context line (26 pt, optional)

The context line explains why the card appeared, and it holds the budget. It renders
only when it has something to show.

- **Leading.** `checkmark.circle.fill` in `.green`, then "Unblocked by ‹cause›". With
  more than one distinct cause it reads "Unblocked by ‹first cause› and N more". The
  causes are the distinct `unblockedBy` entries across the offer rows, keyed by
  `notePath` + `blockID` and kept in first-seen order. Truncate the cause text with
  `SuccessorRows.truncate`. Omit this part when no row carries a cause, which can happen
  on refusal payloads.
- **Trailing.** When `planBudget` is present, show the projected Task Link meter in
  `CapturePlanBudgetMeterRow`'s exact capsule style.
  - The capsule reads "Links ‹projected›/‹cap›": `.green` on `green.opacity(0.15)`, or
    `.red` on `red.opacity(0.15)` when over the cap.
  - Add a `.quaternary` delta chip "+N" when `effectivePickCount > 0`.
  - The numbers animate with `.contentTransition(.numericText())`.
  - The accessibility label reads "Projected links 6 of 10", prefixed "Warning: " when
    over the cap.
- Omit the line entirely when there is neither a cause nor a budget.

### List (fixed viewport, pinned headers)

Build the list as `ScrollViewReader` → `ScrollView` →
`LazyVStack(spacing: 0, pinnedViews: [.sectionHeaders])`, padded by `pullInListPadding`,
with a fixed height from the height policy below. Scroll the highlighted row into view
with `proxy.scrollTo(key, anchor: nil)`, without animation, the way the picker does.
That keeps keyboard movement crisp and avoids the current centered ease-out jump.

**Section header (28 pt, pinned).** Draw it on `.regularMaterial`, as the picker does.

- Leading: the collapse chevron (`chevron.right` rotated 90° when expanded, animated)
  and the group glyph:
  - hand-offs: `arrow.turn.down.right`, `.secondary`;
  - project: `folder`, `.secondary`;
  - project finished: `checkmark.circle`, `.green`;
  - any other group: `lock.open`, `.secondary`.
- Then the title (`section.title`, `.callout.weight(.semibold)`) and the selectable
  count (`.caption`, `.secondary`, monospaced digits).
- Trailing: "N marked" in `.caption.weight(.semibold)` with the accent color, only when
  the group has marks. This matters most when the group is collapsed.
- The whole header stays a plain `Button` that calls `togglePullInChoiceGroup`, with
  `.accessibilityAddTraits(.isHeader)`.

**Fold row (22 pt).** It appears only when `section.folded` has a nonzero count.

- Join the nonzero parts into one line with " · ": "5 hidden", "3 already planned", and
  "2 sub-projects stay blocked until the next reconcile".
- Style it `.caption`, `.secondary`, one line with tail truncation, inset to the title
  column, with `.help(fullText)`.

**Offer row (44 pt, two lines).** This takes after the Refs rows.

- **Leading 22 pt column: a checkbox, vertically centered.** Make it a new small
  `CapturePullInCheckbox` view:
  - It is a 16×16 `RoundedRectangle(cornerRadius: 4.5)`.
  - Unmarked: a 1.25 pt `.secondary` stroke on `.background.opacity(0.5)`.
  - Marked: an accent-color fill with a white `checkmark` at 9.5 pt bold.
  - The checkmark animates in with `.scale.combined(with: .opacity)` and
    `.snappy(duration: 0.18)`. Use no animation when `accessibilityReduceMotion` is on.
- **Line 1.**
  - The title is `.body`, `.primary`, one line, tail truncated, `layoutPriority(1)`.
  - Render it through `TaskDisplayText(parsing: row.text)` and
    `CapturePickerRichText.displayText(display.text, segments: display.segments, matches:)`.
    Code spans and wikilinks then render as they do in the picker, and the filter match
    shows semibold in the accent color. The match is the first case-insensitive
    occurrence of the filter text in `display.text`, converted to a `Character`-offset
    range.
  - Trailing: the **destination capsule** (`CapturePullInDestinationCapsule`).
    - Its text is `row.destinationText` in `.caption.weight(.semibold)`, padded 7/3, on
      `.quaternary` in a `Capsule`, which is the preview's linked-row capsule.
    - It is `.green` when the row is in `effectiveSelectionKeys` and `.secondary`
      otherwise, and the color animates. This is the core intuition: the rows Return
      will pull in "light up" in the same green they wear in the preview after linking.
      With nothing marked, the capsule follows the highlight.
- **Line 2 (meta).** Use `.caption` and `.secondary`, one line, middle truncated. Join
  these parts with " · ":
  - "Offer N", with monospaced digits;
  - the locator `notePath ^blockID`;
  - "adds ^id" in the block-ID palette color
    (`CaptureEditorPalette.color(for: .blockID)`) when `needsBlockID`;
  - the row's meta details (see "State helpers" below);
  - "from ‹cause›", only when the choice has more than one distinct cause.
- **Background.**
  - Highlighted: `Color.accentColor.opacity(contrast == .increased ? 0.30 : 0.16)` in a
    `RoundedRectangle(cornerRadius: 8)`, with no stroke.
  - Hovered: `Color.primary.opacity(contrast == .increased ? 0.12 : 0.06)`. This is the
    picker's `rowBackground`.
- **Interaction.** Keep the existing `Button { model.togglePullInOffer(key) }` with
  `.buttonStyle(.plain)` and `.id(key)`.
- **Accessibility.** Keep `row.accessibilityLabel`. Append ", marked" or ", will be
  pulled in" for effective rows. Add `.isSelected` when highlighted.

**Muted resident row (44 pt).** Show `lock.fill` in
`Color(nsColor: .tertiaryLabelColor)` centered in the checkbox column. The title is in
`.secondary`. The meta line is `row.detailText` (for example "stays Blocked · blocked by
project sase_foo") followed by the locator. It has no capsule, no hover, and no button.

**Empty filter result.** Show it centered in the viewport, in the style of
`CapturePickerEmptyStateView`. The headline is "No matching offers" and the message is
"Clear the filter to see all N offers."

### Footer (replaces `CapturePanelFooter` while the card is open)

Add a new `CapturePullInChoiceFooter` and branch to it in `CapturePanelView.body`.
Branch on `model.pullInChoiceVisible` first, then the picker hints, then the panel
footer. Keep the same `.fixedSize` / `.layoutPriority(2)` / `onGeometryChange` →
`updateMeasuredFooterHeight` hooks. Give it a minimum height of 28 pt, the panel button
height, so the panel does not jump.

- **Left: keycap hints.** Use the exact keycap look of `CapturePickerKeyHints`: a
  `.caption.monospaced()` keycap on `.background.opacity(0.6)`, a 4 pt radius, and a 0.5
  pt `primary.opacity(0.12)` stroke, with a `.caption` `.secondary` action.
  - The items are a static list so tests can pin the contract: `("⌃N ⌃P", "Move")`,
    `("space", "Mark")`, `("⌘A", "All")`, `("esc", "Back")`.
  - Wrap them in `ViewThatFits(in: .horizontal)`: first all four, then without "⌘A All",
    then only Move and Mark. That keeps 620 pt clean.
- **Right: two action buttons.** Use a new `CapturePullInActionButtonStyle(prominent:)`:
  - Pad it 10/5 in a `RoundedRectangle(cornerRadius: 7)`.
  - Its label is the title plus a trailing keycap glyph at 70% opacity.
  - The pressed state is 0.85 opacity; the disabled state is 0.45.
  - **Skip ⇧↩** (secondary, `Color.primary.opacity(0.07)` fill) calls
    `model.chooseNoPullInOffers()`.
  - **Pull In N ↩** (prominent, accent fill, white `.callout.weight(.semibold)` text)
    calls `model.confirmPullInChoice()`. The title is "Pull In N" with
    `N = effectivePickCount`, and plain "Pull In" when N is 0, in which case the button
    is disabled. The count animates with `.numericText()`.
  - Both buttons are `.disabled(model.isSubmitting)`.
  - Do **not** attach `.keyboardShortcut`. The router owns Return.
- **Accessibility.** The footer label spells out every key: "Up, Down, Control-N, or
  Control-P moves the highlight. Space marks the highlighted offer. Command-A marks
  every offer. Return pulls in the marked offers, or the highlighted offer when none are
  marked. Shift-Return skips pulling in any offer. Escape goes back to the draft without
  writing."
- **Status announcements.** The panel footer that read `statusText` is gone while the
  card is open. So the choice footer observes `model.statusAnnouncementTick` and posts
  `AccessibilityNotification.Announcement(model.statusText)`. An example is "No visible
  successor offer is selected".

### Motion and accessibility

- Animations are limited to the checkbox, the capsule tint, the chevron rotation, and
  the numeric counts. Each is `.snappy(duration: 0.15–0.18)` and each is `nil` under
  Reduce Motion.
- Never animate the card's appearance or size, because the panel owns resizing.
- Post announcements the way `CapturePickerCard` does:
  - on appear: "Choose successor offers, N offers";
  - on highlight change: the row's label;
  - on mark toggle: "Marked ‹text›" / "Unmarked ‹text›";
  - on filtered count change: "1 match" / "N matches".
- Every color is a system color that adapts to light, dark, and increased contrast.

## Changes (all in bob-mac-capture)

### 1. New file `Sources/BobMacCapture/CapturePullInChoiceView.swift`

Move the card out of `CapturePanelView.swift` into this file, rebuilt per "The design".
Use the 4-space style of `CapturePickerView.swift` and `@available(macOS 26.0, *)`, and
make the types `internal` (not `private`) so design tests can render them. It holds:

- An `extension CapturePanelLayout` with:
  - `pullInBarHeight = 46`
  - `pullInContextHeight = 26`
  - `pullInSectionHeaderHeight = 28`
  - `pullInFoldRowHeight = 22`
  - `pullInRowHeight = 44`
  - `pullInListPadding = 6`
  - `pullInMaxVisibleRows = 6`
  - `pullInMinVisibleRows = 2`
- `struct CapturePullInChoiceHeightPolicy: Equatable`, modelled on
  `CapturePickerHeightPolicy`. Its inputs are `sectionCount`, `foldRowCount`, `rowCount`
  (offers plus muted residents), `hasContextLine`, and `displayScale`.
  - `listContentHeight = 2·padding + sections·header + folds·foldRow + rows·row`
  - `maxListHeight = 2·padding + header + maxVisibleRows·row`
  - `minListHeight = 2·padding + header + minVisibleRows·row`
  - `listViewportHeight = min(content, max)`
  - `idealHeight = bar + (context ? contextHeight : 0) + 1 + listViewportHeight`
  - `minimumVisibleHeight` uses `min(content, minListHeight)` in place of the viewport.
  - Round every value to the pixel, as the picker does. Expose `auxiliaryHeight`.
  - Build it with `init(choice:displayScale:)`, which counts **all** sections, fold
    rows, and rows of the state and ignores the filter and collapse. Filtering or
    collapsing therefore never resizes the panel, matching the picker's contract.
- `CapturePullInChoiceCard(model:choice:previewMode:)`. The `previewMode` default is
  `false`. Add a doc comment citing the Refs precedent: `ImageRenderer` snapshots
  `ScrollView` content blank. When `previewMode` is `true`:
  - The list is a plain `VStack` with the same headers and rows, framed to
    `listViewportHeight` with top alignment and clipped.
  - The filter bar draws the filter text, or the placeholder in `.tertiary`, as `Text`
    instead of hosting the AppKit field.
- The private subviews: filter bar, context line, section header, fold row, offer row,
  muted row, empty state, `CapturePullInCheckbox`, `CapturePullInDestinationCapsule`,
  `CapturePullInActionButtonStyle`.
- `CapturePullInChoiceFooter(model:)` with `static let hintItems` (the four pairs above)
  and the accessibility label as a static string, so tests pin both.

### 2. `CapturePanelView.swift`: host the new card like the picker

- Delete the old `CapturePullInChoiceCard` and `CapturePullInChoiceRowView`. Delete
  `CapturePanelLayout.pullInChoiceViewportHeight` if nothing else uses it.
- `auxiliaryRegion`: replace `if model.pullInChoiceVisible { auxiliaryScrollRegion }`
  with the card itself:
  - `CapturePullInChoiceCard(model: model, choice: choice)`
  - `.layoutPriority(0)`
  - `.focused($focusedControl, equals: .pullInOfferList)`
  - `.onGeometryChange` → `updateMeasuredAuxiliaryContentHeight`, as the
    `CapturePickerCard` branch does.
- `auxiliaryContent`: remove the pull-in branch, unwrap its `else { … }` body, and drop
  `AuxiliarySection.pullInChoice`.
- `currentAuxiliaryHeight`: add a first branch for
  `model.pullInChoiceVisible, let choice = model.pullInChoice`. It returns
  `CapturePanelAuxiliaryHeight(idealHeight: max(policy.idealHeight, measuredAuxiliaryContentHeight), minimumVisibleHeight: policy.minimumVisibleHeight)`,
  the same shape as the picker branch.
- Footer branch in `body`:
  `if model.pullInChoiceVisible { CapturePullInChoiceFooter } else if model.pickerVisible { CapturePickerKeyHints … } else { CapturePanelFooter }`,
  with the existing geometry hooks on each.
- Leave `pullInPreviewSummary` (the preview's offer list) unchanged.

### 3. `CapturePullInChoice.swift`: pure presentation helpers

Add these helpers, all derived from data the state already holds:

- `CapturePullInChoiceRow.metaDetails: [String]` for offer rows. It returns "removed
  #hide" when `hideRemoved`. It returns "project finished" when
  `trigger == "project_empty"` **and** `group != "project_finished"`, so it never
  repeats the header. It returns "trigger: X" for any other non-empty trigger. It never
  includes the recovered "Ready · not pulled in" caption. Keep `detailText` unchanged:
  the muted rows, the filter haystack, and
  `testMutedResidentsAndFoldedProjectCountsAreNeverSelectable` still use it.
- `CapturePullInChoiceState`:
  - `causes: [TaskCompleteUnblockedCause]`: distinct, in first-seen order across offer
    rows.
  - `contextText: String?`: "Unblocked by X" / "Unblocked by X and N more" / `nil`.
  - `showsRowCauses: Bool`: `causes.count > 1`.
  - `countText`: "3 offers", "1 offer", or "1 of 3" while filtering.
  - `isEffectivelySelected(_ key: String) -> Bool`.
  - `markedCount(in section:) -> Int`: valid marks only.
  - `pullInActionTitle`: "Pull In N" / "Pull In".
  - `projectedLinksCapsuleText`: "Links 6/10".
  - `projectedLinksDeltaText`: "+N" or `nil`.
  - `filterMatchRange(in text: String) -> Range<Int>?`: the `Character` offsets of the
    first case-insensitive match, or `nil` when the filter is empty.
- Keep `projectedLinksText` and `projectedLinksOverCap`, since tests use them.

### 4. README

Rewrite the successor-choice paragraph (around README lines 1551–1561). Keep every key
and semantic sentence, and adjust the wording:

- Return is "Pull In". Shift-Return is "Skip": it commits with no offers.
- Add one sentence on the look:
  - a full-width card like the task picker;
  - a context line naming what unblocked the offers;
  - destination capsules that turn green on the offers Return will pull in;
  - a live Links projection when Bob supplied the budget;
  - footer hints with **Skip** and **Pull In N** buttons that also work by pointer.

## Tests

- **`CapturePullInChoiceTests`** (logic, macOS CI):
  - **Height policy.**
    - 1 offer: the list equals its content.
    - 3 offers with a fold row and a muted resident: exact sum.
    - 20 offers: clamped to the max.
    - The minimum is clamped to `pullInMinVisibleRows`.
    - The context line adds `pullInContextHeight`.
    - `init(choice:)` gives identical values after `updateFilter("zzz")` and after
      `toggleGroup`.
  - **Helpers.**
    - `metaDetails` never contains "not pulled in", and it omits "project finished" on
      `project_finished` rows.
    - `contextText` covers one cause, three causes, and no causes.
    - `countText` covers singular, plural, and filtered.
    - `isEffectivelySelected` follows the highlight with no marks and only the marks
      otherwise.
    - `markedCount(in:)` ignores keys that are not selectable.
    - `pullInActionTitle` covers 0, 1, and 2 picks.
    - The projected capsule and delta texts are correct.
    - `filterMatchRange` is correct for a mid-string, case-insensitive match and is
      `nil` for an empty filter.
  - **Footer contract.** `CapturePullInChoiceFooter.hintItems` equals the four pairs
    above, and the accessibility string mentions Control-N, Space, Command-A, Return,
    Shift-Return, and Escape.
  - Existing tests stay green unchanged, including the router and controller tests from
    `76faeb6`.
- **`CapturePullInChoiceDesignTests`** (renders):
  - Keep the panel-level `capture-pull-in-choice-wide` / `-minimum` renders. They verify
    composition, footer, and chrome even though `ImageRenderer` blanks the live list.
  - Add card-level renders with `previewMode: true`. Each composes
    `VStack(spacing: 12) { card; CapturePullInChoiceFooter }.padding(18)` and renders
    through `RenderFixtureWriter` at 760 and 620, in `.aqua` and `.darkAqua`. Drive the
    states through `installPullInChoiceForPreviews`, `togglePullInOffer`,
    `updatePullInChoiceFilter`, and `togglePullInChoiceGroup`.
    - `capture-pull-in-choice-open`: 3 hand-offs with one cause, nothing marked, the
      first row highlighted, and the budget under the cap.
    - `capture-pull-in-choice-marked`: 2 marked, the budget over the cap, two causes so
      rows show "from …", and one `blockIDNeeded` row.
    - `capture-pull-in-choice-project`: a hand-off, a `project` group with folded counts
      and a muted resident, and a `project_finished` row with `hideRemoved`.
    - `capture-pull-in-choice-filtered`: the filter text "book" with the match
      highlighted.
    - `capture-pull-in-choice-no-match`: the filter text "zzz", showing the empty state.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

1. **Portable check.** Run `just portable-check` on athena in a scratch rsync copy, then
   delete the copy. It format-lints every Swift file, including the new view file. Run
   `just format-lint` there too if `portable-check` does not cover the test files you
   touched.
2. **Mac compile (best effort).** Rsync a scratch copy without `.build`/`.git`. Run
   `./Scripts/bundle.sh --identity -`, then
   `./Scripts/xcode-swift.sh build --build-tests`, then delete the copy.
   - Never use Bryan's checkout or `just install`.
   - XCTest cannot import with the mac's Command Line Tools, so only the app product
     build counts.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.
3. **CI and visual review.** macOS CI is the test and render authority. After the commit
   lands, watch its CI run with `/sase_monitor`. The `--next` follow-up must:
   - Confirm that Lint Swift formatting, Build, and the three `CapturePullInChoice*`
     test classes pass. The `RefsInspectorScrollTests` failures (bob-cli-7h) are known
     and pre-existing; report them but do not let them block. After a red lint step,
     check the later steps.
   - Run
     `gh run download <run-id> -R bobs-org/bob-mac-capture -n render-fixtures -D <fresh scratch dir>`.
   - **Look at every `capture-pull-in-choice-*` PNG** in light and dark at 760 and 620.
     Confirm:
     - nothing clips or overlaps;
     - the footer fits on one line at 620;
     - the green capsules appear only on the effective rows;
     - the over-cap meter is red;
     - the muted row reads as muted;
     - the empty state is centered.
   - Fix any defect it finds the same way: compile, then land.
4. **Manual (best effort, only if Bryan's installed app updates through his normal
   flow).** On a `!<task with dependents>` draft:
   - Return opens the new card, and the first capsule is green.
   - Ctrl-N moves the green capsule with the highlight.
   - Space marks; the capsule stays green on the marked row, and **Pull In 1** updates.
   - Clicking **Skip** commits with none.
   - Esc returns to the draft.

## Out of scope

- Key bindings and routing, selection semantics, and Bob's JSON contract.
- The live-preview offer summary (`pullInPreviewSummary`) and the notification copy,
  which belong to bob-cli-76.
- The per-row destination retarget, which belongs to bob-cli-7j. This plan only keeps
  the capsule as its own view.
- The Obsidian successor picker and the bob-cli docs: no doc pins the Mac card's
  visuals.
