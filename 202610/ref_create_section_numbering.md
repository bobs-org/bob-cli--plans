---
tier: tale
title: Number ref create Markdown sections from the top level
goal:
  A lone H1 is the unnumbered title heading, so the first H2 section is numbered 1 in
  bob ref create PDFs.
size: small
decisions:
  empty_contents:
    ask:
      Should the Markdown PDF skip its Contents list whenever no listed heading is left,
      including in heading-less files?
    choices:
      any:
        Any PDF with no listed heading skips Contents; heading-less chat exports lose
        the empty one
      title_only:
        Only title-only files skip Contents; heading-less files keep today's empty
        Contents
    default: any
    why:
      An empty Contents heading never helps a reader, and one rule is simpler to
      document and test
    answer: any
proposed_by: bbugyi200.athena.14e
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-25 16:06:59
status: wip
---

# Plan: Number `bob ref create` Markdown sections from the top section level

## Problem

The Markdown route of `bob ref create` renders through `render_temp_pdf` in
`src/native/highlights_ref/create.rs`. This route covers plain `.md` files and
vault-note `-M` renders. The route runs pandoc with
`--standalone --toc --toc-depth=3 --number-sections`. Pandoc's LaTeX writer always maps
H1 to `\section`, H2 to `\subsection`, and so on, so the numbers start at the wrong
level for the two most common Markdown shapes. Both results below were checked with
pandoc 3.1.11.1:

- **One H1 used as the title** (the usual convention, and the shape markdownlint MD025
  enforces). The title heading prints as `1 Report`, then `1.1 Setup`, `1.2 Results`.
  The Contents list and PDF bookmarks nest every section under the title.
- **No H1** (the title lives in frontmatter or the filename, as in most Obsidian notes).
  The sections print as `0.1 Setup`, `0.1.1 Details`, and so on.

The vault-note snapshot print profile (`note_snapshot/profile.rs`, `render_note_pdf`)
never numbers sections. Its test asserts that pandoc gets no `--number-sections` flag,
and it must stay unchanged. It shares `PANDOC_CODE_BREAK_FILTER` with the Markdown route
through `write_render_filters`.

## Required behavior

These rules apply only when pandoc numbers sections for LaTeX output, which today means
only `render_temp_pdf`.

1. **Exactly one H1 in the document.** Count headings in pandoc's AST, so both ATX and
   Setext H1s count, and a `# …` line inside a code fence does not. That lone H1 is the
   title heading:
   - It prints unnumbered (pandoc class `unnumbered`).
   - It stays out of the Contents list and the PDF bookmarks (class `unlisted`).
   - It stays where it is and keeps its identifier/`\label`, so `#` links to it and
     their return pills keep working.
   - It is not removed, and the `\maketitle` title block is unchanged.
2. **Every other heading moves up** so that the shallowest remaining heading level
   becomes level 1. With a lone H1 and H2 sections, the H2s number `1`, `2`, …, H3s
   number `1.1`, and H4s number `1.1.1`. If a lone H1 has only H3 children, those H3s
   number `1`, `2`.
3. **No H1.** The shallowest heading becomes level 1, so an H2-only file numbers `1`,
   `2`, … with H3s at `1.1`. This removes the `0.1` output.
4. **Two or more H1s.** Nothing changes: H1s number `1`, `2` and H2s number `1.1`.
5. **Contents.** `--toc-depth=3` now counts from the new level 1, so it lists the
   original H2–H4 under a lone H1. When no listed heading is left, the filter sets
   document metadata `toc: false`. This was verified to drop `\tableofcontents` on
   pandoc 3.1.11.1. Without it, a file whose only heading is its lone H1 would print an
   empty Contents heading.

   > [!decision] empty_contents = title_only Set `toc: false` only when the lone H1
   > existed and no other heading exists. A heading-less file keeps today's empty
   > Contents heading. The docs sentence and the heading-less contents test change to
   > match.

6. **Out of scope.** Skipped depths render as pandoc already renders them (for example,
   H2 followed directly by H4 gives `2.0.1`). Do not add `bob articles`-style level
   normalization. Existing PDFs are not re-rendered. The snapshot profile, web-article,
   and collection routes are untouched.

## Implementation

### 1. Add a gated heading pass to `PANDOC_CODE_BREAK_FILTER`

In `src/native/highlights_ref/create.rs`, add a third pass to the Lua filter's returned
list and make it the last one, so it runs after listen-card Divs have become raw LaTeX:
`return {{Meta = Meta}, {Code = Code, Div = Div}, {Pandoc = <new fn>}}`.

- **Gate.** The pass runs only when
  `FORMAT:match("latex") and PANDOC_WRITER_OPTIONS and PANDOC_WRITER_OPTIONS.number_sections`.
  On pandoc 3.1.11.1 the field is `true` with `--number-sections` and `false` without
  it, for both `--to=latex` and `-o file.pdf`. `PANDOC_WRITER_OPTIONS` has existed since
  pandoc 2.17, and the tests mention 3.1.3 as an older host. The gate turns the pass
  into a no-op for the snapshot profile and for the `return_links.rs` filter tests,
  because neither passes `--number-sections`.
- **Logic.** The sketch below was prototyped and checked on pandoc 3.1.11.1, including
  Contents and `mutool show … outline` bookmark output:

  ```lua
  local function number_from_top_level(doc)
    if not (FORMAT:match("latex") and PANDOC_WRITER_OPTIONS
        and PANDOC_WRITER_OPTIONS.number_sections) then
      return nil
    end
    local h1 = 0
    doc:walk({ Header = function(h) if h.level == 1 then h1 = h1 + 1 end end })
    local lone = h1 == 1
    local shallowest = nil
    doc:walk({ Header = function(h)
      if not (lone and h.level == 1) and (shallowest == nil or h.level < shallowest) then
        shallowest = h.level
      end
    end })
    local shift = (shallowest or 1) - 1
    local listed = 0
    doc = doc:walk({ Header = function(h)
      if lone and h.level == 1 then
        for _, class in ipairs({ "unnumbered", "unlisted" }) do
          if not h.classes:includes(class) then h.classes:insert(class) end
        end
      elseif shift > 0 then
        h.level = h.level - shift
      end
      if not h.classes:includes("unlisted") then listed = listed + 1 end
      return h
    end })
    if listed == 0 then doc.meta.toc = false end  -- see decision empty_contents
    return doc
  end
  ```

- **Doc comments.** Update the `PANDOC_CODE_BREAK_FILTER` doc comment to add the new
  responsibility. It currently mentions only inline-code breaks and listen cards. Keep
  the `RenderFilters` comment accurate.

Rejected alternatives, so the implementer does not revisit them:

- **`--shift-heading-level-by=-1`.** It applies unconditionally, turns every other H1
  into a plain paragraph (wrong for multi-H1 files), swaps the leading H1 into the
  metadata title so its anchor disappears, and does not fix H3-only or no-H1 files.
- **A separate filter file.** It would add a `RenderFilters` field that about six struct
  literals must set, including snapshot-only test helpers in `profile.rs`. The gated
  pass needs no plumbing, and `render_temp_pdf` is the only caller that passes
  `--number-sections`.

### 2. Tests (in `create.rs`'s `mod tests`)

**Pandoc-only tests.** Model these on `code_break_filter_splits_long_inline_code_paths`:
skip with an `eprintln!` when `pandoc_command()` is `None`, write the filter to a
`TempDir`, and run `pandoc <src> --to=latex --number-sections --lua-filter <filter>`.
Assert on separate substrings such as `\section{Setup}` and `\section*{Report}` rather
than on full lines, so label formatting can differ between pandoc versions. Cover:

- **Lone ATX H1.** Source: `# Report`, `## Setup`, `### Details`, `## Results`, plus a
  fenced block containing `# not a heading`. Expect:
  - `\section*{Report}` and `\label{report}`;
  - no `\addcontentsline{toc}{section}{Report}`;
  - `\section{Setup}`, `\subsection{Details}`, `\section{Results}`;
  - no `\subsection{Setup}`.
- **Lone Setext H1** (`Report\n======`). Expect the same result.
- **Lone H1 with only H3 children.** The H3s become `\section{…}`.
- **No H1** (`## Setup`, `### Details`). Expect `\section{Setup}` and
  `\subsection{Details}`.
- **Two H1s** (`# One`, `## Child`, `# Two`). Expect no change: `\section{One}`,
  `\subsection{Child}`, `\section{Two}`.
- **Gate off.** Run the lone-H1 source without `--number-sections`. It keeps
  `\section{Report}` and `\subsection{Setup}`.
- **Contents** (add `-s --toc`). A lone H1 with sections keeps `\tableofcontents`. A
  lone-H1-only file has no `\tableofcontents`. For a heading-less file, follow the
  decision: none under `any`, present under `title_only`.

**XeLaTeX render test.** Skip unless pandoc, xelatex, and pdftotext exist, following
`skip_pandoc_xelatex`. Render a lone-H1 fixture through `render_temp_pdf`, building the
plan and filters the way `render_tall_pdf`, `mermaid_plan`, and
`filters_with_optional_diagrams` do, then load the PDF with lopdf. Expect:

- `outline_titles(&document)` equals `["Setup", "Details", "Results"]`, with no title
  bookmark;
- the pdftotext text has lines matching `1\s+Setup`, `1\.1\s+Details`, and
  `2\s+Results`;
- no line matches `^\s*1\s+Report` or `0\.1`.

**Existing render tests to re-run.** Run them where xelatex exists; on athena it does.

- `tall_diagram_shares_page_with_heading_and_stays_in_margin` and its opt-out sibling.
  Their fixture has `# Tall Diagrams` plus an H3, and that H3 now renders as a larger
  `\section`. Confirm the heading and diagram still share a page. If either test fails,
  find out whether the diagram height budget must reserve more room. Do not loosen the
  assertion.
- `return_link_xelatex_render_pairs_pills_with_destinations`. The paired and target
  counts must not change.
- The listen-card and Mermaid render tests.

### 3. Documentation

`docs/highlights-create.md` is the `bob ref create` contract. In
`## Other single targets`, add a short paragraph after the paragraph on Markdown
rendering style (ragged-right, ligatures). It covers both plain Markdown and `-M`
vault-note renders and says:

- Sections are numbered from the file's top section level.
- A lone H1 prints as an unnumbered title heading and stays out of the Contents list and
  the bookmarks, so its H2s number `1`, `2`, ….
- A file with no H1 numbers from its shallowest heading.
- Files with two or more H1s number those H1s.
- A PDF with no listed heading skips the Contents list (worded to match the decision).

Do not change the `create` clap help, so `just docs-gen` is not needed. Do not edit
`CHANGELOG.md` or the `Cargo.toml` version, because the daily release train owns those.
Use a conventional commit such as
`fix(ref): number Markdown sections from the top heading level`. This is not a CLI or
JSON contract break, so it gets no `!`.

## Verification

1. Run the new and touched tests: `cargo test --lib highlights_ref::create` (narrow
   further by test name as needed). Confirm on athena that the pandoc and xelatex tests
   actually ran and did not print their skip line.
2. Run `just check` and `just docs-check`.
3. Do not run a real `bob ref create` against the live vault to verify the change. The
   xelatex test above is the end-to-end check.
