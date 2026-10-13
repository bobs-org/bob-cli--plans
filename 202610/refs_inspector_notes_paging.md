---
tier: tale
title: Bob Refs inspector notes paging with Ctrl+D
goal: Ctrl+D at the bottom of the Bob Refs preview reveals the next configurable page
  of your notes until all show.
size: medium
decisions:
  reveal_memory:
    ask: Should revealed notes stay revealed when you reselect a reference while the
      panel stays open?
    choices:
      reset: Every selection opens on the first page of notes, like the preview scroll
        resets to top
      keep: Each reference remembers its revealed notes until the panel hides; scroll
        still resets
    default: reset
    why: The inspector already rebuilds at the top on every selection
    answer: reset
proposed_by: bbugyi200.apollo.7e
decided_by: auto
status: done
---

# Bob Refs inspector: Ctrl+D at the bottom shows the next page of your notes

## Goal

In the Bob Refs panel (repo `bob-mac-capture`), the inspector's **YOUR NOTES** block
shows the first 3 commented highlights, then a dim `+N more · ⌘↵ opens the note` line.
You can't see the rest without leaving the panel. After this change:

- **Ctrl+D when the preview is already scrolled to the very bottom** reveals the next
  page of notes. A page is a new, configurable setting, **Preview notes per page**
  (default 3). The same number sets how many notes the preview shows at first. Pressing
  it again keeps paging until every note is shown.
- The preview **scrolls down only if needed**. If the content still fits, nothing moves.
  If it now overflows, the preview stays pinned to the tail so the next Ctrl+D pages
  again at once. If the new page is taller than the space allows, the scroll stops so
  the first new note stays in view.
- Ctrl+D anywhere above the bottom still scrolls half a viewport, as it does today.
  Ctrl+U is unchanged.

Worked example (the screenshot case): a reference has 8 commented highlights. The
preview shows 3 plus `+5 more · ⌃D shows 3 more`. Scroll to the bottom and press Ctrl+D:
6 notes show, then `+2 more · ⌃D shows the rest`, and the view rests at the new bottom.
Ctrl+D again: all 8 show and the hint row goes away. More Ctrl+D presses at the bottom
do nothing, as today.

## Design (the decisions this plan makes)

### Interaction

1. **One verb, two meanings by position.** Ctrl+D always moves forward through the
   preview. Above the bottom, forward means half a viewport. At the bottom, it means the
   next page of notes. No new key is needed, and the vim-style muscle memory from
   `4c22d09` (half-page scroll) still holds.
2. **"At the bottom"** means the measured offset is within 1 pt of the inset-aware max
   offset. Content that fits the viewport is always at the bottom, so a short preview
   reveals on the first press.
3. **Rapid presses stack.** A reveal waits for SwiftUI layout before it scrolls. A
   Ctrl+D that arrives during that wait counts as at-the-bottom and reveals the next
   page, and the newer follow-scroll replaces the older one. Two quick presses at the
   bottom give two pages, never a half-page jump in between. A Ctrl+U during the wait
   cancels the pending follow, then scrolls up as usual.
4. **Follow-the-tail scroll rule** (pure and unit-tested):
   `target = clamp(min(maxOffset, batchTop − revealContext))`. Here `batchTop` is the
   new page's first note, measured from the content top, and `revealContext = 24 pt`
   keeps the last line of the previous note visible as context. Scroll only when
   `target > currentOffset + 0.5`, so the view never scrolls up and never moves when the
   content still fits. The jump is instant, like the existing Ctrl+D/U jumps; the motion
   comes from the notes fading in (below).
5. **Click works too.** The hint row can be clicked. It calls the same model reveal and
   gets the same follow-the-tail scroll. A tap gesture does not take first responder, so
   the search field keeps focus.
6. **Reset.** The revealed-page state resets on every selection change and on every
   panel presentation (see the `reveal_memory` decision). It also resets when the
   page-size setting changes. When notes rehydrate for the same selection, the visible
   count is capped at the new total.
7. Narrow panel (no inspector), notes still loading, a `ref show` failure, or no hidden
   notes: Ctrl+D behaves exactly as today. It is consumed, and at the bottom it does
   nothing.

### Visuals

- **Section header gains a count**, in the same grammar as the list's pinned headers
  (`TODAY 5`). It reads `YOUR NOTES` in caption semibold secondary, then the total
  commented-note count in `caption2.monospacedDigit()` tertiary, spacing 6. For
  accessibility it reads "Your notes, 8".
- **The hint row becomes a quiet disclosure row.** A `chevron.down` glyph
  (`caption2.weight(.semibold)`) and the text, both `.caption`, `.monospacedDigit()`,
  tertiary. On hover the row brightens to secondary and shows the `.link` pointer. The
  text is set by a pure function (see RefsCore below):
  - hidden > next page: `+5 more · ⌃D shows 3 more`
  - hidden == 1: `+1 more · ⌃D shows it`
  - otherwise (the last page): `+2 more · ⌃D shows the rest`
  - The `⌘↵ opens the note` clause is dropped: the footer already shows `⌘↵ Note`, and
    the row now teaches the new key.
  - Accessibility: label "5 more notes", the button trait, and a named action "Show more
    notes" that runs the reveal.
- **Revealed notes fade in.** Every note at an index ≥ the page size (that is, every
  note beyond the first page) fades in once when it appears: opacity 0→1 plus a 6 pt
  rise, `easeOut(0.22)`. Notes in a batch are staggered by 0.04 s each. Layout is never
  animated, so content height is final at once and the follow-scroll measures promptly.
  The user sees the jump to the tail, then the new notes settle in. Under Reduce Motion,
  or in design-fixture `previewMode`, the notes render fully visible at once. Initialize
  the appear state from that flag in `init`, because `ImageRenderer` does not run
  `onAppear`.
- **VoiceOver:** each reveal posts `AccessibilityNotification.Announcement`, for example
  "Showing 6 of 8 notes", the same way the panel already announces counts.
- **Fix "Page Page 1".** `bob ref show` emits page labels that already read `Page 1`
  (from the `### Page 1` sidecar headings), and the view prefixes another `Page `. The
  screenshot shows `Page Page 1`. A longer notes list makes this more visible, so fix it
  in the same block: a label that already starts with `page`, `p.` or `p `
  (case-insensitive, the same prefixes as bob's `is_sidecar_page_label`) shows as is; a
  bare label such as `3` still becomes `Page 3`.

### Configuration

- New `AppSettings` property `refsInspectorNotesPageSize: Int`, UserDefaults key
  `refsInspectorNotesPageSize`, default 3, clamped to `1...20` in `didSet`, the same way
  as `canceledDraftStashCapacity`. Missing or garbage values fall back to 3, and
  out-of-range values clamp.
- Settings › References gains a `Stepper` over `1...20` with
  `LabeledContent("Preview notes per page", value: "\(n)")`, plus a caption: "The
  preview shows this many of your notes at first; Control-D at the bottom of the preview
  shows this many more." It can also be set with
  `defaults write org.bobs.bob-mac-capture refsInspectorNotesPageSize -int 5` (verify
  the bundle id in `Resources/Info.plist` before you document it).
- Wiring: `RefsPanelModel` gets `@Published public var notesPageSize: Int`, clamped.
  AppDelegate sets the initial value when it builds the model and adds a sink in
  `observeRefsSettings()` that applies the emitted value (follow that function's "apply
  the value you receive" comment).

## Implementation

Open the linked repo with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`). Read
its `AGENTS.md` first. All paths below are relative to that checkout.

### 1. RefsCore (portable, Linux-testable): `Sources/RefsCore/RefsInspector.swift`

- `RefsShowRow.commentedHighlights` returns **every** commented highlight in note order
  (drop the `prefix(3)`). Delete `remainingCommentCount`, which the loader no longer
  needs. Update the doc comments.
- Add `RefsShowAnnotation.displayPageLabel: String?`: nil for nil or blank; as-is when
  it already starts with `page`, `p.`, or `p ` (case-insensitive); otherwise
  `"Page \(label)"`.
- Add `public struct RefsNotesPage: Equatable, Sendable`:
  - `static let defaultSize = 3`, `static let sizeRange = 1...20`,
    `static func clampedSize(_:) -> Int`
  - `let size: Int` (clamped in `init(size:)`), `private(set) var shown: Int` (starts at
    `size`), `private(set) var batchStart: Int?` (the first index of the latest revealed
    page), `private(set) var generation: Int` (starts at 0 and increases by one per
    reveal)
  - `func visibleCount(total:) -> Int` = `min(total, shown)`; `hiddenCount(total:)`;
    `nextBatchCount(total:)` = `min(size, hidden)`
  - `mutating func reveal(total:) -> Range<Int>?`: nil when nothing is hidden; otherwise
    it reveals `start..<min(total, shown + size)` where `start = visibleCount(total:)`,
    sets `batchStart`, increases `generation`, and returns the range. Use `visibleCount`
    as the start, not the stored `shown`, so a total that shrank never leaves a gap.
  - `static func moreText(hidden:nextBatch:) -> String?` returns the three strings from
    the Visuals section, or nil when `hidden == 0`.

### 2. Loader: `Sources/BobMacCapture/Refs/RefsInspectorLoader.swift`

- `RefsInspectorContent.notes` now holds every commented highlight. Remove
  `remainingNoteCount` and its init parameter. `load` sets
  `built.notes = row.commentedHighlights`.

### 3. Model: `Sources/BobMacCapture/Refs/RefsPanelModel.swift`

- `@Published public var notesPageSize: Int` (default `RefsNotesPage.defaultSize`;
  `didSet` clamps and clears the stored page).
- Stored reveal state: `@Published private(set)` page plus the id it belongs to. Read it
  with `public func inspectorNotesPage(for id: String) -> RefsNotesPage`, which returns
  the stored page when the id matches and a fresh `RefsNotesPage(size: notesPageSize)`
  otherwise.

  > [!decision] reveal_memory = reset Keep one `(id, page)` entry. Clear it in a
  > `didSet` on `selectedID` (when the value changes) and in `prepareForPresentation()`.

  > [!decision] reveal_memory = keep Keep `[String: RefsNotesPage]` keyed by id. Clear
  > it in `prepareForPresentation()`, in `panelDidHide()`, and when `notesPageSize`
  > changes. Leave it alone on selection changes.

- `@discardableResult public func revealMoreInspectorNotes() -> Bool`. It returns false
  unless there is a selected id with hydrated content (`inspectorLoader.content(for:)`)
  that still has hidden notes. On success it stores the page, posts the "Showing X of Y
  notes" announcement, sends a `RefsInspectorNotesReveal(id:generation:range:)` on a new
  `PassthroughSubject` exposed as `public var inspectorNoteReveals: AnyPublisher<…>`
  (not published, matching `inspectorScrolls`), and returns true.
- `perform(.scrollInspector)` is unchanged. The controller decides between a reveal and
  a scroll, because only it can measure the geometry.

### 4. Scroll math: `Sources/BobMacCapture/Refs/RefsInspectorScroll.swift`

- `RefsInspectorScrollGeometry.isAtBottom(tolerance: CGFloat = 1) -> Bool` (false when
  invalid).
- `static func RefsInspectorScrollPlanner.revealTarget(geometry:batchTop:context:) -> CGFloat?`
  implements interaction rule 4. It returns nil for invalid geometry or when the target
  is not below the current offset.
- `RefsVisualTokens.inspectorRevealContext: CGFloat = 24`.

### 5. Controller: `Sources/BobMacCapture/Refs/RefsPanel.swift`

- Refactor `scrollInspector(_:)` into two helpers: one measures `(geometry, flipped)`
  from the inspector `NSScrollView`, the other sets an offset (from the top) on the clip
  view. The existing half-page path reuses them, so its behavior does not change.
- In `scrollInspector(.down)`: after measuring, if a reveal follow is pending **or**
  `geometry.isAtBottom()`, and `model.revealMoreInspectorNotes()` returns true, return.
  Otherwise run today's planner path. `.up` cancels any pending follow first. When there
  is no inspector scroll view, return before any reveal (narrow panel).
- Subscribe to `model.inspectorNoteReveals` in `init` (next to the `inspectorScrolls`
  sink) and call `followReveal(_:)`. This runs for keyboard and click reveals alike:
  - Cancel any previous follow task, then start a main-actor task that tries every ~16
    ms for up to ~0.5 s. Each try aborts if the panel is hidden, the selected id
    changed, or the model's page generation moved past this reveal. Each try calls
    `layoutSubtreeIfNeeded()`, finds the inspector scroll view, and finds the
    `RefsInspectorRevealAnchorView` whose `generation` matches and whose converted rect
    has nonzero height. It converts that rect into document-view coordinates and
    computes `batchTop` as a distance from the top: `rect.minY` when the document view
    is flipped, `docHeight − rect.maxY` otherwise. Then it measures the geometry, asks
    `revealTarget`, and sets the offset if the target is non-nil.
  - Track "follow pending" for interaction rule 3. Clear it when the follow finishes,
    times out, or is cancelled. A timeout does nothing.
- Add `static func findRevealAnchor(generation:in:)`, modeled on
  `findInspectorScrollView`.

### 6. View: `Sources/BobMacCapture/Refs/RefsInspectorView.swift` (+ `RefsPanelView.swift`)

- `RefsInspectorView` stays value-driven. It gets `notesPage: RefsNotesPage` (default
  `RefsNotesPage(size: RefsNotesPage.defaultSize)`) and `onRevealMoreNotes: () -> Void`
  (default no-op). `RefsPanelView` passes `model.inspectorNotesPage(for: selected)` and
  `{ model.revealMoreInspectorNotes() }`.
- `notesSection`: the header with the count, then `ForEach(0..<visible)` over
  `inspector.notes`, then the disclosure row when `RefsNotesPage.moreText(...)` is
  non-nil.
- `RefsInspectorRevealAnchor` (an `NSViewRepresentable` around
  `RefsInspectorRevealAnchorView`, an `NSView` whose `hitTest` returns nil and which
  stores `generation`). Attach it as a `.background` on the note at
  `notesPage.batchStart`. Put it **outside** the fade/rise modifiers, so it measures the
  layout frame and not the animated offset.
- Factor the fade-in into a small wrapper or modifier that reads
  `accessibilityReduceMotion` and the `previewMode` flag.
- `noteView` uses `note.displayPageLabel`.

### 7. Settings: `AppSettings.swift`, `SettingsView.swift`, `AppDelegate.swift`

As described under Configuration.

### 8. Docs: `README.md`

- Keyboard table, Ctrl-D/Ctrl-U row: "Scroll the preview down/up by half its height; at
  the very bottom, Ctrl-D shows the next page of your notes; search keeps focus". Update
  the paragraph under the table to match. Ctrl-D is still consumed at edges when nothing
  is hidden.
- Inspector bullets: replace "up to 3 commented highlights … ("+N more" beyond that)"
  with the paging description (first page by default 3, the Settings name, the hint row,
  click or Ctrl-D at the bottom, reset on selection or show, page labels shown as bob
  writes them).
- Opening Bob Refs / Settings bullet: mention Preview notes per page.
- Privacy: add the notes page size to the list of References `UserDefaults` keys.

## Tests

RefsCore (`Tests/RefsCoreTests/RefsInspectorTests.swift`, runnable via
`just portable-check`):

- `commentedHighlights` returns all of 8 commented highlights in note order. Update
  `testCommentedHighlightsKeepOnlyCommentedHighlights`, which drops
  `remainingCommentCount`.
- `RefsNotesPage`:
  - default 3; size clamping at 0, 1, 20, and 99
  - 8 notes: 3 → reveal `3..<6` → `6..<8` → nil
  - `generation`/`batchStart` progression
  - a total that shrinks below `shown` (visible count clamps, and the next reveal is nil
    or starts at the clamped count)
  - total ≤ size: nothing hidden, `reveal` returns nil
- `moreText`: the three strings, plus nil at 0.
- `displayPageLabel`: `"Page 1"` stays as is, `"page 2"` stays as is, `"p. 4"` stays as
  is, `"3"` becomes `"Page 3"`, blank and nil give nil.

App (`Tests/BobMacCaptureTests`):

- `RefsInspectorScrollTests`:
  - `isAtBottom` (tolerance, fitting content, invalid geometry)
  - `revealTarget`: content fits gives nil; a short batch pins to the new `maxOffset`; a
    tall batch is capped at `batchTop − 24`; never upward
  - Make the hosted harness take an optional `notes:` parameter, defaulting to none, so
    the existing boundary test stays notes-free and unchanged.
  - New hosted tests:
    - Notes above the bottom: Ctrl+D scrolls half a page and reveals nothing.
    - Scroll the clip view programmatically to the bottom with 8 notes and page size 3.
      One Ctrl+D gives shown 6 and generation 1; wait until the offset rises past the
      old bottom and lands on the new bottom (±2 pt). A second Ctrl+D gives all 8.
      Search keeps first responder, and the selection and query are unchanged.
    - Short content in a tall panel: Ctrl+D reveals with no offset change.
    - Two Ctrl+D presses back to back at the bottom reveal two pages.
- `RefsPanelModelTests`:
  - The reveal returns false with no hydrated content and false when nothing is hidden.
  - A reveal emits on `inspectorNoteReveals`, with the right range and generation.
  - Reset behavior per the chosen `reveal_memory` branch, across `.move(.next)` / back
    and `prepareForPresentation()`.
  - `notesPageSize` clamps and is used for fresh pages.
- `BobMacCaptureTests` (the AppSettings tests near `canceledDraftStashCapacity`):
  default 3, persistence, clamping, and a garbage value.
- `RefsPanelDesignTests`:
  - Fix the `remainingNoteCount:` call site. Give the chat fixture 5 notes so the
    disclosure row renders.
  - Add a `refs-inspector-notes-revealed-880` render: 8 notes, one reveal, the header
    count, and `+2 more · ⌃D shows the rest`.
  - Use real `Page N` labels in one fixture, to pin the double-prefix fix.

## Verification (do not skip; see the repo's AGENTS.md)

1. `just portable-check` on a host with Swift. This host has no `swift`; use the tailnet
   host that does (athena). It lints every Swift file and builds and tests RefsCore and
   CaptureCore.
2. The macOS targets: rsync a scratch copy (without `.build`/`.git`) to the mac, run
   `Scripts/bundle.sh` and `Scripts/xcode-swift.sh build --build-tests`, then, as a best
   effort,
   `Scripts/xcode-swift.sh test --filter 'RefsInspectorScrollTests|RefsPanelModelTests|RefsInspectorTests|BobMacCaptureTests'`.
   Delete the copy afterwards. Never use Bryan's checkout or `just install`. If the mac
   is offline, say in the final report that the macOS targets are unverified.
3. After the commit lands, check the repo CI (`.github/workflows/ci.yml`). A red **Lint
   Swift formatting** step skips the later steps, so check those too.
