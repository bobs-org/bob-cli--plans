---
tier: tale
title: "Bob Refs inspector: one HIGHLIGHTS section with note cards"
goal:
  The Bob Refs preview pane lists every highlight in one note-ordered HIGHLIGHTS
  section, and each highlight that carries a note shows it on a sticky-note card under
  its quote.
size: medium
decisions:
  ci_compile_fix:
    ask:
      Also fix bob-cli-9m's test-compile errors so CI can render and check the new
      design?
    choices:
      fix:
        Fix both errors in a first commit and close bob-cli-9m; CI then uploads the
        renders
      skip:
        Leave bob-cli-9m alone; CI still can't compile tests, so the design gets no
        render check
    default: fix
    why:
      A two-line rename plus a helper; no XCTest or render has run on CI since 2c007c7
    answer: fix
proposed_by: bbugyi200.athena.11q.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 09:57:21
status: wip
---

# Bob Refs inspector: one HIGHLIGHTS section, with a note card on noted highlights

## Goal

The Bob Refs panel's preview pane (the inspector column, repo `bob-mac-capture`)
currently splits a reference's highlights into two blocks:

- **YOUR NOTES** holds highlights that carry a comment.
- **HIGHLIGHTS** holds the highlights that do not.

Both blocks page as one Ctrl+D sequence, notes first (commit 7533626, plan
`202610/refs_inspector_highlights.md`). Bryan wants **every highlight in one section, in
the order it appears in the reference**, with or without a note. A highlight that has a
note must be easy to spot at a glance: its note sits directly under the quote on a warm
"sticky note" card with a note glyph, and the section header counts the notes with the
same glyph.

This is a Mac-only change. `bob ref show <path> -f json` already returns every
annotation in note order, so **do not change bob-cli**.

### Worked examples (default page size 3)

1. **Mixed, notes deep in the document** (29 highlights; the ones on Page 9 and Page 14
   carry notes). The header reads `HIGHLIGHTS 29` followed by the note chip
   `[note glyph] 2`. The first page shows highlights 1–3 in page order, and the
   disclosure row reads `+26 more · 2 with notes · ⌃D shows 3 more`. Each Ctrl+D at the
   bottom reveals 3 more. When a noted highlight scrolls in, its quote looks like every
   other quote, and its note card sits right under it.
2. **No notes at all** (5 highlights). The header reads `HIGHLIGHTS 5` with no chip, and
   the disclosure reads `+2 more · ⌃D shows the rest`. This matches today's highlights
   block.
3. **Every highlight noted** (8 highlights, all with comments). After one reveal there
   are 6 note cards on screen, and the disclosure reads
   `+2 more · 2 with notes · ⌃D shows the rest`.

## Design (decided; implement as written)

### Why one stream in note order

- The order of the reference is the order of the argument. With one stream, the
  highlights read top to bottom as the author wrote them, and each note sits next to the
  passage that prompted it. Today a note is cut off from the highlights around it.
- The previous plan put notes first so that Bryan's own words would lead. That trade-off
  is now reversed on purpose, and three things make up for it:
  - **The header chip** shows how many notes exist before you scroll.
  - **The disclosure row** says how many of the hidden highlights carry notes.
  - **The note card** stands out from plain quotes even when you scroll fast.
- Ordering: sort by nothing. Keep `ref show`'s annotation order, which is note order.
  Never re-sort by page label. Labels such as `pp. 3, 5` are display strings, not keys.
- A highlight with a note shows **once**, as a highlight with its note card. Nothing is
  duplicated.

### Entry anatomy

Every entry is a `VStack(alignment: .leading, spacing: 4)`. Its parts, from top to
bottom:

1. **Page label** (optional). `displayPageLabel` in `.caption`, `.secondary`. It follows
   the page-run rule below.
2. **Quote** (when the trimmed quote is non-empty). Render it exactly as the current
   `highlightView` does:
   - `.callout`, upright, `.primary`, `lineSpacing(2)`;
   - at most `inspectorHighlightLineLimit` (4) lines, truncated at the tail;
   - a 10 pt leading inset with a 2 pt yellow rule (`inspectorQuoteInset`,
     `inspectorQuoteRuleWidth`);
   - balanced `**bold**` runs through `quoteText(_:)`;
   - `.help(quote)`.

   Every quote looks the same whether or not it has a note. The yellow rule still means
   "text you highlighted".

3. **Note card** (only when `annotation.hasComment`). It sits `inspectorNoteGap` (6 pt)
   below the quote, or directly under the page label when the quote is empty:
   - Content: `HStack(alignment: .firstTextBaseline, spacing: 6)` with:
     - `Image(systemName: RefsVisualTokens.inspectorNoteSymbol)` (`"note.text"`, the
       sticky-note glyph) in `.callout.weight(.medium)`, `.imageScale(.small)`,
       `.foregroundStyle(RefsVisualTokens.inspectorNoteTint)`;
     - `Text(comment)` in `.callout`, `.primary`, with no line limit (your own words are
       never truncated) and `.fixedSize(horizontal: false, vertical: true)`.
   - Padding: horizontal `inspectorQuoteInset` (10 pt), vertical
     `inspectorNoteVerticalPadding` (6 pt). Then
     `.frame(maxWidth: .infinity, alignment: .leading)`.
   - Fill: `RoundedRectangle(cornerRadius: inspectorNoteRadius, style: .continuous)` (8
     pt), filled with `Color.yellow.opacity(inspectorNoteFillOpacity)` (0.14).
   - Border: the same shape with
     `.strokeBorder(Color.yellow.opacity(stroke), lineWidth: 0.5)`. `stroke` is
     `inspectorNoteStrokeOpacity` (0.30), or `inspectorNoteStrokeOpacityIncreased`
     (0.60) when `@Environment(\.colorSchemeContrast) == .increased`. This is the same
     pattern the Copy card uses.
   - Geometry: the card has no extra leading offset. Its left edge lines up with the
     yellow rule (x = 0), and its glyph lines up with the quote text (x = 10).

   The result is a clean two-column grid: the highlighter stroke, and under it a sticky
   note.

Mock (light appearance; `▌` is the yellow rule, `✎` stands for the `note.text` glyph):

```
HIGHLIGHTS 7  (✎ 2)

Page 1
▌ Factual memory tells the agent what it knows.          ← bold run, no **

Page 2
▌ The pipeline drains while the child runs, so the
▌ fetcher never blocks on a full pipe.
╭────────────────────────────────────────────────────╮
│ ✎ Pin this for the memory design doc.              │   ← pale-yellow card
╰────────────────────────────────────────────────────╯

Page 3
▌ A long highlight that runs past four lines ends …
⌄ +4 more · 1 with a note · ⌃D shows 3 more
```

Why this look:

- The yellow family ties the note to its highlight (highlighter, then sticky note).
- Orange is the darker sibling of yellow and stays legible on the pale card in light
  mode. Plain yellow would almost vanish there, and orange stays warm on the dark card.
- `note.text` is literally a sticky note, and it matches the app's own word, "note".
- A speech bubble was rejected because it would echo the chat kind's
  `bubble.left.and.bubble.right`.
- The accent color was rejected because it already means selection, matches, and the
  unopened dot.

### Section header and note chip

- `HStack(spacing: 6)` with these parts:
  - `Text("HIGHLIGHTS")` in `.caption.weight(.semibold)`, `.secondary`;
  - the total, `Text("\(feed.total)")`, in `.caption2.monospacedDigit()`, `.tertiary`;
  - when `feed.notedCount > 0`, the **note chip**:
    - content: `HStack(spacing: 3)` with the `inspectorNoteSymbol` image and
      `Text("\(feed.notedCount)")`, both in
      `.caption2.weight(.semibold).monospacedDigit()`, colored `inspectorNoteTint`;
    - padding: 6 pt horizontal and 1.5 pt vertical;
    - background: a `Capsule` filled with
      `Color.yellow.opacity(inspectorNoteFillOpacity)`.

  The chip doubles as the legend for the card glyph.

- The header is one combined element with the `.isHeader` trait and the label
  `feed.headerAccessibilityLabel`, for example `Highlights, 29, 2 with notes`.
- The header shows whenever the feed is non-empty. The first page always shows at least
  one entry, so the header never fades: drop the header-fade logic.

### Page runs, paging, disclosure, motion

- **Page runs.** The rule now runs over the single stream:
  - an entry shows its label if it is the first entry, or if its label differs from the
    previous entry's label;
  - a nil label never shows, and the next labelled entry shows its label;
  - a run that starts after the first entry gets `inspectorPageRunGap` (4 pt) of extra
    top padding, as today.
- **Paging.** Paging is unchanged. `RefsNotesPage` pages over `feed.total`, and Ctrl+D
  at the very bottom, or a click on the disclosure row, reveals the next page below.
- **Disclosure row.** It keeps today's look and its place at the end of the section. Its
  text is `feed.moreText(visible:nextBatch:)`:
  - nothing hidden → nil (no row);
  - no hidden noted highlights → exactly `RefsNotesPage.moreText(hidden:nextBatch:)`
    (`+24 more · ⌃D shows 3 more`, `+1 more · ⌃D shows it`,
    `+2 more · ⌃D shows the rest`);
  - hidden noted highlights → `+H more · K with notes · <clause>`, where K = 1 reads
    `1 with a note`, and `<clause>` follows today's rule (`⌃D shows it` when H == 1,
    `⌃D shows N more` when H > next batch, `⌃D shows the rest` otherwise).
- **Motion.**
  - Revealed entries keep today's fade (`isRevealed = index >= page.size`, staggered
    from `batchStart`). Reduce Motion and `previewMode` stay static.
  - The reveal anchor goes on the entry whose `index == notesPage.batchStart`, as a
    `.background`, outside the fade.
  - Remove the `RevealAnchor` enum and the `.highlightsHeader` case: one section never
    crosses a header.

### VoiceOver

Each entry is one combined element, labelled `entry.accessibilityLabel` (computed in
RefsCore so it can be tested on Linux). The quote text is the displayed text, with `**`
stripped through `RefsQuoteEmphasis`.

| Entry             | Label                                                                                                                            |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| no note           | `Highlight, Page 3: <quote>` (`Highlight: <quote>` with no page)                                                                 |
| note and quote    | `Highlight with note, Page 3: <quote>. Your note: <comment>` (`Highlight with note: <quote>. Your note: <comment>` with no page) |
| note, empty quote | `Highlight with note, Page 3. Your note: <comment>` (`Highlight with note. Your note: <comment>` with no page)                   |

- The page always appears in the label, even when the visual label is grouped into a
  run. It is left out only when it is nil.
- The comment is trimmed of surrounding whitespace.

Other strings:

- Disclosure row label (`hiddenAccessibilityLabel`):
  - `24 more highlights`;
  - `26 more highlights, 2 with notes`;
  - `1 more highlight, 1 with a note`;
  - nil when nothing is hidden.

  Keep the button trait and the `Show more` action.

- Reveal announcement: `Showing 6 of 29 highlights`.
- Header label (`headerAccessibilityLabel`): `Highlights, 29, 2 with notes`,
  `Highlights, 7`, or `Highlights, 1, 1 with a note`.

### What the section shows (absence stays absence)

- **Included:** `kind == "highlight"` with a non-empty trimmed quote **or** a comment. A
  comment made only of whitespace counts as no comment, so the entry shows as a plain
  highlight.
- **Excluded, as today:** a highlight with an empty quote and no comment, standalone
  `note` annotations, and `image` annotations.
- No highlights → no section. If `ref show` fails, keep today's dim "Notes unavailable".
- The "Notes" fact row (`29 highlights · 2 comments`) is unchanged.
- The Copy card (its YOUR NOTES header and the H picker) is unchanged.

## Implementation

Open the linked repo with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`) and
read its `AGENTS.md` before editing. Paths below are relative to that checkout. Match
the surrounding doc-comment density and naming.

### 0. Test-compile fix (bob-cli-9m)

> [!decision] ci_compile_fix = fix
>
> Do this first, as its own commit, before any feature work. First check that the latest
> master CI run still fails at `RefsNewestTests.swift:12`. If master already compiles,
> skip this step and leave the bead alone.
>
> - `Tests/RefsCoreTests/RefsNewestTests.swift`: rename `static let zone` to
>   `static let newYork` and update its uses. An `XCTestCase` static named `zone`
>   clashes with `NSObject.zone()` on macOS only.
> - `Tests/BobMacCaptureTests/RefsLibraryTests.swift` (lines ~252, 259, 297, 304) calls
>   `RefsNewestTests.instant(...)`, which lives in the separate RefsCoreTests target.
>   Give `RefsLibraryTests` its own private `instant(_:)` helper with the same
>   `ISO8601DateFormatter` (`.withInternetDateTime`) and call that instead.
> - Read bob-cli-9m with `sase bead read bob-cli-9m -r "…"` first. After CI shows the
>   Test step getting past emit-module, close it:
>   `sase bead close bob-cli-9m --note "<sha, CI run id, Test step compiles>"`.

> [!decision] ci_compile_fix = skip
>
> Do not touch `RefsNewestTests` or `RefsLibraryTests`. CI cannot compile the tests, so
> no render fixtures exist. In Verification, skip the render review, and say in the
> final report that the design has no render check because of bob-cli-9m.

### 1. RefsCore: `Sources/RefsCore/RefsInspector.swift` (portable, Linux-tested)

- `RefsShowAnnotation`:
  - Add `public var isInspectorHighlight: Bool`: `kind == "highlight"` and
    (`hasNonEmptyQuote` or `hasComment`). Doc it as the one HIGHLIGHTS section's
    membership rule.
  - Remove `isUncommentedHighlight`.
  - Update the `hasComment` doc comment: it now marks a highlight that shows a note
    card.
- `RefsShowRow`:
  - Add `public var inspectorHighlights: [RefsShowAnnotation]`
    (`annotations.filter(\.isInspectorHighlight)`, note order).
  - Remove `uncommentedHighlights`.
  - Keep `commentedHighlights`, which the Copy card uses, and update its doc comment (it
    no longer feeds an inspector block).
  - Update the `RefsShowResponse` doc comment to describe the single projection.
- Rewrite `RefsInspectorFeed` as one stream:
  - `public let highlights: [RefsShowAnnotation]`, `init(highlights:)`,
    `public var total: Int`, and `public var notedCount: Int` (count of `hasComment`).
  - `public struct Entry: Equatable, Sendable` with `index` (position in the feed),
    `annotation`, and `showsPageLabel`, plus:
    - `public var hasNote: Bool { annotation.hasComment }`;
    - `public var accessibilityLabel: String`, built per the VoiceOver table.
  - `entries(visible:)`: the first `min(max(visible, 0), total)` entries, with the
    single-stream page-run rule.
  - `moreText(visible:nextBatch:)`, `hiddenAccessibilityLabel(visible:)`,
    `revealAnnouncement(visible:)`, and `headerAccessibilityLabel`, with exactly the
    strings in Design. Delegate the no-hidden-notes disclosure case to
    `RefsNotesPage.moreText(hidden:nextBatch:)` so those strings stay identical. Keep
    the private `revealClause` and `counted` helpers where they still help.
  - Delete `Section`, `RevealAnchor`, `revealAnchor(batchStart:)`, and the
    notes/highlights split helpers.
  - Update the type's doc comment.
- Leave `RefsNotesPage`, `RefsQuoteEmphasis`, and `RefsTextRun` unchanged.

### 2. Loader: `Sources/BobMacCapture/Refs/RefsInspectorLoader.swift`

- `RefsInspectorContent`: replace `notes` and `highlights` with one
  `public var highlights: [RefsShowAnnotation]`, which holds every inspector highlight
  in note order. Drop the `notes:` init parameter and keep `highlights:` in its slot
  after `outline:`.
- `feed` becomes `RefsInspectorFeed(highlights: highlights)`.
- `load` sets `built.highlights = row.inspectorHighlights`.

### 3. Model: `Sources/BobMacCapture/Refs/RefsPanelModel.swift`

- `revealMoreInspectorNotes()` already pages over `content.feed.total` and posts
  `revealAnnouncement`, so no logic changes. Keep the name, because the controller, the
  view, and the tests call it. Change its doc comment to say "highlights".

### 4. Tokens: `Sources/BobMacCapture/Refs/RefsVisualTokens.swift`

- Add, with one-line doc comments:
  - `inspectorNoteSymbol = "note.text"`;
  - `inspectorNoteTint: Color = .orange`;
  - `inspectorNoteFillOpacity = 0.14`;
  - `inspectorNoteStrokeOpacity = 0.30`;
  - `inspectorNoteStrokeOpacityIncreased = 0.60`;
  - `inspectorNoteRadius: CGFloat = 8`;
  - `inspectorNoteVerticalPadding: CGFloat = 6`;
  - `inspectorNoteGap: CGFloat = 6`.
- Remove `inspectorNoteQuoteLineLimit`, which has no use left.
- Change `inspectorEntrySpacing`'s doc comment to "between HIGHLIGHTS entries".
- In the type's doc comment on colors, add one clause: yellow (rule, card, and chip) and
  orange (note glyph) mean your highlights and notes in the inspector.

### 5. View: `Sources/BobMacCapture/Refs/RefsInspectorView.swift`

- Replace `notesSection` with `highlightsSection`, a single
  `VStack(alignment: .leading, spacing: inspectorEntrySpacing)` holding:
  - the header and chip;
  - `ForEach(entries, id: \.index)` with `highlightEntryView(entry)`, keeping the
    page-run padding, `RefsInspectorNoteFade`, and the reveal-anchor background on the
    `batchStart` entry;
  - the disclosure row last.

  Keep the `showFailed` "Notes unavailable" branch. Update the `column` call site.

- Collapse `annotationBlock`, `headerFadeValues`, `noteView`, and `highlightView` into
  `highlightEntryView` plus small private helpers: `quoteView`, `noteCard`, and
  `noteChip`.
  - Read `@Environment(\.colorSchemeContrast)` for the card stroke.
  - Use `entry.accessibilityLabel` and remove the view-side label builders
    (`noteAccessibilityLabel`, `highlightAccessibilityLabel`, `displayedQuote`) that
    RefsCore now owns.
- Keep `quoteText(_:)`, `disclosureRow`, `revealDelay`, and `pageRunTopPadding`.

### 6. Settings: `Sources/BobMacCapture/SettingsView.swift`

- Rename the stepper label to "Preview highlights per page". Keep the
  `refsInspectorNotesPageSize` key.
- Change the caption to: "The preview shows this many highlights at first; Control-D at
  the bottom of the preview shows this many more."

### 7. Docs: `README.md`

- **Inspector bullet** (≈ lines 1758–1767). Replace "your annotations in two sections:
  YOUR NOTES … then HIGHLIGHTS …, paged as one sequence" with a description of the new
  section:
  - one HIGHLIGHTS section in note order;
  - a highlight that carries your comment shows it beneath the quote on a pale-yellow
    note card marked with the sticky-note glyph;
  - the header's note chip counts noted highlights;
  - the disclosure row says how many hidden highlights carry notes.

  Keep the page-run, bold, tooltip, paging, and reset sentences, worded for highlights.

- **Keyboard table Ctrl-D row and the paragraph under it** (≈ 1566, 1588): "shows the
  next page of highlights".
- **Settings bullet** (≈ 1688): "Preview highlights per page (1–20, default 3)". Keep
  the `defaults write …refsInspectorNotesPageSize` example.

## Tests

### RefsCore: `Tests/RefsCoreTests/RefsInspectorTests.swift`

These run under `just portable-check`. Replace the two-section tests (fixture
projection, section/page-run, more-text, accessibility, boundary paging, and reveal
anchor) with:

- **Fixture projection.** Use `refs-show-highlights.json`, unchanged:
  - `inspectorHighlights.map(\.quote)` is the five quotes in note order: the Page 1
    commented one, Page 2, the Page 3 whitespace-comment one, and the two Page 5 ones;
  - `inspectorHighlights.filter(\.hasComment).map(\.comment)` is
    `["Keep this in the notes section."]`;
  - `commentedHighlights.count == 1` and `highlightPickerEntries.count == 6`
    (unchanged).
- **`isInspectorHighlight` (inline annotations).** True for a quote without a comment, a
  quote with a comment, and an empty quote with a comment. False for an empty quote with
  no comment, an empty quote with a whitespace-only comment, `note`, and `image`.
- **Page runs over one stream.** Labels `[P1 noted, P2, P2 noted, P3, nil, P3]` give
  `showsPageLabel == [T, T, F, T, F, T]`, `hasNote == [T, F, T, F, F, F]`, and indices
  0–5. Also test that `visible` clamps at -1, 0, total, and 99.
- **`moreText`**:
  - 27 plain, visible 3, next 3 → `+24 more · ⌃D shows 3 more`, equal to
    `RefsNotesPage.moreText(hidden: 24, nextBatch: 3)`;
  - 29 with notes at indices 8 and 13, visible 3, next 3 →
    `+26 more · 2 with notes · ⌃D shows 3 more`;
  - 7 with notes at 1 and 4, visible 3, next 3 →
    `+4 more · 1 with a note · ⌃D shows 3 more`;
  - 2 with a note at 1, visible 1, next 1 → `+1 more · 1 with a note · ⌃D shows it`;
  - 8 all noted, visible 6, next 3 → `+2 more · 2 with notes · ⌃D shows the rest`;
  - a visible note with only plain highlights hidden → no notes clause;
  - nothing hidden → nil.
- **Strings.** `hiddenAccessibilityLabel`, `revealAnnouncement`, and
  `headerAccessibilityLabel` produce every string in Design, singular forms included.
- **`Entry.accessibilityLabel`.** All six shapes in the VoiceOver table, plus one
  `**bold**` quote whose label has no asterisks and one comment with surrounding
  whitespace that comes out trimmed.
- **Paging.** `RefsNotesPage(size: 3)` over a 7-entry feed: `reveal` returns `3..<6`,
  and `entries(visible: 6).count == 6`.

### App: `Tests/BobMacCaptureTests`

- **Mechanical updates.** Every `RefsInspectorContent(… notes: X …)` becomes
  `highlights: X`. That covers `RefsInspectorTests`, `RefsPanelModelTests`,
  `RefsInspectorScrollTests` (rename the harness's `notes:` parameter to `highlights:`),
  and `RefsPanelDesignTests`. Where a call passed both arrays, merge them into one array
  in page order.
- **`RefsInspectorTests` (loader).**
  - The fixture test asserts `content.highlights` quotes in note order (the same five as
    RefsCore) and the single noted comment.
  - The show-failure test still asserts `highlights` empty with `showFailed`.
- **`RefsInspectorTests` (model reveal).**
  - Keep the 5-highlight `3..<5` reveal test, now with highlights 2 and 4 noted.
  - The fits-the-page test uses one 3-entry array (2 noted, 1 plain) and still returns
    false.
- **Placement.** Keep new app tests in `RefsInspectorTests` and `RefsPanelDesignTests`,
  not `RefsPanelModelTests`. bob-cli-7p's xctest crash stops every test that sorts after
  it.
- **Hosted Ctrl+D tests.** Add none; they are red on CI (bob-cli-7h).

### Renders: `Tests/BobMacCaptureTests/RefsPanelDesignTests.swift`

- **In `testRefsPanelStatesToPNG`:**
  - `refs-inspector-chat-880`, `refs-inspector-paper-880`, and
    `refs-inspector-encrypted-880` keep their names. They now show note cards.
  - `refs-inspector-notes-revealed-880` asserts
    `content.feed.moreText(…) == "+2 more · 2 with notes · ⌃D shows the rest"` instead
    of the bare `RefsNotesPage` string.
  - Delete `renderInspectorHighlights`, `renderInspectorNotesAndHighlights`, and
    `renderInspectorHighlightsPending` and their calls; they depict the old split.
- **New method `testRefsInspectorHighlightsToPNG`.** Same model setup as the monolith.
  It is separate so that bob-cli-8j's later failure inside the monolith can never
  swallow these renders. Every render writes light and dark at 880 pt through
  `write(model:name:width:)`. Use realistic synthetic text, never vault content.
  - `refs-inspector-highlights-880`: an article with 7 highlights in page order:
    - P1, plain, with a `**bold**` run;
    - P2, noted with a one-line note;
    - P3, plain, long enough to truncate at 4 lines (reuse `longHighlightQuote`);
    - P3, plain (the same page run);
    - P4, noted with a two-sentence note that wraps;
    - P6, plain;
    - P7, plain.

    First page. Assert `+4 more · 1 with a note · ⌃D shows 3 more`.

  - `refs-inspector-highlights-revealed-880`: the same content after one
    `revealMoreInspectorNotes()`. The same-page run, the second card, and
    `+1 more · ⌃D shows it` are on screen. Assert the string.
  - `refs-inspector-highlights-noted-880`: 3 highlights, all noted:
    - one with a normal quote;
    - one with an empty quote and a multi-line comment (a card directly under its page
      label);
    - one with a bold quote.

    Header chip 3, no disclosure. Assert `moreText` is nil.

## Verification (do not skip; see the repo's AGENTS.md)

1. **Portable targets.** Run `just portable-check` in the linked checkout on athena
   (Swift 6 is installed there). It lints every Swift file and builds and tests RefsCore
   and CaptureCore, including every new RefsCore test.
2. **macOS targets (best effort).**
   - Rsync a scratch copy without `.build`/`.git` to the mac, run `Scripts/bundle.sh`
     and `just typecheck-tests` there, then delete the copy.
   - Never use Bryan's checkout or `just install`.
   - If `typecheck-tests` cannot load the standard library (bob-cli-7o), apply that
     bead's `-sdk "$(/usr/bin/xcrun --sdk macosx --show-sdk-path)"` workaround in the
     scratch copy only.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.
3. **Commit and CI.**
   - Commit in bob-mac-capture with `/sase_git_commit`; this approved plan is the
     explicit request. Under `ci_compile_fix = fix`, the compile fix goes in its own
     commit first.
   - Wait for each `ci.yml` run with `/sase_monitor`; never end the turn to wait. A red
     **Lint Swift formatting** step skips every later step, so check them all.
   - Once tests compile, expect the Test step to stay red because of known beads:
     bob-cli-8j, bob-cli-7h, bob-cli-7p (its crash stops later classes), and bob-cli-8e.
     - Any failure in a test you touched, or in `RefsInspectorTests`,
       `RefsPanelDesignTests` inspector renders, or RefsCore, is yours to fix.
     - For any other failure, use `/sase_new_task`: corroborate the matching bead or
       file a new one. Do not fix it here.
     - Confirm in the log that `testRefsInspectorHighlightsToPNG` and the new
       `RefsInspectorTests` cases ran and passed.
4. **The look (the point of this plan).**
   - Download the PNGs:
     `gh run download <run-id> -R bobs-org/bob-mac-capture -n render-fixtures -D <new empty tmp dir>`.
   - Open every `refs-inspector-*` PNG, light and dark, with the image reader. Check:
     - the card's left edge lines up with the yellow rule, and its glyph lines up with
       the quote text;
     - the glyph sits on the note's first baseline;
     - the card reads as a pale sticky note in light mode and a warm amber tint in dark
       mode, without muddying the text;
     - the orange glyph is clearly visible on the card in both appearances;
     - the header chip matches the card and does not crowd the count;
     - a plain highlight next to a noted one is obviously different from 2 m away;
     - page-run rhythm, 4-line truncation ending in "…", and no stray `**`;
     - nothing is left invisible by the fade in `previewMode`.
   - **Tuning.** You may tune only these values: fill and stroke opacity by ±0.06,
     `inspectorNoteGap` / `inspectorNoteVerticalPadding` / `inspectorEntrySpacing` by ±2
     pt, and the glyph weight or scale. Re-render after each change. Keep every string,
     the glyph, and the structure as written.
   - **Report.** Store the light and dark `refs-inspector-highlights-880` and
     `refs-inspector-highlights-revealed-880` PNGs as artifacts (read the
     `sase_artifacts.md` reference memory first), and list them in the final report so
     Bryan can see the design.

## Out of scope

- bob-cli and the `ref show` contract.
- The Copy card (its YOUR NOTES header and the H picker).
- The Notes fact row.
- Standalone notes and images in the inspector.
- The hosted Ctrl+D test failures (bob-cli-7h) and the other known CI beads.
