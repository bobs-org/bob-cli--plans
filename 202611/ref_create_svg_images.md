---
tier: epic
title: Print SVG image embeds in bob ref create Markdown PDFs
goal: 'A Markdown bob ref create never fails on an SVG image embed; each SVG prints
  as a vector PDF, else a same-stem sibling raster, else a visible placeholder with
  a warning.

  '
decisions:
  recapture:
    ask: Should the final phase install the new bob and re-run the keep_moving_link_marks
      capture?
    default: true
    why: The hook already does this on every report write; the failure only left the
      reference missing.
    answer: true
phases:
- id: svg-image-guard
  title: Always-on SVG image guard
  depends_on: []
  size: medium
  description: 'svg-image-guard: add an always-on Lua image filter that swaps SVG
    embeds for a same-stem sibling raster or an Image-not-printable placeholder, reports
    them as images:/warning: lines, fixes the misleading svg.sty hint, and documents
    the behavior.'
- id: svg-vector-render
  title: Vector SVG rendering through the Chrome diagram renderer
  depends_on:
  - svg-image-guard
  size: medium
  description: 'svg-vector-render: extend the Mermaid diagram pass and adapter protocol
    so local SVG embeds render to vector PDFs in the same browser launch, falling
    back to the guard, then optionally recapture the failed report.'
proposed_by: bbugyi200.apollo.d9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 07:53:16
status: done
bead_id: bob-cli-k6
---

- **PROMPT:** [prompts/202611/ref_create_svg_images.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/ref_create_svg_images.md)
- **BEAD:** [bob-cli-k6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k6/README.md)

# Print SVG image embeds in `bob ref create` Markdown PDFs

## Why the hook failed

The SASE file hook ran `bob_research_capture` on the research report
`202611/keep_moving_link_marks/keep_moving_link_marks.md` (in the `bob-cli--research`
sidecar; open it with `sase repo open bob-cli--research`). The hook called
`bob ref create <report> --include-id -N keep_moving_link_marks -P bob-cli -f`, which
exited 1 because pandoc exited 43.

- Line 92 of the report embeds an SVG figure carried over from the grk sub-report:
  `![Which projects should keep-moving preview?](leaf_task_link_keep_moving_mark__grk_assets/work-leaf.svg)`.
  A same-stem `work-leaf.png` (and `work-leaf.mmd`) sits next to it, because the
  `mermaid_diagrams` skill always renders SVG and PNG.
- The generic Markdown route (`render_temp_pdf_from` in
  `src/native/highlights_ref/create.rs`) passes image embeds straight to pandoc.
  pandoc's PDF writer converts SVG through `rsvg-convert`. That tool is not installed on
  this host, so pandoc falls back to `\includesvg` plus `\usepackage{svg}`, and XeLaTeX
  aborts on the missing `svg.sty`. The failure reproduces with a two-line Markdown file
  and plain `pandoc --pdf-engine=xelatex`, while a PNG embed renders fine.
- bob's hint (`render_tex::render_failure_hint`) told the user to find `svg.sty` with
  `tlmgr search`. That is the wrong direction: `svg.sty` would then need Inkscape and
  `--shell-escape`.
- Installing librsvg is not a good fix either. Mermaid CLI SVGs put every label in
  `<foreignObject>` (12 of them in `work-leaf.svg`), and librsvg does not render
  `foreignObject`. The figure would print as empty boxes with no error.
- `docs/highlights-create.md` already lists "SVG image embeds" as outside the diagram
  renderer's scope. The vault-note snapshot route already turns SVG into an
  `Image not printable` placeholder (`note_snapshot/assets.rs`). Only the generic
  Markdown route aborts the whole capture.

This is 1 failure in 148 research-capture runs. However, nine per-model sub-reports
(`*__grk.md`, `*__cdx.md`, `*__gem.md`) already embed SVGs, and the hook skips those.
Every consolidated report that carries one of those figures over will fail the same way.
Fixing this belongs in bob, not in the hook script, the research prompts, or the host's
packages.

## Goal

A Markdown `bob ref create` never fails because of an SVG image embed, and each SVG
figure prints in the best form available. In priority order:

1. a vector PDF from the Chrome diagram renderer (phase `svg-vector-render`),
2. a same-stem sibling raster (`.png`, `.jpg`, `.jpeg`),
3. a visible `Image not printable` placeholder with a `warning:` line.

## Phase: svg-image-guard

- id: `svg-image-guard`
- size: medium
- depends on: none

This phase makes the capture succeed with no new host dependency. It ships on its own.

1. Add an always-on pandoc Lua filter, `image_guard.lua`, next to the other
   `highlights_ref` filters. Embed it with `include_str!`, write it in
   `write_render_filters`, add its path and a report path to `RenderFilters`, and pass
   both in `render_temp_pdf_from`. It must run after the diagrams swap filter, so a
   later vector swap wins. Pass the original source's directory (the same directory as
   `--resource-path`) as metadata such as `bob-image-base`. The frozen copy lives in
   scratch, so that directory cannot be inferred from the input path.
2. For every `Image` whose target ends in `.svg` (case-insensitive, ignoring any
   `?query` or `#fragment`):
   - If the target is local and a same-stem sibling `.png`, `.jpg`, or `.jpeg` exists,
     checked in that order, rewrite `src` to that sibling. Check both the raw and the
     percent-decoded target. Keep the alt text, caption, identifier, and attributes such
     as `width`.
   - Otherwise (no sibling, an `http(s)` URL, or a `data:` URI), replace the image with
     the same `Image not printable · <target>` placeholder wording that the snapshot
     route uses (`note_snapshot/serialize.rs`). Inside a pandoc `Figure`, keep the
     caption.
   - Like `diagrams.lua`, the filter must never make pandoc fail. Wrap file probes and
     report writes in `pcall`.
3. Have the filter write a JSON report listing each handled image as
   `{target, outcome: "sibling" | "placeholder", replacement}`. Read it in Rust in the
   style of `table_layout::read_report` / `emit_outcome`, and emit it wherever
   `table_layout::emit_outcome` is called today (generic create, the overwrite path, and
   the `-M` plain route):
   - one `images:` summary line, for example
     `images: 1 SVG image printed from its sibling PNG`;
   - one `warning:` line per placeholder, naming the target and the fix: add a same-stem
     PNG, or use a Mermaid fence.

   Warnings never change the exit status.

4. Extend `render_tex::render_failure_hint`. When the detail mentions `svg.sty` or
   `rsvg-convert`, return an SVG-specific hint saying that an SVG image could not be
   converted and to add a same-stem PNG next to it. Check this before the generic
   missing-file branch, so `tlmgr search` is no longer suggested for `svg.sty`. Add a
   unit test next to the existing hint tests.
5. Tests, following the pandoc-gated test patterns already used by `table_layout` and
   `diagrams`:
   - the filter swaps to a sibling PNG and keeps the caption and width;
   - it falls back to a placeholder when no sibling exists, and for a remote URL;
   - it strips a query string before matching;
   - it leaves non-SVG images untouched;
   - an end-to-end render of a Markdown file with an SVG embed succeeds with
     `rsvg-convert` hidden from `PATH` (both the sibling and the placeholder case);
   - report parsing and output lines are covered.
6. Docs: per the `docs` reference memory, this is a behavior change, so update
   `docs/highlights-create.md`. Describe SVG embeds in the Markdown route (sibling
   raster, then placeholder plus warning). In the `docs/reference/compatibility.md`
   Markdown requirements, note that `rsvg-convert` is not needed. Run `just docs-check`.
7. Run `just check` (or the repo's standard Rust gate) and keep it green.

## Phase: svg-vector-render

- id: `svg-vector-render`
- size: medium
- depends on: `svg-image-guard`

This phase upgrades local SVG embeds to vector PDFs. It uses the Chrome renderer that
already turns Mermaid fences into PDFs, inside the same browser launch.

1. Collect: in `diagrams_collect.lua`, also record each `Image` with a local `.svg`
   target as `{id = sha1("svg\0" .. target), kind = "svg", target = target}`, in
   document order with the Mermaid items.
2. Prepare: in `diagrams.rs`:
   - Make `prescan` also count SVG image embeds, so the pass is not skipped for a report
     with no Mermaid fences.
   - Resolve each SVG target against the original source directory, not the frozen
     copy's directory. Accept only regular files, cap the size at a few MiB, and read
     them as UTF-8.
   - An unreadable target produces no item; the guard handles it.
   - SVG items share `MAX_DIAGRAM_ITEMS` and the per-item timeout.
   - `BOB_HIGHLIGHTS_DIAGRAMS=off`, a missing adapter, or a browser failure skip vector
     rendering, and the guard still applies.
3. Protocol: add `kind` (`"mermaid"` or `"svg"`) to `clip_adapter::DiagramItem`. In
   `scripts/web_clip/diagram_render.py`, make `validate_request` accept it, treating a
   missing `kind` as `mermaid`. Bump `ADAPTER_PROTOCOL_VERSION` only if the adapter's
   version handshake requires it; the adapter ships inside bob.
4. Render: add an SVG branch to `_render_one`.
   - Load the SVG as an `<img>` from a `data:image/svg+xml;base64,` URI. In SVG-as-image
     mode, scripts and event handlers never run, and the context still aborts every
     network route.
   - Await `img.decode()`, take the size from `naturalWidth` and `naturalHeight`
     (falling back to `viewBox`), and call `page.pdf` at that size, as the Mermaid path
     does.
   - Verify that `foreignObject` labels print and that the text stays vector, using the
     existing vector-font and no-raster checks plus `pdftotext` containing a label. If
     SVG-as-image loses the labels in Chrome, switch to inline DOM insertion with
     `<script>` elements and `on*` attributes stripped, and record why.
5. Swap: in `diagrams.lua`, add an `Image` handler. When a local `.svg` target has a
   rendered PDF, set `src` to the PDF and keep the caption and attributes. Apply the
   Mermaid fit budget (`\BobFitBegin{}`/`\BobFitEnd{}`) when the image is the only
   inline in its paragraph or figure. On failure, leave the image unchanged, so the
   guard's sibling or placeholder applies. Include the renderer's error in the guard's
   warning when one is available.
6. Output: keep the existing Mermaid `diagrams:` line format and add an SVG count, for
   example `svg: rendered 1 of 1 SVG images`. In the guard summary, count only the
   images the guard handled.
7. Tests:
   - Rust prepare and collect tests, including an SVG-only document and a kill-switch
     run.
   - Adapter request validation for `kind`.
   - A gated live render test using a small SVG fixture with a `foreignObject` label,
     asserting the PDF text contains the label.
   - An end-to-end test with a fake adapter showing that a failed SVG item falls back to
     the sibling PNG.
8. Docs: in `docs/highlights-create.md`, remove "SVG image embeds" from the out-of-scope
   sentence and describe the order: vector render, then sibling raster, then
   placeholder. Run `just docs-check` and the repo's standard gate, including
   `just check-web-clip-adapter`.

> [!decision] recapture After this phase is committed:
>
> 1. Run `just install` from the updated checkout.
> 2. Open the research sidecar with `sase repo open bob-cli--research`.
> 3. Re-run the hook command with `SASE_FILE_HOOK_PROJECT=bob-cli` set:
>    `bob_research_capture <research>/202611/keep_moving_link_marks/keep_moving_link_marks.md`.
> 4. Confirm it exits 0 and prints the expected `images:`/`svg:` lines.
>
> No reference exists yet, so this is a first capture.

## Out of scope

- The vault-note snapshot route keeps its current SVG placeholder behavior. Routing
  snapshot SVG assets through the same renderer can follow as its own task.
- Other formats XeLaTeX cannot include and pandoc cannot convert without external tools
  (WebP, AVIF, HEIC) are not handled. Extend the guard's extension list only after a
  test proves one of them aborts a render.
- No change to `bob_research_capture`, the research prompts, the `mermaid_diagrams`
  skill, or host packages.
