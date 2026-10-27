---
tier: epic
title: Saved Markdown, code, and link catalogs for web-article references
goal: 'References that bob ref create captures from web-article URLs (single URL,
  -H saved HTML, rereads, multi-URL collections, and their -f overwrite replacements)
  get a saved source.md and a code/link catalog, so the Bob Mac Capture Refs Copy
  card''s M, C, and L modes work for them exactly as for Markdown-born references.

  '
phases:
- id: adapter-source-view
  title: Renderer emits the source view and code anchors
  depends_on: []
  size: medium
  description: 'adapter-source-view: web-clip adapter keeps image origins, writes
    render/source.html and render/code-anchors.json, injects Chrome-destination anchors
    for every printed pre, and reports both paths; the Rust client decodes them.'
- id: web-article-store
  title: Single web articles save source.md and a catalog
  depends_on:
  - adapter-source-view
  size: medium
  description: 'web-article-store: read Chrome catalog /Dests, convert the source
    view to GFM, catalog it with the existing filters, map anchor and annotation pages,
    and write the web_article store and pointer on every single-article create path,
    with tests and docs.'
- id: web-collection-store
  title: Multi-URL collections save one source and catalog
  depends_on:
  - web-article-store
  size: small
  description: 'web-collection-store: persist a web_collection store from the compose
    response before collection intake, numbering anchors across parts.'
- id: web-overwrite-store
  title: Web overwrites repoint instead of clearing
  depends_on:
  - web-collection-store
  size: medium
  description: 'web-overwrite-store: stage the new web store inside the overwrite
    transaction after capture instead of clearing source_markdown, keeping Clear for
    PDF URL and arXiv and as the fallback.'
- id: mac-hint-copy
  title: Hidden-row reasons name web pages
  depends_on:
  - web-article-store
  size: small
  description: 'mac-hint-copy: in bob-mac-capture, reword the no-saved-Markdown code
    and link reasons to say Markdown or a web page, with tests and docs; no logic
    change.'
decisions:
  decision_record:
    ask: Should the epic record a new accepted decision describing how web-article
      sources and catalogs are derived?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.ao
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 11:21:26
status: wip
bead_id: bob-cli-d3
---

- **PROMPT:** [prompts/202610/web_ref_saved_sources.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/web_ref_saved_sources.md)
- **BEAD:** [bob-cli-d3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-d3/README.md)

# Saved Markdown, code, and link catalogs for web-article references

## Findings (answer to the question)

Bryan suspected the Bob Mac Capture Refs Copy card's **M** (Markdown), **C** (code block
picker), and **L** (link picker) modes only work for references that `bob ref create`
rendered from Markdown. That is correct.

- The Mac app only asks bob for code and links when the reference note has a
  `source_markdown` pointer (`RefsPanelModel` `wantsCode`, plus the
  `item.sourceMarkdown` guards in `Sources/RefsCore/RefsCopy.swift`). All three modes
  then read `bob ref code -f json` / `bob ref links -f json`, which serve
  `.bob/ref-sources/<type>/<stem>--<hash12>/{source.md,code.json}`.
- In bob-cli, only three routes write that store and pointer: the plain `.md` route,
  `-M`, and vault-note snapshots (`persist_markdown_store`, `persist_snapshot_store`,
  and the Markdown/vault-note overwrite paths in `src/native/highlights_ref/create.rs`).
- Every URL route skips it:
  - Web article: single URL and `-H` saved HTML in `clip.rs`; multi-URL collections in
    `collection.rs`. Both call `finish_create_intake(..., None)`.
  - PDF URL and arXiv. Web overwrites also pass `source_store: None`, which clears the
    pointer.
- So every URL-captured reference shows "No saved Markdown — code copy needs a reference
  bob rendered from Markdown". In the vault today, 24 `blogs` and 1 `docs` URL
  references, plus 13 URL `papers`, have no saved source; 255 of 481 `chat` references
  do.
- The original research (`research:202610/ref_code_block_copy/ref_code_block_copy.md`)
  scoped this deliberately: "Web clips can join later. PDFs and arXiv papers never
  will". It also listed "Web-clip sources (reader-mode Markdown)" under **Later**.

**Scope of this epic.** Web articles (single URL, `-H`, rereads, multi-URL collections,
and their `-f overwrite` replacements) gain a saved Markdown source and a code/link
catalog at create time. PDF URLs and arXiv stay unsupported because no Markdown exists
for them. The accepted decision "Code Copy Reads bob's Render-Time Catalog" forbids the
PDF text layer as a code-copy source. Their hidden-row reason is already correct apart
from the wording fix in the last phase.

## Design

The Mac stays a thin client (decision `mac-capture-is-a-thin-client`). Once bob writes a
valid store and pointer, M/C/L light up with **no Mac logic change**. The Mac decodes
`route` and `page_source` as free strings (`Sources/RefsCore/RefsCode.swift`,
`RefsLinks.swift`).

Web articles have no Markdown that a renderer consumed: the adapter prints Bob-owned
reader HTML with headless Chrome (`scripts/web_clip/web_clip_render.py` `print_pdf`).
So:

1. **Source view.** The renderer writes `render/source.html`. It is the printed article
   as plain semantic HTML, with no CSS and no Bob anchors:
   - The masthead becomes an `<h1>` title, dek, byline, and a provenance line that links
     the source URL with the captured date and word count.
   - The body is `shift_headings(article html)`, so headings match the PDF outline.
   - Every `<img>` keeps only `src`, `alt`, and `title`, and `src` is its original
     absolute URL. The adapter records that URL as `data-bob-origin` when
     `_localize_images` rewrites `src` to `assets/…`. Images with no remote origin (data
     URIs, inline SVG) are removed.
   - Relative hrefs are absolutized against `source_url`.
2. **`source.md`.** bob converts that file with the existing `create::pandoc_command()`:
   `pandoc -f html -t gfm --wrap=none --preserve-tabs`. The result is stored write-once
   like every other source. A probe with pandoc 3.1.3 confirmed:
   - `--preserve-tabs` is required, because the HTML reader otherwise expands tabs.
   - HTML → GFM → re-read as `gfm` keeps code bytes identical: literal tabs, trailing
     spaces, an embedded ` ``` ` line, NBSP, and a `mermaid` class.
   - An `<img>` with extra attributes survives only as raw HTML, which is why step 1
     strips image attributes.
3. **Catalog content.** Reuse
   `ref_library::attach::record_source_catalog(bytes, route)`. It already runs the real
   catalog Lua chain (`--preserve-tabs`, record filter, code-break filter, report
   filter) over saved Markdown and returns a pageless catalog. Map the new routes
   `web_article` / `web_collection` to reader `gfm` (new constant beside
   `MARKDOWN_PANDOC_READER`). Pandoc stays the single semantic authority, and C and M
   agree by construction because both derive from `source.md`.
4. **Code pages from Chrome anchors.** Before `page.pdf`, a JS pass in the print page
   handles each `pre` inside the article body, or every part body, in document order
   (`k = 1..`):
   - Record `textContent`.
   - Insert empty `<span id="bob-code-k">` / `<span id="bob-code-k-end">` as the first
     and last children. Never touch an existing id.
   - Append one `display:none` `<nav>` of `<a href="#bob-code-k">` and `#bob-code-k-end`
     links.

   A probe with Playwright Chromium 1248 confirmed:
   - Chrome writes named destinations only for ids that a same-document link targets. A
     link inside a `display:none` container still counts, and the hidden links add no
     link annotations.
   - Untargeted ids get no destination. Existing clips without internal links have no
     `/Dests`.
   - Chrome writes them as a PDF-1.1 catalog `/Dests` dictionary, not a `/Names` →
     `/Dests` tree.

   The renderer writes `render/code-anchors.json`:
   `{"version":1,"blocks":[{"k":1,"text":"…"}]}`.

   bob matches catalog blocks to anchors greedily in order by normalized equal text
   (CR/CRLF → LF, strip trailing `\n`). A match takes `page` / `page_end` from the
   `bob-code-k` / `bob-code-k-end` destinations with `page_source: "anchor"`. An
   unmatched block keeps `page: null` and is still copyable. Print
   `warning: N of M code blocks have no page` and/or
   `warning: K printed code blocks are not in the saved Markdown` when that applies.

5. **Link pages.** Reuse `pdf_links::annotate_catalog_links(&mut links, pdf)`, which
   matches the PDF's URI annotations and is already used by `bob ref recatalog`
   (`page_source: "annotation"`).
6. **Store write.** Mirror `persist_markdown_store`:
   - Call `source_store::write_for_create(...)` with route `web_article` or
     `web_collection` and `origin` set to the final URL. For a collection, `origin` is
     the cleaned URLs joined by `\n` in argument order.
   - Print `source_markdown:`, `code:`, and `links:` lines.
   - Pass the pointer to `finish_create_intake`.
   - `-o` prints `source_markdown: none (output outside the library)`.
   - Any failure (no pandoc, adapter reported no source view, conversion or catalog
     error, store lock timeout) prints
     `warning: …; skipping source store (reference still filed)` and
     `source_markdown: none (…)`, and never fails the create.
   - Dry runs are unchanged, because source bytes do not exist before capture.
   - Queued intake (`-k`, or intake unavailable) keeps working through the existing
     birth lookup (`find_birth_store` by PDF digest).

Adapter protocol stays `1`; the new response fields are additive. The adapter ships
inside the bob binary (materialized scripts), so it never lags bob. A
`BOB_WEB_CLIP_ADAPTER` override that omits the fields just yields
`source_markdown: none`.

## Phases

### Phase `adapter-source-view` — renderer emits the source view and code anchors

- **Size:** medium
- **Depends on:** none
- **Repo:** bob-cli (`scripts/web_clip/`, `src/native/highlights_ref/clip_adapter.rs`)

Work:

- **Image origins.** In `web_clip_adapter.py` `_localize_images`, add
  `data-bob-origin="<absolute original URL>"` when an http(s) image is localized.
  Confirm that `nh3` sanitizing (which runs earlier) and
  `web_clip_render.normalize_images` / `prefix_part_assets` keep the attribute or
  re-derive it.
- **Source view.** In `web_clip_render.py`, add a source-view assembler that reuses the
  masthead field logic in `assemble_masthead`. Both `render()` and `render_collection()`
  write `render/source.html`.
  - A collection gets an `<h1>` collection title and byline/captured line.
  - Each part gets an `<h2>` part title, that part's provenance line, and its body
    shifted one more level, mirroring `assemble_collection`.
  - Omit the internal contents list.
- **Anchors.** Add the anchor JS pass in `print_pdf` (or a hook it calls) and write
  `render/code-anchors.json`. Verify that empty `<span>` anchors produce destinations.
  If Chrome skips zero-size spans, put the start anchor on the `pre` itself when it has
  no id, and keep a non-empty-box end marker.
- **Responses.** The `capture` (when rendering) and `compose` responses add absolute
  `source_html` and `code_anchors` paths.
- **Self-test.** Extend `--self-test` and fixtures. Cover: a fixture with tabs, trailing
  spaces, a `language-*` class, a `mermaid` block, an image with a remote origin, and a
  data-URI image (removed). Also cover that `code-anchors.json` texts equal the DOM
  texts, and that the printed PDF's catalog `/Dests` contains `bob-code-1` and
  `bob-code-1-end`.
- **Rust client.** `clip_adapter.rs` response structs gain
  `#[serde(default)] source_html: Option<String>` and `code_anchors: Option<String>`,
  with byte-identical request tests unchanged and new deserialization tests.

Done when: `just check-web-clip-adapter` passes, and a real
`bob ref create <article URL> -k` leaves `render/source.html` and
`render/code-anchors.json` in a kept workdir (`BOB_WEB_CLIP_KEEP_WORKDIR=1`) whose texts
match the article's code.

### Phase `web-article-store` — single web articles save source.md and a catalog

- **Size:** medium
- **Depends on:** adapter-source-view
- **Repo:** bob-cli

Work:

- **`/Dests` reader.** `pdf_dests::named_dest_pages` also reads the catalog-level
  `/Dests` dictionary (name → array, or a dictionary with `/D`). Merge it with the name
  tree, where the name tree wins on duplicates. Add a lopdf unit test shaped like
  Chrome's output.
- **New module `src/native/highlights_ref/web_source.rs`.** It covers HTML → GFM
  conversion, catalog via `record_source_catalog`, anchor page matching, link pages via
  `annotate_catalog_links`, and the warnings above. Use the rendered scratch PDF for
  page mapping, matching the Markdown route.
- **Routes.** In `ref_library/attach.rs`, `record_source_catalog` maps
  `web_article | web_collection` to `gfm`. The orphan-adoption allowlist in
  `classify_store_destination` accepts the web routes. Confirm that `recatalog` skips
  them ("already recorded") and that `bob ref doctor`'s `ref sources` row handles them.
- **Create wiring.** In `clip.rs` `capture_article`, persist the store after install on
  every branch (plain, `--audio`, `--listen`, reread `__vN`), print the three lines, and
  pass the pointer to `finish_create_intake`. `-H` replays share this path.
- **Tests.** Extend `tests/cli/highlights/fake_clip.rs` so the fake adapter can return
  `source_html` / `code_anchors` plus a fixture PDF with catalog `/Dests` and URI
  annotations. Assert:
  - the store files and `route: "web_article"`, plus the note `source_markdown`;
  - `bob ref code <ref> -f json` returns `status: available`, exact block text with
    tabs, and anchor pages;
  - `bob ref code --source` returns the GFM bytes;
  - `bob ref links -f json` is available with annotation pages;
  - `-o` prints `none`, and a fake adapter without the fields still files the reference
    with `source_markdown: none`;
  - a queued (`-k`) create's later scan birth adopts the pointer.
- **Docs.** Update `docs/highlights-create.md` "Saved Markdown": a route-table row for
  `web_article` ("pandoc GFM of the printed article HTML") and wording that is no longer
  Markdown-only. Also update `docs/highlights-clip.md` and the `docs/ref.md` row
  wording. Add one sentence to the `bob ref create` help "Web articles:" paragraph, then
  run `just docs-gen` and `just docs-check`.

Done when: a real article with code, captured on this host, opens in Bob Refs on the Mac
and M, C, and L all work. `just` lint and test targets pass.

### Phase `web-collection-store` — multi-URL collections save one source and catalog

- **Size:** small
- **Depends on:** web-article-store
- **Repo:** bob-cli

Work:

- **Wiring.** In `collection.rs`, persist a `web_collection` store from the compose
  response before its `finish_create_intake` call, using the shared `web_source.rs`
  helpers. `origin` is the newline-joined cleaned URLs.
- **Anchors.** Numbering spans all parts in document order.
- **Tests.** Extend the collection tests with a two-part fake compose response.
- **Docs.** Add a route-table row in `docs/highlights-create.md`.

Done when: a two-URL collection create yields one store whose `source.md` has both parts
and whose catalog blocks carry anchor pages.

### Phase `web-overwrite-store` — web overwrites repoint instead of clearing

- **Size:** medium
- **Depends on:** web-collection-store
- **Repo:** bob-cli

Today a web `-f overwrite` passes `source_store: None` (`clip.rs`
`capture_article_overwrite`; `collection.rs` around the `StagedOverwrite`). Its request
comes from `prepare_web_overwrite` → `prepare_local_overwrite` with
`OverwriteSourcePolicy::Clear`, so the live note loses `source_markdown`.

Work:

- **Staging.** After the adapter returns and before `overwrite::execute::execute`, build
  the source bytes and catalog, then stage them with `stage_overwrite_source_store`
  (route `web_article` / `web_collection`). Set `request.planned_source_store` and stop
  clearing the pointer, so the transaction installs, repoints, and rolls back the store
  like the Markdown overwrite.
- **Policy timing.** Eligibility is still proven before any fetch. Only the source
  policy is decided after capture.
- **Fallback.** If the source view or conversion is unavailable, keep today's Clear
  behavior with a warning.
- **Unchanged routes.** PDF URL and arXiv overwrites keep Clear.
- **Tests.** In `tests/cli/highlights/overwrite_web.rs`, cover: new bytes repoint; the
  old directory stays as history; same bytes reuse the store; rollback removes the new
  directory; the annotated-overwrite frozen note keeps the old pointer.
- **Docs.** Update the overwrite table in `docs/highlights-create.md` (split
  "Non-Markdown" into "Web article/collection" and "PDF URL, arXiv, local PDF").

Done when: overwriting a filed web article keeps M/C/L working on the replaced reading.

### Phase `mac-hint-copy` — hidden-row reasons name web pages

- **Size:** small
- **Depends on:** web-article-store
- **Repo:** bob-mac-capture (open with `/sase_repo`; read its `AGENTS.md`)

Work:

- **Hint strings.** In `Sources/RefsCore/RefsCopy.swift`, change the four `no_source` /
  `notRequested` reasons from "…needs a reference bob rendered from Markdown" to "…needs
  a reference bob created from Markdown or a web page" (code and link variants).
- **Tests and docs.** Update `Tests/RefsCoreTests/RefsCodeTests.swift`,
  `RefsLinksTests.swift`, and the wording in `docs/refs.md` / `docs/keyboard.md` where
  they describe which references have saved Markdown. Keep the decoding and gating logic
  unchanged.

Done when: `portable-check` passes and the Mac targets compile per that repo's
`AGENTS.md`.

> [!decision] decision_record The last implementing phase (`web-overwrite-store`) also
> adds an accepted strand
> `sase/memory/decisions/web-clip-sources-are-printed-html-as-gfm.md` through
> `/sase_memory_write`. It records: web-article `source.md` is pandoc GFM of the printed
> article HTML; its catalog is parsed from that source by bob's catalog filters; code
> pages come from Chrome named-destination anchors and link pages from URI annotations;
> PDF URLs and arXiv never get saved sources. Cite this epic's commits and the research
> ref. Then run `sase memory init`.

## Out of scope

- **PDF URL / arXiv M and C.** There is no Markdown, and the text layer is forbidden as
  a copy source.
- **Link copy for PDF-only references.** L could be built from URI annotations without a
  saved source, but that would change the `bob ref links` and Mac gating contracts. It
  deserves its own proposal.
- **Backfilling the ~24 web references captured before this change.** Live pages drift
  from the filed PDF, so a backfill would need a re-fetch plus the `attach-source`
  verifier. Recapture any reference that matters with `-f overwrite` (the
  `web-overwrite-store` phase keeps M/C/L on the replacement), or ask for a
  `bob ref attach-source` refetch mode as a follow-up.
- **Changing the web PDF renderer to pandoc/xelatex.** That would lose the Bob-owned
  reader template.

## Verification (whole epic)

- `just` test, lint, and docs gates in bob-cli; `just check-web-clip-adapter`.
- On a real host:
  1. `bob ref create <blog-with-code URL> -P <parent>` prints a
     `source_markdown: .bob/ref-sources/blogs/<stem>--<hash>/source.md` line plus
     `code:` and `links:` lines.
  2. `bob ref code <stem> 1 | od -c` shows exact tabs.
  3. `bob ref links <stem> -f json` lists pages.
  4. In Bob Mac Capture, the Refs Copy card (⌘Y in Highlights) shows working **M**,
     **C**, and **L** rows for that reference.
