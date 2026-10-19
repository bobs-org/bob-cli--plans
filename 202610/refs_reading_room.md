---
tier: epic
title: 'Bob Refs reading room: near-full-screen panel, 5-per-page previews, inline
  images'
goal: 'The Bob Refs panel in Bob Mac Capture opens nearly full screen (1344 × 798
  pt on Bryan''s 1440 × 870 pt MacBook), its preview column shows 5 annotations per
  page and reveals 5 more on each Ctrl+D, papers, articles, and docs show a larger
  150 × 194 pt first-page thumbnail beside the facts, and image highlights and image
  notes render inline with stable layout, so Ctrl+D paging never jumps.

  '
decisions:
  standalone_notes:
    ask: Should standalone notes join the preview feed as note cards, in note order?
    choices:
      include: Notes show as yellow note cards between highlights; the feed previews
        every annotation
      exclude: The feed shows only highlights and image highlights; standalone notes
        stay Copy-card only
    default: include
    why: A complete preview beats a partial one, and only 4 standalone notes exist
      today
    answer: include
phases:
- id: refs-core-model
  title: Portable RefsCore model, geometry, and 5-per-page default
  depends_on: []
  size: medium
  description: 'refs-core-model: decode image fields from ref show, define the generalized
    preview feed (image highlights, attached images, unavailable images, standalone
    notes), add pure panel/inspector geometry, image sizing, and vault-path resolution
    in RefsCore with Linux tests, and raise the page size default to 5.'
- id: big-panel
  title: Near-full-screen panel, wide inspector, larger thumbnail
  depends_on:
  - refs-core-model
  size: medium
  description: 'big-panel: wire the new geometry into the panel and column split,
    add the wide inspector layout (hero with a 150 × 194 thumbnail beside the facts,
    readable 720 pt column, body typography), render thumbnails at the new size behind
    a versioned cache, let Ctrl+D reveal while the disclosure row is visible, and
    refresh fixtures and README.'
- id: inspector-images
  title: Inline images and standalone notes in the preview feed
  depends_on:
  - refs-core-model
  - big-panel
  size: medium
  description: 'inspector-images: add an ImageIO-backed image loader with metadata
    prefetch so every image reserves its exact size before layout, render image highlights,
    attached images, unavailable-image rows, and standalone notes in the feed, and
    cover them with loader, hosted reveal, and render-fixture tests plus README updates.'
proposed_by: bbugyi200.athena.11v.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 12:32:43
status: done
bead_id: bob-cli-9v
---

- **PROMPT:** [prompts/202610/refs_reading_room.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/refs_reading_room.md)
- **BEAD:** [bob-cli-9v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9v/README.md)

# Plan: Bob Refs reading room

## Where the work happens

All code changes land in the **`bob-mac-capture`** linked repository. Open it with the
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<reason>"`), work only in the
path it prints, and read its `AGENTS.md` before editing. **No bob-cli change is
needed:** `bob ref show <path> -f json` already emits every field this plan consumes
(verified on the live vault, see "Data contract" below).

Compile rules from the repo's `AGENTS.md` apply to every phase and are not optional:

- Run `just portable-check` on athena (Swift 6.0.3 is installed there). It lints every
  Swift file and builds and tests `CaptureCore` and `RefsCore`.
- `portable-check` cannot type-check `Sources/BobMacCapture*` or `Tests/BobMacCapture*`.
  When a phase changes either, compile a scratch copy on the `mac` tailnet host (best
  effort): rsync the repo without `.build` and `.git` to a temporary directory, run
  `Scripts/bundle.sh` and `just typecheck-tests` there, then delete the copy. Never use
  Bryan's checkout on the Mac and never run `just install`. If the Mac is offline, say
  in the final report that the macOS targets are unverified.
- XCTest runs only in GitHub Actions CI (`macos-26`). When your commit is on origin,
  confirm the CI run for it is green (wait with `/sase_monitor`, for example on
  `gh run watch`), fix any failure inside your phase, and look at the PNGs you added or
  changed in the uploaded `render-fixtures` artifact. If your commit is not on origin
  yet when you finish, say so in the report so the land step checks CI. After a red
  **Lint Swift formatting** step, check the later steps too.

## Today (facts the design builds on)

- Panel: `RefsVisualTokens` (`Sources/BobMacCapture/Refs/RefsVisualTokens.swift`) fixes
  the panel at **880 × 560 pt**, clamped to `visibleFrame − 80`, top edge at 16 % of the
  visible height. The list column is `round(width × 0.52)`; the inspector column shows
  at ≥ 760 pt. `RefsPanelController.orderFront` (`RefsPanel.swift`) recomputes the frame
  on every show from the screen under the mouse.
- Bryan's Mac (checked over SSH): one built-in display, 1440 × 900 pt, visible frame
  **1440 × 870** at origin (0, 0) (30 pt menu bar, Dock auto-hidden). The
  `refsInspectorNotesPageSize` default is **not set** there, so changing the code
  default changes what Bryan sees.
- Inspector: `RefsInspectorView` is one `ScrollView` column: hero (112 × 145 thumbnail
  or kind tile + chips + title), byline, facts grid, SUMMARY/ABSTRACT, CONTENTS, one
  HIGHLIGHTS feed, missing-PDF callout, footer. Quotes use `.callout`, 4-line limit;
  comments render as pale-yellow note cards. The feed pages through `RefsNotesPage`
  (`RefsCore/RefsInspector.swift`, `defaultSize = 3`, range 1…20, Settings › References
  › "Preview highlights per page").
- Ctrl+D (`RefsPanelController.scrollInspector`) scrolls half a viewport; only at the
  very bottom (or while a reveal is pending) does it call `revealMoreInspectorNotes()`.
  `followReveal` then pins the first revealed entry (a `RefsInspectorRevealAnchor`) near
  the top with 24 pt of context, measuring layout within 0.5 s. **Any height change
  after that measurement (for example an image that loads late) breaks the follow.**
- Feed membership is `RefsShowRow.inspectorHighlights`: `kind == "highlight"` with a
  non-empty quote or comment. Image highlights (`kind == "image"`), attached images, and
  standalone notes (`kind == "note"`) never appear. `RefsShowAnnotation` does not decode
  `asset`, `images`, or `unresolved_images`.
- Thumbnails: `RefsPDFIntrinsicsLoader` renders page 1 at 224 × 290 px (112 × 145 @2x)
  and caches by (path, size, mtime) in memory and on disk under
  `~/Library/Caches/org.bobs.bob-mac-capture/refs/` (`intrinsics.json` + `thumbs/`).
- Design fixtures: `RefsPanelDesignTests` renders PNGs through `ImageRenderer` at 880 pt
  (and 700 pt list-only); `ImageRenderer` cannot host `ScrollView`, PDFKit, or async
  loads, so fixtures inject content and images synchronously (`previewMode`).

## Data contract (`bob ref show -f json`, schema 1, unchanged)

Each annotation carries `page_label`, `kind` (`highlight` | `note` | `image`), `quote`,
`comment`, `asset`, `images`, `unresolved_images`, `block_id`, `link`. Real rows from
the vault today:

```json
{"page_label":"Page 5","kind":"image","quote":null,"comment":null,
 "asset":"ref/docs/gastown_readme.assets/h-9584a9d023de.jpg",
 "images":[],"unresolved_images":[],"block_id":"h-9584a9d023de"}
{"page_label":"Page 9","kind":"highlight",
 "quote":"Gas Town crew, Mayor, and Beads Crew tmux sessions","comment":null,"asset":null,
 "images":["ref/blogs/gas_town_emergency_user_manual.assets/20261019_062455-ec3b885e.png"],
 "unresolved_images":[],"block_id":"h-36fa970fb80b"}
```

- `asset` (image highlights): a vault-relative path of a region snapshot from the PDF.
- `images`: image notes Bryan attached to an annotation (bob copies them into
  `<note stem>.assets/` and embeds them); they are Bryan's own content.
- `unresolved_images`: sources bob could not import (not found, unreadable, or an
  unsupported type such as HEIC); the per-image state is not serialized.
- Supported embed types are png, jpg, jpeg, gif, webp, svg (bob's `image_notes.rs`).
  Assets currently in the vault: 2 jpg + 1 png. Standalone notes: 4.

Reading these image files from the vault is presentation, exactly like the PDFKit
thumbnail reads the app already does. bob stays the authority on which images belong to
which annotation, and the app never writes the vault.

## The design

### 1. The panel becomes a reading room

Size adapts to the visible frame of the screen under the mouse, recomputed on every
show:

- `width = min(1440, visible.width − 2 × 48)`,
  `height = min(1000, visible.height − 2 × 36)`, each floored to whole points and never
  negative.
- Centered on both axes: `x = minX + ((visible.width − width) / 2).rounded()`, same for
  y. (The old 16 % top offset suited a small Spotlight-like slab; a panel this large
  looks right centered.)
- Bryan's MacBook (1440 × 870 at 0,0) → **frame (48, 36, 1344, 798)**: 48 pt side
  margins, 36 pt top and bottom. A 1920 × 1055 display → 1440 × 983. A 2560 × 1415
  display → 1440 × 1000, centered.
- Column split: `listWidth = min(round(width × 0.52), 560)`. At 880 pt this is still 458
  (all compact fixtures and Copy-card math stay valid); at 1344 the list is 560 and the
  inspector column is `1344 − 16 − 560 − 0.5 = 767.5` pt.
- The inspector still appears at ≥ 760 pt panel width; list-only mode below that is
  unchanged. Escape, resign-key hiding, and the fade/scale-in behave as today.

### 2. Two inspector layouts, chosen by pure geometry

`RefsInspectorLayout` is computed from the panel width (no `GeometryReader` inside the
scroll view, so fixtures and live panels agree):

- **wide** when the inspector column is ≥ 568 pt (that is, content ≥ 520 pt): padding
  24, content width `min(column − 48, 720)`, and the content column is centered in the
  inspector when the column is wider than 720 + 48 (large external displays). Bryan's
  Mac gets wide with a 719.5 pt content width.
- **compact** otherwise: today's layout, unchanged (padding 18, 112 × 145 thumbnail,
  `.callout`, 4-line quotes). A 1280 pt-wide screen yields a 1184 pt panel and stays
  wide (column 607.5).

Wide inspector, top to bottom:

```
┌──────────────┐  [Paper] [Ready] [● New] [Today in BLOG]
│              │  Title in .title2 semibold, up to 3 lines
│   page 1     │  Author · 2024-05-01 · arxiv.org · arXiv:2405.0001
│  150 × 194   │
│  thumbnail   │     Added  Oct 16 at 5:15 PM
│              │     Pages  22 · ≈ 70 min · 3 Pomodoros
│              │    Opened  2 days ago · 3 times
└──────────────┘     Notes  15 highlights · 8 comments
                Reading task  📖 sase_blog · Reading
ABSTRACT  (body text)
CONTENTS  › headings
HIGHLIGHTS 15  [📝 8]
Page 5
┃ quote text in .body, line spacing 3, up to 6 lines …
  ┌ 📝 your comment, then any attached images, inside the yellow card ┐
Page 9
┃ [image highlight, fitted to the column, rounded 6, hairline]
⌄ +10 more · 2 with notes · 1 image · ⌃D shows 5 more
why-here line · PDF path
```

- Hero: thumbnail (150 × 194 pt, US-letter ratio like today's 112 × 145, corner radius
  6, hairline stroke, a soft shadow) on the left. Chips, title, byline, and the facts
  grid sit in the right column, so the taller thumbnail is matched by content instead of
  whitespace. Chats keep the 44 pt kind tile with the same right column.
- Typography (wide only): title `.title2.weight(.semibold)`, 3 lines; summary, quotes,
  and note cards `.body`; quote line spacing 3; quote line limit 6 (the full text stays
  in the hover tooltip); section spacing 18; entry spacing 12. Section labels, chips,
  facts, and the footer keep their current fonts.
- The Copy, Rename, and Today Link cards that replace the inspector keep their current
  views but are constrained to the same readable content column (max 720, centered), so
  nothing stretches edge to edge on large displays.

### 3. Five per page, and Ctrl+D does what the row says

- `RefsNotesPage.defaultSize` becomes **5** (first page and each reveal). The Settings
  stepper (1–20) keeps working and an explicit stored value still wins.
- While the disclosure row (`+N more · ⌃D shows …`) is fully visible in the viewport,
  Ctrl+D reveals the next page at once instead of first scrolling the last few points of
  footer. At the very bottom it reveals as today; elsewhere it scrolls half a viewport
  as today. The follow-tail scroll and the per-entry fade are unchanged.

### 4. Images, rendered with stable layout

Feed entries (`RefsInspectorFeed.Entry`) gain a kind, in note order:

- **highlight**: member when it has a non-empty quote, a comment, attached images, or
  unresolved images. Renders page label (page-run rule unchanged), quote, then a note
  card when it has a comment, attached images, or unresolved images.
- **image** (image highlight): member when `asset` is non-empty. The image takes the
  quote's slot with the same 2 pt yellow rule on its left (it _is_ the highlighted
  content), then a note card when it has a comment, attached images, or unresolved
  images, exactly like a highlight.
- **note** (standalone note), per the decision below. Renders page label + note card.

> [!decision] standalone_notes = include `kind == "note"` entries with a non-blank
> comment, attached images, or unresolved images join the feed in note order and render
> as a note card with no quote. They count toward `notedCount`, the header chip, and the
> "with notes" clause. The header label stays `HIGHLIGHTS`.

> [!decision] standalone_notes = exclude `kind == "note"` entries stay out of the feed
> exactly as today; only highlights and image highlights are members. Everything else in
> this plan is unchanged.

Note card contents: the sticky-note glyph and comment text (when present), then each
attached image on its own line (8 pt apart) fitted to the card's inner width, then one
muted "unavailable" row per unresolved image. Attached images live inside the yellow
card because they are Bryan's own notes; the yellow/orange language keeps meaning
"yours".

Image sizing (pure, in RefsCore):

- Intrinsic point size from pixels and DPI: when the image's DPI is ≥ 90 (1.25 × 72,
  bob's own retina threshold in `image_notes.rs`), points = pixels × 72 / DPI; otherwise
  points = pixels. Width and height are swapped for EXIF orientations 5–8.
- Display size: scale down uniformly to fit `maxWidth` (the slot's width) and
  `maxHeight` (wide 420 pt, compact 260 pt); **never upscale**; round both sides down to
  whole points (so the result never exceeds either cap), minimum 1.
- The image slot of an image highlight is `contentWidth − inspectorQuoteInset` wide: the
  yellow rule sits inside the inset, exactly as it does for quotes.
- Example: the 1417 × 686 px, 72 DPI asset above displays at 709 × 343 pt in the wide
  layout (slot 719.5 − 10 = 709.5); a 1170 × 2532 phone screenshot is capped at 420 pt
  tall.

Stable layout, the core reliability rule: the inspector loader reads each image's header
(pixel size, DPI, orientation, alpha) **before** it publishes the row's content, so
every image reserves its exact frame from the first layout pass. Pixel data then loads
lazily, only for entries that are mounted (the visible pages), downsampled with ImageIO
to `ceil(max(displayWidth, displayHeight) × 2)` pixels. A placeholder (rounded rect,
`.secondary.opacity(0.08)`, centered `photo` symbol) holds the exact size until the
bitmap arrives, then cross-fades in over 0.15 s (instant under Reduce Motion and in
`previewMode`). Images never change height after first layout, so Ctrl+D's follow-tail
measurement stays correct.

Image chrome: corner radius 6, 0.5 pt `.primary.opacity(0.08)` stroke; images with an
alpha channel (and every SVG) draw on a white backing so dark diagrams stay legible in
dark mode; hover tooltip shows the file name; VoiceOver reads them through the entry's
label. Images are not interactive (no Quick Look yet; see "Out of scope").

Unavailable images (asset or attached path that does not resolve inside the vault, a
missing or unreadable file, or an `unresolved_images` source): a compact caption row in
tertiary — `photo.badge.exclamationmark` (fall back to `exclamationmark.triangle` if the
symbol is missing), "Image unavailable", then the file's base name in monospaced
caption, middle-truncated. Absence stays honest and takes one line, not a box.

Disclosure row: `+N more` keeps its notes clause and adds an images clause when hidden
entries hold resolvable images: `+12 more · 3 with notes · 2 images · ⌃D shows 5 more`,
`+12 more · 2 images · ⌃D shows 5 more`, `+1 more · 1 image · ⌃D shows it`. Image-free
feeds produce exactly today's strings.

Accessibility labels (entry level, combined element):

- Highlight labels stay byte-identical when there are no images.
- Image highlight: `Image highlight, Page 5` (no page: `Image highlight`), then
  `. Your note: <comment>` when commented.
- Standalone note: `Note, Page 3: <text>` (no page: `Note: <text>`).
- Any entry with attached images appends `, 1 attached image` / `, N attached images`;
  with unresolved or unavailable images appends `, 1 image unavailable` /
  `, N images unavailable`.

## Phase `refs-core-model`: portable RefsCore model, geometry, and page size

Everything here is pure Swift in `Sources/RefsCore` with tests in `Tests/RefsCoreTests`,
so `just portable-check` on athena covers it. The only BobMacCapture edits are test
expectations that depend on the page size default.

1. **Decode image fields** (`RefsInspector.swift`): add `asset: String?`,
   `images: [String]`, and `unresolvedImages: [String]` (`unresolved_images`) to
   `RefsShowAnnotation`, with init parameters defaulted at the end so every existing
   call site compiles unchanged. Decode them lossily (`try?` around `decodeIfPresent`,
   falling back to nil or `[]`) so a malformed value degrades to "no images" instead of
   failing the annotation and turning the whole row into "Notes unavailable". Trim and
   drop empty strings.
2. **Feed membership**: add `RefsShowAnnotation.isInspectorEntry` and
   `RefsShowRow.inspectorEntries` per "4. Images" and the `standalone_notes` decision.
   Keep `inspectorHighlights`, `isInspectorHighlight`, `commentedHighlights`, and the
   Copy-card members unchanged (the loader keeps using `inspectorHighlights` until phase
   `inspector-images` switches it).
3. **Generalize `RefsInspectorFeed`** without breaking current outputs:
   - `Entry.kind` (`.highlight`, `.image`, `.note`), `Entry.imagePath` (the asset for
     `.image`), `Entry.attachedImages`, `Entry.unresolvedImages`, and
     `Entry.hasNoteCard` (comment, attached, or unresolved images for highlight/image
     kinds; always true for `.note`). Keep `hasNote`, defined as `hasNoteCard`.
   - `notedCount` counts entries with a note card (identical to today for highlight-only
     feeds); keep `headerAccessibilityLabel` and `hiddenAccessibilityLabel` strings
     identical for image-free feeds.
   - Kind-aware `accessibilityLabel` per the accessibility rules above.
   - `hiddenImageCount(visible:)` (hidden `imagePath` + attached images, excluding
     unresolved) and the images clause in `moreText` / `hiddenAccessibilityLabel`
     (`, 2 images`) only when that count is > 0.
   - Rename nothing public that BobMacCapture uses; the feed's stored array may stay
     named `highlights`.
   - Until phase `inspector-images` lands, the only visible effect is that a highlight
     carrying an attached image counts in the note chip (one such highlight exists
     today). That brief inconsistency is accepted.
4. **Images math** (new `Sources/RefsCore/RefsInspectorImages.swift`):
   - `RefsImageMetrics` (Equatable, Sendable): `pixelWidth`, `pixelHeight`,
     `dpi: Double?`, `orientation: Int` (EXIF 1–8, default 1), `hasAlpha`, plus
     `pointSize: CGSize` per the DPI and orientation rule.
   - `RefsImageFit.displaySize(pointSize:maxWidth:maxHeight:) -> CGSize` per the sizing
     rule (never upscale, round down to whole points, minimum 1, zero or negative inputs
     return `.zero`).
   - `RefsVaultAssetPath.resolve(_ relative: String, root: URL) -> URL?`: trims; rejects
     empty, absolute (`/…`, `~…`), any `..` component, and anything whose standardized
     path is not inside the standardized root; strips a leading `./`. No symlink
     resolution (vaults may contain symlinks legitimately).
5. **Panel geometry** (new `Sources/RefsCore/RefsPanelGeometry.swift`, using
   Foundation's CGFloat/CGRect so it builds on Linux): constants (max 1440 × 1000,
   margins 48 / 36, outer padding 8, list fraction 0.52, max list width 560, inspector
   threshold 760, divider 0.5, wide column threshold 568, compact padding 18, wide
   padding 24, readable width 720) and functions `panelSize(visible:)`,
   `panelFrame(visible:)`, `showsInspector(panelWidth:)`, `listWidth(panelWidth:)`,
   `inspectorColumnWidth(panelWidth:)`, and
   `inspectorLayout(panelWidth:) -> RefsInspectorLayout` (`mode`, `columnWidth`,
   `padding`, `contentWidth`). Image max heights (420 / 260) can live on
   `RefsInspectorLayout` too.
6. **Page size**: `RefsNotesPage.defaultSize = 5`. Update every test that assumed 3:
   `Tests/RefsCoreTests/RefsInspectorTests.swift`, and the BobMacCapture tests that
   depend on the default (`RefsInspectorScrollTests` eight-note reveal counts,
   `RefsPanelModelTests` around the notes-page tests, `RefsPanelDesignTests` revealed
   fixtures). Update the README's two "3 by default" / "default 3" mentions to 5.
7. **Tests** (`Tests/RefsCoreTests`): decoding (present, absent, null, malformed
   `images` → `[]` while the annotation and row survive; the two real JSON rows above);
   membership for every kind and both decision branches as implemented; entry kinds,
   `hasNoteCard`, `notedCount`; accessibility labels (existing highlight strings
   unchanged, new kinds, image clauses); `moreText` with and without hidden images;
   `RefsImageMetrics.pointSize` (72 DPI, 144 DPI, nil DPI, orientation 6 swap);
   `RefsImageFit` (no upscale, width cap, height cap, rounding, zero input);
   `RefsVaultAssetPath` (nested ok, `./` ok, `..` rejected, absolute rejected, empty
   rejected, prefix-sibling `vault2/` rejected); geometry (1440 × 870 → (48, 36, 1344,
   798), list 560, column 767.5, wide, content 719.5; 2560 × 1415 → 1440 × 1000 centered
   with content capped at 720; 1280 × 770 → wide; 880-pt panel → list 458, compact,
   content 369.5; 1024 × 640 → 928 × 568 compact; a tiny visible frame never yields
   negative sizes).

Verification: `just portable-check` green; mac scratch compile with
`just typecheck-tests` for the BobMacCapture test edits; CI green.

## Phase `big-panel`: near-full-screen panel, wide inspector, larger thumbnail

1. **Geometry wiring**: `RefsVisualTokens` window statics delegate to
   `RefsPanelGeometry` (`panelSize`, `panelFrame`, `showsInspector`, `listWidth`);
   remove `topFraction`, `screenMargin`, and the fixed 880 × 560 statics or keep them
   only as the compact reference used by tests, documented as such.
   `RefsPanelController.makePanel()` seeds its content rect from the geometry for the
   current visible frame (prewarm still happens at launch; `orderFront` still recomputes
   every show). `RefsPanelView.columns(panelWidth:)` computes
   `RefsPanelGeometry.inspectorLayout(panelWidth:)` once and passes it down.
2. **Wide inspector** (`RefsInspectorView.swift`): add `var layout: RefsInspectorLayout`
   (default: the compact layout, so existing call sites and fixtures keep today's look).
   In wide mode apply the padding, centered `contentWidth` frame, hero arrangement
   (thumbnail left; chips, title, byline, facts right), typography, spacing, and 6-line
   quote limit from "2. Two inspector layouts". Compact mode must render exactly as
   today. Put every new number in `RefsVisualTokens` (wide thumbnail 150 × 194, wide
   spacing 18, wide entry spacing 12, wide quote line limit 6, wide title line limit 3,
   quote line spacing 3). The Copy, Rename, and Today Link cards in the inspector column
   get the same centered max-720 constraint.
3. **Thumbnail at the new size** (`RefsInspectorLoader.swift`):
   `RefsPDFIntrinsicsLoader.thumbnailSize` becomes 300 × 388 px (150 × 194 @2x). Version
   the disk cache so stale 224 × 290 thumbnails are never upscaled: write
   `intrinsics-v2.json` and `thumbs-v2/`, and on the first cache-directory setup delete
   the legacy `intrinsics.json` and `thumbs/` (best effort, inside the same cache
   directory only). Keep modes 0600/0700. Compact layout downsamples the same image to
   112 × 145. Update the doc comments that cite 224 × 290 and 112 × 145.
4. **Ctrl+D reveals when the row is visible**: add a `RefsInspectorDisclosureAnchor` (a
   non-hit-testing `NSViewRepresentable` background, like `RefsInspectorRevealAnchor`)
   on the disclosure row. In `RefsPanelController.scrollInspector(.down)`, reveal when a
   reveal is pending, the geometry is at the bottom, **or** the anchor's rect (converted
   to document coordinates, flipped-aware like `followReveal`) lies fully inside the
   visible document rect. Put the containment test in a pure helper on
   `RefsInspectorScrollGeometry` (for example `containsRow(top:bottom:tolerance:)`).
5. **Tests** (`Tests/BobMacCaptureTests`):
   - `RefsInspectorScrollTests`: unit tests for the containment helper; a hosted test
     where the disclosure row is visible but the view is not at the bottom and one
     Ctrl+D reveals 5 more; existing hosted tests keep passing (move them to the
     production-like size where they rely on panel height, for example 1344 × 798, and
     keep one 880-pt case).
   - Intrinsics loader: the reader is asked for, and caches, a 300 × 388 thumbnail; the
     v2 file names are used; a pre-seeded legacy `intrinsics.json` and `thumbs/` are
     removed.
   - `RefsPanelDesignTests`: add 1344-pt renders of browse, inspector paper, inspector
     chat, highlights unified, highlights revealed (5 → 10), noted, and the wide Copy
     card; keep the existing 880-pt and 700-pt renders as the compact and list-only
     references. Use a quote long enough to clip at 6 body lines in the wide fixture.
     Use a 300 × 388 stand-in thumbnail.
6. **README** (`README.md`): "The panel" (adaptive near-full-screen size and margins,
   centered, list column cap, wide and compact inspector layouts), "Inspector" (150 ×
   194 thumbnail beside the facts in the wide layout, body typography, 6-line quotes, 5
   per page), the keyboard table's Ctrl-D row and the Ctrl-D paragraph (reveals when the
   `+N more` row is visible), and "Privacy" (`intrinsics-v2.json`, `thumbs-v2/`).

Verification: `just portable-check`; mac scratch compile (`Scripts/bundle.sh`,
`just typecheck-tests`); CI green; review the new 1344-pt fixtures in both appearances
from the CI artifact (hero alignment, thumbnail shadow, readable column, no clipping).

## Phase `inspector-images`: inline images and standalone notes

1. **Image loading** (new `Sources/BobMacCapture/Refs/RefsAnnotationImages.swift`):
   - `protocol RefsAnnotationImageProviding: Sendable` with
     `metrics(for url: URL) async -> RefsImageMetrics?` and
     `image(for url: URL, maxPixel: Int) async -> NSImage?`, so tests inject stubs.
   - `actor RefsAnnotationImageLoader`: raster metrics from
     `CGImageSourceCopyPropertiesAtIndex` (pixel size, DPI, orientation, alpha; no
     decode); bitmaps from `CGImageSourceCreateThumbnailAtIndex` with
     `kCGImageSourceCreateThumbnailFromImageAlways`, `…WithTransform` (EXIF
     orientation), and `kCGImageSourceThumbnailMaxPixelSize = maxPixel`; GIF uses
     frame 0. SVG goes through `NSImage(contentsOf:)` on the main actor (size from the
     image, `hasAlpha = true`). Metrics cache keyed by (path, file size, mtime); bitmaps
     in an `NSCache` with a byte-cost limit of about 128 MB keyed by (path, mtime,
     maxPixel). Decoding runs off the main actor. Failures return nil and are not
     cached.
   - `@MainActor final class RefsInspectorImageStore`: `cachedImage(path:maxPixel:)`
     (synchronous), `loadImage(path:maxPixel:) async`, and
     `installImagesForPreviews([String: NSImage])` (preview images are returned for any
     `maxPixel`). It resolves vault-relative paths with `RefsVaultAssetPath` against
     `library.vaultRootURL`.
2. **Metadata prefetch** (`RefsInspectorLoader.swift`): switch `load` to
   `row.inspectorEntries`; collect every `asset` and attached image path; resolve each
   with `RefsVaultAssetPath`; fetch metrics concurrently (task group) and store them in
   a new `RefsInspectorContent.images: [String: RefsImageMetrics]` before
   `rememberContent` publishes, so content and image sizes arrive together. Paths that
   fail to resolve or read are simply absent (the view shows the unavailable row). The
   loader owns one `RefsAnnotationImageLoader` (injectable in `init`) and the
   `RefsInspectorImageStore`. Extend `installContentForPreviews` (and
   `RefsPanelModel.installForPreviews`) with an `images: [String: NSImage]` parameter
   whose metrics are derived from the given images.
3. **Views** (new `Sources/BobMacCapture/Refs/RefsInspectorImageViews.swift`, plus
   `RefsInspectorView.swift`):
   - `RefsInspectorImageView(path:metrics:displaySize:store:previewMode:)`: fixed
     `displaySize` frame; placeholder → bitmap cross-fade; white backing when
     `hasAlpha`; radius 6; hairline; `.help(fileName)`; `.task(id:)` loads with
     `maxPixel = ceil(max(w, h) × 2)` and is cancelled when the entry unmounts.
   - `RefsInspectorImageUnavailableRow(fileName:)` per "Unavailable images".
   - `RefsInspectorView`: render `.image` entries (image in the quote slot with the
     yellow rule; slot width `contentWidth − inspectorQuoteInset`), attached images and
     unresolved rows inside the note card (inner width = card width minus its horizontal
     padding), and `.note` entries as a note card with no quote. Fit every image with
     `RefsImageFit.displaySize` and the layout's max height. Header chip counts
     `notedCount`; the disclosure row uses the new `moreText`. Pass the image store from
     `RefsPanelView` (via the model) into the inspector.
4. **Tests**:
   - Loader (`RefsInspectorTests` in BobMacCaptureTests): stub provider; content
     publishes with `images` metrics for resolvable paths, omits missing ones, and never
     publishes content before metrics are ready; `..` paths are never handed to the
     provider; `inspectorEntries` membership reaches the view (image highlight, attached
     image, standalone note per the decision).
   - Hosted reveal (`RefsInspectorScrollTests`): a feed whose second page holds image
     entries with a stub provider that delays bitmaps; after Ctrl+D the follow-tail
     target is applied, and the document height measured before the bitmaps arrive
     equals the height after they arrive (no layout jump).
   - Image store: cached lookup after load; preview-installed images returned for any
     `maxPixel`.
   - `RefsPanelDesignTests` at 1344 and 880, both appearances: image highlight; a
     highlight with comment + attached image; attached image with no comment; a
     standalone note (when the decision includes them); an unavailable image row; a
     transparent PNG on the white backing in dark mode; a revealed page containing an
     image. Generate the fixture images in code (solid fills and simple shapes) and
     install them through `installForPreviews(..., images:)`.
5. **README**: "Inspector" (image highlights in the quote slot, attached images inside
   the note card, sizing and the never-upscale rule, unavailable rows, standalone notes
   per the decision, the images clause in the disclosure row) and "Privacy" (the app
   reads image assets that `bob ref show` names from the vault, read-only, and keeps
   decoded previews in memory only; nothing new is written to disk).

Verification: `just portable-check`; mac scratch compile; CI green; review the image
fixtures in both appearances from the CI artifact. If the Mac is reachable, also say in
the report which real notes exercise the feature (`ref/docs/gastown_readme.md` has an
image highlight on Page 5; `ref/blogs/gas_town_emergency_user_manual.md` has an attached
image on Page 9) so Bryan can check them after his next `just install`.

## Out of scope (possible follow-ups)

- Quick Look or zoom for images, animated GIF playback, and images in the Copy card or H
  picker.
- A user setting for the panel size; any bob-cli change (the JSON contract already
  carries everything).
- Changing the Copy card's notes semantics (`commentedHighlights` stays comment-based).
