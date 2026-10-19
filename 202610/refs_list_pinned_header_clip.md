---
tier: tale
title: Keep the Bob Refs selected row clear of the pinned section header
goal: Opening Bob Refs always shows the list from its top with the first row fully
  visible, and keyboard moves never leave the selected row's title hidden under a
  pinned section header.
size: medium
proposed_by: bbugyi200.athena.129
proposed_with: claude/opus
status: done
---

# Fix: Bob Refs list hides the selected row's title under the pinned section header

All changes are in the linked **bob-mac-capture** repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "..."`), read its `AGENTS.md` first, and make every
read and write through the path that command prints. Nothing changes in bob-cli.

## Symptom

Sometimes when the Bob Refs panel opens, the first row of the list is clipped. In
Bryan's screenshot (Oct 19, 18:25) the `NEW 1` header is pinned flush to the top of the
list column. Only the caption line of the selected NEW row shows ("Chat · added today ·
29 pp"); its title is hidden. Exactly 18 pt of the 44 pt row is visible, so the hidden
part is 26 pt, which is `RefsVisualTokens.sectionHeaderHeight`. The row's top edge sits
at the very top of the scroll viewport, under the pinned header.

## Root cause

`Sources/BobMacCapture/Refs/RefsListView.swift` keeps the selection visible with
`ScrollViewReader`. It calls `proxy.scrollTo(selected)` with a **nil anchor** from
`.onChange(of: model.selectedID)` and `.onChange(of: model.listing.orderedIDs)`. The
rows sit in `LazyVStack(spacing: 0, pinnedViews: [.sectionHeaders])`, and each 26 pt
`RefsSectionHeader` is drawn on `.regularMaterial` over the rows.

A nil-anchor `scrollTo` scrolls the minimum distance that fits the row's frame inside
the scroll view's bounds. SwiftUI does not account for pinned section headers when it
does this. So whenever it scrolls **up** to a row, the row's top lands flush with the
top of the viewport, and the row's own section header is pinned over its top 26 pt,
which is the title line. A row whose top is already inside the viewport but under the
header counts as "wholly visible", so in that case no scroll happens at all.

Why this shows up "sometimes" when the panel loads:

- `RefsPanelController` builds the `NSHostingView` once (`prewarm` /
  `makePanelIfNeeded`), and `hide()` is only `orderOut`. The list's `ScrollView`
  therefore keeps its scroll offset across hides.
- `show()` calls `prepareForPresentation()`, then `rerankSelectingFirst()`, which
  selects `orderedIDs.first`. The `onChange` handler then calls `scrollTo(firstRow)`,
  starting from wherever the previous session left the list.
- If that offset is past the first row, the first row is above the viewport. The minimal
  scroll then parks it under its pinned header (NEW, TODAY, Just scanned, …). The offset
  is past the first row whenever the user moved down with the keyboard or scrolled with
  the trackpad last time, or when content grew above the old top. The latter is common
  now: commit `f7e635a` pulls the vault on every show and lets arrivals join, and the
  NEW section sits above Today.

The same root cause also causes these defects, which this fix also resolves:

1. Suppose the first row's id and `orderedIDs` are both unchanged since the last
   session. Then neither `onChange` fires on show, and the panel opens wherever the list
   was scrolled, with the selected first row possibly entirely offscreen.
2. Moving up with ↑, Ctrl-P/Ctrl-K, Page Up, or the previous-section jump leaves the
   newly selected row's title under the pinned header.
3. Several other paths select the first row and hit the same minimal-scroll clip: a
   visible-panel scan that puts Just scanned first, a query or scope edit, and ⌘R with a
   changed first row.

## Fix design

Replace the header-blind `scrollTo(id)` with a **header-aware reveal**:

- **Selecting the first row of the listing always means "show the top of the list."**
  The view scrolls to the top edge, so the top padding, the first header, and the first
  row all sit in their natural rest positions. It also does this on every presentation.
- **Any other row gets a minimal scroll that treats the pinned header as covering the
  viewport's top band.**
  - When the row is above that band (or partly under it), it lands with its top exactly
    at `offset + 26`, just under its section's header. For the first row of a section,
    that puts the section header in its natural place.
  - When the row is below the viewport, its bottom aligns with the viewport bottom. For
    the last row of the listing, scroll to the max offset so the bottom padding shows.

The list layout is fixed by construction: 6 pt vertical padding, a 26 pt header for
every section whose `kind` is non-nil (search mode's single section has `kind == nil`
and no header), 44 pt rows, and spacing 0. `rowContent(for:)` is non-nil for every
listed id, so every row is exactly 44 pt. That lets row spans be computed in pure,
Linux-testable RefsCore code instead of measured from materialized lazy rows. This
matters because rows far from the viewport, such as the targets of Home, End, or Page
Up/Down, are not materialized.

### 1. RefsCore: pure list geometry and reveal policy

Add `Sources/RefsCore/RefsListScroll.swift`. Use Foundation/CoreGraphics types only, in
the same style as `RefsPanelGeometry.swift`, with `#if canImport(CoreGraphics)` imports
so it builds on Linux.

- `public enum RefsListGeometry` holds the list metrics as statics:
  `rowHeight: CGFloat = 44`, `sectionHeaderHeight: CGFloat = 26`,
  `verticalPadding: CGFloat = 6`.
- `public struct RefsListRowSpan: Equatable, Sendable` has `top`, `bottom`, and
  `pinnedInset`. All values are measured from the content top. `pinnedInset` is
  `sectionHeaderHeight` when the row's section has a kind, else 0.
- `public enum RefsListScrollTarget: Equatable, Sendable` has two cases: `.top` and
  `.offset(CGFloat)`.
- `public enum RefsListScrollPolicy` has the following members:
  - `static func contentHeight(of listing: RefsListing) -> CGFloat` returns
    `2 × verticalPadding + Σ over sections (header if kind != nil) + rowHeight × ids.count`.
    It mirrors the view exactly, so an empty section that has a kind still counts its
    header.
  - `static func rowSpan(for id: String, in listing: RefsListing) -> RefsListRowSpan?`
    walks the sections in order. Use the first section that contains the id
    (`listing.sectionIndex(for:)`) and the id's first index within that section. It
    returns nil when the listing does not contain the id.
  - `static func reveal(_ id: String, in listing: RefsListing, offset: CGFloat, viewportHeight: CGFloat) -> RefsListScrollTarget?`
    applies these rules in order:
    1. Return nil if the listing does not contain the id, or if `offset` or
       `viewportHeight` is non-finite.
    2. Return `.top` if `id == listing.orderedIDs.first`. Do this unconditionally, even
       when the measured offset is already 0, because the measurement can be stale
       across a re-rank. The view applies `.top` idempotently.
    3. Return nil if `viewportHeight <= 0`, because geometry has not been measured yet.
    4. Compute `maxOffset = max(0, contentHeight − viewportHeight)` and
       `current = min(max(offset, 0), maxOffset)`. A stale offset from taller,
       pre-re-rank content is clamped to the new content.
    5. Let `span = rowSpan(...)`. If `span.top < current + span.pinnedInset − 0.5`, the
       target is `span.top − span.pinnedInset`.
    6. Otherwise, if `span.bottom > current + viewportHeight + 0.5`, the target is
       `maxOffset` when the id is `orderedIDs.last`, else
       `span.bottom − viewportHeight`.
    7. Otherwise return nil. Clamp the target to `0...maxOffset`. If
       `abs(target − current) <= 0.5`, return nil; otherwise return `.offset(target)`.

    Rule 5 wins over rule 6 when the viewport is too short for both the header and the
    row, so the title stays readable.

Document each rule briefly in doc comments, in the same style as
`RefsInspectorScrollPlanner`.

### 2. Tokens forward to the RefsCore metrics

In `Sources/BobMacCapture/Refs/RefsVisualTokens.swift`, define `rowHeight`,
`sectionHeaderHeight`, and `listVerticalPadding` as the `RefsListGeometry` values. Write
them as `static let rowHeight: CGFloat = RefsListGeometry.rowHeight`, and so on. This
way the view layout and the policy cannot drift apart. Values stay 44, 26, and 6.

### 3. RefsListView: header-aware reveal via `ScrollPosition`

In `Sources/BobMacCapture/Refs/RefsListView.swift`, change only the live (non-preview)
path. The `previewMode` static stack stays as it is.

- Drop `ScrollViewReader`. Add
  `@State private var scrollPosition = ScrollPosition(edge: .top)` and attach
  `.scrollPosition($scrollPosition)` to the `ScrollView`. These APIs need macOS 15+; the
  view is already `@available(macOS 26.0, *)`.
- Track the viewport with
  `.onScrollGeometryChange(for: Viewport.self) { geometry in ... } action: { old, new in ... }`.
  Use a private `Equatable` value with `offset = contentOffset.y + contentInsets.top`
  (measured from the content top), `height = containerSize.height`, and
  `topInset = contentInsets.top`.
  - Store it in a small private `final class` box held in `@State`, **not** a plain
    `@State` value. The action fires on every scroll frame, and writing a `@State` value
    would re-evaluate the list body on every frame. Keep in mind the
    `testLiveListBuildsOnlyRowsNearViewport` budget and the README's latency work.
  - In the action, call `revealSelection()` when the old height was `<= 0` and the new
    height is `> 0`. That is the first valid measurement, for example a list rebuilt
    after the empty, skeleton, or Tabs-loading states with a non-first selection.
- Call `revealSelection()` from `.onChange(of: model.selectedID)` and
  `.onChange(of: model.listing.orderedIDs)`, as today, and also from a new
  `.onChange(of: model.presentationCount)`. That makes every show reveal even when the
  selection and order did not change. This fixes defect 1.
- `revealSelection()` does the following:
  - If `model.selectedID` is nil, it does nothing.
  - Otherwise it calls
    `RefsListScrollPolicy.reveal(selected, in: model.listing, offset: box.offset, viewportHeight: box.height)`.
  - On `.top`, it calls `scrollPosition.scrollTo(edge: .top)`.
  - On `.offset(y)`, it calls `scrollPosition.scrollTo(y: y − box.topInset)`. That is
    the `ScrollGeometry.contentOffset` convention; the inset is 0 in this panel in
    practice, but keep the conversion consistent.
  - Do not wrap the scroll in `withAnimation`. It inherits the caller's transaction
    exactly as `scrollTo` did, so the tab-close path stays animated and everything else
    stays instant.
- If the mac check or the hosted test shows that SwiftUI ignores an equal `.top`
  request, make `.top` requests distinct. This would happen when the binding already
  holds the top edge but content above the old top grew without a user scroll. One way:
  when the binding already equals `ScrollPosition(edge: .top)`, use
  `scrollPosition.scrollTo(y: −box.topInset)` instead. The hosted test below covers this
  case.
- Keep `.id(id)` on rows and the `GeometryReader` budget reporting unchanged. Update the
  `RefsListView` doc comment to say the selection stays fully visible below the pinned
  header, and that every show and every first-row selection returns the list to its top.

### 4. Tests

**Portable** (`Tests/RefsCoreTests/RefsListScrollTests.swift`, run by
`just portable-check` on Linux). Build `RefsListing` values directly with sections such
as `.new`, `.today`, and `.ready`, plus a nil-kind search listing.

- `contentHeight` and `rowSpan` for a multi-section browse listing: the first row starts
  at 32 (6 + 26), and a second section's first row starts after the previous rows plus
  its own header. In the nil-kind listing, the first row starts at 6 and `pinnedInset`
  is 0. An id that appears in two sections resolves to the first one.
- The first row returns `.top` at offsets 0 and 500, with a stale offset, and with
  `viewportHeight` 0. **This is the regression test for the screenshot.**
- A row above the viewport returns `.offset(top − 26)`. For a section's first row, that
  value equals the section header's natural top.
- A row whose top is inside the viewport but within 26 pt of the offset (under the
  pinned header) is revealed. This is the ↑ regression. The same row fully below the
  header returns nil.
- A row below the viewport returns `.offset(bottom − viewport)`, and the last row
  returns `.offset(maxOffset)`.
- In the nil-kind search listing, a row above the viewport lands at its exact top (inset
  0).
- A stale offset larger than the new `maxOffset` is clamped before the decision.
  `viewportHeight <= 0` returns nil for non-first rows. Non-finite input and an unknown
  id return nil.

**Hosted** (new `Tests/BobMacCaptureTests/RefsListScrollHostedTests.swift`,
`@MainActor`). Follow the hosted-panel pattern in `RefsInspectorScrollTests`: a real
`RefsPanelController`, `makePanelIfNeeded()`, `setFrame`, `orderFrontRegardless`, and
bounded `RunLoop` polling with a deadline. Copy the small helpers you need; they are
private there. Find the list's `NSScrollView` with
`RefsPanelController.scrollViews(in:)`, excluding the inspector's. Read the offset from
`documentVisibleRect.origin.y`.

Per `AGENTS.md`, never await an untimed continuation or gate. Every wait is a deadline
loop that ends in `XCTFail` with diagnostics: clip bounds, document height, selected id,
and the policy's expected span.

Use fixtures that produce at least two headed browse sections and enough rows to
overflow the viewport several times. Cover these cases:

1. Move the selection to the last row (`perform(.move(.last))`) and wait until the list
   is scrolled down. Then call `model.prepareForPresentation()`. Wait until the list
   offset is ≤ 0.5, and assert that the selected id is `orderedIDs.first`.
2. With the list at rest at the top and its first row selected, install library data
   whose ranking adds a NEW (or Just scanned) section above the old first row. Then call
   `model.prepareForPresentation()`. The offset ends ≤ 0.5. This covers content growing
   above the old top.
3. From a deep row with the list scrolled down, perform `.move(.previous)` until the
   selection crosses the viewport's top band. After each move, the list offset must
   satisfy `span.top ≥ offset + span.pinnedInset − 0.5`, using
   `RefsListScrollPolicy.rowSpan` for the selected id.

## Docs

In `README.md`, in the Bob Refs layout paragraph, extend the sentence "Section headers
are 26 pt and pinned." with this behavior: the selected row always stays fully visible
below the pinned header; every show, and every re-rank that selects the first row,
returns the list to its top; moving up lands the selected row just under its section
header.

## Verification

1. On athena (Linux Swift), run `just portable-check` from the bob-mac-capture checkout:
   swift-format lint, then build and test the portable targets, including the new
   RefsCore tests. It must pass.
2. The macOS sources (`Sources/BobMacCapture*`, `Tests/BobMacCapture*`) cannot be
   type-checked on Linux. Follow `AGENTS.md`:
   1. rsync a scratch copy to the mac, excluding `.build` and `.git`.
   2. Run `Scripts/bundle.sh` and `just typecheck-tests` in it.
   3. Delete the copy.

   Never use Bryan's checkout or `just install`. If the mac is offline, state in the
   final report that the macOS targets are unverified. CI runs the hosted tests on
   macOS 26.

3. Commit in the bob-mac-capture repo, following that repo's conventions.
