---
tier: tale
title: Note snapshot print profile and observable XeLaTeX rendering
goal: Complete bob-cli-87.4 with an AST-preserving note print profile, two-step renderer,
  actionable diagnostics, and verified PDF appearance and text.
size: medium
proposed_by: bbugyi200.athena.bob-cli-87.4
bead: bob-cli-87.4
status: done
---

- **PARENT:**
  [202610/note_snapshots.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/note_snapshots.md)
- **BEAD:**
  [bob-cli-87.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-87/bob-cli-87.4.md)

# Note snapshot print profile and observable XeLaTeX rendering

Implement the `print-profile` phase of `plan:202610/note_snapshots.md`, tracked by
`bob-cli-87.4`. Produce a callable renderer that turns the approved projected Markdown
contract into a polished PDF, preserves links and selectable text, and reports TeX
failures and missing glyphs. Finish and close only `bob-cli-87.4`.

## Scope and planning evidence

`sase bead read bob-cli-87.4` showed no previous implementation notes or unfinished
attempt. This is one medium implementation unit: fonts and coverage support are already
landed, the input contract is settled, and the existing code provides command discovery,
scratch ownership, return links, package hints, and PDF test helpers. The work is one
coupled template/filter/runner with one contract fixture; it needs no new architectural
or reviewer decision.

Read the parent design through `sase artifact read` before implementation. The audited
research report
`research:202610/area_project_note_reading_pdfs/area_project_note_reading_pdfs__cld.md`
documents the spike and TeX traps. The parent design overrides prototype behavior. The
research companion `.tex` and `.lua` files were located with `/sase_repo`, but the
`research:` reader filters non-Markdown files. If consulting those prototypes, open the
research repo through `/sase_repo`, register those two companion files with
`sase artifact create`, and read the resulting `file:` references; do not silently read
sidecar artifacts directly. This plan supplies the implementation contract independently
of those prototypes.

Relevant existing code:

- `src/native/highlights_ref/note_snapshot/fonts.rs`: `GlyphCoverage::new`, `covers`,
  `write_fonts`, and `write_glyph_table`; thirteen embedded static faces.
- `src/native/highlights_ref/create.rs`: `pandoc_command`, `RenderFilters`,
  `PANDOC_HEADER_INCLUDES`, `PANDOC_CODE_BREAK_FILTER`, and `render_temp_pdf`.
- `src/native/highlights_ref/return_links.{rs,lua,tex}`: existing filter, macros, typed
  report, and diagnostics.
- `src/native/highlights_ref/workdir.rs`: `ScratchDir` already implements
  `BOB_HIGHLIGHTS_KEEP_WORKDIR=1` and secure scratch cleanup.
- `src/native/highlights_ref/render_tex.rs`: package inventory, `render_failure_hint`,
  and `header_packages_are_all_declared`.
- PDF tests near `return_link_xelatex_render_pairs_pills_with_destinations` in
  `create.rs` already inspect named destinations, annotations, bookmarks, and pills.

At planning time athena has pandoc 3.1.11.1, xelatex, pdffonts, pdftotext, pdftoppm, and
the anticipated extra TeX packages. `commonmark_x` supports YAML metadata, fenced divs,
bracketed spans, and attributes. `sase bead epic-symbols bob-cli-87.4` currently reports
no entries; check again before closing.

Honor the epic's final decisions: `default_text_layer=yes`, `obsidian_command=yes`, and
`decision_record=no`. The plain-Markdown text-layer phase and plugin-command phase
implement the first two outside this unit. Do not edit memory or decisions. Do not add
projection semantics, CLI options, intake or provenance wiring, image conversion, corpus
tooling, or Mac acceptance here; those have their own phases. Fixtures must be
synthetic. Never copy live vault content.

## 1. Establish the renderer seam and contract fixture

Add `note_snapshot/profile.rs`, `note_snapshot/note.tex`, and `note_snapshot/note.lua`,
wiring the module without overwriting sibling modules that other epic phases may have
added. Embed the TeX and Lua assets with `include_str!`.

Expose a narrow renderer callable from the later create-wiring phase:

`render_note_pdf(projected_md, scratch, filters, meta) -> Result<RenderOutcome>`.

- `projected_md` is the already serialized Markdown text. `scratch` identifies the
  caller-owned work directory; rendering does not create, stamp, or install a vault PDF.
  The caller's `ScratchDir` keeps its existing cleanup/debug ownership.
- Reuse `RenderFilters` (code-break, return-links, and report paths), minimally widening
  its type/field visibility to the enclosing Highlights module if needed. Keep the
  renderer and its exposed types at compatible visibility for the `private_interfaces`
  lint. Avoid requiring the concurrently developed view model.
- A small metadata input carries the identity title, vault-relative source path, note
  kind, and optional listen URI needed for PDF Info and the existing filter. Do not
  interpret Bob task fields or build a second metadata parser here.
- `RenderOutcome` returns the scratch PDF path, typed render warnings, and the existing
  `return_links::ReportOutcome` so create-wiring can emit the report and stamp its
  marker correctly. Document this seam in Rust comments and the phase's completion note.

Create or extend `tests/fixtures/note_snapshot/contract.projected.md` without discarding
another phase's fixture changes. It must use every contract element, with invented
content, all six statuses, nested ordinary and task lists, and links in the masthead,
task text, chips, and children. Stage a small synthetic image in the test scratch
directory for its `assets/...` image reference.

The contract is unchanged:

- YAML: `title`, `bob-snapshot: true`, `bob-kicker`, `bob-display-title`, optional
  `bob-dek`, `bob-meta` inline-item list, `bob-stats` list of `{status, count}`,
  optional `bob-omitted`, `bob-snapshot-line`, `bob-footer`, `bob-glyph-table`, and
  boolean `bob-return-links`. String values contain inline Markdown.
- Sections: `# Requirements {#requirements}` and lower levels. Lane headings:
  `## Next & In Progress {.lane}`. Appendix: `# Closed tasks {#closed .appendix}`.
- Tasks: list item starting with `[]{.st .st-blocked}`, then an optional anchor such as
  `[]{#t-example}`, text, and chips. Status classes are `st-open`, `st-next`, `st-wip`,
  `st-blocked`, `st-done`, and `st-canceled`.
- Spans: `.chip`, `.chip.chip-accent`, `.label`, `.muted`, and `.missing`. A link may
  itself have `.chip`; retain its live link.
- Nested labeled/log rows retain their paragraphs, emphasis, links, and lists.
- Divs: `.embed` with a `source` label, `.livequery`, `.callout` with `kind` and
  `title`, and `.compact`. Images use standard Pandoc attributes such as `width=62%`.

## 2. Implement the visual system in note.tex

Use the parent design's visual tokens, tuning only after looking at actual pages:

- US Letter, side margins about 1.1in, top 0.9in, bottom 1in. Source Serif 4 SmText at
  10.75/14.75pt; 5pt paragraph skip and no indent. Use the actual bundled SmText file
  names from `assets/print_fonts/README.md`.
- Inter structure: 7.5pt tracked uppercase kicker and lane labels; 26/30pt Inter Display
  SemiBold title; H1 15pt SemiBold with a 1pt rule; H2 11.5pt; H3 10.5pt muted. Dek is
  Source Serif italic 13/17pt. Metadata/nav is 8.5pt and footer 7.5pt.
- JetBrains Mono at 0.86 scale, with the existing code-break behavior. Load all faces by
  relative `Path=fonts/`, including static italic/bold variants and Noto Emoji fallback;
  use `Ligatures=NoCommon` on every family and explicit font.
- Ink `1F2328`, muted `6E7781`, rule `D0D7DE`, existing `BobLinkInk=2F5E96`, next/wip
  `3B6EA8`, blocked `B4443C`, done `2E7D4F`. Chips: Inter Medium 7pt, 1.5pt corners,
  fill `EEF1F4`; accent fill `FFF4D6` and ink `9A6700`.
- Implement masthead, footer, `BobTasks`/`BobTask`, `BobChip`, `BobChipAccent`,
  `BobLabel`, `BobMuted`, `BobFallback`, and the embed/livequery/callout/compact
  environments. Use TikZ paths for open, next, wip, blocked, done, and canceled glyphs;
  never place their Unicode symbols in the PDF text layer.
- Task glyphs sit on the first baseline with hanging indentation. Ordinary lists share
  the indent column, with muted bullets and 2–3pt item spacing. Preserve nesting and
  paragraph breaks. Keep the task with its first child while allowing long child content
  and embeds to paginate.
- Masthead is unbroken and ends with a rule; disable automatic `maketitle`. Headings
  reserve about four lines with needspace; widows/orphans are 10000. Images are
  centered, framed with a hairline, and do not split. Embeds have an accent left rule,
  1em inset, and muted source label.
- No header, standalone TOC, or section numbering. Keep heading bookmarks and explicit
  anchors. Footer is short title/date on the left and `n / N` on the right. Build the H1
  navigation line with page suffixes only when LastPage > 8, using refcount and lastpage
  inside TeX across the normal TeX passes.
- Ragged right, no hyphenation (`hyphenpenalty` and `exhyphenpenalty` 10000), emergency
  stretch, and xurl for URLs. For long text tokens, Lua adds zero-text break
  opportunities after `_`, `/`, `-`, and `.`; no visible spacer characters.

Compose the header with existing code/listen and return-link macros without duplicate
package-option conflicts. Ensure display-font commands do not introduce missing glyphs
in return tags/pills or the existing listen card.

## 3. Map the contract through Pandoc AST nodes

Run `note.lua` after code-break/listen and return-links. Implement its processing order
explicitly, using Pandoc APIs available in 3.1.11.1 and newer.

Wrap inline/block AST content with fixed RawInline/RawBlock macro delimiters. Never
stringify or render user content into a TeX string with `pandoc.write`. Links, emphasis,
code, anchors, and return-link additions must remain AST content. Use AST inline parsing
for textual div attributes such as source/title. Whitelist status/kind selectors and
validate numeric counts before using them as macro selectors. Keep user labels, titles,
paths, and URLs out of raw macro arguments.

Convert only status-marked list items to Bob task rows. Preserve the other items of a
mixed list and every child block. Consume the empty status marker while retaining task
anchors and the return-link filter's injected content. Map spans, link chips, lanes,
divs, and appendix headings without dropping attributes or turning content into flat
text. Do not reimplement Bob parsing or project-status decisions in Lua.

Build the masthead from the metadata AST and H1 navigation from headers. Generate safe
internal labels for TeX page references rather than interpolating arbitrary heading ids.
Create navigation after return-link pairing so its generated links do not add return
pills to every section. Preserve the report produced by the existing return-link pass
and the `bob-return-links: false` opt-out.

Remove U+FE0F and apply cmap-driven glyph selection to visible body and metadata text.
The existing flat `glyphs.lua` integer set determines Noto Emoji wrapping. There is one
verified coverage edge to fix within this renderer: `▸` is present in JetBrains Mono but
absent from Source Serif, Inter, and Noto Emoji; the existing union-of-body-fonts
coverage table does not select a usable body face for it. Extend the generated table
compatibly with per-role body-font overrides, retaining its existing integer-keyed
fallback set. Select another bundled body face where necessary before Noto; do not
hard-code this character or claim union coverage means the currently selected face can
render it. Add focused coverage and PDF regressions, including metadata text and inline
code. Avoid changing font assets.

## 4. Run pandoc and XeLaTeX as separate steps

In `profile.rs`:

1. Write `projected.md`, embedded profile assets, `fonts/`, and `glyphs.lua` into
   scratch. Use a header name distinct from the generated `note.tex`. Use the
   `GlyphCoverage` APIs and map I/O failures into contextual command errors.
2. Resolve pandoc via `create::pandoc_command`; discover xelatex through
   `BOB_XELATEX_COMMAND` then PATH, using the same override conventions. Resolve
   relative executable/filter/report paths appropriately before switching cwd; arguments
   with spaces must remain single arguments. Use `Command`, no shell.
3. Run pandoc in scratch with `-f commonmark_x-tex_math_dollars-emoji -t latex -s` to
   `note.tex`, filters in the established order, the note header, and variables for
   geometry and colors. Omit `--toc`, `--number-sections`, and `--pdf-engine`. Retain
   syntax highlighting/code wrapping and optional listen metadata.
4. Set PDF Info through escaped Pandoc metadata: Title = identity title, Subject =
   `Snapshot of <vault-relative path>`, Keywords = `bob, snapshot, <kind>`, and Creator
   = `bob ref create (note snapshot)`. Preserve the display-title/identity distinction.
   Never splice user text into `hypersetup` strings.
5. Run xelatex with cwd=scratch and `-interaction=nonstopmode -halt-on-error`, producing
   `note.pdf`. Run twice for references/footer; run a bounded third pass if the second
   log requests `Rerun`. Stop immediately on any failed pass.
6. Scan available logs even on failure. Return typed, deduplicated warnings for
   `Missing character` and `Overfull` wider than 20pt; 20pt and smaller do not warn. On
   failure include the first `!` diagnostic plus ten following context lines, exit
   information, and `render_tex::render_failure_hint` when applicable. Fall back to
   captured stderr/stdout if no useful log exists. Handle missing executables, failed
   pandoc, absent logs, and success-without-output clearly.
7. Return the PDF and `return_links::read_report` outcome without stamping or
   installing. Leave debug artifacts under existing ScratchDir ownership.

## 5. Extend doctor and the installation instructions

Add every package loaded by `note.tex` to `RENDER_TEX_PACKAGES`, with correct TeX Live
package mappings. Expected additions include titlesec, enumitem, fancyhdr, lastpage,
refcount, and xurl; fontspec, needspace, and tikz/pgf already exist. Extend
`header_packages_are_all_declared` to inspect the embedded note header. Update
package-count expectations and fake kpsewhich cases in the doctor tests as necessary;
preserve warning-level behavior and the existing package hint.

Update the BasicTeX `tlmgr --usermode install` example and package explanation in
`docs/highlights-create.md` to include the new requirements. Avoid stale hard-coded
package totals. Briefly document the xelatex override and kept note-render files where
renderer environment variables are documented, without prematurely claiming that the
CLI's snapshot route is already wired.

## 6. Verify behavior and appearance, then close the phase

Keep tests focused on observable behavior:

- Hermetic fake pandoc/xelatex tests verify flags, cwd, filter order, argument
  boundaries, two passes, conditional third pass, stopping on failure, output
  validation, warnings at the >20pt boundary, missing-glyph detection, missing package
  hints, and useful context when logs or diagnostics are absent. Use the repository's
  environment-isolation helpers for command overrides.
- Pandoc-gated AST/TeX checks cover every contract construct, mixed/nested lists, all
  statuses, metadata Markdown, live chip links, anchor preservation, and the return-link
  opt-out. Include `% & _ # $ { }` and backslashes as literal text, adversarial
  title/URL labels, long tokens, `@BANK`, `$69,044.29`, and literal `:shortcode:`. Do
  not reintroduce struck links into the projected contract.
- A real pandoc+xelatex test renders the synthetic fixture with zero missing glyphs and
  no >20pt overfull warnings. Inspect PDF font dictionaries with lopdf (and pdffonts
  when available) for zero Type 3 fonts. Assert bookmarks, PDF Info, internal and
  obsidian URI annotations, resolved footer `1 / N`, and paired return links with valid
  destinations using the existing test patterns.
- Use pdftotext to check that an isolated task row contains its words/chips and no
  status symbol, test words such as `office` for ligature-safe extraction, and check
  literal special characters and long-token text preservation. Include the parent
  design's corpus glyph string and the per-role `▸` regression. Clear skips are allowed
  only when external render tools are absent; pure and fake command tests always run.
- Generate a short and a >8-page synthetic document to verify both navigation branches,
  page labels, headings near breaks, and a wrapped task with children. Render
  representative pages to PNG with pdftoppm and actually inspect them with the
  image-view tool. Tune the masthead, chip baseline, indents, spacing, and footer until
  they match the design. Keep generated PDFs/PNGs out of the repo.
- Run targeted profile, coverage, return-link, and package/doctor tests, then
  `just check`. Use `/sase_monitor` for a long command when a handoff is needed; never
  leave a yielded process running at turn end. If a check fails identically on the clean
  base, record the reproduction as `PROPOSED FOLLOW-UP:` on this phase (cite an existing
  tracker if one exists) and close after in-scope checks pass. Do not broaden this phase
  to fix an unrelated baseline failure.

Record discoveries only with `sase bead note bob-cli-87.4 'PROPOSED FOLLOW-UP: ...'`; do
not create task beads. Document the final renderer API and verification evidence for the
dependent create-wiring agent. Run `sase bead epic-symbols bob-cli-87.4` immediately
before close. Resolve remaining entries or re-key legitimate future integration entries
to a still-open bead, such as the create-wiring phase, checking its current status
first. Do not add broad lint exemptions to hide unfinished work.

Finally run
`sase bead close bob-cli-87.4 --note "<renderer, PDF/text/visual checks, and just check results actually verified>"`.
Do not manually change bead status or close `bob-cli-87`, any ancestor plan, or another
phase. Use `/sase_final` as the last action before the normal implementation turn's
final response.
