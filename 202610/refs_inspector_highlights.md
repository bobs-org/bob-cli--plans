---
tier: tale
title: Bob Refs inspector shows highlights that have no notes
goal:
  The Bob Refs inspector shows every highlight with no comment in a HIGHLIGHTS section
  after YOUR NOTES. Both sections page together with Ctrl+D.
size: medium
proposed_by: bbugyi200.apollo.7s
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.7s](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.7s.md)
- **COMMITS:**
  - [7533626](https://github.com/bobs-org/bob-mac-capture/commit/7533626537d1a51c8a78c220558ef646b33bd663)
    — feat(refs): show uncommented highlights in the inspector

# Bob Refs inspector: show highlights that have no notes

## Goal

In the Bob Refs panel (repo `bob-mac-capture`), the inspector's **YOUR NOTES** block
shows only highlights that carry a comment (`RefsShowRow.commentedHighlights`). A
highlight with no comment shows up only as part of the "Notes: N highlights" fact. Bryan
cannot see any of its text without leaving the panel.

That gap is large. In the live library, 140 of 304 highlights (46%) have no comment. 28
references have only uncommented highlights, so their inspector shows no annotation text
at all. A typical heavily read blog has 29 highlights and 2 comments, and today the
preview shows just those 2.

After this change, the inspector shows those highlights in their own **HIGHLIGHTS**
section, directly after YOUR NOTES. The two sections page as one sequence through the
existing Ctrl+D-at-the-bottom reveal.

`bob` already provides the data. The panel runs `bob ref show <path> -f json` without
`-c`, and that returns every live annotation. An uncommented highlight arrives as
`{"kind":"highlight","quote":"…","comment":null,…}`. **This is a Mac-only change. Do not
change bob-cli.**

### Worked examples (default page size 3)

1. **Only uncommented highlights** (5 highlights, no comments). Today the preview shows
   no annotations. After the change it shows `HIGHLIGHTS 5`, the first 3 highlights, and
   `+2 more · ⌃D shows the rest`. Scroll to the bottom and press Ctrl+D: all 5 show, and
   the disclosure row goes away.
2. **Mixed** (29 highlights, 2 commented). The preview shows `YOUR NOTES 2` with both
   notes, then `HIGHLIGHTS 27` with 1 highlight, then `+26 more · ⌃D shows 3 more`. Each
   Ctrl+D at the bottom reveals 3 more highlights. Consecutive highlights from the same
   page sit under a single `Page 3` label.
3. **Notes fill the first page** (16 commented, 6 not). The preview shows
   `YOUR NOTES 16`, 3 notes, and `+13 notes · 6 highlights · ⌃D shows 3 more`. Ctrl+D
   pages through the notes. The page after the last note brings in the HIGHLIGHTS
   header, which fades in with its first rows.

## Design (decided; implement as written)

### Why two sections and one pager

- **Your own words stay first.** Today the 14 mixed references lead with Bryan's
  comments. Interleaving all highlights in page order would push those comments down
  behind author text, so the plan rejects interleaving. YOUR NOTES stays exactly where
  it is, and HIGHLIGHTS holds the rest: highlights with no comment. Commented highlights
  are never shown twice, so `YOUR NOTES n` plus `HIGHLIGHTS m` adds up to the highlight
  count.
- **One forward motion.** The feed is notes first, then highlights, and the existing
  `RefsNotesPage` pages through that whole feed. Ctrl+D at the bottom always reveals the
  next entries _below_ what is already shown. The follow-the-tail scroll, the fade-in,
  and the reveal anchor all keep working unchanged, keyed by an entry's index in the
  feed.
  - Rejected: one pager per section. Ctrl+D at the bottom would then have to reveal
    hidden notes _above_ the viewport.
  - Rejected: a toggle key. It adds a mode, and printable keys belong to the search
    field.
- **Highlights fill whatever the first page leaves.** If a reference has few notes, its
  highlights appear at once. If it has many notes, the notes keep the first page and the
  disclosure row says what is still to come.

### Visuals

- **Section header.** `HIGHLIGHTS` uses the same style as `YOUR NOTES`: caption semibold
  secondary, then the total highlight count in `caption2.monospacedDigit()` tertiary,
  with spacing 6. It carries the `.isHeader` trait, and its accessibility label is
  "Highlights, 27".
  - The header shows only once at least one highlight is visible.
  - The HIGHLIGHTS block is its own child of the inspector column. It therefore sits
    `inspectorSpacing` (14 pt) below YOUR NOTES, like every other inspector section.
- **Highlight entry: the quote is the content.**
  - Font and color: `.callout`, upright (not italic), `.primary`, with `lineSpacing(2)`.
  - Length: up to 4 lines (`inspectorHighlightLineLimit`), truncated at the tail.
  - Layout: the same 10 pt leading inset and 2 pt yellow rule that note quotes already
    use, so the yellow bar keeps meaning "text you highlighted".
  - Hover: `.help(quote)` shows the full text, so a truncated highlight is never lost.
  - Contrast with notes: in YOUR NOTES the quote is context, so it stays italic,
    secondary, and limited to 3 lines, with the comment as the primary text. Do not
    change the note entry's look.
  - Measured lengths: the median uncommented highlight is about 130 characters, and the
    75th percentile is about 215. Four lines (about 55 characters each at the
    inspector's width) therefore show most highlights in full.
- **Page labels once per run.** Within each section, an entry shows its page label
  (`displayPageLabel`, `.caption`, secondary) only in two cases: it is the section's
  first entry, or its label differs from the previous entry's label.
  - A run that starts after the first one gets 4 pt of extra top padding
    (`inspectorPageRunGap`), so page groups read as groups.
  - The rule applies to YOUR NOTES too. Each section starts with its own label, so a
    highlight on Page 2 that follows a note on Page 2 still shows "Page 2".
  - A nil label never renders.
- **Bold runs, not asterisks.** Highlights exports keep the PDF's bold as `**…**`. This
  happens in 7 of the 304 live highlights, for example
  `**Factual memory tells the agent what it knows.**`. The inspector renders each
  balanced `**…**` pair as strong emphasis
  (`inlinePresentationIntent = .stronglyEmphasized`, the same mechanism
  `CapturePickerRichText` uses) and removes the delimiters.
  - This applies to quotes in both sections. An italic note quote becomes bold italic in
    those runs.
  - It does not apply to comments.
  - The Copy card is unchanged and keeps copying raw text.
- **Disclosure row.** It has the same look and position as today: 8 pt below the last
  visible entry, inside the last visible block. Its text names the kinds only when the
  header above it does not already make them clear:
  - Hidden entries in both sections: `+13 notes · 6 highlights · ⌃D shows 3 more`.
  - Only highlights hidden, and no highlight visible yet (the HIGHLIGHTS header is not
    showing): `+5 highlights · ⌃D shows 3 more`.
  - Every other case: today's exact strings from
    `RefsNotesPage.moreText(hidden:nextBatch:)`: `+24 more · ⌃D shows 3 more`,
    `+1 more · ⌃D shows it`, and `+2 more · ⌃D shows the rest`.
  - Use singular forms where they apply: `1 note`, `1 highlight`. The `⌃D` clause
    follows today's rule: hidden == 1 gives `shows it`, hidden > next page gives
    `shows K more`, and anything else gives `shows the rest`.
- **Reveal motion.**
  - Revealed entries keep today's fade, keyed by feed index:
    `isRevealed = index >= page.size`, with a stagger counted from `batchStart`.
  - The HIGHLIGHTS header uses the same fade values as the first highlight entry, so the
    header and its first rows arrive together. Under Reduce Motion and in `previewMode`,
    everything renders static, as today.
  - The reveal anchor normally sits on the entry at `batchStart`. When that entry is the
    first highlight (the batch crosses into HIGHLIGHTS), put the anchor on the
    HIGHLIGHTS header instead, so the follow-scroll keeps the header in view.
  - The anchor always stays outside the fade modifiers.
- **VoiceOver.**
  - A highlight entry is one combined element with the label "Highlight, Page 3:
    <quote>". It always includes the page, even when the label is grouped visually.
    Leave the page out only when it is nil.
  - A note entry becomes one combined element: "Page 2: <quote>. Your note: <comment>".
    This is needed because grouping can now hide its visible label. With no page, start
    at the quote; with no quote, give just the note.
  - Disclosure row accessibility label: `5 more notes`, `24 more highlights`, or
    `13 more notes and 6 highlights`, with singular forms where they apply. Keep the
    button trait. Rename the named action to "Show more".
  - Reveal announcement: `Showing 6 of 8 notes` (no highlights, unchanged),
    `Showing 6 of 27 highlights` (no notes), or `Showing 6 of 29 notes and highlights`
    (both).
- **Absence stays absence.**
  - With no notes, there is no YOUR NOTES header.
  - With no highlights, the panel behaves exactly as today.
  - When a `ref show` fails, keep today's dim "Notes unavailable".
  - Standalone notes and image selections stay out of the inspector, as today.
    - The live library has 4 standalone notes, and 2 of them are leaked marker mirrors.
    - The live library has 1 image.

## Implementation

Open the linked repo with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`) and
read its `AGENTS.md` first. All paths below are relative to that checkout. Follow the
surrounding code's doc-comment density and naming.

### 1. RefsCore (portable, Linux-testable): `Sources/RefsCore/RefsInspector.swift`

- `RefsShowAnnotation.isUncommentedHighlight: Bool`: `kind == "highlight"`, then
  `!hasComment`, then a non-empty trimmed quote (reuse the private `hasNonEmptyQuote`).
  A whitespace-only comment counts as uncommented.
- `RefsShowRow.uncommentedHighlights: [RefsShowAnnotation]`, in note order. Update the
  `RefsShowResponse` doc comment ("projects `commentedHighlights` for "Your notes"") to
  name both projections.
- New `public struct RefsInspectorFeed: Equatable, Sendable`, a pure projection:
  - `init(notes:highlights:)`, `let notes`, `let highlights`, `var total: Int`.
  - `enum Section { case notes, highlights }`, and
    `struct Entry: Equatable, Sendable { index, section, annotation, showsPageLabel }`.
    Here `index` is the entry's position in the whole feed.
  - `func entries(visible: Int) -> [Entry]` returns the first
    `min(max(visible, 0), total)` entries in feed order (notes first). `showsPageLabel`
    follows the page-run rule in Visuals and is false when `displayPageLabel` is nil.
  - `func moreText(visible: Int, nextBatch: Int) -> String?`: the disclosure strings in
    Visuals. Return nil when nothing is hidden. Delegate the bare `+N more` case to
    `RefsNotesPage.moreText(hidden:nextBatch:)` so those strings stay identical.
  - `func hiddenAccessibilityLabel(visible: Int) -> String?` and
    `func revealAnnouncement(visible: Int) -> String`, using the strings in Visuals.
  - `enum RevealAnchor: Equatable { case entry(Int), highlightsHeader }` and
    `func revealAnchor(batchStart: Int) -> RevealAnchor`. It returns `.highlightsHeader`
    when `batchStart == notes.count` and highlights exist; otherwise it returns
    `.entry(batchStart)`. The view consults it, so the header-anchor rule is unit-tested
    in RefsCore. Hosted Ctrl+D tests cannot pin it today (see Tests).
- New `public enum RefsQuoteEmphasis` with
  `static func runs(_ text: String) -> [RefsTextRun]`, where
  `public struct RefsTextRun: Equatable, Sendable { text: String; isStrong: Bool }`:
  - Pair `**` delimiters from left to right. The text between a pair becomes a strong
    run, and the delimiters disappear.
  - A final unpaired `**` stays literal, together with the text after it.
  - An empty pair (`****`) disappears.
  - Drop empty runs, and merge adjacent runs that have the same strength.
  - Text without `**` returns one plain run, or `[]` when empty.
  - Single `*`, `_`, `__`, and backticks stay literal. PDF quotes use them as real
    characters (footnote stars, snake_case).

### 2. Loader: `Sources/BobMacCapture/Refs/RefsInspectorLoader.swift`

- `RefsInspectorContent` gains `public var highlights: [RefsShowAnnotation]`. Add the
  init parameter `highlights: [RefsShowAnnotation] = []` right after `notes`, so
  existing labeled call sites still compile. Also add
  `public var feed: RefsInspectorFeed { RefsInspectorFeed(notes: notes, highlights: highlights) }`.
- `load` sets `built.highlights = row.uncommentedHighlights` next to `built.notes`.

### 3. Model: `Sources/BobMacCapture/Refs/RefsPanelModel.swift`

- `revealMoreInspectorNotes()` pages over `content.feed.total` instead of
  `content.notes.count`, in all three places. It posts
  `content.feed.revealAnnouncement(visible:)`.
- Keep the method name, because the controller, the view, and the tests call it. Update
  its doc comment to say "notes and highlights". Nothing else in the reveal path
  changes. That includes `RefsInspectorNotesReveal`, the reset rules, and
  `notesPageSize`.

### 4. Tokens: `Sources/BobMacCapture/Refs/RefsVisualTokens.swift`

Add named statics for the literals this touches, and use them in the view:

- `inspectorQuoteRuleWidth = 2`
- `inspectorQuoteInset = 10`
- `inspectorNoteQuoteLineLimit = 3`
- `inspectorHighlightLineLimit = 4`
- `inspectorPageRunGap: CGFloat = 4`
- `inspectorEntrySpacing: CGFloat = 8` (today's literal section spacing)

### 5. View: `Sources/BobMacCapture/Refs/RefsInspectorView.swift`

- Replace `notesSection` with a `@ViewBuilder` that emits up to two sibling blocks: YOUR
  NOTES (when `feed.notes` is non-empty) and HIGHLIGHTS (when at least one highlight
  entry is visible). Use
  `feed.entries(visible: notesPage.visibleCount(total: feed.total))`.
  - Each block is a `VStack(alignment: .leading, spacing: inspectorEntrySpacing)` with
    its header and its entries. `ForEach` over the entries uses `entry.index` as the id.
  - The disclosure row goes at the end of the last emitted block. It uses the
    `feed.moreText(visible:nextBatch:)` text, the `hiddenAccessibilityLabel`, the hover
    and click behavior, and the "Show more" action.
  - Keep the `showFailed` branch.
- `noteView` reads `showsPageLabel` from the entry, uses the shared quote rendering
  (italic, secondary, 3 lines, the rule), and gets the combined accessibility label.
- New `highlightView` for the highlight entry described in Visuals.
- New private helper `quoteText(_:) -> AttributedString`. It maps
  `RefsQuoteEmphasis.runs` to runs with `.stronglyEmphasized` applied where needed. Both
  entry views render quotes through it.
- The fade and the anchor follow Visuals → Reveal motion. Reuse `RefsInspectorNoteFade`
  for the header, with the first highlight's `isRevealed` and delay. Attach
  `RefsInspectorRevealAnchor(generation:)` as a `.background` to whatever
  `feed.revealAnchor(batchStart:)` names, and always outside the fade.
- `RefsPanelView` needs no change. It already passes `notesPage` and the reveal closure.

### 6. Settings: `Sources/BobMacCapture/SettingsView.swift`

- Keep the label ("Preview notes per page") and the key unchanged. Change the caption
  to: "The preview shows this many of your notes and highlights at first; Control-D at
  the bottom of the preview shows this many more."

### 7. Docs: `README.md`

- **Inspector** bullet. Replace "your notes with their page labels, quotes, and
  comments" with a short description of:
  - YOUR NOTES (highlights with a comment), then HIGHLIGHTS (highlights without one),
    paged as one sequence;
  - one page label per run of same-page entries;
  - bold runs;
  - the hover tooltip with the full highlight.

  Keep the paging sentence ("Ctrl-D at the very bottom reveals the next page") and
  extend it to notes and highlights.

- **Keyboard** table, Ctrl-D row, and the paragraph under the table: "shows the next
  page of your notes and highlights".
- **Opening Bob Refs** settings bullet: Preview notes per page counts notes and
  highlights alike.
- No Privacy change. No new keys or files.

## Tests

### RefsCore (`Tests/RefsCoreTests/RefsInspectorTests.swift`; runs under `just portable-check`)

- Add the fixture `Tests/Fixtures/refs-show-highlights.json`.
  - Use the real `ref show` shape, with `"comment": null` on uncommented highlights,
    `page_label` values like `"Page 3"`, and `block_id`/`link`.
  - Use synthetic text. Do not copy vault content.
  - Include, in order: a commented highlight, an uncommented highlight, one with a
    whitespace-only comment, one with an empty quote, a standalone note, an image, and
    two more uncommented highlights on the same page.
  - Decode it and assert `commentedHighlights`, `uncommentedHighlights` (the
    whitespace-comment one included; the empty-quote one, the note, and the image
    excluded; note order kept), and the unchanged `highlightPickerEntries` count.
- `RefsInspectorFeed.entries`:
  - With notes `[Page 2, Page 6]` and highlights
    `[Page 1, Page 2, Page 3, Page 3, Page 3]`: indices 0–6, sections, and
    `showsPageLabel` flags `[T, T, T, T, T, F, F]`.
  - A highlight on the same page as the last note still shows its label.
  - A nil label never shows.
  - `visible` is clamped at 0 and at `total`.
- `moreText` pins these strings:
  - notes 0, highlights 27, visible 3, next 3 → `+24 more · ⌃D shows 3 more`
  - notes 16, highlights 6, visible 3, next 3 →
    `+13 notes · 6 highlights · ⌃D shows 3 more`
  - notes 3, highlights 5, visible 3, next 3 → `+5 highlights · ⌃D shows 3 more`
  - notes 1, highlights 1, visible 1, next 1 → `+1 highlight · ⌃D shows it`
  - notes 2, highlights 2, visible 1, next 1 →
    `+1 note · 2 highlights · ⌃D shows 1 more`
  - notes 8, highlights 0, visible 3, next 3 → `+5 more · ⌃D shows 3 more` (equal to
    `RefsNotesPage.moreText`)
  - nothing hidden → nil
- `hiddenAccessibilityLabel` and `revealAnnouncement`: every string in Visuals,
  including the singular forms.
- `RefsQuoteEmphasis.runs`:
  - plain text
  - one pair inside a sentence
  - `**A** **B**` → strong A, plain space, strong B
  - a trailing unpaired `**` stays literal
  - `a ** b` stays literal
  - `****` → `[]`
  - an empty string → `[]`
  - single `*` and `_` stay literal
- Paging across the boundary: with `RefsNotesPage(size: 3)` over a feed of 2 notes and 5
  highlights, `reveal` returns `3..<6`. Then `entries(visible: 6)` is 2 notes plus 4
  highlights.
- `revealAnchor(batchStart:)`:
  - 2 notes and 5 highlights: `batchStart` 2 → `.highlightsHeader`; 3 → `.entry(3)`; 1 →
    `.entry(1)`.
  - 0 notes and 5 highlights: `batchStart` 3 → `.entry(3)`. A batch can only start at 0
    on the first page, which never reveals.
  - 4 notes and 0 highlights: `batchStart` 4 → `.entry(4)`.

### App (`Tests/BobMacCaptureTests`)

macOS CI is red on master for two pre-existing reasons. Plan the app tests around both,
and do not try to fix either one here.

- **bob-cli-7h.** Every hosted Ctrl+D test in `RefsInspectorScrollTests` fails on CI:
  the hosted inspector measures `docHeight=239` and never reveals. **Add no new hosted
  Ctrl+D test.** The anchor rule is pinned in RefsCore instead.
- **bob-cli-7p.** `RefsPanelModelTests.testQueuedDigitCopiesWhenTheShowLoaderReturns`
  crashes xctest (Index out of range). No test that sorts after it runs on CI, and that
  includes every existing `testReveal…` model test. **Put the new app tests in
  `RefsInspectorTests`**, which runs before the crash, and not in `RefsPanelModelTests`.
  `RefsPanelDesignTests` also runs before the crash.

The tests:

- `RefsInspectorTests` (loader):
  - Serve `refs-show-highlights.json` through the existing fetcher stub.
  - Assert that hydrated content carries both `notes` and `highlights`, in order.
  - Assert that a `ref show` failure still leaves `highlights` empty with `showFailed`.
- `RefsInspectorTests` (model reveal; reuse the model setup the copy-action tests in
  that file already use):
  - With 0 notes and 5 highlights, `revealMoreInspectorNotes()` returns true and emits
    range `3..<5` on `inspectorNoteReveals`.
  - With 2 notes and 1 highlight (total equal to the page size), it returns false.
- `RefsPanelDesignTests`: add three renders to `testRefsPanelStatesToPNG`. Each writes
  light and dark at 880 pt through the existing `write(model:name:width:)`. Use
  realistic synthetic text: one `**bold**` run, one highlight long enough to truncate at
  4 lines, and a run of three highlights on one page.
  - `refs-inspector-highlights-880`: an article with 0 notes and 7 highlights, first
    page. Assert `+4 more · ⌃D shows 3 more`.
  - `refs-inspector-notes-and-highlights-880`: 2 notes and 6 highlights after one
    reveal, so both headers, the grouped page labels, and `+2 more · ⌃D shows the rest`
    are on screen. Assert the string.
  - `refs-inspector-highlights-pending-880`: 3 notes and 5 highlights, first page.
    Assert `+5 highlights · ⌃D shows 3 more`.

## Verification (do not skip; see the repo's AGENTS.md)

1. **Portable targets.** Run `just portable-check` on a host with Swift. This host has
   no `swift`, so use athena. Rsync a scratch copy without `.build`/`.git`, run it
   there, then delete the copy. It lints every Swift file and builds and tests RefsCore
   and CaptureCore, including every new RefsCore test above.
2. **macOS targets.** Rsync a scratch copy without `.build`/`.git` to the mac, then run
   `Scripts/bundle.sh` and `just typecheck-tests`. Delete the copy afterwards. Never use
   Bryan's checkout or `just install`. If the mac is offline, say in the final report
   that the macOS targets are unverified.
   - If `typecheck-tests` fails with "unable to load standard library" (bob-cli-7o, open
     at planning time), apply that bead's
     `-sdk "$(/usr/bin/xcrun --sdk macosx --show-sdk-path)"` workaround **in the scratch
     copy only**, then rerun. Do not commit the script change; bob-cli-7o owns it.
3. **CI and the look.** Once the commit is on GitHub:
   - Wait for the `ci.yml` run with `/sase_monitor`; never end the turn to wait. A red
     **Lint Swift formatting** step skips the later steps, so check those steps too.
   - Expect the **Test** step to fail anyway, because of bob-cli-7h and bob-cli-7p.
     Compare the failure list with the master baseline (run 38665543905): failures only
     in the hosted `RefsInspectorScrollTests` and then the
     `testQueuedDigitCopiesWhenTheShowLoaderReturns` crash.
     - Any other failure is yours to fix.
     - Confirm in the log that the new `RefsInspectorTests` and `RefsPanelDesignTests`
       cases executed and passed. The RefsCore suite must be fully green.
   - Download the `render-fixtures` artifact. It is uploaded even when Test fails
     (`gh run download <run-id> -R bobs-org/bob-mac-capture -n render-fixtures -D <tmp dir>`).
   - Open the six new PNGs (light and dark) with an image reader, and also the existing
     `refs-inspector-notes-revealed-880` and `refs-inspector-paper-880`. Check:
     - the rule aligns with the first text line;
     - the page-run rhythm, including the 4 pt gap;
     - the 4-line truncation ends in "…";
     - bold runs show with no stray `**`;
     - the HIGHLIGHTS header sits 14 pt below YOUR NOTES;
     - quote and secondary text keep their contrast on the dark material;
     - in `previewMode`, no header or entry is left invisible by the fade.

     Fix anything off, and repeat the check.

   - If the commit does not reach GitHub during the run, say in the final report that
     the render fixtures were not reviewed.

## Out of scope

- bob-cli. `ref show` already returns every highlight, so the JSON contract does not
  change.
- Standalone notes and image selections in the inspector. They stay as they are. The
  leaked marker-mirror notes are bob-cli-4r, reopened during planning.
- The pre-existing CI failures: bob-cli-7h, bob-cli-7p, and bob-cli-7o.
- The Copy card. Its H picker already lists every highlight, and copied text stays raw.
- Opening the PDF at a highlight's page on click.
