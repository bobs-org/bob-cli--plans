---
tier: tale
title: Fix blank pages before tall Mermaid diagrams in bob ref create PDFs
goal:
  "Tall Mermaid diagrams fit on one page with their heading and return links on both
  Markdown routes: no body-empty page, no bottom-margin overflow."
size: medium
decisions:
  height_budget:
    ask: How much of the text height may one rendered diagram fill?
    choices:
      p80:
        Leaves ~130pt for heading, pills, and caption; tall diagrams print ~6% smaller
        than at 85%
      p85:
        Research-tested value with larger labels, but only ~100pt for heading, pills,
        and caption
    default: p80
    why:
      Planning stress tests fit H2 + two-line H3 + two pill rows; first diagram still
      prints at 75%
    answer: p80
  image_scope:
    ask:
      Should ordinary standalone Markdown images get the same height budget as Mermaid
      diagrams?
    choices:
      mermaid:
        Only Mermaid diagrams change; a tall image right after a heading can still leave
        a blank page
      all_images:
        Caps every standalone image on both Markdown routes; recreated tall screenshots
        print smaller
    default: mermaid
    why:
      The defect and evidence are Mermaid-specific; wider scope changes layouts nobody
      reported
    answer: mermaid
proposed_by: bbugyi200.athena.13q
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 11:20:26
status: wip
---

# Fix blank pages before tall Mermaid diagrams in `bob ref create` PDFs

## Context

Research report: `202610/research_mermaid_blank_pages.md` in the GitHub repo
`sase-org/sase--research`. Open it with `/sase_repo` if you need more detail. Summary:

- `bob ref create` swaps each Mermaid fence for a bare `pandoc.Image` paragraph
  (`src/native/highlights_ref/diagrams.lua`). Pandoc's LaTeX template only caps an
  unsized image at the _whole_ `\linewidth` × `\textheight` (`\maxwidth`/`\maxheight` on
  pandoc ≤ 3.1, `\pandocbounded` on newer pandoc).
- A tall diagram (the reported one is 514.5 × 714.75pt, while the generic text box is
  489.6 × 669.6pt) therefore fills the entire text height. It sits right after a heading
  that keeps with the next block. Bob's return-link row (`\BobBacklinks` in
  `return_links.tex`: `\nopagebreak[4]` + `\@afterheading`) makes that group bigger and
  harder to break. The group "heading + pills + diagram" is taller than a fresh page.
  TeX then emits a **body-empty page** followed by a page whose diagram **overflows into
  the bottom margin** ("Overfull \vbox … while \output is active"). TeX still exits 0.
- Return links make it worse but are not required. A plain `![](tall.pdf)` of the same
  size reproduces it too, so the cause is the image layout, not Mermaid.

Reproduced during planning with this host's pandoc 3.1.11.1. The repro used Bob's
generic-route flags, `return_links.lua`/`.tex`, and a synthetic 514.5 × 714.75pt PDF
placed after `### Data model and ownership` with one inbound link:

| Variant                                                                                      | Result                                                                                                                                 |
| -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Today                                                                                        | 4 pages, page 2 body-empty, `Overfull \vbox (52pt too high)`                                                                           |
| Wrapper below, budget 0.8                                                                    | 2 pages, no vbox overflow; diagram exactly 535.68pt tall (0.8 × 669.6); caption on the same page, above the 730.8pt bottom-margin line |
| Stress (H2 + two-line H3 + 14 inbound links; plus a `<div id>` figure with two links), today | 7 pages, one body-empty page, two vbox overflows                                                                                       |
| Same stress, wrapper 0.8                                                                     | 5 pages, no body-empty page, no vbox overflow                                                                                          |
| Pathological (H1 + H2, both wrapped, + 60 links) at 0.8                                      | no blank page, no overflow; TeX breaks inside the multi-line pill row instead                                                          |

**Rejected alternatives (do not implement):**

- _Override `\maxheight` in a header._ It only works on pandoc ≤ 3.1. Pandoc ≥ 3.2 sizes
  unsized images with `\pandocbounded`. It would also change every image.
- _Rust-computed `width=N%` attributes._ Pandoc 3.1.11 emits
  `width=0.N\textwidth,height=\textheight`, so the image is sized against `\textwidth`
  rather than `\linewidth`. That overflows inside list items and blockquotes, where
  Mermaid fences are supported. Pandoc also drops the image id.
- _A break penalty (e.g. `\penalty9999`) between the heading group and the diagram as a
  "safety valve"._ Tested and harmful. TeX then splits the heading and pills from the
  diagram even in the ordinary 2-page case: page 1 ends with the heading, and the
  diagram starts page 2. That happens because a nearly empty page before the heading
  costs "deplorable" while the penalty break costs less.

## Design

Bob owns the diagram's height budget in TeX, in context, independent of the pandoc
template version:

1. The swap filter wraps the rendered diagram's `Image` in fixed RawInline delimiters:
   `Para{ \BobFitBegin{}, Image, \BobFitEnd{} }`. The Image stays Pandoc AST (path
   escaping, identifier, and the snapshot route's frame all keep working). This follows
   the existing `\Bob…Begin{}`/`\Bob…End{}` convention that `note.lua` already
   recognizes.
2. `\BobFitBegin{}` opens an `lrbox`. Inside it, the image gets pandoc's normal defaults
   (≤ `\linewidth`, ≤ `\textheight`). `\BobFitEnd{}` measures the box. Only when its
   total height exceeds `\BobFitHeightBudget\textheight` does it apply
   `\resizebox*{!}{<budget>}`. It never upscales, and short diagrams are byte-for-byte
   the same layout as today. Width shrinks proportionally, so it always stays within
   `\linewidth`.
3. The heading, return pills, and (usually) the first caption line then fit together on
   a fresh page. That removes both the blank page and the overflow. Figure ids and
   return-link destinations are untouched.
4. The Rust-side 50% "labels may be small" warning uses the same budget, so its estimate
   matches the printed scale.

Tested macro body (budget 0.8). Use dedicated length registers, not
`\@tempdima`/`\@tempdimb`, which graphicx may use internally:

```tex
% bob ref create: diagram height budget (see docs/highlights-create.md).
\makeatletter
\newcommand{\BobFitHeightBudget}{0.8}
\newsavebox{\BobFitBox}
\newlength{\BobFitMaxHeight}
\newlength{\BobFitHeight}
\newcommand{\BobFitBegin}{\begingroup\begin{lrbox}{\BobFitBox}}
\newcommand{\BobFitEnd}{\end{lrbox}%
  \setlength{\BobFitMaxHeight}{\BobFitHeightBudget\textheight}%
  \setlength{\BobFitHeight}{\dimexpr\ht\BobFitBox+\dp\BobFitBox\relax}%
  \ifdim\BobFitHeight>\BobFitMaxHeight
    \resizebox*{!}{\BobFitMaxHeight}{\usebox{\BobFitBox}}%
  \else\usebox{\BobFitBox}\fi\endgroup}
\makeatother
```

`graphicx` (for `\resizebox`) is loaded by pandoc's template whenever the document
contains an image, which is always true when the macros expand. The macros are only
_defined_ in the preamble, so including the header unconditionally is safe. No new LaTeX
package is needed.

> [!decision] height_budget = p80 `\BobFitHeightBudget` is `0.8` and the Rust constant
> is `0.8`.

> [!decision] height_budget = p85 `\BobFitHeightBudget` is `0.85` and the Rust constant
> is `0.85`. Everything else is identical. Adjust the expected scale numbers in the
> tests (e.g. 0.85 × 669.6 / 714.75 ≈ 0.796).

## Changes

### 1. New `src/native/highlights_ref/diagrams.tex`

The macro block above, with the budget from the decision. `/src/**` is already in
Cargo's `include` list, so it ships automatically.

### 2. `src/native/highlights_ref/diagrams.rs`

- `pub(crate) const HEADER_INCLUDES: &str = include_str!("diagrams.tex");`, next to
  `SWAP_FILTER`, with a one-line doc comment.
- `pub(crate) const FIT_HEIGHT_BUDGET: f64 = 0.8;` (per decision). Its doc comment says
  it must equal `\BobFitHeightBudget` in `diagrams.tex`.
- `effective_scale(...)`: use `text_height_pt * FIT_HEIGHT_BUDGET` as the height limit;
  the width limit stays the same. Update the comment on `DiagramRoute` and the "A8"
  scale comments so they describe the budgeted height.
- Unit tests:
  - `HEADER_INCLUDES` contains
    `format!("\\newcommand{{\\BobFitHeightBudget}}{{{FIT_HEIGHT_BUDGET}}}")`, so the TeX
    and Rust budgets cannot drift.
  - Extend `scale_helpers_match_the_a8_contract`.
    `effective_scale(514.5, 714.75, 489.6, 669.6)` equals
    `FIT_HEIGHT_BUDGET * 669.6 / 714.75` (≈ 0.7495 at 0.8). The existing 200×100 (→ 1.0)
    and 1200×100 (width-bound) assertions stay unchanged.
- Add a `#[cfg(test)] pub(crate) fn write_sized_pdf(path, width_pt, height_pt)` beside
  `write_one_page_pdf`. It writes a hand-built 7-bit-ASCII one-page PDF with the same
  object layout as `diagrams_blank.pdf`, with a computed `xref` table and
  `/MediaBox [0 0 w h]`. The content stream draws two Helvetica 10pt words: `diagramtop`
  near the top edge and `diagrambottom` about 6pt above the bottom edge. pdftotext can
  then locate both edges and measure the scale on the rendered page. Don't use lopdf for
  this; the existing comment explains that XeLaTeX rejects lopdf's empty-stream output.

### 3. `src/native/highlights_ref/diagrams.lua`

In `CodeBlock`, the success branch returns
`pandoc.Para{ pandoc.RawInline("latex", "\\BobFitBegin{}"), pandoc.Image({}, pdf, "", pandoc.Attr(ident)), pandoc.RawInline("latex", "\\BobFitEnd{}") }`.
Update the header comment to name the wrapper and `diagrams.tex`. The error and fallback
branches are unchanged.

> [!decision] image_scope = all_images Leave `diagrams.lua` returning a bare
> `Para{Image}`. Add the wrapping instead to the shared `PANDOC_CODE_BREAK_FILTER` in
> `create.rs`, which runs right after `diagrams.lua` on both routes. Add a `Para`
> handler that wraps any `Para` whose content is exactly one `Image` in the same
> delimiters (add it to the existing `{Code = Code, Div = Div}` table). Every standalone
> image, including diagrams, is then budgeted. Add one generic-route test with a tall
> plain `![](tall.pdf)` after a heading. Also add a test that an image inside a sentence
> (not standalone) is not wrapped. Update the docs wording to say "figures and images".

### 4. Header wiring (both Markdown routes)

- `create.rs::render_temp_pdf`: the `header-includes=` value becomes
  `PANDOC_HEADER_INCLUDES`, `return_links::HEADER_INCLUDES`, and
  `diagrams::HEADER_INCLUDES`, newline-joined.
- `note_snapshot/profile.rs::render_note_pdf`: the same three-part `header-includes=`
  value.
- `render_tex.rs::header_packages_are_all_declared`: add `diagrams::HEADER_INCLUDES` to
  the scanned headers. It has no `\usepackage` today; this guards future edits.

### 5. `src/native/highlights_ref/note_snapshot/note.lua`

In `process_blocks`, `standalone_image` must also be true for the wrapped shape. That is
a `Para` with exactly three inlines: `RawInline latex "\BobFitBegin{}"`, `Image`,
`RawInline latex "\BobFitEnd{}"`. Snapshot diagrams then stay centered
(`\begin{center}`) and framed (`process_inlines` still wraps the Image in
`\BobImageBegin{}`/`\BobImageEnd{}` inside the fit box). Match those exact strings;
don't loosen the check to "ignore all raw inlines".
`diagrams_swap_snapshot_centers_image_and_drops_source` must keep passing unchanged.

### 6. Docs: `docs/highlights-create.md`, "Mermaid diagrams" section

Replace "Figures scale down to fit the page and are never rearranged or split." The new
text says figures scale down to fit the text width and at most 80% (per decision) of the
text height. That leaves room on the same page for the heading, return links, and
opening caption line, so a tall diagram no longer leaves a blank page or runs into the
margin. Figures are still never rearranged or split, and short figures print at natural
size. Say that the existing 50% "labels may be small" warning uses the same budget. Keep
the note that existing library PDFs are not backfilled. This is a behavior-contract
page; run `just docs-check`. No CLI help or flag changes.

## Regression tests (layout, not just exit status)

All failing reproductions exited 0, so these tests must inspect the rendered pages. Gate
them like the existing diagram PDF tests (`skip_pandoc_xelatex` /
`skip_real_diagram_render`; pdftotext required). Build them on the existing helpers:
`mermaid_plan`, `filters_with_optional_diagrams`, `collected_flow_id`, `write_results`,
and the per-page `pdftotext -f N -l N` loop in
`return_link_xelatex_render_pairs_pills_with_destinations`. Feed each test a fake
`results.json` that points at `write_sized_pdf(…, 514.5, 714.75)`, so no browser is
needed.

Generic route (`create.rs` tests, via `render_temp_pdf`):

1. **Heading + return link + tall diagram + caption.** The source has an early paragraph
   linking `[model](#data-model-and-ownership)`, then `### Data model and ownership`, a
   tall Mermaid fence, and a caption paragraph. Assert:
   - No body-empty page. Every page's pdftotext text, minus bare page-number tokens, is
     non-empty.
   - The heading text, the `↩` pill, `diagramtop`, and `diagrambottom` are all on the
     same page.
   - With `pdftotext -bbox`, `diagrambottom`'s `yMax` ≤
     `LETTER_HEIGHT_PT - GENERIC_MARGIN_IN * PT_PER_IN` (730.8pt, the bottom-margin
     line).
   - The vertical distance between the two labels' baselines ≈ the fixture's baseline
     distance × `FIT_HEIGHT_BUDGET * GENERIC_DIAGRAM_TEXT_HEIGHT_PT / 714.75`, within a
     few points. This proves the diagram is scaled to the budget and no further.
2. **Same document with frontmatter `bob-return-links: false`.** Same no-blank-page and
   same-page assertions.
3. **Anchored figure div.** `<div id="figure-tall">` wraps the fence, two inbound links
   point at it, and enough body text comes first that the div starts low on a page.
   Assert no body-empty page, pills and `diagramtop` on the same page, `diagrambottom`
   above the margin line, and every return-link pill still resolves to its named
   destination (reuse `named_dest_pages` / `goto_dest_names`).
4. **Short diagram unchanged.** The existing 200×80 fixture after a heading is not
   scaled: its label's bbox height matches the 12pt Helvetica label at natural size
   within tolerance. Keep `diagrams_swap_pdf_replaces_source_without_figure_caption`
   passing as is.

Snapshot route (`note_snapshot/profile.rs` tests, via `render_note_pdf`):

5. **Heading + inbound link + tall diagram.** Assert:
   - The scratch `note.tex` contains `\BobFitBegin{}`, `\BobImageBegin{}`, and
     `\begin{center}`.
   - `note.log` has no `Overfull \vbox`.
   - The heading text and `diagramtop` share a page.
   - No page is body-empty. Ignore the footer: the title and the `n / N` tokens.

Red/green: before applying the TeX/Lua change, confirm that test 1 (or 3) fails on the
current code with a body-empty page or `diagrambottom` below 730.8pt. Then confirm it
passes after the change. Record the before/after in the commit body.

## Verification

- `cargo test --locked` for the touched modules, then the full suite. The real
  pandoc/xelatex tests must run on this host, not skip. Check the test output for
  "skipping".
- `just lint` (or the repo's standard lint target) and `just docs-check`.
- Manual end-to-end check with the real Mermaid renderer, writing only to scratch.
  Create a scratch directory with `doc.md` (heading + inbound link + a tall
  `flowchart TB` with ~14 stacked nodes + caption) and an empty `out/` subdirectory. Run
  `cargo run -- ref create <scratch>/doc.md -P bob -o <scratch>/out/doc.pdf -d` first;
  it must end with `writes: none`, and the output must live outside the vault. Then run
  the same command without `-d`. Inspect the pages with `pdftotext`/`pdfinfo`: no
  body-empty page, and the heading shares a page with the diagram. Do not pass the
  source's own directory as `-o`'s parent: bob refuses because the `.md` would look like
  the sidecar.
- Commit as `fix(ref): …`. Do not edit `Cargo.toml`'s version or `CHANGELOG.md` (the
  daily release train owns them).

## Out of scope

- Re-rendering existing library PDFs (e.g. `~/bob/lib/chat/pomodoro_theme_catalog.pdf`).
  Bryan can recreate that reference after the release; don't touch the vault.
- The research swarm's Mermaid validation (`sase-research-artifacts`, a different
  project). Its size check should eventually use the same composed-page budget, but that
  change belongs in that repo.
- Splitting oversized diagrams, landscape pages, or changing heading/return-link
  keep-together rules.
