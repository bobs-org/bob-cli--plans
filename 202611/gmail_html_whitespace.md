---
tier: tale
title: Gmail HTML bodies follow browser whitespace rules
goal:
  Templated HTML mail (Alertmanager, CI, newsletters) saves as clean cards with no &#32;
  noise lines.
size: medium
decisions:
  layout_rows:
    ask:
      Should a layout-table row with one filled cell keep that cell's paragraphs instead
      of flattening them?
    choices:
      keep_blocks:
        Single-cell rows keep paragraphs, headings, and lists; multi-cell rows still
        join with ·
      flatten: Rows stay one paragraph; Alertmanager-style mail becomes one run-on line
    default: keep_blocks
    why: Mail layout tables are mostly one content cell between empty spacers
    answer: keep_blocks
proposed_by: bbugyi200.athena.1fa.2
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 18:01:16
status: wip
---

# Gmail HTML bodies: follow browser whitespace rules

## Why

`bob gmail` saved the Alertmanager email `[athena alert] FIRING MacDiskLow` into the
vault as a mail note whose conversation card is mostly noise. Here is an excerpt of the
saved card body:

```text
> &#32;
> &#32;1 alert for&#32;
> &#32;alertname=MacDiskLow
> &#32;
> &#32;
> &#32;[View in Alertmanager](<…>)
> &#32;
> &#32;**\[1\] Firing**
> &#32;
> &#32;**Labels**
>
> &#32;alertname = MacDiskLow
> instance = 127.0.0.1:9100
> …
> [​](<…?b=1>)[&#32;](<…?b=3>)
```

Every templated HTML mail with indented source (Alertmanager, CI and notification mail,
most newsletters) gets this. In reading mode it shows as a broken, gappy layout. In
source and live-preview mode the `&#32;` entities show as literal text.

## Root cause (verified by reproduction)

`src/native/gmail/body/html.rs` passes the HTML's source formatting through as if it
were content:

1. **Text nodes keep raw source whitespace.** `parse_container` and
   `parse_inline_children` push `Inline::Text(text.to_string())` unchanged, newlines and
   indentation included. `<br>` also becomes `Inline::Text("\n")`, so the renderer
   cannot tell a real break from a source newline.
2. **The renderer turns every newline into a Markdown line.** `render_inline_text`
   (`src/native/gmail/body/inert.rs`) splits on `\n`. It renders whitespace-only lines
   as `&#32;` and leading or trailing line whitespace as `&#32;`. So `<br />` plus a
   source newline becomes a blank line (a paragraph break), and each indented source
   line becomes an `&#32;`-prefixed line.
3. **Whitespace-only text between blocks becomes paragraphs.** `flush_inline` wraps a
   whitespace-only inline run in `Block::Paragraph`, and `render_block` writes it as an
   `&#32;` paragraph.
4. **Layout tables flatten whole cells into one paragraph.** Alertmanager nests tables,
   so `scan_table` marks the table unsupported and `linear_table` runs. Each cell's
   blocks go through `blocks_to_cell_inlines` (paragraphs joined with `" "`) into one
   paragraph per row. Today the source newlines happen to keep this readable. Once fix 1
   lands, the whole message would collapse into one line unless rows keep their blocks
   (decision `layout_rows`).
5. **Invisible tracking anchors survive.** Mailjet appends `<a href=…>&#8203;</a>` and
   `<a href=…> </a>`. They render as links with invisible labels (`[​](…)`,
   `[&#32;](…)`), and they count toward `has_link`.

Reproduction: a scratch unit test that fed `render_html` a cut-down copy of
Alertmanager's default `email.html` template (nested `body-wrap` → `main` →
`content-wrap` tables, `content-block` cells, `<strong>Labels</strong><br />` followed
by `name = value<br />` runs, and a footer table) reproduced the saved card. The output
is copied below and is what the regression test in step 6 should pin against.

```text
&#32;
&#32;
&#32;1 alert for&#32;
&#32;alertname=MacDiskLow
&#32;
&#32;
&#32;
&#32;[View in Alertmanager](<http://athena:9093/>)
&#32;
&#32;**\[1\] Firing**
&#32;
&#32;**Labels**

&#32;alertname = MacDiskLow
instance = 127.0.0.1:9100

&#32;**Annotations**

&#32;summary = low

&#32;[Source](<http://athena:9090/graph>)

&#32;
&#32;
&#32;
&#32;[Sent by Alertmanager](<http://athena:9093/>)&#32;
&#32;
&#32;
&#32;


[​](<http://t.example/a>)[&#32;](<http://t.example/b>)
```

## Target behavior

HTML-derived inline content follows CSS `white-space: normal`, the browser default:

- Every run of collapsible whitespace in a text node becomes one space. Collapsible
  whitespace means HTML ASCII whitespace (U+0020, U+0009, U+000A, U+000C, U+000D) plus
  U+0085, U+2028, and U+2029, so that `normalize_sender` cannot turn them back into
  newlines later. U+00A0 (no-break space) and other non-collapsible characters stay as
  they are.
- `<br>` is the only source of a line break inside a paragraph. Block boundaries still
  separate paragraphs.
- Spaces at the start or end of a line are dropped: at paragraph edges, right after a
  `<br>`, and right before a `<br>`. This also applies when the space sits inside inline
  structure (`Strong`, `Emphasis`, a link label). A space that follows another space
  collapses even across inline element boundaries (`a <b> b</b>` → `a **b**`).
- Leading and trailing `<br>` breaks in a paragraph are dropped.
- A paragraph left with no visible content (only whitespace and breaks, with no `Code`,
  safe link, or image) is dropped. This applies in every segment (main, quoted,
  signature), in list items, and in table cells.
- An `Inline::Link` whose label has no visible character is dropped from the output, and
  it no longer counts toward `has_link`. A label has no visible character when, after
  normalization, it contains only whitespace and these invisible format characters:
  U+00AD, U+034F, U+180E, U+200B through U+200F, U+2060 through U+2064, and U+FEFF.
  Rust's std has no general-category lookup, so use an explicit set. A label that holds
  `Code` or an image link counts as visible. Remote-image links (label `🖼 …`) are
  unaffected.
- (`layout_rows = keep_blocks`) In `linear_table`, a row with exactly one non-empty cell
  emits that cell's main blocks as separate `RenderNode::Block`s, in order, instead of
  one flattened paragraph. Rows with two or more non-empty cells keep today's `" · "`
  join. Semantic Markdown tables are unchanged apart from cell whitespace normalization.

Unchanged on purpose:

- A space between renderer-owned inline structure on the same line still renders as
  `&#32;` (`Hello&#32;**friend**.`). That is a deliberate inert-safety choice in
  `render_inline_text`.
- The plain-text body path (`plain.rs`, `prose_blocks`) still treats newlines as
  meaningful. Normalization is HTML-only.
- `<pre>` (`Block::Code`), headings (already collapsed by `render_text_single`), and
  `Inline::Code` contents.
- Every inert-safety guarantee in `inert.rs`. Trimming must not let sender text start a
  line with Markdown or Obsidian syntax. `render_inline_text` already runs
  `escape_line_start` on each trimmed line, so that guarantee holds, and the adversarial
  corpus test enforces it.

## Implementation

1. **Parse-time collapse (`html.rs`).** Add a helper (e.g.
   `collapse_html_whitespace(&str) -> String`) that folds each collapsible run to one
   `' '`. Apply it wherever an HTML text node becomes `Inline::Text`: in
   `parse_container`, including the `quote_tail` branch, and in `parse_inline_children`.
   Leave `visible_text` callers (`pre`, headings, `code`, `svg` titles, object
   fallbacks, quote-divider detection) alone. Keep `<br>` as `Inline::Text("\n")`, or
   introduce a dedicated `Inline::LineBreak` if that is cleaner. If you add the variant,
   handle it in every `match` over `Inline` (`cell_is_empty`, `inlines_have_link`,
   `coalesce_inlines`, the `render_*` functions) and render it exactly as `"\n"` renders
   today.
2. **Line-edge and duplicate-space normalization.** Add a pass (e.g.
   `normalize_inline_whitespace(Vec<Inline>) -> Vec<Inline>`) that walks the inline tree
   in document order, descending into `Strong`, `Emphasis`, and `Link` labels:
   - Track "at line start, or previous emitted char was a space".
   - Drop leading spaces at paragraph start and after each break.
   - Drop a space that follows a space.
   - Drop trailing spaces before each break and at paragraph end. A mirrored backward
     pass is fine.
   - Remove leading and trailing breaks.
   - Prune `Text` nodes and empty `Strong`/`Emphasis` wrappers that become empty.

   Treat `Code`, safe links, and image-link labels as visible content.

3. **Drop invisible links** in the same pass, or in a sibling helper.
4. **One choke point.** In `render_html`, after `split_segments` and **before**
   computing `has_table` and `has_link`, run `normalize_blocks(&mut Vec<Block>)` over
   `segments.main`, `segments.quoted`, and `segments.signature`. It should normalize
   `Paragraph` inlines (dropping paragraphs with no visible content), recurse into
   `List` items (dropping items that become empty, and the list if every item does), and
   normalize `Table` header and row cells. The `<p>`, `<dt>`, `<dd>`, and `<address>`
   arms of `parse_element_nodes` and table captions are covered by this pass, so no
   per-site trimming is needed. Leave `cell_is_empty` and `blocks_to_cell_inlines`
   working on the parsed (collapsed) inlines. Any double spaces their `" "` joins
   produce get collapsed by the final pass.

   > [!decision] layout_rows = keep_blocks

5. **Layout rows.** Make `TableRow` keep each cell's main `Vec<Block>` (still routing
   `Quote`/`Signature` nodes to `scan.extra` exactly as today). Derive the inline form
   with `blocks_to_cell_inlines` wherever the semantic `Block::Table` path or the
   multi-cell linear path needs it. In `linear_table`, use
   `cell_is_empty(&blocks_to_cell_inlines(..))` to decide which cells are filled. If
   exactly one is filled, extend the output with its blocks. Otherwise keep the current
   `" · "` paragraph. (With `flatten`, skip this step and keep `TableRow` as is.)
6. **Tests (`html.rs` tests module, plus `inert.rs` if helpers live there).**
   - Alertmanager regression: build the fixture from the cut-down template described in
     Root cause (two nested layout tables, a footer table, and the two Mailjet anchors
     `<a href="http://t.example/a">&#8203;</a><a href="http://t.example/b"> </a>`).
     Assert the exact markdown. With `keep_blocks`, the expected shape is:

     ```text
     1 alert for alertname=MacDiskLow

     [View in Alertmanager](<http://athena:9093/>)

     **\[1\] Firing**

     **Labels**
     alertname = MacDiskLow
     instance = 127.0.0.1:9100
     **Annotations**
     summary = low
     [Source](<http://athena:9090/graph>)

     [Sent by Alertmanager](<http://athena:9093/>)
     ```

     The shape is authoritative: no `&#32;` anywhere, and no invisible-label links.
     Exact escaping or spacing may differ where existing inert rules require it.

   - Focused cases:
     - `<p>  a\n   b  </p>` → `a b`
     - `<p>a<br>\n   b</p>` → `a\nb`, with no blank line
     - `<p>a <b> b</b></p>` → `a&#32;**b**`
     - `<p><br>a<br></p>` → `a`
     - whitespace-only text between two `<div>` blocks emits no paragraph
     - `&nbsp;` survives (the U+00A0 path stays non-collapsible)
     - a link whose label is only `&#8203;` or a space is dropped and leaves `has_link`
       false when no other link exists
     - `<a href=…><img alt="x" src="https://…"></a>`-style image links survive
     - a single-filled-cell layout row with two `<p>` children yields two paragraphs
     - a two-cell row still yields `A · B`
   - Update existing expectations only where they encoded the old raw-whitespace
     behavior or the old single-cell flattening. Do not weaken the inert-safety
     assertions.
   - The existing inert-safety suites must stay green unchanged in intent:
     `adversarial_corpus_stays_inert_and_readable`,
     `helpers_neutralize_hostile_fragments`,
     `checkbox_fragments_never_survive_as_requirements`, the scanner assertions, the
     `render.rs` body tests, and the `tests/gmail_*.rs` integration tests (`gmail_sync`,
     `gmail_shell`, and so on). Where an integration fixture's saved-card text changes
     because of this fix, update the fixture and say so in the commit body.

7. **Docs.** First read the `docs.md` reference memory through `/sase_memory_read`.
   Behavior changes route to the contract/reference page. In `docs/gmail.md` →
   `## Body rendering`, add one bullet stating the observable rules: HTML whitespace
   collapses like a browser's, only `<br>` and block boundaries break lines, whitespace
   with nothing else drops, links with no visible label drop, and a layout-table row
   with one filled cell keeps that cell's paragraphs. Do not touch `CHANGELOG.md` or the
   crate version (release-train rule). Use a `fix(gmail): …` commit subject. This is not
   a CLI or JSON contract break, so no `!`.

## Verification

- `just fmt`, `just lint`, and `just test` (or `just all`) pass.
- `just docs-check` passes.
- The new Alertmanager regression test fails on the current renderer and passes after
  the change. Confirm this once by running it before you implement steps 1–5.

## After it ships (for Bryan, not part of this change)

Existing mail notes keep their old cards until rebuilt. Once a release containing this
fix reaches the vault's Gmail writer host, rebuild the alert note with
`bob gmail save mail/2026/261122_athena_alert_firing_macdisklow.md -R` (`-R` rebuilds
managed message cards and keeps the authored sections). Do the same for any other noisy
templated-mail notes.
