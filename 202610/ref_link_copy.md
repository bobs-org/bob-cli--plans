---
tier: epic
title: Copy any link from a Markdown-born reference (Copy card L)
goal: 'While reading a bob-rendered report in Highlights, Command-Y L Return puts
  the URL of the link on the current page on the clipboard, exactly the URL the PDF
  opens. Option-Return copies it as a Markdown link, A copies every link, and `bob
  ref links` gives the CLI and agents the same data.

  '
decisions:
  l_key:
    ask: Should L open Link… only on references with saved Markdown, and stay Markdown
      link elsewhere?
    choices:
      shared: L opens Link… on Markdown-born references and stays Markdown link elsewhere;
        no card changes
      links_only: L always means Link…; the Markdown link moves to Option-U and web-article
        cards lose their L row
    default: shared
    why: No reference with saved Markdown has a source URL (0 of 235), so the two
      meanings never meet
    answer: shared
  backfill:
    ask: How far should this epic take link records for references saved before bob
      recorded links?
    choices:
      run: Build bob ref recatalog, dry-run it on athena, check the gates, then write
        in revertable batches
      build: Build and test the command and run only the dry run; Bryan runs the write
        pass himself
      defer: Skip the recatalog phase and file a follow-up task; L stays hidden on
        today's references
    default: run
    why: The 95 saved reports with links are the ones Bryan reads; without backfill
      L never shows there
    answer: run
  bare_urls:
    ask: Should bare URLs (https://… without brackets) become real links in new PDFs,
      so L lists them too?
    choices:
      link: Markdown readers gain autolink_bare_uris; new PDFs make bare URLs clickable
        and L lists them
      skip: Readers stay as they are; new PDFs render as today and L lists only written
        links and autolinks
    default: link
    why: Pandoc stays the only judge of what a link is; a URL you can't click or copy
      helps no one
    answer: link
phases:
- id: links-catalog
  title: Render-time link records in the code catalog
  depends_on: []
  size: medium
  description: 'links-catalog: extend the render-time catalog filter to record every
    URI link (exact target, text, autolink flag, lead-in context, section) with an
    invisible inline PDF anchor for its page, store them as an additive `links` array
    in code.json, and apply the bare_urls reader decision.'
- id: links-cmd
  title: bob ref links read surface
  depends_on:
  - links-catalog
  size: medium
  description: 'links-cmd: add the read-only `bob ref links <REF> [N]` command (human,
    json, markdown, raw; page filter) with bob-computed label, site, and Markdown
    forms, golden-tested, with completion, catalog, help, and docs.'
- id: recatalog
  title: Annotation pages, attach-source links, and bob ref recatalog
  depends_on:
  - links-catalog
  - links-cmd
  size: medium
  description: 'recatalog: map links to pages from an existing PDF''s link annotations,
    record links in attach-source, add `bob ref recatalog` and the doctor count, then
    act on the backfill decision on athena.'
- id: mac-core
  title: RefsCore link catalog, L row, and the link picker model
  depends_on:
  - links-cmd
  size: medium
  description: 'mac-core: decode `bob ref links`, add the L row, its key resolution,
    loading queue and hidden reasons, the Option-U Markdown-link alternate, and a
    pure link picker plus link preview model, all unit-tested with portable-check.'
- id: mac-ui
  title: Link picker, link well, and panel wiring
  depends_on:
  - mac-core
  size: medium
  description: 'mac-ui: fetch links beside code when the Copy card opens, draw the
    L row, the link picker rows, and the link well in light and dark, extend the design
    fixtures and docs, and run the Mac acceptance.'
proposed_by: bbugyi200.athena.14k
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 15:37:25
status: wip
bead_id: bob-cli-ct
---

- **PROMPT:** [prompts/202610/ref_link_copy.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_link_copy.md)
- **BEAD:** [bob-cli-ct](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ct/README.md)

# Plan: Copy any link from a Markdown-born reference (Copy card `L`)

## The experience we are building

Bryan is on page 7 of an agent's reading-list report in Highlights. He wants the URL of
the Imagen model card it links to, so he can paste it into a chat. He presses **⌘Y**:
Bob Refs opens on that PDF with the Copy card, and a new row reads
`[L] Link…   75 links · ● 4 on this page · github.com, arxiv.org, ai.google.dev`. He
presses **L**. The link picker opens with page 7's first link already selected. Each row
reads like a search result: the link's text on top, the site and path below, and
`● p. 7` on the right. Under the rows, a small well shows the exact URL that will be
copied, wrapped and typeset with its host emphasized. He presses **↓ ↵**. The pasteboard
is verified, Refs hides, a glass HUD says `Copied link · ai.google.dev`, and Highlights
is still frontmost.

`⌘Y L ↵` does the right thing even when typed faster than bob answers. **⌥↵** copies
`[Imagen 4 model card](https://…)`. **A** copies every distinct URL in the report, one
per line, and **⌥A** copies them as a Markdown list. `bob ref links <ref> 12 | pbcopy`
gives the CLI and agents the same bytes.

## Design principles

1. **Copy what the PDF opens.** A link's URL is the target pandoc handed to hyperref,
   recorded during the render that made the PDF. A test proves that every recorded URL
   equals the URI of the PDF link annotation that follows its anchor. Nothing parses
   Markdown at read time, and nothing reads the PDF text layer. Older stores are
   re-cataloged by the same pandoc filter, never by a second parser
   (`decisions:code-copy-reads-render-time-catalog`).
2. **The same grammar as `C`.** `L` opens a picker and copies nothing by itself. Digits
   copy a visible row, `↵` copies the plain URL, and `⌥↵` copies the Markdown form. `⌥`
   swaps the previews in place, `A` copies all of them (like `H`), and `esc` goes back.
3. **Start where the reader is.** When the Highlights page is known, the `L` row counts
   that page's links and the picker opens on them.
4. **Read like a search result.** The row shows the human label first and the site and
   path second. The well shows the full payload before anything is copied.
5. **Additive everywhere.** `code.json` stays version 1. A missing `links` array means
   "not recorded yet", and an empty array means "no links". Older bobs ignore the new
   field. An older app ignores the new command, and an app talking to an older bob says
   "Link copy needs a newer bob" (`decisions:mac-capture-is-a-thin-client`).
6. **Fast keys are never misread.** `L` queues behind a pending links load exactly as
   `C` queues behind code.

## What counts as a link

A link is a pandoc `Link` element whose target starts with a URI scheme of at least two
characters (`[A-Za-z][A-Za-z0-9+.-]+:`). The research survey of today's 236 saved
sources shapes this table. It found about 1,360 external link occurrences in 95 reports:
1,205 written links and 156 `<…>` autolinks. The median report has 9 links, the 90th
percentile has 33, and the largest has 75. About 7% of occurrences repeat a URL. It also
found roughly 39 bare URLs, plus 1,850 internal and 413 relative links, which `L` skips
on purpose.

| In the Markdown                                  | Recorded                                      |
| ------------------------------------------------ | --------------------------------------------- |
| `[text](https://…)`, `[text][ref]`, titled links | yes                                           |
| `<https://…>`                                    | yes, `autolink: true`                         |
| `<me@example.com>`                               | yes, target `mailto:me@example.com`, autolink |
| bare `https://…`                                 | per decision `bare_urls`                      |
| `[x](#section)` (internal cross-reference)       | no                                            |
| `[x](sources/gem.md)` (relative path)            | no                                            |
| `![alt](https://…)` (image)                      | no                                            |
| `[![badge](img.svg)](https://…)` (linked image)  | yes; text is the alt text                     |
| URLs inside inline code or code blocks           | no (they are code; `C` copies them)           |
| raw HTML `<a href>`                              | no (pandoc keeps it raw)                      |

Every occurrence is its own row, in document order. Repeats stay, because each sits in
its own place on its own page. Only `A` deduplicates.

> [!decision] bare_urls = link Add `autolink_bare_uris` to every Markdown reader bob
> renders with: the plain `.md` route (today pandoc infers `markdown` from the file
> extension, so pass an explicit `-f markdown+autolink_bare_uris`), the `-M` vault-note
> route, the snapshot reader `SNAPSHOT_PANDOC_READER` (`commonmark_x` does not enable it
> either), the diagrams collect pass, and the native record pass that attach and
> recatalog run. Use one shared constant per reader so that they cannot drift. New PDFs
> then render bare URLs as `BobLinkInk` links, breakable at slashes, instead of
> unbreakable plain text. Document this in `docs/highlights-create.md`.

> [!decision] bare_urls = skip Leave every reader unchanged. Bare URLs are not links:
> the PDF and `L` both ignore them. Say so in the `docs/highlights-create.md` catalog
> section.

## Architecture at a glance

```text
bob ref create report.md                                  (bob-cli, any host)
  pandoc ─► code_catalog.lua record pass: CodeBlocks (today) + URI Links (new)
       │      + \BobLinkAnchor{bob-link-N} just before each anchorable link
       ▼
  render.pdf ─► /Dests reader ─► page per link (page_source "anchor")
       ▼
  .bob/ref-sources/<type>/<stem>--<hash12>/code.json   "links": [ … ]  (additive, v1)

older stores ─► bob ref recatalog ─► native record pass + PDF /URI annotations
                                     (page_source "annotation")

bob ref links <ref> -f json ─► verified links + label, site, markdown   (read-only)
       │
Bob Mac Capture: Copy card L row ─► link picker + link well ─► pasteboard
       ▲
⌘Y in Highlights ─► Copy card (keys replayed) ─► L ↵
```

## Shared contracts

Every phase builds against these contracts. A phase that has to deviate records why in
its bead and updates the docs that own the contract.

### Catalog records: `code.json` `links`

The report pass writes `{version: 1, blocks: […], links: […]}`. `code.json` keeps
`"version": 1` and gains an optional top-level array:

```json
"links": [
  {
    "n": 12,
    "url": "https://ai.google.dev/gemini-api/docs/imagen",
    "text": "Imagen 4 model card",
    "title": null,
    "autolink": false,
    "context": "Google's current image model is documented in the",
    "section": { "number": "4", "title": "Image models" },
    "page": 7,
    "page_source": "anchor"
  }
]
```

- `n` is the 1-based document order over recorded links. Footnote links count at their
  reference position.
- `url` is the pandoc `Link` target, byte for byte. That is what hyperref writes as the
  annotation's `/URI`.
- `text` is `pandoc.utils.stringify` of the link content, with whitespace collapsed.
- `title` is the link title, or `null`.
- `autolink` is true for the `uri` and `email` classes (the `<…>` forms and bare URLs).
- `context` is the plain text of the enclosing block before the link,
  whitespace-collapsed. It keeps the last 160 characters, cut on a word boundary with a
  leading `…` when cut. It is `null` when empty.
- `section` uses the same rule as code blocks: the nearest preceding heading, with
  numbers from the report pass.
- `page` is the 1-based physical page where the link starts. `page_source` is `anchor`
  (this render), `annotation` (recatalog or attach), or `null`.
- A missing `links` array means the catalog predates link recording. Readers must
  tolerate it. `StoredCatalog` keeps ignoring unknown fields so that older bobs can read
  new stores. Never add `deny_unknown_fields`.

### Anchors

- `code_catalog.tex` gains
  `\newcommand{\BobLinkAnchor}[1]{\special{pdf:dest (#1) [@thispage /XYZ @xpos @ypos null]}}`.
  It is a bare whatsit: no `\par`, no penalty, and it takes no space.
- The record pass inserts `RawInline("latex", "\\BobLinkAnchor{bob-link-N}")`
  immediately before each recorded `Link`, where hyperref already emits its own
  `pdf:bann` special. It does this only when `FORMAT` matches latex.
- **No anchor inside moving arguments:** headings, figure and table captions, and table
  header rows. Links there are still recorded, with a `null` page. (An anchor there
  would be copied into the TOC, the bookmarks, or repeated table heads.)
- Links must be recorded in the same document-order walk that counts headings. Today the
  filter is typewise, and an inline `Link` handler would run before the block-level
  `Header` handler, which breaks section association. Use a `traverse = 'topdown'`
  filter, or block handlers that walk their own inlines.

### `bob ref links`

```text
bob ref links <REF> [N] [-f, --format human|json|markdown|raw] [-p, --page PAGE]
```

- `REF` resolves exactly like `bob ref show` and `bob ref code`.
- Without `N`, the default format is `human`. It prints the title, then
  `<path> · N links · M sites`. Then come dim section headers, and per link a bold
  number, the label, and a dim `p. 7` label, with a second line of the cyan site and the
  dim path. It ends with `hint: bob ref links <ref> <n> | pbcopy`, where `<n>` is the
  first link's number. Plain layout applies off a TTY.
- With `N`, the default is `raw`: exactly that link's `url` bytes, with no trailing
  newline (the same rule as `bob ref code N`). `-f markdown` with `N` prints its
  `markdown` form. `-f json` with `N` keeps only that link.
- `-f markdown` without `N` prints `# <title>`, then for each section with links a
  `## <number> <title>` heading, then `- <markdown> · p. 7` lines.
- `-p PAGE` keeps links whose `page` equals `PAGE`. Human output with no match prints
  `no links on page 7`.
- The command is read-only, never runs pandoc, and verifies the source digest through
  `source_store::read_store`. It accepts only a `source_markdown` pointer under
  `.bob/ref-sources/` (the same guard as `bob ref code`).
- Exit codes:
  - 0 for success, and for every JSON response that reports a `status`.
  - 1 for an unknown or ambiguous reference, using `ref show`'s envelopes and codes.
  - 1 for a missing `N` (`no link 7 (6 links)`).
  - 1 for any non-`available` status in a non-JSON format. It prints a one-line reason
    and a hint. The exception is `no_links` with human output, which prints `no links`
    and exits 0.
  - 2 for usage errors.

JSON (schema version 1, the envelope shape of `ref code`):

```json
{
  "ok": true,
  "schema_version": 1,
  "command": "ref links",
  "generated_at": "2026-10-27T09:12:44-04:00",
  "reference": {
    "path": "ref/chat/agent_image_generation_toolset.md",
    "title": "Agent Image Generation Toolset",
    "marker_id": "agent_image_generation_toolset"
  },
  "status": "available",
  "detail": null,
  "source": {
    "path": ".bob/ref-sources/chat/agent_image_generation_toolset--91c03b27b252/source.md"
  },
  "links": [
    {
      "id": "91c03b27b252/link-12",
      "n": 12,
      "url": "https://ai.google.dev/gemini-api/docs/imagen",
      "text": "Imagen 4 model card",
      "title": null,
      "autolink": false,
      "label": "Imagen 4 model card",
      "label_kind": "text",
      "context": "Google's current image model is documented in the",
      "site": "ai.google.dev",
      "markdown": "[Imagen 4 model card](https://ai.google.dev/gemini-api/docs/imagen)",
      "section": { "number": "4", "title": "Image models" },
      "page": 7,
      "page_source": "anchor"
    }
  ]
}
```

- `status`:

  | Status           | Meaning                                                             |
  | ---------------- | ------------------------------------------------------------------- |
  | `available`      | Verified, and the catalog records at least one link.                |
  | `no_links`       | Verified, and the catalog records zero links.                       |
  | `not_recorded`   | Verified, but the catalog predates link recording (no `links`).     |
  | `no_catalog`     | Source verified, but the catalog is missing or has `catalog_error`. |
  | `no_source`      | No pointer.                                                         |
  | `source_missing` | Pointer set, but `source.md` is absent on this host.                |
  | `source_invalid` | Digest mismatch, an unreadable file, or an unsupported version.     |

  `detail` is a human sentence for every non-`available` status. For `not_recorded` it
  reads `links were not recorded for this reference; run bob ref recatalog`.

- bob computes these fields (never the app):
  - `id` is generation-scoped (`hash12/link-n`).
  - `label`: for written links, `text`, with `label_kind: "text"`. For autolinks,
    `context` with trailing whitespace and `: — – - (` trimmed when that leaves
    something (`"context"`). Otherwise the URL without its `scheme://`, or without
    `mailto:` (`"url"`).
  - `site`: for `http`, `https`, `ftp`, and `ftps`, the lowercase host without a leading
    `www.`. For `mailto`, `email`. For anything else, the scheme name.
  - `markdown`: for an autolink, `<` + text + `>`. Otherwise `[text](url)`, with `\`,
    `[` and `]` in the text backslash-escaped. The destination is wrapped in `<…>` when
    it contains whitespace or unbalanced parentheses. No title is included.
- Growth is additive: the app decodes everything except `schema_version` loosely.

### Clipboard contract

- **Plain (default):** `url`, exactly, with no trailing newline.
- **Markdown (⌥):** `markdown`, exactly, with no trailing newline.
- **All (`A` in the picker):** the distinct `url`s in document order (first occurrence
  wins), joined by `\n`, with no trailing newline. **⌥A** joins `- ` + `markdown` for
  those first occurrences.
- **Never:** percent-decoding, normalization, tracking-parameter stripping, or
  truncation. The display may wrap, style, or clip a URL. The payload never changes.

## Bob Mac Capture UX

### The `L` row

`L` joins `C` and `M` in the existing `.contents` group, between them, so the group
reads C · L · M whichever load finishes first:

```text
╭───────────────────────────────────────────────────────────────────╮
│ ▣ COPY                                         p. 7 in Highlights │
│   Agent Image Generation Toolset                                  │
│ ▌[I] Marker ID      agent_image_generation_toolset                │
│  [T] Title          Agent Image Generation Toolset                │
│  [W] Wiki link      [[ref/chat/agent_image_generation_toolset]]   │
│  ──────────────────────────────────────────────────────────────── │
│  [N] Note path …  [O] …  [P] …  [F] …  [R] …                      │
│  ──────────────────────────────────────────────────────────────── │
│  [C] Code block…    6 blocks · ● 2 on this page · python, bash    │
│  [L] Link…          75 links · ● 4 on this page · github.com, …   │
│  [M] Markdown       source.md · 41 KB · captured Oct 20           │
│  ──────────────────────────────────────────────────────────────── │
│ ↵ copies Marker ID · ↑↓ choose · esc back                         │
╰───────────────────────────────────────────────────────────────────╯
```

- **L · Link…** opens the link picker and copies nothing itself, like `C` and `H`.
- Its preview is `N links` (`1 link`). When a page is known and K > 0, it adds
  `● K on this page`, with the dot in accent color. Then come up to three sites by
  frequency, with ties in first-appearance order.
- While links load, the preview reads `Loading links…`. Pressing `L` then goes pending
  ("Loading links… esc cancels") and opens the picker when the links arrive.
- With `L` selected, the hint reads `↵ opens Link…`.
- Every Source URL row gains an **Option-U** alternate: label `Markdown link`, the same
  payload and HUD as today's `L` Markdown-link row. That keeps the Markdown link one
  chord away on any card where `L` means Link….

> [!decision] l_key = shared `RefsCopyKey.parse("l")` keeps returning `.markdownLink`.
> `RefsCopyCard.handle` resolves `.key(.markdownLink, alt)` to `.key(.linkPicker, alt)`
> whenever `item.sourceMarkdown != nil`. It does this before queueing, so replayed keys
> are already resolved. On those cards the `.sources` Markdown-link row is not built
> (Option-U carries it). On every card without saved Markdown, `L` behaves exactly as
> today, including its "No source URL for this reference" reason. `docs/refs.md`
> documents both meanings in one sentence.

> [!decision] l_key = links_only `RefsCopyKey.parse("l")` returns `.linkPicker`, and
> `.markdownLink` stops being a fixed key: the Markdown link lives only on Option-U.
> Pressing `L` on a card without saved Markdown shows "L picks links in saved Markdown —
> ⌥U copies the Markdown link" when a source URL exists. Otherwise it shows "No saved
> Markdown — link copy needs a reference bob rendered from Markdown". Update the
> `RefsCopyTests` that assert the old `L` row.

### Visibility

A hidden row explains itself when its key is pressed (on cards where `L` means Link…):

| State                        | L row   | Message on key press                                                       |
| ---------------------------- | ------- | -------------------------------------------------------------------------- |
| links loading                | loading | (pending) Loading links… esc cancels                                       |
| `available`                  | shown   | —                                                                          |
| `no_links`                   | hidden  | No links in this reference                                                 |
| `not_recorded`               | hidden  | Links weren't recorded for this reference yet — bob ref recatalog          |
| `no_catalog`                 | hidden  | Links weren't recorded for this reference — M copies its Markdown          |
| `no_source`                  | hidden  | No saved Markdown — link copy needs a reference bob rendered from Markdown |
| `source_missing`             | hidden  | Saved Markdown isn't on this Mac yet — sync the vault                      |
| `source_invalid`             | hidden  | Saved Markdown doesn't match its digest — run bob ref doctor               |
| older `bob` (no `ref links`) | hidden  | Link copy needs a newer bob                                                |
| fetch failed or timed out    | hidden  | Links unavailable — nothing copied                                         |

With `backfill = defer`, the `not_recorded` message drops its `— bob ref recatalog`
tail.

### The link picker

```text
╭───────────────────────────────────────────────────────────────────╮
│ ▣ LINKS 75                                     p. 7 in Highlights │
│   Agent Image Generation Toolset                                  │
│   4  Image models                                                 │
│  [1] OpenAI image guide                                      p. 6 │
│      developers.openai.com /docs/guides/image-generation          │
│ ▌[2] Imagen 4 model card                                   ● p. 7 │
│      ai.google.dev /gemini-api/docs/imagen                        │
│  [3] github.com/black-forest-labs/flux                     ● p. 7 │
│      github.com                                                   │
│ ╭ ai.google.dev · Image models ─────────────────────── p. 7 ╮    │
│ │ https://ai.google.dev/gemini-api/docs/imagen?hl=en#model-  │    │
│ │ versions                                                    │    │
│ ╰─────────────────────────────────────────────────────────────╯    │
│ ↵ copies URL · ⌥↵ Markdown · A all · 1–9 pick · esc back          │
╰───────────────────────────────────────────────────────────────────╯
```

- **Header:** caption `LINKS N`, the title, and the `p. N in Highlights` chip when a
  page is known (the existing header logic).
- **Keys** follow `C`:
  - `↑↓` and `⌃N/P/J/K` move the selection and the well; the window scrolls.
  - `1`–`9` copy the link at that visible position.
  - `↵`, ⌘Y, or a click copies the selection plain. `⌥↵`, ⌥+digit, ⌘⌥Y, and ⌥-click copy
    the Markdown form.
  - `A` copies all, and `⌥A` copies all as Markdown.
  - `esc` and `⌃[` return to the card. A letter with no binding shows `Nothing on ‘x’`.
- **Rows** use the 36 pt picker metrics and the existing sectioned window
  (`RefsSectionedWindow`).
  - Section headers are drawn exactly like the code picker's.
  - Line 1: the keycap digit, then the `label`. It uses callout primary for `text` and
    `context` labels, and 11.5 pt monospaced primary for `url` labels, tail-truncated.
    On the right is the page label (`p. 7`, caption monospacedDigit, secondary), led by
    an accent `●` when the link is on the known page.
  - Line 2: the `site` (caption semibold, secondary), then the rest of the URL after the
    host (`/path?query#fragment`, 11 pt monospaced, tertiary, middle-truncated). The
    path is omitted when line 1 already shows the URL (`label_kind: "url"`). A link
    without a page shows no page label.
  - There are no favicons and no per-site hues: nothing is fetched from the network, and
    the reference-kind tiles already own color.
- **Initial selection** reuses the code picker rule on `page`. With page P known, select
  the first link on P. Otherwise select the first link after P, else the last link
  before P, and ignore pageless links. Scroll so the selection is the first visible row
  when possible. Without a page, select link 1.
- **Never auto-copy**, even when there is one link.

### The link well

This is the one new visual element. It is a compact sibling of the code preview well,
reusing its tokens:

- **Surface:** the code well's fill, hairline stroke, corner radius, Increase Contrast
  values, and Reduce Transparency behavior. Light and dark are both designed.
- **Header strip** (18 pt): `site · Section title` on the left (caption2 semibold,
  secondary) and `p. 7` on the right (caption2 monospacedDigit, tertiary). The right
  side gains `· Markdown` while ⌥ is held.
- **Body:** the exact payload in 12 pt monospaced, wrapped, never soft-truncated:
  - Wrap greedily to the well's column count. When a line must split, prefer to break
    after `/ ? & = # - _ .` if one falls in the last 12 columns; otherwise hard-break.
  - Styled runs (display only): the `scheme://` in tertiary, the host in semibold
    primary, the path in primary, and `?query` and `#fragment` in secondary. In the ⌥
    Markdown form, `[`, `](` and `)` (or `<` and `>`) are tertiary and the text is
    secondary.
  - When the payload cannot be split into URL parts, draw it unstyled in primary.
- **Height:** set once when the picker opens and again only when the card width changes.
  Moving the selection never resizes anything. Body lines = the lines the longest plain
  URL in this picker needs at the well's width, clamped to 2–5. The row window gets the
  rest: at most 9 rows and at least 2.
- **Overflow:** when the payload needs more lines than the well has (for example, a long
  ⌥ Markdown form), the last visible line fades and a capsule `+N chars` sits at bottom
  right. The well's tooltip shows the full payload.
- **Display cap:** the first 2,000 characters. The payload is never touched.

### HUD and VoiceOver

- **HUD titles:** `Copied link · ai.google.dev`, `Copied Markdown link · ai.google.dev`,
  `Copied 67 links`, and `Copied 67 links · Markdown`. The value is the link's `label`,
  or for `A` the first URL followed by `…`.
- **Row:** "Link 2 of 75, Imagen 4 model card, ai.google.dev, page 7, on this page, key
  2".
- **Well:** "Preview, link to ai.google.dev". The URL is never spoken.
- **Opening:** "Links. Press a number or Return."

### ⌘Y in Highlights

Nothing new is registered. ⌘Y already opens the Copy card on the focused PDF and replays
buffered keys through `handleCopyCardKey`. `L` (with the `l_key` resolution) and the
links pending queue make `⌘Y L ↵` and `⌘Y L 2` copy the right URL when typed within 150
ms of ⌘Y.

## Conventions for every phase

- **bob-cli:** never edit the `Cargo.toml` version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`). Every change here is additive
  (`feat`), not `!`. Run `just check` and `just docs-check` before finishing.
  - A new public `bob ref` leaf needs four things: a completion `Entry` plus a kinds
    decision for each value-taking argument (`src/native/completion/kinds.rs`), a
    `__catalog` entry or an exclusion (the `public_leaves_are_entries_or_excluded`
    test), an updated grouped help snapshot (`tests/cli/help.rs`), and regenerated
    `docs/reference/cli/ref.md` (`just docs-gen`). The `bob-cli-cc` epic broke master by
    missing these; check them explicitly.
  - Follow `cli_rules.md`: excellent `--help` with examples, a short alias for every
    long option, options alphabetical, and subcommands alphabetical in the `Library`
    help group.
- **bob-mac-capture:** read its `AGENTS.md` first. Never land Swift you have not
  compiled.
  - Run `just portable-check` on athena.
  - For app-target changes, compile a scratch copy on the Mac: rsync without
    `.build`/`.git`, then `bundle.sh` and `just typecheck-tests`, then delete the copy.
    If the Mac is offline, say the macOS targets are unverified.
  - macOS CI may still show the pre-existing RefsPanel failures tracked by
    `bob-cli-ci`/`bob-cli-cr`. Do not chase them; confirm that your own tests pass.
- **Thin client:** the app decodes and presents. It never decides what a link is, and
  never derives `label`, `site`, or `markdown`. Splitting a URL into styled display runs
  is presentation only and never changes a payload.

## Phase links-catalog

Repository: bob-cli. Goal: every new Markdown render records its links with pages and
stores them in `code.json`. Code blocks, the store layout, and every existing reader
behave exactly as before.

- **Record pass** (`src/native/highlights_ref/code_catalog.lua`):
  - Record each URI `Link` per "What counts as a link", in document order, in the same
    walk that counts headings (see Anchors). Store the records in document metadata
    under `bob-link-records`, next to `bob-code-records`.
  - Compute `context` from the enclosing block (`Para`, `Plain`, `Header`, or
    `LineBlock`) by stringifying the inlines before the link.
  - Insert the inline anchor before each anchorable link, latex only. Skip headings,
    captions, and table heads.
  - Keep the `pcall` guards: the filter never writes to stderr and never fails pandoc.
- **Report pass** (`code_catalog_report.lua`):
  - Emit `links` with sections numbered by the existing `section_numbers` logic.
  - Encode an empty list as `[]`, extending the existing `gsub` array fix.
- **TeX:** add `\BobLinkAnchor` to `code_catalog.tex`.
- **Rust** (`code_catalog.rs`, `source_store.rs`):
  - `CodeCatalog` gains `links: Option<Vec<CatalogLink>>`.
  - `build()` maps `bob-link-N` named destinations to `page` with
    `page_source: "anchor"`.
  - `read_report_catalog` (attach) reads links pageless.
  - `StoredCatalog` gains `links: Option<Vec<StoredLink>>`, with
    `#[serde(default, skip_serializing_if = "Option::is_none")]`. `build_stored_catalog`
    always writes `Some` for a successful catalog, so a fresh store has `"links": []`
    when there are none.
  - `code_summary` and the create output gain one line:
    `links: 14 (github.com, arxiv.org, ai.google.dev)` or `links: none`. Dry run is
    unchanged apart from that line.
- **Reader flags:** apply the `bare_urls` branch.
- **Tests** use real pandoc and XeLaTeX, skipped with the existing `skip_pandoc_xelatex`
  pattern. Fixtures:
  - a written link and a reference-style link;
  - a titled link;
  - an autolink and an email autolink;
  - a bare URL followed by `).` (asserting the recorded target excludes the punctuation
    under `link`, and that nothing is recorded under `skip`);
  - an internal `#` link and a relative link (not recorded);
  - an image (not recorded) and a linked image (recorded);
  - links inside a nested list, a blockquote, a table cell, and a footnote;
  - a link in a heading (recorded, `page: null`, and no anchor in the TOC or bookmarks);
  - a link whose text wraps across a page break;
  - a repeated URL;
  - a URL with `%`, `#`, `&`, `~`, and parentheses.

  Assertions:
  - For every anchored link, the recorded `url` equals the `/URI` of the first link
    annotation at or after its anchor on that page. This proves that copy equals click.
  - Each `page` equals the page where `pdftotext` finds the link text.
  - Code-block records and pages are unchanged for the existing catalog fixtures.
  - Under `skip`, the PDF's per-page text is identical with and without the record pass
    (the anchors add no text).
  - `bob ref code` reads a store with `links` and one without them identically.

- **Docs:** the `docs/highlights-create.md` "Code catalog" subsection becomes "Code and
  link catalog": what is recorded, the anchors, the moving-argument rule, and the bare
  URL policy. Update the documented `code.json` shape wherever it lives (`docs/ref.md`
  and the saved-Markdown section).
- **Done when** a fresh `bob ref create` of a report with links writes `"links"` into
  `code.json` with anchor pages, and `just check` and `just docs-check` are green.

## Phase links-cmd

Repository: bob-cli. Goal: the read surface from the shared contract, golden-tested.

- New `src/native/ref_library/links.rs`, with a clap builder beside `code` in
  `ref_library/cli.rs`, dispatch in `highlights_ref/mod.rs`, and `links` in the
  `Library` help group.
- Move the helpers that `code.rs` and `links.rs` share into one small module rather than
  copying them: `source_read`, `valid_store_pointer`, `shortest_ref`, `shell_quote`, the
  page label, and the error and unavailable printing. `bob ref code` output must stay
  byte-identical, and its tests prove that.
- Derive the status from `read_store`. `Available` or `NoCode` plus `links: None` gives
  `not_recorded`. `Some([])` gives `no_links`. A non-empty list gives `available`.
  `NoCatalog`, `NoSource`, `SourceMissing`, and `SourceInvalid` pass through.
- Compute `label`, `label_kind`, `site`, and `markdown` here, never in the app. Use the
  `url` crate only if it is already a dependency; otherwise write a small scheme and
  host splitter with tests.
- **Human output** uses `Styler`, mirroring `bob ref code`: a bold title, then the
  metadata line, dim section headers, bold numbers, a cyan site, a dim path, and dim
  pages. It ends with the hint line. Plain output applies off a TTY.
- **Help** gets examples:
  - `bob ref links agent_image_generation_toolset`
  - `bob ref links agent_image_generation_toolset 12 | pbcopy`
  - `bob ref links agent_image_generation_toolset -p 7`
  - `bob ref links agent_image_generation_toolset -f markdown`
  - `bob ref links agent_image_generation_toolset -f json`
- **Tests** go in `tests/cli/ref_library/links.rs`, against hand-built fixture stores:
  - a JSON golden whose key set matches what the Mac decodes (`mac-core` uses this
    golden as its fixture);
  - raw `N` bytes with no trailing newline;
  - `-f markdown` for `N`, and for the whole list, with escaping (brackets in text,
    spaces and parentheses in the URL);
  - autolink labels from context, and URL labels when there is no context;
  - `site` for `www.` hosts, `mailto`, and `obsidian://`;
  - `-p` filtering;
  - all seven statuses, including `not_recorded` for a store written before this epic;
  - unknown and ambiguous references;
  - a missing `N`;
  - human output without color.
- **Docs:** `docs/ref.md` gets a "`bob ref links`" section in the contract's terms,
  linked from "Choose a command". `docs/reference/json-outputs.md` gets the `ref links`
  envelope. Then run `just docs-gen` and `just docs-check`.
- **Done when** the goldens pass, `bob ref code`'s tests are untouched and green, and
  `just check` and `just docs-check` pass.

## Phase recatalog

Repository: bob-cli. Goal: references whose catalogs predate links gain them, and
attach-source records links from now on.

- **Annotation pages** (new `src/native/highlights_ref/pdf_links.rs`, beside
  `pdf_dests.rs`):
  - Read every page's `/Annots` in page order, then annotation order. Keep
    `/Subtype /Link` annotations with an `/A` action of `/S /URI`.
  - A link that wraps produces one annotation per line, each with the same URI (this was
    verified on athena with xdvipdfmx). Treat a following same-URI annotation as a
    continuation piece of the previous one when it begins a new line: it is lower on the
    same page with a smaller left edge, or it is the first annotation on the next page.
  - Match catalog links to pieces greedily in document order. Each link takes the first
    unconsumed annotation, at or after the cursor, whose URI equals its `url`. Compare
    exactly first, then after percent-decoding both sides. Consume its continuation
    pieces. The link gets `page_source: "annotation"`. A link with no match keeps
    `page: null`; bare URLs never had annotations in older PDFs.
  - Unit tests use lopdf-built PDFs covering wrapped pieces, a page-spanning piece,
    adjacent same-URL list items (not merged), a missing annotation, and
    percent-encoding differences.
- **Record pass without XeLaTeX:** generalize attach-source's native pandoc pass
  (`extract_candidate_blocks` in `ref_library/attach.rs`) into one shared helper. It
  runs the record and report filters on given bytes with the route's reader (markdown
  for `markdown`, `plain_note`, and `attached`; the snapshot reader for
  `note_snapshot`), with `--number-sections` and the same top-level shift as the render.
  It returns blocks and links. attach-source keeps using it for blocks.
- **attach-source** records `links` with annotation pages in every catalog it writes.
  This is relevant to the open `bob-cli-co` follow-up.
- **`bob ref recatalog [REF]... [-d, --dry-run]`** goes in the `Library` help group. Its
  help says: "Add link records to saved-source catalogs written before bob recorded
  links. Never changes saved Markdown or recorded code blocks."
  - Eligible references have a store that reads `available` or `no_code` with no `links`
    array. `REF` arguments restrict the set; the default is every eligible reference.
  - Per reference:
    - read and verify the store;
    - run the record pass on `source.md`;
    - take pages from the current library PDF's annotations;
    - rewrite `code.json` atomically with `links` added. Every other field stays
      byte-identical after re-serialization, including `blocks` and `captured_at`.
  - Run under a reference-mutation session (`bob_sync.lock`, then `scan.lock`) in
    batches of up to 100. Each batch is one scoped commit
    (`bob ref recatalog: N references`), revertable with `git revert`. Warn with the
    existing allowlist message if `code.json` is ignored.
  - **Output:** one line per reference, either `added 14 links (12 on pages)` or
    `skipped (<reason>)`. The reasons are `already recorded`, `no catalog`,
    `source missing`, `source invalid`, `no PDF`, and `pandoc failed: …`. Then a
    summary, then `writes: none` on a dry run. If pandoc is missing, exit 1 with an
    install hint.
- **Doctor:** the `ref sources` row's counts gain links, for example
  `ok (412 saved · 1,903 code blocks · 1,361 links)`. Catalogs without links append
  `· N without links — bob ref recatalog` without changing the row's level.
- **Tests:**
  - a fixture store without `links` gains them, and its other fields are unchanged;
  - a dry run writes nothing;
  - already-recorded stores are skipped;
  - the batch commit's contents are correct;
  - attach-source writes `links`;
  - the doctor counts.
- **Docs:** a `docs/ref.md` "`bob ref recatalog`" section and the doctor row, plus the
  `just docs-gen` output.

> [!decision] backfill = run Build and test the command, then on athena:
>
> 1. Dry-run over every eligible reference. The survey expects about 95 references with
>    links and about 1,360 links, out of roughly 236 stores.
> 2. Gate. Stop and report instead of writing if more than 2% of stores fail the record
>    pass, if any skip reason looks systematic, or if fewer than 85% of non-bare links
>    get an annotation page. That last case signals a URI matching bug.
> 3. Write.
> 4. Spot-check 10 references across sizes. For each, compare every paged link's `url`
>    and `page` from `bob ref links <ref> -f json` against the PDF's `/URI` annotations.
> 5. Record the counts and commit SHAs in the phase bead.

> [!decision] backfill = build Build and test the command, then run only the dry run on
> athena. Record its summary and the exact write command for Bryan in the phase bead.

> [!decision] backfill = defer Do not build this phase. File one follow-up task bead
> with `/sase_new_task` (type `feature`, size `large`) that cites this section and the
> attach-source gap, then close the phase. In that case, also drop the
> `— bob ref recatalog` tail from the `not_recorded` message in phase `mac-core`, and
> from the `not_recorded` detail in phase `links-cmd` if it already shipped.

## Phase mac-core

Repository: bob-mac-capture, `Sources/RefsCore` plus `Tests/RefsCoreTests` (verifiable
with `just portable-check` on athena).

- **Decoding** (new `RefsLinks.swift`, modelled on `RefsCode.swift`):
  - `RefsLinksStatus`, `RefsLink`, `RefsLinksResponse` (`SchemaVersioned`, rejecting any
    `schema_version` other than 1), and lossy element decoding.
  - Use `links-cmd`'s goldens as fixtures.
  - `RefsLinksLoadState` (`notRequested`, `loading`, `loaded`, `unsupported`, `failed`).
- **Fetcher:** `RefsFetching.links(path:)`, whose default implementation throws
  `.unsupported`. `BobRefsFetcher.links` runs `["ref","links",path,"-f","json"]` on lane
  `refs-links`, and maps exit 2 to `.unsupported`.
- **Card model** (`RefsCopy.swift`):
  - `RefsCopyKey.linkPicker` (glyph `L`). Apply the `l_key` branch.
  - `RefsCopyCatalog` takes `links:`. It builds the `L` row between `C` and `M`, the
    Option-U alternate on Source URL, and the hidden reasons from the Visibility table.
  - `RefsCopyCard` gains `links`, `linksLoaded(path:response:)`,
    `linksFailed(path: unsupported:)`, a `.links` pending load, `isLinkKey`, the links
    timeout message, and a `linkPicker` layer.
  - `perform` routes to `handleLinkPicker` the way it routes to `handleCodePicker`.
  - Replay and stop-at-write semantics follow the existing queue.
  - Links loading never blocks other rows.
- **Picker model:** `RefsLinkPicker` on `RefsSectionedWindow`.
  - It provides entries with section groups, the page-aware initial selection (reuse the
    code rule), digit and arrow handling, and `copyRow(markdown:)` with the HUD strings
    from the contract.
  - It also provides `allLinksRow(markdown:)` (dedupe by `url`, first occurrence, in
    order), the `L` row preview string, and the VoiceOver labels.
- **Well model:** `RefsLinkPreview` is pure.
  - It has the display-run splitter (scheme, host, path, query, fragment, and the
    Markdown punctuation and text roles), the greedy column wrap with preferred break
    characters, and the overflow count.
  - `RefsLinkPreviewLayout.make(availableHeightUnderHeader:columns:longestURLLength:…)`
    returns the well height, the body lines (2–5), and the picker window size (2–9).
- **Tests:**
  - decoding for every status, plus an older payload missing optional fields;
  - `l_key` resolution, both with and without saved Markdown and with a source URL;
  - Option-U;
  - row visibility and messages for each state;
  - the `L` row preview, including the on-page count and site ranking;
  - initial selection (on the page, between links, after the last link, no page,
    pageless links);
  - the window plus headers;
  - plain, Markdown, all, and ⌥all payloads byte for byte;
  - wrap points and styled runs (URL, unsplittable string, Markdown form), and overflow;
  - `L ↵` queued while links load opens the picker and copies the preselected link;
  - `L` after `linksFailed` shows the right message and copies nothing.

> [!decision] backfill = defer The `not_recorded` message reads "Links weren't recorded
> for this reference yet".

## Phase mac-ui

Repository: bob-mac-capture, the app target. Compile per its `AGENTS.md`.

- **Panel model** (`RefsPanelModel.swift`):
  - `openCopyCard` starts `library.links(path:)` in parallel with notes and code
    whenever `item.sourceMarkdown != nil`.
  - The task is token-guarded, cancelled in `closeCopyCard`, and maps
    `BobClientError.unsupportedCommand` like code does.
  - Apply effects through the existing `applyCopyCardEffect` (pasteboard check, hide,
    and HUD). Announce the picker when it opens after a load.
- **Views:**
  - In `RefsCopyCardView`: the `L` row, the `LINKS N` header, the link picker rows, the
    `↵ opens Link…` hint, the picker hint line, and the ⌥ Markdown previews.
  - A new `RefsLinkPreviewWellView` draws `RefsLinkPreview` with the code well's surface
    tokens.
  - Pass the well's column count from `NSFont.monospacedSystemFont(ofSize: 12, …)`
    advancement and the measured width.
  - Put new metrics in `RefsVisualTokens`.
  - Respect light and dark, Increase Contrast, and Reduce Transparency throughout.
    Follow the UX spec above exactly; it is the design.
- **Design fixtures:** extend `RefsCopyCardDesignTests` with light and dark versions of:
  - a card with C, L, and M rows and an on-page link count;
  - the link picker on page 7 with sections and `●`;
  - the picker with ⌥ held (the Markdown well);
  - autolink rows with a context label and with a URL label;
  - a long URL that overflows the well;
  - a loading `L` row;
  - a `not_recorded` card.

  Review the CI render artifacts and iterate until they look right; spacing, alignment,
  line-2 rhythm, and the well's proportions matter.

- **Docs:**
  - `docs/refs.md` "Copy card": `L` (and its `l_key` meaning), the picker, the well, the
    messages, Option-U, and `⌘Y L ↵`.
  - `docs/keyboard.md`: `L`, `A` and `⌥A` in the link picker, `⌥↵` Markdown, and the
    Return row's picker list.
  - Then run `just docs-check`.
- **Acceptance on the Mac** (best effort):
  - It needs a `bob` with `ref links` on the Mac: the next release-train build, or a
    local build if the Mac has a Rust toolchain. If neither is available, report the
    acceptance as unverified rather than skipping silently.
  1. On athena, `bob ref create` a fixture report with:
     - a written link and a reference link;
     - an autolink in a "Sources" list;
     - a bare URL;
     - a mailto;
     - a link wrapping across a page break;
     - a heading link;
     - a repeated URL;
     - a URL with a query and a fragment;
     - about 30 filler links in later sections, so the picker scrolls.
  2. Let the vault sync. Open the PDF in Highlights at a later page.
  3. Type `⌘Y L ↵` fast and paste. Compare against `bob ref links <ref> <n>` on athena
     and against the URL Highlights opens on click.
  4. Repeat with a digit, `⌥↵`, `A`, and `⌥A`.
  5. Open a backfilled reference (if `recatalog` wrote) and check that its pages and `●`
     look right.
  6. On a web article with a source URL, check that `L` and Option-U behave per `l_key`.
  7. Report any mismatch as a failure, not a caveat.
