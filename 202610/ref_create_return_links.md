---
tier: epic
title: Paired return links for bob ref create Markdown PDFs
goal: "Every same-document link in a Markdown PDF rendered by `bob ref create` carries a
  small raised letter tag, and its target shows a matching `↩ p. N` return pill that
  jumps back to the passage the reader left. Link targets resolve robustly, dead links
  are visible instead of silent, and the tags and pills never leak into highlights
  synced into the vault.

  "
phases:
  - id: render
    title: Render paired return links in Markdown PDFs
    depends_on: []
    size: medium
    description:
      "render: add a second pandoc Lua filter plus TeX macros that resolve, tag, and
      pair every eligible same-document link with a return-pill row at its target, unify
      the link ink, report pairing counts and dead-link warnings from bob, and cover it
      with filter, XeLaTeX, and CLI tests plus docs."
  - id: export
    title: Keep return-link glyphs out of synced highlights
    depends_on:
      - render
    size: medium
    description:
      "export: stamp a `return_links: true` marker key on PDFs that actually carry
      return links, register it as a standard synced field, and strip tag glyphs and
      pill text from highlight text when `bob ref sync` renders the note region for
      those PDFs."
proposed_by: bbugyi200.athena.research.3x.linker.w0
create_time: 2026-10-07 14:27:21
status: wip
---

- **PROMPT:**
  [prompts/202610/ref_create_return_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_create_return_links.md)

# Plan: Paired return links for `bob ref create` Markdown PDFs

## Context

Bryan reads bob-rendered Markdown research reports in Highlights (Mac and iPad). The
reports carry 20–33 same-document links each (`[What I verified](#what-i-verified)`).
Forward jumps already work, but there is no way back except a viewer Back command that
does not exist on every device, and that does nothing on paper.

The design comes from the research report
`research:202610/ref_create_pdf_return_links/ref_create_pdf_return_links.md`. Read it
with `sase artifact read` before starting. Bryan accepted all of its recommendations.
Its working prototype (`return_links.lua`, `return_links.tex`) lives in the
`bob-cli--research` sidecar under
`202610/ref_create_pdf_return_links/ref_create_pdf_return_links__final_assets/`. Open it
with `sase repo open bob-cli--research -r "<reason>"`; `sase artifact read` filters out
non-Markdown files. Treat the prototype as a starting point, not a spec. This plan fixes
several prototype gaps, listed under
[Refinements over the prototype](#refinements-over-the-prototype).

Today's Markdown route: `create_markdown_route` → `render_temp_pdf` in
`src/native/highlights_ref/create.rs` runs pandoc with:

- `--toc --toc-depth=3 --number-sections --pdf-engine=xelatex`;
- one Lua filter, `PANDOC_CODE_BREAK_FILTER` (inline-code breaks and listen cards),
  written to scratch as `filter.lua`;
- the `PANDOC_HEADER_INCLUDES` preamble passed as `-V header-includes=…`;
- `-V colorlinks=true` with pandoc's default Maroon internal links and Blue URLs.

On success pandoc's stderr is thrown away. `stamp_and_install` (lopdf) keeps link
annotations, named destinations, and outlines; the research verified this.

Verified on athena while planning:

- Pandoc 3.1.11.1 and XeTeX (TeX Live 2025) are installed.
- DejaVu Sans Bold has all 23 tag glyphs and `↩`.
- `pandoc.json` exists in the filter Lua API, but it encodes an empty Lua table as `{}`,
  not `[]`.
- `-M key=false` and frontmatter `key: false` both reach the filter as Lua boolean
  `false`.
- A filter can write a file whose path arrives through `-M`.
- The macro set in [Visual specification](#visual-specification) compiles with bob's
  exact pandoc arguments. It renders a 21-pill wrapped row, a run-in level-4 heading
  target, a Div target, and an inline Span target cleanly.

## What the reader experiences

1. On page 2 the reader sees "…grows about 2–3 tasks a day (What I verifiedᵈ, row 12)".
   The phrase and its small bold ᵈ form one link, drawn in the document's single link
   ink.
2. Tapping it lands on **1.4 What I verified**. Directly under the heading is a row of
   rounded pills: `↩ p. 2ᵈ  ↩ p. 4ⁱ  ↩ p. 8ⁿ …`. One pill carries the reader's ᵈ, and
   "p. 2" tells them where they were even if they never noticed the tag.
3. Tapping `↩ p. 2ᵈ` lands about a line and a half above the original sentence. Zoom and
   horizontal position stay as they were, and the ᵈ marks the spot.
4. Before tapping anything, the row also shows that this section is cited from pages 2,
   4, and 8.

The heading text, TOC, and PDF bookmarks never change. Links that cannot resolve render
as plain text, and `bob ref create` warns about them.

## Design decisions

These are adopted from the research and are binding for both phases:

- **One occurrence, one tag.** Every eligible link occurrence gets its own sequential
  letter tag in reading order. The alphabet is `abcdefghijkmnprstuvwxyz`: no `l` (reads
  as 1), no `o` (reads as a degree sign), and no `q` (no modifier glyph). Tags count in
  bijective base 23: `a…z`, then `aa, ab, …`. Every eligible link is tagged, even
  one-to-one targets.
- **Tags are Unicode modifier letters** drawn right after the link text, inside the
  link's clickable area, in DejaVu Sans Bold at 1.3× size and muted link blue. Never
  splice ASCII IDs into the prose. PDFKit ignores `/ActualText`, so the glyphs are
  stripped at sync time instead (phase `export`).
- **One pill row per target.** Each pill is `↩ p. N` plus the tag, in source reading
  order. Rows wrap and are never capped. Placement depends on the target: a row under a
  heading, the first block inside a Div, or inline right after a Span.
- **Return landing.** Each return lands on a raised named destination
  `[@thispage /XYZ null @ypos null]`, about 1.6 lines above the source line, with x and
  zoom left unspecified.
- **Resolution order:** exact pandoc id, then the percent-decoded id, then a
  GitHub-style alias computed by pandoc's own `gfm` reader. Only unique aliases count.
  Never fuzz by case or heading text.
- **Dead links render as plain text** (formatting kept, no link), and bob prints a
  warning for each.
- **One link ink, `#2F5E96`,** for internal links, URLs, file links, citations, tags,
  and pills. The TOC stays black.
- **Escape hatch:** source frontmatter `bob-return-links: false` makes the filter return
  the document unchanged. There are no new CLI flags.
- **Code lives in a separate second filter,** `return_links.lua`, run after the existing
  code-break/listen filter. By then listen cards are raw LaTeX, so they are never
  decorated, and `create.rs` does not grow a second embedded filter.
- **Sync strips glyphs** only for PDFs flagged as carrying return links, and only at
  render time, so block IDs stay stable.

### Refinements over the prototype

This plan makes these decisions on top of the research. Implement them as stated.

1. **GitHub alias via a write-then-read round trip.** Compute the alias as the
   identifier of
   `pandoc.read(pandoc.write(pandoc.Pandoc({pandoc.Header(1, h.content)}), "gfm"), "gfm").blocks[1]`.
   Do not use `pandoc.utils.stringify`: it drops code-span markup, so
   `` `__init__` method`` re-parses as bold and yields `init-method` instead of GitHub's
   `__init__-method` (reproduced on pandoc 3.1.11.1).
2. **Tag only return-capable targets.** Return-capable targets are a Header, a Div, or a
   Span that is not inside a heading, caption, or table head or foot row. Links to other
   id-bearing elements (CodeBlock, Table, Figure, Image, Code, Link, or a Span in a skip
   context) still resolve and stay working forward links, but they are untagged. The
   prototype tagged them with no pill to return to; with this rule every tag has exactly
   one pill.
3. **Duplicate explicit ids.** When an id that some link resolves to is defined more
   than once, its links keep working but stay untagged, and the report lists the id. A
   pill row is never attached twice.
4. **Table foot rows are a skip context**, like table head rows. Longtable can repeat
   them, which would duplicate anchors.
5. **Structured report file, not stderr lines.** Bob passes
   `-M bob-return-links-report=<scratch>/return-links.json`, and the filter writes one
   JSON object there. pandoc and XeLaTeX warnings share stderr, and Lua `%q` quoting is
   awkward to parse. The filter writes nothing to stderr.
6. **Wrapped pill rows get breathing room** (`\lineskiplimit=3pt\lineskip=3pt`). Inline
   Span rows use a compact pill that never increases a 10 pt line's height.
7. **The sync flag is precise.** The marker key `return_links: true` is written only
   when the render actually paired at least one link, rather than on every Markdown
   render. A document without return links keeps its legitimate modifier letters
   untouched. The key is a standard synced field that mirrors `captured`, so it never
   needs `highlights_marker_fields`.

### Eligibility matrix

| Link location or target                                                                                                                                                    | Result                                                                                            |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Paragraphs, plain blocks, line blocks, definition lists, block quotes, Div content, list items (not nav lists), table **body** cells, figure content, footnotes            | **Tagged** + paired (when the target is return-capable)                                           |
| Headings, figure captions, table captions, table head rows (including intermediate body heads), table foot rows, hand-written TOC lists (`is_nav_list` from the prototype) | **Untagged**: target rewritten to the canonical id; still a working forward link                  |
| Target is a CodeBlock, Table, Figure, Image, Code, Link, Span in a skip context, or a duplicated id                                                                        | **Untagged**: working forward link                                                                |
| Listen cards (already raw LaTeX) and metadata (title, abstract)                                                                                                            | Untouched                                                                                         |
| Unresolvable target (`#`, missing, or ambiguous alias)                                                                                                                     | **Dead**: plain text (a Span of the link content) plus a report record. Applies in every context. |
| Non-`#` links                                                                                                                                                              | Untouched                                                                                         |

## Visual specification

The tag is a modifier-letter glyph in DejaVu Sans Bold at 1.3× the surrounding size,
colour `#5C7FA8`, kerned 0.04 em from the text, with no space before it, so it never
wraps away from the word.

A block pill is a TikZ rounded rectangle:

- 2.4 pt corner radius, fill `#EEF3FA`, 0.4 pt rule `#C9D7EA`, sans `\footnotesize`;
- `↩` in link ink, `p. N` in muted gray `#6E7781` (`\pageref*` of the source anchor),
  then the tag at 1.3× in link ink;
- the whole pill is the link.

A block row sits under its heading (or as the first block of a Div):

- left-aligned and ragged right, with 0.35 em between pills, allowed to wrap with 3 pt
  between lines;
- `\nopagebreak[4]` on both sides, then `\@afterheading`, so the first paragraph stays
  with the row.

Before a heading that has pills **and** is directly followed by a Table, emit
`\needspace{16\baselineskip}`. Longtable forces a page break when its header plus first
row do not fit, which strands the heading and its row at the page bottom.

An inline row uses a compact pill with smaller padding, so the paragraph's line spacing
is undisturbed.

Ship this macro file verbatim as `src/native/highlights_ref/return_links.tex`. Adjust it
only if visual QA shows a real defect, and only in the spirit of this specification:

```tex
% bob ref create: paired return links (see docs/highlights-create.md).
\usepackage{needspace}
\usepackage{tikz}
\definecolor{BobLinkInk}{HTML}{2F5E96}
\definecolor{BobTagInk}{HTML}{5C7FA8}
\definecolor{BobPillFill}{HTML}{EEF3FA}
\definecolor{BobPillRule}{HTML}{C9D7EA}
\definecolor{BobMuted}{HTML}{6E7781}
\makeatletter
\newcommand{\BobReturnAnchor}[1]{\raisebox{1.6\baselineskip}[0pt][0pt]{\special{pdf:dest (#1) [@thispage /XYZ null @ypos null]}}\label{#1}}
\newcommand{\BobTag}[1]{{\sffamily\bfseries\fontsize{1.3em}{0pt}\selectfont\color{BobTagInk}\kern0.04em #1}}
\newcommand{\BobPillBody}[2]{{\color{BobLinkInk}↩}\hspace{0.3em}{\color{BobMuted}p.\,\pageref*{#1}}{\color{BobLinkInk}\bfseries\fontsize{1.3em}{0pt}\selectfont\,#2}}
\newcommand{\BobBack}[2]{\hyperlink{#1}{\tikz[baseline=(p.base)]\node[fill=BobPillFill,draw=BobPillRule,line width=0.4pt,rounded corners=2.4pt,inner xsep=3.4pt,inner ysep=1.8pt,text height=1.75ex,text depth=0.45ex](p){\BobPillBody{#1}{#2}};}}
\newcommand{\BobBackCompact}[2]{\hyperlink{#1}{\tikz[baseline=(p.base)]\node[fill=BobPillFill,draw=BobPillRule,line width=0.4pt,rounded corners=2pt,inner xsep=2.6pt,inner ysep=0.5pt,text height=1.5ex,text depth=0.35ex](p){\BobPillBody{#1}{#2}};}}
\newcommand{\BobBackSep}{\hspace{0.35em}\allowbreak}
\newcommand{\BobBacklinks}[1]{\par\nopagebreak[4]\vspace{0.15\baselineskip}\noindent{\raggedright\sffamily\footnotesize\lineskiplimit=3pt\lineskip=3pt #1\par}\nopagebreak[4]\addvspace{0.55\baselineskip}\@afterheading}
\newcommand{\BobBackInline}[1]{\,{\sffamily\footnotesize #1}}
\makeatother
```

Block rows are `\BobBacklinks{\BobBack{a1}{g1}\BobBackSep{}\BobBack{a2}{g2}…}`. Inline
rows are `\BobBackInline{\BobBackCompact{a1}{g1}\BobBackSep{}…}`. Level-4+ headings are
run-in in pandoc's article template, so their row follows the heading text on the same
line. That looked right in the planning probe; keep it.

## Render paired return links in Markdown PDFs

### Files

- New `src/native/highlights_ref/return_links.lua`: the filter.
- New `src/native/highlights_ref/return_links.tex`: the macros above.
- New `src/native/highlights_ref/return_links.rs`, registered in
  `src/native/highlights_ref/mod.rs`. It holds:
  - `FILTER` (`include_str!("return_links.lua")`);
  - `HEADER_INCLUDES` (`include_str!("return_links.tex")`);
  - `REPORT_METADATA_KEY` (`"bob-return-links-report"`);
  - the report types, the reader, and the summary and warning formatting, plus their
    unit tests and the pandoc-gated filter tests. The existing `include_str!` assets
    under `src/native/completion/adapters/` are the precedent.
- `src/native/highlights_ref/create.rs`: wiring only.
- `docs/highlights-create.md` and the `create` `after_help` text.
- Tests in `return_links.rs`, the `create.rs` test module, and
  `tests/cli/highlights/create.rs`.

### Filter algorithm (`return_links.lua`)

There is a single `Pandoc(doc)` entry point. It does nothing unless `FORMAT` matches
`latex`. The report path is
`pandoc.utils.stringify(doc.meta["bob-return-links-report"])` when that key is set;
otherwise the filter writes no report.

0. **Opt-out.** If `doc.meta["bob-return-links"]` is boolean `false`, or stringifies to
   `"false"`, write `{"version":1,"enabled":false}` and return the doc unchanged.
1. **Index.** Walk the blocks with context and record, for every identifier:
   - its definition count;
   - whether it is return-capable (Refinement 2): Header, Div, or a Span outside
     heading, caption, and table head/foot contexts. Every other id-bearing element is
     resolvable but not capable.

   For each Header in document order, compute its GitHub alias (Refinement 1). Number
   duplicates the way GitHub does: the first stays bare, the next gets `-1`, then `-2`.
   An alias claimed by two different header ids is ambiguous.

   Choose the anchor prefix: `bob:ret:`, unless some indexed id starts with it, in which
   case use the first of `bob:ret1:`, `bob:ret2:`, … that no id starts with.

2. **Resolve** `target` (leading `#` stripped) to an id, a route, and a reason:
   - empty → dead (`missing`);
   - an existing id → that id (`exact`);
   - the percent-decoded (`%XX`) form is an existing id → that id (`decoded`);
   - an unambiguous alias for the decoded form → its header id (`github`);
   - an ambiguous alias → dead (`ambiguous`);
   - otherwise → dead (`missing`).

   Real ids always beat aliases.

3. **Tag** in reading order. Recurse like the prototype's `walk_blocks`, with the skip
   contexts from the [eligibility matrix](#eligibility-matrix), including table foot
   rows. Footnote links are numbered where their note mark sits. For each Link whose
   target starts with `#`:
   - Dead: replace it with `pandoc.Span(link.content)` and record
     `{target, text, reason}`. `text` is the stringified link text, whitespace-collapsed
     and cut to 80 characters with `…`.
   - Resolved: set `link.target = "#" .. id`, and count it in `github` when the route
     was `github`.
   - In a skip context, or when the id is not capable or is duplicated: count it in
     `untagged` and keep the link.
   - Otherwise: `n = n + 1`, anchor = prefix .. n, tag = `letters(n)`. Append to the
     id's inbound list, remembering the first-seen target order. Append
     `RawInline("latex", "\\BobTag{" .. glyphs(tag) .. "}")` to `link.content`, and
     return `{RawInline("latex", "\\BobReturnAnchor{" .. anchor .. "}"), link}`.

   Pills are raw LaTeX, so generated return links are never visible to this pass.

4. **Attach** exactly one row per capable id with inbound links, at its single defining
   element:
   - Header: insert `RawBlock("latex", "\\BobBacklinks{…}")` right after it. Insert
     `RawBlock("latex", "\\needspace{16\\baselineskip}")` right before it when the next
     block in the same list is a Table.
   - Div: insert the same row as the Div's first block.
   - Span: return `{span, RawInline("latex", "\\BobBackInline{…}")}` using
     `\BobBackCompact`.
5. **Report.** Write the report (see below) with `pandoc.json.encode`, omitting empty
   lists because `pandoc.json` encodes them as `{}`.

Keep the prototype's `letters`, `MOD` glyph table, `glyphs`, and `is_nav_list` logic.
Keep author text out of raw LaTeX: anchors and glyph strings are generated, and ids come
from pandoc.

### Report contract (version 1)

```json
{
  "version": 1,
  "enabled": true,
  "prefix": "bob:ret:",
  "paired": 27,
  "targets": 11,
  "untagged": 2,
  "github": 4,
  "dead": [{ "target": "#nope", "text": "the old ranking", "reason": "missing" }],
  "duplicates": [{ "id": "setup", "count": 2 }]
}
```

- `paired` counts tagged links.
- `targets` counts ids that received a row.
- `github` counts resolved links (tagged or not) that needed the GitHub alias.
- `duplicates` lists only duplicated ids that at least one link resolved to.

In Rust, deserialize with `serde` and use `#[serde(default)]` for every field except
`version`; `enabled` defaults to `true`. Any `version` other than 1 is invalid.

### Rust wiring (`create.rs`)

- `create_markdown_route` writes `return_links::FILTER` to `<scratch>/return-links.lua`
  beside `filter.lua`, and passes both paths plus `<scratch>/return-links.json` to
  `render_temp_pdf`. Group them in a small struct rather than growing the positional
  arguments.
- `render_temp_pdf` adds these arguments:
  - `--lua-filter <return-links.lua>` **after** the existing filter;
  - `-V linkcolor=BobLinkInk -V urlcolor=BobLinkInk -V filecolor=BobLinkInk -V citecolor=BobLinkInk`;
  - `header-includes = PANDOC_HEADER_INCLUDES + "\n" + return_links::HEADER_INCLUDES`;
  - `--metadata bob-return-links-report=<path>`.

  On success it reads the report and returns `ReportOutcome`:
  - `Missing`: no file. This happens with test-double pandoc wrappers; stay silent.
  - `Invalid(reason)`: unreadable, bad JSON, or the wrong version.
  - `Report(report)`.

  pandoc failure handling is unchanged.

- Both render branches (with and without `--listen`) carry the outcome to the success
  output. Print the `links:` line right after `pages:`, then the warnings on stderr with
  `styler.warning_prefix()`, then `print_next_step`. Dry runs never render, so they
  print no `links:` line.
- `links:` values:

  | Outcome                                     | Line                                                                                                                                |
  | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
  | `Missing` or `Invalid`                      | no `links:` line (`Invalid` adds one warning: `return-link report unreadable: <reason>`)                                            |
  | `enabled: false`                            | `links: off (bob-return-links: false)`                                                                                              |
  | No `#` links at all (all counts 0, no dead) | `links: none`                                                                                                                       |
  | Otherwise                                   | `links: 27 paired (11 targets)`, plus ` · 4 via GitHub-style slugs`, ` · 2 untagged`, ` · 1 dead` when nonzero, singular `1 target` |

- Warnings, one per distinct item:
  - `local link #nope ("the old ranking") has no target; rendered as plain text`, with
    ` (3 links)` appended when the same target was dead more than once;
  - `local link #setup ("setup") matches more than one heading by GitHub-style slug; rendered as plain text; give the heading an explicit {#id}`;
  - `anchor #setup is defined 2 times; its links still jump but get no return pills`.

Example success output:

```text
✓ created Highlights-ready PDF
source: /…/retire_now_sticky_lanes_ledger_today.md
pdf: /…/xlib/chat/retire_now_sticky_lanes_ledger_today.pdf
audio: none
title: …
status: ready
parent: obsidian_ref
pages: 17
links: 27 paired (11 targets) · 4 via GitHub-style slugs
warning: local link #nope ("the old ranking") has no target; rendered as plain text
next: bob ref scan
```

### Tests

Filter → LaTeX tests in `return_links.rs` follow the style of the existing
`code_break_filter_*` tests: skip when `pandoc_command()` is `None`, run `--to=latex`
with **both** filters in bob's order, and pass a report path through `-M`. Each test
asserts on the LaTeX and on the parsed report.

1. Pairing and order: one heading targeted from a paragraph, a list item, a table body
   cell, and a footnote produces `\BobReturnAnchor{bob:ret:1..4}` before each link and
   `\BobTag{ᵃ}`…`\BobTag{ᵈ}` inside each `\hyperref`. Exactly one `\BobBacklinks` sits
   right after that `\section`, with pills in source order. The report has `paired=4`,
   `targets=1`.
2. Skip contexts: links in a heading, a table caption, a table head row, a figure
   caption, and a nav list are rewritten to the canonical id, have no tag or anchor, and
   count in `untagged`.
3. Resolution:
   - a `%20`-encoded id;
   - `#7-ranked-recommendations` for `# 7. Ranked recommendations`;
   - `#__init__-method` for ``# `__init__` method``;
   - GitHub `-1` duplicate numbering;
   - an ambiguous alias becomes dead with reason `ambiguous`;
   - a real id beats an alias.
4. A dead link becomes plain text (no `\hyperref`), its emphasis is kept, and the report
   has a `missing` record.
5. Non-capable targets: links to a CodeBlock id and a Table id keep `\hyperref`, are
   untagged, and get no row. A duplicated explicit id appears in `duplicates`, is
   untagged, and gets no row. A link to a Span inside a heading is untagged, and the
   heading gets no inline pills.
6. Placement: a Div target's row is its first block, and a Span target is followed by
   `\BobBackInline{\BobBackCompact…}`. A heading target followed directly by a table
   gets `\needspace{16\baselineskip}` before the `\section`.
7. Opt-out: with frontmatter `bob-return-links: false`, the LaTeX is byte-identical to
   running only the code-break filter, and the report is `enabled=false`.
8. A listen card with a `#` link is untouched (no `\BobTag` inside `\BobListenCard`).
9. Prefix collision: an explicit `{#bob:ret:1}` makes anchors use `bob:ret1:`.
10. Tag sequence: with 24 links, the 23rd tag is `ᶻ` and the 24th is `ᵃᵃ`.
11. Invariant on a mixed fixture: the count of `\BobReturnAnchor{` equals the count of
    `\BobBack{` plus `\BobBackCompact{`, and every anchor name appears in exactly one
    pill.

Rust unit tests in `return_links.rs`:

- report parsing: full report, missing lists, `enabled=false`, a wrong version, and bad
  JSON;
- every `links:` line variant;
- every warning variant, including the repeated-dead ` (N links)` form.

XeLaTeX integration in the `create.rs` test module, next to
`listen_card_xelatex_render_uses_only_existing_packages`. Skip unless pandoc and xelatex
exist; also skip the page checks when `pdftotext` is missing.

- Render a multi-page fixture through `render_temp_pdf` and `stamp_and_install`. It has:
  - targets with fan-in 0, 1, 8, and 21 (the last wraps);
  - a link long enough to wrap across lines;
  - a level-4 heading target, a Div target, and a Span target;
  - a pill-bearing heading that would land near a page bottom followed by a table.
- Assert with lopdf on the **stamped** PDF:
  - every Link annotation whose GoTo `/D` starts with the prefix resolves to a named
    destination in the catalog's `/Names /Dests` tree;
  - the number of prefixed destinations equals `paired`;
  - no outline title contains a tag glyph or `↩`.
- Assert with `pdftotext -layout` per page:
  - each pill's printed `p. N` + tag equals the 1-based page of that tag's destination;
  - the near-bottom heading and the first cell of its table sit on the same page.
- Keep the existing listen-card render test passing. It now proves `tikz` and
  `needspace` load alongside the listen card.

CLI tests in `tests/cli/highlights/create.rs`:

- Gated on pandoc and xelatex: a small fixture with two links to one heading and one
  dead link prints `links: 2 paired (1 target) · 1 dead` on stdout and the dead-link
  warning on stderr.
- The existing fake-pandoc tests still pass, with no `links:` line and no new stderr.

### Visual QA

Before finishing, render the realistic multi-page fixture with
`bob ref create <fixture> -o <tmpdir>/return_links.pdf` (an external target, so no vault
writes). Rasterize it with `pdftoppm -r 110` and look at the pages:

- tag weight and kern after plain, code, and parenthesised link text;
- tags in table cells;
- a fan-in-8 row and the wrapped 21-pill row;
- the run-in level-4 row;
- the inline Span pill (line spacing unchanged);
- the near-bottom heading.

Optionally also render a real report opened through `sase repo open bob-cli--research`.

### Docs

- `docs/highlights-create.md`: add a `## Local links and return pills` section after
  `## Targets`. It should cover:
  - the reader experience in brief;
  - the eligibility matrix;
  - the resolution order, and the advice to give often-linked headings an explicit
    `{#id}`;
  - dead-link and duplicate warnings, and the `links:` line;
  - the `bob-return-links: false` frontmatter opt-out;
  - the single link ink;
  - that existing PDFs gain the feature only when re-rendered.
- `create` `after_help`, in the `Output:` paragraph, one sentence: same-document `#`
  links get a raised letter tag and a matching `↩ p. N` return pill under their target,
  dead ones render as plain text with a warning, and frontmatter
  `bob-return-links: false` turns this off.

### Acceptance

`just all` passes (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
`cargo test`) on athena with pandoc and xelatex present. A real render shows tags,
pills, and the `links:` line. No new CLI options are added.

## Keep return-link glyphs out of synced highlights

Highlights exports exactly what is drawn, because PDFKit ignores `/ActualText`. A
highlight across a tagged link therefore exports as `What I verifiedᵈ, row 12`, and a
highlighted pill row exports as `↩ p. 2ᵈ ↩ p. 4ⁱ`. This phase marks the PDFs that carry
return links and cleans their highlight text when `bob ref sync` renders the note's
generated region.

### Marker key `return_links`

- Add `FIELD_RETURN_LINKS = "return_links"` in `src/native/highlights_ref/mod.rs`. Add
  it to `COMMON_USER_FIELDS` right after `captured`, and to `MARKER_EXTRA_ORDER` in
  `stamp.rs` right after `captured`. This mirrors how the `captured` provenance key
  works: it rides in the page-1 marker (the one place Highlights is proven to preserve
  data), round-trips into note frontmatter as a standard field, and never needs
  `highlights_marker_fields`.
- In `create_markdown_route`, after a successful render whose report is `enabled` with
  `paired >= 1`, recompose the marker with the extra `("return_links", "true")` and
  stamp that marker. Factor the Markdown marker composition from `plan_markdown` into a
  helper both call. Otherwise the marker is unchanged, including for `Missing` and
  `Invalid` outcomes. The marker line reads `- return_links: true`, and frontmatter
  reads `return_links: true`.
- Dry runs still preview the planned marker without the key. Document that the key is
  added only after a render pairs at least one link.
- The flag is "on" when the synced projection's `return_links` value is `Bool(true)` or
  the string `"true"`. Any other value, or no value, is off. Confirm that marker and
  frontmatter validation accept the boolean for a standard field. If the user deletes
  the key from frontmatter or the marker, the normal three-way sync applies and
  stripping stops.

### Strip rules

Add `pub(super) const TAG_GLYPHS: [char; 23]` to `return_links.rs`, in alphabet order:
U+1D43 U+1D47 U+1D9C U+1D48 U+1D49 U+1DA0 U+1D4D U+02B0 U+2071 U+02B2 U+1D4F U+1D50
U+207F U+1D56 U+02B3 U+02E2 U+1D57 U+1D58 U+1D5B U+02B7 U+02E3 U+02B8 U+1DBB. Add a
drift test asserting that the set equals the non-ASCII glyphs of the `MOD` table in
`return_links::FILTER`.

Add `pub(super) fn strip_return_link_glyphs(text: &str) -> String` to
`src/native/highlights_ref/text.rs`. Use the existing `regex` dependency or a small
scanner. Apply in this order:

1. **Pill fragments:** `↩`, optional whitespace, `p.`, optional whitespace, one or more
   digits, optional whitespace, one or more `TAG_GLYPHS`. Remove each fragment together
   with one adjacent whitespace run, so the neighbouring words stay single-spaced.
2. **Tag runs:** a maximal run of `TAG_GLYPHS` that directly follows a non-whitespace
   character, or starts the text, is removed.

Leave everything else to the existing `beautify_annotation_text` normalization.

| Input (flag on)                                          | Output                                                  |
| -------------------------------------------------------- | ------------------------------------------------------- |
| `grows about 2–3 tasks a day (What I verifiedᵈ, row 12)` | `grows about 2–3 tasks a day (What I verified, row 12)` |
| `see the rankingᵃᵈ.`                                     | `see the ranking.`                                      |
| `↩ p. 2ᵈ ↩ p. 4ⁱ ↩ p. 13ⁿ`                               | empty                                                   |
| `↩ p.2 ᵈ`                                                | empty                                                   |
| `1.4 What I verified ↩ p. 2ᵈ Text continues`             | `1.4 What I verified Text continues`                    |
| `ᵈ, row 12` (highlight started on the tag)               | `, row 12`                                              |
| `aspirated pʰ` with the **flag off**                     | unchanged                                               |

### Threading

`sync.rs` already holds `synced_projection` before it calls `render_sidecar_highlights`.
Derive the flag from it and pass it down through `render_sidecar_highlights` →
`render_annotation_block` (a small options value or a bool; match the surrounding
style). Apply the strip **only** to `SidecarAnnotationKind::Highlight` text, before
`beautify_annotation_text`. Comments and standalone notes are the user's own words and
stay untouched.

Block IDs (`annotation_block_id`), annotation-task sources, and `[h:: …]` markers keep
using the raw sidecar text, exactly like the existing render-only cleanup. Update every
existing caller in `tests/sidecar.rs` and `tests/region.rs`.

### Tests

- Unit tests for `strip_return_link_glyphs` covering every row of the table, plus the
  `TAG_GLYPHS` drift test.
- Sidecar render tests in the style of `tests/sidecar.rs`:
  - with the flag on, a highlight containing a tag and a pill row renders clean;
  - the block ID is identical with the flag on and off;
  - a comment containing `ᵈ` is untouched;
  - with the flag off, `pʰ` survives.
- Marker tests:
  - `compose_marker` places `return_links: true` after `captured`;
  - it parses back as a standard field;
  - it renders as `return_links: true` in note frontmatter with no
    `highlights_marker_fields` entry.
- Create tests:
  - a pandoc+xelatex-gated render with at least one paired link stamps
    `- return_links: true` (check with `bob highlights marker`);
  - a render with no `#` links, or with the opt-out, does not stamp it;
  - fake-pandoc renders (report `Missing`) do not stamp it.

### Docs

- `docs/highlights-ref-sync.md`:
  - add `return_links` to the standard synced user fields sentence, with one line on
    what it means and who writes it;
  - add a bullet to the "Annotation text is beautified only while rendering" list: for
    PDFs whose projection has `return_links: true`, highlight text drops return-link tag
    glyphs and `↩ p. N` pill fragments.
- `docs/highlights-create.md`: in the new section, note that bob stamps
  `return_links: true` when a render pairs links, so `bob ref sync` can keep exports
  clean.

### Acceptance

`just all` passes. Syncing a note whose sidecar contains `What I verifiedᵈ, row 12`
renders `What I verified, row 12` for a flagged PDF and leaves unflagged PDFs
byte-identical to today.

## Out of scope

- Ingested PDFs, arXiv, and web clips have no Markdown AST. The Chromium web-clip
  footnote back-references would be a separate design.
- Existing PDFs are not retrofitted; re-render to get the feature.
- Deferred target kinds: code blocks, tables, and figures as return targets; raw-HTML
  `<a id>`; Obsidian `^block` and `[[#heading]]` links; same-file `doc.md#x`
  normalization. None appear in Bryan's corpus.
- Viewer `/Named /GoBack` actions, margin notes, semantic breadcrumbs, and per-page
  letters were rejected by the research.
- Surfacing pandoc's other stderr warnings.

## Device checks for Bryan after landing

These take about 15 minutes. Each conditional fix becomes a follow-up task only if its
check fails.

1. **iPad Highlights.** Tap a tagged link, then its pill: do both land as described? Is
   an 8.8 pt-tall pill comfortable to tap at page-width zoom? _If not:_ in `stamp.rs`,
   grow the `/Rect` of links whose GoTo destination starts with `bob:ret:` by about 4 pt
   vertically and 1.4 pt horizontally.
2. **Mac Highlights.** Run the same checks, plus Go ▸ Back after a forward tap. This one
   is informational only.
3. **Annotate, save, reopen**, then tap a pill. _If_ named destinations broke, convert
   GoTo actions to explicit destinations at stamp time.
4. **Highlight a sentence containing a tag**, and highlight a pill row, then run
   `bob ref sync`. Is the exported text clean?
5. **Run one `--listen` episode** on a document with local links. _If_ sase-listen
   narrates tags or pills, render a separate narration copy with
   `-M bob-return-links=false` on listen runs only.
