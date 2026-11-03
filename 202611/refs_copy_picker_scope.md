---
tier: tale
title: Bob Refs Copy pickers show count, selection, and visible range
goal:
  Make the Bob Refs Copy card's code, link, and highlight pickers say how many items
  exist, which one is selected, and which are on screen, and fix the width and height
  bugs that hide that information.
size: medium
decisions:
  highlight_picker:
    ask:
      Should the Highlights (H) picker get the same count header, scope line, and rail
      as the code and link pickers?
    choices:
      all_pickers:
        H, C, and L pickers all get the count header, scope line, rail, and live digit
        hint
      code_and_links:
        Only C and L change; the H picker keeps today's header, hint, and messages
    default: all_pickers
    why:
      All three pickers share RefsSectionedWindow and the card, so one component keeps
      them consistent
    answer: all_pickers
proposed_by: bbugyi200.athena.17c
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 10:46:08
status: wip
---

# Plan: Bob Refs Copy pickers always show how many, which one, and which are on screen

All work happens in the linked **`bob-mac-capture`** repo. Open it with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`) and do every read and
write in the path it prints. Read that repo's `AGENTS.md` first: it sets the compile,
format, and docs rules this plan relies on. There are **no bob-cli changes**. This is
presentation only, which the `decisions:mac-capture-is-a-thin-client` and
`decisions:code-copy-reads-render-time-catalog` records leave to the app. Do not change
what is copied or how `bob ref code` / `bob ref links` are read.

## 1. What is wrong today

Bryan's screenshot shows the code picker (Command-B / `C`) for a 44-page PDF, with
Highlights on page 13. The same problems affect the link picker (Command-L / `L`) and,
to a lesser degree, the highlight picker (`H`):

1. **Count is easy to miss and ambiguous.** The only count is the header caption
   `CODE BLOCKS 13` (`RefsCopyCardView.headerCaption`), in small secondary text. In the
   screenshot, the reference row next to it also reads `Current · p. 13/44`, so "13"
   reads as a page as easily as a count.
2. **Nothing names the selected item.** The keycaps `1`–`9` are positions in the visible
   window, not ordinals (`RefsSectionedWindow.entry(atWindowPosition:)`). After a
   scroll, keycap `1` can be block 7. No label says "block 7 of 13".
3. **Nothing shows which items are on screen.** The screenshot shows 5 of 13 blocks,
   with no hint that 8 more lie below or that rows can scroll off the top. ↑/↓ also
   wraps from the last item to the first (`RefsSectionedWindow.move`), which looks like
   a silent jump.
4. **Width bug: the code well widens the card past the inspector.**
   `RefsCodePreviewWellView.codeLine` puts a `.fixedSize(horizontal: true)` `Text` in an
   `HStack` and then calls `.clipped()`. `clipped()` only clips drawing; it does not
   shrink layout. The line's full width becomes the card's minimum width, so with a long
   line the card overflows its fixed-width column (`RefsPanelView.readableCard`). The
   screenshot shows the result: row summaries run to the panel edge with no ellipsis,
   and everything trailing is pushed off screen: each row's `● p. N` label, the header's
   `p. 13 in Highlights` chip, the well's trailing fade, and the `+N more lines`
   capsule. That hides "which one is on this page" exactly where it matters.
   `RefsLinkPreviewWellView.payloadLine` uses the same pattern, so it can overflow the
   same way whenever the column estimate comes out too wide.
5. **Height bug: the link picker window ignores its well.**
   `RefsPanelView.copyCardPickerWindowSize(for:height:)` only special-cases
   `card.codePicker`. The link picker falls through to
   `RefsCopyCardLayout.highlightWindowSize`, which sizes rows as if the card had no
   well, and then `linkPickerBlock` adds the well anyway. In a short panel, rows plus
   the well overflow the card and crowd the hint line.
   `RefsLinkPreviewLayout.pickerWindowSize` exists for this but is never used.

## 2. The design

One idea organizes everything: **the header answers "how many", the scope line answers
"which one and which are on screen", and the rail shows the same thing as a picture.**
Each fact appears in exactly one text location, and the text is never ambiguous. The
mockup below is illustrative (the rows are block 6–8 of a 13-block catalog, with block 7
selected).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ [▤] 13 CODE BLOCKS · ● 2 ON THIS PAGE                      p. 13 in Highlights │  ← header: how many
│     The AI-native SDLC playbook                                               │
│   6.4.2  What it looks like (CLAUDE.md)                                    ╷  │
│  [1] javascript  # Payments service                              p. 15     │  │
│      # Payments service · 20 lines                                         │  │
│   6.5.2  What it looks like (.claude/skills/secure-api-review/SKILL.md)    █  │  ← rail thumb = rows on screen
│  [2] markdown    name: secure-api-review                       ● p. 13   ━█━ │  ← selected notch (accent)
│      name: secure-api-review · 19 lines                                    █  │
│  [3] javascript  name: verifier                                  p. 16     █  │
│      name: verifier · 9 lines                                              │  │
│                                                                            ╵  │
│  Block 7 of 13 · showing 6–8                                                  │  ← scope line: which one / which shown
│ ┌ markdown · What it looks like (…SKILL.md)              19 lines · p. 13 ┐  │
│ │ name: secure-api-review                                                  │  │  ← well (now clipped to the column)
│ └──────────────────────────────────────────────────────────────────────────┘  │
│ ↵ copies · ⌥↵ fenced · 1–3 pick · ↑↓ choose · esc back                        │  ← hint names the live digit range
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Header: how many (count first, never ambiguous)

- The caption puts the count before a plural-aware noun: `13 CODE BLOCKS`,
  `1 CODE BLOCK`, `8 LINKS`, `1 LINK`, `9 HIGHLIGHTS`, `1 HIGHLIGHT`. Render the count
  digits in `.primary` with `.monospacedDigit()` and the noun in the current secondary
  caption style, so the number stands out without a new font size.
- Code and link pickers add an on-page tally when a Highlights page is known and the
  tally is greater than 0: ` · ● 2 ON THIS PAGE`, with `●` in `Color.accentColor` to
  match the row markers. Use the existing `RefsCodePicker.pageBlockCount` and
  `RefsLinkPicker.pageLinkCount`. The trailing `p. 13 in Highlights` chip stays as it
  is; "this page" refers to it.
- The total is `entries.count`, which is what the user can copy, never bob's `n` field.
  Lossy decoding can drop malformed blocks, so `n` can exceed the count.

### 2.2 Scope line: which one is selected, and which are on screen

A single 20 pt line under the visible rows, above the well, leading-aligned with the
group headers (6 pt inset):

- **`Block 7 of 13`**: the word in secondary, the selected ordinal in `.primary`
  semibold `.monospacedDigit()`, `of 13` in secondary. The ordinal is
  `selectedIndex + 1`. The noun is `Block`, `Link`, or `Highlight`. Pairing a noun with
  the number keeps it from being mistaken for a keycap digit.
- Then, in tertiary: ` · showing 6–8` while some entries are off screen, or
  ` · all shown` when every entry is visible and there are 2 or more. Nothing follows
  the ordinal when there is exactly 1 entry.
- The ordinal digits use `.contentTransition(.numericText())` so they roll as the
  selection moves. Skip the transition under Reduce Motion and in `previewMode`.
- Accessibility: one element whose label is `Code block 7 of 13, showing 6 to 8` (or
  `…, all shown`). Hide the rail from accessibility; this line carries the same facts.
- The line is always present while a picker is open, so the layout never jumps as the
  window changes.

### 2.3 Overview rail: a picture of the whole list

A slim, non-interactive vertical rail on the trailing edge of the rows block, drawn
**only when some entries are off screen**. When it is drawn, rows reserve a trailing
gutter (`copyCardPickerRailGutter`, about 12 pt) so page labels and the selected-row
tint never sit under it. It reads like a native overlay scroller, with more information:

- **Track**: 3 pt capsule, full rows-block height, `primary` at about 0.07 opacity
  (about 0.18 under Increase Contrast).
- **Thumb**: a 3 pt capsule covering exactly the visible entries, `primary` at about
  0.28 opacity (about 0.5 under Increase Contrast), at least 16 pt tall.
- **Selected notch**: a 7×3 pt `accentColor` capsule centered on the track at the
  selected entry. It always sits inside the thumb.
- **On-page ticks** (code and link pickers with a known page): 3×2 pt `accentColor`
  marks at about 0.55 opacity, one per entry on the Highlights page. They echo the rows'
  `●`, so you can see where page-13 blocks sit relative to the window. The H picker has
  no ticks.
- Motion: the thumb and notch animate with `.smooth(duration: 0.18)` keyed on the scope
  value. Disable the animation under `accessibilityReduceMotion` and in `previewMode`.
- Draw it with a `GeometryReader` (or `Canvas`) inside an `.overlay`, so the size is
  available at layout and draw time. Do **not** round-trip the height through `@State` /
  `onGeometryChange`: `ImageRenderer` fixtures would snapshot an empty rail.

**Geometry** (pure, in RefsCore, unit-tested). N is the total, s the first visible index
(0-based), v the visible count, H the track height, and t the thumb height,
`min(H, max(minThumb, H·v/N))`. The thumb top is `(H − t) · s / (N − v)`. Map each entry
index k to a y center piecewise, so the thumb covers exactly the visible entries:

- k < s: `thumbTop · (k + 0.5) / s`
- s ≤ k < s+v: `thumbTop + t · (k − s + 0.5) / v`
- k ≥ s+v: `thumbTop + t + (H − thumbTop − t) · (k − s − v + 0.5) / (N − s − v)`

The notch is at `y(selectedIndex)`. Ticks are `[y−1, y+1]` spans, merged when they
touch, so 200 links draw a few shapes, not 200.

### 2.4 Hints and messages that tell the truth about digits

- The hint line names the live digit range instead of a hard-coded `1–9`: `1–3 pick` (or
  `1 picks` when the window shows one row). This applies to all hint variants: code
  `↵ copies · ⌥↵ fenced · 1–3 pick · ↑↓ choose · esc back`; links
  `↵ copies · ⌥↵ Markdown · ⇧↵ opens · A all · 1–3 pick · esc back`; link-open
  `⇧↵ opens in browser · ⇧1–3 open that link · esc back`; highlights
  `↵ copies · 1–3 pick · A copies all · esc back`.
- A digit past the window replaces `No code block 7`, `No link 7`, and `No highlight 7`,
  which read as "there is no 7th" while 13 exist. The new text is
  `Only 1–3 are showing · ↑↓ scrolls` when entries are off screen, and
  `Only 1–3 are listed` when all are shown. Keep the existing non-transient, non-error
  message style.
- Opening announcements gain the count: `13 code blocks. Press a number or Return.`, and
  the link announcement becomes
  `8 links. Press a number or Return to copy. Hold Shift to open.`
  (`RefsLinkPicker.openingAnnouncement` becomes count-aware).
- Row VoiceOver labels use the same index-based ordinal. The code row currently says
  `Code block \(entry.block.number) of …`; use `windowStart + position`. The link row
  currently uses `entries.firstIndex(of:)`, which is wrong for duplicate entries; use
  `windowStart + position`.

### 2.5 Reliability fixes (part of this change, not optional)

- **Clip well lines to the column.** In `RefsCodePreviewWellView.codeLine` and
  `RefsLinkPreviewWellView.payloadLine`, give the fixed-size `Text` a frame of
  `.frame(minWidth: 0, maxWidth: .infinity, alignment: .leading)` followed by
  `.clipped()`, replacing the `HStack { Text; Spacer }.clipped()` pattern. A frame that
  sets both min and max adopts the proposed width, so a long line can no longer widen
  the card. The existing trailing fade and `+N more lines` capsule then land at the
  well's real edge. Check that the code well's trailing-fade overlay still covers the
  clipped edge.
- **Size the link window from its well.** `copyCardPickerWindowSize` needs the card
  width, because the link well's line count depends on columns. Change
  `RefsCopyCardView.onMeasuredHeight: (CGFloat) -> Void` to
  `onMeasuredSize: (CGSize) -> Void`, report the measured size, and change
  `RefsPanelView.copyCardPickerWindowSize(for:height:)` to
  `copyCardPickerWindowSize(for:size:)`. Add a link branch that uses
  `RefsVisualTokens.linkWellLayout(cardWidth:availableHeight:longestURL:)` with the same
  inputs `RefsCopyCardView.linkPreviewLayout` uses. Keep `copyCardNoteCapacity` working
  from `size.height`.
- **Reserve the new footer in all three window budgets.** Add a
  `footerHeight: Double = 0` parameter to `RefsCodePreviewWellLayout.make`,
  `RefsLinkPreviewLayout.make`, and `RefsCopyCardLayout.highlightWindowSize`, and
  subtract it before dividing by the row height. Code and link pickers pass the scope
  line height plus the well's top padding, which is 6 pt today and not counted anywhere.
  The H picker passes the scope line height. Define these constants once in
  `RefsVisualTokens` (`copyCardPickerScopeLineHeight`, `copyCardPreviewWellTopPadding`)
  and use them in both the view and the panel's window-size function, so the two can
  never disagree.

## 3. Implementation

### RefsCore (portable; built and tested on Linux)

1. **New `Sources/RefsCore/RefsPickerScope.swift`:**
   - `RefsPickerNoun` (`.codeBlock`, `.link`, `.highlight`) with the plural caption noun
     (`CODE BLOCK(S)`, `LINK(S)`, `HIGHLIGHT(S)`), the ordinal word (`Block`, `Link`,
     `Highlight`), and the VoiceOver noun (`Code block`, `Link`, `Highlight`), plus
     spoken plurals for the announcements.
   - `RefsPickerScope: Equatable, Sendable` holds `noun`, `total`, `selectedIndex`,
     `visibleStart`, `visibleCount`, and `onPageIndices` (sorted, 0-based). It derives
     `isWindowed` (`visibleCount < total`), `visibleRange` (1-based), `selectedOrdinal`,
     `countCaption` (`13 CODE BLOCKS`), `ordinalLabel` (`Block 7 of 13`), `scopeLabel`
     (`showing 6–8` / `all shown` / nil), `digitRangeText` (`1–3` / `1`), `pickHint`
     (`1–3 pick` / `1 picks`), `digitMissMessage`, and `accessibilityLabel`. Use the en
     dash `–` in ranges, as existing strings do.
   - `RefsPickerRailGeometry` has
     `static func make(scope:trackHeight:minimumThumbHeight:tickHalfHeight:) -> Self?`.
     It returns nil when the list is not windowed or `trackHeight <= 0`. Otherwise it
     returns `thumb: ClosedRange<Double>`, `selectedY: Double`, merged
     `ticks: [ClosedRange<Double>]`, and a `y(forIndex:)` that implements the §2.3
     mapping.
2. **Give `RefsSectionedWindow` a `func scope(noun:onPageIndices:) -> RefsPickerScope`**
   (or build the scope from its public fields), and expose `var scope: RefsPickerScope`
   on `RefsCodePicker`, `RefsLinkPicker`, and `RefsHighlightPicker`. Code and link
   pickers take on-page indices from their entries' `isOnPage`; the highlight picker
   passes an empty list.
3. **`RefsCopy.swift`:** route the four digit-miss sites (`copyLinkPickerWindowRow`,
   `openLinkPickerWindowRow`, `copyCodePickerWindowRow`, `copyPickerWindowRow`) through
   `scope.digitMissMessage`. Add `footerHeight` to `highlightWindowSize`.
4. **`RefsCode.swift` / `RefsLinks.swift`:** add `footerHeight` to the two layout `make`
   functions, and make `RefsLinkPicker.openingAnnouncement` take the count (keep a
   non-static accessor that `RefsPanelModel` uses).

### BobMacCapture (macOS; compile on the Mac per `AGENTS.md`)

5. **`RefsVisualTokens.swift`:** add the scope line height (20), well top padding (6),
   rail gutter, track width, minimum thumb height, notch size, and the rail opacities
   (normal and Increase Contrast).
6. **New `Sources/BobMacCapture/Refs/RefsPickerScopeViews.swift`:**
   `RefsPickerScopeLine` (§2.2) and `RefsPickerScopeRail` (§2.3), both taking a
   `RefsPickerScope` plus `previewMode`.
7. **`RefsCopyCardView.swift`:**
   - Build the header caption from `scope.countCaption` plus the on-page suffix (§2.1).
     `headerAccessibilityLabel` and `cardAccessibilityLabel` gain `showing 6 to 8` when
     windowed.
   - `linkPickerBlock`, `codePickerBlock`, and `pickerRows` (H): wrap the window rows in
     a block with the trailing rail overlay and gutter (only when windowed), then the
     scope line, then the well (code and link) with `copyCardPreviewWellTopPadding`.
   - The hint text uses `scope.pickHint` (§2.4). The row accessibility ordinals change
     as described in §2.4.
   - Rename the size callback to `onMeasuredSize` (§2.5), and keep the `onChange`
     re-report hooks.
8. **`RefsCodePreviewWell.swift` / `RefsLinkPreviewWell.swift`:** the width-clipping fix
   (§2.5).
9. **`RefsPanelView.swift`:** `copyCardPickerWindowSize(for:size:)` with code, link, and
   highlight branches, all passing their footer heights. Update the doc comment, which
   currently says highlight pickers "fill the space".
10. **`RefsPanelModel.swift`:** count-aware opening announcements in
    `announceCodePickerIfNeeded` and `announceLinkPickerIfNeeded`.

> [!decision] highlight_picker = code_and_links Skip the H-picker wiring in steps 2, 3,
> and 7 (`pickerRows` keeps today's layout, hint, and `No highlight N` message, and
> `highlightWindowSize` gets no footer). Also skip the H fixture and the H sentence in
> the docs. Shared types still live in RefsCore, so H can adopt them later.

Follow `.swift-format` (4-space indent, 100 columns) and format only the lines you
change.

## 4. Tests

**RefsCore (must pass in `just portable-check`):**

- New `Tests/RefsCoreTests/RefsPickerScopeTests.swift`:
  - Captions and plurals for every noun: `1 CODE BLOCK`, `13 CODE BLOCKS`, `1 LINK`,
    `9 HIGHLIGHTS`.
  - `ordinalLabel` and `scopeLabel` for a windowed list (`Block 7 of 13`,
    `showing 6–8`), an all-shown list (`all shown`), and a single entry (no scope
    label).
  - `pickHint` and `digitMissMessage` for windows of size 1 and 3, windowed and
    all-shown.
  - The `accessibilityLabel` wording.
  - Rail geometry: nil when not windowed. The thumb covers the visible entries exactly;
    at `s = 0` the thumb top is 0, and at `s = N − v` the thumb bottom equals H.
    `selectedY` lies within the thumb for every selection the window can hold. The
    minimum thumb height is honored for N = 200, v = 3. `y(forIndex:)` is strictly
    increasing. Adjacent on-page ticks merge.
- `RefsCodeTests`: the scope ordinal is index-based even when the catalog's `n` values
  skip (decode a response with a malformed block, as
  `testCodeEnvelopeSkipsMalformedBlocksButKeepsEnvelope` does). Moving past the window
  updates `visibleStart`. The on-page indices match `pageBlockCount`.
  `RefsCodePreviewWellLayout.make(footerHeight:)` shrinks `pickerWindowSize` and never
  goes below 2.
- `RefsLinksTests`: the same scope coverage for links, plus
  `RefsLinkPreviewLayout.make(footerHeight:)`. Update the assertion on `"No link 99"`
  (around line 577) to the new message.
- `RefsCopyTests`: update the assertion on `"No highlight 9"` (around line 738) and the
  `highlightWindowSize` cases for `footerHeight`.

**BobMacCapture (compiled on the Mac; run by macOS CI):**

- `RefsPanelModelTests` (around line 1383): move to
  `copyCardPickerWindowSize(for:size:)`. Add a link-picker case asserting its window is
  smaller than the highlight window at 480 pt and stays within 2...9; that case fails
  before this change.
- Width regression (in `RefsCopyComponentsTests` or a new file): host `RefsCopyCardView`
  for a code picker whose selected block has a single 400-character line, and for the
  long-link picker, then assert
  `NSHostingController(rootView:).sizeThatFits(in: CGSize(width: column, height: 480)).width <= column + 0.5`,
  where `column` is the design tests' `inspectorColumnWidth`. Before the fix, the code
  case fails.
- `RefsCopyCardDesignTests`: add windowed fixtures `refs-copy-code-picker-windowed` (13
  blocks across `5.1.2`…`6.7.2`-style sections, Highlights page 13 with two on-page
  blocks, selection moved to block 7), `refs-copy-link-picker-windowed`, and (with
  `all_pickers`) `refs-copy-highlight-picker-windowed`. Build the 13-block and many-link
  responses in test code by decoding synthesized JSON; `RefsCodeBlock` has only a
  `Decodable` init. Derive each fixture's `pickerWindowSize` from
  `RefsPanelView.copyCardPickerWindowSize(for:size:)` at the fixture size, so the
  fixtures show what the panel would show. Existing fixtures keep their names.

## 5. Docs

- `docs/refs.md` § Copy card: replace `LINKS N` with `N LINKS`. Describe the count
  header with its on-page tally, the scope line (`Block 7 of 13 · showing 6–8`), and the
  rail with its selected notch and on-page ticks, and say that the hint names the live
  digit range. Do this in one short paragraph shared by the code and link pickers, and
  mention H when `all_pickers` applies.
- `docs/keyboard.md` § Copy card keys: the `1–9` row says the digits match the keycaps
  currently shown, and the hint names the live range.
- Run `just docs-check`. If no sibling `bob-cli` checkout exists, point `BOB_CLI_DIR` at
  a bob-cli checkout; the script explains its resolution order.

## 6. Verification (all required; report each outcome)

1. `just portable-check` on a host with `swift` (athena has it; `swift-format` is absent
   there, so the recipe falls back to `swift format lint`). RefsCore builds and every
   RefsCore test passes.
2. Type-check `Sources/BobMacCapture*` and `Tests/BobMacCapture*` on the Mac, following
   the `AGENTS.md` procedure: rsync a scratch copy without `.build`/`.git`, run
   `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
   Bryan's checkout or `just install` there. For host access, read
   `sase memory read tailnet.md -r "<why>"`. If the Mac is offline, say in the final
   report that the macOS targets are unverified. Never land Swift that has not compiled
   anywhere.
3. `just docs-check` passes.
4. If a macOS CI run for the change is reachable (`gh run list` in the repo), review its
   `render-fixtures` artifact in light and dark mode. Check that the windowed fixtures
   show the rail, notch, ticks, and scope line, and that no row text, page label, or
   well runs past the column edge. Otherwise, say in the final report that the fixtures
   were not reviewed.

## 7. Acceptance criteria

- With 13 blocks, the header reads `13 CODE BLOCKS · ● 2 ON THIS PAGE`, and the bare
  count `CODE BLOCKS 13` no longer appears.
- The scope line always names the selected item (`Block 7 of 13`) and the visible range
  (`showing 6–8`) or `all shown`. The rail appears only when entries are off screen, and
  its notch tracks the selection, including after a wrap from the last item to the
  first.
- A long code line or URL never widens the card. Row page labels, the
  `p. N in Highlights` chip, the well's fade, and the `+N more` capsule stay inside the
  column.
- In a short panel, the link picker's rows, scope line, well, and hint all fit in the
  card.
- Digit hints and digit-miss messages name the real visible range.
- VoiceOver hears the same count, ordinal, and range as the screen shows.
- Copy payloads, key bindings, the selection and wrap behavior, and bob JSON contracts
  are unchanged.

## Out of scope

- Changing ↑/↓ wrap-around, page-snapping the window, or making the rail clickable.
- Resizing the code preview well to fit short blocks. The well keeps today's fixed 45%
  height rule, which keeps window sizes stable while arrowing.
- Any bob-cli change.
