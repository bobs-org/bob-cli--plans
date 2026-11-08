---
tier: epic
title: ⌘↵ opens a highlight's page in Highlights
goal: 'In Obsidian, ⌘↵ on a reference-note highlight, a 🔖 bookmark line, or a Task
  Link to an annotation task opens the exact PDF in Highlights and lands on that highlight''s
  page. A status-bar chip and a highlighter rail show beforehand what ⌘↵ will do,
  on reading tasks too. One Bob toast announces each press before Highlights takes
  focus.

  '
decisions:
  page_jump_decision_record:
    ask: Add a decisions-web record "Highlights page jumps go through Hammerspoon
      by exact file identity"?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: hs_page_url
  title: Hammerspoon page-jump URL handler
  depends_on: []
  size: medium
  description: 'hs_page_url: in chezmoi, bind hammerspoon://bob-highlights-page. It
    waits for the exact PDF to settle in front, jumps through the existing page runner
    with ⌃O history, re-checks the landing, and reports the result inside Highlights.
    Includes busted specs and README.'
- id: locator
  title: Pure annotation locator
  depends_on: []
  size: medium
  description: 'locator: in bob-plugins, a pure fragment that maps a ref-note line
    to its managed ^h- block. It returns the block kind, the 1-based page (caption
    or piece), a snippet, and the rail range. It also parses 🔖 links and the archive
    PDF wikilink. Tested against golden fixtures copied from bob-cli.'
- id: open_at_page
  title: Resolver, exact-path launch, and page request
  depends_on:
  - locator
  - hs_page_url
  size: medium
  description: 'open_at_page: extend the ⌘↵ command so that one resolver handles reading
    tasks, 🔖 lines, Task Links, and managed blocks. It follows hops, stubs, and the
    unique-ID fallback, and opens ref_history archives through their annotated_pdf.
    It opens the exact path, then sends the Hammerspoon page request. Adds a dry mode,
    an announce seam with plain notices, a re-entry lock, and the additive api page
    option.'
- id: toast
  title: The Highlights toast card
  depends_on:
  - open_at_page
  size: small
  description: 'toast: replace the plain announce notices with one rich .bob-nh-notice
    Highlights card in highlighter ink. It shows a page receipt, the title, and a
    quote snippet, appears before dispatch, and updates in place on failure.'
- id: hints
  title: Availability chip, rail, and 📖 tooltip
  depends_on:
  - toast
  size: medium
  description: 'hints: a status-bar chip and a line rail, both driven by the dry resolver,
    show what ⌘↵ will open. Adds a session toggle and the ledger-tools 📖 tooltip mention.
    Tests that the hint always matches what the key does.'
- id: contract_docs
  title: bob-cli client contract docs and decision record
  depends_on:
  - open_at_page
  - hs_page_url
  size: small
  description: 'contract_docs: declare the generated captions, piece metadata, and
    block shapes as a client contract in bob-cli docs. Extend the ⌘↵ sentence in docs/ref.md
    and record the Hammerspoon page-jump decision.'
proposed_by: bbugyi200.apollo.research.3o.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 10:40:39
status: done
bead_id: bob-cli-iq
---

- **PROMPT:** [prompts/202611/highlights_annotation_page_jump.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/highlights_annotation_page_jump.md)
- **BEAD:** [bob-cli-iq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-iq/README.md)

# Plan: ⌘↵ opens a highlight's page in Highlights

## Context

`Cmd+Enter` (vault binding `Meta+Enter` → command
`bob-navigation-hotkeys:open-reference-pdf-in-highlights`) already opens a reading
task's `source_pdf` in Highlights
(`plugins/bob-navigation-hotkeys/src/677-plugin-ref-pdf-open.js` in bob-plugins). This
epic extends it to the highlights themselves.

The design follows the consolidated research report, which Bryan accepted in full:
`research:202611/highlights_annotation_page_jump/highlights_annotation_page_jump.md`.
Read it with `sase artifact read file:explicit:9b87920af61a6b9d0190fe7d "<why>"` for the
evidence behind each choice (bookmark census, page-label counts, Highlights binary
strings). This plan is the source of truth where the two differ. Every difference is
called out below.

Repositories: this epic lives in the bob-cli project. Phases that edit bob-plugins or
chezmoi must open them with `/sase_repo` (`sase repo open bob-plugins -r "…"`,
`sase repo open chezmoi -r "…"`) and read that repo's `AGENTS.md` first. bob-cli source
(`src/native/highlights_ref/`, `docs/highlights-ref-sync.md`) is the reference for the
generated note grammar and stays unchanged except for docs.

## The rule

**⌘↵ opens the source of whatever the cursor is on.** One rule covers the old cases and
the new ones:

| Cursor line (Source or Live Preview)                                              | ⌘↵ does                                                         |
| --------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `#task #ref` reading task, or a Task Link / managed embed pointing at one         | Open the PDF; Highlights resumes where you left off (unchanged) |
| A line with a `[[…#^h-…\|🔖]]` bookmark (task or bullet, any note, incl. `done/`) | Open the PDF at that highlight's page                           |
| A Task Link pointing at a 🔖 task                                                 | Same as the 🔖 line                                             |
| Inside a managed `^h-` annotation block in a ref note (`ref/` or `ref_history/`)  | Open the PDF at that block's (or piece's) page                  |
| On a page caption heading inside the managed region (`### Page 12`)               | Open the PDF at that page                                       |
| A tombstone (`Removed highlight`)                                                 | Explain; launch nothing                                         |
| Anything else (section cards, outside the region, ordinary links)                 | Obsidian's open-link-in-new-tab (unchanged)                     |

The column never matters. A link inside a quote does not change the target, which keeps
the chip steady. Opening is read-only for the vault and for task, lane, freshness, and
review state.

## Design language

All three surfaces share one color: **highlighter ink**,
`color-mix(in srgb, var(--color-yellow) 65%, var(--color-orange))`. This is the same
gold the `ref-highlights` snippet uses to underline emphasis in quote callouts. The chip
icon, the rail, and the toast's accent bar all use it, so wherever you see that gold, ⌘↵
goes to Highlights.

```text
Rail + chip (status bar)
▍> [!quote] You technically only need the Mayor…          [▮ Highlights · p. 4  ⌘↵]
▍- [ ] #task Compare with the appendix [[…|🔖]]            [▮ Highlights · p. 4  ⌘↵]
▍- [ ] #task #ref [[lib/papers/x.pdf]] …                   [▮ Highlights  ⌘↵]

Toast (top right, before Highlights takes focus)
┃ ▮ HIGHLIGHTS  Opening                                p. 4
┃ Gas Town Emergency User Manual
┃ “You technically only need the Mayor. Running with just…”
```

## Shared contracts

### Hammerspoon URL (between `open_at_page` and `hs_page_url`)

```text
hammerspoon://bob-highlights-page?path=<encodeURIComponent(absolute PDF path)>&page=<N>
```

- `path` is absolute and ends in `.pdf`. `page` is a decimal integer from 1 to 999999,
  1-based and physical (the `Page N of M` number in Highlights' title).
- The plugin opens the PDF itself (`open -b net.highlightsapp.universal <path>`, as
  today) and only then sends this URL with `open -g`. **The handler never opens files or
  launches apps.** It acts only on a document that is already in front. A failed or
  missing handler therefore never stops the PDF from opening.
- Hammerspoon hands the callback decoded query parameters. The handler must not decode
  them again.

### Resolver target (between `open_at_page`, `toast`, and `hints`)

`open_at_page` adds `resolveHighlightsTargetAtCursor(editor, file, { dry })`. It returns
one frozen target, and the toast and the hints read only this shape:

```js
{
  kind: "reading" | "page" | "removed" | "missing-highlight" | "error" | "none",
  pdfPath,        // vault-relative, validated; null unless reading/page
  page,           // integer >= 1, or null (unknown page → opens where you left off)
  title,          // ref note frontmatter `title`, else note basename, else PDF basename
  snippet,        // quote/note text (<= ~72 chars) or null
  detail,         // "quote" | "note" | "image" | "caption" | null
  captionCount,   // highlights under a caption, when detail === "caption"
  archived,       // { replaced: "YYYY-MM-DD" | null } for ref_history archives, else null
  blockId,        // "h-…" or null
  via,            // "cursor" | "bookmark" | "task-link" | "embed"
  railLines,      // { from, to } 0-based inclusive lines in the ACTIVE editor, or null
  message,        // user-facing reason for removed / missing-highlight / error
}
```

The dry mode does exactly the same resolution with no notices and no `exec`. It may be
served from caches (see `hints`). The same function powers the key and the hints, so
they cannot disagree.

### Announce seam (between `open_at_page` and `toast`)

The plugin calls `this.announceHighlightsOpen(target)` once per press. The call comes
after resolution and pre-launch validation and **before** any `exec`. It returns a
handle `{ pageJumpUnavailable(detail), failed(reason) }`. `open_at_page` ships a
plain-`Notice` implementation; `toast` swaps in the rich card and keeps the plain text
as its `aria-label` and as the fallback when no DOM is available. Removed,
missing-highlight, and pre-launch failures (unsafe path, missing PDF, off macOS, no
desktop) go through `this.reportHighlightsOutcome(target)`, which shows a single notice
and launches nothing.

## Phase hs_page_url: Hammerspoon page-jump URL handler

Repo: chezmoi. Files: `home/dot_hammerspoon/highlights_pages.lua` (pure, `hs`-free
helpers), `home/dot_hammerspoon/highlights_tab_keys.lua` (runtime wiring),
`tests/hammerspoon/highlights_pages_spec.lua`,
`tests/hammerspoon/highlights_tab_keys_spec.lua`, `README.md` ("Highlights page keys").
For background, the page runner's own design is `plan:202611/highlights_page_keys.md`.

1. **Binding.** In `M.start()`, bind `H.urlevent.bind("bob-highlights-page", handler)`.
   `M.stop()` unbinds it (`bind(name, nil)`) and cancels any pending wait. Add an
   `options.urlevent` test override, in the same style as the existing `options.open`
   and `options.alert` overrides.
2. **Parse** (pure `HighlightsPages.parse_page_request(params)`). Returns
   `{ path, page }` or `nil, reason`. Rules: `path` is a string starting with `/`,
   ending in `.pdf` (case-insensitive), with no NUL. `page` matches `^%d+$` and is
   between 1 and 999999. Invalid requests are dropped and only a reason is logged:
   **never log paths or titles.**
3. **Wait** (pure step function plus a runtime poller). Poll every 0.1 s for up to 8 s,
   which is enough for a cold launch. The request is ready when all of these hold:
   - Highlights is frontmost.
   - The focused window's `AXDocument` is the requested file. Compare `{dev, ino}` from
     `hs.fs.attributes` for both paths, so NFC/NFD and symlinks match. Fall back to
     comparing the decoded path strings when attributes are unavailable. A same-named
     PDF in another directory must not match.
   - The title parses (`parse_title`) and the PDF view is found.
   - **Settle:** the title's page number is the same on two consecutive polls. This lets
     Highlights finish restoring the last-viewed page before the jump, so the restore
     cannot undo the jump.
   - Neither a tab-close nor a page run is in progress. When one is busy, keep waiting;
     do not fail.
4. **Cancel and order.** The latest request wins: a new request silently cancels the
   pending one. If Highlights came to the front during this wait and then another app
   took the front, cancel silently. Before Highlights first comes to the front, keep
   waiting until the deadline (cold launches start in the background).
5. **Jump.** Call the existing `dispatch_page({ kind = "goto", target = N })`. The
   runner pushes the restored page onto ⌃O history, so ⌃O returns to where you left off.
   Add an `origin = "url"` flag that only changes the feedback copy. About 0.4 s after
   landing, read the page once more; if it is the same document but a different page (a
   late restore), re-jump once.
6. **Feedback** (pure copy in `highlights_pages.lua`; key-driven jumps keep their quiet
   success):
   - Landed: `Page 4 of 44`. When the jump moved from a different page, add
     ` · ⌃O back to p. 30`; this teaches the back key.
   - Timeout: `Couldn't jump to page 4: the PDF didn't come to the front`.
   - Out of range and failures: reuse `M.message` (`No page 60 — this PDF has 44 pages`,
     …).
7. **Specs (busted).** Cover: parse accept and reject cases (relative path, non-PDF,
   page `0`, `4a`, `1000000`, missing params); ready only after settle; the restore race
   (page 30 → 30, then jump to 4); timeout copy; keep waiting before the first front;
   cancel after leaving the front; latest wins; an inode match versus a same-named file
   in another directory; busy keeps waiting; ⌃O returns to the restored page; the
   post-landing re-check re-jumps exactly once; success copy with and without the back
   hint; `stop()` unbinds and cancels; the handler never calls `open_fn`.
8. **README.** Under "Highlights page keys", add an "Open at a page from Obsidian"
   paragraph: the URL, the guarantee that the handler never opens files, the ⌃O
   behavior, and the alert copy.

Verify with `just test-hammerspoon` and `stylua --check` on the touched Lua files. Then
follow chezmoi's `AGENTS.md` (`chezmoi update -a --force` after the commit). The Mac
picks the change up on its next chezmoi apply, and the existing restart hook reloads
Hammerspoon.

## Phase locator: Pure annotation locator

Repo: bob-plugins. New fragment
`plugins/bob-navigation-hotkeys/src/677a-ref-annotation-locator.js`, listed in
`src/fragments.json` right after `677-plugin-ref-pdf-open.js`. Pure functions only: no
Obsidian API and no plugin state. Keep it at or below 1000 lines. Mirror bob-cli's
generator (`src/native/highlights_ref/region.rs`, `continuation.rs`, and the "Generated
Body Contract" in `docs/highlights-ref-sync.md`). Read them; do not change them.

Functions:

- `parseRefPageLabel(text)` returns `{ label, page }` or `null`. It accepts `Page N`,
  `Pages A–B` (en dash or hyphen), `Pages 11, 14`, `Pages 11–12, 14`, `p. N`, and
  `pp. A–B`. `page` is the first integer. A non-numeric label (`Page xii`) returns
  `page: null`. Anything else is not a caption.
- `locateRefAnnotation(lines, lineIndex)` returns `null` (ineligible) or a frozen
  `{ kind, blockId, page, pageLabel, snippet, annotationKind, stubTarget, rail, count }`.
- `findRefAnnotationBlockLine(lines, blockId)` returns the index of the `^<blockId>` ID
  line inside a managed region, or `-1`.
- `parseRefBookmarkLink(lineText)` returns the first `[[target#^h-…|🔖]]` link
  (`[[#^h-…|🔖]]` means `target: ""`, the same note) as `{ target, blockId }`, or
  `null`. Only `h-` IDs and the exact `🔖` alias qualify.
- `parseRefArchivePdfLink(value)` turns an `annotated_pdf` wikilink
  (`'[[ref_history/…__annotated.pdf]]'`, with an optional alias) into a linkpath, or
  returns `null`.

Rules:

1. **Region.** The line must lie strictly between `<!-- highlights:begin -->` and
   `<!-- highlights:end -->` and outside fenced code (reuse `OPENING_FENCE_RE` and
   `CLOSING_FENCE_RE`). A user-written `### Page 12` outside the region never counts.
2. **Captions.** Use the nearest ATX heading above the line within the region. If it
   parses as a page label, it is the governing caption. If it does not (`### Memory`,
   `### Other highlights`, `### Removed highlights`), the page is unknown. This is
   equivalent to the H3/H4 rules for both unsectioned and sectioned notes.
3. **Caption line.** A cursor on a caption heading returns `kind: "caption"`, that page,
   `rail` = that line, and `count` = the number of `^h-` blocks before the next heading.
4. **Block bounds.** A block runs from its first callout line through its `^h-…` ID
   line, including nested comments and the blank line before the ID. From the cursor,
   scan down to the ID line; stop with `null` if a heading or the end marker comes
   first. Scan up to just after the previous ID line or heading. **Gaps** (blank lines
   after an ID line and before the next callout) return `null`, so the rail never jumps
   ahead to the next block.
5. **Classify** by the block's first callout header:
   - `[!quote]` or `[!quote|…]` is an annotation (`annotationKind: "quote"`, or
     `"image"` for `[!quote] Image`).
   - `[!note]` is an annotation (`"note"`).
   - `[!info] Grouped highlight` and `[!info] Merged emphasis` are a `stub`.
     `stubTarget` is the first `[[…#^h-…]]` inside the block.
   - `[!warning] Removed highlight` is a `tombstone`.
   - `[!summary|section]` and `^s-` blocks are a `section` (ineligible).
6. **Page and pieces.** A continuation group has a head `> [!quote|Page 11] …` and
   followers at quote depth 2 (`> > [!quote|Page 12] …`). Count leading `>` markers to
   find the depth.
   - Cursor inside a piece (from its header to the line before the next depth-2
     `[!quote|` header or the end of the callout; nested comments belong to their
     piece): the page is that piece's page and `rail` covers that piece's lines.
   - **Cursor on the ID line or the blank line before it: the page is the head's (first)
     page and `rail` covers the whole block.** This corrects the research report. Its
     "nearest piece header above" rule would pick the _last_ piece whenever a 🔖 hop
     enters at the ID line. A bookmark opens the passage where it starts, which also
     matches the caption's first page.
   - A block without piece metadata uses the governing caption's page and the whole
     block as its rail. A block before the first caption has page `null`.
7. **Snippet.** Use the text of the piece (or head) quote. Strip the `>` markers, the
   `[!…]` header token, `**`, and embeds, and collapse whitespace. Truncate at a word
   boundary to about 72 characters with `…`. Note blocks use the note text. Image blocks
   use `null` (the toast says "Image highlight").

Tests: add `scripts/test-navigation-ref-annotation-locator.cjs` and list it in the
`package.json` `test` script. Golden fixtures go in `scripts/fixtures/ref-annotations/`,
copied from bob-cli's docs examples and render tests (grep `src/native/highlights_ref/`
for `Pages 11–12`): an unsectioned note, the `Pages 11–12` group, the three-piece
`Pages 6–8` group with images and comments, a sectioned note (`### Memory` /
`#### Page 4` / `### Other highlights`), `### Removed highlights` with a tombstone and
both stub kinds, an image block, a standalone note, a `Page xii` caption, a preamble
block, a fake region inside a fence, and a user `### Page 12` outside the region.

Cases: every line kind of a block (callout, nested comment, blank, ID), the caption,
`Pages A–B`, each continuation piece, the ID line of a group (head page, whole rail),
the H4 caption in a sectioned note, the section card (null), the gap line (null), the
fence (null), outside the region (null), the non-numeric label (page null), and the 🔖
and archive-link parsers.

Run `npm run build`, `npm test`, and `npm run validate`, then `bob plugins sync`. The
fragment adds no behavior yet; nothing calls it until `open_at_page`.

## Phase open_at_page: Resolver, exact-path launch, and page request

Repo: bob-plugins. Edit `677-plugin-ref-pdf-open.js` and add
`677b-ref-pdf-target-resolver.js` (after `677a` in `fragments.json`), keeping each
fragment at or below 1000 lines. Wire `api.openReferencePdf` in
`480-review-jump-and-nav-api.js` and rename the command in `490-plugin-lifecycle.js`.

1. **Resolution order** in `resolveHighlightsTargetAtCursor`:
   1. Reading task line → `kind: "reading"`. The existing behavior and error messages
      stay. Extend `resolveReferencePdfFromTaskLine` so it also returns the ref note's
      `title`.
   2. A line with a 🔖 bookmark (`parseRefBookmarkLink`) → bookmark hop
      (`via: "bookmark"`).
   3. A Task Link or managed embed line (existing `parseRefPdfBlockLinkLine`) → read the
      target line. If it is a reading task, use the reading rule (`via: "task-link"` or
      `"embed"`). If the line has a 🔖, do the bookmark hop. Otherwise return none.
   4. Cursor inside the managed region of the active note → `locateRefAnnotation`.
   5. Otherwise → none, which delegates to `editor:open-link-in-new-leaf` as today.
2. **Bookmark hop.** If `target` is empty, use the source note; otherwise resolve it
   with `resolveLinkTargetFile`. Read it with `readLinkPickerNoteContent` (this prefers
   the live buffer), call `findRefAnnotationBlockLine`, and then call
   `locateRefAnnotation` at the ID line.
   - **Unique-ID fallback:** when the file or the block is missing, look for
     `metadataCache.getFileCache(f)?.blocks?.[blockId]` across Markdown files under
     `ref/` and `ref_history/`. Exactly one match wins. Zero or several matches give
     `missing-highlight` with `Can't find highlight h-…`. The fallback runs only on this
     miss. Today 4 of 17 live 🔖 tasks need it: same-note links on tasks that moved out
     of their ref note before `bob task move` qualified links.
   - **Stubs:** follow `stubTarget` for at most 3 hops, with a cycle check on `path#id`.
     Beyond that the result is `missing-highlight` with `Can't follow highlight h-…`.
   - **Tombstone** → `removed`, with the message
     `This highlight was removed in Highlights — no page to open`. Nothing launches.
3. **PDF identity.**
   - A `ref/…` note with `source_pdf` uses that path (`normalizeRefSourcePdfPath`).
   - A `ref_history/…` note uses its `annotated_pdf` wikilink, resolved with
     `metadataCache.getFirstLinkpathDest(linkpath, notePath)` and falling back to the
     normalized literal. It goes through the same validation (no `..`, not absolute,
     `.pdf` only) and sets `archived: { replaced: fm.replaced ?? null }`. **Never fall
     back to the live PDF.**
   - Any other note holding the block is an error naming the note.
4. **Launch.**
   - `openReferencePdfPath(path, options)` gains `options.page`. First run the
     pre-launch checks: safe path, `adapter.exists`, desktop base path, macOS. Failures
     go to `reportHighlightsOutcome` using today's messages verbatim.
   - Next, `announceHighlightsOpen(target)`, then the existing two `open` attempts.
   - After a successful open with a known page, run
     `execFile("/usr/bin/open", ["-g", url])`. Use the argument array, never a shell,
     with the existing timeout. A non-zero exit calls
     `handle.pageJumpUnavailable(detail)` and the function still returns `true`. Reading
     tasks send no page request.
   - A launch failure calls `handle.failed(reason)`.
5. **Re-entry lock.** Ignore a press while a launch is in flight and for 400 ms after
   one, so a double press never sends two page requests.
6. **API.** `api.openReferencePdf(path, options?)` passes `{ page }` through when it is
   a positive integer. The API version stays the same; the Reading Board needs no
   change.
7. **Command.** Rename it to "Open in Highlights (reading task, highlight page, or link
   in new tab)", keeping the ID. It gets no default hotkey; the vault's `Meta+Enter`
   binding stays.
8. **Plain announce implementation** (until `toast` replaces it):
   - Announce: `Opening <title> in Highlights at p. 4`, or
     `Opening <title> in Highlights — resumes where you left off` for reading tasks, or
     `… — no page number on this highlight` when the page is unknown.
   - Page jump unavailable: `Opened <title> — jumping to p. 4 needs Hammerspoon`.
   - Failure: the reason verbatim.
9. **Dry mode.** Pass `{ dry: true }` for the same resolution with no notices and no
   `exec`. Fill `railLines` for the active editor: the single line for a reading task,
   🔖 line, Task Link, or embed, and the locator's `rail` for an in-note block or
   caption. A `missing`-PDF result is computed synchronously with
   `vault.getAbstractFileByPath`.

Tests: extend `scripts/test-navigation-hotkeys-ref-pdf.cjs`, or add a sibling
`scripts/test-navigation-hotkeys-ref-pdf-page.cjs` listed in `package.json`.

- Hops: same-note and routed 🔖, a 🔖 bullet in `done/`, a dangling same-note 🔖 with a
  unique ID elsewhere, zero and duplicate IDs, a Task Link to an annotation task, a Task
  Link to a reading task (unchanged), stub → group, a stub cycle, a tombstone, and a
  `ref_history` archive that uses `annotated_pdf` and never `source_pdf`.
- Launch: the exact-path argument array; paths with spaces, Unicode, `&`, `#`, and `%`
  encoded by `encodeURIComponent`; a page request only when the page is known; a failed
  page request reports `pageJumpUnavailable`; the re-entry lock; off macOS; the archive
  never opening the live PDF; no vault or editor writes (the existing throwing stubs
  cover the new paths).
- Dry/live parity: for every fixture case, the dry target equals the live target.
- Update the existing assertions on `Opened x.pdf in Highlights`.

Update `docs/highlights.md` (the plugin reference) with the new targets, the page
behavior, the archive branch, and the Hammerspoon requirement. Run `npm run build`,
`npm test`, `npm run validate`, and `npm run docs:check`, then `bob plugins sync`.

## Phase toast: The Highlights toast card

Repo: bob-plugins. New fragment `677c-highlights-notice.js` (after `677b`). Add CSS next
to the `.bob-nh-notice` block in `plugins/bob-navigation-hotkeys/styles.css`. Reuse the
structure and helpers of the lane-walk card (`543-lane-walk-notice.js`): `createDiv` /
`createSpan`, `applyIcon`, the hide-and-replace singleton, and the plain-text fallback.

Card: `.bob-nh-notice.bob-nh-highlights`, with `--bob-nh-level-color` set to highlighter
ink.

- **Header:** a Lucide `highlighter` icon, the level pill `Highlights`, a muted status
  word (`Opening`), and a right-aligned receipt `p. 4` (`.bob-nh-notice-receipt`,
  tabular numerals). There is no receipt when the page is unknown or for reading tasks.
- **Title:** the reference title, semibold, on one line with an ellipsis.
- **Line 3** depends on the situation:

| Situation                    | Line 3 or change                                                         | Duration |
| ---------------------------- | ------------------------------------------------------------------------ | -------- |
| Quote or note with a page    | The snippet in curly quotes, italic `--text-muted`, clamped to two lines | 4 s      |
| Image block                  | `Image highlight`                                                        | 4 s      |
| Caption                      | `3 highlights on this page` (singular when 1)                            | 4 s      |
| Reading task                 | `Resumes where you left off`                                             | 4 s      |
| Unknown page                 | `No page number on this highlight — opens where you left off`            | 4 s      |
| Archived copy                | Adds the chip `Archived copy · replaced 2026-10-28`                      | 4 s      |
| Page jump unavailable        | Amber `is-warn`; status `Opened`; `Jumping to p. 4 needs Hammerspoon`    | 8 s      |
| Launch or pre-launch failure | Red `is-error`; status `Couldn't open`; today's reason verbatim          | 8 s      |
| Removed highlight            | Muted accent; status `Removed`; the removed message; nothing launches    | 4 s      |
| Highlight not found          | Red `is-error`; `Can't find highlight h-…`                               | 8 s      |

Behavior:

- Show one Highlights card at a time: each new card hides the previous one
  (`plugin.highlightsNotice`).
- Updates replace the card in place: use `notice.setMessage(fragment)` when the duration
  does not change, otherwise hide it and show a new 8 s card in the same slot.
- The copy always says "Opening", never "Opened at page N". Obsidian cannot see where
  Highlights lands; Hammerspoon's alert confirms that.
- Give the card a plain-text `aria-label`. With no DOM, fall back to the `open_at_page`
  text.
- The card has no animation, so reduced motion needs nothing extra.
- Check the CSS in light and dark themes and in a narrow window. It inherits
  `.notice:has(.bob-nh-notice)` sizing.

Tests: add `scripts/test-navigation-highlights-notice.cjs`, using the fake-DOM approach
of the existing lane-walk notice tests. Cover every row of the table, the aria text, the
replacement singleton, the in-place update, durations, and the no-DOM fallback. Extend
`docs/highlights.md` with the card. Run the same build, test, validate, and docs checks,
then `bob plugins sync`.

## Phase hints: Availability chip, rail, and 📖 tooltip

Repo: bob-plugins. New fragment `677d-highlights-hints.js` (after `677c`). Wire it into
`490-plugin-lifecycle.js` with setup and teardown, as `setupBracketModeIndicator` does.
Add CSS to `styles.css`. Change one constant in bob-ledger-tools.

**Chip** (one `addStatusBarItem`, `.bob-highlights-chip`):

- Contents: the `highlighter` icon in highlighter ink, `Highlights` in `--text-muted`,
  ` · p. 4` in tabular numerals when the page is known, and a keycap.
- The keycap has a 1px `--background-modifier-border` outline, a 4px radius, and
  `--font-ui-smaller`. Its text comes from the vault's real binding: guarded
  `app.hotkeyManager` custom keys, then defaults. Format `Meta`/`Mod`→⌘, `Ctrl`→⌃,
  `Alt`→⌥, `Shift`→⇧, `Enter`→↵. An unbound command shows no keycap.
- States:
  - reading task: `Highlights ⌘↵`
  - page known: `Highlights · p. 4 ⌘↵`
  - page unknown: `Highlights ⌘↵`
  - removed: muted `Highlight removed`, no keycap
  - missing highlight: muted `Highlight not found`
  - missing PDF: muted `PDF missing`
  - none, off macOS, Reading view, hints toggled off, or no Markdown editor: the chip is
    **hidden**, never shown empty.
- Tooltip and `aria-label`: `Open “<title>” at page 4 in Highlights (⌘↵)`. A reading
  task reads `Open “<title>” in Highlights — resumes where you left off (⌘↵)`.
- Clicking opens the **cached target** through the launch path
  (`openHighlightsTarget(target)`). It never re-resolves from focus, because clicking
  the status bar blurs the editor.
- Timing: recompute after 120 ms of cursor stillness, debouncing selection and document
  updates (extend the existing `EditorView.updateListener`), `active-leaf-change`, and
  `file-open`. Fade in over 120 ms of opacity, with no transition under
  `prefers-reduced-motion`. Repaint only when the target key changes (kind, pdfPath,
  page, title, rail).

**Rail:**

- Add a `StateField` plus `StateEffect` in the pattern of
  `bobOpenNotesLandingFlashField`, registered with `registerEditorExtension`. It applies
  `Decoration.line({ class: "bob-highlights-rail" })` to `railLines` in the active
  editor only.
- Paint it as
  `background-image: linear-gradient(to right, <ink at 55%> 0 2px, transparent 2px)`
  with `no-repeat`. **Do not use `box-shadow`.** The landing flash animates `box-shadow`
  with fill-mode `both` and would erase the rail, and themes paint active-line
  backgrounds; a background image composes with both.
- Ranges: the whole annotation block, only the current piece in a continuation group,
  the caption line, or the single reading-task, 🔖, Task Link, or embed line. In Live
  Preview, rendered callout widgets have no line decorations, so the rail shows on
  revealed source lines. That is expected.
- Clear the rail when the target goes away, the editor changes, or the hints toggle off.
  Remove all DOM and decorations on unload.

**Toggle:** add the command `toggle-highlights-hints` ("Toggle Highlights hints"). It
hides the chip and the rail for the session and is on by default. It has no default
hotkey.

**Cost:**

- Resolve the same note from `editor.getValue()`, caching the locator result by document
  version.
- Cross-note hops (🔖 and Task Links) resolve asynchronously through a cache keyed by
  `(target path, mtime, blockId)`. Show nothing until they resolve, and drop stale
  results with a request token.
- Run no subprocess, PDF parsing, or vault-wide scan on cursor moves. The unique-ID
  fallback runs only on a cache miss.

**📖 tooltip:** in `plugins/bob-ledger-tools/src/138-task-tag-marks.js`, set
`REF_TASK_MARK_TOOLTIP = "#task #ref · reference reading task · ⌘↵ Highlights"` and
update the ledger-tools tests that assert it.

**Tests:** add `scripts/test-navigation-highlights-hints.cjs`, covering:

- the chip view model for every state;
- keycap formatting (Meta+Enter → ⌘↵, a remapped key, unbound → none);
- hidden off macOS, in Reading view, and when toggled off;
- a click using the cached target;
- rail ranges per context;
- debounce and repaint-only-on-change;
- unload cleanup;
- **hint ⇔ key equivalence:** for every resolver fixture case, the chip state implies
  the key's action (opens with page N / opens without a page / explains / delegates).

Extend the harness stubs only as needed; it already fakes `Decoration` and `StateField`.
Document the chip, the rail, and the toggle in `docs/highlights.md`, and in
`docs/bob-navigation-hotkeys.md` if it lists commands. Run the build, test, validate,
and docs checks, then `bob plugins sync`.

## Phase contract_docs: bob-cli client contract docs and decision record

Repo: bob-cli (this project). Read the `docs.md` reference memory first. Docs only: no
Cargo.toml version or CHANGELOG edits (the release train owns those).

1. In `docs/highlights-ref-sync.md`, under "Generated Body Contract", add a short
   **Client contract** paragraph. bob-navigation-hotkeys reads the following to resolve
   ⌘↵ page jumps: the region markers, the H3/H4 page caption grammar (`Page N`,
   `Pages A–B`, `Pages 11, 14`, `p.`/`pp.`), `[!quote|Page N]` piece metadata,
   standalone `^h-` ID lines, the `Grouped highlight` / `Merged emphasis` /
   `Removed highlight` callout headers, and `annotated_pdf` on `ref_history` archives.
   Changing any of them requires a coordinated plugin update. Note that pages are
   physical and 1-based (no annotated PDF has `/PageLabels` today).
2. In `docs/ref.md`, extend the ⌘↵ sentence (currently "In Obsidian, `Cmd+Enter` on a
   reading task … without writing the vault."): on a highlight in a reference note, a 🔖
   bookmark line, or a Task Link to an annotation task, it opens the PDF at that
   highlight's page, and the page jump runs through Hammerspoon on the Mac.
3. Run `just docs-check`.

> [!decision] page_jump_decision_record Use `/sase_memory_write`, then add the strand
> `sase/memory/decisions/highlights-page-jumps-use-hammerspoon.md` and run
> `sase memory init`. Model its shape on `decisions:open-pdf-writes-are-coordinated`
> (status `accepted`, `decided` = the landing date).
>
> - **Applies to:** bob-navigation-hotkeys ⌘↵, chezmoi Hammerspoon, and any future "open
>   at page" client.
> - **Claim:** clients open the exact PDF path in Highlights, then hand the 1-based
>   physical page to Hammerspoon's `bob-highlights-page` URL handler. The handler never
>   opens files; it verifies file identity by inode, waits for the restore to settle,
>   lands through the page runner with ⌃O history, and reports inside Highlights.
>   Opening writes nothing.
> - **Rejected:** `highlights://<name>#page=N` (bookmarks are looked up by file name,
>   and duplicate basenames exist); Obsidian driving Accessibility through JXA or
>   osascript (it would grant Obsidian Accessibility and duplicate the runner); a new
>   `bobcapture://` scheme (none exists, and every change pays the macOS CI gate).
> - **Cost:** depends on Hammerspoon and its Accessibility grant; assumes page labels
>   equal physical pages.
> - **Reopens when:** Highlights ships scripting or a path-exact URL form, or the first
>   annotated PDF with `/PageLabels` appears (then bob renders the page index into the
>   note).
> - **Evidence:** the research ref above and this plan.

## Acceptance on the Mac (Bryan, after deploy)

The phases verify everything that can run on Linux. These checks need the real apps. Run
them after `chezmoi update` on the Mac and `bob plugins sync`.

1. Press ⌘↵ on a quote, on a follower piece of a `Pages 11–12` group, on a caption, on a
   🔖 task, and on a Task Link to a 🔖 task. Highlights opens the right PDF, lands on
   the page, and alerts `Page N of M · ⌃O back to p. X`. ⌃O returns.
2. Test cold (Highlights quit) and warm, and on a PDF whose last position is page 30
   (the restore race).
3. Test the duplicate-basename pair `lib/papers/harness_engineering.pdf` and
   `lib/blogs/harness_engineering.pdf`. Each lands in its own document.
4. Test an already-open tab: `open -b` selects it and the jump lands.
5. Moving with j/k into a Live Preview callout reveals its source lines, and the rail
   and chip follow. The toast and chip look right in light and dark themes.
6. Quit Hammerspoon and press ⌘↵ on a quote: the PDF still opens, and the page jump
   either silently does not happen or the toast says it needs Hammerspoon.

## Out of scope (follow-ups)

- Rendering bob's `page_index` into the note when a label differs from the physical page
  (trigger: the first PDF with `/PageLabels`).
- Repairing the dangling same-note 🔖 links (the fallback covers them at read time).
- Reading-view click targets, Bob Refs "open at page" through the same URL contract,
  in-page landing from the annotation's `/Rect`, and a duplicate-basename warning at
  intake.
- Matching a 🔖 task to the exact follower piece it came from (bookmarks open the
  passage start by design).
