---
tier: epic
title: Highlight-friendly table layout for bob ref create PDFs
goal: "Tables whose cells hold prose print in bob-typeset reference PDFs (Markdown,
  vault-note, and web-article routes) as single-column row records that Highlights can
  select and annotate on macOS 26. Short-cell data tables keep a more readable grid. The
  choice can be reversed at the config, document, and table level, existing PDFs are
  never touched, and the result is checked for text fidelity, page counts, and PDFKit
  selection on the Mac.

  "
decisions:
  grid_hairlines:
    ask: Should data-table grids also get hairline rules between body rows?
    default: false
    why:
      Row air already separates rows; hairlines need a riskier final-position LaTeX
      rewrite
    answer: false
  decision_record:
    ask: Add a decisions memory strand recording that prose tables print as records?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: records-filter
    title: Table-layout Lua pass with prose-table records
    depends_on: []
    size: medium
    description:
      "records-filter: build the embedded table_layout.lua pass (classification,
      override precedence, complex-table guard, row-record layout, JSON report) plus its
      TeX macros and Rust embedding module, tested by running pandoc on the filter
      directly."
  - id: pandoc-wiring
    title: Wire the table pass into both pandoc chains, config, and docs
    depends_on:
      - records-filter
    size: medium
    description:
      "pandoc-wiring: add highlights.table_layout and highlights.table_prose_words, run
      the pass first in the Markdown and note-snapshot chains, print the tables summary
      line and warnings, cover the return-link and code-catalog contracts end to end,
      and document the feature in help and docs."
  - id: data-grid
    title: Improved grid for data and complex tables
    depends_on:
      - pandoc-wiring
    size: small
    description:
      "data-grid: give every table that stays a grid content-aware column widths and row
      air at body size, plus optional hairline row rules behind the grid_hairlines
      decision."
  - id: web-clip
    title: Records and grid cleanup on the web-clip print route
    depends_on:
      - pandoc-wiring
    size: medium
    description:
      "web-clip: carry the table settings in the adapter request, rewrite prose tables
      into record markup on the print DOM only, clean up reader.css grids, report counts
      back to bob, and add a tables fixture to the adapter self-test."
  - id: pdfkit-report
    title: Draft the PDFKit table-selection reports
    depends_on: []
    size: small
    description:
      "pdfkit-report: write ready-to-send drafts for the Highlights developer and Apple
      Feedback that describe the macOS 26 table-cell selection takeover, for Bryan to
      send himself."
  - id: acceptance
    title: Fidelity, page-count, and Mac probe acceptance
    depends_on:
      - data-grid
      - web-clip
    size: medium
    description:
      "acceptance: compare baseline and new renders of fixtures and real chat reports
      for text fidelity and page counts, rerun the PDFKit cell probe on the Mac (with a
      bulleted-record fallback), write the acceptance report and Bryan's manual
      checklist, and optionally record the decision."
proposed_by: bbugyi200.apollo.research.2u.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-29 20:03:03
status: wip
---

- **PROMPT:**
  [prompts/202610/ref_table_layout_for_highlights.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_table_layout_for_highlights.md)

# Plan: Highlight-friendly table layout for `bob ref create` PDFs

## Context

Bryan finds it hard to highlight text inside table cells in Highlights. The consolidated
research report explains why and recommends a fix. Read it before starting any phase:

```bash
sase artifact read research:202610/ref_create_table_layout_for_highlights/ref_create_table_layout_for_highlights.md "Table layout epic context"
```

Key findings to carry into every phase:

- **Root cause.** macOS 26 PDFKit runs asynchronous per-page document analysis. It
  recognizes tables **visually** and then takes over drags inside them. A drag returns a
  private whole-cell `isTableCellSelection` with no geometry, or nothing at all.
  Highlights then saves invisible annotations with zero `/QuadPoints`.
- **No restyled grid escapes detection.** The probe tried booktabs, content-aware
  widths, extra row spacing, hairlines, a full grid, Chrome's tagged table, and a
  CSS-grid table. PDFKit detected every one. Only a single-column **row-record** layout
  kept normal text selection (22/22 cells and 7/7 phrase drags in the Mac probe).
- **Tables are mostly prose.** 68% of tables in Bryan's preserved sources have a cell
  outside column 1 with at least 12 words, and those tables hold 89% of all table text.
- **Today's routes.**
  - Markdown (`chat`, `-M`) and vault-note snapshots go through pandoc → XeLaTeX
    `longtable` + booktabs.
  - Web articles go through Defuddle → Chromium print with `reader.css`: 0.5 pt borders,
    zebra rows, 8.5 pt type.
  - Local PDFs, arXiv, and PDF URLs are stamped as-is. They are out of bob's reach.

The prototype filters, the Mac probe (`cells_probe.swift` + `cells_probe_Info.plist`),
the probe README, and the three-table fixture (`test.md`) live in the research sidecar
under
`202610/ref_create_table_layout_for_highlights/ref_create_tables_vs_pdfkit_cell_selection__cld_assets/`.
Open the sidecar with `/sase_repo` (`sase repo open research -r "<why>"`) and read them
from the printed path. `table_hybrid.lua`, `table_stacked.lua`, and `table_widths.lua`
are prototypes to learn from, not code to copy verbatim.

## Requirements (Bryan accepted all of these)

- **R1 (reframed).** Print the tables Bryan would annotate in a layout PDFKit does not
  classify as a table. Keep grids where scanning matters more than quoting.
  - **Success test:** in Highlights, select a word, a phrase, a wrapped passage, and a
    whole former cell. Each must save with real geometry, and `bob ref scan` must import
    exactly the intended text.
- **R2 (no promise for grids).** Cells in a grid stay unreliable on macOS 26. Document
  the stopgaps: Highlights' rectangle selection, or a per-table `records` override.
- **R3 (testable readability).**
  - **Per document:** no more pages than today, no cramped columns, and visibly
    separated rows.
  - **Per table:** body-size text everywhere. Never shrink a table to fit.
  - Prose comparison tables lose side-by-side scanning. That is an accepted trade-off,
    reversible per table.
- **R4 (scope, in order).**
  1. The generic Markdown route and the note-snapshot route, which share the filter.
  2. The web-clip route.
  3. PDFs bob does not typeset get a reader-side report only.
- **R5 (fidelity).**
  - The preserved `source.md` keeps the authored table; only the print changes.
  - Records preserve wording, order, empty cells (shown as `—`), links, inline code,
    emphasis, captions, table numbering, and ids.
  - Complex tables stay grids, with a diagnostic.
- **R6 (reversible).** Provide a global default, a per-document override, and a
  per-table override. Changing the default never touches existing PDFs or notes.
- **R7 (new renders only).** Never bulk re-render. Only an explicit create or
  `-f overwrite` uses the new layout.
- **R8 (accepted loss).** Highlights' image-selection table-to-CSV export stops working
  on tables printed as records.

## Non-goals

- No CLI flag in v1. The overrides live in config, frontmatter, and the document.
- No engine switch, no tagged-PDF work, no rasterized tables, and no duplicate
  "appendix" copies.
- No changes to the source store, the `bob ref code` / `bob ref links` JSON contracts,
  marker writes, or PDF write coordination.
- No re-rendering of existing references, and no change to `bob ref attach-source`. Its
  verification pandoc run matches already-filed grid PDFs, so it must not gain the pass.
- Out of scope, as separate follow-ups:
  - the emoji tofu on the generic Markdown route;
  - a `bob ref doctor` warning for zero-geometry ghost annotations;
  - nudging research agents to prefer numbered lists over prose tables.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`; releases ride the daily train.

## Shared contract (all phases must agree on this)

### Layout choice per table

1. A table is **complex** when any of these hold:
   - a cell has `row_span` or `col_span` > 1;
   - there is more than one head row, an intermediate body head, a foot row, or more
     than one table body;
   - `row_head_columns` > 0;
   - a cell contains a nested table;
   - a header cell contains a Link, Note, or Image, or any element with a non-empty
     identifier.

   A complex table **always** stays a grid, even when records were requested. A table
   with no body rows is left untouched.

2. Otherwise the **forced layout**, if any, wins. Precedence from highest:
   1. **Per table:** a fenced Div with class `bob-records` or `bob-grid` around the
      table. The nearest enclosing Div wins. A Div with both classes is ignored, with a
      warning.
   2. **Per document:** frontmatter `bob-tables: auto | records | grid`, trimmed and
      case-insensitive. An invalid value warns and is ignored.
   3. **Global:** pandoc metadata `bob-tables-default`, which bob passes from config
      `highlights.table_layout` (default `auto`). It must be a different key from
      `bob-tables`, because a `--metadata` value would override the document's
      frontmatter.
3. In `auto`, a table is **prose** when some body-row cell in column 2 or later has at
   least `prose_words` whitespace-separated words of `pandoc.utils.stringify` text. bob
   passes `prose_words` as metadata `bob-table-prose-words` from config
   `highlights.table_prose_words` (default 12). Header and caption words do not count.
   Prose tables print as **records**; all others print as **grids**.

### Report

- The pass writes a JSON report to the path in metadata `bob-table-layout-report`, the
  same way `return_links.lua` handles `bob-return-links-report`:

  ```json
  {
    "version": 1,
    "tables": 5,
    "records": 3,
    "grids": 2,
    "kept_grid": [{ "table": 4, "reason": "row or column spans", "requested": "auto" }],
    "warnings": ["bob-tables: unknown value 'rows' (use auto, records, or grid)"]
  }
  ```

  - `table` is the 1-based index in document order.
  - `requested` is `auto` (it classified as prose) or `records` (forced).
  - `kept_grid` lists only tables that would otherwise have printed as records.

- The pass must never fail a render. Wrap each table's rewrite in `pcall`. On error,
  keep the original table and add a warning.

### Records layout (pandoc routes)

- Replace the Table with a Div. The Div carries the table's identifier, its classes, and
  class `bob-table-records`, so `#id` links and return pills still resolve.
- Inside that Div, in order:
  1. A `\begin{BobTableRecords}` RawBlock.
  2. If the table has a caption: a caption Para made of a `\BobTableCaption{}` RawInline
     prefix plus the caption's inlines. The macro steps the `table` counter and prints
     the bold "Table N:" lead, so numbering continues across records and grids.
  3. Each body row, as follows.
  4. `\end{BobTableRecords}`.
- **Each body row:**
  - Start the row with a `\BobRecordStart` RawBlock: about 4 pt of space plus
    `\needspace{3\baselineskip}`, so the lead-in stays with its first field.
  - **Lead-in.** Use column 1 as a bold lead-in Para (`pandoc.Strong` of the cell's
    inlines) only when _every_ body row's column-1 cell is empty, or a single Plain/Para
    of at most 8 words. The column-1 header label is then not repeated. Otherwise column
    1 becomes an ordinary labeled field.
  - **Single-block field.** One Para: a label group (`{\BobRecordLabel ` + a deep copy
    of the header cell's inlines + `:}`), a Space, then the cell's own inlines.
  - **Multi-block field** (lists or code from grid tables): a label-only Para, then the
    cell's blocks between `\begin{BobRecordValue}` / `\end{BobRecordValue}` (an indented
    list environment).
  - **No label** (empty header or headerless table): the value goes inside
    `BobRecordValue`.
  - **Two-column table with a lead-in:** the column-2 value is an indented
    `BobRecordValue` paragraph with no label.
  - **Empty cell:** prints `—` (Plain `Str "—"`).
- **Move cell content as AST nodes.** Never use `stringify` or `RawBlock` for cell
  bodies. Every cell's content must appear exactly once. Header labels are the only
  repeated text. Never pass cell content as a LaTeX macro argument: use prefix/suffix
  RawInlines and environments, so footnotes, code, and catalog anchors behave as body
  text.
- No side-by-side columns anywhere. Column alignment is ignored.
- Walk top-down (`traverse = 'topdown'`, available in pandoc 3.1.3) so a wrapper Div's
  class is known when its table is visited. Tables nested in lists, block quotes, and
  Divs must work.
- Use only Lua APIs that pandoc 3.1.3 (athena) supports. They must also work on the
  Mac's newer pandoc.

### TeX macros (`src/native/highlights_ref/table_layout.tex`)

- Use only already-declared packages: `needspace` and xcolor via the template.
- Define:
  - `BobTableRecords` (medskip space before and after);
  - `\BobRecordStart`;
  - `\BobRecordLabel`: a font switch, `\sffamily\bfseries` in a muted
    `BobRecordLabelInk` (about `#4B5563`);
  - `BobRecordValue` (left margin about 1.5em, no extra `topsep`);
  - `\BobTableCaption`: `\refstepcounter{table}` plus a bold `\tablename~\thetable:`.
- Labels use the route's sans font, so DejaVu Sans on the Markdown route and the
  snapshot sans font in note snapshots.

### Summary output (`bob ref create`)

- Print a stdout line beside the existing `links:` line, only when the document has at
  least one table: `tables: 3 records · 2 grids`, with singular forms `1 record` and
  `1 grid`.
- Print to stderr, using the styled warning prefix:
  - one line per `kept_grid` entry, for example
    `warning: table 4 stayed a grid: row or column spans`;
  - one line per report warning.
- A missing report prints nothing. An unreadable report prints one warning, mirroring
  `return_links::emit_outcome`.

## Phase records-filter: Table-layout Lua pass with prose-table records

Build the pass and its embedding without wiring it into `bob ref create` yet. This phase
makes no user-visible change.

1. Add `src/native/highlights_ref/table_layout.lua` implementing the shared contract:
   - classification;
   - override precedence;
   - the complex guard;
   - the records layout;
   - the report, written in the filter's final `Pandoc` stage like `return_links.lua`.

   For now, grid tables are returned unchanged. Leave a clearly named
   `improve_grid(tbl, ctx)` hook returning the table as-is, which `data-grid` fills in.

2. Add `src/native/highlights_ref/table_layout.tex` with the macros above.
3. Add `src/native/highlights_ref/table_layout.rs`, registered in the module tree:
   - `FILTER` and `HEADER_INCLUDES` via `include_str!`;
   - constants for the metadata keys `bob-tables`, `bob-tables-default`,
     `bob-table-prose-words`, and `bob-table-layout-report`;
   - a `Report` / `ReportOutcome` reader that accepts only version 1;
   - a `tables_line(&Report) -> Option<String>` formatter;
   - an `emit_outcome(&ReportOutcome, &Styler)`, modeled on `return_links.rs`.
4. Add `table_layout::HEADER_INCLUDES` to `render_tex.rs`'s
   `header_packages_are_all_declared` list. It must pass without new packages.
5. Tests in `table_layout.rs` (pandoc-gated, skip with a message when pandoc is missing,
   following the existing `create.rs` pattern):
   - Run
     `pandoc -f <code_catalog::MARKDOWN_PANDOC_READER> -t native|latex --lua-filter table_layout.lua`
     with explicit `--metadata` values.
   - **Classification:** a 3-column prose table yields no `longtable` in the LaTeX, and
     has `BobTableRecords` with labels in row order. A short numeric table stays a
     `longtable`. The 12-word boundary is honored, and `bob-table-prose-words` moves it.
   - **Overrides:** each precedence level works:
     - `.bob-records` forces records on a data table;
     - `.bob-grid` keeps a prose table a grid;
     - frontmatter beats `bob-tables-default`;
     - a Div with both classes warns.
   - **Complex:** a grid table with a row span, a table with a foot, and a prose table
     whose header has a link stay grids and appear in `kept_grid` with reasons.
   - **Fidelity:** in the native output, every body cell's text appears exactly once and
     in order. Links, inline code, emphasis, and a footnote inside prose cells survive
     as nodes. An empty cell renders `—`. A caption and a table identifier survive on
     the wrapper Div with `\BobTableCaption`.
   - **Lead-in rule:** a table whose column 1 has a 9-word cell gets labeled column-1
     fields and no bold lead-in. A 2-column table with short keys gets an indented
     unlabeled value.
   - **Robustness:** a malformed `bob-table-prose-words` value falls back to 12 with a
     warning. A nested table inside a grid-table cell is complex.
   - When xelatex is available, render one fixture to PDF with `HEADER_INCLUDES` and
     check that the extracted text holds labels and cell text.

## Phase pandoc-wiring: Wire the table pass into both pandoc chains, config, and docs

1. **Config** (`src/native/config/mod.rs`):
   - Add `highlights.table_layout` (`auto | records | grid`, default `auto`). Any other
     value is `ConfigError::Invalid`, naming the key, the config path, and the allowed
     values.
   - Add `highlights.table_prose_words` (an integer from 1 to 200, default 12).
     Out-of-range or non-integer values are `Invalid`.
   - Add accessors with defaults, plus parse tests modeled on the
     `coordinate_pdf_writes` tests.
2. **Filters:**
   - `write_render_filters` (`create.rs`) also writes `table-layout.lua`.
   - `RenderFilters` gains `table_layout`, `table_layout_report` (`table-layout.json`),
     and the resolved `layout_default` / `prose_words` settings.
   - Load the settings once from `load_highlights_config` in the route callers (create,
     snapshot, and overwrite) and pass them into `write_render_filters`. Do not load
     config inside it, because tests call it directly. Tests pass the defaults.
   - A config error must surface like the other `highlights.*` errors.
   - Update every caller, including the overwrite paths and both test helpers.
3. **Chains.** Pass the filter as the **first** `--lua-filter`, before `code_catalog`,
   in both `render_temp_pdf_from` (`create.rs`) and `render_note_pdf`
   (`note_snapshot/profile.rs`).
   - Remember that `profile.rs` resolves filter paths for `cwd=scratch`.
   - Add `--metadata` arguments for `bob-tables-default`, `bob-table-prose-words`, and
     `bob-table-layout-report`.
   - Append `table_layout::HEADER_INCLUDES` to both chains' `header-includes`.
   - The snapshot projection escapes `|` today, so vault-note tables never reach pandoc
     as Tables. Wiring the pass there keeps the chains identical, and the note's `-M`
     route already gets it through `render_temp_pdf_from`. Do not add `bob-tables` to
     the snapshot masthead.
4. **Reports.** `RenderReports` and the snapshot `RenderOutcome` gain the table-layout
   outcome. Call `table_layout::emit_outcome` next to every `return_links::emit_outcome`
   call in `create.rs`: the Markdown route and the snapshot route. Overwrite renders
   should print it wherever they print render summaries; if they print none, skip them.
5. **End-to-end tests** (pandoc/xelatex-gated, in `create.rs` and `profile.rs`), each
   with a prose table fixture:
   - The kept `.tex` has no `longtable` for the prose table and does have one for a data
     table.
   - A same-document `#` link inside a prose cell still gets its raised tag and a `↩`
     return pill.
   - A URI link inside a prose cell lands in the code catalog's links with a page, and a
     code block inside a grid-table prose cell keeps its catalog entry and page range.
   - Existing table head/foot/caption return-link ineligibility tests still pass for
     grids.
   - Filter order: `table-layout.lua` precedes `code-catalog.lua` on both chains. Extend
     the existing order assertions in `profile.rs`.
   - Frontmatter `bob-tables: grid` keeps the prose table a grid.
   - The `tables:` line and kept-grid warnings print.
6. **Docs and help** (read the `docs.md` and `cli_rules.md` memory first):
   - Add a "Tables" section to `docs/highlights-create.md` covering:
     - why prose tables print as records (PDFKit's table-cell takeover on macOS 26);
     - the auto rule and its threshold;
     - the three override levels with examples;
     - complex tables;
     - the `tables:` line;
     - R2's stopgaps for grids;
     - R8's CSV-export loss;
     - that existing PDFs never change (R7).
   - Update the "Local links and return pills" table so it says that cells and captions
     of a table printed as records are ordinary body text, and its id is a Div target.
   - Add two rows to `docs/reference/configuration.md`'s `highlights` table.
   - Add one sentence to `bob ref create --help`'s Output paragraph in `create.rs`, next
     to the `bob-return-links: false` sentence.
   - Run `just docs-gen` and `just docs-check`.
7. Run `just all` and fix any fmt, lint, or test failures.

## Phase data-grid: Improved grid for data and complex tables

Fill in `improve_grid` for every table that stays a grid: data tables, forced grids, and
complex tables.

1. **Widths.** Port `table_widths.lua`'s CSS-auto rule onto `colspecs`, skipping complex
   tables with spans.
   - The minimum is the longest word, capped at 24 characters. The maximum is the
     longest cell.
   - If the summed maximum plus `3 × columns` fits the route's line length, use
     `ColWidthDefault` for natural `l` columns.
   - Otherwise give each column its minimum and share the slack in proportion to
     `max − min`.
   - Keep author widths when they are deliberate: non-default relative widths whose
     largest and smallest differ by more than 10%. Equal dash-count widths like
     `|---|---|---|` are not deliberate.
   - The line length comes from new metadata `bob-table-line-chars`. Rust passes a
     conservative per-route value, calibrated so natural columns never overflow: about
     84 for the Markdown route's 6.8in measure, and the equivalent for the snapshot
     geometry. Add the key constant to `table_layout.rs`.
2. **Row air.** Wrap each grid Table in RawBlocks
   `\begingroup\renewcommand{\arraystretch}{1.2}` … `\endgroup`. Keep body-size text and
   never add `\small`.
3. > [!decision] grid_hairlines
   >
   > **If yes:**
   >
   > - Add `table_rules.lua`, run as the **last** filter on both chains, after
   >   `code-catalog-report.lua` (and after `note.lua`).
   > - It converts each Table that `table_layout.lua` marked with class `bob-table-grid`
   >   into a LaTeX RawBlock via `pandoc.write`, then inserts
   >   `\noalign{\vskip2pt{\color{BobRule}\hrule height .4pt}\vskip2pt}` between body
   >   rows. The booktabs top, mid, and bottom rules stay.
   > - Because pandoc then stops loading the table packages, add
   >   `\usepackage{longtable,booktabs,array,calc}` to the table header includes and
   >   declare any missing ones in `RENDER_TEX_PACKAGES`.
   > - Prove with tests that footnotes, code-catalog link anchors, and return pills in
   >   data cells still work.
   >
   > **If no:** skip hairlines. Mention in the Tables docs that grids get widths and row
   > air only.
4. **Tests:**
   - widths for a fitting table (default colspecs) and an overflowing one
     (proportional);
   - deliberate author widths kept;
   - the `\arraystretch` group present around grids;
   - complex tables untouched apart from row air;
   - the existing grid return-link and catalog tests still pass.
5. Update the grid sentence in the docs' Tables section, then run `just docs-check` and
   `just all`.

## Phase web-clip: Records and grid cleanup on the web-clip print route

1. **Request.** In `src/native/highlights_ref/clip_adapter.rs`, the capture request
   (with render) and the compose request gain an optional
   `"tables": {"layout": "auto|records|grid", "prose_words": N}`, built from
   `HighlightsConfig` using the `pandoc-wiring` accessors.
   - Protocol stays 1, and the field is optional, so adapters that omit it default to
     `auto` / 12.
   - Update the request JSON fixture tests.
2. **Adapter** (`scripts/web_clip/web_clip_adapter.py`). Validate `tables`: a malformed
   value is `invalid_request`. Thread it through `web_clip_render.render(...)` and
   `render_collection(...)` into `print_pdf`.
3. **Print DOM rewrite** (`scripts/web_clip/web_clip_render.py`). In `print_pdf`, run a
   new `page.evaluate` with the settings **before** the code-anchor evaluate.
   - It rewrites `.article-body` tables (outermost only) using the same rule as the Lua
     pass:
     - **Prose:** a body cell beyond column 1 has at least N words of `textContent`.
     - **Complex:** `rowspan` or `colspan` > 1, more than one `thead` row, a `tfoot`, a
       nested table, or a header cell containing `a`, `img`, or `[id]`.
     - **Header row:** the single `thead` row, else a leading row made only of `th`
       cells.
   - Record markup moves the original child nodes (`appendChild`), never `innerHTML`, so
     links, code, and images survive:
     - Wrapper: `div.bob-records`, carrying the table's `id`.
     - Caption: `p.bob-records-caption` with the moved caption nodes.
     - Each row: a `div.bob-record` holding:
       - `p.bob-record-lead > strong`, using the same lead-in rule;
       - fields, each a `div.bob-record-field` with a `span.bob-record-label` (a cloned
         header plus `:`), then the moved cell nodes;
       - `div.bob-record-value` for an unlabeled value;
       - `—` for empty cells.
   - Return counts in the shared report shape (`tables`, `records`, `grids`,
     `kept_grid`).
   - `source.html` and the saved Markdown keep the original `<table>`s (R5).
4. **`reader.css`.**
   - Record styles:
     - body text font and size;
     - labels in Inter semibold, muted (`#4b5563`);
     - a bold lead with `break-after: avoid`;
     - about 4pt between records.
   - Remaining grids:
     - drop vertical cell borders, keeping a 0.5pt bottom rule per row;
     - drop zebra fills;
     - raise table type from 8.5pt to 9.5pt;
     - keep `thead` repeating across pages.
5. **Response.** The adapter response gains optional `tables` counts. Rust parses them
   leniently: absent or invalid means no line. Print the same `tables:` line and
   kept-grid warnings with the `pandoc-wiring` helpers.
6. **Self-test.**
   - Add `scripts/web_clip/fixtures/tables.html` with one prose table, one numeric
     table, and one prose table with a `rowspan`.
   - Extend the browser-backed self-test (skipped without a browser) to check that:
     - counts are 1 record, 2 grids, and 1 kept grid;
     - `source.html` still has 3 `<table>` elements;
     - with mutool, printed text holds the label lines and each prose cell's text
       exactly once.
   - Run `just check-web-clip-adapter`.
7. **Docs.** In `docs/highlights-clip.md`, cover the web route's table behavior:
   config-only overrides, since there is no frontmatter or per-table class on this
   route. Cross-link the Tables section in `docs/highlights-create.md`. Then run
   `just docs-check` and `just all`.

## Phase pdfkit-report: Draft the PDFKit table-selection reports

Write drafts only. Never send email, file feedback, or post anything; Bryan sends them.

1. Open the research sidecar with `/sase_repo`. Add
   `202610/ref_create_table_layout_for_highlights/pdfkit_table_selection_report_drafts.md`
   holding two drafts.
2. **The Highlights developer email** covers:
   - macOS 26 `PDFView` runs document analysis by default;
   - drags in detected tables yield `isTableCellSelection` with infinite bounds, or
     `nil`;
   - Highlights then saves zero-QuadPoint annotations;
   - disabling the private `documentAnalysisEnabled` restored selection in a probe.

   It asks for a preference if one is feasible, and at least a refusal to save
   geometry-less annotations.

3. **The Apple Feedback Assistant report** (PDFKit / Preview) covers:
   - minimal steps to reproduce;
   - expected and actual behavior;
   - the environment: macOS 26.5 (25F71), Highlights 2026.1.2;
   - the public `PDFPage.selection(from:to:)` returning `nil` after analysis;
   - the Apple Community threads 256257514 and 256139925;
   - a request for a public opt-out.
4. Optionally render a minimal repro PDF beside the drafts: today's pandoc booktabs
   chain on a three-row prose table, with no vault content. Commit it in the sidecar.
5. Keep private vault content out of the drafts. Commit the sidecar change.

## Phase acceptance: Fidelity, page-count, and Mac probe acceptance

1. **Baselines.**
   - Build a baseline `bob` from the commit just before `records-filter` landed, using a
     scratch git worktree outside the vault. Build the current `bob` too.
   - Pick at least five table-heavy preserved chat sources from
     `~/bob/.bob/ref-sources/chat/*/source.md` (read-only). Include
     `now_tag_vs_in_progress_status` if present. Add the cld `test.md` fixture.
2. **Renders.** Render each source with both binaries, using
   `bob ref create <scratch copy> --output <scratch>/<variant>/<stem>.pdf --no-audio`
   (plus `-P` if required).
   - External output writes nothing to the vault. Confirm that the vault's
     `git status --porcelain` is identical before and after, as
     `scripts/dev/note_snapshot_corpus_gate` does.
   - Also capture the web `tables.html` fixture with `-H` to an external output.
3. **Checks.** Keep scripts in the research sidecar's assets directory for this report,
   not in bob-cli.
   - **Pages:** the new page count is at most the baseline for every document (R3).
   - **Fidelity:** parse each source's tables with `pandoc -t json`. Every body cell's
     whitespace-normalized text appears in the new PDF text (`mutool draw -F txt`) once,
     in order, with nothing dropped or duplicated.
   - Record the `tables:` lines and any kept-grid warnings.
4. **Mac probe** (best effort; `ssh mac`, user `bbugyi`, see the `tailnet.md` memory).
   - If the Mac is reachable, copy the renders and the research `cells_probe.swift` /
     `cells_probe_Info.plist`. Build and run them per the probe README in `on` mode,
     with targets for record cells and phrase drags in both a pandoc render and the web
     render.
   - **Expected:** 0 table-cell selections on record content, and every phrase drag
     selects text.
   - **If any record content is detected as a table cell:** switch record fields in
     `table_layout.lua` and the web JS to the measured bulleted "stacked" layout (fields
     as list items under the bold lead, as in `table_stacked.lua`). Re-render, re-probe,
     update the tests and the docs wording, and run `just all`.
   - If the Mac is unreachable, record "probe not run" and continue; never wait on it.
5. **Acceptance report.** Write
   `202610/ref_create_table_layout_for_highlights/acceptance.md` in the research
   sidecar, holding:
   - a per-document table of pages before and after, records and grids, and fidelity;
   - the probe results;
   - notes on the threshold;
   - a manual checklist for Bryan:
     - the R1 Highlights session (word, phrase, wrapped passage, and whole former cell;
       save, close, and reopen; non-zero `/QuadPoints`; `bob ref scan` quotes match);
     - the R3 readability check at normal zoom;
     - when to tune `highlights.table_prose_words` or use `.bob-grid`.

   Commit the sidecar.

6. > [!decision] decision_record
   >
   > **If yes:** use `/sase_memory_write`. Add a `decisions` strand
   > `prose-tables-print-as-records` in the web's existing record format:
   >
   > - **Applies to:** `bob ref create` pandoc and web-clip renders.
   > - **Claim:** prose tables print as single-column records and other tables as grids,
   >   reversible at three levels.
   > - **Why:** PDFKit's visual table detection takes over drags, and no restyled grid
   >   escapes it.
   > - **Cost:** comparison tables lose side-by-side scanning, and CSV export is lost.
   > - **Reopens when:** a macOS PDFKit change stops table-drag takeover.
   > - **Evidence:** the research ref, this epic's commits, and the acceptance report.
   >
   > Run `sase memory init`, `just docs-gen` (which regenerates
   > `docs/develop/decisions.md`), and `just docs-check`.
   >
   > **If no:** do not touch memory. Follow `/sase_memory_write`'s declined-change rule
   > (a `PROPOSED FOLLOW-UP:` note on the phase bead only when no human reviewed the
   > plan).

## Risks

- **Prose comparison tables Bryan wants as grids.** `.bob-grid`, `bob-tables: grid`, and
  the threshold config cover this. If misclassification is common, consider a
  short-cell-majority rule rather than a higher threshold.
- **The final record styling was never probed.** The 22/22 result came from the bulleted
  stacked variant. The acceptance phase's fallback covers this.
- **Pandoc version drift between athena (3.1.3) and the Mac.** Lua APIs are restricted
  to 3.1.3, and the tests run on athena.
- **Natural-width columns overflowing in `data-grid`.** Calibrate the line length
  conservatively, and check page counts and overfull boxes in acceptance.
- **Apple fixes PDFKit.** Flip `highlights.table_layout: grid`. There is no stored data
  to migrate.
